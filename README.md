# Jenkins IAM Roles Anywhere - Homelab Implementation

## Overview
Eliminates static AWS credentials from Jenkins by implementing certificate-based 
authentication via AWS IAM Roles Anywhere. Jenkins authenticates using a self-managed 
PKI instead of long-lived access keys.

## Architecture
```
Jenkins Container
  └── aws_signing_helper (credential_process)
        └── jenkins.crt (signed by JenkinsLabCA)
              └── IAM Roles Anywhere Trust Anchor
                    └── IAM Role (JenkinsRolesAnywhereRole)
                          └── AWS APIs
```

## Prerequisites
- AWS CLI configured with admin access
- Docker running Jenkins container
- OpenSSL

## Components Created
| Component | Value |
|---|---|
| Trust Anchor | JenkinsLabCA |
| Profile | JenkinsProfile |
| IAM Role | JenkinsRolesAnywhereRole |
| Cert Location | /var/jenkins_home/pki/ |
| AWS Config | /var/jenkins_home/.aws/config |

## Implementation Steps

### 1. Create Self-Managed CA
```bash
mkdir ~/jenkins-pki && cd ~/jenkins-pki

openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 -key ca.key \
  -out ca.crt \
  -subj "/C=US/ST=NJ/L=Clifton/O=DLAGroupInc/OU=DevOps/CN=JenkinsLabCA" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign,cRLSign"
```

### 2. Create Jenkins End-Entity Certificate
```bash
# Create extensions file
cat > jenkins-ext.cnf << 'EXTEOF'
[req_ext]
subjectAltName = DNS:jenkins.dlagroupinc.net
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
EXTEOF

# Sign with CA
openssl x509 -req -days 3650 \
  -in ~/jenkins-cert/jenkins.dlagroupinc.net.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out jenkins.crt \
  -extfile jenkins-ext.cnf \
  -extensions req_ext
```

### 3. Create AWS IAM Roles Anywhere Trust Anchor
```bash
TRUST_ANCHOR_ARN=$(aws rolesanywhere create-trust-anchor \
  --name "JenkinsLabCA" \
  --source "sourceType=CERTIFICATE_BUNDLE,sourceData={x509CertificateData=$(cat ~/jenkins-pki/ca.crt | base64 -w 0)}" \
  --enabled \
  --region us-east-1 \
  --profile IAMadmin \
  --query 'trustAnchor.trustAnchorArn' \
  --output text)
```

### 4. Create IAM Role
```bash
cat > /tmp/roles-anywhere-trust-policy.json << 'POLEOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "rolesanywhere.amazonaws.com" },
    "Action": ["sts:AssumeRole","sts:TagSession","sts:SetSourceIdentity"],
    "Condition": {
      "StringEquals": {
        "aws:PrincipalTag/x509Subject/CN": "jenkins.dlagroupinc.net"
      }
    }
  }]
}
POLEOF

aws iam create-role \
  --role-name JenkinsRolesAnywhereRole \
  --assume-role-policy-document file:///tmp/roles-anywhere-trust-policy.json \
  --profile IAMadmin

aws iam attach-role-policy \
  --role-name JenkinsRolesAnywhereRole \
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess \
  --profile IAMadmin
```

### 5. Create Roles Anywhere Profile
```bash
aws rolesanywhere create-profile \
  --name "JenkinsProfile" \
  --role-arns "arn:aws:iam::YOUR_ACCOUNT:role/JenkinsRolesAnywhereRole" \
  --enabled \
  --region us-east-1 \
  --profile IAMadmin
```

### 6. Install aws_signing_helper
```bash
wget https://rolesanywhere.amazonaws.com/releases/1.4.1/X86_64/Linux/aws_signing_helper
chmod +x aws_signing_helper
sudo mv aws_signing_helper /usr/local/bin/
```

### 7. Configure Jenkins Container
```bash
docker exec jenkins mkdir -p /var/jenkins_home/pki
docker cp ~/jenkins-pki/jenkins.crt jenkins:/var/jenkins_home/pki/
docker cp ~/jenkins-pki/jenkins.key jenkins:/var/jenkins_home/pki/
docker cp ~/jenkins-pki/ca.crt jenkins:/var/jenkins_home/pki/
docker cp $(which aws_signing_helper) jenkins:/usr/local/bin/aws_signing_helper
docker exec jenkins chmod +x /usr/local/bin/aws_signing_helper
docker exec jenkins chmod 600 /var/jenkins_home/pki/jenkins.key

docker exec jenkins mkdir -p /var/jenkins_home/.aws
docker exec jenkins bash -c 'cat > /var/jenkins_home/.aws/config << AWSEOF
[default]
credential_process = aws_signing_helper credential-process \
  --certificate /var/jenkins_home/pki/jenkins.crt \
  --private-key /var/jenkins_home/pki/jenkins.key \
  --trust-anchor-arn YOUR_TRUST_ANCHOR_ARN \
  --profile-arn YOUR_PROFILE_ARN \
  --role-arn arn:aws:iam::YOUR_ACCOUNT:role/JenkinsRolesAnywhereRole \
  --region us-east-1
region = us-east-1
AWSEOF'
```

## Validation
```bash
# Test from container
docker exec jenkins aws_signing_helper credential-process \
  --certificate /var/jenkins_home/pki/jenkins.crt \
  --private-key /var/jenkins_home/pki/jenkins.key \
  --trust-anchor-arn YOUR_TRUST_ANCHOR_ARN \
  --profile-arn YOUR_PROFILE_ARN \
  --role-arn arn:aws:iam::YOUR_ACCOUNT:role/JenkinsRolesAnywhereRole \
  --region us-east-1
# Expected: JSON with AccessKeyId, SecretAccessKey, SessionToken
```

### Jenkins Test Pipeline
```groovy
pipeline {
    agent any
    stages {
        stage('Verify IAM Roles Anywhere') {
            steps {
                sh 'aws sts get-caller-identity'
                sh 'aws s3 ls'
            }
        }
    }
}
```

## Security Considerations
- CA private key (ca.key) should be stored securely - not in Jenkins
- Jenkins cert is scoped via CN condition in IAM trust policy
- No static credentials stored anywhere in Jenkins
- Session tokens expire every 3600s and auto-refresh via credential_process
- Certificate renewal required before expiry (3650 days from issuance)

## Cert Renewal Process
```bash
# Re-sign jenkins.crt with existing CA (no need to recreate trust anchor)
openssl x509 -req -days 3650 \
  -in ~/jenkins-cert/jenkins.dlagroupinc.net.csr \
  -CA ~/jenkins-pki/ca.crt -CAkey ~/jenkins-pki/ca.key -CAcreateserial \
  -out ~/jenkins-pki/jenkins.crt \
  -extfile ~/jenkins-pki/jenkins-ext.cnf \
  -extensions req_ext

docker cp ~/jenkins-pki/jenkins.crt jenkins:/var/jenkins_home/pki/
```
