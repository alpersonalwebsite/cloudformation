# EC2 client

## Create Key Pair

### Generate SSH key pair

```shell
ssh-keygen -t rsa -f ~/.ssh/YourServiceClientKP -m PEM

openssl rsa -in YourServiceClientKP -outform pem > YourServiceClientKP.pem

chmod 400 YourServiceClientKP.pem
```

### Import key to AWS
```shell
aws ec2 import-key-pair \
  --region us-east-1 \
  --key-name 'YourServiceClientKP' \
  --public-key-material file://~/.ssh/YourServiceClientKP.pub
```

**Example output:**
```json
{
    "KeyName": "user-service-client-kp", 
    "KeyFingerprint": "a3:71:e7:92:40:1d:11:19:c9...", 
    "KeyPairId": "key-62hdwdtf2..."
}

```

## SSH into the EC2

```shell
ssh -i ~/.ssh/user-service-client-kpYourKP.pem ec2-user@EC2-public-IP-or-DNS
```

## Install PostgreSQL

```shell
sudo yum update -y

sudo amazon-linux-extras enable postgresql10
```

*Note:* Remember -currently- Aurora Serverless (postgresql) supports `PostgreSQL version 10.7 `

[You can check other limitations...](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless.html#aurora-serverless.limitations)