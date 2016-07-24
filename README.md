# AWS Code Deploy

## EC2 Instance

Launch an EC2 instance with the IAM role that has S3 Full Access.

This EC2 instance will be running Apache HTTP Server to serve a test webpage. And we are going to deploy a webpage to this instance using AWS CodeDeploy.

When launching an EC2 instance, enter the following bash script into `User Data` section:

```bash
#!/bin/bash
yum -y update
yum install -y ruby aws-cli
yum install -y httpd
chkconfig httpd on
service httpd start
cd /home/ec2-user
aws s3 cp s3://aws-codedeploy-us-east-1/latest/install . --region us-east-1
chmod +x ./install
./install auto
```

Note that we are using `aws-codedeploy-us-east-1` for instances in the US East (N. Virginia) region.

Also, tag the instance with `Name=test-website`. This tag will be later used to tell AWS CodeDeploy to which instance(s) the application should be deployed.

## Preparation

### Service Role

Service roles are used to grant permissions to an AWS service so it can access AWS resources. The policies that you attach to the service role determine which AWS resources the service can access and what it can do with those resources.

The service role you create for AWS CodeDeploy must be granted the permissions to access the instances to which you will deploy applications. 

Create a text file named `service-role.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "codedeploy.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Create a service role:

```bash
aws iam create-role \
  --role-name=CodeDeployServiceRole \
  --assume-role-policy-document=file://service-role.json \
  --profile=dchun
```

give the service role the permissions based on the IAM managed policy named `AWSCodeDeployRole`:

```bash
aws iam attach-role-policy \
  --role-name=CodeDeployServiceRole \
  --policy-arn=arn:aws:iam::aws:policy/service-role/AWSCodeDeployRole \
  --profile=dchun
```

Get the ARN of the service role:

```bash
aws iam get-role \
  --role-name=CodeDeployServiceRole \
  --query="Role.Arn" \
  --output=text \
  --profile=dchun
```

### S3

Create a S3 bucket from which AWS CodeDeploy can deploy your application.

```bash
aws s3 mb s3://dchun-codedeploy --profile=dchun
```

### CodeDeploy

`create-application` command to register a new application

```bash
aws deploy create-application \
  --application-name=test-website \
  --profile=dchun
```

Create a deployment group:

```bash
aws deploy create-deployment-group \
  --application-name=test-website \
  --deployment-group-name=test-website-deploy-group \
  --deployment-config-name=CodeDeployDefault.OneAtATime \
  --ec2-tag-filters Key=Name,Value=test-website,Type=KEY_AND_VALUE \
  --service-role-arn=arn:aws:iam::713746723246:role/CodeDeployServiceRole \
  --profile=dchun
```

`push` command to bundle the files together, upload the revisions to Amazon S3, and register information with AWS CodeDeploy about the uploaded revision, all in one action.

```bash
aws deploy push \
  --application-name=test-website \
  --s3-location=s3://dchun-codedeploy/test-website.zip \
  --ignore-hidden-files \
  --profile=dchun
```

Finally, `create-deployment` command to deploy the application to the designated EC2 instance(s):

```bash
aws deploy create-deployment \
  --application-name test-website \
  --deployment-config-name CodeDeployDefault.OneAtATime \
  --deployment-group-name test-website-deploy-group \
  --s3-location bucket=dchun-codedeploy,bundleType=zip,key=test-website.zip \
  --profile dchun
```

## Verification

Get the deployment's ID:

```bash
aws deploy list-deployments \
  --application-name test-website \
  --deployment-group-name test-website-deploy-group \
  --query 'deployments' \
  --output text \
  --profile dchun
```

Get the deployment's overall status:

```bash
aws deploy get-deployment \
  --deployment-id d-RX4CG0ZXG \
  --query 'deploymentInfo.status' \
  --output text \
  --profile dchun
```

## Re-deploy

Note that this time we bundle everything into a different zip file named `test-website-v2.zip`.

```bash
aws deploy push \
  --application-name=test-website \
  --s3-location=s3://dchun-codedeploy/test-website-v2.zip \
  --ignore-hidden-files \
  --profile=dchun
```

```bash
aws deploy create-deployment \
  --application-name test-website \
  --deployment-config-name CodeDeployDefault.OneAtATime \
  --deployment-group-name test-website-deploy-group \
  --s3-location bucket=dchun-codedeploy,bundleType=zip,key=test-website-v2.zip \
  --profile dchun
```

## Clean Up

Delete the S3 bucket:

```bash
aws s3 rm s3://dchun-codedeploy --recursive --profile dchun
```

Delete the CodeDeploy setup:

```bash
aws deploy delete-application \
  --application-name test-website \
  --profile dchun
```

Terminate any EC2 instances manually created.
