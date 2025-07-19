# Quick-Start Guide to Deploying a Spring Boot Backend on AWS EC2 with Docker and GitHub Actions CI/CD

This guide simplifies the process of launching an AWS EC2 instance with Amazon Linux 2023, connecting via SSH, dockerizing a Spring Boot backend, testing it locally with Docker, setting up GitHub secrets, and creating a CI/CD pipeline using GitHub Actions to deploy directly to EC2. The focus is on simplicity, using basic Docker commands and avoiding complex tools like Amazon ECR or Docker Compose.

## Prerequisites

- An active AWS account with IAM permissions to manage EC2.
- A GitHub account for hosting the repository and CI/CD pipeline.
- A local machine with Docker, Java 17, Maven, and AWS CLI installed.
- A basic Spring Boot project (or use the example provided).
- Basic knowledge of Git, Docker, and Spring Boot.

## Step 1: Launch an AWS EC2 Instance with Amazon Linux 2023

### 1.1 Log in to AWS Management Console

1. Go to the [AWS Management Console](https://aws.amazon.com/console/).
2. Sign in with your AWS credentials.

### 1.2 Create an EC2 Instance

1. **Navigate to EC2**:
   - Search for "EC2" in the AWS Console and select the EC2 service.
   - Click **Launch Instance** > **Launch Instance**.

2. **Select Amazon Linux 2023 AMI**:
   - Choose **Amazon Linux 2023 AMI** (ensure 64-bit x86 architecture).
   - Click **Select**.

3. **Choose Instance Type**:
   - Pick **t2.micro** (free tier eligible for cost-effective testing).
   - Click **Next: Configure Instance Details**.

4. **Configure Instance Details**:
   - Keep defaults for VPC, Subnet, etc.
   - Enable **Auto-assign Public IP** to allow external access.

5. **Add Storage**:
   - Use the default 8 GiB gp3 (general-purpose SSD) volume.

6. **Add Tags**:
   - Add a tag, e.g., Key: `Name`, Value: `SpringBoot-EC2`.

7. **Configure Security Group**:
   - Create a new security group or select an existing one.
   - Add these inbound rules:
     - **SSH**: Type: SSH, Protocol: TCP, Port: 22, Source: Your IP (or 0.0.0.0/0 for testing; restrict in production).
     - **HTTP**: Type: HTTP, Protocol: TCP, Port: 80, Source: 0.0.0.0/0.
     - **Custom TCP**: Protocol: TCP, Port: 8080 (for Spring Boot), Source: 0.0.0.0/0.

8. **Review and Launch**:
   - Click **Review and Launch**, then **Launch**.
   - Create or select a key pair (e.g., `my-ec2-key.pem`).
   - Download the key pair and store it securely (e.g., `~/.ssh/my-ec2-key.pem`).
   - Click **Launch Instances**.

9. **Get Public IP**:
   - Once the instance is running, note its **Public IPv4 address** from the EC2 dashboard.

## Step 2: Connect to EC2 Instance via SSH

1. **Set Key Pair Permissions**:
   - On your local machine, secure the key file:
     ```bash
     chmod 400 ~/.ssh/my-ec2-key.pem
     ```

2. **SSH into EC2**:
   - Use the public IP from the EC2 dashboard:
     ```bash
     ssh -i ~/.ssh/my-ec2-key.pem ec2-user@<public-ip>
     ```
   - If successful, you'll be logged into the EC2 instance.

3. **Install Docker on EC2**:
   - Update the instance and install Docker:
     ```bash
     sudo dnf update -y
     sudo dnf install docker -y
     sudo systemctl start docker
     sudo systemctl enable docker
     sudo usermod -a -G docker ec2-user
     ```
   - Log out and back in to apply group changes:
     ```bash
     exit
     ssh -i ~/.ssh/my-ec2-key.pem ec2-user@<public-ip>
     ```
   - Verify Docker:
     ```bash
     docker --version
     ```

## Step 3: Create and Dockerize a Spring Boot Application

### 3.1 Create a Simple Spring Boot Application

If you have a Spring Boot project, skip to the Dockerization step. Otherwise, create a basic one:

1. **Generate a Project**:
   - Use [Spring Initializr](https://start.spring.io/) with:
     - **Group**: `com.example`
     - **Artifact**: `springboot-backend`
     - **Dependencies**: Spring Web
     - **Java Version**: 17
     - **Packaging**: JAR
   - Download and unzip the project locally.

2. **Add a REST Controller**:
   - Create `src/main/java/com/example/springbootbackend/controller/HelloController.java`:
     ```java
     package com.example.springbootbackend.controller;

     import org.springframework.web.bind.annotation.GetMapping;
     import org.springframework.web.bind.annotation.RestController;

     @RestController
     public class HelloController {
         @GetMapping("/hello")
         public String hello() {
             return "Hello from Spring Boot on EC2!";
         }
     }
     ```

3. **Build Locally**:
   - Navigate to the project directory:
     ```bash
     cd springboot-backend
     ./mvnw clean package
     ```
   - This creates `target/springboot-backend-0.0.1-SNAPSHOT.jar`.

### 3.2 Dockerize the Application

1. **Create a Dockerfile**:
   - In the project root, create `Dockerfile`:
     ```dockerfile
     FROM openjdk:17-jdk-slim
     WORKDIR /app
     COPY target/springboot-backend-0.0.1-SNAPSHOT.jar app.jar
     EXPOSE 8080
     ENTRYPOINT ["java", "-jar", "app.jar"]
     ```

2. **Build the Docker Image**:
   - Run:
     ```bash
     docker build -t springboot-backend:latest .
     ```

3. **Test Locally with Docker (Optional)**:
   - Run the container locally:
     ```bash
     docker run -d -p 8080:8080 --name springboot-backend springboot-backend:latest
     ```
   - Test the endpoint:
     ```bash
     curl http://localhost:8080/hello
     ```
     Expected output: `Hello from Spring Boot on EC2!`
   - Stop and remove the container:
     ```bash
     docker stop springboot-backend
     docker rm springboot-backend
     ```

## Step 4: Set Up GitHub Secrets for EC2 Deployment

### 4.1 Generate an SSH Key Pair

1. **Create an SSH Key Pair**:
   - On your local machine:
     ```bash
     ssh-keygen -t rsa -b 4096 -f ~/.ssh/deploy-key -N ""
     ```
   - This creates `deploy-key` (private key) and `deploy-key.pub` (public key).

2. **Add Public Key to EC2**:
   - Copy the public key to the EC2 instance:
     ```bash
     ssh-copy-id -i ~/.ssh/deploy-key.pub ec2-user@<public-ip>
     ```
   - If `ssh-copy-id` is unavailable, manually append `deploy-key.pub` content to `~/.ssh/authorized_keys` on EC2:
     ```bash
     cat ~/.ssh/deploy-key.pub
     ```
     SSH into EC2, edit `~/.ssh/authorized_keys`, and paste the public key.

3. **Add Private Key to GitHub**:
   - Go to your GitHub repository > **Settings** > **Secrets and variables** > **Actions** > **New repository secret**.
   - Add:
     - **Name**: `EC2_SSH_KEY`
     - **Value**: Copy the contents of `~/.ssh/deploy-key` (private key).
   - Add another secret:
     - **Name**: `EC2_HOST`
     - **Value**: Your EC2 instance’s public IP (e.g., `3.123.45.67`).

## Step 5: Create a GitHub Actions CI/CD Pipeline

### 5.1 Push Code to GitHub

1. **Create a GitHub Repository**:
   - Create a new repository on GitHub (e.g., `springboot-backend`).
   - Initialize and push your project:
     ```bash
     cd springboot-backend
     git init
     git add .
     git commit -m "Initial commit"
     git branch -M main
     git remote add origin <repository-url>
     git push -u origin main
     ```

### 5.2 Create a GitHub Actions Workflow

1. **Create Workflow File**:
   - In your repository, create `.github/workflows/deploy.yml`:
     ```yaml
     name: Deploy to EC2

     on:
       push:
         branches:
           - main

     jobs:
       build-and-deploy:
         runs-on: ubuntu-latest

         steps:
         - name: Checkout code
           uses: actions/checkout@v3

         - name: Set up JDK 17
           uses: actions/setup-java@v3
           with:
             java-version: '17'
             distribution: 'corretto'

         - name: Build with Maven
           run: mvn clean package

         - name: Build Docker image
           run: docker build -t springboot-backend:latest .

         - name: Deploy to EC2
           env:
             EC2_HOST: ${{ secrets.EC2_HOST }}
             EC2_USER: ec2-user
           run: |
             echo "${{ secrets.EC2_SSH_KEY }}" > deploy-key
             chmod 600 deploy-key
             ssh -o StrictHostKeyChecking=no -i deploy-key $EC2_USER@$EC2_HOST << 'EOF'
               docker stop springboot-backend || true
               docker rm springboot-backend || true
               docker rmi springboot-backend:latest || true
             EOF
             scp -i deploy-key -r Dockerfile $EC2_USER@$EC2_HOST:/home/ec2-user/
             scp -i deploy-key target/springboot-backend-0.0.1-SNAPSHOT.jar $EC2_USER@$EC2_HOST:/home/ec2-user/
             ssh -o StrictHostKeyChecking=no -i deploy-key $EC2_USER@$EC2_HOST << 'EOF'
               docker build -t springboot-backend:latest .
               docker run -d --name springboot-backend -p 8080:8080 springboot-backend:latest
             EOF
     ```

2. **Explanation**:
   - The workflow triggers on pushes to the `main` branch.
   - It builds the Spring Boot JAR, creates a Docker image, and uses SCP/SSH to transfer the Dockerfile and JAR to EC2.
   - On EC2, it stops/removes old containers/images, builds the new image, and runs the container.

### 5.3 Test the CI/CD Pipeline

1. **Push Changes**:
   - Commit and push the workflow file:
     ```bash
     git add .github/workflows/deploy.yml
     git commit -m "Add GitHub Actions CI/CD"
     git push origin main
     ```

2. **Monitor the Pipeline**:
   - Go to your GitHub repository > **Actions** tab.
   - Check the workflow run for errors or success.

3. **Verify Deployment**:
   - Once the pipeline completes, access:
     ```bash
     curl http://<ec2-public-ip>:8080/hello
     ```
     Expected output: `Hello from Spring Boot on EC2!`

## Step 6: Troubleshooting Tips

- **SSH Connection Fails**: Ensure the EC2 security group allows port 22 and the SSH key is correctly set up.
- **Docker Not Running**: Verify Docker is running on EC2 (`sudo systemctl status docker`).
- **Pipeline Fails**: Check GitHub Actions logs for issues with SSH keys or file transfers.
- **Port 8080 Inaccessible**: Confirm the security group allows inbound traffic on port 8080.
- **Application Errors**: View container logs on EC2:
  ```bash
  docker logs springboot-backend
  ```

## Conclusion

You’ve successfully set up an AWS EC2 instance with Amazon Linux 2023, connected via SSH, dockerized a Spring Boot backend, tested it locally, and deployed it using a GitHub Actions CI/CD pipeline. This setup is straightforward, avoids external registries like ECR, and uses simple Docker commands for clarity. For production, consider adding a reverse proxy (e.g., Nginx), HTTPS, and monitoring with AWS CloudWatch.