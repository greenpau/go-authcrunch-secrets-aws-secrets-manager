# go-authcrunch-secrets-aws-secrets-manager

<a href="https://github.com/greenpau/go-authcrunch-secrets-aws-secrets-manager/actions/" target="_blank"><img src="https://github.com/greenpau/go-authcrunch-secrets-aws-secrets-manager/workflows/build/badge.svg?branch=main"></a>
<a href="https://pkg.go.dev/github.com/greenpau/go-authcrunch-secrets-aws-secrets-manager" target="_blank"><img src="https://img.shields.io/badge/godoc-reference-blue.svg"></a>

[AuthCrunch](https://github.com/greenpau/go-authcrunch) Secrets Plugin
for AWS Secrets Manager Integration.

<!-- begin-markdown-toc -->
## Table of Contents

* [Getting Started](#getting-started)
  * [IAM Policy and Role](#iam-policy-and-role)
  * [Password Hashing](#password-hashing)
  * [Secrets Management](#secrets-management)
    * [User Credentials](#user-credentials)
    * [Access Token Secret](#access-token-secret)

<!-- end-markdown-toc -->

## Getting Started

### IAM Policy and Role

In "IAM" section of AWS Console, create `AuthCrunchSecretsManagerAccess` IAM Policy.
Change `123456789012` with your own account number.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "secretsmanager:ListSecrets",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetResourcePolicy",
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret",
                "secretsmanager:ListSecretVersionIds"
            ],
            "Resource": "arn:aws:secretsmanager:*:123456789012:secret:authcrunch*"
        },
        {
            "Effect": "Allow",
            "Action": "secretsmanager:GetRandomPassword",
            "Resource": "*"
        }
    ]
}
```

Next, create a role for "EC2" AWS Service and attach the `AuthCrunchSecretsManagerAccess` to it.

Then, attach the IAM role to an EC2 instance.

From the EC2 instance, run the following command to list secrets.

```bash
aws secretsmanager list-secrets
```

The output for an account without any existing secrets follows.

```json
{
    "SecretList": []
}
```

### Password Hashing

Install `bcrypt-cli` for password hashing:

```bash
go install github.com/bitnami/bcrypt-cli@latest
```

Install `pwgen` for password generation:

```bash
sudo yum -y install pwgen
```

Generate a password:

```
$ pwgen -cnvB1 32
rbrH97m9bpbk3qRphHFNM9ksJfRcWdvr
```

Next, hash the `password` and the `api_key`:

```
$ echo -n "rbrH97m9bpbk3qRphHFNM9ksJfRcWdvr" | bcrypt-cli -c 10
$2a$10$iqq53VjdCwknBSBrnyLd9OH1Mfh6kqPezMMy6h6F41iLdVDkj13I6
```

Repeat the same thing for `api_key`.

```
$ pwgen -cnvB1 32
kqvc7cgk44dtpX9nXx4NL9krH4g7fqdJ
$ echo -n "kqvc7cgk44dtpX9nXx4NL9krH4g7fqdJ" | bcrypt-cli -c 10
$2a$10$TEQ7ZG9cAdWwhQK36orCGOlokqQA55ddE0WEsl00oLZh567okdcZ6
```

### Secrets Management

#### User Credentials

First, create a set of credentials for a management user, `jsmith`.

Next, browse to "AWS Secrets Manager" to add a secret by selecting secret
type as "Other type of secret (API key, OAuth token, other.)" and put
the following key values.

* `username`: `jsmith`
* `password`: `bcrypt:10:$2a$10$iqq53VjdCwknBSBrnyLd9OH1Mfh6kqPezMMy6h6F41iLdVDkj13I6`
* `api_key`: `bcrypt:10:$2a$10$TEQ7ZG9cAdWwhQK36orCGOlokqQA55ddE0WEsl00oLZh567okdcZ6`
* `email`: `jsmith@localhost.localdomain`
* `name`: 'John Smith`

Use `aws/secretsmanager` for the "Encryption key".

Set secret name to `authcrunch/caddy/users/jsmith` and description
to `Caddy User Credentials for jsmith`

After the creation of the secret, list secrets with `aws secretsmanager list-secrets` again.

```json
{
    "SecretList": [
        {
            "Name": "authcrunch/caddy/users/jsmith",
            "Tags": [],
            "LastChangedDate": 1673135119.189,
            "SecretVersionsToStages": {
                "278a2e61-f3e3-4280-a444-333d7186d5ce": [
                    "AWSCURRENT"
                ]
            },
            "CreatedDate": 1673135119.15,
            "ARN": "arn:aws:secretsmanager:us-east-1:123456789012:secret:authcrunch/caddy/users/jsmith-tz6d06",
            "Description": "Caddy User Credentials for jsmith"
        }
    ]
}
```

Next, retrieve the previously created secret:

```bash
aws secretsmanager get-secret-value --secret-id authcrunch/caddy/users/jsmith
```

The expected output follows:

```json
{
    "Name": "authcrunch/caddy/users/jsmith",
    "VersionId": "278a2e61-f3e3-4280-a444-333d7186d5ce",
    "SecretString": "{\"username\":\"jsmith\",\"password\":\"bcrypt:10:$2a$10$iqq53VjdCwknBSBrnyLd9OH1Mfh6kqPezMMy6h6F41iLdVDkj13I6\",\"api_key\":\"bcrypt:10:$2a$10$TEQ7ZG9cAdWwhQK36orCGOlokqQA55ddE0WEsl00oLZh567okdcZ6\"}",
    "VersionStages": [
        "AWSCURRENT"
    ],
    "CreatedDate": 1673135119.183,
    "ARN": "arn:aws:secretsmanager:us-east-1:123456789012:secret:authcrunch/caddy/users/jsmith-tz6d06"
}
```

#### Access Token Secret

Next, follow the same proceduce as with `jsmith` to generate the shared key
to be used for JWT token signing and verification.

Browse to "AWS Secrets Manager" to add a secret by selecting secret
type as "Other type of secret (API key, OAuth token, other.)" and put
the following key values.

* `id`: `0`
* `usage`: `sign-verify`
* `value`: `b006d65b-c923-46a1-8da1-7d52558508fe`

Use `aws/secretsmanager` for the "Encryption key".

Set secret name to `authcrunch/caddy/access_token` and description
to `Caddy Access Token Secret`
