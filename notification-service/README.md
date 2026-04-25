# Notification Service

Simple, secure notification microservice for Book-Corner.

## Scope
- HTTP-only integration (no message queue).
- Receives internal events from other services.
- Stores in-app notifications.
- Sends email notifications (SMTP sandbox-ready).

## API Endpoints
- `POST /api/events` (internal service JWT required)
- `GET /api/notifications/me` (user JWT required)
- `PATCH /api/notifications/:id/read` (user JWT required)
- `GET /health`

## Security Controls
- JWT authentication for internal and user routes.
- Input validation and sanitization with Zod.
- Rate limiting on event ingestion endpoint.
- Helmet, CORS restrictions, and payload size limit.
- Secrets via environment variables only.

## Local Run
1. Copy `.env.example` to `.env` and fill values.
2. Install dependencies:
   - `npm install`
3. Start service:
   - `npm run dev`

## Docker
Build:
- `docker build -t book-corner-notification-service .`

Run:
- `docker run --env-file .env -p 3004:3004 book-corner-notification-service`

## AWS ECS Deployment
This repository includes a GitHub Actions workflow at `../.github/workflows/deploy-ecs.yml` for building the Docker image, pushing it to Amazon ECR, and updating an ECS/Fargate service.

To make the service public, place the ECS service behind an internet-facing Application Load Balancer and attach the listener to port `3004` on the container.

Public access checklist:
- Create an internet-facing ALB in public subnets.
- Attach the ECS service to the ALB target group.
- Use `/health` as the target group health check path.
- Open inbound `80` and/or `443` on the ALB security group.
- Keep the task security group private except for traffic from the ALB.
- Set `ALLOWED_ORIGINS=*` if you want browser clients from any domain to call the API.

Required GitHub secrets:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_REGION`
- `ECR_REPOSITORY`
- `ECS_CLUSTER`
- `ECS_SERVICE`
- `CONTAINER_NAME`

Task definition template:
- `../.aws/task-definition.json`

Replace the placeholder IAM role ARNs and AWS account ID in the task definition before deploying.

After deployment, the public host URL is the ALB DNS name, for example:
- `http://your-alb-name-123456.ap-south-1.elb.amazonaws.com`

Verify the service with:
- `GET /health`

## Producer Integration Contract
Send event payload to `POST /api/events`:

```json
{
  "eventType": "USER_REGISTERED",
  "userId": "<user-id>",
  "email": "user@example.com",
  "title": "Optional title",
  "message": "Optional message",
  "channels": ["in-app", "email"],
  "metadata": {
    "source": "user-auth-service"
  }
}
```

Headers:
- `Authorization: Bearer <internal-service-jwt>`
- `x-correlation-id: <uuid-optional>`

## Suggested JWT Payloads
Internal service token payload:

```json
{
  "service": "user-auth-service",
  "scope": "internal"
}
```

User token payload:

```json
{
  "id": "<user-id>",
  "role": "customer"
}
```
