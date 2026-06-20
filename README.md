# Coworking Space Analytics Service

## Overview
This project deploys a Flask-based analytics API for the Coworking Space Service onto AWS EKS using a CI/CD pipeline built with CodeBuild and ECR. The API exposes endpoints for daily usage and user visit reports, backed by a PostgreSQL database running as a Kubernetes deployment.

## Architecture
The application follows a microservices pattern: the analytics service and PostgreSQL database run as separate Kubernetes deployments within the same cluster. A ConfigMap holds plaintext environment variables (DB host, port, name, username), while a Secret stores the base64-encoded database password. The application service is exposed via a Kubernetes LoadBalancer, and the database is accessible only within the cluster through a ClusterIP service.

## Build & Deploy Pipeline
Pushing to the `main` branch triggers an AWS CodeBuild pipeline defined in `buildspec.yaml`, which builds the Docker image from `analytics/Dockerfile`, tags it using semantic versioning (`1.0.<BUILD_NUMBER>`), and pushes it to Amazon ECR. To deploy a new build, update the image tag in `deployment/coworking.yaml` and run `kubectl apply -f deployment/coworking.yaml`.

## Deploying Changes
```bash
# Apply config changes
kubectl apply -f deployment/configmap.yaml
kubectl apply -f deployment/secret.yaml
kubectl apply -f deployment/coworking.yaml

# Verify
kubectl get pods
kubectl get svc coworking
```

## Monitoring
AWS CloudWatch Container Insights is enabled via the `amazon-cloudwatch-observability` EKS add-on. Fluent Bit ships container logs to CloudWatch Log Groups under `/aws/containerinsights/sheu-cluster/`, and the CloudWatch agent collects pod-level CPU and memory metrics.

## Recommended Instance Type
`t3.medium` is well-suited for this workload: it provides 2 vCPUs and 4 GiB RAM, which comfortably runs both the analytics service and PostgreSQL with headroom, while remaining cost-efficient for a low-traffic analytics API.

## Cost Optimisation
Using a single-node EKS cluster with a `t3.medium` spot instance for non-production workloads can reduce compute costs by up to 70%. Enabling EKS Fargate for the analytics pod eliminates idle EC2 costs entirely, as you only pay for vCPU and memory consumed per second. Storing infrequently accessed CloudWatch logs in S3 via log expiry policies further reduces ongoing storage costs.
