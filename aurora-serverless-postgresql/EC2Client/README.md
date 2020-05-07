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
    "KeyName": "YourServiceClientKP", 
    "KeyFingerprint": "a3:71:e7:92:40:1d:11:19:c9...", 
    "KeyPairId": "key-62hdwdtf2..."
}

```

## SSH into the EC2

```shell
ssh -i ~/.ssh/YourServiceClientKP.pem ec2-user@EC2-public-IP-or-DNS
```

### Some considerations about the EC2

1. It must have the `Security Group` that we are exporting on `aurora-serverless.yml` in the **Outputs -> ClientSecurityGroup**
1. It must be in the same `VPC` and `subnets`. This is also a current limitations: "You can't give an Aurora Serverless DB cluster a public IP address. You can access an Aurora Serverless DB cluster only from within a virtual private cloud (VPC) based on the Amazon VPC service..."

Note: If you want to add the `security group` to a pre-existing `EC2`...
```shell
aws ec2 modify-instance-attribute --instance-id i-00b*** --groups sg-0aa*** sg-060***
```

sg-0aa3f144857515f40 -> this is the one that was created by default

sg-06073e06512ed572b -> the one created in aurora template for the client


## Install PostgreSQL

```shell
sudo yum update -y

sudo amazon-linux-extras enable postgresql10

yum clean metadata

sudo yum install postgresql
```

*Note:* Remember -currently- Aurora Serverless (postgresql) supports `PostgreSQL version 10.7 `

[You can check other limitations...](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless.html#aurora-serverless.limitations)

## Connect to the DB

```shell
psql -h the-db-endpoint-or-cname -U ThisIsTheUser -d YourDatabaseName
```

Then, enter the password defined in `aurora-serverless-parameters.json`. Example: `ThisIsThePassword`

If you want to exit from `PostgreSQL command line utility`, type: `\q`
