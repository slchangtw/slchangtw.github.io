---
title: "Configuring AWS Lambda Deployment via  AWS SAM"
date: 2024-02-17T20:58:54+01:00
draft: true
---

AWS Lambda is a serverless computing service provided by Amazon Web Services. It is a compute service that runs code in response to events and automatically manages the compute resources required by that code. AWS SAM (Serverless Application Model) is an open-source framework for building serverless applications. It provides a simplified way of defining the AWS services needed by your serverless application. In this post, we will see how to configure AWS Lambda deployment via AWS SAM configuration file.

<!--more-->

# Prerequisites

1. AWS credentials configuration: You need to have AWS credentials configured on your local machine. If you haven't done so, you can follow the instructions in the [official documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html).

2. AWS SAM CLI: You need to have the AWS SAM CLI installed on your local machine. If you haven't done so, you can follow the instructions in the [official documentation](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html).

3. Docker Desktop: You need to have Docker Desktop installed on your local machine. If you haven't done so, you can follow the instructions in the [official documentation](https://docs.docker.com/desktop/).

# Create a new AWS Lambda project from Template

To start, you can use the `sam init` command to initialize a new AWS Lambda project. You can choose either from the available templates offered by AWS or use a custom template. The following command initializes a new AWS Lambda project using a template created by me and located on GitHub. 

```bash
mkdir my-lambda-project && cd my-lambda-project # Create a new directory and navigate to it
sam init —-location gh:slchangtw/lambda_template_example # Use --location option to specify the location of the template
```

The custom template can locate online or locally. Here are examples for both cases [https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/using-sam-cli-init.html#using-sam-cli-init-examples](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/using-sam-cli-init.html#using-sam-cli-init-examples). Note that if the template is located on GitHub, it must be in the first level of the repository.

# Code of the Lambda function

Before we deploy the Lambda function, let's take a look at the code of the Lambda function. Firstly we navigate to the `src` directory of the newly created project, you will find a file named `app.py`. This file contains the code of the Lambda function. The function makes a request to The Cat API and returns the URL of a random cat image. DON'T forget to add 3rd party libraries to the `requirements.txt` file located in the directory to use them in the Lambda function.


```python
import requests


def lambda_handler(event, context):
    try:
        # Making a request to The Cat API
        response = requests.get("https://api.thecatapi.com/v1/images/search")

        # Checking if request was successful
        if response.status_code == 200:
            data = response.json()
            image_url = data[0]["url"]
            return {"statusCode": 200, "body": image_url}
        else:
            return {
                "statusCode": response.status_code,
                "body": f"Failed to fetch cat image: {response.text}",
            }
    except Exception as e:
        return {"statusCode": 500, "body": f"An error occurred: {str(e)}"}
```

# Lambda template configuration

The `template.yaml` file located in the root directory of the project contains the configuration used by AWS CloudFormation to set up Lambda function. The file is written in YAML format and contains the following sections:

```yaml
# A
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Get Cat Image API

# B
Globals:
  Function:
    Timeout: 3
    MemorySize: 128

Resources:
  # C
  LambdaExecRoles:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - lambda.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      ManagedPolicyArns:
        - "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"

  # D 
  GetCatImageFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: app.lambda_handler
      Runtime: python3.10
```

Let's go through the sections of the `template.yaml` file:

A. The `AWSTemplateFormatVersion` and `Transform` fields are required. The `AWSTemplateFormatVersion` field specifies the capabilities of the template. The `Transform` field specifies the macro that AWS CloudFormation uses to process the template.

B. The `Globals` section contains the default settings for all functions in the template. In this case, we set the `Timeout` and `MemorySize` for all functions.

C. The `Resources` section contains the resources that will be created by AWS CloudFormation. In this case, we define an IAM role to allow the execution of the Lambda function.

D. The `GetCatImageFunction` resource defines the Lambda function. The `CodeUri` field specifies the directory where the Lambda function code is located. The `Handler` field specifies the name of the function that AWS Lambda will call to start execution. The `Runtime` field specifies the runtime environment for the Lambda function.

# SAM configuration file

