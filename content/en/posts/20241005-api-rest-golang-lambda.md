---
date: '2024-11-05T21:09:33-03:00'
title: 'Creating a REST API with Golang and AWS Lambda'
---

In this post, we will explore how to build a REST API with Golang and deploy it on AWS Lambda, leveraging the benefits of serverless to create a scalable and cost-effective application.

One of the great advantages of using AWS Lambda is the cost reduction, especially with AWS's free tier, in addition to the decrease in development and maintenance time.

## What is AWS Lambda

AWS Lambda is AWS's serverless service. In the *serverless* model, you don't need to directly manage the servers where your application runs. In the case of AWS Lambda, AWS takes care of all the infrastructure, automatically scaling your application as needed. This means you only pay for the execution time of your function, which is ideal for applications with variable or intermittent loads.

With serverless, you can focus exclusively on business logic without worrying about server maintenance or complex scalability configurations. AWS Lambda handles resource management, freeing you to work on application features.

Additionally, the cost is a major attraction. In many cases, AWS Lambda can be free, thanks to AWS's generous free tier. This plan covers a considerable number of executions per month, making Lambda a great option for testing, development, and even production applications with moderate use.

### Free Tier

AWS offers a free tier for AWS Lambda that includes 1 million requests and 400,000 GB-seconds of compute time per month. This means you can process up to 1 million requests monthly at no additional cost, making Lambda very attractive for small projects and testing purposes.

### Cost Considerations

Despite the free tier, it's important to monitor AWS Lambda usage to avoid unexpected costs. Ideally, always track consumption and set up alerts to be notified if usage exceeds the free limit.

### How to Create a Budget in AWS Budgets

To ensure you keep costs under control, it's recommended to set up a budget in AWS. AWS Budgets allows you to set a spending limit and receive alerts when usage approaches that limit, helping to avoid surprises on your bill.

1. **Access AWS Budgets**  
   In the AWS Console, go to the "Billing" section and click on "Budgets" in the left menu.

2. **Create a New Budget**  
   Click on "Create a budget" and select the budget type ("Cost Budget") to set a limit based on value.

3. **Set the Value and Alert Settings**  
   Enter the limit value you want for the month and configure email notifications to receive alerts when consumption reaches, for example, 80% and 100% of the budget.

With these settings, you can use AWS Lambda more safely, taking advantage of the free tier without risking unexpected costs.

## Setting Up the Environment

To deploy our API on AWS Lambda, we need to configure some essential tools. We will install AWS CLI and AWS SAM, as well as create a user with appropriate permissions in AWS.

### 1. Installing AWS CLI
AWS CLI is a command-line tool that allows you to interact with AWS services directly from the terminal.

To install AWS CLI, follow the instructions in the official documentation:

[Installation Documentation for AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

After installation, verify that AWS CLI was installed correctly with the command:

```bash
aws --version
```

### 2. Installing AWS SAM

AWS SAM can be considered a form of Infrastructure as Code (IaC). With it, you describe the infrastructure of your serverless application using a YAML configuration file. This allows you to version and automate the deployment process of the infrastructure in a consistent and repeatable manner, without the need for manual configurations.

To install AWS SAM, follow the instructions in the official documentation:

[Installation Documentation for AWS SAM](https://docs.aws.amazon.com/pt_br/serverless-application-model/latest/developerguide/install-sam-cli.html)


### 3. Configuring a User in AWS

To perform the deployment, we will need a user in AWS with the appropriate permissions.
1. Access the IAM console in AWS.
2. Click on `Add user`.
3. Choose a name for the user and check the `Programmatic access` option.
4. In `Permissions`, select `Attach policies directly` and choose the `Administrator` policy.
5. Complete the creation and save the access credentials.

Now, configure AWS CLI with the command below and enter the user's credentials:
```bash
aws configure
```

## Creating the REST API with Golang

### 1. Starting the Project
For this project, we will use `aws-lambda-go-api-proxy`, a library that facilitates the creation of REST APIs in Golang, allowing you to use frameworks like `Gin` or `Fiber` to develop your APIs in a more structured and efficient way. The great advantage is that, with a few lines of code, you can switch between using the local framework and AWS Lambda, making migration easier if you decide to stop using Lambda in the future.

You can access the official project documentation at this link:  
[aws-lambda-go-api-proxy Documentation](https://github.com/awslabs/aws-lambda-go-api-proxy)

In this post, we will show an example using the Fiber framework.

### 2. Creating the API with Golang and Fiber

In the `main.go` file, add the following code to create a simple route that returns a JSON with the message "Hello, World!":
```golang
// main.go
package main

import (
	"context"
	"log"
	"os"

	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
	fiberadapter "github.com/awslabs/aws-lambda-go-api-proxy/fiber"
	"github.com/gofiber/fiber/v2"
	"github.com/joho/godotenv"
)

var fiberLambda *fiberadapter.FiberLambda
var app *fiber.App

func init() {
	err := godotenv.Load()
	if err != nil {
		log.Fatalf("error loading .env file: %v", err)
	}

	app = fiber.New()

	app.Get("/", func(c *fiber.Ctx) error {
		return c.SendString("Hello, World!")
	})

	fiberLambda = fiberadapter.New(app)
}

func Handler(ctx context.Context, req events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
	return fiberLambda.ProxyWithContext(ctx, req)
}

func main() {
	if os.Getenv("LAMBDA_ENABLED") == "true" {
		lambda.Start(Handler)
	} else {
		err := app.Listen(":3000")
		if err != nil {
			panic(err)
		}
	}
}
```

Don't forget to run `go mod init` to initialize the Go module and `go mod tidy` to install the dependencies.

### 3. Creating the SAM Template File

Now, let's create the `template.yaml` file to configure the deployment on AWS Lambda. The content of the file will be as follows:

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 5
    MemorySize: 128
    Environment:
      Variables:
        LAMBDA_ENABLED: true
        
  Api:
    Cors:
      AllowMethods: "'GET,POST,OPTIONS'"
      AllowHeaders: "'Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token'"
      AllowOrigin: "'*'"

Resources:
  Fiber:
    Type: AWS::Serverless::Function
    Metadata:
      BuildMethod: go1.x
    Properties:
      CodeUri: .
      Handler: bootstrap
      Runtime: provided.al2023
      Events:
        GetResource:
          Type: Api
          Properties:
            Path: /{proxy+}
            Method: any

Outputs:
  ApiUrl:
    Description: "URL for the API endpoint"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/"
```

This file defines the Lambda function and the API Gateway configuration to expose your API. It will be used to deploy the application on AWS Lambda.

### 4. Repository with the Complete Code
This complete application example can be found in our GitHub repository here.
[Example application with Golang and AWS Lambda](https://github.com/cmparrela/ddev-api-rest-golang-lambda)

### 5. Preparing for Deployment with AWS SAM
To build the application, run:
```bash
sam build
```

To perform the first deployment, use the --guided command to configure the deployment interactively:
```bash
sam deploy --guided
```

At the end of the process, you will be able to see the URL of your API in the terminal. Like this:

```bash
Key                 ApiUrl                                                                                              
Description         URL for the API endpoint                                                                            
Value               https://xq5vqngao8.execute-api.us-east-1.amazonaws.com/Prod/ 
```

In future deployments, you can use the command:
```bash
sam build && sam deploy
```