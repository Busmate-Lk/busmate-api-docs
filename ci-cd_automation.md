## A complete guide to build a fully automated CI/CD pipeline using GitHub Actions to deploy multiple microservices (from separate GitHub repositories) to a single AWS EC2 instance using Docker containers

---

Here's a **revised and complete step-by-step guide** to build a **fully automated CI/CD pipeline using GitHub Actions** to deploy **multiple Spring Boot microservices** (each in **separate GitHub repositories**) to a **single AWS EC2 instance using Docker containers**.

---

## 🧭 Overview

* 🛠️ Stack: Spring Boot (microservices), Docker, GitHub Actions, Amazon EC2 (Amazon Linux 2023)
* 🧳 Goal: On every push to main branch, auto-build Docker image, copy it to EC2, and deploy.
* 📦 One Docker container per microservice
* 🔐 Secure SSH via secrets

---

## 🧱 Assumptions

* You have:

  * Two GitHub repos: `service-a`, `service-b`
  * A single EC2 instance (Amazon Linux 2023)
  * Docker installed on EC2
  * Ports open in EC2 Security Group (e.g., 8080, 8081, etc.)

---

## ✅ Step 1: Prepare EC2 instance

### 🔧 1.1 Install Docker

```bash
sudo yum update -y
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```

Log out and log back in or run:

```bash
newgrp docker
```

---

### 🔐 1.2 Generate SSH Key for GitHub Actions

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/github-actions -C "github-actions"
```

* Leave passphrase **empty** or set one (you’ll store it later in GitHub Secrets).
* This creates:

  * Private key: `~/.ssh/github-actions`
  * Public key: `~/.ssh/github-actions.pub`

### 🗝️ 1.3 Add public key to `authorized_keys`

```bash
cat ~/.ssh/github-actions.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

---

## 🔐 Step 2: Add GitHub Secrets to Each Repo

In each GitHub repo (`Settings > Secrets and variables > Actions > Repository secrets`), add the following:

| Name             | Value                                             |
| ---------------- | ------------------------------------------------- |
| `EC2_HOST`       | `<your-ec2-public-ip>`                            |
| `EC2_USER`       | `ec2-user`                                        |
| `EC2_SSH_KEY`    | Contents of `~/.ssh/github-actions` (private key) |
| `SSH_PASSPHRASE` | *Only if you set one during keygen (optional)*    |



### What to Use for `EC2_SSH_KEY`?

You need to **copy the contents** of the **private key file**, like this:

```bash
cat ~/.ssh/github-actions
```

This will output something like:

```
-----BEGIN OPENSSH PRIVATE KEY-----
MIIJ...
...rest of the private key...
...+1w==
-----END OPENSSH PRIVATE KEY-----
```

> ⚠️ **Do not share this content publicly.**

### What to Use for `SSH_PASSPHRASE`? (optional)

You need to give passphrase that you given when creating the **private key file**.

---


## 🧾 Step 3: Create Dockerfile in each repo

Each Spring Boot microservice must have a `Dockerfile` like this:

```Dockerfile
FROM openjdk:21-jdk
WORKDIR /app
COPY target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Ensure your jar is built as `target/*.jar` during `mvn package`.

---

## ⚙️ Step 4: Create `.github/workflows/deploy.yml` in both repos

```yaml
name: Deploy to EC2

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Set up JDK
      uses: actions/setup-java@v3
      with:
        java-version: '21'
        distribution: 'temurin'

    - name: Build Spring Boot App
      run: |
        ./mvnw clean package -DskipTests

    - name: Copy files to EC2
      uses: appleboy/scp-action@v0.1.6
      with:
        host: ${{ secrets.EC2_HOST }}
        username: ${{ secrets.EC2_USER }}
        key: ${{ secrets.EC2_SSH_KEY }}
        passphrase: ${{ secrets.SSH_PASSPHRASE || '' }}
        source: "Dockerfile,target/*.jar"
        target: "~/app-${{ github.repository_owner }}-${{ github.event.repository.name }}"

    - name: Deploy via SSH
      uses: appleboy/ssh-action@v1.0.3
      with:
        host: ${{ secrets.EC2_HOST }}
        username: ${{ secrets.EC2_USER }}
        key: ${{ secrets.EC2_SSH_KEY }}
        passphrase: ${{ secrets.SSH_PASSPHRASE || '' }}
        script: |
          cd ~/app-${{ github.repository_owner }}-${{ github.event.repository.name }}
          docker stop ${GITHUB_REPOSITORY##*/} || true
          docker rm ${GITHUB_REPOSITORY##*/} || true
          docker build -t ${GITHUB_REPOSITORY##*/}-image .
          docker run -d --name ${GITHUB_REPOSITORY##*/} -p <PORT>:8080 ${GITHUB_REPOSITORY##*/}-image
```

### 🔄 Replace `<PORT>` above:

* For `service-a`: `-p 8080:8080`
* For `service-b`: `-p 8081:8080`

You can parameterize this using environment variables or just hard-code per repo.

---

## 📦 Step 5: Test Deployment

1. Push a change to `main` branch of `service-a` repo.
2. GitHub Actions should:

   * Build the app
   * Copy Dockerfile + JAR to EC2
   * SSH into EC2, rebuild Docker image, restart container
3. Visit `http://<EC2_PUBLIC_IP>:8080` or `:8081`

---

## 🔐 Optional: Secure EC2 Access

* Use `UFW` or EC2 Security Groups to allow only ports you need (e.g., 8080, 8081).
* Limit SSH access to GitHub Actions IP range if needed.

---

## 📌 Additional Improvements

* Add `nginx` reverse proxy for multiple services on port 80
* Push Docker images to ECR or Docker Hub if scaling
* Use `.env` files for container config
* Add Slack notification in GitHub Actions for deployment status

---

## 🏁 Summary

| Task              | Tool Used           |
| ----------------- | ------------------- |
| App Packaging     | Spring Boot + Maven |
| Dockerization     | Dockerfile          |
| CI/CD Pipeline    | GitHub Actions      |
| Secure Deployment | SSH Keys + Secrets  |
| Server Runtime    | Amazon Linux 2023   |

---

Let me know if you'd like a prebuilt example repo or an Nginx reverse proxy setup!
