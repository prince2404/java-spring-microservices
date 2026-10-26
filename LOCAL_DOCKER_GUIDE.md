# Run Patient Management Locally

## What this starts

`docker compose up -d --build` starts two PostgreSQL databases, one single-node Kafka broker in KRaft mode, and all five Spring Boot services. Only the API Gateway is published to the laptop at `http://localhost:4004`; every other service is reachable only from the Compose network.

The services use these internal addresses:

| Caller | Target |
| --- | --- |
| API Gateway | `auth-service:4005`, `patient-service:4000` |
| Patient service | `billing-service:9001` (gRPC), `kafka:9092` |
| Analytics service | `kafka:9092` |
| Auth service | `auth-service-db:5432` |
| Patient service | `patient-service-db:5432` |

## Start the stack

1. Install and start Docker Desktop.
2. The repository includes a local-only `.env` file. If it is missing, copy the template and replace its values:

   ```powershell
   Copy-Item .env.example .env
   ```

   `JWT_SECRET` must be Base64-encoded and decode to at least 32 bytes. Generate one in PowerShell with:

   ```powershell
   $bytes = New-Object byte[] 32
   [System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
   [Convert]::ToBase64String($bytes)
   ```

3. Build and start all services:

   ```powershell
   docker compose up -d --build
   ```

4. Wait for PostgreSQL and Kafka to become healthy, then confirm every application is running:

   ```powershell
   docker compose ps
   docker compose logs --tail=100 auth-service patient-service billing-service analytics-service api-gateway
   ```

   `auth-service-db`, `patient-service-db`, and `kafka` should show `healthy`. Each Java service should contain a `Started ...Application` log line.

## Test with Postman

1. Import `postman/patient-management.postman_collection.json`.
2. Import and select `postman/patient-management.local.postman_environment.json`.
3. Run requests in this order: **Login (saves JWT)**, **Validate saved JWT**, **Get patients**, **Create patient**, **Update saved patient**, then **Delete saved patient**.

The login request uses `testuser@test.com` / `password123`, which comes from `auth-service/src/main/resources/data.sql`. Its test script saves the token to the selected Postman environment as `jwt`; the create request saves the returned ID as `patientId`.

## Verify the full workflow

### JWT validation through the gateway

`GET {{baseUrl}}/api/patients` without an `Authorization` header must return `401`. After Login, the same request with `Bearer {{jwt}}` must return `200`. The protected patient route uses the gateway filter, which calls `auth-service:4005/validate` before it forwards the request.

### gRPC billing call

Create a patient, then inspect both sides of the call:

```powershell
docker compose logs --tail=100 patient-service billing-service
```

Look for `Received response from billing service via GRPC` in `patient-service` and `createBillingAccount request received` in `billing-service`.

### Kafka event received by analytics

The `patient` topic is auto-created by Kafka on its first producer/consumer use. After creating a patient, run:

```powershell
docker compose logs --tail=100 analytics-service
```

Look for `Received Patient Event` with the created patient's ID, name, and email.

### Database persistence

Use the internal `psql` client instead of exposing database ports:

```powershell
docker compose exec auth-service-db psql -U admin_user -d auth-service-db -c "SELECT email, role FROM users;"
docker compose exec patient-service-db psql -U admin_user -d patient-service-db -c "SELECT id, name, email FROM patient ORDER BY name;"
```

If a GUI client is required temporarily, add a `ports` mapping such as `"5433:5432"` under the appropriate database service, restart it with `docker compose up -d`, and remove the mapping afterwards. Keeping it absent is safer and matches the default Compose file.

## Useful commands

```powershell
docker compose ps
docker compose logs -f api-gateway
docker compose logs -f patient-service billing-service analytics-service
docker compose down
```

`docker compose down` retains the two named PostgreSQL volumes. Use `docker compose down -v` only when you intentionally want to delete all local database data and re-run the seed scripts.

## Troubleshooting

| Symptom | What to do |
| --- | --- |
| Port `4004` is already in use | Find the process with `Get-NetTCPConnection -LocalPort 4004`, stop it, or temporarily change the left side of `"4004:4004"` and update Postman's `baseUrl`. |
| A service exits before its database is ready | Run `docker compose ps` and `docker compose logs auth-service-db patient-service-db <service-name>`. The Compose health checks should prevent this; after the database is healthy, use `docker compose up -d <service-name>`. |
| Kafka connection refused | Run `docker compose logs kafka`, wait for its `healthy` state, and confirm service logs use `kafka:9092`, not `localhost:9092`. Then restart the affected service with `docker compose up -d patient-service analytics-service`. |
| Docker runs out of memory | In Docker Desktop, allocate at least 6 GB of memory, then rebuild with `docker compose up -d --build`. The five Java builds can need several GB while compiling. |
