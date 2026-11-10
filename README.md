# DataBridge deploy

Local Docker Compose stack and Helm deployment for the DataBridge frontend, contributor service, analytics service, PostgreSQL, and S3Mock object storage.

## Run the full application locally

```sh
cp .env.example .env
docker compose up --build
```

- Contributor UI: http://localhost:5173
- Analytics UI: http://localhost:5174/analytics
- Contributor API: http://localhost:8081/docs
- Analytics API: http://localhost:8082/docs
- S3Mock endpoint: http://localhost:9090

Compose mounts the contributor service migration into PostgreSQL initialization for a fresh local database. PostgreSQL data persists in its named Docker volume; S3Mock is a local development/test substitute and should not be used to store durable data.

## Deploy to EKS

```sh
helm upgrade --install databridge-dev . -n databridge-dev -f values-dev.yaml --create-namespace
helm upgrade --install databridge-test . -n databridge-test -f values-test.yaml --create-namespace
helm upgrade --install databridge-prod . -n databridge-prod -f values-prod.yaml --create-namespace
```

The contributor service uses an IRSA-backed Kubernetes service account for AWS S3 access in EKS. The contributor and analytics services run independently and share the configured PostgreSQL endpoint in the initial deployment topology. Database and JWT secrets must be created in each namespace; do not commit them to values files.
