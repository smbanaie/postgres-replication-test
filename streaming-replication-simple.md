# PostgreSQL 18 Streaming Replication - Simple Hands-On Tutorial

## What We'll Build

A PostgreSQL cluster with:

- **1 Primary** (accepts writes)
- **1 Standby** (read-only replica)
- **Sample data** to test with

Total time: **15 minutes** ⏱️

---

## Step 1: Setup (2 minutes)

### Create Project Directory

```bash
mkdir pg-replication-tutorial
cd pg-replication-tutorial
```

### Create Docker Compose File

**File: `docker-compose.yml`**

```yaml
version: '3.8'

services:
  # Primary Database
  primary:
    image: postgres:18
    container_name: pg-primary
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password123
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      - primary-data:/var/lib/postgresql/data
      - ./scripts:/scripts
    command: |
      postgres 
      -c wal_level=replica 
      -c max_wal_senders=3 
      -c max_replication_slots=3
      -c hot_standby=on
    networks:
      - pg-network

  # Standby Database (Replica)
  standby:
    image: postgres:18
    container_name: pg-standby
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password123
    ports:
      - "5433:5432"
    volumes:
      - standby-data:/var/lib/postgresql/data
    depends_on:
      - primary
    networks:
      - pg-network

volumes:
  primary-data:
  standby-data:

networks:
  pg-network:
```

---

## Step 2: Start Primary Database (1 minute)

```bash
# Start only the primary
docker-compose up -d primary

# Wait a few seconds
sleep 5

# Check it's running
docker-compose ps
```

You should see:

```
NAME          STATUS
pg-primary    Up
```

---

## Step 3: Configure Primary for Replication (2 minutes)

### Create Replication User

```bash
docker exec -it pg-primary psql -U postgres << 'EOF'
-- Create replication user
CREATE USER replicator WITH REPLICATION PASSWORD 'replicator123';

-- Grant necessary permissions
GRANT CONNECT ON DATABASE myapp TO replicator;
EOF
```

### Allow Replication Connections

```bash
docker exec -it pg-primary bash -c "echo 'host replication replicator 0.0.0.0/0 md5' >> /var/lib/postgresql/data/pg_hba.conf"

# Reload configuration
docker exec -it pg-primary psql -U postgres -c "SELECT pg_reload_conf();"
```

---

## Step 4: Create Sample Data (2 minutes)

```bash
docker exec -it pg-primary psql -U postgres -d myapp << 'EOF'
-- Create tables
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    title VARCHAR(200),
    content TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Insert sample data
INSERT INTO users (username, email) VALUES
    ('alice', 'alice@example.com'),
    ('bob', 'bob@example.com'),
    ('charlie', 'charlie@example.com');

INSERT INTO posts (user_id, title, content) VALUES
    (1, 'First Post', 'Hello World!'),
    (1, 'Second Post', 'PostgreSQL is awesome'),
    (2, 'Bobs Post', 'Learning replication');

-- Verify data
SELECT COUNT(*) as user_count FROM users;
SELECT COUNT(*) as post_count FROM posts;
EOF
```

---

## Step 5: Setup Standby Database (3 minutes)

### Take Base Backup

```bash
# Create backup using pg_basebackup
docker exec -it pg-primary pg_basebackup \
    -h localhost \
    -U replicator \
    -p 5432 \
    -D /tmp/standby_backup \
    -Fp \
    -Xs \
    -P \
    -R

# Copy backup to standby container
docker cp pg-primary:/tmp/standby_backup/. /tmp/standby_backup/

# Stop standby if running
docker-compose stop standby

# Copy backup to standby volume
docker run --rm \
    -v $(pwd)/tmp/standby_backup:/source \
    -v pg-replication-tutorial_standby-data:/target \
    busybox sh -c "cp -r /source/* /target/"

# Or simpler approach - just use docker cp
docker start pg-standby 2>/dev/null || true
sleep 2
docker exec pg-primary tar -czf /tmp/backup.tar.gz -C /tmp/standby_backup .
docker cp pg-primary:/tmp/backup.tar.gz /tmp/
docker cp /tmp/backup.tar.gz pg-standby:/tmp/
docker exec pg-standby bash -c "rm -rf /var/lib/postgresql/data/* && tar -xzf /tmp/backup.tar.gz -C /var/lib/postgresql/data"
```

### primary_conninfo

### Simple Method (Recommended)

Let's use an easier approach:

```bash
# Stop standby container
docker-compose stop standby

# Remove old data
docker volume rm pg-replication-tutorial_standby-data

# Recreate and setup
docker-compose up -d standby
sleep 3

# Take fresh backup directly to standby
docker exec pg-standby rm -rf /var/lib/postgresql/data/*

docker exec pg-primary pg_basebackup \
    -h primary \
    -U replicator \
    -D /tmp/standby_data \
    -Fp -Xs -P -R

# This creates standby.signal automatically with -R flag
```

**Even Simpler - Manual Setup:**

```bash
# Stop everything
docker-compose down -v

# Restart primary
docker-compose up -d primary
sleep 10

# Setup replication user again
docker exec pg-primary psql -U postgres << 'EOF'
CREATE USER replicator WITH REPLICATION PASSWORD 'replicator123';
EOF

docker exec pg-primary bash -c "echo 'host replication replicator 0.0.0.0/0 md5' >> /var/lib/postgresql/data/pg_hba.conf"
docker exec pg-primary psql -U postgres -c "SELECT pg_reload_conf();"

# Create sample data again
docker exec pg-primary psql -U postgres -d myapp << 'EOF'
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO users (username, email) VALUES
    ('alice', 'alice@example.com'),
    ('bob', 'bob@example.com'),
    ('charlie', 'charlie@example.com');
EOF

# Now setup standby properly
docker exec pg-standby pg_basebackup \
    -h primary \
    -U replicator \
    -D /var/lib/postgresql/data \
    -Fp -Xs -P -R -c fast

# Start standby
docker-compose restart standby
```

---

## Step 6: The EASIEST Way (Start Fresh)

Let's do this the simplest way possible:

```bash
# Clean up everything
docker-compose down -v
rm -rf scripts/
mkdir scripts
```

**File: `scripts/setup.sh`**

```bash
#!/bin/bash
sleep 10  # Wait for primary to be ready

# Setup replication user
PGPASSWORD=password123 psql -h primary -U postgres << 'EOF'
CREATE USER replicator WITH REPLICATION PASSWORD 'replicator123';
EOF

# Configure pg_hba.conf
docker exec pg-primary bash -c "echo 'host replication replicator 0.0.0.0/0 md5' >> /var/lib/postgresql/data/pg_hba.conf"
docker exec pg-primary psql -U postgres -c "SELECT pg_reload_conf();"

# Create sample data
PGPASSWORD=password123 psql -h primary -U postgres -d myapp << 'EOF'
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO users (username, email) VALUES
    ('alice', 'alice@example.com'),
    ('bob', 'bob@example.com');
EOF

# Setup standby
PGPASSWORD=replicator123 pg_basebackup \
    -h primary \
    -U replicator \
    -D /var/lib/postgresql/data \
    -Fp -Xs -P -R -c fast
```

Actually, let me give you the **ABSOLUTE SIMPLEST** working version:

---

## SIMPLE VERSION - Step by Step

### Step 1: Create Files

```bash
mkdir pg-simple-replication
cd pg-simple-replication
```

**File: `docker-compose.yml`**

```yaml
version: '3.8'

services:
  primary:
    image: postgres:18
    container_name: primary
    environment:
      POSTGRES_PASSWORD: pass123
    ports:
      - "5432:5432"
    command: |
      postgres 
      -c wal_level=replica 
      -c hot_standby=on
      -c max_wal_senders=3
    networks:
      - pgnet

  standby:
    image: postgres:18
    container_name: standby
    environment:
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "5433:5432"
    networks:
      - pgnet

networks:
  pgnet:
```

### Step 2: Start Primary

```bash
docker-compose up -d primary
sleep 5
```

### Step 3: Setup Primary

```bash
# Create replication user
docker exec -it primary psql -U postgres -c \
    "CREATE USER repuser WITH REPLICATION PASSWORD 'reppass';"

# Allow replication connections
docker exec primary bash -c \
    "echo 'host replication repuser 0.0.0.0/0 md5' >> /var/lib/postgresql/data/pg_hba.conf"

docker exec primary psql -U postgres -c "SELECT pg_reload_conf();"

# Create sample data
docker exec -it primary psql -U postgres << 'EOF'
CREATE TABLE test_data (
    id SERIAL PRIMARY KEY,
    name TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO test_data (name) VALUES ('Alice'), ('Bob'), ('Charlie');

SELECT * FROM test_data;
EOF
```

### Step 4: Setup Standby

```bash
# Take backup from primary
docker exec standby pg_basebackup \
    -h primary \
    -U repuser \
    -D /var/lib/postgresql/data/pgdata \
    -Fp -Xs -P -R

# You'll be prompted for password: reppass

# Start standby
docker-compose restart standby
sleep 5
```

### Step 5: Test Replication

```bash
# Check replication status on primary
docker exec primary psql -U postgres -c \
    "SELECT application_name, state, sync_state FROM pg_stat_replication;"
```

You should see:

```
 application_name | state     | sync_state 
------------------+-----------+------------
 walreceiver      | streaming | async
```

```bash
# Check data on standby
docker exec standby psql -U postgres -c \
    "SELECT * FROM test_data;"
```

You should see the same 3 rows!

### Step 6: Test Live Replication

```bash
# Insert on primary
docker exec primary psql -U postgres -c \
    "INSERT INTO test_data (name) VALUES ('Dave'), ('Eve');"

# Check on primary
docker exec primary psql -U postgres -c \
    "SELECT * FROM test_data;"

# Wait 1 second
sleep 1

# Check on standby
docker exec standby psql -U postgres -c \
    "SELECT * FROM test_data;"
```

**Both should show 5 rows!** ✅

---

## Quick Tests

### Test 1: Check Replication is Working

```bash
# Terminal 1: Watch standby
watch -n 1 'docker exec standby psql -U postgres -t -c "SELECT COUNT(*) FROM test_data;"'

# Terminal 2: Insert on primary
for i in {1..10}; do
    docker exec primary psql -U postgres -c \
        "INSERT INTO test_data (name) VALUES ('User $i');"
    sleep 1
done
```

Watch the count increase on standby!

### Test 2: Standby is Read-Only

```bash
# Try to write on standby (should fail)
docker exec standby psql -U postgres -c \
    "INSERT INTO test_data (name) VALUES ('This will fail');"
```

Expected error:

```
ERROR:  cannot execute INSERT in a read-only transaction
```

### Test 3: Check Lag

```bash
docker exec primary psql -U postgres -c \
    "SELECT 
        application_name,
        pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS lag
    FROM pg_stat_replication;"
```

Should show very small lag (usually 0 bytes or a few KB).

---

## Troubleshooting

### Problem: Standby won't start

```bash
# Check logs
docker logs standby

# Common issue: Wrong password
# Solution: Make sure you typed 'reppass' correctly

# Or setup .pgpass to avoid password prompt
docker exec standby bash -c \
    "echo 'primary:5432:replication:repuser:reppass' > ~/.pgpass && chmod 600 ~/.pgpass"
```

### Problem: No replication connection

```bash
# Check primary logs
docker logs primary | grep replication

# Check network
docker exec standby ping primary

# Verify pg_hba.conf
docker exec primary cat /var/lib/postgresql/data/pg_hba.conf | grep replication
```

### Problem: Standby has old data

```bash
# Force WAL sync
docker exec primary psql -U postgres -c "SELECT pg_switch_wal();"

# Wait a moment
sleep 2

# Check again
docker exec standby psql -U postgres -c "SELECT COUNT(*) FROM test_data;"
```

---

## Simple Monitoring Script

**File: `monitor.sh`**

```bash
#!/bin/bash

echo "=== PostgreSQL Replication Status ==="
echo ""

echo "PRIMARY:"
docker exec primary psql -U postgres -c \
    "SELECT * FROM pg_stat_replication;" 2>/dev/null || echo "Primary not running"

echo ""
echo "STANDBY:"
docker exec standby psql -U postgres -c \
    "SELECT pg_is_in_recovery(), pg_last_wal_replay_lsn();" 2>/dev/null || echo "Standby not running"

echo ""
echo "DATA COUNT:"
echo -n "Primary: "
docker exec primary psql -U postgres -t -c "SELECT COUNT(*) FROM test_data;" 2>/dev/null || echo "N/A"
echo -n "Standby: "
docker exec standby psql -U postgres -t -c "SELECT COUNT(*) FROM test_data;" 2>/dev/null || echo "N/A"
```

```bash
chmod +x monitor.sh
./monitor.sh
```

---

## Fun Experiments

### Experiment 1: Bulk Insert

```bash
# Insert 10,000 rows on primary
docker exec primary psql -U postgres << 'EOF'
INSERT INTO test_data (name)
SELECT 'User ' || i
FROM generate_series(1, 10000) i;
EOF

# Immediately check standby
docker exec standby psql -U postgres -c "SELECT COUNT(*) FROM test_data;"
```

Should replicate almost instantly!

### Experiment 2: Stop Standby

```bash
# Stop standby
docker-compose stop standby

# Insert more data on primary
docker exec primary psql -U postgres -c \
    "INSERT INTO test_data (name) VALUES ('While standby was down');"

# Start standby
docker-compose start standby
sleep 5

# Check - new data should appear!
docker exec standby psql -U postgres -c \
    "SELECT * FROM test_data ORDER BY id DESC LIMIT 1;"
```

### Experiment 3: Promote Standby (Failover)

```bash
# Promote standby to primary
docker exec standby pg_ctl promote -D /var/lib/postgresql/data/pgdata

# Wait
sleep 3

# Now standby accepts writes!
docker exec standby psql -U postgres -c \
    "INSERT INTO test_data (name) VALUES ('I am the primary now!');"

docker exec standby psql -U postgres -c \
    "SELECT * FROM test_data ORDER BY id DESC LIMIT 1;"
```

---

## Cleanup

```bash
# Stop everything
docker-compose down

# Remove volumes
docker volume prune

# Remove containers
docker container prune
```

---

## Summary

**What you learned:**

1. ✅ Setup PostgreSQL streaming replication
2. ✅ Create replication user
3. ✅ Use `pg_basebackup` to clone database
4. ✅ Monitor replication status
5. ✅ Test read queries on standby
6. ✅ Understand replication lag
7. ✅ Perform failover (promotion)
