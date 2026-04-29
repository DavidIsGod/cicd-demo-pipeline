
# CICD Demo

This repository contains a small Spring Boot application adapted for a local Jenkins CI/CD assignment.

## Application

- Java Spring Boot application
- Maven build
- Docker image for local deployment

## Pipeline

The Jenkins pipeline reads `Jenkinsfile` from SCM and runs these stages:

1. Checkout
2. Test
3. Build
4. Docker Build
5. Static Analysis with SonarQube
6. Quality Gate and Security Hotspot validation
7. Container Security Scan with Trivy
8. Local Deploy

## Test scope

The pipeline runs the JUnit categories for:

- `UnitTest`
- `IntegrationTest`

It excludes the Selenium-based `SystemTest`, because the assignment only requires basic validation before deployment.

## Deployment

On `main` or `master`, Jenkins deploys the image locally with Docker and exposes the app on port `8081`.
