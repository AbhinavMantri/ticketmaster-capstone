# AWS CI/CD Deployment Frame

This workspace contains five deployable Spring Boot microservices:

- `user-service`
- `event-management-service`
- `seats-allocation-service`
- `booking-service`
- `payment-service`

Each service now owns the same AWS deployment contract:

- `buildspec.yml`: AWS CodeBuild build and deploy script.
- `Dockerfile`: Runtime image for the Spring Boot jar.
- `.dockerignore`: Keeps the Docker context focused.
- `docs/aws-codebuild.md`: Service-specific CodeBuild variables and flow.

## Target AWS flow

1. Source change lands in the service repository.
2. CodeBuild runs that service's `buildspec.yml`.
3. Maven verifies the service with `./mvnw -B clean verify`.
4. Docker builds the service image.
5. CodeBuild pushes the immutable image tag and `latest` to Amazon ECR.
6. CodeBuild emits `imagedefinitions.json` for an ECS deploy action.
7. ECS deploys the new task definition revision through CodePipeline, or CodeBuild directly forces a new ECS deployment when `DEPLOY_TO_ECS=true`.

## Recommended AWS resources

Use one ECR repository and one ECS service per microservice:

| Service | ECR repository | ECS container name |
| --- | --- | --- |
| `user-service` | `ticketmaster/user-service` | `user-service` |
| `event-management-service` | `ticketmaster/event-management-service` | `event-management-service` |
| `seats-allocation-service` | `ticketmaster/seats-allocation-service` | `seats-allocation-service` |
| `booking-service` | `ticketmaster/booking-service` | `booking-service` |
| `payment-service` | `ticketmaster/payment-service` | `payment-service` |

## Required CodeBuild environment variables

- `AWS_DEFAULT_REGION`: AWS region, for example `ap-south-1`.
- `ECR_REPOSITORY`: The service's ECR repository name.

## Optional CodeBuild environment variables

- `AWS_ACCOUNT_ID`: If omitted, CodeBuild resolves it with STS.
- `IMAGE_TAG`: If omitted, the build uses `<service-name>-<short-commit-sha>`.
- `CONTAINER_NAME`: ECS task definition container name. Defaults to the service name.
- `DEPLOY_TO_ECS`: Defaults to `false`. Set to `true` for direct ECS rollout from CodeBuild.
- `ECS_CLUSTER`: Required only when `DEPLOY_TO_ECS=true`.
- `ECS_SERVICE`: Required only when `DEPLOY_TO_ECS=true`.

## CodeBuild role permissions

The CodeBuild service role needs permissions for:

- `sts:GetCallerIdentity`
- `ecr:DescribeRepositories`
- `ecr:CreateRepository`
- `ecr:GetAuthorizationToken`
- `ecr:BatchCheckLayerAvailability`
- `ecr:InitiateLayerUpload`
- `ecr:UploadLayerPart`
- `ecr:CompleteLayerUpload`
- `ecr:PutImage`
- `ecs:UpdateService` only if `DEPLOY_TO_ECS=true`

The CodeBuild project must run with privileged mode enabled so Docker image builds can run.
