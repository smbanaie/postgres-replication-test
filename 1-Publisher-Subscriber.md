# Comprehensive Guide to PostgreSQL Replication (Publish/Subscribe)

## Introduction: PostgreSQL Replication Types

### 1. **Streaming Replication (Physical)**

- Binary replication of entire database cluster
- Replicas are exact copies
- Read-only replicas
- Fast and efficient

### 2. **Logical Replication (Publish/Subscribe)**

- Table-level replication
- Selective replication (specific tables/databases)
- Replicas can be written to
- Cross-version replication possible
- More flexible but slightly more overhead

We'll focus on **Logical Replication** with detailed testing scenarios.

---

## Docker Compose Setup

### Complete Docker Compose Configuration

**File: `docker-compose.yml`**

```yaml
version: '3.8'

services:
  # Publisher (Primary Database)
  pg-publisher:
    container_name: pg-publisher
    hostname: pg-publisher
    image: postgres:18
    volumes:
      - ./publisher_data:/var/lib/postgresql/data
      - ./publisher_init:/docker-entrypoint-initdb.d
      - ./backups:/backups
    environment:
      - POSTGRES_PASSWORD=postgres123
      - POSTGRES_USER=postgres
      - POSTGRES_DB=company_db
      - POSTGRES_HOST_AUTH_METHOD=md5
    networks:
      - replication_network
    ports:
      - "5433:5432"
    restart: always
    command: >
      postgres
      -c wal_level=logical
      -c max_wal_senders=10
      -c max_replication_slots=10
      -c max_logical_replication_workers=10

  # Subscriber (Replica Database)
  pg-subscriber:
    container_name: pg-subscriber
    hostname: pg-subscriber
    image: postgres:18
    volumes:
      - ./subscriber_data:/var/lib/postgresql/data
      - ./subscriber_init:/docker-entrypoint-initdb.d
      - ./backups:/backups
    environment:
      - POSTGRES_PASSWORD=postgres123
      - POSTGRES_USER=postgres
      - POSTGRES_DB=company_db
      - POSTGRES_HOST_AUTH_METHOD=md5
    networks:
      - replication_network
    ports:
      - "5434:5432"
    restart: always
    depends_on:
      - pg-publisher
    command: >
      postgres
      -c wal_level=logical
      -c max_wal_senders=10
      -c max_replication_slots=10
      -c max_logical_replication_workers=10

  # Optional: Second Subscriber for multi-subscriber testing
  pg-subscriber2:
    container_name: pg-subscriber2
    hostname: pg-subscriber2
    image: postgres:18
    profiles:
      - multi-subscriber
    volumes:
      - ./subscriber2_data:/var/lib/postgresql/data
      - ./subscriber2_init:/docker-entrypoint-initdb.d
    environment:
      - POSTGRES_PASSWORD=postgres123
      - POSTGRES_USER=postgres
      - POSTGRES_DB=company_db
      - POSTGRES_HOST_AUTH_METHOD=md5
    networks:
      - replication_network
    ports:
      - "5435:5432"
    restart: always
    depends_on:
      - pg-publisher
    command: >
      postgres
      -c wal_level=logical
      -c max_wal_senders=10
      -c max_replication_slots=10

  # PgAdmin for GUI management
  pgadmin:
    container_name: pgadmin-replication
    hostname: pgadmin-replication
    image: dpage/pgadmin4
    restart: always
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
      - replication_network

volumes:
  publisher_data:
  subscriber_data:
  subscriber2_data:
  pgadmin_data:

networks:
  replication_network:
    name: postgres_replication_network
    driver: bridge
```

---

## Initial Setup

### Step 1: Create Directory Structure

```bash
# Create directories
mkdir -p publisher_init subscriber_init subscriber2_init backups

# Set permissions
chmod -R 777 publisher_init subscriber_init subscriber2_init backups
```

### Step 2: Configure Publisher Initialization

**File: `publisher_init/01-setup-publisher.sql`**

```sql
-- Create replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replicator123';

-- Grant necessary permissions
GRANT ALL PRIVILEGES ON DATABASE company_db TO replicator;

-- Create sample schema and tables
CREATE SCHEMA IF NOT EXISTS sales;
CREATE SCHEMA IF NOT EXISTS hr;

-- Sales tables
CREATE TABLE sales.orders (
    id SERIAL PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    order_date TIMESTAMP DEFAULT NOW(),
    total_amount NUMERIC(10,2),
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE sales.order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES sales.orders(id) ON DELETE CASCADE,
    product_name VARCHAR(200),
    quantity INTEGER,
    price NUMERIC(10,2),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE sales.customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    phone VARCHAR(20),
    address TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- HR tables
CREATE TABLE hr.employees (
    id SERIAL PRIMARY KEY,
    employee_code VARCHAR(20) UNIQUE NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    department VARCHAR(50),
    salary NUMERIC(10,2),
    hire_date DATE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE hr.attendance (
    id SERIAL PRIMARY KEY,
    employee_id INTEGER REFERENCES hr.employees(id),
    check_in TIMESTAMP,
    check_out TIMESTAMP,
    work_date DATE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Create indexes
CREATE INDEX idx_orders_customer ON sales.orders(customer_name);
CREATE INDEX idx_orders_date ON sales.orders(order_date);
CREATE INDEX idx_order_items_order ON sales.order_items(order_id);
CREATE INDEX idx_employees_dept ON hr.employees(department);
CREATE INDEX idx_attendance_emp ON hr.attendance(employee_id);

-- Insert sample data
INSERT INTO sales.customers (name, email, phone, address) VALUES
    ('John Doe', 'john@example.com', '555-0101', '123 Main St'),
    ('Jane Smith', 'jane@example.com', '555-0102', '456 Oak Ave'),
    ('Bob Johnson', 'bob@example.com', '555-0103', '789 Pine Rd'),
    ('Alice Williams', 'alice@example.com', '555-0104', '321 Elm St'),
    ('Charlie Brown', 'charlie@example.com', '555-0105', '654 Maple Dr');

INSERT INTO sales.orders (customer_name, total_amount, status) VALUES
    ('John Doe', 1250.50, 'completed'),
    ('Jane Smith', 875.25, 'pending'),
    ('Bob Johnson', 2100.00, 'shipped'),
    ('Alice Williams', 450.75, 'completed'),
    ('Charlie Brown', 1875.00, 'processing');

INSERT INTO sales.order_items (order_id, product_name, quantity, price) VALUES
    (1, 'Laptop', 1, 1200.00),
    (1, 'Mouse', 1, 50.50),
    (2, 'Keyboard', 2, 75.00),
    (2, 'Monitor', 1, 725.25),
    (3, 'Desk', 1, 800.00),
    (3, 'Chair', 2, 650.00);

INSERT INTO hr.employees (employee_code, first_name, last_name, email, department, salary, hire_date) VALUES
    ('EMP001', 'Michael', 'Scott', 'michael@company.com', 'Management', 85000, '2020-01-15'),
    ('EMP002', 'Jim', 'Halpert', 'jim@company.com', 'Sales', 65000, '2020-03-20'),
    ('EMP003', 'Pam', 'Beesly', 'pam@company.com', 'Reception', 45000, '2020-02-10'),
    ('EMP004', 'Dwight', 'Schrute', 'dwight@company.com', 'Sales', 70000, '2020-01-20'),
    ('EMP005', 'Angela', 'Martin', 'angela@company.com', 'Accounting', 55000, '2020-04-01');

INSERT INTO hr.attendance (employee_id, check_in, check_out, work_date) VALUES
    (1, '2024-11-24 08:00:00', '2024-11-24 17:00:00', '2024-11-24'),
    (2, '2024-11-24 08:15:00', '2024-11-24 17:10:00', '2024-11-24'),
    (3, '2024-11-24 08:30:00', '2024-11-24 17:00:00', '2024-11-24'),
    (4, '2024-11-24 07:45:00', '2024-11-24 18:00:00', '2024-11-24'),
    (5, '2024-11-24 08:00:00', '2024-11-24 17:00:00', '2024-11-24');

-- Grant permissions to replicator
GRANT USAGE ON SCHEMA sales TO replicator;
GRANT USAGE ON SCHEMA hr TO replicator;
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO replicator;
GRANT SELECT ON ALL TABLES IN SCHEMA hr TO replicator;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA sales TO replicator;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA hr TO replicator;

-- Create publication for all tables
CREATE PUBLICATION company_publication FOR ALL TABLES;

-- Alternative: Create selective publications
-- CREATE PUBLICATION sales_publication FOR TABLE sales.orders, sales.order_items, sales.customers;
-- CREATE PUBLICATION hr_publication FOR TABLE hr.employees, hr.attendance;
```

### Step 3: Configure Subscriber Initialization

**File: `subscriber_init/01-setup-subscriber.sql`**

```sql
-- Create same schema structure (without data)
CREATE SCHEMA IF NOT EXISTS sales;
CREATE SCHEMA IF NOT EXISTS hr;

-- Sales tables (same structure as publisher)
CREATE TABLE sales.orders (
    id SERIAL PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    order_date TIMESTAMP DEFAULT NOW(),
    total_amount NUMERIC(10,2),
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE sales.order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES sales.orders(id) ON DELETE CASCADE,
    product_name VARCHAR(200),
    quantity INTEGER,
    price NUMERIC(10,2),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE sales.customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    phone VARCHAR(20),
    address TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- HR tables
CREATE TABLE hr.employees (
    id SERIAL PRIMARY KEY,
    employee_code VARCHAR(20) UNIQUE NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    department VARCHAR(50),
    salary NUMERIC(10,2),
    hire_date DATE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE hr.attendance (
    id SERIAL PRIMARY KEY,
    employee_id INTEGER REFERENCES hr.employees(id),
    check_in TIMESTAMP,
    check_out TIMESTAMP,
    work_date DATE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Create same indexes
CREATE INDEX idx_orders_customer ON sales.orders(customer_name);
CREATE INDEX idx_orders_date ON sales.orders(order_date);
CREATE INDEX idx_order_items_order ON sales.order_items(order_id);
CREATE INDEX idx_employees_dept ON hr.employees(department);
CREATE INDEX idx_attendance_emp ON hr.attendance(employee_id);

-- Note: We'll create subscription after containers are up
```

### Step 4: Start the Containers

```bash
# Start both publisher and subscriber
docker-compose up -d pg-publisher pg-subscriber

# Check logs
docker-compose logs -f

# Verify both are running
docker-compose ps
```

---

## Setting Up Replication

### Step 1: Configure pg_hba.conf on Publisher

```bash
# Access publisher container
docker exec -it pg-publisher bash

# Add replication entry to pg_hba.conf
echo "host    all    replicator    0.0.0.0/0    md5" >> /var/lib/postgresql/data/pg_hba.conf

# Reload configuration
psql -U postgres -c "SELECT pg_reload_conf();"

exit
```

### Step 2: Create Subscription on Subscriber

```bash
# Connect to subscriber
docker exec -it pg-subscriber psql -U postgres -d company_db

-- Create subscription
CREATE SUBSCRIPTION company_subscription
    CONNECTION 'host=pg-publisher port=5432 dbname=company_db user=replicator password=replicator123'
    PUBLICATION company_publication
    WITH (copy_data = true);

-- Verify subscription
\dRs+

-- Check subscription status
SELECT * FROM pg_stat_subscription;

\q
```

### Step 3: Verify Initial Data Sync

```bash
# Check data on subscriber
docker exec -it pg-subscriber psql -U postgres -d company_db

SELECT COUNT(*) FROM sales.orders;
SELECT COUNT(*) FROM sales.customers;
SELECT COUNT(*) FROM hr.employees;

-- Should match publisher data
\q
```

---

## Testing Scenarios

### Test 1: Basic INSERT Replication

**On Publisher:**

```bash
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
-- Insert new order
INSERT INTO sales.orders (customer_name, total_amount, status) 
VALUES ('New Customer', 999.99, 'pending');

-- Insert new employee
INSERT INTO hr.employees (employee_code, first_name, last_name, email, department, salary, hire_date)
VALUES ('EMP006', 'Stanley', 'Hudson', 'stanley@company.com', 'Sales', 60000, '2024-11-24');

-- Check publisher data
SELECT id, customer_name, total_amount FROM sales.orders ORDER BY id DESC LIMIT 3;
SELECT id, employee_code, first_name, last_name FROM hr.employees ORDER BY id DESC LIMIT 3;
EOF
```

**On Subscriber (wait 2-3 seconds):**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Verify replicated data
SELECT id, customer_name, total_amount FROM sales.orders ORDER BY id DESC LIMIT 3;
SELECT id, employee_code, first_name, last_name FROM hr.employees ORDER BY id DESC LIMIT 3;

-- Should see the new records
EOF
```

### Test 2: UPDATE Replication

**On Publisher:**

```bash
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
-- Update order status
UPDATE sales.orders 
SET status = 'shipped', updated_at = NOW() 
WHERE id = 2;

-- Update employee salary
UPDATE hr.employees 
SET salary = 75000, updated_at = NOW() 
WHERE employee_code = 'EMP002';

-- Show updated data
SELECT id, customer_name, status, updated_at FROM sales.orders WHERE id = 2;
SELECT employee_code, first_name, salary, updated_at FROM hr.employees WHERE employee_code = 'EMP002';
EOF
```

**On Subscriber:**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Verify updates
SELECT id, customer_name, status, updated_at FROM sales.orders WHERE id = 2;
SELECT employee_code, first_name, salary, updated_at FROM hr.employees WHERE employee_code = 'EMP002';
EOF
```

### Test 3: DELETE Replication

**On Publisher:**

```bash
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
-- Delete an order (will cascade to order_items)
DELETE FROM sales.orders WHERE id = 5;

-- Verify deletion
SELECT COUNT(*) FROM sales.orders;
EOF
```

**On Subscriber:**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Should also be deleted
SELECT COUNT(*) FROM sales.orders;
SELECT * FROM sales.orders WHERE id = 5;  -- Should return no rows
EOF
```

### Test 4: Bulk Operations

**On Publisher:**

```bash
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
-- Bulk insert
INSERT INTO sales.customers (name, email, phone)
SELECT 
    'Customer ' || i,
    'customer' || i || '@example.com',
    '555-' || LPAD(i::text, 4, '0')
FROM generate_series(100, 200) i;

-- Bulk update
UPDATE sales.customers 
SET address = 'Updated Address ' || id
WHERE id > 5;

-- Check count
SELECT COUNT(*) as customer_count FROM sales.customers;
EOF
```

**On Subscriber:**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Should match
SELECT COUNT(*) as customer_count FROM sales.customers;
SELECT * FROM sales.customers WHERE id > 100 LIMIT 5;
EOF
```

### Test 5: Transaction Replication

**On Publisher:**

```bash
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
BEGIN;

-- Create order and items in transaction
INSERT INTO sales.orders (customer_name, total_amount, status) 
VALUES ('Transaction Test', 5000.00, 'pending') 
RETURNING id;

-- Assume returned id is 6 (adjust based on your data)
INSERT INTO sales.order_items (order_id, product_name, quantity, price) VALUES
    (6, 'Product A', 2, 1000.00),
    (6, 'Product B', 3, 1000.00);

COMMIT;

-- Verify
SELECT o.id, o.customer_name, o.total_amount, 
       COUNT(oi.id) as item_count
FROM sales.orders o
LEFT JOIN sales.order_items oi ON o.id = oi.order_id
WHERE o.id = 6
GROUP BY o.id, o.customer_name, o.total_amount;
EOF
```

**On Subscriber:**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Should see complete transaction
SELECT o.id, o.customer_name, o.total_amount, 
       COUNT(oi.id) as item_count
FROM sales.orders o
LEFT JOIN sales.order_items oi ON o.id = oi.order_id
WHERE o.id = 6
GROUP BY o.id, o.customer_name, o.total_amount;
EOF
```

### Test 6: Schema Changes (DDL Replication)

**Note:** Logical replication does NOT automatically replicate DDL changes. You must apply them manually.

**On Publisher:**

```bash
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
-- Add new column
ALTER TABLE sales.orders ADD COLUMN shipping_address TEXT;

-- Update some records
UPDATE sales.orders 
SET shipping_address = '123 Shipping St' 
WHERE id = 1;

\d sales.orders
EOF
```

**On Subscriber (must apply manually):**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Apply same schema change
ALTER TABLE sales.orders ADD COLUMN shipping_address TEXT;

\d sales.orders

-- Now data will replicate
SELECT id, customer_name, shipping_address FROM sales.orders WHERE id = 1;
EOF
```

### Test 7: Conflict Detection

**On Publisher:**

```bash
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
UPDATE sales.orders SET total_amount = 9999.99 WHERE id = 1;
SELECT id, total_amount, updated_at FROM sales.orders WHERE id = 1;
EOF
```

**On Subscriber (create conflict):**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Stop subscription temporarily
ALTER SUBSCRIPTION company_subscription DISABLE;

-- Make conflicting change
UPDATE sales.orders SET total_amount = 8888.88 WHERE id = 1;
SELECT id, total_amount, updated_at FROM sales.orders WHERE id = 1;
EOF
```

**Re-enable and check:**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Re-enable subscription
ALTER SUBSCRIPTION company_subscription ENABLE;

-- Wait a few seconds, then check
-- Publisher's value should win (9999.99)
SELECT pg_sleep(3);
SELECT id, total_amount, updated_at FROM sales.orders WHERE id = 1;

-- Check for conflicts in logs
SELECT * FROM pg_stat_subscription_stats;
EOF
```

### Test 8: Replication Lag Monitoring

```bash
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
-- Check replication slots
SELECT slot_name, slot_type, active, 
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag
FROM pg_replication_slots;
EOF
```

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Check subscription status and lag
SELECT subname, 
       pid, 
       received_lsn,
       latest_end_lsn,
       last_msg_send_time,
       last_msg_receipt_time,
       latest_end_time
FROM pg_stat_subscription;
EOF
```

---

## Advanced Testing: Multiple Subscribers

### Start Second Subscriber

```bash
docker-compose --profile multi-subscriber up -d pg-subscriber2
```

### Setup Second Subscriber

**Copy schema structure:**

```bash
# Copy init script
cp subscriber_init/01-setup-subscriber.sql subscriber2_init/

# Restart to apply
docker-compose restart pg-subscriber2
```

**Configure second subscription:**

```bash
docker exec -it pg-subscriber2 psql -U postgres -d company_db << EOF
-- Create subscription with different name
CREATE SUBSCRIPTION company_subscription_2
    CONNECTION 'host=pg-publisher port=5432 dbname=company_db user=replicator password=replicator123'
    PUBLICATION company_publication
    WITH (copy_data = true);

-- Verify
SELECT * FROM pg_stat_subscription;
EOF
```

### Test Multi-Subscriber Replication

**On Publisher:**

```bash
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
INSERT INTO sales.orders (customer_name, total_amount, status) 
VALUES ('Multi-Sub Test', 7777.77, 'pending');

SELECT id, customer_name, total_amount FROM sales.orders ORDER BY id DESC LIMIT 1;
EOF
```

**On Subscriber 1:**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
SELECT id, customer_name, total_amount FROM sales.orders ORDER BY id DESC LIMIT 1;
EOF
```

**On Subscriber 2:**

```bash
docker exec -it pg-subscriber2 psql -U postgres -d company_db << EOF
SELECT id, customer_name, total_amount FROM sales.orders ORDER BY id DESC LIMIT 1;
EOF
```

---

## Selective Table Replication

### Create Separate Publications

**On Publisher:**

```bash
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
-- Drop existing publication
DROP PUBLICATION company_publication;

-- Create separate publications
CREATE PUBLICATION sales_only FOR TABLE sales.orders, sales.order_items, sales.customers;
CREATE PUBLICATION hr_only FOR TABLE hr.employees, hr.attendance;

-- List publications
\dRp+
EOF
```

### Subscribe to Specific Publication

**On Subscriber:**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Drop existing subscription
DROP SUBSCRIPTION company_subscription;

-- Subscribe only to sales tables
CREATE SUBSCRIPTION sales_subscription
    CONNECTION 'host=pg-publisher port=5432 dbname=company_db user=replicator password=replicator123'
    PUBLICATION sales_only
    WITH (copy_data = true);

-- Verify
SELECT * FROM pg_stat_subscription;
EOF
```

**Test selective replication:**

```bash
# On Publisher - insert to sales (should replicate)
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
INSERT INTO sales.orders (customer_name, total_amount, status) 
VALUES ('Sales Only Test', 1111.11, 'pending');
EOF

# On Subscriber - check sales (should appear)
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
SELECT * FROM sales.orders ORDER BY id DESC LIMIT 1;
EOF

# On Publisher - insert to HR (should NOT replicate)
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
INSERT INTO hr.employees (employee_code, first_name, last_name, email, department, salary, hire_date)
VALUES ('EMP999', 'Test', 'User', 'test@company.com', 'IT', 80000, '2024-11-24');

SELECT COUNT(*) FROM hr.employees;
EOF

# On Subscriber - check HR (should NOT appear)
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
SELECT COUNT(*) FROM hr.employees;
-- Count should be less than publisher
EOF
```

---

## Row Filtering (PostgreSQL 15+)

### Create Publication with Row Filter

**On Publisher:**

```bash
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
-- Create publication with row filter (only completed orders)
CREATE PUBLICATION completed_orders_pub 
FOR TABLE sales.orders 
WHERE (status = 'completed');

-- Grant access
GRANT SELECT ON sales.orders TO replicator;
EOF
```

**On Subscriber:**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Create new subscription for filtered data
CREATE SUBSCRIPTION completed_orders_sub
    CONNECTION 'host=pg-publisher port=5432 dbname=company_db user=replicator password=replicator123'
    PUBLICATION completed_orders_pub
    WITH (copy_data = true);
EOF
```

**Test row filtering:**

```bash
# On Publisher - insert completed order
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
INSERT INTO sales.orders (customer_name, total_amount, status) 
VALUES ('Completed Order', 5555.55, 'completed');

INSERT INTO sales.orders (customer_name, total_amount, status) 
VALUES ('Pending Order', 6666.66, 'pending');

SELECT id, customer_name, status FROM sales.orders ORDER BY id DESC LIMIT 2;
EOF

# On Subscriber - only completed should appear
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
SELECT id, customer_name, status FROM sales.orders ORDER BY id DESC LIMIT 2;
-- Should only see 'completed' orders
EOF
```

---

## Monitoring and Maintenance

### Comprehensive Monitoring Script

**File: `monitor-replication.sh`**

```bash
#!/bin/bash

echo "=== PUBLISHER STATUS ==="
docker exec pg-publisher psql -U postgres -d company_db << EOF
SELECT slot_name, 
       active, 
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS replication_lag
FROM pg_replication_slots;

SELECT * FROM pg_stat_replication;
EOF

echo ""
echo "=== SUBSCRIBER STATUS ==="
docker exec pg-subscriber psql -U postgres -d company_db << EOF
SELECT subname, 
       pid,
       relid::regclass,
       received_lsn,
       last_msg_send_time,
       last_msg_receipt_time,
       latest_end_time,
       EXTRACT(EPOCH FROM (NOW() - last_msg_receipt_time)) AS lag_seconds
FROM pg_stat_subscription;

SELECT * FROM pg_stat_subscription_stats;
EOF

echo ""
echo "=== DATA COMPARISON ==="
echo "Publisher counts:"
docker exec pg-publisher psql -U postgres -d company_db -t << EOF
SELECT 'orders: ' || COUNT(*) FROM sales.orders;
SELECT 'customers: ' || COUNT(*) FROM sales.customers;
SELECT 'employees: ' || COUNT(*) FROM hr.employees;
EOF

echo ""
echo "Subscriber counts:"
docker exec pg-subscriber psql -U postgres -d company_db -t << EOF
SELECT 'orders: ' || COUNT(*) FROM sales.orders;
SELECT 'customers: ' || COUNT(*) FROM sales.customers;
SELECT 'employees: ' || COUNT(*) FROM hr.employees;
EOF
```

Make it executable:

```bash
chmod +x monitor-replication.sh
./monitor-replication.sh
```

### Performance Testing Script

**File: `performance-test.sh`**

```bash
#!/bin/bash

echo "Starting replication performance test..."

# Insert 10000 records on publisher
START_TIME=$(date +%s)

docker exec pg-publisher psql -U postgres -d company_db << EOF
BEGIN;
INSERT INTO sales.customers (name, email, phone)
SELECT 
    'Perf Test Customer ' || i,
    'perftest' || i || '@example.com',
    '555-' || LPAD(i::text, 4, '0')
FROM generate_series(1, 10000) i;
COMMIT;

SELECT COUNT(*) as publisher_count FROM sales.customers;
EOF

END_TIME=$(date +%s)
INSERT_DURATION=$((END_TIME - START_TIME))

echo "Insert completed in ${INSERT_DURATION} seconds"
echo "Waiting for replication..."

# Wait and check subscriber
sleep 5

REPL_START=$(date +%s)
docker exec pg-subscriber psql -U postgres -d company_db << EOF
SELECT COUNT(*) as subscriber_count FROM sales.customers;
EOF
REPL_END=$(date +%s)
REPL_DURATION=$((REPL_END - REPL_START))

echo "Replication lag: ~${REPL_DURATION} seconds"
```

---

## Troubleshooting

### Check Replication Status

```bash
# On Publisher
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
-- Check active connections
SELECT * FROM pg_stat_replication;

-- Check replication slots
SELECT * FROM pg_replication_slots;

-- Check publications
SELECT * FROM pg_publication;
SELECT * FROM pg_publication_tables;
EOF
```

```bash
# On Subscriber
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Check subscriptions
SELECT * FROM pg_subscription;

-- Check subscription status
SELECT * FROM pg_stat_subscription;

-- Check for errors
SELECT * FROM pg_stat_subscription_stats;
EOF
```

### Common Issues and Solutions

**Issue 1: Subscription not receiving data**

```bash
# Check if subscription is enabled
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
SELECT subname, subenabled FROM pg_subscription;

-- If disabled, enable it
ALTER SUBSCRIPTION company_subscription ENABLE;

-- Refresh subscription
ALTER SUBSCRIPTION company_subscription REFRESH PUBLICATION;
EOF

# Check network connectivity
docker exec -it pg-subscriber psql \
    "host=pg-publisher port=5432 dbname=company_db user=replicator password=replicator123" \
    -c "SELECT 'Connection successful';"
```

**Issue 2: Replication slot conflicts**

```bash
# On Publisher - check for inactive slots
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
SELECT slot_name, active, 
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag
FROM pg_replication_slots
WHERE NOT active;

-- Drop inactive slot if needed
-- SELECT pg_drop_replication_slot('slot_name');
EOF
```

**Issue 3: Schema mismatch**

```bash
# Compare table structures
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
\d+ sales.orders
EOF

docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
\d+ sales.orders
EOF

# Structures must match exactly
```

**Issue 4: High replication lag**

```bash
# Check WAL accumulation on publisher
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
SELECT pg_size_pretty(
    pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)
) AS lag_bytes
FROM pg_replication_slots;

-- Check subscriber worker processes
SELECT * FROM pg_stat_activity 
WHERE backend_type = 'logical replication worker';
EOF
```

---

## Failover and Disaster Recovery

### Scenario 1: Planned Publisher Maintenance

**Step 1: Prepare for maintenance**

```bash
# Stop writes to publisher (application level)

# Wait for replication to catch up
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
SELECT slot_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag
FROM pg_replication_slots;
-- Wait until lag is 0 bytes or very small
EOF
```

**Step 2: Promote subscriber to primary**

```bash
# On subscriber - disable subscription
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
ALTER SUBSCRIPTION company_subscription DISABLE;

-- Drop subscription (optional)
-- DROP SUBSCRIPTION company_subscription;

-- Subscriber is now standalone and can accept writes
INSERT INTO sales.orders (customer_name, total_amount, status) 
VALUES ('Written on new primary', 1234.56, 'pending');

SELECT * FROM sales.orders ORDER BY id DESC LIMIT 1;
EOF
```

**Step 3: Reverse replication (subscriber becomes new publisher)**

```bash
# On new publisher (former subscriber)
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Create replication user if not exists
DO $$ 
BEGIN
    IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'replicator') THEN
        CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replicator123';
    END IF;
END $$;

GRANT USAGE ON SCHEMA sales, hr TO replicator;
GRANT SELECT ON ALL TABLES IN SCHEMA sales, hr TO replicator;

-- Create publication
CREATE PUBLICATION reverse_publication FOR ALL TABLES;
EOF

# Configure pg_hba.conf
docker exec -it pg-subscriber bash -c \
    "echo 'host all replicator 0.0.0.0/0 md5' >> /var/lib/postgresql/data/pg_hba.conf"

docker exec -it pg-subscriber psql -U postgres -c "SELECT pg_reload_conf();"
```

**Step 4: Setup old publisher as subscriber**

```bash
# On old publisher (now becoming subscriber)
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
-- Drop old publication
DROP PUBLICATION IF EXISTS company_publication;

-- Create subscription to new publisher
CREATE SUBSCRIPTION reverse_subscription
    CONNECTION 'host=pg-subscriber port=5432 dbname=company_db user=replicator password=replicator123'
    PUBLICATION reverse_publication
    WITH (copy_data = false);  -- Don't copy, already have data

SELECT * FROM pg_stat_subscription;
EOF
```

### Scenario 2: Emergency Failover

**Step 1: Detect publisher failure**

```bash
# Try to connect
docker exec -it pg-publisher psql -U postgres -d company_db -c "SELECT 1;" 2>&1 || echo "Publisher is down!"
```

**Step 2: Immediately promote subscriber**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Drop subscription (can't connect to publisher anyway)
ALTER SUBSCRIPTION company_subscription DISABLE;
DROP SUBSCRIPTION company_subscription;

-- Subscriber is now standalone
-- Point your application to this server (port 5434)

-- Verify data integrity
SELECT COUNT(*) FROM sales.orders;
SELECT COUNT(*) FROM hr.employees;
EOF
```

**Step 3: Redirect application traffic**

```bash
# Update application connection string from:
# host=localhost port=5433 (publisher)
# to:
# host=localhost port=5434 (subscriber, now primary)
```

### Scenario 3: Split-Brain Prevention

**File: `check-split-brain.sh`**

```bash
#!/bin/bash

echo "Checking for potential split-brain scenario..."

# Check if both are accepting writes
PUB_WRITABLE=$(docker exec pg-publisher psql -U postgres -d company_db -t -c "SELECT NOT pg_is_in_recovery();")
SUB_WRITABLE=$(docker exec pg-subscriber psql -U postgres -d company_db -t -c "SELECT NOT pg_is_in_recovery();")

echo "Publisher writable: $PUB_WRITABLE"
echo "Subscriber writable: $SUB_WRITABLE"

if [[ "$PUB_WRITABLE" == *"t"* ]] && [[ "$SUB_WRITABLE" == *"t"* ]]; then
    echo "WARNING: Both databases are writable!"
    echo "Check replication status:"

    docker exec pg-subscriber psql -U postgres -d company_db -c \
        "SELECT subname, subenabled FROM pg_subscription;"

    echo "If subscription is disabled, you may have a split-brain situation."
fi
```

---

## Advanced Configurations

### Bi-Directional Replication

**On Publisher:**

```bash
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
-- Create publication (already exists)
-- Ensure we can also subscribe
CREATE SUBSCRIPTION bidirectional_sub
    CONNECTION 'host=pg-subscriber port=5432 dbname=company_db user=replicator password=replicator123'
    PUBLICATION company_publication
    WITH (copy_data = false, origin = none);  -- origin=none prevents loops
EOF
```

**On Subscriber:**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Ensure publication exists
CREATE PUBLICATION company_publication FOR ALL TABLES;

-- Ensure subscription exists (already created)
-- Modify to prevent loops
ALTER SUBSCRIPTION company_subscription SET (origin = none);
EOF
```

**Test bi-directional:**

```bash
# Write to publisher
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
INSERT INTO sales.orders (customer_name, total_amount, status) 
VALUES ('From Publisher', 1111.11, 'pending');
EOF

# Write to subscriber
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
INSERT INTO sales.orders (customer_name, total_amount, status) 
VALUES ('From Subscriber', 2222.22, 'pending');
EOF

# Check both
sleep 3

echo "Publisher data:"
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
SELECT id, customer_name, total_amount FROM sales.orders ORDER BY id DESC LIMIT 2;
EOF

echo "Subscriber data:"
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
SELECT id, customer_name, total_amount FROM sales.orders ORDER BY id DESC LIMIT 2;
EOF
```

### Column-Level Replication (PostgreSQL 15+)

**On Publisher:**

```bash
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
-- Create publication with specific columns (e.g., hide salary)
CREATE PUBLICATION hr_limited 
FOR TABLE hr.employees (id, employee_code, first_name, last_name, email, department, hire_date);

-- Note: salary column excluded
EOF
```

**On Subscriber:**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Subscribe to limited publication
CREATE SUBSCRIPTION hr_limited_sub
    CONNECTION 'host=pg-publisher port=5432 dbname=company_db user=replicator password=replicator123'
    PUBLICATION hr_limited
    WITH (copy_data = true);
EOF
```

### Conflict Resolution Strategies

**Create conflict detection function:**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Create function to log conflicts
CREATE TABLE IF NOT EXISTS replication_conflicts (
    id SERIAL PRIMARY KEY,
    table_name TEXT,
    conflict_type TEXT,
    conflict_data JSONB,
    detected_at TIMESTAMP DEFAULT NOW()
);

-- Create trigger function to detect conflicts
CREATE OR REPLACE FUNCTION detect_conflicts()
RETURNS TRIGGER AS \$\$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        IF OLD.updated_at > NEW.updated_at THEN
            INSERT INTO replication_conflicts (table_name, conflict_type, conflict_data)
            VALUES (TG_TABLE_NAME, 'timestamp_conflict', 
                    jsonb_build_object('old', row_to_json(OLD), 'new', row_to_json(NEW)));
        END IF;
    END IF;
    RETURN NEW;
END;
\$\$ LANGUAGE plpgsql;

-- Apply to tables
CREATE TRIGGER orders_conflict_detection
    BEFORE UPDATE ON sales.orders
    FOR EACH ROW EXECUTE FUNCTION detect_conflicts();
EOF
```

---

## Performance Optimization

### Parallel Apply Workers (PostgreSQL 16+)

**On Subscriber:**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Enable parallel apply
ALTER SUBSCRIPTION company_subscription 
SET (streaming = parallel);

-- Set number of parallel workers
ALTER SYSTEM SET max_parallel_workers = 8;
ALTER SYSTEM SET max_parallel_workers_per_gather = 4;

SELECT pg_reload_conf();
EOF
```

### Large Object Replication

**On Publisher:**

```bash
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
-- Create table with large objects
CREATE TABLE sales.product_images (
    id SERIAL PRIMARY KEY,
    product_id INTEGER,
    image_data BYTEA,
    image_size INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Insert sample large data
INSERT INTO sales.product_images (product_id, image_data, image_size)
SELECT 
    i,
    decode(repeat('FF', 1024 * 100), 'hex'),  -- 100KB of data
    1024 * 100
FROM generate_series(1, 10) i;

-- Add to publication
ALTER PUBLICATION company_publication ADD TABLE sales.product_images;
EOF
```

**On Subscriber:**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Create same table structure
CREATE TABLE sales.product_images (
    id SERIAL PRIMARY KEY,
    product_id INTEGER,
    image_data BYTEA,
    image_size INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Refresh subscription
ALTER SUBSCRIPTION company_subscription REFRESH PUBLICATION;

-- Verify data
SELECT id, product_id, image_size, 
       pg_size_pretty(length(image_data)::bigint) as data_size
FROM sales.product_images;
EOF
```

### Tune Replication Settings

**On Publisher:**

```bash
docker exec -it pg-publisher psql -U postgres << EOF
-- Increase WAL settings for better performance
ALTER SYSTEM SET wal_writer_delay = '200ms';
ALTER SYSTEM SET wal_writer_flush_after = '1MB';
ALTER SYSTEM SET max_wal_size = '4GB';
ALTER SYSTEM SET min_wal_size = '1GB';

SELECT pg_reload_conf();

-- Check settings
SHOW wal_writer_delay;
SHOW max_wal_size;
EOF
```

**On Subscriber:**

```bash
docker exec -it pg-subscriber psql -U postgres << EOF
-- Increase logical replication workers
ALTER SYSTEM SET max_logical_replication_workers = 8;
ALTER SYSTEM SET max_sync_workers_per_subscription = 4;

SELECT pg_reload_conf();

-- Verify
SHOW max_logical_replication_workers;
EOF
```

---

## Comprehensive Testing Suite

**File: `test-suite.sh`**

```bash
#!/bin/bash

set -e

echo "======================================"
echo "PostgreSQL Replication Test Suite"
echo "======================================"

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Test counter
PASSED=0
FAILED=0

# Helper function to run test
run_test() {
    local test_name=$1
    local test_command=$2

    echo -e "\n${YELLOW}Running: $test_name${NC}"

    if eval "$test_command"; then
        echo -e "${GREEN}✓ PASSED${NC}"
        ((PASSED++))
    else
        echo -e "${RED}✗ FAILED${NC}"
        ((FAILED++))
    fi
}

# Test 1: Check containers are running
run_test "Containers Running" \
    "docker-compose ps | grep -q 'pg-publisher.*Up' && docker-compose ps | grep -q 'pg-subscriber.*Up'"

# Test 2: Check database connectivity
run_test "Publisher Connectivity" \
    "docker exec pg-publisher psql -U postgres -d company_db -c 'SELECT 1;' > /dev/null 2>&1"

run_test "Subscriber Connectivity" \
    "docker exec pg-subscriber psql -U postgres -d company_db -c 'SELECT 1;' > /dev/null 2>&1"

# Test 3: Check replication setup
run_test "Replication Slot Active" \
    "docker exec pg-publisher psql -U postgres -d company_db -t -c \"SELECT active FROM pg_replication_slots;\" | grep -q 't'"

run_test "Subscription Active" \
    "docker exec pg-subscriber psql -U postgres -d company_db -t -c \"SELECT subenabled FROM pg_subscription;\" | grep -q 't'"

# Test 4: Data consistency check
echo -e "\n${YELLOW}Testing Data Consistency...${NC}"

# Insert test data on publisher
TEST_ID=$(docker exec pg-publisher psql -U postgres -d company_db -t -c \
    "INSERT INTO sales.orders (customer_name, total_amount, status) VALUES ('Test User', 999.99, 'test') RETURNING id;" | tr -d ' ')

echo "Inserted test order with ID: $TEST_ID"
sleep 2  # Wait for replication

# Check on subscriber
SUBSCRIBER_COUNT=$(docker exec pg-subscriber psql -U postgres -d company_db -t -c \
    "SELECT COUNT(*) FROM sales.orders WHERE id = $TEST_ID;" | tr -d ' ')

if [ "$SUBSCRIBER_COUNT" = "1" ]; then
    echo -e "${GREEN}✓ Data replicated successfully${NC}"
    ((PASSED++))
else
    echo -e "${RED}✗ Data not replicated${NC}"
    ((FAILED++))
fi

# Test 5: Replication lag
echo -e "\n${YELLOW}Checking Replication Lag...${NC}"
LAG=$(docker exec pg-publisher psql -U postgres -d company_db -t -c \
    "SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn) FROM pg_replication_slots;" | tr -d ' ')

if [ "$LAG" -lt 1048576 ]; then  # Less than 1MB
    echo -e "${GREEN}✓ Replication lag is acceptable: $LAG bytes${NC}"
    ((PASSED++))
else
    echo -e "${YELLOW}⚠ Replication lag is high: $LAG bytes${NC}"
    ((PASSED++))
fi

# Test 6: Bulk insert performance
echo -e "\n${YELLOW}Testing Bulk Insert Performance...${NC}"
START_TIME=$(date +%s)

docker exec pg-publisher psql -U postgres -d company_db -c \
    "INSERT INTO sales.customers (name, email, phone) 
     SELECT 'Bulk Test ' || i, 'bulktest' || i || '@test.com', '555-' || LPAD(i::text, 4, '0')
     FROM generate_series(1, 1000) i;" > /dev/null 2>&1

END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

echo "Bulk insert completed in $DURATION seconds"
sleep 3  # Wait for replication

PUBLISHER_COUNT=$(docker exec pg-publisher psql -U postgres -d company_db -t -c \
    "SELECT COUNT(*) FROM sales.customers;" | tr -d ' ')
SUBSCRIBER_COUNT=$(docker exec pg-subscriber psql -U postgres -d company_db -t -c \
    "SELECT COUNT(*) FROM sales.customers;" | tr -d ' ')

if [ "$PUBLISHER_COUNT" = "$SUBSCRIBER_COUNT" ]; then
    echo -e "${GREEN}✓ Bulk data replicated: $PUBLISHER_COUNT records${NC}"
    ((PASSED++))
else
    echo -e "${RED}✗ Count mismatch: Publisher=$PUBLISHER_COUNT, Subscriber=$SUBSCRIBER_COUNT${NC}"
    ((FAILED++))
fi

# Test 7: Transaction consistency
echo -e "\n${YELLOW}Testing Transaction Consistency...${NC}"

docker exec pg-publisher psql -U postgres -d company_db << EOF > /dev/null 2>&1
BEGIN;
INSERT INTO sales.orders (customer_name, total_amount, status) VALUES ('TX Test', 5000, 'pending') RETURNING id;
INSERT INTO sales.order_items (order_id, product_name, quantity, price) VALUES 
    (currval('sales.orders_id_seq'), 'TX Product 1', 1, 2000),
    (currval('sales.orders_id_seq'), 'TX Product 2', 1, 3000);
COMMIT;
EOF

sleep 2

ORDER_ITEMS=$(docker exec pg-subscriber psql -U postgres -d company_db -t -c \
    "SELECT COUNT(*) FROM sales.order_items WHERE order_id = (SELECT id FROM sales.orders WHERE customer_name = 'TX Test');" | tr -d ' ')

if [ "$ORDER_ITEMS" = "2" ]; then
    echo -e "${GREEN}✓ Transaction replicated atomically${NC}"
    ((PASSED++))
else
    echo -e "${RED}✗ Transaction incomplete: $ORDER_ITEMS items found${NC}"
    ((FAILED++))
fi

# Summary
echo -e "\n======================================"
echo -e "Test Summary"
echo -e "======================================"
echo -e "${GREEN}Passed: $PASSED${NC}"
echo -e "${RED}Failed: $FAILED${NC}"
echo -e "Total: $((PASSED + FAILED))"

if [ $FAILED -eq 0 ]; then
    echo -e "\n${GREEN}All tests passed!${NC}"
    exit 0
else
    echo -e "\n${RED}Some tests failed!${NC}"
    exit 1
fi
```

Make it executable:

```bash
chmod +x test-suite.sh
./test-suite.sh
```

---

## Backup and Recovery with Replication

### Backup Strategy with Active Replication

**Take backup from subscriber (no impact on publisher):**

```bash
# Physical backup from subscriber
docker exec pg-subscriber pg_basebackup \
    -h localhost -U postgres \
    -D /backups/subscriber_backup_$(date +%Y%m%d) \
    -Ft -z -Xs -P -c fast

# Logical backup
docker exec pg-subscriber pg_dump -U postgres company_db \
    -Fp -v -f /backups/logical_backup_$(date +%Y%m%d).sql
```

### Point-in-Time Recovery with Replication

**Setup WAL archiving on subscriber:**

```bash
docker exec -it pg-subscriber psql -U postgres << EOF
ALTER SYSTEM SET archive_mode = on;
ALTER SYSTEM SET archive_command = 'test ! -f /backups/archive/%f && cp %p /backups/archive/%f';
SELECT pg_reload_conf();
EOF

# Create archive directory
docker exec pg-subscriber mkdir -p /backups/archive
```

---

## Cleanup and Maintenance

### Clean Test Data

```bash
# Remove test data
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
DELETE FROM sales.orders WHERE customer_name LIKE '%Test%';
DELETE FROM sales.customers WHERE name LIKE '%Test%' OR name LIKE '%Bulk%';
VACUUM ANALYZE;
EOF

# Wait for replication
sleep 3

docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
VACUUM ANALYZE;
EOF
```

### Drop Replication

**On Subscriber:**

```bash
docker exec -it pg-subscriber psql -U postgres -d company_db << EOF
-- Drop subscription
DROP SUBSCRIPTION IF EXISTS company_subscription;
DROP SUBSCRIPTION IF EXISTS sales_subscription;
DROP SUBSCRIPTION IF EXISTS completed_orders_sub;
EOF
```

**On Publisher:**

```bash
docker exec -it pg-publisher psql -U postgres -d company_db << EOF
-- Drop publications
DROP PUBLICATION IF EXISTS company_publication;
DROP PUBLICATION IF EXISTS sales_only;
DROP PUBLICATION IF EXISTS hr_only;
DROP PUBLICATION IF EXISTS completed_orders_pub;

-- Drop replication slot (if needed)
SELECT pg_drop_replication_slot(slot_name) 
FROM pg_replication_slots 
WHERE NOT active;
EOF
```

### Complete Cleanup

```bash
# Stop containers
docker-compose down

# Remove volumes
docker volume rm postgres_replication_network

# Remove data directories
sudo rm -rf publisher_data subscriber_data subscriber2_data pgadmin_data

# Start fresh
docker-compose up -d
```

---

## Production Best Practices

### 1. **Monitoring Setup**

**Create monitoring table:**

```sql
CREATE TABLE replication_monitoring (
    id SERIAL PRIMARY KEY,
    server_name VARCHAR(50),
    lag_bytes BIGINT,
    lag_seconds INTEGER,
    check_time TIMESTAMP DEFAULT NOW()
);

-- Function to log replication lag
CREATE OR REPLACE FUNCTION log_replication_lag()
RETURNS void AS $$
BEGIN
    INSERT INTO replication_monitoring (server_name, lag_bytes, lag_seconds)
    SELECT 
        'publisher',
        pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn),
        EXTRACT(EPOCH FROM (NOW() - last_msg_receipt_time))::INTEGER
    FROM pg_replication_slots
    WHERE active;
END;
$$ LANGUAGE plpgsql;
```

### 2. **Alerting**

**File: `alert-replication-lag.sh`**

```bash
#!/bin/bash

THRESHOLD_MB=100
THRESHOLD_SECONDS=60

LAG_BYTES=$(docker exec pg-publisher psql -U postgres -d company_db -t -c \
    "SELECT COALESCE(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn), 0) FROM pg_replication_slots WHERE active;" | tr -d ' ')

LAG_MB=$((LAG_BYTES / 1024 / 1024))

if [ $LAG_MB -gt $THRESHOLD_MB ]; then
    echo "ALERT: Replication lag is ${LAG_MB}MB (threshold: ${THRESHOLD_MB}MB)"
    # Send email, Slack notification, etc.
fi
```

### 3. **Security Hardening**

```sql
-- Use SSL for replication
-- In postgresql.conf:
-- ssl = on
-- ssl_cert_file = '/path/to/server.crt'
-- ssl_key_file = '/path/to/server.key'

-- Connection string with SSL:
CREATE SUBSCRIPTION secure_subscription
    CONNECTION 'host=pg-publisher port=5432 dbname=company_db 
                 user=replicator password=replicator123 
                 sslmode=require sslcert=/path/to/client.crt sslkey=/path/to/client.key'
    PUBLICATION company_publication;
```

### 4. **Capacity Planning**

```bash
# Monitor WAL generation rate
docker exec pg-publisher psql -U postgres << EOF
SELECT 
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0')) as total_wal_generated,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') / 
        EXTRACT(EPOCH FROM (NOW() - pg_postmaster_start_time()))) as wal_per_second;
EOF
```

---

## Documentation and Runbook

**File: `REPLICATION_RUNBOOK.md`**

```markdown
# PostgreSQL Replication Runbook

## Quick Reference

### Connection Details
- Publisher: localhost:5433
- Subscriber: localhost:5434
- Username: postgres
- Password: postgres123

### Common Commands

**Check Replication Status:**
```bash
./monitor-replication.sh
```

**Restart Replication:**

```bash
# On subscriber
docker exec pg-subscriber psql -U postgres -d company_db \
    -c "ALTER SUBSCRIPTION company_subscription DISABLE;"
docker exec pg-subscriber psql -U postgres -d company_db \
    -c "ALTER SUBSCRIPTION company_subscription ENABLE;"
```

**Failover to Subscriber:**

```bash
# 1. Stop application writes
# 2. Wait for lag to be zero
# 3. Drop subscription on subscriber
# 4. Point application to subscriber
```

### Emergency Contacts

- DBA On-Call: [phone]
- DevOps: [phone]
- Manager: [phone]

```
---

This comprehensive guide covers all aspects of PostgreSQL 18 replication with practical, testable scenarios. You can run through each section step-by-step to understand how logical replication works in practice.
```
