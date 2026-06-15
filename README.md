# autofik.plateform.dev.enviorment

A collection of Docker Compose configurations for running the Autofik platform's infrastructure services locally. Each service is isolated in its own directory with its own compose file and network, so you can start only what you need.

## Services

| Service | Directory | Port(s) | Description |
|---------|-----------|---------|-------------|
| PostgreSQL (PostGIS) | `database/pgdb/` | `5432` | Primary database with spatial extension |
| PgAdmin | `database/pgadmin/` | `8080` | PostgreSQL web UI |
| Redis | `database/redis/` | `6379` | In-memory cache |
| Kafka | `kafka/` | `9092` (broker), `9000` (UI) | Message broker (KRaft mode) |
| Zipkin | `zipkin/` | `9411` | Distributed tracing UI and API |

Each service has its own README with detailed setup and troubleshooting instructions.

## Quick Start

Run these steps in order to bring up the full stack.

### 1. Create Docker networks

```powershell
docker network create postgres-network
docker network create kafka-network
docker network create zipkin-network
```

### 2. Start PostgreSQL + PgAdmin

```powershell
docker compose -f database/pgdb/docker-compose.yml up -d
docker compose -f database/pgadmin/docker-compose.yml up -d
```

> Linux/Mac: use `database/pgdb/linux/docker-compose.yml` for PostgreSQL instead.

### 3. Start Redis

```powershell
docker compose -f database/redis/docker-compose.yml up -d
```

### 4. Start Kafka + Kafka UI

```powershell
docker compose -f kafka/docker-compose.yml up -d
```

### 5. Start Zipkin

```powershell
docker compose -f zipkin/docker-compose.yml up -d
```

## Access

| Service | URL / Address |
|---------|--------------|
| PgAdmin | http://localhost:8080 |
| Kafka UI | http://localhost:9000 |
| Zipkin UI | http://localhost:9411 |
| PostgreSQL | `localhost:5432` |
| Redis | `localhost:6379` |
| Kafka | `localhost:9092` |

## Stop All Services

```powershell
docker compose -f database/pgdb/docker-compose.yml down
docker compose -f database/pgadmin/docker-compose.yml down
docker compose -f database/redis/docker-compose.yml down
docker compose -f kafka/docker-compose.yml down
docker compose -f zipkin/docker-compose.yml down
```
