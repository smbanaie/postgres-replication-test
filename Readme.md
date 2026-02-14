#### Replication in PostgreSQL

One of the main features of PostgreSQL is the capability of ***Replication***.

## Overview

PostgreSQL replication is a powerful feature that allows you to maintain multiple copies of your database across different servers. This ensures high availability, data redundancy, and improved read performance.

## Key Benefits

- **High Availability**: Automatic failover in case of primary server failure
- **Data Redundancy**: Multiple copies of your data for disaster recovery
- **Load Balancing**: Distribute read queries across replica servers
- **Data Protection**: Safeguard against data loss with off-site backups
- **Zero Downtime Upgrades**: Upgrade PostgreSQL with minimal service disruption

## Types of Replication

### 1. Streaming Replication
- Real-time replication of WAL (Write-Ahead Log) records
- Synchronous or asynchronous modes
- Best for high availability setups

### 2. Logical Replication
- Replicates changes at the logical level (DML statements)
- More flexible - can replicate specific tables or databases
- Useful for selective data replication across versions

## Architecture
