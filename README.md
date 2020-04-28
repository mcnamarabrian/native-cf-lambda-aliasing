# native-cf-lambda-aliasing

# Introduction

The purpose of this project is to illustrate how to shift traffic between different versions of the same AWS Lambda function using [AWS CloudFormation](https://aws.amazon.com/cloudformation/).  It is possible to shape traffic using different methods:

* Using the AWS CLI *lambda* sub-command to **update-function-code**, **publish-version**, and **update-alias**.

* Using the native integration between the *Serverless Application Model (SAM)* and *AWS CodeDeploy* to automate the continued deployment.

This project does not automate the deployment.  There are three separate CloudFormation templates that are used to move application v0 to application v2.

# Prerequisites

* [aws-cli](https://aws.amazon.com/cli/)

# Deployment steps

## Deploy and test v0 of the application

The initial Lambda function is quite simple.  It deploys a function that prints 'Hello world' and returns the string 'Hello world' to the caller.  There is a new version defined with this function (v1) and a new alias (PROD) that directs all invokes to v1 of the function.

### Package the function

```bash
aws cloudformation package \
    --template-file template-v0.yml \
    --output-template-file packaged-template-v0.yml \
    --s3-bucket your_s3_bucket_to_hold_artifacts
```

### Deploy the function

```bash
aws cloudformation deploy \
    --template-file packaged-template-v0.yml \
    --stack-name versionAliasTest \
    --capabilities CAPABILITY_IAM
```

### Test the function

First, determine the ARN of the function alias in the CloudFormation stack.

```bash
HELLO_ALIAS=$(aws cloudformation describe-stack-resource --stack-name versionAliasTest --logical-resource-id HelloAlias --query "StackResourceDetail.PhysicalResourceId" --output text)
```

Once the function alias has been identified, invoke the function.  It should only ever return and print 'Hello world'.

```bash
aws lambda invoke \
    --function-name ${HELLO_ALIAS} \
    --payload '{}' \
    response.json
```

## Deploy and test v1 of the application

The template deploys an update to our Lambda function.  It prints 'Hola mundo' and returns the string 'Hola mundo' to the caller.  There is a new version defined with this function (v2).  The alias (PROD) directs 80% of traffic to v1 of the function and 20% to v2 of the function.

### Package the function

```bash
aws cloudformation package \
    --template-file template-v1.yml \
    --output-template-file packaged-template-v1.yml \
    --s3-bucket your_s3_bucket_to_hold_artifacts
```

### Deploy the function

```bash
aws cloudformation deploy \
    --template-file packaged-template-v1.yml \
    --stack-name versionAliasTest \
    --capabilities CAPABILITY_IAM
```

### Test the function

First, determine the ARN of the function alias and function name in the CloudFormation stack.

```bash
HELLO_ALIAS=$(aws cloudformation describe-stack-resource --stack-name versionAliasTest --logical-resource-id HelloAlias --query "StackResourceDetail.PhysicalResourceId" --output text)
HELLO_FUNCTION_NAME=$(aws cloudformation describe-stack-resource --stack-name versionAliasTest --logical-resource-id HelloFunction --query "StackResourceDetail.PhysicalResourceId" --output text)
```

Once the function alias has been identified, examine the function alias configuration. The Alias is routing 20% of the traffic to `v2` of the function and 80% of the traffic to `v1`.

```bash
aws lambda get-alias \
    --function-name ${HELLO_FUNCTION_NAME} \
    --name PROD
```

Invoke the function.  The `ExecutedVersion` should be `1` 80% of the time and `2` 20% of the time.  The file `response.json` will contain 'Hello world' 80% of the time and 'Hola mundo' 20% of the time.  

```bash
aws lambda invoke \
    --function-name ${HELLO_ALIAS} \
    --payload '{}' \
    response.json
```

## Deploy and test v2 of the application

The template updates our function alias (PROD) and directs 100% of traffic to v2 of the function.

### Package the function

```bash
aws cloudformation package \
    --template-file template-v2.yml \
    --output-template-file packaged-template-v2.yml \
    --s3-bucket your_s3_bucket_to_hold_artifacts
```

### Deploy the function

```bash
aws cloudformation deploy \
    --template-file packaged-template-v2.yml \
    --stack-name versionAliasTest \
    --capabilities CAPABILITY_IAM
```

### Test the function

First, determine the ARN of the function alias and function name in the CloudFormation stack.

```bash
HELLO_ALIAS=$(aws cloudformation describe-stack-resource --stack-name versionAliasTest --logical-resource-id HelloAlias --query "StackResourceDetail.PhysicalResourceId" --output text)
HELLO_FUNCTION_NAME=$(aws cloudformation describe-stack-resource --stack-name versionAliasTest --logical-resource-id HelloFunction --query "StackResourceDetail.PhysicalResourceId" --output text)
```

Once the function alias has been identified, examine the function alias configuration. The Alias is routing 100% of the traffic to `v2` of the function and 0% of the traffic to `v1`.

```bash
aws lambda get-alias \
    --function-name ${HELLO_FUNCTION_NAME} \
    --name PROD
```

Invoke the function.  The `ExecutedVersion` should be `2` 100% of the time and `1` 0% of the time.  The file `response.json` will contain 'Hello world' 0% of the time and 'Hola mundo' 100% of the time.  

```bash
aws lambda invoke \
    --function-name ${HELLO_ALIAS} \
    --payload '{}' \
    response.json
```

# References

* [AWS Blog - Implementing Canary Deployments of AWS Lambda Functions with Alias Traffic Shifting](https://aws.amazon.com/blogs/compute/implementing-canary-deployments-of-aws-lambda-functions-with-alias-traffic-shifting/) and the [code example](https://github.com/aws-samples/aws-lambda-deploy)

* [Deploying Serverless Applications Gradually](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/automating-updates-to-serverless-apps.html)

* [AWS Blog - Implementing safe AWS Lambda deployments with AWS CodeDeploy](https://aws.amazon.com/blogs/compute/implementing-safe-aws-lambda-deployments-with-aws-codedeploy/) and the [code example](https://github.com/aws-samples/aws-safe-lambda-deployments).