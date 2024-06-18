# Fully Automated DevSecOps CI/CD Pipeline Using Jenkins

**Note:** This project i took from **cloud champ** youtube channel and i used it for learning purposes and changed certain things in the configuration as fit for my local work environment, you can find his channel below.
**https://www.youtube.com/@cloudchamp?**

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
ENTRYPOINT ["nginx", "-g", "daemon of;"]
````

Now build it using:
```
docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .
```
we can test the container now on port 80.
### CI/CD pipeline
Now we already have the trivy cli installed on jenkins container **(mentionned in /jenkins/Dockerfile : view in the repository)**
Start the docker compose stack and access Jenkins in a web browser using localhost.
localhost:8080
```
docker-compose up -d
```
Now we should have four containers deployed in order to be used and integrated later in the jenkins pipeline.
Goto Manage Jenkins →Plugins → Available Plugins →

Install below plugins

1. Eclipse Temurin Installer (Install without restart)
2. SonarQube Scanner (Install without restart)
3. NodeJs Plugin (Install Without restart)
4. Email Extension Plugin

Configure Java and Nodejs in Global Tool Configuration
Goto Manage Jenkins → Tools → Install JDK(17) and NodeJs(16)→ Click on Apply and Save

**SonarQube**
Create the token
Goto Jenkins Dashboard → Manage Jenkins → Credentials → Add Secret Text. It should look like this
After adding sonar token
Click on Apply and Save
The Configure System option is used in Jenkins to configure different server
Global Tool Configuration is used to configure different tools that we install using Plugins
We will install a sonar scanner in the tools.
Create a Jenkins webhook in the sonarqube web app 
**Configure the ci/cd pipeline**
````
pipeline{
    agent any
    tools{
        jdk 'jdk17'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
        NODEJS_HOME=tool 'node16'
        PATH = "${NODEJS_HOME}/bin:${env.PATH}"
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/Achref-dot-afk/DevSecOps-Project.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
         stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
  
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=YOUR_KEY -t netflix ."
                       sh "docker tag netflix YOUR_TAG/netflix:latest "
                       sh "docker push YOUR_TAG/netflix:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image YOUR_TAG/netflix:latest > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d -p 8081:80 YOUR_TAG/netflix:latest'
            }
        }
    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'YOUR_EMAIL',                           
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
````
**IMPORTANT NOTE**
In my case i deployed the tools on docker containers so any url i will include in the configuration(webhook in sonarqube,sonarqube server setup in jenkins system section,...) will use the service(container) name not localhost because containers talk to each other using their service name on the same network.

**Install Docker Tools and Docker Plugins:**

Go to "Dashboard" in your Jenkins web interface.
Navigate to "Manage Jenkins" → "Manage Plugins."
Click on the "Available" tab and search for "Docker."
Check the following Docker-related plugins:
Docker
Docker Commons
Docker Pipeline
Docker API
docker-build-step
Click on the "Install without restart" button to install these plugins.
Add DockerHub Credentials:

**To securely handle DockerHub credentials in your Jenkins pipeline, follow these steps:**
Go to "Dashboard" → "Manage Jenkins" → "Manage Credentials."
Click on "System" and then "Global credentials (unrestricted)."
Click on "Add Credentials" on the left side.
Choose "Secret text" as the kind of credentials.
Enter your DockerHub credentials (Username and Password) and give the credentials an ID (e.g., "docker").
Click "OK" to save your DockerHub credentials.
### Configure monitoring
We already have the prometheus container running on port **9090** and we can access it through localhost, but we need first setup jenkins to collect metrics , so we use prometheus metrics plugin and install it first.
Then we can see metrics are collected in understandable format for prometheus and they are exposed on localhost:8080/prometheus **(we can change it)**
Now we should add the new target in the prometheus config file.
```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'CI/CD pipeline'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ["customjenkins:8080"]
```
Remember adding the service name in the url not localhost because it is containers communication.
Now we should be able to see the new target in prometheus interface.

### Setup visualization using grafana
Add Prometheus Data Source:

To visualize metrics, you need to add a data source. Follow these steps:

Click on the gear icon (⚙️) in the left sidebar to open the "Configuration" menu.

Select "Data Sources."

Click on the "Add data source" button.

Choose "Prometheus" as the data source type.

In the "HTTP" section:

Set the "URL" to http://prometheus:9090 (assuming Prometheus is the container's name).
Click the "Save & Test" button to ensure the data source is working.
**Import a Dashboard:**

To make it easier to view metrics, you can import a pre-configured dashboard. Follow these steps:

Click on the "+" (plus) icon in the left sidebar to open the "Create" menu.

Select "Dashboard."

Click on the "Import" dashboard option.

Enter the dashboard code you want to import (e.g., code 1860).

Click the "Load" button.

Select the data source you added (Prometheus) from the dropdown.

Click on the "Import" button.

You should now have a Grafana dashboard set up to visualize metrics from Prometheus.

Grafana is a powerful tool for creating visualizations and dashboards, and you can further customize it to suit your specific monitoring needs.

That's it! You've successfully installed and set up Grafana to work with Prometheus for monitoring and visualization.
