---
layout: post
title:  "Deploy Docker images pushed to AWS Elastic Container Registry"
author: Vincent Lesierse
date:   2018-03-18
tags: [docker,aws,aws-ecs]
comments: true
cover: aws.png
---
Using containers in AWS is very easy using ECS. When EKS arrives, there will be even more options for you to choose from. You create a cluster, task definitions and services and ECS figures our where to run your container on a EC2 instance. I even takes care of the Application Load Balancer. However, for deployment the options are quite limited. Of course you can control everything with CloudFormation or Terraform, but how does the automation work in practice? Code Pipeline supports the whole from Code to Deployment flow, but how do you deploy a new container when you get the image pushed to the AWS Elastic Container Registry and the source and creation of the image takes place somewhere else.

## ecr-deploy
**ecr-deploy** is an AWS Lambda function which you can deploy into you AWS account. It's basic function is to deploy you container to the ECS cluster if the image is pushed to ECR.

When you install **ecr-deploy** in your account it will automatically create an AWS CloudWatch Event Rule which will listen for `PutImage` API calls to ECR. Whenever you push an image using `docker push`, ECR will call this API get the data in the registry. The event includes all the information about the image, tag and even the manifest with all the layers.

```json
{
    "version": "0",
    "id": "597eb076-4eb8-4d4b-b019-203b18a55cce",
    "detail-type": "AWS API Call via CloudTrail",
    "source": "aws.ecr",
    "account": "1234567890",
    "time": "2018-03-18T00:00:00Z",
    "region": "eu-west-1",
    "resources": [],
    "detail": {
        "eventVersion": "1.04",
        "userIdentity": {},
        "eventTime": "2018-03-18T00:00:00Z",
        "eventSource": "ecr.amazonaws.com",
        "eventName": "PutImage",
        "awsRegion": "eu-west-1",
        "sourceIPAddress": "prod.frontend.ecr.aws.internal",
        "userAgent": "prod.frontend.ecr.aws.internal",
        "requestParameters": {
            "repositoryName": "busybox",
            "imageTag": "v1",
            "registryId": "123456789012",
            "imageManifest": "{}"
        },
        "responseElements": {
            "image": {
                "repositoryName": "busybox",
                "imageManifest": "{}",
                "registryId": "123456789012",
                "imageId": {
                    "imageDigest": "sha256:eaa261b73dcc87f1a1880a6320a06867fcb6db77b4bb6139cf9b9059c9af95eb",
                    "imageTag": "v1"
                }
            }
        },
        "requestID": "d99b47db-1b3c-11e8-a2b6-b1dd3cc73df2",
        "eventID": "04caef66-e52b-4c4d-bbb5-547b85fec1f8",
        "resources": [
            {
                "ARN": "arn:aws:ecr:eu-west-1:123456789012:repository/busybox",
                "accountId": "123456789012"
            }
        ],
        "eventType": "AwsApiCall"
    }
}
```
When the event is triggered, CloudWatch will push it toward the Lambda function. The function is created in Python and will get the required information out of the event. It's interested in the region, registry, image (repository) and tag.

First it will look for active task definitions which are referencing the image. A new version of those task definitions will be created with the new version tag. 

Next it will look at active services which are referencing the task definitions. Those services will be updated with the new revision. ECS will take care of the deployment. When you specified a min capacity of 50% for your service it will perform a rolling deployment. When 100% is specified it will execute a blue/green deployment. The health checks will make sure that if a container instance fails during deployment, the previous revision of the task definition will be used.

## Usage
If this sounds a good solution for deploying your containers in AWS ECS, then you most likely give it a try.

### Install from AWS Serverless Application Repository
AWS recently introduced the Serverless Application Repository which contains all kind of different Lambda functions. I've pushed **ecr-deploy** to the repository. You can find the Lambda following the link. [AWS Serverless Application Repository](https://serverlessrepo.aws.amazon.com/applications/arn:aws:serverlessrepo:us-east-1:109625914275:applications~ecr-deploy)

You can deploy the Lambda to your account by clicking the Deploy button.

> Please be aware that you have to change the Lambda's execution role and attach the `AmazonEC2ContainerServiceFullAccess` policy. At the moment of writing it's only possible to use policy templates with the repository and there is no template for ECS.

### Install from Git Repository
**ecr-deploy** is build using Python and the AWS Serverless Application Model. This allows you to deploy the Lambda using the AWS CLI.

Clone the repository from GitHub and get the latest version.
```sh
git clone https://github.com/vlesierse/aws-ecs-automation.git
cd ecr-deploy
```

Prepare the repository for deployment. Like I mentioned before, this Lambda is written using Python. You have to create a Virtual Environment using [`Virtualenv`](https://virtualenv.pypa.io/en/stable), active you console and install the required packages.
```sh
virtualenv -p python3 --no-site-packages --distribute .env && source .env/bin/activate && pip install -r requirements.txt
```

When you would like to deploy a Lambda you have to push it to an AWS S3 bucket first. You can easily do this using the AWS Command Line Interface.
```sh
aws s3 mb s3://your-aws-sam-bucket
```

Before deploying the Lambda you have to package it first. This will transform your SAM template file to a CloudFormation template and reference the created ZIP file which contains your code.
```sh
aws cloudformation package --template-file template.yml --output-template-file template-packaged.yml --s3-bucket your-aws-sam-bucket
```

Finally you can deploy the CloudFormation template which will create the required AWS resources and the Lambda.
```sh
aws cloudformation deploy --capabilities CAPABILITY_IAM --template-file template-packaged.yml --stack-name <YOUR STACK NAME> --parameter-overrides Cluster=<YOUR CLUSTER>
```

## Next steps
If you think that **ecr-deploy** solves the same problem you have when working with containers in AWS, please give it a try. If your use case is sightly different or your are experiencing problems with the Lambda function, please provide feedback in the GitHub repository or send me pull request.

[GitHub](https://github.com/vlesierse/aws-ecs-automation)
