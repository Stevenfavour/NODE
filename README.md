## Deployment and Cloud Platforms

In this section, we are going to explore how we can leverage Github Actions to automate deployments processes, effectively pushing applications to various cloud envirnoments. 
Github provides the automation, ensuring that deployments processes are as smooth, error-free and as efficient as possible.  

### Defining Deployment Stages


* Development: This is where the work begins. Developers write and test code on their local machines, ensuring the new features or fixes work as expected in a confined, individual environment before sharing them with the team.

* Integration: Once a developer's code is ready, it's merged into a shared branch (like staging or main). This is a critical stage where the individual pieces of work come together, and potential conflicts between different developers' code are resolved.

* Testing: After integration, automated tests are run (unit, integration, end-to-end) to verify that the combined code is stable and meets quality standards. This step ensures that the new changes haven't introduced regressions or bugs into existing functionality.

* Staging: The stable, tested code is then deployed to a production-like environment. This Staging environment is designed to closely mimic the real Production environment. The goal is to perform final, comprehensive quality checks (manual testing, performance testing, user acceptance testing) in an environment that is as close to live as possible without impacting actual end-users.

* Production: This is the final step: releasing the final, verified version of the code to the end-users. The code is now live and accessible to the public or the target audience.


![](/Img29/14.png)

The image above shows a remote repository with two branches: 

`staging` is used for the staging environment, while `main` is commonly used for the production environment. This is the most important branch where codes are deployed from to Live environments. 

### Understanding Deployment Strategies
Here is an explanation of each strategy:

* Blue-Green Deployment:

Description: This involves running two identical production environments ("Blue" and "Green"). At any given time, only one environment (e.g., "Blue") is live and serving all user traffic. The new version is deployed to the idle environment (e.g., "Green"). Once testing is complete on "Green," traffic is instantly switched from "Blue" to "Green."

Key Benefit: Provides instant rollback capabilities. If an issue is found, traffic can be instantly switched back to the stable "Blue" environment with zero downtime.

* Canary Releases:

Description: Changes are rolled out to a small, controlled subset of users before a full deployment. The small group of "Canary" users helps test the new version in a real production setting. If the new version is stable, it is gradually rolled out to the rest of the user base.

Key Benefit: Reduces the risk of a widespread failure. If the canary release fails, only a small percentage of users are affected.

* Rolling Deployment:

Description: This involves gradually replacing instances of the previous version of an application with the new version. In an environment with multiple servers/instances, the update is applied to one or a few servers at a time until all instances are running the new version.

Key Benefit: Allows for zero-downtime deployment by ensuring that some instances are always running the older, stable version while others are being updated.

### Deploying Application to Cloud (AWS) with GitHub Actions

To deploy applications automatically and efficiently, we first need to create a workflow. A workflow is a structured, repeatable sequence of tasks, activities, and decision points that takes a piece of work, information, or material from initiation to completion to achieve a specific goal or outcome.

It essentially defines how work gets done, acting as a roadmap that ensures consistency and clarity at every stage.

Workflow files are YAML files stored in the repository inside the `.github/workflow` folder. Since we are working on the main branch, we will name our workflow based on this. 

![](/Img29/15.png)

From the image above, we have two different deployment workflows for each environment (production and staging)


#### Defining the workflow

A workflow is defined with a series of steps that run on specified events. 

```name: Deploy via AWS CodeDeploy

on:
  push:
    branches:
      - main
  workflow_dispatch:

env:
  AWS_REGION: eu-north-1 # <-- Change this to your instance's region
  APPLICATION_NAME: MyNodeApp # <-- Must match the name in AWS CodeDeploy
  DEPLOYMENT_GROUP_NAME: MyDeploymentGroup # <-- Must match the name in AWS CodeDeploy

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22.x'
          cache: 'npm'

      - name: Install Dependencies
        run: npm ci

      #- name: Code Quality Checks (lint test)
       # run: npm run lint
       # I want to suspend the code quality check gate

      - name: Build, and Test
        run: |
          npm run build --if-present
          npm test
          # NOTE: If tests fail, the workflow will stop here.

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Create Deployment Bundle (Zip)
        run: |
          # 1. Create the appspec.yml file using the cleaner 'cat <<EOF' syntax
          cat <<EOF > appspec.yml
          version: 0.0
          os: linux
          files:
            - source: /
              destination: /var/www/html/mynodeapp # <-- Your destination path
          hooks:
            ApplicationStart:
              # *** FIXED: Reference the script in the root ***
              - location: start_server.sh
                timeout: 300
                runas: ec2-user # <-- Recommend setting runas to the user running your app
          EOF
          
          # 2. Create the start_server.sh script file in the root
          cat <<'EOF' > start_server.sh
          #!/bin/bash
          
          # Navigate to the application's installation directory
          cd /var/www/html/mynodeapp/
          
          # Use 'npm start' or a process manager like PM2 to run your app
          # Ensure Node.js is installed on your EC2 instance and npm start works.
          /usr/bin/npm start &
          EOF
          
          # 3. Make the script executable (CRITICAL STEP)
          chmod +x start_server.sh
          
          # 4. Bundle all necessary files (including the newly created appspec.yml and start_server.sh)
          zip -r deployment-package.zip . -x "**/.git/*"

      - name: Upload to S3
        # Uploads the deployment package to a temporary S3 bucket
        run: |
          # Use AWS CLI to securely copy the zip file to your S3 bucket
          aws s3 cp deployment-package.zip s3://iamusermary001/${{ github.sha }}.zip


      - name: Deploy to EC2 via CodeDeploy
        # Initiates the deployment using the CodeDeploy service
        run: |
          aws deploy create-deployment \
            --application-name ${{ env.APPLICATION_NAME }} \
            --deployment-group-name ${{ env.DEPLOYMENT_GROUP_NAME }} \
            --deployment-config-name CodeDeployDefault.OneAtATime \
            --s3-location bucket=iamusermary001,key=${{ github.sha }}.zip,bundleType=zip


```

The above workflow deploys the application to the AWS cloud platform when changes are pushed to the main branch. 

Now let's explore some aspects of the above workflow.

#### Configuring Deployments Environments

* Environment variables are non-sensitive information that can be used within the workflow. E.g., Application name, AWS region, Deployment Group name, etc.
They are written within the `env` block of the workflow.

![](/Img29/16.png)

* Secrets are sensitive information that can't be hardcoded into the workflow. Hence, we leverage GitHub to store that information and reference it from the workflow. This way this information are not exposed on the workflow and logs.

![](/Img29/11.png)

This is how we reference the above secrets inside the workflow file.

![](/Img29/17.png)

#### AWS code deployment using CodeDeploy on AWS

Earlier, we talked about deployment stategies. Here, we are going to be focused on the two branches and they will both have different Codedeploy applications and Deployment groups. 

![](/Img29/1.png)

The image above shows the deployment application for the staging environment. A blue/green deployment strategy is also implemented. 

![](/Img29/2.png)

We also create a deployment group for this application.

![](/Img29/3.png)

Once all these are done, we can push the changes to the remote repo, and the workflow is automatically triggered (by push).

![](/Img29/36.png)

![](/Img29/24.png)

### Configuring releases

Releases are basically the  act of creating a formal, versioned package of your application that is ready to be delivered.

In a complete and safe Continuous Delivery (CD) pipeline, the order is almost always:

1. Deployment Action

2. Release Action

The Deployment Action must come first because the formal Release Action (versioning, tagging, and creating release notes) should only happen after you have confirmed the code is successfully running in the Production environment.

#### Why Deployment Must Precede Release
If you create the version tag (v1.2.3) before deployment and the deployment then fails, you have a critical problem:

The code that failed in production is now officially tagged as a successful release (v1.2.3).

You must either delete the tag (bad practice) or create a confusing hotfix release (v1.2.4) to fix code that was never stable in the first place.

Here, we will update our previous workflow to include a release job.

![](/Img29/4.png)

Commit and Push.

![](/Img29/5.png)

Althoug from the highligted points from the image abovw, we see that deployment was successful, the release job failed.

Upon investigation, we discovered that the failure was caused by a permission denial. One of the scripts does not have execution permission, which will allow the script to be executed. To solve this, we add a step to grant permission to execute the necessary file.

![](/Img29/6.png)

Push the changes. 

![](/Img29/7.png)

The release job failed again, but this time the permission to push changes to the repository is absent. This is because the GitHub Actions [bot] user, which is what semantic-release typically uses when running inside a GitHub Action, is being denied permission to push to the repository. This happens because the default GITHUB_TOKEN provided to workflows only has read permissions for the repository content, or its permissions were explicitly limited.

We need to explicitly grant write permissions to the default GITHUB_TOKEN for the workflow job that runs semantic-release.

Modify the GitHub Actions Workflow file (.github/workflows/your-workflow.yml)

Add a block of permission to the workflow to add neccessary permission.

![](/Img29/8.png)

Push the changes.

The release job failed one more time.

![](/Img29/9.png)

This is a clear and common issue when using semantic-release to publish packages to npm.
Since we are publishing a package, semantic-release needs a secret token with publish rights to authenticate with the npm registry. This token must be set as an environment variable named NPM_TOKEN in your CI/CD environment (e.g., GitHub Actions).


Step 1: Create an npm Automation Access Token
Go to your npm account settings on https://www.npmjs.com/settings/YOUR_USERNAME/tokens.

Click "Generate New Token".

Choose the "Automation" type. This type has the necessary permissions to publish packages.

Copy the token immediately after it's generated, as you won't be able to see it again.


![](/Img29/10.png)

Step 2: Add the Token to GitHub Secrets
Go to your GitHub repository.

Navigate to Settings → Secrets and variables → Actions.

Click "New repository secret".

Set the Name to NPM_TOKEN.

Paste the npm Automation Token you copied in Step 1 into the Secret field.

Click "Add secret".

![](/Img29/11.png)

Step 3: Pass the Secret to the semantic-release Step
Ensure your GitHub Actions workflow file passes this new secret as an environment variable to the job running semantic-release.

![](/Img29/12.png)

Push the changes.

![](/Img29/13.png)

The image above shows a successful release job 

In this project, we have been able to configure a workflow that uses github action to deploy the  application to a cloud platform (AWS) as well as configure Releases that handle application versioning.









