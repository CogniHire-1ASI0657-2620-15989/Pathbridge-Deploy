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

3. Download the RDS CA bundle to the deployment directory:

   ```bash
   mkdir -p certs
   curl -fsSLo certs/rds-global-bundle.pem https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
   chmod 644 certs/rds-global-bundle.pem
   ```

4. Replace every `replace-with-...` value. `RDS_PASSWORD`,
   `JWT_SECRET_KEY`, `INTERNAL_API_KEY`, Decolecta, Jooble and YouTube keys are required.
   `RDS_PASSWORD` must be URL-safe (letters, digits, underscores and hyphens only).
5. Set `CORS_ORIGINS` to the frontend's exact origin.
6. Validate and start:

   ```bash
   docker compose -f docker-compose.yml -f docker-compose.production.yml --env-file .env config --quiet
   docker compose -f docker-compose.yml -f docker-compose.production.yml --env-file .env up -d --build
   ```

7. Verify the service state and Gateway:

   ```bash
   docker compose -f docker-compose.yml -f docker-compose.production.yml ps
   curl -f http://127.0.0.1:8080/health
   ```

The initial public endpoint is port `80`. Configure TLS before handling real user
credentials over the public internet.

## Data and network boundaries

- Only `gateway` publishes a host port.
- PostgreSQL lives in a private Amazon RDS instance. Only the EC2 security group
  is allowed to connect to it on port 5432.
- Every service owns a distinct logical RDS database: `identity_db`,
  `job_discovery_db`, `gap_analysis_db`, `learning_pathway_db`, and
  `interview_coach_db`. Each service's migrations manage its own schema.
- RDS backups and migrations must be verified before a deployment that changes
  persistent data.

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
