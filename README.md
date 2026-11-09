# PathBridge deployment on one EC2 instance

This repository runs PathBridge on a single EC2 host using Docker Compose. It deliberately does **not** include RabbitMQ yet.

## Required sibling layout

The Compose build contexts use relative paths. Clone the repositories as siblings, exactly as follows:

```text
/opt/pathbridge/
├── Pathbridge-Deploy/
├── Api-Gateway/
├── Identity-Service/
├── Job-Discovery-Service/
├── Gap-Analysis-Service/
├── Learning-Pathway-Service/
└── Interview-Coach-Service/
```

`Learning-Path-Service` is not used by this deployment.

## First start

1. Clone or update each repository at the intended release commit.
2. In this repository, create the real secrets file:

   ```bash
   cp .env.example .env
   chmod 600 .env
   ```

3. Replace every `replace-with-...` value. `POSTGRES_PASSWORD`,
   `JWT_SECRET_KEY`, `INTERNAL_API_KEY`, Decolecta, Jooble and YouTube keys are required.
4. Set `CORS_ORIGINS` to the frontend's exact origin.
5. Validate and start:

   ```bash
   docker compose -f docker-compose.yml -f docker-compose.production.yml --env-file .env config --quiet
   docker compose -f docker-compose.yml -f docker-compose.production.yml --env-file .env up -d --build
   ```

6. Verify the service state and Gateway:

   ```bash
   docker compose -f docker-compose.yml -f docker-compose.production.yml ps
   curl -f http://127.0.0.1:8080/health
   ```

The initial public endpoint is port `8080`. Add Caddy or Nginx and HTTPS before exposing the system to end users.

## Data and network boundaries

- Only `gateway` publishes a host port.
- All PostgreSQL instances live solely on the internal Docker network.
- Named volumes persist the five PostgreSQL databases across container recreation.
- `docker compose down -v` deletes all database data; do not use it on the server.
- Back up PostgreSQL before deployments that contain migrations.

## Deploying one service change

From `Pathbridge-Deploy`, after updating the corresponding sibling repository:

```bash
docker compose -f docker-compose.yml -f docker-compose.production.yml --env-file .env build gap-analysis-service
docker compose -f docker-compose.yml -f docker-compose.production.yml --env-file .env up -d --no-deps gap-analysis-service
docker compose -f docker-compose.yml -f docker-compose.production.yml logs --tail=100 gap-analysis-service
```

Replace `gap-analysis-service` with the affected service name. If the change modifies a public route, deploy the corresponding Gateway change too.

## Future RabbitMQ integration

RabbitMQ must be added as an internal service with a persistent volume. Do not publish ports `5672` or `15672` publicly. Background consumers should be separate worker containers and should process durable, idempotent messages.
