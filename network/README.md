# Network: VPC

## Creating VPC stack
```terminal
aws cloudformation create-stack --stack-name production-vpc --template-body file://VPC.yml  --parameters file://VPC-parameters.json 
```

## Updating VPC stack
```shell
aws cloudformation update-stack --stack-name production-vpc --template-body file://VPC.yml --parameters file://VPC-parameters.json
```

## Deleting VPC stack

Example:
```shell
aws cloudformation delete-stack --stack-name production-vpc
```
