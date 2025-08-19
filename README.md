## PostgreSQL 18 Production Database Server

Production-grade PostgreSQL 18 beta3 instance optimized for 8-core/4GB memory allocation.

## Quick Start

```bash
# Create environment file
cp .env.example .env
# Edit .env and set POSTGRES_PASSWORD

# Start the service
docker-compose up -d

# Check status
docker logs postgres18 -f
```

## Database Management

### Create New Database
```sql
-- Connect as postgres user
psql -h localhost -U postgres -d postgres

-- Create database and user
CREATE DATABASE myapp_prod;
CREATE USER myapp_prod WITH ENCRYPTED PASSWORD 'secure_password';
GRANT ALL PRIVILEGES ON DATABASE myapp_prod TO myapp_prod;
```

## Connection Details

- **Host**: `localhost`
- **Port**: `5432`
- **Username**: `postgres`
- **Password**: Set in `.env` file
- **Format**: `postgresql://username:password@localhost:5432/database_name`

## Monitoring

```bash
# Container health
docker ps | grep postgres18
docker exec postgres18 pg_isready -U postgres

# Connection count
docker exec postgres18 psql -U postgres -c "SELECT count(*) FROM pg_stat_activity;"
```

## Directory Structure

```
~/postgres/
├── docker-compose.yml      # Container configuration
├── .env                    # Environment variables
├── README.md              # This file
├── init/                  # Database initialization scripts
└── data/18/docker/        # PostgreSQL data files
```

## Troubleshooting

```bash
# Check container logs
docker logs postgres18 -f

# Connect to container shell
docker exec -it postgres18 bash

# Check PostgreSQL configuration
docker exec postgres18 psql -U postgres -c "SHOW shared_buffers;"
```
