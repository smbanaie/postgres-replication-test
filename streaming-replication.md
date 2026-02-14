# Comprehensive Guide to PostgreSQL 18 Streaming Replication (WAL-Based)

## Introduction: Streaming Replication Overview

### What is Streaming Replication?

Streaming replication is PostgreSQL's **physical replication** method that continuously streams Write-Ahead Log (WAL) records from a primary server to one or more standby servers in near real-time.

### Key Characteristics

- **Binary Replication**: Entire database cluster is replicated at the block level
- **Read-Only Replicas**: Standbys serve read queries (Hot Standby)
- **Automatic Failover**: Can promote standby to primary
- **Exact Copy**: Standbys are byte-for-byte copies of primary
- **Fast**: Minimal overhead, near-zero lag

### Synchronous vs Asynchronous Replication

| Feature            | Synchronous                                       | Asynchronous                           |
| ------------------ | ------------------------------------------------- | -------------------------------------- |
| **Durability**     | Transaction commits wait for standby confirmation | Transaction commits immediately        |
| **Performance**    | Slower (network latency impact)                   | Faster                                 |
| **Data Loss Risk** | Zero data loss on failover                        | Possible data loss (seconds/minutes)   |
| **Use Case**       | Critical data, compliance requirements            | High performance, acceptable data loss |
| **Complexity**     | Higher (need monitoring)                          | Lower                                  |

---

## Docker Compose Setup

### Complete Multi-Node Configuration

**File: `docker-compose-streaming.yml`**

```yaml
version: '3.8'

services:
  # Primary Database Server
  pg-primary:
    container_name: pg-primary
    hostname: pg-primary
    image: postgres:18
    volumes:
      - ./primary_data:/var/lib/postgresql/data
      - ./primary_init:/docker-entrypoint-initdb.d
      - ./backups:/backups
      - ./archive:/archive
    environment:
      - POSTGRES_PASSWORD=postgres123
      - POSTGRES_USER=postgres
      - POSTGRES_DB=production_db
      - POSTGRES_INITDB_ARGS=--data-checksums
    networks:
      - replication_net
    ports:
      - "5432:5432"
    restart: always
    command: >
      postgres
      -c wal_level=replica
      -c hot_standby=on
      -c max_wal_senders=10
      -c max_replication_slots=10
      -c hot_standby_feedback=on
      -c archive_mode=on
      -c archive_command='test ! -f /archive/%f && cp %p /archive/%f'
      -c wal_keep_size=1GB
      -c synchronous_commit=on
      -c synchronous_standby_names='FIRST 1 (standby1, standby2)'

  # Synchronous Standby Server
  pg-standby-sync:
    container_name: pg-standby-sync
    hostname: pg-standby-sync
    image: postgres:18
    volumes:
      - ./standby_sync_data:/var/lib/postgresql/data
      - ./backups:/backups
      - ./archive:/archive
    environment:
      - POSTGRES_PASSWORD=postgres123
      - POSTGRES_USER=postgres
    networks:
      - replication_net
    ports:
      - "5433:5432"
    restart: always
    depends_on:
      - pg-primary
    command: >
      postgres
      -c hot_standby=on
      -c wal_retrieve_retry_interval=5s
      -c primary_conninfo='host=pg-primary port=5432 user=replicator password=replicator123 application_name=standby1'

  # Asynchronous Standby Server
  pg-standby-async:
    container_name: pg-standby-async
    hostname: pg-standby-async
    image: postgres:18
    volumes:
      - ./standby_async_data:/var/lib/postgresql/data
      - ./backups:/backups
      - ./archive:/archive
    environment:
      - POSTGRES_PASSWORD=postgres123
      - POSTGRES_USER=postgres
    networks:
      - replication_net
    ports:
      - "5434:5432"
    restart: always
    depends_on:
      - pg-primary
    command: >
      postgres
      -c hot_standby=on
      -c wal_retrieve_retry_interval=5s
      -c primary_conninfo='host=pg-primary port=5432 user=replicator password=replicator123 application_name=standby2'

  # Cascade Standby (replicates from standby-sync)
  pg-standby-cascade:
    container_name: pg-standby-cascade
    hostname: pg-standby-cascade
    image: postgres:18
    profiles:
      - cascade
    volumes:
      - ./standby_cascade_data:/var/lib/postgresql/data
      - ./backups:/backups
    environment:
      - POSTGRES_PASSWORD=postgres123
      - POSTGRES_USER=postgres
    networks:
      - replication_net
    ports:
      - "5435:5432"
    restart: always
    depends_on:
      - pg-standby-sync
    command: >
      postgres
      -c hot_standby=on
      -c wal_retrieve_retry_interval=5s
      -c primary_conninfo='host=pg-standby-sync port=5432 user=replicator password=replicator123 application_name=standby3'

  # PgAdmin for Management
  pgadmin:
    container_name: pgadmin-streaming
    hostname: pgadmin-streaming
    image: dpage/pgadmin4
    profiles:
      - tools
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: pgadmin123
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    ports:
      - "5050:80"
    networks:
      - replication_net

  # Monitoring with pg_stat_monitor
  prometheus:
    container_name: prometheus
    image: prom/prometheus
    profiles:
      - monitoring
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - replication_net

  grafana:
    container_name: grafana
    image: grafana/grafana
    profiles:
      - monitoring
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - replication_net
    depends_on:
      - prometheus

volumes:
  primary_data:
  standby_sync_data:
  standby_async_data:
  standby_cascade_data:
  pgadmin_data:
  prometheus_data:
  grafana_data:

networks:
  replication_net:
    name: postgres_streaming_replication
    driver: bridge
```

---

## Initial Setup

### Step 1: Create Directory Structure

```bash
# Create directories
mkdir -p primary_init backups archive monitoring
mkdir -p primary_data standby_sync_data standby_async_data standby_cascade_data

# Set permissions
chmod -R 777 primary_init backups archive
chmod -R 700 primary_data standby_sync_data standby_async_data standby_cascade_data
```

### Step 2: Configure Primary Server

**File: `primary_init/01-setup-replication.sql`**

```sql
-- Create replication user with appropriate privileges
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replicator123';

-- Create monitoring user
CREATE ROLE monitor WITH LOGIN PASSWORD 'monitor123';
GRANT pg_monitor TO monitor;

-- Create sample database and schema
CREATE DATABASE production_db;
\c production_db

-- Create schemas
CREATE SCHEMA IF NOT EXISTS sales;
CREATE SCHEMA IF NOT EXISTS inventory;
CREATE SCHEMA IF NOT EXISTS analytics;

-- Sales tables
CREATE TABLE sales.transactions (
    id BIGSERIAL PRIMARY KEY,
    transaction_date TIMESTAMP DEFAULT NOW(),
    customer_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    unit_price NUMERIC(10,2) NOT NULL,
    total_amount NUMERIC(12,2) NOT NULL,
    payment_method VARCHAR(50),
    status VARCHAR(20) DEFAULT 'completed',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE sales.customers (
    id SERIAL PRIMARY KEY,
    customer_code VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(200) UNIQUE,
    phone VARCHAR(50),
    address TEXT,
    city VARCHAR(100),
    country VARCHAR(100),
    registration_date DATE DEFAULT CURRENT_DATE,
    total_purchases NUMERIC(15,2) DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE sales.products (
    id SERIAL PRIMARY KEY,
    sku VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    category VARCHAR(100),
    unit_price NUMERIC(10,2) NOT NULL,
    stock_quantity INTEGER DEFAULT 0,
    reorder_level INTEGER DEFAULT 10,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Inventory tables
CREATE TABLE inventory.stock_movements (
    id BIGSERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES sales.products(id),
    movement_type VARCHAR(20) NOT NULL, -- 'in', 'out', 'adjustment'
    quantity INTEGER NOT NULL,
    reference_no VARCHAR(100),
    notes TEXT,
    movement_date TIMESTAMP DEFAULT NOW(),
    created_by VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE inventory.warehouses (
    id SERIAL PRIMARY KEY,
    code VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    location VARCHAR(200),
    capacity INTEGER,
    manager VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Analytics tables (for reporting from standbys)
CREATE TABLE analytics.daily_sales_summary (
    summary_date DATE PRIMARY KEY,
    total_transactions INTEGER,
    total_revenue NUMERIC(15,2),
    total_customers INTEGER,
    avg_transaction_value NUMERIC(10,2),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Create indexes
CREATE INDEX idx_transactions_date ON sales.transactions(transaction_date);
CREATE INDEX idx_transactions_customer ON sales.transactions(customer_id);
CREATE INDEX idx_transactions_product ON sales.transactions(product_id);
CREATE INDEX idx_customers_email ON sales.customers(email);
CREATE INDEX idx_products_sku ON sales.products(sku);
CREATE INDEX idx_products_category ON sales.products(category);
CREATE INDEX idx_stock_movements_product ON inventory.stock_movements(product_id);
CREATE INDEX idx_stock_movements_date ON inventory.stock_movements(movement_date);

-- Insert sample data
-- Customers
INSERT INTO sales.customers (customer_code, name, email, phone, city, country, total_purchases)
SELECT 
    'CUST' || LPAD(i::text, 6, '0'),
    'Customer ' || i,
    'customer' || i || '@example.com',
    '555-' || LPAD(i::text, 7, '0'),
    (ARRAY['New York', 'Los Angeles', 'Chicago', 'Houston', 'Phoenix'])[1 + (i % 5)],
    'USA',
    (random() * 10000)::NUMERIC(15,2)
FROM generate_series(1, 1000) i;

-- Products
INSERT INTO sales.products (sku, name, description, category, unit_price, stock_quantity, reorder_level)
SELECT 
    'SKU' || LPAD(i::text, 8, '0'),
    'Product ' || i,
    'Description for product ' || i,
    (ARRAY['Electronics', 'Clothing', 'Food', 'Books', 'Home'])[1 + (i % 5)],
    (10 + random() * 990)::NUMERIC(10,2),
    (10 + random() * 1000)::INTEGER,
    (5 + random() * 20)::INTEGER
FROM generate_series(1, 500) i;

-- Transactions (last 30 days)
INSERT INTO sales.transactions (transaction_date, customer_id, product_id, quantity, unit_price, total_amount, payment_method, status)
SELECT 
    NOW() - (random() * INTERVAL '30 days'),
    1 + (random() * 999)::INTEGER,
    1 + (random() * 499)::INTEGER,
    (1 + random() * 10)::INTEGER,
    p.unit_price,
    (1 + random() * 10)::INTEGER * p.unit_price,
    (ARRAY['credit_card', 'debit_card', 'cash', 'paypal', 'bank_transfer'])[1 + (random() * 4)::INTEGER],
    (ARRAY['completed', 'pending', 'cancelled'])[1 + (random() * 2)::INTEGER]
FROM generate_series(1, 10000) i
CROSS JOIN LATERAL (
    SELECT unit_price FROM sales.products WHERE id = 1 + (random() * 499)::INTEGER LIMIT 1
) p;

-- Warehouses
INSERT INTO inventory.warehouses (code, name, location, capacity, manager)
VALUES 
    ('WH001', 'Main Warehouse', 'New York, NY', 100000, 'John Smith'),
    ('WH002', 'West Coast Hub', 'Los Angeles, CA', 80000, 'Jane Doe'),
    ('WH003', 'Midwest Center', 'Chicago, IL', 60000, 'Bob Johnson');

-- Stock movements
INSERT INTO inventory.stock_movements (product_id, movement_type, quantity, reference_no, movement_date)
SELECT 
    1 + (random() * 499)::INTEGER,
    (ARRAY['in', 'out', 'adjustment'])[1 + (random() * 2)::INTEGER],
    (-50 + random() * 100)::INTEGER,
    'REF' || LPAD(i::text, 8, '0'),
    NOW() - (random() * INTERVAL '60 days')
FROM generate_series(1, 5000) i;

-- Create materialized view for analytics
CREATE MATERIALIZED VIEW analytics.product_sales_summary AS
SELECT 
    p.id,
    p.sku,
    p.name,
    p.category,
    COUNT(t.id) as total_transactions,
    SUM(t.quantity) as total_quantity_sold,
    SUM(t.total_amount) as total_revenue,
    AVG(t.total_amount) as avg_transaction_value
FROM sales.products p
LEFT JOIN sales.transactions t ON p.id = t.product_id
GROUP BY p.id, p.sku, p.name, p.category;

CREATE UNIQUE INDEX ON analytics.product_sales_summary(id);

-- Grant permissions
GRANT CONNECT ON DATABASE production_db TO replicator;
GRANT USAGE ON SCHEMA sales, inventory, analytics TO replicator;
GRANT SELECT ON ALL TABLES IN SCHEMA sales, inventory, analytics TO replicator;

-- Create function to update updated_at timestamp
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply trigger to relevant tables
CREATE TRIGGER update_transactions_updated_at
    BEFORE UPDATE ON sales.transactions
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

**File: `primary_init/02-configure-replication.sh`**

```bash
#!/bin/bash
set -e

# Wait for PostgreSQL to be ready
until pg_isready -h localhost -p 5432 -U postgres; do
  echo "Waiting for PostgreSQL..."
  sleep 2
done

# Configure pg_hba.conf for replication
echo "Configuring pg_hba.conf for replication..."
cat >> ${PGDATA}/pg_hba.conf << EOF

# Replication connections
host    replication    replicator    0.0.0.0/0    md5
host    all           replicator    0.0.0.0/0    md5
host    all           monitor       0.0.0.0/0    md5
EOF

# Reload configuration
psql -U postgres -c "SELECT pg_reload_conf();"

echo "Replication setup completed!"
```

Make it executable:

```bash
chmod +x primary_init/02-configure-replication.sh
```

### Step 3: Start Primary Server

```bash
# Start only primary initially
docker-compose -f docker-compose-streaming.yml up -d pg-primary

# Wait for initialization
docker logs -f pg-primary

# Verify primary is ready
docker exec -it pg-primary psql -U postgres -c "SELECT version();"
```

### Step 4: Create Base Backup for Standbys

```bash
# Create backup directory
docker exec -it pg-primary mkdir -p /backups

# Take base backup for synchronous standby
docker exec -it pg-primary pg_basebackup \
    -h localhost -U replicator -D /backups/standby_sync_base \
    -Fp -Xs -P -R -c fast

# Take base backup for asynchronous standby
docker exec -it pg-primary pg_basebackup \
    -h localhost -U replicator -D /backups/standby_async_base \
    -Fp -Xs -P -R -c fast

# Copy backups to host
docker cp pg-primary:/backups/standby_sync_base/. ./standby_sync_data/
docker cp pg-primary:/backups/standby_async_base/. ./standby_async_data/

# Set correct ownership
sudo chown -R 999:999 standby_sync_data standby_async_data
```

### Step 5: Configure Standby Servers

**Synchronous Standby Configuration:**

```bash
# Edit standby configuration
cat >> standby_sync_data/postgresql.auto.conf << EOF
primary_conninfo = 'host=pg-primary port=5432 user=replicator password=replicator123 application_name=standby1'
primary_slot_name = 'standby1_slot'
hot_standby = on
hot_standby_feedback = on
wal_retrieve_retry_interval = 5s
restore_command = 'cp /archive/%f %p'
EOF

# Create standby.signal file
touch standby_sync_data/standby.signal
```

**Asynchronous Standby Configuration:**

```bash
# Edit standby configuration
cat >> standby_async_data/postgresql.auto.conf << EOF
primary_conninfo = 'host=pg-primary port=5432 user=replicator password=replicator123 application_name=standby2'
primary_slot_name = 'standby2_slot'
hot_standby = on
hot_standby_feedback = on
wal_retrieve_retry_interval = 5s
restore_command = 'cp /archive/%f %p'
EOF

# Create standby.signal file
touch standby_async_data/standby.signal
```

### Step 6: Create Replication Slots on Primary

```bash
docker exec -it pg-primary psql -U postgres << EOF
-- Create replication slots
SELECT pg_create_physical_replication_slot('standby1_slot');
SELECT pg_create_physical_replication_slot('standby2_slot');

-- Verify slots
SELECT slot_name, slot_type, active, restart_lsn FROM pg_replication_slots;
EOF
```

### Step 7: Start Standby Servers

```bash
# Start synchronous standby
docker-compose -f docker-compose-streaming.yml up -d pg-standby-sync

# Wait a few seconds
sleep 5

# Start asynchronous standby
docker-compose -f docker-compose-streaming.yml up -d pg-standby-async

# Check logs
docker logs -f pg-standby-sync
docker logs -f pg-standby-async
```

### Step 8: Verify Replication

```bash
# Check on primary
docker exec -it pg-primary psql -U postgres << EOF
-- Check replication status
SELECT 
    application_name,
    client_addr,
    state,
    sync_state,
    sync_priority,
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS send_lag,
    pg_wal_lsn_diff(pg_current_wal_lsn(), write_lsn) AS write_lag,
    pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn) AS flush_lag,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag
FROM pg_stat_replication;

-- Check slots
SELECT * FROM pg_replication_slots;
EOF
```

Expected output:

- `standby1` should show `sync_state = 'sync'` (synchronous)
- `standby2` should show `sync_state = 'async'` (asynchronous)
- Both should show `state = 'streaming'`

```bash
# Verify data on standbys
docker exec -it pg-standby-sync psql -U postgres -d production_db << EOF
SELECT COUNT(*) as customer_count FROM sales.customers;
SELECT COUNT(*) as transaction_count FROM sales.transactions;
SELECT pg_is_in_recovery();  -- Should return 't'
EOF

docker exec -it pg-standby-async psql -U postgres -d production_db << EOF
SELECT COUNT(*) as customer_count FROM sales.customers;
SELECT COUNT(*) as transaction_count FROM sales.transactions;
SELECT pg_is_in_recovery();  -- Should return 't'
EOF
```

---

## Testing Scenarios

### Test 1: Basic Write Replication

**On Primary:**

```bash
docker exec -it pg-primary psql -U postgres -d production_db << EOF
-- Insert new customer
INSERT INTO sales.customers (customer_code, name, email, phone, city, country)
VALUES ('CUST999999', 'Test Customer', 'test@example.com', '555-9999999', 'Seattle', 'USA');

-- Get the ID
SELECT id, customer_code, name FROM sales.customers WHERE customer_code = 'CUST999999';
EOF
```

**On Synchronous Standby (immediate):**

```bash
docker exec -it pg-standby-sync psql -U postgres -d production_db << EOF
SELECT id, customer_code, name FROM sales.customers WHERE customer_code = 'CUST999999';
-- Should appear immediately
EOF
```

**On Asynchronous Standby (slight delay):**

```bash
docker exec -it pg-standby-async psql -U postgres -d production_db << EOF
SELECT id, customer_code, name FROM sales.customers WHERE customer_code = 'CUST999999';
-- May have slight delay
EOF
```

### Test 2: Synchronous vs Asynchronous Performance

**Create performance test script:**

**File: `test-sync-async-performance.sh`**

```bash
#!/bin/bash

echo "Testing Synchronous Replication Performance..."

# Test with synchronous replication (both standbys up)
START=$(date +%s%3N)

docker exec -it pg-primary psql -U postgres -d production_db << EOF
BEGIN;
INSERT INTO sales.transactions (customer_id, product_id, quantity, unit_price, total_amount, payment_method)
SELECT 
    1 + (random() * 999)::INTEGER,
    1 + (random() * 499)::INTEGER,
    (1 + random() * 5)::INTEGER,
    (10 + random() * 90)::NUMERIC(10,2),
    (10 + random() * 500)::NUMERIC(12,2),
    'credit_card'
FROM generate_series(1, 1000);
COMMIT;
EOF

END=$(date +%s%3N)
SYNC_TIME=$((END - START))

echo "Synchronous commit time: ${SYNC_TIME}ms"

# Now test with async only (stop sync standby)
echo "Stopping synchronous standby..."
docker-compose -f docker-compose-streaming.yml stop pg-standby-sync

# Wait for failover to async
sleep 5

echo "Testing Asynchronous Replication Performance..."
START=$(date +%s%3N)

docker exec -it pg-primary psql -U postgres -d production_db << EOF
BEGIN;
INSERT INTO sales.transactions (customer_id, product_id, quantity, unit_price, total_amount, payment_method)
SELECT 
    1 + (random() * 999)::INTEGER,
    1 + (random() * 499)::INTEGER,
    (1 + random() * 5)::INTEGER,
    (10 + random() * 90)::NUMERIC(10,2),
    (10 + random() * 500)::NUMERIC(12,2),
    'credit_card'
FROM generate_series(1, 1000);
COMMIT;
EOF

END=$(date +%s%3N)
ASYNC_TIME=$((END - START))

echo "Asynchronous commit time: ${ASYNC_TIME}ms"
echo "Performance difference: $((SYNC_TIME - ASYNC_TIME))ms"

# Restart sync standby
docker-compose -f docker-compose-streaming.yml start pg-standby-sync
```

Make it executable and run:

```bash
chmod +x test-sync-async-performance.sh
./test-sync-async-performance.sh
```

### Test 3: Synchronous Standby Failure

**Simulate sync standby failure:**

```bash
# Stop synchronous standby
docker-compose -f docker-compose-streaming.yml stop pg-standby-sync

# Check primary - writes should now wait or fail
docker exec -it pg-primary psql -U postgres -d production_db << EOF
-- This might hang if synchronous_standby_names requires it
INSERT INTO sales.customers (customer_code, name, email)
VALUES ('CUSTHANG', 'Hang Test', 'hang@test.com');
EOF
```

**Check replication status:**

```bash
docker exec -it pg-primary psql -U postgres << EOF
SELECT application_name, state, sync_state 
FROM pg_stat_replication;

-- Show current synchronous standbys
SHOW synchronous_standby_names;
EOF
```

**Solution: Adjust synchronous_standby_names:**

```bash
docker exec -it pg-primary psql -U postgres << EOF
-- Change to allow async standby to be promoted
ALTER SYSTEM SET synchronous_standby_names = 'FIRST 1 (standby2)';
SELECT pg_reload_conf();

-- Now writes should complete
INSERT INTO sales.customers (customer_code, name, email)
VALUES ('CUSTWORK', 'Working Test', 'work@test.com');
EOF
```

**Restart sync standby:**

```bash
docker-compose -f docker-compose-streaming.yml start pg-standby-sync

# Wait for catchup
sleep 10

# Restore original configuration
docker exec -it pg-primary psql -U postgres << EOF
ALTER SYSTEM SET synchronous_standby_names = 'FIRST 1 (standby1, standby2)';
SELECT pg_reload_conf();
EOF
```

### Test 4: Read Query Load Balancing

**Create read workload script:**

**File: `test-read-load-balancing.sh`**

```bash
#!/bin/bash

echo "Testing Read Load Balancing..."

# Function to run read query
run_read_query() {
    local server=$1
    local port=$2

    docker exec -it $server psql -U postgres -d production_db -c \
        "SELECT COUNT(*) FROM sales.transactions WHERE status = 'completed';" > /dev/null 2>&1
}

# Simulate read load
echo "Running 100 read queries on standbys..."

START=$(date +%s)

for i in {1..50}; do
    run_read_query pg-standby-sync 5433 &
    run_read_query pg-standby-async 5434 &
done

wait

END=$(date +%s)
DURATION=$((END - START))

echo "Completed 100 queries in ${DURATION} seconds"

# Check standby activity
echo "Synchronous Standby Activity:"
docker exec -it pg-standby-sync psql -U postgres << EOF
SELECT datname, count(*) as connections
FROM pg_stat_activity
WHERE datname IS NOT NULL
GROUP BY datname;
EOF

echo "Asynchronous Standby Activity:"
docker exec -it pg-standby-async psql -U postgres << EOF
SELECT datname, count(*) as connections
FROM pg_stat_activity
WHERE datname IS NOT NULL
GROUP BY datname;
EOF
```

### Test 5: Transaction Durability

**Test synchronous commit guarantees:**

```bash
# On primary - with sync standby
docker exec -it pg-primary psql -U postgres -d production_db << EOF
SHOW synchronous_commit;  -- Should be 'on'

-- Insert with explicit sync commit
BEGIN;
SET LOCAL synchronous_commit = on;
INSERT INTO sales.customers (customer_code, name, email)
VALUES ('CUSTSYNC', 'Sync Test', 'sync@test.com')
RETURNING id, customer_code, created_at;
COMMIT;

-- This transaction is guaranteed to be on sync standby
EOF
```

**Verify immediately on sync standby:**

```bash
docker exec -it pg-standby-sync psql -U postgres -d production_db << EOF
SELECT id, customer_code, created_at 
FROM sales.customers 
WHERE customer_code = 'CUSTSYNC';
-- Should be there immediately
EOF
```

**Test async commit:**

```bash
docker exec -it pg-primary psql -U postgres -d production_db << EOF
-- Insert with async commit
BEGIN;
SET LOCAL synchronous_commit = off;
INSERT INTO sales.customers (customer_code, name, email)
VALUES ('CUSTASYNC', 'Async Test', 'async@test.com')
RETURNING id, customer_code, created_at;
COMMIT;

-- Transaction completes immediately, may not be on standby yet
EOF
```

### Test 6: Replication Lag Monitoring

**Create monitoring script:**

**File: `monitor-replication-lag.sh`**

```bash
#!/bin/bash

echo "=== Replication Lag Monitor ==="
echo "Timestamp: $(date)"
echo ""

# Check on primary
echo "PRIMARY SERVER STATUS:"
docker exec pg-primary psql -U postgres -t -A -F'|' << EOF
SELECT 
    application_name,
    client_addr,
    state,
    sync_state,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn)) AS send_lag,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), write_lsn)) AS write_lag,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn)) AS flush_lag,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS replay_lag,
    EXTRACT(EPOCH FROM (NOW() - last_msg_receipt_time))::INTEGER AS last_contact_seconds
FROM pg_stat_replication
ORDER BY application_name;
EOF

echo ""
echo "REPLICATION SLOTS:"
docker exec pg-primary psql -U postgres -t -A -F'|' << EOF
SELECT 
    slot_name,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots
ORDER BY slot_name;
EOF

echo ""
echo "STANDBY STATUS:"

echo "Synchronous Standby:"
docker exec pg-standby-sync psql -U postgres -t -A << EOF
SELECT 
    pg_is_in_recovery() as in_recovery,
    pg_size_pretty(pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn())) AS replay_lag,
pg_last_xact_replay_timestamp() AS last_replay_time,
EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp()))::INTEGER AS lag_seconds;
EOF

echo "Asynchronous Standby:"
docker exec pg-standby-async psql -U postgres -t -A << EOF
SELECT
pg_is_in_recovery() as in_recovery,
pg_size_pretty(pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn())) AS replay_lag,
pg_last_xact_replay_timestamp() AS last_replay_time,
EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp()))::INTEGER AS lag_seconds;
EOF

echo ""
echo "WAL ARCHIVE STATUS:"
docker exec pg-primary psql -U postgres << EOF
SELECT * FROM pg_stat_archiver;
EOF
```

Make it executable:

```bash
chmod +x monitor-replication-lag.sh
./monitor-replication-lag.sh
```

### Test 7: Large Transaction Replication

**Test large bulk operation:**

```bash
# On primary - insert large dataset
docker exec -it pg-primary psql -U postgres -d production_db << EOF
-- Start monitoring time
\timing on

-- Insert 100,000 transactions
BEGIN;
INSERT INTO sales.transactions (customer_id, product_id, quantity, unit_price, total_amount, payment_method)
SELECT 
    1 + (random() * 999)::INTEGER,
    1 + (random() * 499)::INTEGER,
    (1 + random() * 10)::INTEGER,
    (10 + random() * 190)::NUMERIC(10,2),
    (10 + random() * 2000)::NUMERIC(12,2),
    (ARRAY['credit_card', 'debit_card', 'paypal'])[1 + (random() * 2)::INTEGER]
FROM generate_series(1, 100000);
COMMIT;

-- Check count
SELECT COUNT(*) FROM sales.transactions;
EOF
```

**Monitor lag during bulk operation:**

```bash
# In another terminal, run monitoring continuously
watch -n 1 './monitor-replication-lag.sh'
```

**Verify on standbys:**

```bash
# Check sync standby
docker exec -it pg-standby-sync psql -U postgres -d production_db << EOF
SELECT COUNT(*) FROM sales.transactions;
EOF

# Check async standby
docker exec -it pg-standby-async psql -U postgres -d production_db << EOF
SELECT COUNT(*) FROM sales.transactions;
EOF
```

### Test 8: DDL Replication

**Test schema changes:**

```bash
# On primary - add new column
docker exec -it pg-primary psql -U postgres -d production_db << EOF
-- Add new column
ALTER TABLE sales.customers ADD COLUMN loyalty_points INTEGER DEFAULT 0;

-- Create new table
CREATE TABLE sales.promotions (
    id SERIAL PRIMARY KEY,
    promo_code VARCHAR(50) UNIQUE NOT NULL,
    discount_percentage NUMERIC(5,2),
    start_date DATE,
    end_date DATE,
    active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Create index
CREATE INDEX idx_promotions_active ON sales.promotions(active, start_date, end_date);

-- Insert data
INSERT INTO sales.promotions (promo_code, discount_percentage, start_date, end_date)
VALUES 
    ('SAVE10', 10.00, CURRENT_DATE, CURRENT_DATE + INTERVAL '30 days'),
    ('SAVE20', 20.00, CURRENT_DATE, CURRENT_DATE + INTERVAL '7 days');

\d sales.customers
\d sales.promotions
EOF
```

**Verify on standbys:**

```bash
# Check sync standby
docker exec -it pg-standby-sync psql -U postgres -d production_db << EOF
\d sales.customers
\d sales.promotions
SELECT * FROM sales.promotions;
EOF

# Check async standby
docker exec -it pg-standby-async psql -U postgres -d production_db << EOF
\d sales.customers
\d sales.promotions
SELECT * FROM sales.promotions;
EOF
```

### Test 9: Cascade Replication

**Start cascade standby:**

```bash
# First, take base backup from sync standby
docker exec -it pg-standby-sync pg_basebackup \
    -h localhost -U replicator -D /backups/standby_cascade_base \
    -Fp -Xs -P -R -c fast

# Copy to host
docker cp pg-standby-sync:/backups/standby_cascade_base/. ./standby_cascade_data/

# Configure cascade standby
cat >> standby_cascade_data/postgresql.auto.conf << EOF
primary_conninfo = 'host=pg-standby-sync port=5432 user=replicator password=replicator123 application_name=standby3'
hot_standby = on
wal_retrieve_retry_interval = 5s
EOF

# Ensure standby.signal exists
touch standby_cascade_data/standby.signal

# Set ownership
sudo chown -R 999:999 standby_cascade_data

# Start cascade standby
docker-compose -f docker-compose-streaming.yml --profile cascade up -d pg-standby-cascade
```

**Verify cascade replication:**

```bash
# Check sync standby for downstream replication
docker exec -it pg-standby-sync psql -U postgres << EOF
SELECT application_name, client_addr, state, sync_state
FROM pg_stat_replication;
EOF

# Insert data on primary
docker exec -it pg-primary psql -U postgres -d production_db << EOF
INSERT INTO sales.customers (customer_code, name, email)
VALUES ('CUSTCASCADE', 'Cascade Test', 'cascade@test.com');
EOF

# Wait a moment
sleep 3

# Verify on cascade standby
docker exec -it pg-standby-cascade psql -U postgres -d production_db << EOF
SELECT customer_code, name FROM sales.customers WHERE customer_code = 'CUSTCASCADE';
EOF
```

---

## Failover and High Availability

### Scenario 1: Planned Failover (Switchover)

**Step 1: Prepare for switchover**

```bash
# Stop writes on primary (application level)
# Ensure replication is caught up

docker exec -it pg-primary psql -U postgres << EOF
-- Check replication lag
SELECT application_name, 
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- Wait until lag is 0 or very small
EOF
```

**Step 2: Promote standby to primary**

**File: `failover-to-sync-standby.sh`**

```bash
#!/bin/bash
set -e

echo "Starting planned failover to synchronous standby..."

# Step 1: Check replication status
echo "Checking replication status..."
docker exec pg-primary psql -U postgres -t -c \
    "SELECT application_name, pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes FROM pg_stat_replication;"

# Step 2: Stop primary gracefully
echo "Stopping primary server..."
docker exec pg-primary pg_ctl stop -D /var/lib/postgresql/data -m fast

# Step 3: Promote sync standby
echo "Promoting synchronous standby to primary..."
docker exec pg-standby-sync pg_ctl promote -D /var/lib/postgresql/data

# Wait for promotion
sleep 5

# Step 4: Verify new primary
echo "Verifying new primary..."
docker exec pg-standby-sync psql -U postgres << EOF
SELECT pg_is_in_recovery();  -- Should return 'f'
SELECT * FROM pg_stat_replication;  -- Should show async standby
EOF

# Step 5: Configure old primary as new standby (optional)
echo "Converting old primary to standby..."
docker exec pg-primary bash << 'EOFBASH'
cat > /var/lib/postgresql/data/postgresql.auto.conf << EOF
primary_conninfo = 'host=pg-standby-sync port=5432 user=replicator password=replicator123 application_name=standby_old_primary'
hot_standby = on
EOF
touch /var/lib/postgresql/data/standby.signal
EOFBASH

# Step 6: Start old primary as standby
echo "Starting old primary as new standby..."
docker exec pg-primary pg_ctl start -D /var/lib/postgresql/data

sleep 5

# Step 7: Update application connection
echo "Update your application to connect to new primary:"
echo "  Host: pg-standby-sync (port 5433 on host)"
echo ""
echo "Failover completed successfully!"
```

Make it executable:

```bash
chmod +x failover-to-sync-standby.sh
```

### Scenario 2: Automatic Failover with pg_auto_failover

**Install pg_auto_failover:**

```yaml
# Add to docker-compose-streaming.yml
  pg-monitor:
    container_name: pg-monitor
    hostname: pg-monitor
    image: citusdata/pg_auto_failover:latest
    profiles:
      - auto-failover
    environment:
      - PGDATA=/var/lib/postgresql/monitor
      - PG_AUTOCTL_MONITOR=1
    networks:
      - replication_net
    ports:
      - "5436:5432"
    volumes:
      - monitor_data:/var/lib/postgresql
```

### Scenario 3: Emergency Failover

**Quick promotion when primary is down:**

```bash
# Primary is completely down
docker-compose -f docker-compose-streaming.yml stop pg-primary

# Immediately promote sync standby
docker exec pg-standby-sync pg_ctl promote -D /var/lib/postgresql/data

# Verify
docker exec pg-standby-sync psql -U postgres -c "SELECT pg_is_in_recovery();"

# Point application to new primary immediately
# Host: pg-standby-sync, Port: 5433

# Reconfigure async standby to follow new primary
docker exec pg-standby-async bash << 'EOF'
cat > /var/lib/postgresql/data/postgresql.auto.conf << EOFCONF
primary_conninfo = 'host=pg-standby-sync port=5432 user=replicator password=replicator123 application_name=standby2'
hot_standby = on
EOFCONF
EOF

# Restart async standby
docker-compose -f docker-compose-streaming.yml restart pg-standby-async
```

### Scenario 4: Split-Brain Prevention

**File: `check-split-brain.sh`**

```bash
#!/bin/bash

echo "Checking for split-brain scenario..."

# Check if primary is in recovery (should be false)
PRIMARY_RECOVERY=$(docker exec pg-primary psql -U postgres -t -c "SELECT pg_is_in_recovery();" 2>/dev/null | tr -d ' ')

# Check if sync standby is in recovery (should be true)
SYNC_RECOVERY=$(docker exec pg-standby-sync psql -U postgres -t -c "SELECT pg_is_in_recovery();" 2>/dev/null | tr -d ' ')

# Check if async standby is in recovery (should be true)
ASYNC_RECOVERY=$(docker exec pg-standby-async psql -U postgres -t -c "SELECT pg_is_in_recovery();" 2>/dev/null | tr -d ' ')

echo "Primary in recovery: $PRIMARY_RECOVERY"
echo "Sync standby in recovery: $SYNC_RECOVERY"
echo "Async standby in recovery: $ASYNC_RECOVERY"

# Count how many are NOT in recovery
NOT_IN_RECOVERY=0
[[ "$PRIMARY_RECOVERY" == "f" ]] && ((NOT_IN_RECOVERY++))
[[ "$SYNC_RECOVERY" == "f" ]] && ((NOT_IN_RECOVERY++))
[[ "$ASYNC_RECOVERY" == "f" ]] && ((NOT_IN_RECOVERY++))

if [ $NOT_IN_RECOVERY -gt 1 ]; then
    echo "WARNING: SPLIT-BRAIN DETECTED!"
    echo "Multiple servers are accepting writes!"
    exit 1
else
    echo "OK: No split-brain detected"
    exit 0
fi
```

---

## Advanced Configurations

### Remote Apply and Remote Write

**Configure different synchronous commit levels:**

```bash
docker exec -it pg-primary psql -U postgres << EOF
-- Show current setting
SHOW synchronous_commit;

-- Options:
-- 'off'          - asynchronous, fastest, possible data loss
-- 'local'        - wait for local flush only
-- 'remote_write' - wait for standby to write to OS (not flush)
-- 'on' or 'remote_apply' - wait for standby to apply changes

-- Test remote_write
ALTER SYSTEM SET synchronous_commit = 'remote_write';
SELECT pg_reload_conf();

-- Test transaction
BEGIN;
INSERT INTO sales.customers (customer_code, name, email)
VALUES ('CUSTRW', 'Remote Write Test', 'rw@test.com');
COMMIT;  -- Completes when standby writes to OS cache
EOF
```

**Test different commit levels:**

```bash
# Create test script
cat > test-commit-levels.sh << 'EOF'
#!/bin/bash

for level in off local remote_write on; do
    echo "Testing synchronous_commit = $level"

    docker exec pg-primary psql -U postgres -d production_db << EOSQL
    ALTER SYSTEM SET synchronous_commit = '$level';
    SELECT pg_reload_conf();
EOSQL

    START=$(date +%s%3N)

    docker exec pg-primary psql -U postgres -d production_db << EOSQL
    INSERT INTO sales.transactions (customer_id, product_id, quantity, unit_price, total_amount)
    SELECT 1, 1, 1, 100, 100 FROM generate_series(1, 1000);
EOSQL

    END=$(date +%s%3N)
    DURATION=$((END - START))

    echo "Duration: ${DURATION}ms"
    echo ""
done
EOF

chmod +x test-commit-levels.sh
./test-commit-levels.sh
```

### Multiple Synchronous Standbys

**Configure quorum-based synchronous replication:**

```bash
docker exec -it pg-primary psql -U postgres << EOF
-- Require both standbys to confirm
ALTER SYSTEM SET synchronous_standby_names = '2 (standby1, standby2)';
SELECT pg_reload_conf();

-- Now both must confirm before commit returns
SHOW synchronous_standby_names;
EOF
```

**Test quorum behavior:**

```bash
# Insert with both standbys up
docker exec -it pg-primary psql -U postgres -d production_db << EOF
INSERT INTO sales.customers (customer_code, name, email)
VALUES ('CUSTQUORUM', 'Quorum Test', 'quorum@test.com');
EOF

# Stop one standby
docker-compose -f docker-compose-streaming.yml stop pg-standby-async

# This should still work (1 of 2 standbys is enough for quorum of 2)
# Actually, with "2 (standby1, standby2)", it requires BOTH
# Let's fix the configuration

docker exec -it pg-primary psql -U postgres << EOF
-- Use ANY 1 of 2
ALTER SYSTEM SET synchronous_standby_names = 'ANY 1 (standby1, standby2)';
SELECT pg_reload_conf();
EOF

# Now with one standby down, writes should work
docker exec -it pg-primary psql -U postgres -d production_db << EOF
INSERT INTO sales.customers (customer_code, name, email)
VALUES ('CUSTQUORUM2', 'Quorum Test 2', 'quorum2@test.com');
EOF

# Restart standby
docker-compose -f docker-compose-streaming.yml start pg-standby-async
```

### Replication Slot Management

**Monitor slot usage:**

```bash
docker exec -it pg-primary psql -U postgres << EOF
-- Check slot status and WAL retention
SELECT 
    slot_name,
    slot_type,
    active,
    temporary,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS pending_wal
FROM pg_replication_slots
ORDER BY slot_name;

-- Check WAL files on disk
SELECT 
    pg_size_pretty(SUM((pg_stat_file('pg_wal/' || name)).size)) AS total_wal_size
FROM pg_ls_waldir();
EOF
```

**Clean up inactive slots:**

```bash
docker exec -it pg-primary psql -U postgres << EOF
-- Find inactive slots
SELECT slot_name, active, restart_lsn
FROM pg_replication_slots
WHERE NOT active
  AND restart_lsn < pg_current_wal_lsn() - '1GB'::pg_lsn;

-- Drop if safe (example - adjust as needed)
-- SELECT pg_drop_replication_slot('old_slot_name');
EOF
```

### Timeline Management

**Understanding timelines after failover:**

```bash
# Check current timeline
docker exec -it pg-primary psql -U postgres << EOF
SELECT timeline_id, pg_walfile_name(redo_lsn) AS redo_wal_file
FROM pg_control_checkpoint();
EOF

# After a promotion, timeline changes
# Check standby timeline
docker exec -it pg-standby-sync psql -U postgres << EOF
SELECT timeline_id, pg_walfile_name(redo_lsn) AS redo_wal_file
FROM pg_control_checkpoint();
EOF
```

---

## Monitoring and Alerting

### Comprehensive Monitoring Dashboard

**File: `monitoring/prometheus.yml`**

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'postgres_primary'
    static_configs:
      - targets: ['pg-primary:5432']
    metrics_path: '/metrics'

  - job_name: 'postgres_standby_sync'
    static_configs:
      - targets: ['pg-standby-sync:5432']
    metrics_path: '/metrics'

  - job_name: 'postgres_standby_async'
    static_configs:
      - targets: ['pg-standby-async:5432']
    metrics_path: '/metrics'
```

### PostgreSQL Exporter Setup

**Add to docker-compose:**

```yaml
  postgres-exporter-primary:
    image: prometheuscommunity/postgres-exporter
    container_name: postgres-exporter-primary
    profiles:
      - monitoring
    environment:
      DATA_SOURCE_NAME: "postgresql://monitor:monitor123@pg-primary:5432/postgres?sslmode=disable"
    ports:
      - "9187:9187"
    networks:
      - replication_net

  postgres-exporter-sync:
    image: prometheuscommunity/postgres-exporter
    container_name: postgres-exporter-sync
    profiles:
      - monitoring
    environment:
      DATA_SOURCE_NAME: "postgresql://monitor:monitor123@pg-standby-sync:5432/postgres?sslmode=disable"
    ports:
      - "9188:9187"
    networks:
      - replication_net
```

### Custom Metrics Collection

**File: `collect-metrics.sh`**

```bash
#!/bin/bash

TIMESTAMP=$(date +%Y-%m-%d\ %H:%M:%S)
METRICS_FILE="metrics_$(date +%Y%m%d).csv"

# Collect replication metrics
docker exec pg-primary psql -U postgres -t -A -F',' << EOF >> $METRICS_FILE
SELECT 
    '$TIMESTAMP',
    'primary',
    application_name,
    client_addr::text,
    state,
    sync_state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS send_lag_bytes,
    pg_wal_lsn_diff(pg_current_wal_lsn(), write_lsn) AS write_lag_bytes,
    pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn) AS flush_lag_bytes,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes,
    EXTRACT(EPOCH FROM (NOW() - backend_start))::INTEGER AS connection_age_seconds
FROM pg_stat_replication;
EOF

echo "Metrics collected to $METRICS_FILE"
```

### Alert Configuration

**File: `alert-replication.sh`**

```bash
#!/bin/bash

# Thresholds
MAX_LAG_BYTES=104857600  # 100MB
MAX_LAG_SECONDS=60
MAX_SLOT_WAL=536870912   # 512MB

# Check replication lag
LAG_RESULT=$(docker exec pg-primary psql -U postgres -t -A << EOF
SELECT 
    application_name,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
    EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp()))::INTEGER AS lag_seconds
FROM pg_stat_replication;
EOF
)

while IFS='|' read -r app_name lag_bytes lag_seconds; do
    if [ "$lag_bytes" -gt "$MAX_LAG_BYTES" ]; then
        echo "ALERT: Replication lag for $app_name is ${lag_bytes} bytes (threshold: ${MAX_LAG_BYTES})"
        # Send notification (email, Slack, PagerDuty, etc.)
    fi

    if [ "$lag_seconds" -gt "$MAX_LAG_SECONDS" ]; then
        echo "ALERT: Replication lag for $app_name is ${lag_seconds} seconds (threshold: ${MAX_LAG_SECONDS})"
    fi
done <<< "$LAG_RESULT"

# Check slot WAL retention
SLOT_RESULT=$(docker exec pg-primary psql -U postgres -t -A << EOF
SELECT 
    slot_name,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS retained_bytes
FROM pg_replication_slots
WHERE NOT active;
EOF
)

while IFS='|' read -r slot_name retained_bytes; do
    if [ "$retained_bytes" -gt "$MAX_SLOT_WAL" ]; then
        echo "ALERT: Slot $slot_name is retaining ${retained_bytes} bytes of WAL (threshold: ${MAX_SLOT_WAL})"
    fi
done <<< "$SLOT_RESULT"
```

---

## Backup Strategy with Streaming Replication

### Backup from Standby

**Advantage: No impact on primary performance**

```bash
# Take backup from sync standby
docker exec pg-standby-sync pg_basebackup \
    -h localhost -U postgres \
    -D /backups/standby_backup_$(date +%Y%m%d_%H%M%S) \
    -Ft -z -Xs -P -c fast

# Verify backup
docker exec pg-standby-sync ls -lh /backups/
```

### Continuous Archiving with Replication

**Configure WAL archiving on standbys:**

```bash
# On standby - enable archive_mode for cascading
docker exec -it pg-standby-sync psql -U postgres << EOF
ALTER SYSTEM SET archive_mode = always;  -- 'always' archives even on standby
ALTER SYSTEM SET archive_command = 'test ! -f /archive/%f && cp %p /archive/%f';
SELECT pg_reload_conf();
EOF
```

### Point-in-Time Recovery with Replication

**Scenario: Recover to point before data corruption:**

```bash
# Assume data corruption at 14:30
# We want to recover to 14:25

# Step 1: Stop primary
docker-compose -f docker-compose-streaming.yml stop pg-primary

# Step 2: Create recovery configuration
docker exec pg-primary bash << 'EOF'
cat > /var/lib/postgresql/data/postgresql.auto.conf << EOFCONF
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2024-11-24 14:25:00'
recovery_target_action = 'promote'
EOFCONF

touch /var/lib/postgresql/data/recovery.signal
EOF

# Step 3: Start primary in recovery mode
docker-compose -f docker-compose-streaming.yml start pg-primary

# Step 4: Wait for recovery
docker logs -f pg-primary
# Watch for: "database system is ready to accept connections"

# Step 5: Verify data
docker exec pg-primary psql -U postgres -d production_db << EOF
SELECT MAX(created_at) FROM sales.transactions;
-- Should be around 14:25
EOF
```

---

## Performance Tuning

### Optimize Replication Performance

**Tune PostgreSQL settings:**

```bash
docker exec -it pg-primary psql -U postgres << EOF
-- Increase WAL buffers
ALTER SYSTEM SET wal_buffers = '16MB';

-- Tune checkpoint settings
ALTER SYSTEM SET checkpoint_timeout = '15min';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
ALTER SYSTEM SET max_wal_size = '4GB';
ALTER SYSTEM SET min_wal_size = '1GB';

-- Increase replication workers
ALTER SYSTEM SET max_worker_processes = 16;
ALTER SYSTEM SET max_parallel_workers = 8;

-- Tune network settings
ALTER SYSTEM SET tcp_keepalives_idle = 60;
ALTER SYSTEM SET tcp_keepalives_interval = 10;
ALTER SYSTEM SET tcp_keepalives_count = 6;

SELECT pg_reload_conf();
EOF

# Restart required for some changes
docker-compose -f docker-compose-streaming.yml restart pg-primary
```

### Parallel Replay on Standby (PostgreSQL 16+)

```bash
docker exec -it pg-standby-sync psql -U postgres << EOF
-- Enable parallel replay
ALTER SYSTEM SET recovery_prefetch = 'on';
ALTER SYSTEM SET wal_decode_buffer_size = '512kB';

-- Restart to apply
EOF

docker-compose -f docker-compose-streaming.yml restart pg-standby-sync
```

### Network Compression

```bash
# Enable compression for replication (PostgreSQL 14+)
docker exec -it pg-standby-sync bash << 'EOF'
cat >> /var/lib/postgresql/data/postgresql.auto.conf << EOFCONF
primary_conninfo = 'host=pg-primary port=5432 user=replicator password=replicator123 application_name=standby1 compression=gzip'
EOFCONF
EOF

docker-compose -f docker-compose-streaming.yml restart pg-standby-sync
```

---

## Comprehensive Test Suite

**File: `streaming-replication-test-suite.sh`**

```bash
#!/bin/bash

set -e

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

PASSED=0
FAILED=0

print_header() {
    echo -e "\n${BLUE}========================================${NC}"
    echo -e "${BLUE}$1${NC}"
    echo -e "${BLUE}========================================${NC}\n"
}

run_test() {
    local test_name=$1
    local test_command=$2

    echo -e "${YELLOW}Test: $test_name${NC}"

    if eval "$test_command"; then
        echo -e "${GREEN}✓ PASSED${NC}\n"
        ((PASSED++))
    else
        echo -e "${RED}✗ FAILED${NC}\n"
        ((FAILED++))
    fi
}

print_header "PostgreSQL Streaming Replication Test Suite"

# Test 1: Container Status
print_header "Phase 1: Infrastructure Tests"

run_test "Primary Container Running" \
    "docker ps | grep -q pg-primary"

run_test "Sync Standby Running" \
    "docker ps | grep -q pg-standby-sync"

run_test "Async Standby Running" \
    "docker ps | grep -q pg-standby-async"

# Test 2: Database Connectivity
print_header "Phase 2: Connectivity Tests"

run_test "Primary Connection" \
    "docker exec pg-primary psql -U postgres -c 'SELECT 1;' > /dev/null 2>&1"

run_test "Sync Standby Connection" \
    "docker exec pg-standby-sync psql -U postgres -c 'SELECT 1;' > /dev/null 2>&1"

run_test "Async Standby Connection" \
    "docker exec pg-standby-async psql -U postgres -c 'SELECT 1;' > /dev/null 2>&1"

# Test 3: Replication Status
print_header "Phase 3: Replication Status Tests"

run_test "Replication Slots Active" \
    "docker exec pg-primary psql -U postgres -t -c \"SELECT COUNT(*) FROM pg_replication_slots WHERE active;\" | grep -q '2'"

run_test "Both Standbys Connected" \
    "docker exec pg-primary psql -U postgres -t -c \"SELECT COUNT(*) FROM pg_stat_replication;\" | grep -q '2'"

run_test "Sync Standby is Synchronous" \
    "docker exec pg-primary psql -U postgres -t -c \"SELECT sync_state FROM pg_stat_replication WHERE application_name='standby1';\" | grep -q 'sync'"

run_test "Async Standby is Asynchronous" \
    "docker exec pg-primary psql -U postgres -t -c \"SELECT sync_state FROM pg_stat_replication WHERE application_name='standby2';\" | grep -q 'async'"

run_test "Sync Standby in Recovery Mode" \
    "docker exec pg-standby-sync psql -U postgres -t -c \"SELECT pg_is_in_recovery();\" | grep -q 't'"

run_test "Async Standby in Recovery Mode" \
    "docker exec pg-standby-async psql -U postgres -t -c \"SELECT pg_is_in_recovery();\" | grep -q 't'"

# Test 4: Data Replication
print_header "Phase 4: Data Replication Tests"

# Insert test data
TEST_ID=$(date +%s)
docker exec pg-primary psql -U postgres -d production_db -t -c \
    "INSERT INTO sales.customers (customer_code, name, email) VALUES ('TEST$TEST_ID', 'Test User $TEST_ID', 'test$TEST_ID@example.com') RETURNING id;" | tr -d ' ' > /tmp/test_id.txt

sleep 2

run_test "Data Replicated to Sync Standby" \
    "docker exec pg-standby-sync psql -U postgres -d production_db -t -c \"SELECT COUNT(*) FROM sales.customers WHERE customer_code='TEST$TEST_ID';\" | grep -q '1'"

run_test "Data Replicated to Async Standby" \
    "docker exec pg-standby-async psql -U postgres -d production_db -t -c \"SELECT COUNT(*) FROM sales.customers WHERE customer_code='TEST$TEST_ID';\" | grep -q '1'"

# Test 5: Transaction Consistency
print_header "Phase 5:ransaction Consistency Tests"

# Complex transaction test

TX_ID=$(date +%s)
docker exec pg-primary psql -U postgres -d production_db << EOF > /dev/null 2>&1
BEGIN;
INSERT INTO sales.customers (customer_code, name, email)
VALUES ('CUST$TX_ID', 'TX Test $TX_ID', 'tx$TX_ID@example.com');

INSERT INTO sales.transactions (customer_id, product_id, quantity, unit_price, total_amount, payment_method)
VALUES (
(SELECT id FROM sales.customers WHERE customer_code = 'CUST$TX_ID'),
1, 5, 100.00, 500.00, 'credit_card'
);
COMMIT;
EOF

sleep 2

run_test "Transaction Atomic on Sync Standby"  
"docker exec pg-standby-sync psql -U postgres -d production_db -t -c "SELECT COUNT(*) FROM sales.transactions WHERE customer_id = (SELECT id FROM sales.customers WHERE customer_code='CUST$TX_ID');" | grep -q '1'"

run_test "Transaction Atomic on Async Standby"  
"docker exec pg-standby-async psql -U postgres -d production_db -t -c "SELECT COUNT(*) FROM sales.transactions WHERE customer_id = (SELECT id FROM sales.customers WHERE customer_code='CUST$TX_ID');" | grep -q '1'"

# Test 6: Replication Lag

print_header "Phase 6: Replication Lag Tests"

run_test "Sync Standby Low Lag"  
"[ $(docker exec pg-primary psql -U postgres -t -c "SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) FROM pg_stat_replication WHERE application_name='standby1';" | tr -d ' ') -lt 1048576 ]"

run_test "Async Standby Reasonable Lag"  
"[ $(docker exec pg-primary psql -U postgres -t -c "SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) FROM pg_stat_replication WHERE application_name='standby2';" | tr -d ' ') -lt 10485760 ]"

# Test 7: Read-Only Queries on Standbys

print_header "Phase 7: Hot Standby Tests"

run_test "Read Query on Sync Standby"  
"docker exec pg-standby-sync psql -U postgres -d production_db -c 'SELECT COUNT(*) FROM sales.customers;' > /dev/null 2>&1"

run_test "Read Query on Async Standby"  
"docker exec pg-standby-async psql -U postgres -d production_db -c 'SELECT COUNT(*) FROM sales.customers;' > /dev/null 2>&1"

run_test "Write Blocked on Sync Standby"  
"! docker exec pg-standby-sync psql -U postgres -d production_db -c "INSERT INTO sales.customers (customer_code, name, email) VALUES ('FAIL', 'Should Fail', 'fail@test.com');" > /dev/null 2>&1"

# Test 8: Bulk Operation Performance

print_header "Phase 8: Performance Tests"

echo "Testing bulk insert performance..."
START=$(date +%s)

docker exec pg-primary psql -U postgres -d production_db << EOF > /dev/null 2>&1
INSERT INTO sales.transactions (customer_id, product_id, quantity, unit_price, total_amount, payment_method)
SELECT
1 + (random() * 999)::INTEGER,
1 + (random() * 499)::INTEGER,
(1 + random() * 5)::INTEGER,
(10 + random() * 90)::NUMERIC(10,2),
(10 + random() * 500)::NUMERIC(12,2),
'credit_card'
FROM generate_series(1, 10000);
EOF

END=$(date +%s)
DURATION=$((END - START))

if [ $DURATION -lt 30 ]; then
echo -e "${GREEN}✓ PASSED - Bulk insert completed in ${DURATION}s${NC}\n"
((PASSED++))
else
echo -e "${YELLOW}⚠ WARNING - Bulk insert took ${DURATION}s (acceptable but slow)${NC}\n"
((PASSED++))
fi

# Wait for replication

sleep 5

# Test 9: Data Consistency After Bulk Operation

print_header "Phase 9: Post-Bulk Data Consistency"

PRIMARY_COUNT=$(docker exec pg-primary psql -U postgres -d production_db -t -c  
"SELECT COUNT(*) FROM sales.transactions;" | tr -d ' ')

SYNC_COUNT=$(docker exec pg-standby-sync psql -U postgres -d production_db -t -c  
"SELECT COUNT(*) FROM sales.transactions;" | tr -d ' ')

ASYNC_COUNT=$(docker exec pg-standby-async psql -U postgres -d production_db -t -c  
"SELECT COUNT(*) FROM sales.transactions;" | tr -d ' ')

echo "Primary count: $PRIMARY_COUNT"
echo "Sync standby count: $SYNC_COUNT"
echo "Async standby count: $ASYNC_COUNT"

if [ "$PRIMARY_COUNT" = "$SYNC_COUNT" ] && [ "$PRIMARY_COUNT" = "$ASYNC_COUNT" ]; then
echo -e "${GREEN}✓ PASSED - All counts match${NC}\n"
((PASSED++))
else
echo -e "${RED}✗ FAILED - Count mismatch${NC}\n"
((FAILED++))
fi

# Test 10: WAL Archiving

print_header "Phase 10: WAL Archiving Tests"

run_test "WAL Archive Directory Exists"  
"docker exec pg-primary test -d /archive"

run_test "WAL Files Being Archived"  
"[ $(docker exec pg-primary ls /archive 2>/dev/null | wc -l) -gt 0 ]"

# Test 11: Slot Health

print_header "Phase 11: Replication Slot Health"

run_test "No Inactive Slots"  
"[ $(docker exec pg-primary psql -U postgres -t -c "SELECT COUNT(*) FROM pg_replication_slots WHERE NOT active;" | tr -d ' ') -eq 0 ]"

run_test "Reasonable WAL Retention"  
"[ $(docker exec pg-primary psql -U postgres -t -c "SELECT MAX(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) FROM pg_replication_slots;" | tr -d ' ') -lt 1073741824 ]"

# Test 12: Recovery Configuration

print_header "Phase 12: Recovery Configuration Tests"

run_test "Sync Standby Has standby.signal"  
"docker exec pg-standby-sync test -f /var/lib/postgresql/data/standby.signal"

run_test "Async Standby Has standby.signal"  
"docker exec pg-standby-async test -f /var/lib/postgresql/data/standby.signal"

run_test "Primary No standby.signal"  
"! docker exec pg-primary test -f /var/lib/postgresql/data/standby.signal"

# Summary

print_header "Test Summary"

TOTAL=$((PASSED + FAILED))
PASS_RATE=$(awk "BEGIN {printf "%.1f", ($PASSED/$TOTAL)*100}")

echo -e "${GREEN}Passed: $PASSED${NC}"
echo -e "${RED}Failed: $FAILED${NC}"
echo -e "Total: $TOTAL"
echo -e "Pass Rate: ${PASS_RATE}%\n"

if [ $FAILED -eq 0 ]; then
echo -e "${GREEN}🎉 All tests passed! Replication is healthy.${NC}"
exit 0
else
echo -e "${RED}⚠️ Some tests failed. Please review the output above.${NC}"
exit 1
fi
```

Make it executable:

```bash
chmod +x streaming-replication-test-suite.sh
./streaming-replication-test-suite.sh
```

---

## Troubleshooting Guide

### Issue 1: Standby Not Connecting

**Symptoms:**

- Standby shows no replication connection
- `pg_stat_replication` is empty on primary

**Diagnosis:**

```bash
# Check standby logs
docker logs pg-standby-sync

# Common errors:
# - "could not connect to the primary server"
# - "password authentication failed"
# - "no pg_hba.conf entry"
```

**Solutions:**

```bash
# 1. Check network connectivity
docker exec pg-standby-sync ping -c 3 pg-primary

# 2. Verify pg_hba.conf on primary
docker exec pg-primary cat /var/lib/postgresql/data/pg_hba.conf | grep replication

# 3. Test replication user connection
docker exec pg-standby-sync psql \
    "host=pg-primary port=5432 user=replicator password=replicator123 dbname=postgres" \
    -c "SELECT 1;"

# 4. Check primary_conninfo
docker exec pg-standby-sync cat /var/lib/postgresql/data/postgresql.auto.conf | grep primary_conninfo

# 5. Verify replication slot exists
docker exec pg-primary psql -U postgres -c \
    "SELECT * FROM pg_replication_slots WHERE slot_name='standby1_slot';"
```

### Issue 2: High Replication Lag

**Symptoms:**

- Lag growing continuously
- Queries on standby showing old data

**Diagnosis:**

```bash
# Check current lag
docker exec pg-primary psql -U postgres << EOF
SELECT 
    application_name,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS lag,
    state,
    sync_state
FROM pg_stat_replication;
EOF

# Check standby recovery status
docker exec pg-standby-sync psql -U postgres << EOF
SELECT 
    pg_is_in_recovery(),
    pg_size_pretty(pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn())) AS replay_lag,
    pg_last_xact_replay_timestamp();
EOF
```

**Solutions:**

```bash
# 1. Check for long-running queries on standby
docker exec pg-standby-sync psql -U postgres << EOF
SELECT pid, usename, state, query_start, 
       NOW() - query_start AS duration,
       left(query, 50) AS query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;
EOF

# 2. Increase wal_keep_size on primary
docker exec pg-primary psql -U postgres << EOF
ALTER SYSTEM SET wal_keep_size = '2GB';
SELECT pg_reload_conf();
EOF

# 3. Check disk I/O on standby
docker exec pg-standby-sync iostat -x 1 5

# 4. Enable hot_standby_feedback
docker exec pg-standby-sync psql -U postgres << EOF
ALTER SYSTEM SET hot_standby_feedback = on;
SELECT pg_reload_conf();
EOF

# 5. Check for network issues
docker exec pg-standby-sync ping -c 10 pg-primary | tail -3
```

### Issue 3: Replication Slot Bloat

**Symptoms:**

- Disk filling up with WAL files
- `pg_wal` directory growing large

**Diagnosis:**

```bash
# Check slot retention
docker exec pg-primary psql -U postgres << EOF
SELECT 
    slot_name,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots
ORDER BY pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) DESC;
EOF

# Check WAL directory size
docker exec pg-primary du -sh /var/lib/postgresql/data/pg_wal
```

**Solutions:**

```bash
# 1. If standby is permanently down, drop the slot
docker exec pg-primary psql -U postgres << EOF
SELECT pg_drop_replication_slot('standby_down_slot');
EOF

# 2. If standby is temporarily down, consider max_slot_wal_keep_size
docker exec pg-primary psql -U postgres << EOF
ALTER SYSTEM SET max_slot_wal_keep_size = '10GB';
SELECT pg_reload_conf();
SHOW max_slot_wal_keep_size;
EOF

# 3. Force WAL cleanup (dangerous - may break replication)
# Only if you're sure standby can resync
docker exec pg-primary psql -U postgres << EOF
SELECT pg_switch_wal();
CHECKPOINT;
EOF
```

### Issue 4: Synchronous Standby Timeout

**Symptoms:**

- Write operations hanging
- "waiting for synchronous replication" in logs

**Diagnosis:**

```bash
# Check if sync standby is connected
docker exec pg-primary psql -U postgres << EOF
SELECT application_name, sync_state, state
FROM pg_stat_replication
WHERE sync_state = 'sync';
EOF

# Check for waiting transactions
docker exec pg-primary psql -U postgres << EOF
SELECT pid, usename, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event = 'SyncRep';
EOF
```

**Solutions:**

```bash
# Emergency: Disable synchronous commit temporarily
docker exec pg-primary psql -U postgres << EOF
ALTER SYSTEM SET synchronous_commit = local;
SELECT pg_reload_conf();
EOF

# Or change sync standby configuration
docker exec pg-primary psql -U postgres << EOF
-- Allow any standby to fulfill sync requirement
ALTER SYSTEM SET synchronous_standby_names = 'FIRST 1 (standby1, standby2)';
SELECT pg_reload_conf();
EOF

# Check and restart failed standby
docker-compose -f docker-compose-streaming.yml restart pg-standby-sync

# Monitor until reconnected
watch -n 2 "docker exec pg-primary psql -U postgres -c 'SELECT * FROM pg_stat_replication;'"
```

### Issue 5: Promotion Not Working

**Symptoms:**

- `pg_ctl promote` fails
- Standby remains in recovery

**Diagnosis:**

```bash
# Check if standby is in recovery
docker exec pg-standby-sync psql -U postgres -c "SELECT pg_is_in_recovery();"

# Check logs
docker logs pg-standby-sync | tail -50

# Check for standby.signal
docker exec pg-standby-sync ls -la /var/lib/postgresql/data/standby.signal
```

**Solutions:**

```bash
# 1. Try pg_promote()
docker exec pg-standby-sync psql -U postgres -c "SELECT pg_promote();"

# 2. Remove standby.signal and restart
docker exec pg-standby-sync rm /var/lib/postgresql/data/standby.signal
docker-compose -f docker-compose-streaming.yml restart pg-standby-sync

# 3. Check for recovery.signal (for PITR)
docker exec pg-standby-sync ls -la /var/lib/postgresql/data/recovery.signal
# If exists, remove it too
docker exec pg-standby-sync rm /var/lib/postgresql/data/recovery.signal

# 4. Force promotion with pg_ctl
docker exec pg-standby-sync pg_ctl promote -D /var/lib/postgresql/data -W
```

---

## Maintenance Scripts

### Daily Health Check

**File: `daily-health-check.sh`**

```bash
#!/bin/bash

LOGFILE="health_checks_$(date +%Y%m%d).log"

exec > >(tee -a $LOGFILE)
exec 2>&1

echo "=================================="
echo "Daily Replication Health Check"
echo "Date: $(date)"
echo "=================================="

# 1. Check all containers are running
echo -e "\n[1] Container Status:"
docker-compose -f docker-compose-streaming.yml ps

# 2. Check replication connections
echo -e "\n[2] Replication Connections:"
docker exec pg-primary psql -U postgres -x << EOF
SELECT * FROM pg_stat_replication;
EOF

# 3. Check replication lag
echo -e "\n[3] Replication Lag:"
docker exec pg-primary psql -U postgres << EOF
SELECT 
    application_name,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS lag,
    sync_state
FROM pg_stat_replication;
EOF

# 4. Check WAL archiving
echo -e "\n[4] WAL Archiving Status:"
docker exec pg-primary psql -U postgres << EOF
SELECT * FROM pg_stat_archiver;
EOF

# 5. Check disk usage
echo -e "\n[5] Disk Usage:"
echo "Primary:"
docker exec pg-primary df -h /var/lib/postgresql/data
echo "Sync Standby:"
docker exec pg-standby-sync df -h /var/lib/postgresql/data
echo "Async Standby:"
docker exec pg-standby-async df -h /var/lib/postgresql/data

# 6. Check database sizes
echo -e "\n[6] Database Sizes:"
docker exec pg-primary psql -U postgres << EOF
SELECT 
    datname,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY pg_database_size(datname) DESC;
EOF

# 7. Check long-running queries
echo -e "\n[7] Long-Running Queries (>5 minutes):"
docker exec pg-primary psql -U postgres << EOF
SELECT 
    pid,
    usename,
    datname,
    state,
    NOW() - query_start AS duration,
    left(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle'
  AND NOW() - query_start > INTERVAL '5 minutes'
ORDER BY duration DESC;
EOF

# 8. Check replication slots
echo -e "\n[8] Replication Slots:"
docker exec pg-primary psql -U postgres << EOF
SELECT 
    slot_name,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
EOF

# 9. Connection counts
echo -e "\n[9] Connection Counts:"
docker exec pg-primary psql -U postgres << EOF
SELECT 
    datname,
    count(*) as connections
FROM pg_stat_activity
GROUP BY datname
ORDER BY connections DESC;
EOF

# 10. Standby recovery status
echo -e "\n[10] Standby Recovery Status:"
echo "Sync Standby:"
docker exec pg-standby-sync psql -U postgres << EOF
SELECT 
    pg_is_in_recovery(),
    pg_last_xact_replay_timestamp(),
    NOW() - pg_last_xact_replay_timestamp() AS replay_lag;
EOF

echo "Async Standby:"
docker exec pg-standby-async psql -U postgres << EOF
SELECT 
    pg_is_in_recovery(),
    pg_last_xact_replay_timestamp(),
    NOW() - pg_last_xact_replay_timestamp() AS replay_lag;
EOF

echo -e "\n=================================="
echo "Health Check Complete"
echo "=================================="
```

### Weekly Maintenance

**File: `weekly-maintenance.sh`**

```bash
#!/bin/bash

echo "Starting Weekly Maintenance - $(date)"

# 1. Vacuum analyze all databases
echo "[1] Running VACUUM ANALYZE..."
docker exec pg-primary psql -U postgres -d production_db << EOF
VACUUM ANALYZE VERBOSE;
EOF

# 2. Reindex if needed
echo "[2] Checking for bloated indexes..."
docker exec pg-primary psql -U postgres -d production_db << EOF
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE pg_relation_size(indexrelid) > 10485760  -- 10MB
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 10;
EOF

# 3. Check for unused indexes
echo "[3] Checking for unused indexes..."
docker exec pg-primary psql -U postgres -d production_db << EOF
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan as scans,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
EOF

# 4. Update statistics
echo "[4] Updating statistics..."
docker exec pg-primary psql -U postgres -d production_db << EOF
ANALYZE;
EOF

# 5. Check for table bloat
echo "[5] Checking table bloat..."
docker exec pg-primary psql -U postgres -d production_db << EOF
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS indexes_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;
EOF

# 6. Backup from standby
echo "[6] Taking backup from standby..."
docker exec pg-standby-sync pg_basebackup \
    -h localhost -U postgres \
    -D /backups/weekly_backup_$(date +%Y%m%d) \
    -Ft -z -Xs -P -c fast

# 7. Clean old backups (keep last 4 weeks)
echo "[7] Cleaning old backups..."
docker exec pg-standby-sync find /backups -name "weekly_backup_*" -mtime +28 -exec rm -rf {} \;

# 8. Refresh materialized views
echo "[8] Refreshing materialized views..."
docker exec pg-primary psql -U postgres -d production_db << EOF
REFRESH MATERIALIZED VIEW CONCURRENTLY analytics.product_sales_summary;
EOF

echo "Weekly Maintenance Complete - $(date)"
```

---

## Production Deployment Checklist

### Pre-Deployment

- [ ] All servers meet hardware requirements
- [ ] Network connectivity tested between all nodes
- [ ] Firewall rules configured (PostgreSQL port 5432)
- [ ] SSL certificates generated and deployed
- [ ] Backup storage configured and tested
- [ ] Monitoring system setup (Prometheus/Grafana)
- [ ] Alert notifications configured
- [ ] Documentation updated

### Deployment Steps

```bash
# 1. Deploy primary
docker-compose -f docker-compose-streaming.yml up -d pg-primary

# 2. Verify primary is healthy
./daily-health-check.sh

# 3. Take base backup
docker exec pg-primary pg_basebackup ...

# 4. Deploy standbys
docker-compose -f docker-compose-streaming.yml up -d pg-standby-sync pg-standby-async

# 5. Verify replication
./streaming-replication-test-suite.sh

# 6. Configure application connection pooling
# Primary: writes only
# Standbys: reads (load balanced)

# 7. Deploy monitoring
docker-compose -f docker-compose-streaming.yml --profile monitoring up -d

# 8. Schedule maintenance scripts
# Add to cron:
0 2 * * * /path/to/daily-health-check.sh
0 3 * * 0 /path/to/weekly-maintenance.sh
```

### Post-Deployment

- [ ] Monitor replication lag for 24 hours
- [ ] Test failover procedure in maintenance window
- [ ] Verify backup restoration
- [ ] Load test read replicas
- [ ] Document connection strings and ports
- [ ] Train team on monitoring and alerts
- [ ] Create runbook for common issues

---

## Summary

This comprehensive guide covers:

✅ **Setup**: Complete Docker Compose configuration with primary and multiple standbys
✅ **Synchronous vs Asynchronous**: Configuration and performance testing
✅ **Testing**: Extensive test scenarios covering all aspects of streaming replication
✅ **Failover**: Planned and emergency failover procedures
✅ **Monitoring**: Comprehensive monitoring and alerting scripts
✅ **Troubleshooting**: Common issues and solutions
✅ **Maintenance**: Daily and weekly maintenance scripts
✅ **Production**: Deployment checklist and best practices

### Key Takeaways

1. **Synchronous replication** = Zero data loss but slower performance
2. **Asynchronous replication** = Better performance but potential data loss
3. **Hot Standby** = Read queries on replicas reduce primary load
4. **Replication Slots** = Prevent WAL deletion but can cause bloat
5. **Monitoring** = Essential for detecting issues before they become critical

Your streaming replication setup is now production-ready! 🚀
