# AWS Code Deploy

## Service Role

service roles are used to grant permissions to an AWS service so it can access AWS resources. The policies that you attach to the service role determine which AWS resources the service can access and what it can do with those resources.

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

## EC2 Instance

Launch an EC2 instance with the IAM role that has S3 Full Access.

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

## S3

`bucket-name` represents one of the following:

- `aws-codedeploy-us-east-1` for instances in the US East (N. Virginia) region
- `aws-codedeploy-us-west-1` for instances in the US West (N. California) region
- `aws-codedeploy-us-west-2` for instances in the US West (Oregon) region
- `aws-codedeploy-ap-south-1` for instances in the Asia Pacific (Mumbai) region
- `aws-codedeploy-ap-northeast-2` for instances in the Asia Pacific (Seoul) region
- `aws-codedeploy-ap-southeast-1` for instances in the Asia Pacific (Singapore) region
- `aws-codedeploy-ap-southeast-2` for instances in the Asia Pacific (Sydney) region
- `aws-codedeploy-ap-northeast-1` for instances in the Asia Pacific (Tokyo) region
- `aws-codedeploy-eu-central-1` for instances in the EU (Frankfurt) region
- `aws-codedeploy-eu-west-1` for instances in the EU (Ireland) region
- `aws-codedeploy-sa-east-1` for instances in the South America (São Paulo) region

```bash
aws s3 mb s3://dchun-codedeploy --profile=dchun
```

## Prep

`create-application` command to register a new application

```bash
aws deploy create-application \
  --application-name=test-website \
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



```bash
aws deploy create-deployment-group \
  --application-name=test-website \
  --deployment-group-name=test-website-deploy-group \
  --deployment-config-name=CodeDeployDefault.OneAtATime \
  --ec2-tag-filters Key=Name,Value=test-website,Type=KEY_AND_VALUE \
  --service-role-arn=arn:aws:iam::713746723246:role/CodeDeployServiceRole \
  --profile=dchun
```

```bash
aws deploy create-deployment \
  --application-name test-website \
  --deployment-config-name CodeDeployDefault.OneAtATime \
  --deployment-group-name test-website-deploy-group \
  --s3-location bucket=dchun-codedeploy,bundleType=zip,key=test-website.zip \
  --profile dchun
```
