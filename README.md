# Fully Automated DevSecOps CI/CD Pipeline Using Jenkins

## Introduction
In today's fast-paced development environment, integrating security into the DevOps pipeline (DevSecOps) has become crucial. This ensures that security is a shared responsibility throughout the IT lifecycle. This document provides a detailed guide on setting up a fully automated DevSecOps CI/CD pipeline using Jenkins, Trivy, SonarQube, Docker, Prometheus, and Grafana.

## Pipeline Overview
Our pipeline consists of the following stages:
1. **Source Code Management**
2. **Build and Test**
3. **Security Scanning**
   - Source Code Analysis with SonarQube
   - Docker Image Scanning with Trivy and filesystem scan
4. **Docker Image Build and Push**
5. **Deployment to Containers**
6. **Monitoring and Logging**
   - Custom Grafana Dashboard for Pipeline Monitoring using Prometheus

## Tools and Technologies
- **Jenkins**: Automation server to orchestrate the CI/CD pipeline.
- **SonarQube**: Static code analysis tool to detect code quality issues.
- **Trivy**: Vulnerability scanner for Docker images and file systems.
- **Docker**: Containerization platform.
- **Prometheus**: Monitoring and alerting toolkit.
- **Grafana**: Analytics and monitoring platform.

## Steps
### Prepare the environment
We need first to prepare the tools for the environment. In my case i used docker compose to run 4 connected containers on the same custom network : **Jenkins container** , **Sonarqube container** , **prometheus container** and **grafana container**.
Using this stack we are able to integrate later these tools together.
**Note:** The jenkins container contains docker and trivy installed on it, because the pipeline is ran on that particular container where will be performed : build and push and scan etc...
### Setup Dockerfile
In our src code if we look at the env variables we notice a variable for an api key **TMDB_V3_API_KEY** then we need to get the key in order to communicate with the backend so we need to include it in the Dockerfile.
1. Open a web browser and navigate to [TMDB (The Movie Database)](https://www.themoviedb.org/) website.
2. Click on "Login" and create an account.
3. Once logged in, go to your profile and select "Settings."
4. Click on "API" from the left-side panel.
5. Create a new API key by clicking "Create" and accepting the terms and conditions.
6. Provide the required basic details and click "Submit."
7. You will receive your TMDB API key.
Now here is the Dockerfile i used:
```
# Stage 1: Build the application
FROM node:16.17.0-alpine as builder
WORKDIR /app
COPY ./package.json .
COPY ./yarn.lock .
RUN yarn install
COPY . .
ARG TMDB_V3_API_KEY
ENV VITE_APP_TMDB_V3_API_KEY=${TMDB_V3_API_KEY}
ENV VITE_APP_API_ENDPOINT_URL="https://api.themoviedb.org/3"
RUN yarn build

# Stage 2: Serve the application with Nginx
FROM nginx:stable-alpine
WORKDIR /usr/share/nginx/html
RUN rm -rf ./*
COPY --from=builder /app/dist .
EXPOSE 80
ENTRYPOINT ["nginx", "-g", "daemon off;"]

