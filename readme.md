# CI/CD Pipeline Setup Using GitHub Actions for React JS and Node.js (Express) on AWS

This guide explains how to set up a CI/CD pipeline for a full-stack application with:

- a React JS frontend (JavaScript, not Vite)
- a Node.js Express backend
- deployment on AWS
- load balancing for the backend using an Application Load Balancer (ALB)

This setup is designed for a harder, production-style deployment where:

- the React app is hosted from Amazon S3
- static content is served through Amazon CloudFront
- the Express backend runs on Amazon EC2 instances
- traffic to the backend goes through an AWS Application Load Balancer
- GitHub Actions handles CI and deployment

## 1. Recommended Architecture

Use this architecture for the full-stack app:

```text
Users
  |
  +--> CloudFront
         |
         +--> S3 bucket for React frontend
         |
         +--> /api requests to ALB
                          |
                          +--> EC2 instance 1 running Express
                          +--> EC2 instance 2 running Express
```

### Frontend

- React app is built in GitHub Actions
- the build output is uploaded to S3
- CloudFront serves the frontend globally

### Backend

- Express app is deployed to one or more EC2 instances
- an Application Load Balancer distributes traffic
- ALB target group performs health checks

## 2. Suggested Project Structure

```text
your-repo/
|-- client/
|   |-- package.json
|   |-- public/
|   |-- src/
|
|-- server/
|   |-- package.json
|   |-- app.js or server.js
|
|-- .github/
|   |-- workflows/
|       |-- ci.yml
|       |-- cd.yml
```

## 3. Prerequisites

Before setting up CI/CD, make sure you have:

- a GitHub repository
- a React app inside `client`
- an Express app inside `server`
- separate `package.json` files for both apps
- an AWS account
- an S3 bucket for frontend hosting
- a CloudFront distribution in front of the S3 bucket
- at least two EC2 instances for the backend
- an Application Load Balancer
- an ALB target group attached to the EC2 instances
- a security group allowing ALB-to-EC2 traffic
- an IAM user or IAM role with deployment permissions

## 4. Add Required npm Scripts

Example `client/package.json` scripts:

```json
{
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test --watchAll=false"
  }
}
```

Example `server/package.json` scripts:

```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test": "node --test"
  }
}
```

If you use Jest or another test runner on the backend, replace the `test` script accordingly.

## 5. Create the GitHub Actions Folder

Create this directory in the repo root:

```text
.github/workflows
```

## 6. Create the CI Workflow

Create:

```text
.github/workflows/ci.yml
```

Use this workflow:

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  frontend:
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

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test -- --watchAll=false

      - name: Build frontend
        run: npm run build

  backend:
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

      - name: Install dependencies
        run: npm ci

      - name: Run backend tests
        run: npm test
```

## 7. What the CI Workflow Does

This workflow:

- runs on push to `main` and `develop`
- runs on pull requests targeting `main` and `develop`
- installs frontend dependencies
- tests and builds the React app
- installs backend dependencies
- runs backend tests

This prevents broken code from reaching deployment.

## 8. AWS Infrastructure Setup

Before writing the CD pipeline, create the AWS infrastructure.

### 8.1 Create an S3 Bucket for React

1. Open AWS S3.
2. Create a bucket for the frontend.
3. Use a globally unique name such as `my-fullstack-react-frontend-prod`.
4. Keep versioning enabled if possible.
5. Use the bucket to store the built React files.

## 8.2 Create a CloudFront Distribution

1. Open CloudFront.
2. Create a distribution.
3. Set the S3 bucket as the origin.
4. Configure default root object as `index.html`.
5. If using React Router, configure custom error handling:
   - `403` -> `/index.html`
   - `404` -> `/index.html`
6. Use CloudFront to cache and serve the frontend.

## 8.3 Launch EC2 Instances for Express

1. Launch at least two EC2 instances in the same VPC.
2. Use Amazon Linux 2 or Ubuntu.
3. Install Node.js, npm, and PM2.
4. Clone or prepare deployment access to your app.
5. Open the backend application port, such as `3000`, only from the ALB security group.

Install example dependencies on EC2:

```bash
sudo yum update -y
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -
sudo yum install -y nodejs git
sudo npm install -g pm2
```

For Ubuntu:

```bash
sudo apt update
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs git
sudo npm install -g pm2
```

## 8.4 Create an Application Load Balancer

1. Open EC2 in AWS console.
2. Create a new Application Load Balancer.
3. Choose internet-facing if your API is public.
4. Attach the ALB to public subnets.
5. Add listeners:
   - `HTTP : 80`
   - `HTTPS : 443` if you have ACM certificates

## 8.5 Create a Target Group

1. Create a target group of type `Instances`.
2. Choose protocol `HTTP`.
3. Use the backend application port, for example `3000`.
4. Set the health check path to something like:

```text
/health
```

5. Register the EC2 instances in the target group.

Your Express app should expose a health route:

```js
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});
```

## 8.6 Configure Security Groups

Recommended security setup:

- ALB security group:
  - allow inbound `80` and `443` from the internet
- EC2 security group:
  - allow inbound app port like `3000` only from the ALB security group
  - allow SSH only from your trusted IP if needed

## 9. Decide How GitHub Actions Will Deploy to AWS

For this setup, a strong approach is:

- deploy frontend directly from GitHub Actions to S3
- invalidate CloudFront cache after upload
- deploy backend to EC2 using AWS Systems Manager (SSM) or SSH

For production, SSM is preferred over plain SSH because it is more secure and easier to audit.

## 10. Store GitHub Secrets

In your GitHub repository:

1. Open `Settings`.
2. Go to `Secrets and variables` > `Actions`.
3. Add these secrets:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_REGION`
- `S3_BUCKET_NAME`
- `CLOUDFRONT_DISTRIBUTION_ID`
- `EC2_INSTANCE_ID_1`
- `EC2_INSTANCE_ID_2`

If your backend uses environment variables, also add:

- `JWT_SECRET`
- `MONGO_URI`
- any app-specific secrets

## 11. Create the CD Workflow for AWS

Create:

```text
.github/workflows/cd.yml
```

Use this example:

```yaml
name: CD Pipeline AWS

on:
  push:
    branches:
      - main

jobs:
  deploy-frontend:
    name: Deploy Frontend to S3 and CloudFront
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

      - name: Build frontend
        run: npm run build

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Upload build to S3
        run: aws s3 sync build/ s3://${{ secrets.S3_BUCKET_NAME }} --delete

      - name: Invalidate CloudFront cache
        run: aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"

  deploy-backend:
    name: Deploy Backend to EC2
    runs-on: ubuntu-latest
    needs: deploy-frontend

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Deploy to EC2 instance 1 using SSM
        run: |
          aws ssm send-command \
            --instance-ids "${{ secrets.EC2_INSTANCE_ID_1 }}" \
            --document-name "AWS-RunShellScript" \
            --comment "Deploy Node backend" \
            --parameters 'commands=[
              "cd /var/www/app/server",
              "git pull origin main",
              "npm ci",
              "pm2 restart server || pm2 start server.js --name server"
            ]'

      - name: Deploy to EC2 instance 2 using SSM
        run: |
          aws ssm send-command \
            --instance-ids "${{ secrets.EC2_INSTANCE_ID_2 }}" \
            --document-name "AWS-RunShellScript" \
            --comment "Deploy Node backend" \
            --parameters 'commands=[
              "cd /var/www/app/server",
              "git pull origin main",
              "npm ci",
              "pm2 restart server || pm2 start server.js --name server"
            ]'
```

## 12. Prepare EC2 for Git Pull and App Restart

Each EC2 backend server should:

1. have your project checked out at a known path such as `/var/www/app`
2. have Node.js installed
3. have PM2 installed
4. have permission to pull from GitHub
5. have SSM agent installed and active
6. have an IAM role allowing SSM access

Example first-time setup on EC2:

```bash
mkdir -p /var/www/app
cd /var/www/app
git clone https://github.com/your-username/your-repo.git .
cd server
npm install
pm2 start server.js --name server
pm2 save
pm2 startup
```

## 13. Recommended Zero-Downtime Backend Deployment Pattern

For better production reliability:

1. Put at least two EC2 instances behind the ALB.
2. Deploy one instance at a time.
3. Let the ALB health check confirm the instance is healthy.
4. Then deploy the next instance.

This avoids taking the whole API offline during deployment.

If you want an even stronger setup, use:

- AWS CodeDeploy with in-place or blue/green deployments
- an Auto Scaling Group behind the ALB
- launch templates for repeatable server creation

That is usually the best long-term AWS deployment model.

## 14. Optional Improved Production Architecture

For a more robust full-stack AWS setup, use:

- React frontend in S3 + CloudFront
- Express backend on EC2 Auto Scaling Group
- ALB in front of EC2
- RDS for relational database if needed
- ElastiCache for caching if needed
- Route 53 for DNS
- ACM for SSL certificates
- CodeDeploy for controlled rolling deployments

This is better than deploying everything to a single EC2 server.

## 15. Branch Protection and Release Flow

In GitHub:

1. open `Settings`
2. open `Branches`
3. add a branch protection rule for `main`
4. require pull requests before merging
5. require CI checks to pass before merging

Recommended flow:

1. push feature code to a branch
2. open a pull request
3. GitHub Actions runs CI
4. merge into `main`
5. GitHub Actions deploys frontend to S3 and backend to EC2
6. ALB continues routing traffic to healthy instances

## 16. Optional Improvements

You can improve this pipeline further by adding:

- ESLint checks for frontend and backend
- test coverage reporting
- Docker images pushed to Amazon ECR
- ECS or EKS instead of EC2 for containerized deployment
- staging and production AWS environments
- Terraform or CloudFormation for infrastructure as code
- rollback logic if health checks fail

## 17. Final Notes

- Keep the frontend and backend deployment steps separate.
- Use S3 plus CloudFront for the React app instead of serving static files from Express.
- Use at least two backend instances behind the ALB.
- Prefer SSM or CodeDeploy over manual SSH deployment.
- Add a `/health` route in Express for ALB health checks.
- Test the workflow in a non-production AWS environment first.

If you want, the next step can be creating actual ready-to-use files for:

- `.github/workflows/ci.yml`
- `.github/workflows/cd.yml`
- a sample `app.get('/health', ...)` Express snippet
- a deployment script for EC2 instances
