## PostgreSQL 18 Production Database Server

Production-grade PostgreSQL 18 beta instance optimized for 8-core/4GB memory allocation, designed for multiple side projects and future services.

## Overview

- **Version**: PostgreSQL 18 beta3
- **Memory**: 4GB container limit (2GB shared_buffers)
- **CPU**: Optimized for 8-core Ryzen 7 (parallel workers enabled)
- **Storage**: Persistent data on host filesystem
- **Features**: UUID7 support, parallel query execution, production logging

## Quick Start

```bash
# Clone/create the postgres directory
cd ~/postgres

# Create environment file
cp .env.example .env
# Edit .env and set POSTGRES_PASSWORD

# Create required directories
mkdir -p data init

# Start the service
docker-compose up -d

# Check status
docker logs postgres18 -f
```

## Configuration

### Memory Allocation (4GB Container)
- `shared_buffers`: 2GB (PostgreSQL cache)
- `work_mem`: 32MB (per operation)
- `maintenance_work_mem`: 512MB (for VACUUM, CREATE INDEX)
- `effective_cache_size`: 3GB (query planner hint)

### CPU Optimization (8-core system)
- `max_worker_processes`: 16 (background workers)
- `max_parallel_workers`: 8 (parallel query workers)
- `max_parallel_workers_per_gather`: 4 (per query)
- `max_parallel_maintenance_workers`: 4 (for maintenance ops)

### Storage Optimization (NVMe)
- `random_page_cost`: 1.1 (SSD optimized)
- `effective_io_concurrency`: 200 (concurrent I/O)
- `wal_buffers`: 32MB (write-ahead log buffering)

## Database Management

### Create New Database
```sql
-- Connect as postgres user
psql -h localhost -U postgres -d postgres

-- Create database and user
CREATE DATABASE myapp_prod;
CREATE USER myapp_prod WITH ENCRYPTED PASSWORD 'secure_password';
GRANT ALL PRIVILEGES ON DATABASE myapp_prod TO myapp_prod;

-- For UUID7 support (PostgreSQL 18 feature)
\c myapp_prod;
-- gen_uuid_v7() is built-in, no extension needed
```

### Environment-Based Databases
```sql
-- Development, staging, production pattern
CREATE DATABASE myapp_dev;
CREATE DATABASE myapp_stage; 
CREATE DATABASE myapp_prod;

-- Separate users for each environment
CREATE USER myapp_dev WITH ENCRYPTED PASSWORD 'dev_password';
CREATE USER myapp_stage WITH ENCRYPTED PASSWORD 'stage_password';
CREATE USER myapp_prod WITH ENCRYPTED PASSWORD 'prod_password';

-- Grant permissions
GRANT ALL PRIVILEGES ON DATABASE myapp_dev TO myapp_dev;
GRANT ALL PRIVILEGES ON DATABASE myapp_stage TO myapp_stage;
GRANT ALL PRIVILEGES ON DATABASE myapp_prod TO myapp_prod;
```

### UUID7 Example
```sql
-- Create table with UUID7 primary key
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Insert test data
INSERT INTO users (email, name) VALUES 
('user1@example.com', 'User One'),
('user2@example.com', 'User Two');

-- UUID7 provides time-ordered UUIDs
SELECT id, email, created_at FROM users ORDER BY id;
```

## Connection Details

### Default Connection
- **Host**: `localhost` (or container IP)
- **Port**: `5432`
- **Username**: `postgres`
- **Password**: Set in `.env` file
- **Database**: `postgres`

### Application Connection Strings
```bash
# PostgreSQL URL format
postgresql://username:password@localhost:5432/database_name

# Examples
postgresql://postgres:your_password@localhost:5432/postgres
postgresql://myapp_prod:prod_password@localhost:5432/myapp_prod
```

### Go Connection Example
```go
import "github.com/jackc/pgx/v5/pgxpool"

config, err := pgxpool.ParseConfig("postgresql://myapp_prod:prod_password@localhost:5432/myapp_prod")
if err != nil {
    log.Fatal(err)
}

// Connection pool settings
config.MaxConns = 10
config.MinConns = 2
config.MaxConnLifetime = time.Hour
config.MaxConnIdleTime = time.Minute * 30

pool, err := pgxpool.NewWithConfig(context.Background(), config)
```

## Monitoring

### Health Check
```bash
# Container health
docker ps | grep postgres18

# Database connectivity
docker exec postgres18 pg_isready -U postgres

# Connection count
docker exec postgres18 psql -U postgres -c "SELECT count(*) FROM pg_stat_activity;"
```

### Performance Monitoring
```sql
-- Active connections
SELECT datname, count(*) 
FROM pg_stat_activity 
GROUP BY datname;

-- Database statistics
SELECT datname, numbackends, xact_commit, xact_rollback, blks_read, blks_hit
FROM pg_stat_database 
WHERE datname NOT IN ('template0', 'template1');

-- Query performance (requires pg_stat_statements)
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements 
ORDER BY total_exec_time DESC 
LIMIT 10;
```

## Backup Strategy

### Integration with Kopia
The data directory (`./data`) is automatically included in your existing Kopia backup setup since it's under `/home/server/postgres/`.

### Manual Backup
```bash
# Database dump
docker exec postgres18 pg_dumpall -U postgres > backup_$(date +%Y%m%d).sql

# Individual database
docker exec postgres18 pg_dump -U postgres myapp_prod > myapp_prod_$(date +%Y%m%d).sql

# Restore
docker exec -i postgres18 psql -U postgres < backup_20240819.sql
```

## Directory Structure

```
~/postgres/
├── docker-compose.yml      # Container configuration
├── .env                    # Environment variables (not in git)
├── .env.example           # Environment template
├── README.md              # This file
├── init/                  # Database initialization scripts
│   └── 01-create-databases.sql
└── data/                  # PostgreSQL data (not in git)
    └── 18/
        └── docker/        # PostgreSQL 18 data files
```

## Scaling

### Memory Scaling
To increase memory allocation:
```yaml
# In docker-compose.yml
deploy:
  resources:
    limits:
      memory: 8GB        # Increase container limit
    reservations:
      memory: 4GB

# Update shared_buffers accordingly
-c shared_buffers=4GB      # 25% of new memory limit
-c effective_cache_size=6GB # 75% of new memory limit
```

### Adding More Databases
Simply create new databases as shown above. The current configuration can handle multiple moderate-sized applications.

## Troubleshooting

### Common Issues
```bash
# Check container logs
docker logs postgres18 -f

# Check container resource usage
docker stats postgres18

# Connect to container shell
docker exec -it postgres18 bash

# Check PostgreSQL configuration
docker exec postgres18 psql -U postgres -c "SHOW shared_buffers;"
docker exec postgres18 psql -U postgres -c "SHOW max_connections;"
```

### Performance Issues
```sql
-- Check for long-running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query 
FROM pg_stat_activity 
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';

-- Check buffer cache hit ratio (should be >90%)
SELECT 
  sum(heap_blks_read) as heap_read,
  sum(heap_blks_hit)  as heap_hit,
  (sum(heap_blks_hit) - sum(heap_blks_read)) / sum(heap_blks_hit) as ratio
FROM pg_statio_user_tables;
```

## Security Notes

- Default `postgres` user password set via environment variable
- Database users have limited permissions to their specific databases
- Container runs with non-root user (postgres)
- No external network exposure by default (localhost only)

## Development vs Production

This configuration is production-ready but also suitable for development:
- **Development**: Create `_dev` databases with relaxed settings
- **Staging**: Use `_stage` databases with production-like data
- **Production**: Use `_prod` databases with full monitoring

## Future Upgrades

When PostgreSQL 18 stable is released:
```yaml
# Simply update the image tag
image: postgres:18  # Remove 'beta3'
```

The data directory structure (`./data/18/docker/`) will remain compatible.
