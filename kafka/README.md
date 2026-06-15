# Kafka Setup

This setup creates an Apache Kafka broker (KRaft mode, no ZooKeeper) and a Kafka UI management interface using Docker Compose.

## Prerequisites

- Docker and Docker Compose installed
- Port 9092 available (Kafka broker)
- Port 9000 available (Kafka UI)

## Quick Start

### 1. Create the Docker Network

```powershell
docker network create kafka-network
```

### 2. Start Kafka + Kafka UI

```powershell
docker compose -f kafka/docker-compose.yml up -d
```

## Access Services

- **Kafka broker (from host):** `localhost:9092`
- **Kafka broker (from other containers):** `kafka:29092`
- **Kafka UI:** `http://localhost:9000`

## Stop Services

```powershell
docker compose -f kafka/docker-compose.yml down
```

## Cleanup

To remove everything including data:

```powershell
docker compose -f kafka/docker-compose.yml down -v
docker network rm kafka-network
```

## Troubleshooting

**Container keeps restarting with `advertised.listeners cannot use 0.0.0.0` error**

The volume may have stale storage formatted with bad config. Wipe the volume and restart:

```powershell
docker compose -f kafka/docker-compose.yml down -v
docker compose -f kafka/docker-compose.yml up -d
```

**Kafka UI shows broker unreachable**

Kafka UI connects to the broker via the internal listener `kafka:29092`. Make sure both containers are on the same `kafka-network`:

```powershell
docker network inspect kafka-network
```

Both `kafka` and `kafka-ui` should appear in the `Containers` list.
