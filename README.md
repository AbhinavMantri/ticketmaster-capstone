# Ticketmaster Capstone Project

This repository is the single documentation and infrastructure entry point for the Ticketmaster backend capstone project.

The backend is implemented as independently deployable Spring Boot microservices. Each service has its own repository, build script, Dockerfile, and AWS CodeBuild deployment notes.

## Microservice Repositories

| Service | Responsibility | Repository |
| --- | --- | --- |
| User Service | Registration, login, refresh tokens, user profile, admin user management, JWT issuance | https://github.com/AbhinavMantri/user-service |
| Event Management Service | Venue management, event creation, event publishing, pricing, inventory initialization events | https://github.com/AbhinavMantri/event-management-service |
| Seats Allocation Service | Seat inventory, locking, release, confirmation, Kafka inventory consumers | https://github.com/AbhinavMantri/seats-allocation-service |
| Booking Service | Checkout orchestration, booking lifecycle, ticket generation, payment outcome processing | https://github.com/AbhinavMantri/booking-service |
| Payment Service | Payment initiation, verification, webhooks, reconciliation, booking outcome callbacks | https://github.com/AbhinavMantri/payment-service |

## Included In This Repository

- `kafka/docker-compose.yml`: shared local Kafka infrastructure.
- `AWS_CICD_DEPLOYMENT.md`: AWS CodeBuild, ECR, and ECS deployment frame.
- `CAPSTONE_NOTES.md`: project notes and implementation summary.
- `Ticketmaster_Project_Report.md`: capstone project report.
- `Ticketmaster_Project_Report_Strict_Format.md`: report in strict submission format.

## Deployment Model

Each microservice repository contains:

- `Dockerfile`
- `buildspec.yml`
- `docs/aws-codebuild.md`

The intended AWS flow is:

1. CodeBuild verifies the service with Maven.
2. CodeBuild builds the Docker image.
3. CodeBuild pushes the image to Amazon ECR.
4. CodePipeline or CodeBuild deploys the image to the corresponding ECS service.

See `AWS_CICD_DEPLOYMENT.md` for the detailed deployment contract and required AWS variables.

## Local Infrastructure

Kafka can be started from the included `kafka` folder:

```bash
docker compose -f kafka/docker-compose.yml up -d
```

Each service README documents its own local configuration and API surface.
