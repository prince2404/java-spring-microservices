# Java Spring Microservices

A microservices project built with Java and Spring Boot to learn how real backend systems are designed. Patient Service is only a small example service, the focus is on how services talk to each other and run together.

## Services

- **API Gateway**: single entry point, routes requests and checks the JWT token
- **Auth Service**: login and JWT creation/validation
- **Patient Service**: a small CRUD service with its own Postgres database
- **Billing Service**: creates a billing account for a new patient (called by Patient Service using gRPC)
- **Analytics Service**: reads patient events from Kafka

```
Client -> API Gateway -> Auth Service
                      -> Patient Service -> Billing Service (gRPC)
                                         -> Kafka -> Analytics Service
```

## Tech

Java, Spring Boot, Spring Security, JWT, PostgreSQL, gRPC, Kafka, Docker Compose, JUnit, REST Assured, AWS CDK

## Run locally

```
docker compose up --build
```

Test with Postman through the API Gateway:
- `POST /auth/login` to get a token
- `GET /api/patients` with header `Authorization: Bearer <token>`

Locally, Postgres containers replace RDS and a single Kafka container replaces MSK.

## AWS

The AWS CDK code is in the `infrastructure` folder. LocalStack's free plan has no RDS, so that deployment failed. I am extending it for real AWS (ECR, RDS, ECS Fargate, ALB) and will deploy later when I have credits, with a budget alert on.

## What I learned

- REST vs gRPC vs Kafka, and when to use each
- API Gateway and JWT authentication
- Running multiple services with Docker Compose
- Unit and integration testing
- AWS basics (VPC, ECS, RDS, ALB) and Infrastructure as Code