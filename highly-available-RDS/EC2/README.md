# EC2 client

## Create Key Pair

### Generate SSH key pair

```shell
ssh-keygen -t rsa -f ~/.ssh/MyStackSetProject -m PEM

openssl rsa -in ~/.ssh/MyStackSetProject -outform pem > ~/.ssh/MyStackSetProject.pem

chmod 400 ~/.ssh/MyStackSetProject.pem
```

### Import key to AWS
```shell
aws ec2 import-key-pair \
  --region us-east-1 \
  --key-name 'MyStackSetProject' \
  --public-key-material file://~/.ssh/MyStackSetProject.pub
```

**Example output:**
```json
{
    "KeyName": "MyStackSetProject", 
    "KeyFingerprint": "a3:71:e7:92:40:1d:11:19:c9...", 
    "KeyPairId": "key-62hdwdtf2..."
}

```

## SSH into the EC2

```shell
ssh -i ~/.ssh/MyStackSetProject.pem ec2-user@EC2-public-IP-or-DNS
```