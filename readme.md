# CI/CD Pipeline Setup Using GitHub Actions for React JS and Node.js (Express)

This guide explains how to set up a basic CI/CD pipeline using GitHub Actions for:

- a frontend built with pure React JS (JavaScript, not Vite)
- a backend built with Node.js and Express

The setup assumes both apps are stored in the same GitHub repository.

## 1. Suggested Project Structure

Use a structure similar to this:

```text
your-repo/
|-- client/
|   |-- package.json
|   |-- src/
|   |-- public/
|
|-- server/
|   |-- package.json
|   |-- src/ or app.js / server.js
|
|-- .github/
|   |-- workflows/
|       |-- ci.yml
|       |-- cd.yml
```

## 2. Prerequisites

Before creating the pipeline, make sure you have:

- a GitHub repository for your project
- a React app inside the `client` folder
- a Node.js Express app inside the `server` folder
- `package.json` files in both `client` and `server`
- scripts defined for install, test, and build

## 3. Add Required npm Scripts

Make sure your frontend `client/package.json` includes scripts like:

```json
{
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test --watchAll=false"
  }
}
```

Make sure your backend `server/package.json` includes scripts like:

```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test": "node --test"
  }
}
```

If your server uses Jest or another test runner, replace the `test` command accordingly.

## 4. Create the GitHub Actions Folder

Inside the root of your repository, create:

```text
.github/workflows
```

This is where GitHub Actions workflow files are stored.

## 5. Create the CI Workflow

Create a file at:

```text
.github/workflows/ci.yml
```

Add the following content:

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  frontend:
    name: Test Frontend
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
    name: Test Backend
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

## 6. What This CI Workflow Does

This workflow:

- runs whenever code is pushed to `main` or `develop`
- runs on pull requests targeting `main` or `develop`
- installs dependencies for the React app
- tests and builds the React app
- installs dependencies for the Express app
- runs backend tests

This helps catch issues before deployment.

## 7. Create the CD Workflow

Create another file at:

```text
.github/workflows/cd.yml
```

Add this example workflow:

```yaml
name: CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  deploy:
    name: Deploy Application
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Deployment placeholder
        run: echo "Add your deployment commands here"
```

## 8. Choose a Deployment Target

GitHub Actions can deploy your frontend and backend to different platforms. Common choices are:

- React frontend to GitHub Pages, Netlify, or Vercel
- Express backend to Render, Railway, AWS, Azure, or a VPS

You should update `cd.yml` based on where you want to deploy.

## 9. Example: Deploy React App to GitHub Pages

If you want to deploy only the React frontend to GitHub Pages:

First install `gh-pages` in the client app:

```bash
cd client
npm install gh-pages --save-dev
```

Then update `client/package.json`:

```json
{
  "homepage": "https://your-username.github.io/your-repo-name",
  "scripts": {
    "predeploy": "npm run build",
    "deploy": "gh-pages -d build"
  }
}
```

Example frontend deployment workflow:

```yaml
name: Deploy React Frontend

on:
  push:
    branches:
      - main

jobs:
  deploy-frontend:
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

      - name: Build app
        run: npm run build

      - name: Deploy to GitHub Pages
        run: npm run deploy
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## 10. Example: Deploy Express App to Render

For Express apps, deployment is usually handled by connecting the GitHub repo directly to the hosting provider.

For Render:

1. Create an account on Render.
2. Click `New +` and choose `Web Service`.
3. Connect your GitHub repository.
4. Select the `server` folder as the root directory if needed.
5. Set:
   - Build Command: `npm install`
   - Start Command: `npm start`
6. Add environment variables in Render dashboard.
7. Enable auto-deploy from the `main` branch.

In this case, your GitHub Actions CI workflow still verifies code quality, while Render handles backend deployment automatically after GitHub updates.

## 11. Store Secrets Safely

If your deployment requires tokens, API keys, or credentials:

1. Open your GitHub repository.
2. Go to `Settings`.
3. Open `Secrets and variables` > `Actions`.
4. Add repository secrets such as:
   - `RENDER_API_KEY`
   - `JWT_SECRET`
   - `MONGO_URI`
   - any deployment token required by your hosting provider

Use secrets in workflows like this:

```yaml
env:
  MONGO_URI: ${{ secrets.MONGO_URI }}
```

Never hardcode secrets directly in workflow files.

## 12. Add Branch Protection

To make CI/CD more reliable:

1. Go to GitHub repository `Settings`.
2. Open `Branches`.
3. Add a branch protection rule for `main`.
4. Enable:
   - `Require a pull request before merging`
   - `Require status checks to pass before merging`
   - select your CI workflow checks

This ensures only tested code reaches production.

## 13. Optional Improvements

After the basic pipeline works, you can improve it by adding:

- linting with ESLint
- code formatting checks
- test coverage reports
- separate staging and production deployments
- Docker-based deployment
- notifications to Slack or email
- environment-specific `.env` management

## 14. Recommended Workflow Summary

A practical setup is:

1. Developer pushes code to a feature branch.
2. Pull request is created to `develop` or `main`.
3. GitHub Actions runs frontend and backend CI checks.
4. If tests pass, code is merged.
5. A push to `main` triggers deployment.
6. Frontend and backend are deployed to their respective platforms.

## 15. Final Notes

- Keep frontend and backend workflows simple at first.
- Make sure both apps run correctly locally before automating them.
- Always test the workflow after pushing the YAML files.
- Adjust folder paths if your project structure is different.

If you want, you can also split the pipeline into:

- one workflow for frontend CI/CD
- one workflow for backend CI/CD
- one workflow for full-stack deployment

That approach is useful for larger projects.
