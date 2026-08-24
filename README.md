# Flask ECS CI/CD Pipeline

A containerized Flask application built with Docker, pushed to Amazon ECR, deployed on Amazon ECS (Fargate), monitored with CloudWatch, and automatically built/deployed through a Jenkins pipeline sourced from this repository.

## Architecture

**Flow:**
1. Code (Flask app, `Dockerfile`, `Jenkinsfile`) is pushed to this GitHub repository.
2. A Jenkins pipeline job pulls the latest code (`checkout scm`) and runs through five stages: build the Docker image, log in to ECR, tag the image, push it to ECR, and trigger an ECS deployment.
3. Amazon ECR stores the built Docker image as a private registry.
4. Amazon ECS (Fargate launch type — serverless, no EC2 instances to manage) runs the container as a Task inside a Service, which keeps the desired count of tasks alive at all times.
5. The running task is reachable publicly on port `5000`.
6. Container logs stream to a CloudWatch Log Group (`/ecs/flask-ecs-task`).
7. A CloudWatch Alarm watches `RunningTaskCount` on the ECS service and, if it drops to 0, publishes to an SNS topic that emails a notification.

## Tech stack

| Layer | Tool |
|---|---|
| Application | Python (Flask) |
| Containerization | Docker |
| Image registry | Amazon ECR |
| Orchestration | Amazon ECS (Fargate) |
| CI/CD | Jenkins (Pipeline as Code via `Jenkinsfile`) |
| Source control | GitHub |
| Monitoring | Amazon CloudWatch (Logs + Alarms), Amazon SNS |

## Repository contents

```
.
├── app.py            # Flask application
├── Dockerfile         # Container image definition
├── requirements.txt   # Python dependencies
├── Jenkinsfile        # CI/CD pipeline definition
├── architecture.svg   # Architecture diagram
└── README.md
```

## How the pipeline works

The `Jenkinsfile` defines a declarative pipeline with the following stages:

1. **Checkout** — pulls the latest commit from this repository's `main` branch.
2. **Build Docker Image** — runs `docker build` to produce a local image tagged `flask-ecs-app:latest`.
3. **Login to ECR** — authenticates Docker to the Amazon ECR registry using AWS credentials stored securely in Jenkins' credential store.
4. **Tag Image** — tags the local image with the full ECR repository URI.
5. **Push to ECR** — uploads the image to Amazon ECR.
6. **Deploy to ECS** — runs `aws ecs update-service --force-new-deployment`, which tells the ECS service to replace its running task with a fresh one pulling the latest pushed image.

AWS credentials are never hardcoded in the script. They're stored as Jenkins "Secret text" credentials (`aws-access-key-id`, `aws-secret-access-key`) under a least-privilege IAM user, and injected as environment variables only at pipeline runtime.

## Running locally

```bash
docker build -t flask-ecs-app .
docker run -p 5000:5000 flask-ecs-app
```

Then visit `http://localhost:5000`.

## AWS setup summary

- **ECR repository**: `flask-ecs-app` (region: `ap-south-1`)
- **ECS cluster**: `flask-ecs-cluster` (Fargate)
- **ECS service**: `flask-ecs-service`
- **Task definition**: `flask-ecs-task` — 0.5 vCPU / 1 GB memory, container port `5000`
- **IAM**: a scoped `docker-ecs-user` with `AmazonEC2ContainerRegistryFullAccess`, `AmazonECS_FullAccess`, and `CloudWatchLogsFullAccess` — no admin-level access
- **Monitoring**: CloudWatch Log Group `/ecs/flask-ecs-task`; CloudWatch Alarm `flask-app-down-alarm` triggers an SNS email notification if the running task count drops to 0

## Notes

This project was built as a hands-on portfolio exercise to demonstrate containerization, cloud deployment, CI/CD automation, and monitoring using genuinely provisioned AWS resources rather than local-only simulation.
