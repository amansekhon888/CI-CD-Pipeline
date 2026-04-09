# Guide 1: Medium Experiment

## CI/CD Pipeline Using GitHub Actions

This guide explains how to build a simple CI/CD pipeline using GitHub Actions for a full-stack project.

For Docker understanding before starting this experiment, refer to this Dockerization example:
[https://github.com/amansekhon888/dockerize-react](https://github.com/amansekhon888/dockerize-react)

For better conceptual understanding of CI/CD, pipelines, and GitHub Actions, refer to [ci-cd-pipelines.md](./ci-cd-pipelines.md).

This version is designed for a medium-level experiment, so it does not require AWS.  
Instead, it focuses on:

- GitHub Actions for automation
- Docker for containerization
- Docker Hub for storing images

---

## 1. Aim of the Experiment

The goal of this experiment is to automate the process of:

1. pulling code from GitHub
2. installing dependencies
3. running tests
4. building the application
5. creating Docker images
6. pushing Docker images to Docker Hub

This is a simple but real CI/CD workflow.

---

## 2. What You Will Build

You will create a pipeline for a full-stack application with:

- a React frontend
- a Node.js + Express backend
- GitHub as the source code repository
- GitHub Actions as the CI/CD automation tool
- Docker Hub as the image registry

---

## 3. Required Tools

Before starting, make sure you have:

- a GitHub account
- a GitHub repository
- Docker installed on your system
- a Docker Hub account
- a React frontend project
- a Node.js Express backend project

---

## 4. Recommended Project Structure

Organize the project like this:

```text
project-root/
|-- client/
|   |-- package.json
|   |-- src/
|   |-- public/
|   |-- Dockerfile
|
|-- server/
|   |-- package.json
|   |-- server.js
|   |-- Dockerfile
|
|-- .github/
|   |-- workflows/
|       |-- ci.yml
|       |-- cd.yml
|
|-- docker-compose.yml
```

This structure helps separate frontend and backend pipelines clearly.

---

## 5. Step 1: Prepare npm Scripts

Your project should include scripts for building and testing.

### Example `client/package.json`

```json
{
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test --watchAll=false"
  }
}
```

### Example `server/package.json`

```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test": "node --test"
  }
}
```

If you use another test framework, replace the test command accordingly.

---

## 6. Step 2: Create Dockerfiles

To package the frontend and backend, create Dockerfiles for both.

### Frontend Dockerfile

Create `client/Dockerfile`

```dockerfile
FROM node:20-alpine AS build

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:stable-alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Backend Dockerfile

Create `server/Dockerfile`

```dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
EXPOSE 5000
CMD ["npm", "start"]
```

---

## 7. Step 3: Optional Docker Compose File

Create `docker-compose.yml` in the root directory:

```yaml
version: '3.9'

services:
  frontend:
    build: ./client
    ports:
      - "3000:80"
    depends_on:
      - backend

  backend:
    build: ./server
    ports:
      - "5000:5000"
```

This helps run both services together locally.

---

## 8. Step 4: Create the GitHub Actions Folder

Create the following folder:

```text
.github/workflows
```

This is where GitHub Actions workflow files are stored.

---

## 9. Step 5: Create the CI Workflow

Create `.github/workflows/ci.yml`

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  frontend-ci:
    name: Frontend CI
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: client

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: client/package-lock.json

      - name: Install frontend dependencies
        run: npm ci

      - name: Run frontend tests
        run: npm test -- --watchAll=false

      - name: Build frontend
        run: npm run build

  backend-ci:
    name: Backend CI
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: server

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: server/package-lock.json

      - name: Install backend dependencies
        run: npm ci

      - name: Run backend tests
        run: npm test
```

### What this CI workflow does

- checks out the source code
- installs dependencies
- runs tests
- builds the frontend
- verifies the backend

This ensures that broken code does not move further in the pipeline.

---

## 10. Step 6: Create the CD Workflow

Create `.github/workflows/cd.yml`

```yaml
name: CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  docker-build-and-push:
    name: Build and Push Docker Images
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build frontend image
        run: docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/frontend-app:latest ./client

      - name: Build backend image
        run: docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/backend-app:latest ./server

      - name: Push frontend image
        run: docker push ${{ secrets.DOCKERHUB_USERNAME }}/frontend-app:latest

      - name: Push backend image
        run: docker push ${{ secrets.DOCKERHUB_USERNAME }}/backend-app:latest
```

### What this CD workflow does

- runs when code is pushed to `main`
- logs in to Docker Hub
- builds Docker images
- pushes those images to Docker Hub

This is the deployment part of the experiment.

---

## 11. Step 7: Add GitHub Secrets

In the GitHub repository:

1. open `Settings`
2. open `Secrets and variables`
3. click `Actions`
4. add the following secrets

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`

### How to get the Docker Hub token

1. log in to Docker Hub
2. open `Account Settings`
3. go to `Security`
4. create a new access token
5. copy the token and store it in GitHub Secrets

---

## 12. Step 8: Test the Pipeline

After adding the workflows:

1. push code to a feature branch
2. open a pull request
3. check the `Actions` tab in GitHub
4. confirm that the CI workflow runs successfully
5. merge the code into `main`
6. confirm that the CD workflow builds and pushes Docker images

---

## 13. Step 9: Verify in Docker Hub

After the CD workflow completes:

1. open Docker Hub
2. log in to your account
3. check whether the repositories exist
4. confirm that the latest images were pushed successfully

You should see:

- `frontend-app`
- `backend-app`

with the `latest` tag.

---

## 14. Advantages of This Setup

This setup gives you:

- automated testing
- automatic builds
- repeatable deployment steps
- Docker-based packaging
- a simple CI/CD process without cloud infrastructure

This makes it ideal for learning and lab experiments.

---

## 15. Expected Output

At the end of this experiment:

- code is automatically tested on GitHub
- frontend and backend images are automatically built
- Docker images are automatically pushed to Docker Hub
- the pipeline becomes reproducible and reliable

---

## 17. Conclusion

In this experiment, you created a medium-level CI/CD pipeline using GitHub Actions.

You learned how to:

- automate build and test stages
- containerize frontend and backend applications
- push Docker images to Docker Hub
- connect source code changes with deployment automation

This experiment provides a strong foundation before moving to more advanced cloud-based deployment pipelines.