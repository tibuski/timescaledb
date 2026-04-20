# TimescaleDB

## Status
- Services running
- **374,876,230 rows imported** (all columns properly filled)
- Database size: **56 GB**
- Chunks: **~1,500+**
- pgAdmin pre-configured with TimescaleDB server

## Services

### TimescaleDB
- Image: timescale/timescaledb-ha:pg18
- Port: 5432
- User: postgres / admin
- Limits: 20GB RAM, 5 CPU
- Database: datalake

### pgAdmin (web UI)
- URL: http://192.168.25.20:8080
- Email: admin@example.com
- Password: admin
- Pre-configured server: TimescaleDB → timescaledb:5432

## InfluxDB2 Migration (Reference)

These commands were used to migrate from InfluxDB2 to the LP file format:

```bash
# Remove old volume (if exists)
docker volume rm influx_v2_data

# Create temporary InfluxDB2 container
docker run -d --user 1000:1000 --name influx_converter \
  -p 8086:8086 \
  -v $(pwd):/backup \
  -v influx_v2_data:/var/lib/influxdb2 \
  -e DOCKER_INFLUXDB_INIT_MODE=setup \
  -e DOCKER_INFLUXDB_INIT_USERNAME=admin \
  -e DOCKER_INFLUXDB_INIT_PASSWORD=adminpassword \
  -e DOCKER_INFLUXDB_INIT_ORG=Tech-ABNF \
  -e DOCKER_INFLUXDB_INIT_BUCKET=temp_bucket \
  -e DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=MON_SUPER_TOKEN_SECRET_123 \
  influxdb:2.7

sleep 15

# Restore backup to InfluxDB2
docker exec influx_converter influx restore \
  --token MON_SUPER_TOKEN_SECRET_123 \
  --org Tech-ABNF \
  --bucket DataLake_ST \
  /backup

# Find Bucket ID
docker exec influx_converter influx bucket list --token MON_SUPER_TOKEN_SECRET_123

# Export to Line Protocol format
docker exec influx_converter influxd inspect export-lp \
  --bucket-id <BUCKET_ID> \
  --engine-path /var/lib/influxdb2/engine \
  --output-path /backup/migration_datalake.lp

# Count lines in exported file
wc -l migration_datalake.lp
```

## Start/Stop

```bash
cd /home/tibus/docker/timescaledb

# Start
docker compose up -d

# Stop
docker compose down

# Restart specific service
docker compose restart timescaledb
docker compose restart pgadmin

# View logs
docker compose logs -f timescaledb

# Check status
docker compose ps
```

## Access pgAdmin
- URL: http://192.168.25.20:8080
- Email: admin@example.com
- Password: admin
- Look for "Servers" → "TimescaleDB" in sidebar

## Import Data

### Source
- File: `/home/tibus/docker/backup/migration_datalake.lp`
- Format: InfluxDB Line Protocol
- Size: 128 GB (~375M lines)

### Schema
```sql
measurements (
    time         BIGINT NOT NULL,    -- nanoseconds timestamp
    measurement TEXT,               -- CLIMATE, OCCUPANCY, ENERGY, etc.
    device_uid   TEXT,              -- device identifier
    site        TEXT,               -- site name
    source      TEXT,              -- LORA, etc.
    category    TEXT,              -- BUILD, etc.
    measure_name TEXT,            -- co2, temperature, occupancy, etc.
    measure_id  TEXT,             -- measurement identifier
    value       DOUBLE PRECISION  -- numeric value
)
```

### Reimport (if needed)
```bash
# 1. Drop and recreate table
docker exec timescaledb psql -U postgres -d datalake -c "
DROP TABLE IF EXISTS measurements;
CREATE TABLE measurements (
    time BIGINT NOT NULL, measurement TEXT, device_uid TEXT, site TEXT,
    source TEXT, category TEXT, measure_name TEXT, measure_id TEXT, value DOUBLE PRECISION
);
SELECT create_hypertable('measurements', 'time', chunk_time_interval => 86400000000000);
"

# 2. Run import
docker exec -d timescaledb python3 /import_lp.py /backup/migration_datalake.lp

# 3. Monitor progress
docker exec timescaledb psql -U postgres -d datalake -c "SELECT count(*) FROM measurements"
```

### Import Script

The import script (`/import_lp.py`) parses InfluxDB Line Protocol files:

- **Location**: Already mounted in container at `/import_lp.py`
- **Input**: LP file path (default: `/backup/migration_datalake.lp`)
- **Batch size**: 50,000 rows
- **Skip lines**: Can resume from specific line (SKIP_LINES parameter)
- **Skips invalid values**: String values like `"LVL_NONE&SEND"` are skipped

#### Usage

```bash
# Basic import
docker exec -d timescaledb python3 /import_lp.py /backup/migration_datalake.lp

# Resume from line X (if interrupted)
# Edit import_lp.py and set SKIP_LINES = X
# Then run the same command
```

#### How It Works

1. Read file line by line (8MB buffer)
2. Parse measurement, tags, field, timestamp
3. Skip lines with string values (invalid)
4. Batch insert via PostgreSQL COPY
5. Commit after each batch (ensures durability)
6. Print progress every 50,000 rows

## Monitoring

### Check row count
```bash
docker exec timescaledb psql -U postgres -d datalake -c "SELECT count(*) FROM measurements"
```

### Check database size
```bash
docker exec timescaledb psql -U postgres -d datalake -c "SELECT pg_size_pretty(pg_database_size('datalake'))"
```

### Check chunks
```bash
docker exec timescaledb psql -U postgres -d datalake -c "
SELECT count(*) FROM timescaledb_information.chunks WHERE hypertable_name = 'measurements'
"
```

## Compression

TimescaleDB native compression reduces storage 85-95%.

### Enable Compression

```bash
# Enable compression on table
docker exec timescaledb psql -U postgres -d datalake -c "
ALTER TABLE measurements SET (
    timescaledb.compress = TRUE,
    timescaledb.compress_segmentby = 'device_uid',
    timescaledb.compress_orderby = 'time DESC'
);"

# Add compression policy (compress after 1 day)
docker exec timescaledb psql -U postgres -d datalake -c "
SELECT add_compression_policy('measurements', compress_after => INTERVAL '1 day');"
```

### Monitor Compression

```bash
# Check compression ratio on chunks
docker exec timescaledb psql -U postgres -d datalake -c "
SELECT
    chunk_name,
    before_compression_size,
    after_compression_size,
    round(100 - (after_compression_size::numeric / before_compression_size::numeric) * 100, 1) as ratio
FROM timescaledb_information.chunks
WHERE hypertable_name = 'measurements'
ORDER BY before_compression_size DESC LIMIT 5;"
```

### Disable Compression

```bash
docker exec timescaledb psql -U postgres -d datalake -c "
SELECT remove_compression_policy('measurements');"
```

### Configuration Explained

| Setting | Value | Purpose |
|---------|-------|---------|
| segmentby | device_uid | Groups rows by device - queries by device only decompress matching segments |
| orderby | time DESC | Sort order within segment - efficient range scans |
| compress_after | 1 day | Recent data stays uncompressed for fast writes/queries |

## Example Queries

These queries are converted from InfluxDB v2 - adjust column names as needed based on your actual data.

### Notes
- `time` is stored as **nanoseconds** (bigint)
- Convert to timestamp: `to_timestamp(time / 1000000000)`
- Time buckets use: `time_bucket('interval', to_timestamp(time / 1000000000))`
- Column mapping (verify in your data):
  - `SiteTech` → likely `site`
  - `LocalisationTech` → may be in tags or use `device_uid`
  - `_field` → `measure_name`
  - `_value` → `value`
  - `Source` → `source`

---

### Query 1: Occupancy counts for TRONE (0/1 sensors)

**InfluxDB:**
```flux
from bucket(xxx)
|> range(start:-xxd)
|> filter(fn: (r) => r["MeasureID"] == "occupancy")
|> filter(fn: (r) => r["MeasureName"] == "occupancy")
|> filter(fn: (r) => r["SiteTech"] == "BRTR")
|> group()
```

**TimescaleDB SQL:**
```sql
-- Daily occupancy counts for a site
SELECT 
    time_bucket('1 day', to_timestamp(time / 1000000000)) AS day,
    site,
    measure_name,
    COUNT(*) AS count
FROM measurements
WHERE time >= (EXTRACT(EPOCH FROM now()) * 1000000000 - 30 * 86400000000000)  -- last 30 days
  AND measure_name = 'occupancy'
  AND site = 'BRTR'
GROUP BY day, site, measure_name
ORDER BY day DESC;

-- Hourly occupancy counts
SELECT 
    time_bucket('1 hour', to_timestamp(time / 1000000000)) AS hour,
    site,
    measure_name,
    COUNT(*) AS count,
    SUM(value) AS total_value  -- if 0/1, sum = occupancy minutes
FROM measurements
WHERE time >= (EXTRACT(EPOCH FROM now()) * 1000000000 - 1 * 86400000000000)  -- last 1 day
  AND measure_name = 'occupancy'
  AND site = 'BRTR'
GROUP BY hour, site, measure_name
ORDER BY hour DESC;
```

---

### Query 2: Count messages grouped by Type (measurement)

**InfluxDB:**
```flux
|> range(start: 2024-01-01T00:00:00Z, stop:2025-01-01T00:00:00Z)
|> keep(columns:["_field","Source","_value"])
|> group(columns:["_field"])
|> count()
```

**TimescaleDB SQL:**
```sql
-- Count by measurement type (CLIMATE, OCCUPANCY, ENERGY, etc.)
SELECT 
    measurement,
    COUNT(*) AS count
FROM measurements
WHERE time >= 1704067200000000000  -- 2024-01-01
  AND time < 1735689600000000000   -- 2025-01-01
GROUP BY measurement
ORDER BY count DESC;

-- Count by measure_name
SELECT 
    measure_name,
    COUNT(*) AS count
FROM measurements
WHERE time >= 1704067200000000000
  AND time < 1735689600000000000
GROUP BY measure_name
ORDER BY count DESC;

-- Count by Source
SELECT 
    source,
    measurement,
    COUNT(*) AS count
FROM measurements
WHERE time >= 1704067200000000000
  AND time < 1735689600000000000
GROUP BY source, measurement
ORDER BY count DESC;
```

---

### Query 3: Climate data for specific zones (line charts)

**InfluxDB:**
```flux
|> filter(fn: (r) => r["LocalisationTech"] == "BRTR.R2.+65.PEP.BUILD.ROOM.OPS" 
    or r["LocalisationTech"] == "BRTR.R2.+65.BLV.BUILD.ROOM.OPS" 
    or r["LocalisationTech"] == "BRTR.R2.+55.PEP.BUILD.ROOM.OPS" 
    or r["LocalisationTech"] == "BRTR.R2.+55.BLV.BUILD.ROOM.OPS")
|> sort(columns: ["_field","_time"])
```

**TimescaleDB SQL:**
```sql
-- Climate data for specific locations (adjust column based on your data)
-- Note: You may need to filter by device_uid or check which column contains LocalisationTech
SELECT 
    time_bucket('5 minutes', to_timestamp(time / 1000000000)) AS time,
    device_uid,
    measure_name,
    AVG(value) AS value
FROM measurements
WHERE time >= (EXTRACT(EPOCH FROM now()) * 1000000000 - 7 * 86400000000000)  -- last 7 days
  AND measurement = 'CLIMATE'
  AND device_uid LIKE 'BRTR.R2.%'
GROUP BY time, device_uid, measure_name
ORDER BY time DESC, device_uid;

-- If you have a specific location column, use IN:
-- SELECT ... WHERE location IN ('BRTR.R2.+65.PEP.BUILD.ROOM.OPS', 'BRTR.R2.+65.BLV.BUILD.ROOM.OPS', ...)

-- Multiple measures (temperature, humidity, co2) for charting
SELECT 
    time_bucket('10 minutes', to_timestamp(time / 1000000000)) AS time,
    measure_name,
    AVG(value) AS value
FROM measurements
WHERE time >= (EXTRACT(EPOCH FROM now()) * 1000000000 - 1 * 86400000000000)
  AND measurement = 'CLIMATE'
  AND measure_name IN ('temperature', 'humidity', 'co2')
GROUP BY time, measure_name
ORDER BY time DESC;
```

---

### Query 4: Time range examples

```sql
-- Last 24 hours
WHERE time >= (EXTRACT(EPOCH FROM now()) * 1000000000 - 1 * 86400000000000)

-- Last 30 days
WHERE time >= (EXTRACT(EPOCH FROM now()) * 1000000000 - 30 * 86400000000000)

-- Specific date range
WHERE time >= 1704067200000000000  -- 2024-01-01 00:00:00 UTC
  AND time < 1735689600000000000   -- 2025-01-01 00:00:00 UTC
```

## Configuration Files

### docker-compose.yaml

```yaml
services:
  timescaledb:
    container_name: timescaledb
    image: timescale/timescaledb-ha:pg18
    environment:
      POSTGRES_PASSWORD: admin
      PGDATA: /pgdata
    volumes:
      - type: bind
        source: ./data
        target: /pgdata
      - type: bind
        source: /home/tibus/docker/backup
        target: /backup
      - type: bind
        source: /home/tibus/docker/timescaledb/import_lp.py
        target: /import_lp.py
    ports:
      - "5432:5432"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 20G
          cpus: '5'
        reservations:
          memory: 4G
          cpus: '1'
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  pgadmin:
    image: dpage/pgadmin4
    container_name: pgadmin
    restart: unless-stopped
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
      PGADMIN_SERVER_JSON_FILE: /pgadmin4/servers.json
      PGADMIN_REPLACE_SERVERS_ON_STARTUP: "True"
    ports:
      - "8080:80"
    volumes:
      - pgadmin_data:/var/lib/pgadmin
      - ./pgadmin-config/servers.json:/pgadmin4/servers.json:ro
    depends_on:
      timescaledb:
        condition: service_healthy

volumes:
  pgadmin_data:
```

### pgadmin-config/servers.json

```json
{
  "Servers": {
    "1": {
      "Name": "TimescaleDB",
      "Group": "Servers",
      "Host": "timescaledb",
      "Port": 5432,
      "MaintenanceDB": "postgres",
      "Username": "postgres",
      "Passwd": "admin",
      "SSLMode": "prefer"
    }
  }
}
```

### import_lp.py

```python
#!/usr/bin/env python3
import sys
import os
import re
import time

import psycopg2

TAG_PATTERN = re.compile(r'([^,]+)=([^,]+)')

BATCH_SIZE = 50000
SKIP_LINES = 0

def parse_line(line):
    line = line.strip()
    if not line:
        return None
    
    parts = line.split(' ')
    if len(parts) < 2:
        return None
    
    timestamp = parts[-1]
    last_field = parts[-2]
    
    measurement_and_tags = ' '.join(parts[:-2])
    first_comma = measurement_and_tags.find(',')
    if first_comma < 0:
        return None
    
    measurement = measurement_and_tags[:first_comma]
    tags_str = measurement_and_tags[first_comma+1:]
    
    eq_pos = last_field.find('=')
    if eq_pos < 0:
        return None
    
    field_name = last_field[:eq_pos]
    field_value = last_field[eq_pos+1:]
    
    # Parse tags - use comma as delimiter
    tags = {}
    for pair in tags_str.split(','):
        if '=' in pair:
            key, val = pair.split('=', 1)
            tags[key] = val
    
    try:
        if field_value.endswith('i'):
            value = float(field_value[:-1])
        elif field_value.startswith('"') and field_value.endswith('"'):
            return None
        else:
            value = float(field_value)
    except (ValueError, IndexError):
        return None
    
    return (
        int(timestamp),
        measurement,
        tags.get('DeviceUID', ''),
        tags.get('Site', ''),
        tags.get('Source', ''),
        tags.get('Category', ''),
        tags.get('MeasureName', field_name),
        tags.get('MeasureID', ''),
        value
    )

def import_data(lp_file, batch_size=BATCH_SIZE, skip_lines=SKIP_LINES, table='measurements'):
    conn = psycopg2.connect(
        host='localhost',
        port=5432,
        dbname='datalake',
        user='postgres',
        password='admin',
        options='-c client_encoding=UTF8'
    )
    conn.autocommit = False
    
    cur = conn.cursor()
    
    total_rows = 0
    batch = []
    start_time = time.time()
    
    print(f"Reading from: {lp_file}")
    print(f"Skipping first {skip_lines:,} lines")
    print(f"Batch size: {batch_size}")
    print("Starting import...")
    
    lines_read = 0
    skipped = 0
    with open(lp_file, 'r', buffering=8192*1024) as f:
        for line in f:
            lines_read += 1
            
            if lines_read <= skip_lines:
                if lines_read % 10000000 == 0:
                    print(f"Skipped: {lines_read:,}")
                continue
            
            row = parse_line(line)
            if row:
                batch.append(row)
                if len(batch) >= batch_size:
                    insert_batch(cur, conn, batch, table)
                    total_rows += len(batch)
                    batch = []
                    
                    now = time.time()
                    elapsed = now - start_time
                    rate = total_rows / elapsed if elapsed > 0 else 0
                    print(f"Imported: {total_rows:,} rows | Rate: {rate:,.0f} rows/sec | Elapsed: {elapsed/60:.1f} min")
    
    if batch:
        insert_batch(cur, conn, batch, table)
        total_rows += len(batch)
    
    cur.close()
    conn.close()
    
    elapsed = time.time() - start_time
    print(f"\n=== COMPLETE ===")
    print(f"Total rows: {total_rows:,}")
    print(f"Time: {elapsed/60:.1f} minutes")
    print(f"Average rate: {total_rows/elapsed:,.0f} rows/sec")

def insert_batch(cur, conn, batch, table='measurements'):
    from io import StringIO
    buffer = StringIO()
    for row in batch:
        buffer.write('\t'.join(str(x) for x in row) + '\n')
    buffer.seek(0)
    
    try:
        cur.copy_expert(
            f"COPY {table} (time, measurement, device_uid, site, source, category, measure_name, measure_id, value) FROM STDIN",
            buffer
        )
        conn.commit()
    except Exception as e:
        print(f"ERROR: {e}")
        print(f"First row: {batch[0] if batch else 'empty'}")
        raise

if __name__ == '__main__':
    import sys as _sys
    lp_file = _sys.argv[1] if len(_sys.argv) > 1 else '/home/tibus/docker/backup/migration_datalake.lp'
    
    if not os.path.exists(lp_file):
        print(f"Error: File not found: {lp_file}")
        sys.exit(1)
    
    print(f"File size: {os.path.getsize(lp_file) / (1024**3):.1f} GB")
    import_data(lp_file)
```