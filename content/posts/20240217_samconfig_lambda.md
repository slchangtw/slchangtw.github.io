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

The `template.yaml` file located in the root directory of the project contains the configuration used by AWS CloudFormation, which is a service to set up Lambda function. The file is written in YAML format and contains the following sections:

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

A basic flow of deployment by SAM follows threes steps: 
1. build: preparing the deployment,
2. package: creating a `zip` file and uploading it to S3,
3. deploy: deploying the package to AWS.

If you check the documentation, you will find that for each you can specify different flags and paramerters. For example, `sam build --use-container` will build the Lambda function inside a Docker container.

However, when having multiple parameters and flags, you may mess up the command and forget some of them. That's where the SAM configuration file coming to rescue. To store a set of flags for each command, you can simply replace the dash with an underscore of the flags and store them in the `samconfig.toml` file. The basic syntax of the file is as follows:

```
version = 0.1
[environment]
[environment.command]
[environment.command.parameters]
flag = parameter value
```

Here is an example of the `samconfig.toml` file for different commands and their parameters:

```toml
version = 0.1

# A
[default]
[default.global.parameters]
s3_bucket = "aws-lambda-deployment-test"
s3_prefix = "get-cat-image"

# B
[default.build.parameters]
cached = true
parallel = true
use_container = true
skip_pull_image = true

# C
[default.package.parameters]
output_template_file = "packaged.yaml"

# D
[default.deploy.parameters]
stack_name = "get-cat-image"
capabilities = "CAPABILITY_IAM"
template_file = "packaged.yaml"
confirm_changeset = true
```

Let's go through the sections of the `samconfig.toml` file:
A. The `samconfig.toml` uses `default` as the default environment name. In future posts, we will specify different sets of flags for different environments such as testing and production. Besides, the `s3_bucket` and `s3_prefix` are the parameters used by the `package` and `deploy` commands to specify the S3 bucket and prefix for the deployment package.

B. The `build` section contains the parameters for the `sam build` command. In this case, we set the `cached`, `parallel`, `use_container`, and `skip_pull_image` flags to `true`. 
- The `cached` flag specifies whether using cache to speed up the build process. 
- The `parallel` flag the specifies whether the functions (if there are many) can be built in parallel. 
- The `use_container` flag specifies whether the Lambda function is built by Docker container.
- The `skip_pull_image=true` flag specifies whether the command should skip pulling the latest Docker image for the Lambda runtime.

C. The `package` section contains the parameters for the `sam package` command. In this case, we set the `output_template_file` flag to `packaged.yaml`, and `output_template_file` flag specifies the name of the output template file.

D. The `deploy` section contains the parameters for the `sam deploy` command. In this case, we set the `stack_name`, `capabilities`, `template_file`, and `confirm_changeset` flags. 
- The `stack_name` flag specifies the name of the CloudFormation stack. 
- The `capabilities` flag specifies the capabilities that must be specified when the stack is created. 
- The `template_file` flag specifies the name of the template file, which is the output of the `package` command.
- The `confirm_changeset` flag specifies whether to confirm the changeset before executing the deployment. If set to `true`, the command will prompt you to confirm the changeset before executing the deployment.

# Deploy the Lambda function



