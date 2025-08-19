# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## PostgreSQL Production Database Server

This repository contains a production-grade PostgreSQL 18 beta3 instance configured for 8-core/4GB memory allocation. It's designed to support multiple side projects and future services.

## Essential Commands

### Docker Operations
```bash
# Start PostgreSQL service
docker-compose up -d

# Check container status and logs
docker ps | grep postgres18
docker logs postgres18 -f

# Stop service
docker-compose down

# Health check
docker exec postgres18 pg_isready -U postgres

# Connect to PostgreSQL
docker exec -it postgres18 psql -U postgres

# Container shell access
docker exec -it postgres18 bash
```

### Database Management
```bash
# Connect to PostgreSQL
docker exec -it postgres18 psql -U postgres

# Connection test
docker exec postgres18 psql -U postgres -c "SELECT version();"
```

## Architecture Overview

### Container Configuration
- **Image**: postgres:18beta3 (production-ready beta)
- **Container**: postgres18 with 4GB memory limit
- **Network**: Custom bridge network (postgres-network)
- **Health Check**: Built-in pg_isready monitoring

### Performance Tuning
The instance is optimized for 8-core Ryzen 7 systems with production-grade parameters:
- `shared_buffers=2GB` (50% of allocated memory)
- `work_mem=32MB` for complex queries
- `max_parallel_workers=8` utilizing all CPU cores
- `effective_io_concurrency=200` for NVMe storage
- `pg_stat_statements` extension for query performance monitoring

### Data Persistence
- **Data Directory**: `./data` maps to `/var/lib/postgresql` in container
- **Init Scripts**: `./init` directory for database initialization SQL files
- **Backups**: Handled by borgmatic with database dump

### Security Model
- Default postgres superuser with environment-controlled password
- Application-specific database users with limited permissions
- localhost-only access (no external network exposure)
- Non-root container execution

## Database Patterns

### Multi-Environment Setup
The configuration supports environment-based database separation:
- `appname_dev` for development
- `appname_stage` for staging  
- `appname_prod` for production

Each environment uses dedicated database users with appropriate permissions.

### UUID7 Support
PostgreSQL 18 includes native UUID7 support via `gen_uuid_v7()` function for time-ordered primary keys without extensions.

## Connection Details
- **Host**: localhost
- **Port**: 5432
- **Format**: `postgresql://username:password@localhost:5432/database_name`
- **Pool Settings**: Recommended 10 max connections, 2 min connections for applications

## Directory Structure
```
~/postgres/
├── docker-compose.yml      # Container configuration with production tuning
├── .env                    # Environment variables (POSTGRES_PASSWORD)
├── README.md              # Essential documentation
├── init/                  # Database initialization scripts
└── data/18/docker/        # PostgreSQL data files (persistent)
```

## Performance Monitoring

Use these SQL queries for monitoring:
```sql
-- Active connections by database
SELECT datname, count(*) FROM pg_stat_activity GROUP BY datname;

-- Query performance (requires pg_stat_statements)
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;

-- Buffer cache hit ratio (should be >90%)
SELECT (sum(heap_blks_hit) - sum(heap_blks_read)) / sum(heap_blks_hit) as ratio
FROM pg_statio_user_tables;
```

## Environment Setup

1. Create `.env` file with `POSTGRES_PASSWORD=your_secure_password`
2. Ensure directories exist: `mkdir -p data init logs`
3. Start with `docker-compose up -d`
4. Monitor with `docker logs postgres18 -f`