# Automated Deployment Documentation

This document describes the automated deployment process for both the Node.js backend and the Next.js frontend projects. The backend is deployed to AWS Elastic Beanstalk (running on EC2), while the frontend is deployed to AWS Amplify with auto-deploy enabled for the main branch.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Backend Deployment Workflow](#backend-deployment-workflow)
    - [Workflow Trigger](#workflow-trigger)
    - [Job Steps Explanation](#job-steps-explanation)
4. [AWS CLI Configuration and Deployment](#aws-cli-configuration-and-deployment)
5. [Frontend Deployment to AWS Amplify](#frontend-deployment-to-aws-amplify)
6. [Security and Secrets Management](#security-and-secrets-management)
7. [Additional Considerations](#additional-considerations)
8. [Conclusion](#conclusion)

---

## Overview

The deployment pipeline leverages GitHub Actions to automate the build, test, and deployment processes:

- **Backend (Node.js):**  
  On every push to the `newmain` branch, the workflow checks out the code, sets up the Node.js environment, installs dependencies, runs tests, builds the project, and then packages it into a zip file. The package is then uploaded to an S3 bucket and deployed to AWS Elastic Beanstalk.

- **Frontend (Next.js):**  
  The frontend is deployed to AWS Amplify. The Amplify service is configured to automatically deploy changes when updates are pushed to the main branch.

---

## Prerequisites

Before using the deployment pipeline, ensure that you have:

- An AWS account with the necessary permissions to create/update Elastic Beanstalk applications and manage S3.
- An S3 bucket (e.g., `elasticbeanstalk-eu-west-2-588738592863`) configured for storing deployment packages.
- AWS credentials (Access Key ID and Secret Access Key) stored as GitHub repository secrets.
- GitHub Actions enabled on your repository.
- An existing Elastic Beanstalk application (e.g., named `kreatoors`) and a specific environment (e.g., `development-env`).

For the frontend:
- An AWS Amplify app configured with auto-deploy from the main branch of your Git repository.

---

## Backend Deployment Workflow

### Workflow Trigger

The workflow is defined in a YAML file (`eb-deploy-newmain.yml`) and is triggered by a push event on the `newmain` branch:

```yaml
on:
  push:
    branches: 
      - newmain
```

This ensures that every update pushed to `newmain` initiates the deployment process.

### Job Steps Explanation

The workflow consists of a single job (`deploy`) that runs on an Ubuntu runner. Below is a breakdown of each step:

1. **Checkout Code:**
   - **Step:** Uses the `actions/checkout@v3` action.
   - **Purpose:** Retrieves the repository code, ensuring that the workflow works with the latest commit.

2. **Set Up Node.js:**
   - **Step:** Uses `actions/setup-node@v3` to set up Node.js version 20.
   - **Purpose:** Ensures that the runtime environment matches your project’s requirements.

3. **Install Yarn:**
   - **Step:** Runs a command to install Yarn globally.
   - **Purpose:** Yarn is used as the package manager to install dependencies.

4. **Install Dependencies:**
   - **Step:** Runs `yarn install`.
   - **Purpose:** Installs all necessary project dependencies.

5. **Run Tests:**
   - **Step:** Executes `yarn test`.
   - **Purpose:** Runs unit and integration tests to ensure code quality before deployment.

6. **Build Project:**
   - **Step:** Runs `yarn build`.
   - **Purpose:** Compiles the project (if applicable) and prepares it for deployment.

7. **Zip Application:**
   - **Step:** Uses the `zip` command to package the project into a zip file, named with the commit SHA.
   - **Purpose:** Creates a deployable artifact while excluding unnecessary files (e.g., Git metadata).

8. **Configure AWS CLI:**
   - **Step:** Sets AWS CLI configuration parameters (access key, secret, and region) using secrets.
   - **Purpose:** Prepares the environment to interact with AWS services securely.

9. **Upload Package to S3:**
   - **Step:** Uses the AWS CLI to copy the zip file to the specified S3 bucket.
   - **Purpose:** Makes the deployment package available to AWS Elastic Beanstalk.

10. **Deploy to Elastic Beanstalk:**
    - **Step 1:** Creates a new application version using `aws elasticbeanstalk create-application-version`, linking the version label to the commit SHA.
    - **Step 2:** Updates the Elastic Beanstalk environment (`development-env`) using `aws elasticbeanstalk update-environment`.
    - **Purpose:** Deploys the new version of your application, triggering an update on the running environment.

---

## AWS CLI Configuration and Deployment

### Configuring AWS CLI

The AWS CLI is configured dynamically within the workflow using GitHub Secrets:

```yaml
- name: Configure AWS CLI
  run: |
    aws configure set aws_access_key_id ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws configure set aws_secret_access_key ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws configure set region ${{ secrets.AWS_REGION }}
```

This step ensures that your deployment scripts can securely access AWS resources.

### Deployment Commands

- **Creating a New Application Version:**

  ```bash
  aws elasticbeanstalk create-application-version \
    --application-name kreatoors \
    --version-label ${{ github.sha }} \
    --source-bundle S3Bucket="elasticbeanstalk-eu-west-2-588738592863",S3Key="${{ github.sha }}-deploy.zip" \
    --region ${{ secrets.AWS_REGION }}
  ```

  This command registers a new version of the application on Elastic Beanstalk using the zip package stored in S3.

- **Updating the Environment:**

  ```bash
  aws elasticbeanstalk update-environment \
    --environment-name development-env \
    --version-label ${{ github.sha }} \
    --region ${{ secrets.AWS_REGION }}
  ```

  After creating the new version, this command updates the environment to deploy the new version, ensuring the latest code is live.

---

## Frontend Deployment to AWS Amplify

For the Next.js frontend:

- **Auto Deploy on Main Branch:**  
  AWS Amplify is configured to monitor the `main` branch. When code is pushed to `main`, Amplify automatically triggers a build and deploy process. This process includes:
  - Pulling the latest code from the repository.
  - Installing dependencies.
  - Building the Next.js application.
  - Deploying the built application to a globally available CDN.

- **Configuration:**  
  Typically, you configure the Amplify app via the AWS Amplify Console by connecting your Git repository and setting build settings (e.g., environment variables, build commands). The build settings might include commands like:

  ```yaml
  version: 1
  frontend:
    phases:
      preBuild:
        commands:
          - npm install
      build:
        commands:
          - npm run build
    artifacts:
      baseDirectory: .next
      files:
        - '**/*'
    cache:
      paths:
        - node_modules/**/*      
  ```

This setup ensures that every commit to the `main` branch automatically updates your frontend deployment.

---

## Security and Secrets Management

- **GitHub Secrets:**  
  AWS credentials (such as `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_REGION`) are stored securely as GitHub repository secrets. These are referenced in the workflow using the `${{ secrets.SECRET_NAME }}` syntax.
  
- **AWS IAM Policies:**  
  Ensure that the IAM user or role used for deployment has the minimal necessary permissions to perform actions on Elastic Beanstalk, S3, and other AWS resources.

---

## Additional Considerations

- **Error Handling & Notifications:**  
  Consider adding error handling steps or notifications (e.g., via Slack or email) in the workflow to alert the team if a deployment fails.

- **Branch Strategy:**  
  The backend deployment is tied to the `newmain` branch, while the frontend uses the `main` branch. Ensure that your branch naming conventions and deployment strategies are well documented and followed by the team.

- **Testing:**  
  The workflow runs tests before deployment. It’s a good practice to ensure comprehensive test coverage to catch issues early.

- **Logging & Monitoring:**  
  After deployment, monitor the Elastic Beanstalk environment and the Amplify app using AWS CloudWatch, AWS Amplify Console logs, or other monitoring tools to quickly address any runtime issues.

---

## Conclusion

This documentation outlines the automated deployment process using GitHub Actions for a Node.js backend and a Next.js frontend. By following the steps detailed above, you can ensure that your deployments are consistent, secure, and automated. Future updates or adjustments can be made by modifying the GitHub workflow or updating AWS configurations as needed.

--- 

Feel free to reach out for any clarifications or additional enhancements to the documentation!
