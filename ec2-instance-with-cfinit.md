Optional: Ensure you are using the right credentials
```terminal
aws sts get-caller-identity
```

Result:
```
{
    "Account": "5737836549463", 
    "UserId": "AIDHGDGKSHGJGHFJKGFHJNF", 
    "Arn": "arn:aws:iam::5737836549463:user/yourusername"
}
```

1. Create EC2 Key pair
```terminal
aws ec2 create-key-pair --key-name EC2KeyPair-yourusername > ~/.ssh/EC2KeyPair-yourusername.pem

chmod 400 ~/.ssh/EC2KeyPair-yourusername.pem
```

<!-- To improve: write just the key material in the file -->

Original permissions: `-rw-r--r--`
New permissions: `-r--------`

Note: You can check the permissions executing: `ls -la ~/.ssh/EC2KeyPair-yourusername.pem`