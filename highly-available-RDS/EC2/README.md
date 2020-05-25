# EC2 client

## Create Key Pair

### Generate SSH key pair

```shell
ssh-keygen -t rsa -f ~/.ssh/udacityProject -m PEM

openssl rsa -in ~/.ssh/udacityProject -outform pem > ~/.ssh/udacityProject.pem

chmod 400 ~/.ssh/udacityProject.pem
```

### Import key to AWS
```shell
aws ec2 import-key-pair \
  --region us-east-1 \
  --key-name 'udacityProject' \
  --public-key-material file://~/.ssh/udacityProject.pub
```

**Example output:**
```json
{
    "KeyName": "udacityProject", 
    "KeyFingerprint": "a3:71:e7:92:40:1d:11:19:c9...", 
    "KeyPairId": "key-62hdwdtf2..."
}

```

## SSH into the EC2

```shell
ssh -i ~/.ssh/YourServiceClientKP.pem ec2-user@EC2-public-IP-or-DNS
```