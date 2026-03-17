# Jenkins IAM Roles Anywhere

Keyless AWS authentication from an on-premises Jenkins instance using
IAM Roles Anywhere and a self-managed PKI — no static IAM access keys anywhere.

## What This Solves

The standard pattern for Jenkins-to-AWS auth is to create an IAM user, generate an
access key, and store it as a Jenkins credential. That key never rotates, lives in
plaintext in Jenkins, and is a credential breach waiting to happen.

IAM Roles Anywhere replaces that entirely. Jenkins presents an X.509 certificate
signed by a trusted CA. AWS validates the cert, issues short-lived STS credentials,
and Jenkins gets a session that expires in an hour. No long-lived keys anywhere.

## Architecture

```
Jenkins (docker01, on-prem)
  |
  |-- aws_signing_helper (reads cert + private key)
  |       |
  |       v
  |   IAM Roles Anywhere Trust Anchor (JenkinsLabCA)
  |       |
  |       v
  |   IAM Roles Anywhere Profile (JenkinsProfile)
  |       |
  |       v
  |   STS AssumeRoleWithWebIdentity
  |       |
  v       v
  JenkinsLabRole (IAM Role)
        |
        v
  AWS Resources (S3, EC2, etc.)
```

## Components

| Component | Details |
|---|---|
| Trust Anchor | JenkinsLabCA — self-managed CA cert uploaded to IAM Roles Anywhere |
| IAM Role | JenkinsLabRole — scoped permissions for CI/CD operations |
| Profile | JenkinsProfile — maps cert subject to role assumption |
| Signing Helper | aws_signing_helper binary — handles STS credential exchange |
| Jenkins Host | docker — Ubuntu VM running Jenkins in Docker |

## Custom Dockerfile

The Jenkins image bakes in the AWS CLI and aws_signing_helper:

```dockerfile
FROM jenkins/jenkins:lts
USER root
RUN apt-get update && apt-get install -y awscli curl
RUN curl -Lo /usr/local/bin/aws_signing_helper \
    https://rolesanywhere.amazonaws.com/releases/1.0.4/X86_64/Linux/aws_signing_helper && \
    chmod +x /usr/local/bin/aws_signing_helper
USER jenkins
```

## Usage

Authenticate from a Jenkins pipeline step:

```groovy
stage('AWS Auth') {
    steps {
        sh '''
            aws_signing_helper credential-process \
                --certificate /var/jenkins_home/certs/jenkins.crt \
                --private-key /var/jenkins_home/certs/jenkins.key \
                --trust-anchor-arn arn:aws:rolesanywhere:us-east-1:YourAccout#:trust-anchor/... \
                --profile-arn arn:aws:rolesanywhere:us-east-1:YourAccout#::profile/... \
                --role-arn arn:aws:iam::YourAccout#::role/JenkinsLabRole
        '''
    }
}
```

## Exam Relevance

| Cert | Concept |
|---|---|
| AWS SAA-C03 | STS, IAM roles, least privilege |
| AWS SCS-C03 | IAM Roles Anywhere, credential management, eliminating long-lived keys |
| SC-300 | Certificate-based authentication patterns |

## Infrastructure as Code

Trust anchor, profile, and role are all Terraform-managed in the
[jenkins-terraform-pipeline](https://github.com/arieldla/jenkins-terraform-pipeline) repo.

## Related Repos

- [jenkins-terraform-pipeline](https://github.com/arieldla/jenkins-terraform-pipeline) — Terraform execution using these credentials
- [jenkins-labs](https://github.com/arieldla/jenkins-labs) — Full lab series building on this auth pattern
