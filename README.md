# DataBridge deploy

Local Docker Compose stack and Helm deployment for the DataBridge frontend, client portal service, analytics service, PostgreSQL, and persistent MinIO object storage. The local frontend is presented through one Nginx gateway so clients and analysts use the same public origin.

## Run the full application locally

```sh
cp .env.example .env
docker compose up --build
```

- Portal selection: http://localhost:5173
- Client portal UI: http://localhost:5173/login (client registration: `/signup`)
- Analytics UI: http://localhost:5173/analytics
- Client portal API: http://localhost:8081/docs
- Analytics API: http://localhost:8082/docs
- Local S3-compatible storage endpoint: http://localhost:9090

The gateway owns port 5173. The client and analytics Vite servers are internal Compose services; do not use their implementation ports as user-facing URLs.

Compose mounts the client portal service migration into PostgreSQL initialization for a fresh local database. PostgreSQL and local object data persist in named Docker volumes. MinIO is a local development substitute for AWS S3; production should use private AWS S3.

Client users can register at the client portal's `/signup` page. No demo accounts are created. To provision a local analyst or administrator, run this from the deployment directory after Compose is up; the command prompts for a password without placing it in shell history:

```sh
docker compose exec client-portal-service \
  python scripts/create_user.py \
  --email analyst@your-org.example \
  --display-name "Data Analyst" \
  --role analyst
```

The local JWT secret in `.env.example` is for development only. In each test/production namespace, create the `databridge-services` Kubernetes Secret with a randomly generated `jwt-secret` of at least 32 characters. Both APIs read that same secret to validate signed bearer tokens and check the corresponding active auth session in PostgreSQL. Do not put secret values in Helm values or source control.

For browser uploads to AWS S3, configure CORS on the private upload bucket to allow `PUT` from that environment's client portal origin and the `Content-Type` request header. The API returns short-lived, object-scoped presigned URLs; do not enable public bucket access or expose AWS credentials to the browser. The client service's IRSA role also needs `s3:ListBucket` on the upload bucket, constrained by `s3:prefix` to `submissions/*`, so analyst file-list requests can recover older uploaded objects that have no database file record.

## Deploy to EKS

```sh
helm upgrade --install databridge-dev . -n databridge-dev -f values-dev.yaml --create-namespace
helm upgrade --install databridge-test . -n databridge-test -f values-test.yaml --create-namespace
helm upgrade --install databridge-prod . -n databridge-prod -f values-prod.yaml --create-namespace
```

The client portal service uses an IRSA-backed Kubernetes service account for AWS S3 access in EKS. The client portal and analytics services run independently and share the configured PostgreSQL endpoint in the initial deployment topology. Database and JWT secrets must be created in each namespace; do not commit them to values files.
