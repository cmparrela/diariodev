---
date: '2024-10-05T21:09:33-03:00'
title: 'How to Deploy AWS Lambda Golang with GitHub Actions'
draft: true
showToc: false
categories:
- AWS Lambda
- Golang
- GitHub Actions
---

In this post, I will continue the series on AWS Lambda with Golang. Now, let's learn how to set up a CI/CD pipeline and deploy using GitHub Actions.

# GitHub Actions

GitHub Actions is GitHub's CI/CD service that allows you to automate workflows in a simple and efficient way. It is highly flexible and integrated with GitHub, making it easy to use in new and existing projects.

Although the official documentation is quite comprehensive ([check it out here](https://docs.github.com/en/actions)), in this post I will present a practical example and explain how you can start using GitHub Actions to deploy AWS Lambda applications written in Golang.

## Pricing

GitHub Actions is free for public repositories, but for private repositories, there is a limit to what you can use for free, which includes:

- **2,000 minutes per month** for running Actions.
- **500 MB of storage** for generated artifacts.

If you exceed the quota, you can opt for paid plans to increase the limits. (Prices may vary, so check the [official pricing page](https://github.com/features/actions#pricing) for more details.)

# Creating the Deployment File

I recommend that you first read this file creation step, understand what is being done, and at the end, you will find the complete file, which can be copied and adapted to your needs.

To start, let's create a folder at the root of the project hosted on GitHub called `.github`. Inside this folder, we will create another one called `workflows`. Finally, inside it, we will create the workflow file for the deployment, with a name of our choice. Here, I will use `sam-pipeline.yml`. The final result will be like this:

```bash
.github/workflows/sam-pipeline.ym
```

Let's start writing inside this file. The logic will be as follows: whenever a release is published on GitHub, the pipeline will be triggered to send the code to production.

```yaml
name: Deploy to Production

on:
  release:
    types: [published]
...
```

To prevent multiple pipelines from running in parallel, we will use the `concurrency` feature. This feature groups executions of the same group, we will give a name to the group and configure it to cancel previous executions if there is more than one running simultaneously.

```yaml
...
concurrency:
  group: sam-pipeline-prod
  cancel-in-progress: true
...
```

Now, let's start creating the jobs. The first jobs are quite simple and self-explanatory, so I will just paste the code:

```yaml
...
jobs:
  deploy-lambda:
    name: Deploy to AWS
    runs-on: ubuntu-latest
    environment:
      name:  'production'

    steps:
      - name: Checkout branch
        uses: actions/checkout@v4

      - name: Install Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22.2'
      
      - name: Download dependencies
        run: go mod tidy

      - name: Build with Go
        run: go build -o bootstrap main.go

      - name: Set up AWS SAM
        uses: aws-actions/setup-sam@v2
...
```

The next step is to configure the **AWS Credentials**. Now, let's add the secrets to our repository, which will be used in the pipeline. In the repository, click on **Settings → Actions secrets and variables → Actions** and add the keys **AWS_ACCESS_KEY_ID** and **AWS_SECRET_ACCESS_KEY** of an AWS user.

The code to access the secrets in GitHub Action will look like this:

```yaml
...
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-2

...
```

To finish, let's add the **sam build** and **sam deploy** steps. In this example, I used the **SAM Template** from AWS and named the production file **template-prod.yaml**.

```yaml
...
      - name: Build with SAM
        run: sam build --template template-prod.yaml
      
      - name: Deploy with SAM
        run: sam deploy --no-confirm-changeset --no-fail-on-empty-changeset
...
```

# Complete File

```yaml
# .github/workflows/sam-pipeline.ym
name: Deploy to Production

on:
  release:
    types: [published]

concurrency:
  group: sam-pipeline-prod
  cancel-in-progress: true
  
jobs:
  deploy-lambda:
    name: Deploy to AWS
    runs-on: ubuntu-latest
    environment:
      name:  'production'

    steps:
      - name: Checkout branch
        uses: actions/checkout@v4

      - name: Install Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22.2'
      
      - name: Download dependencies
        run: go mod tidy

      - name: Build with Go
        run: go build -o bootstrap main.go

      - name: Set up AWS SAM
        uses: aws-actions/setup-sam@v2
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-2

      - name: Build with SAM
        run: sam build --template template-prod.yaml
      
      - name: Deploy with SAM
        run: sam deploy --no-confirm-changeset --no-fail-on-empty-changeset
````