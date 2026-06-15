# Zipkin Setup

This setup runs Zipkin, a distributed tracing system, using Docker Compose. It collects and visualizes timing data (traces and spans) from your microservices to help diagnose latency issues.

## Prerequisites

- Docker and Docker Compose installed
- Port 9411 available (Zipkin API and UI)

## Quick Start

### 1. Create the Docker Network

```powershell
docker network create zipkin-network
```

### 2. Start Zipkin

```powershell
docker compose -f zipkin/docker-compose.yml up -d
```

## Access Services

- **Zipkin UI:** `http://localhost:9411`
- **Zipkin API (from other containers):** `http://zipkin:9411`

## Sending Traces from Other Services

Any service running on `zipkin-network` can send spans to Zipkin over HTTP:

- **Endpoint:** `http://zipkin:9411/api/v2/spans`

For Spring Boot with Micrometer/Zipkin, set:

```yaml
management:
  tracing:
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

For other frameworks, configure the Zipkin reporter/exporter to use `http://zipkin:9411` as the base URL.

> Note: `zipkin` resolves as a hostname only from containers attached to `zipkin-network`. Run `docker network connect zipkin-network <container_name>` to add an existing container, or declare `zipkin-network` as an external network in that service's own compose file.

## Stop Services

```powershell
docker compose -f zipkin/docker-compose.yml down
```

## Cleanup

To remove the container and the network:

```powershell
docker compose -f zipkin/docker-compose.yml down
docker network rm zipkin-network
```

## Troubleshooting

**Zipkin UI is blank / shows no traces**

Zipkin uses in-memory storage by default. Traces are lost when the container restarts. This is intentional for a dev environment. If you need persistence across restarts, configure Zipkin with an Elasticsearch or MySQL storage backend.

**Services cannot reach `http://zipkin:9411` from within Docker**

Both the calling service container and the Zipkin container must be on the same `zipkin-network`. Verify with:

```powershell
docker network inspect zipkin-network
```

The `zipkin` container should appear in the `Containers` list alongside any service trying to send traces.

**Port 9411 already in use**

Another process is using that port. Stop it first, or change the host-side mapping in `docker-compose.yml` to a free port (e.g., `"9412:9411"`).
