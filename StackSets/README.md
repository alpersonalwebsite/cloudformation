# StackSets

Check first... https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs.html

I'm going with `Grant Self-Managed Permissions`
https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs-self-managed.html

## Create role AWSCloudFormationStackSetAdministrationRole in the root account

```shell
aws cloudformation create-stack --stack-name UD-AWSCloudFormationStackSetAdministrationRole --template-body file://IAM-root.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

## Create role AWSCloudFormationStackSetExecutionRole in root and ALL accounts

```shell
aws cloudformation create-stack --stack-name UD-AWSCloudFormationStackSetExecutionRole --template-body file://IAM.yaml \
  --parameters  ParameterKey=AdministratorAccountId,ParameterValue=your-aws-account-id \
  --capabilities CAPABILITY_NAMED_IAM
```

## Create Stack Set

```shell
aws cloudformation create-stack-set \
    --stack-set-name MyStackSet \
    --template-body file://vpc.yaml \
    --description "VPC" \
    --parameters  ParameterKey=VpcName,ParameterValue=Primary
```

## Create Stack Instances

```shell
aws cloudformation create-stack-instances --stack-set-name MyStackSet \
  --accounts your-aws-account-id \
  --regions "us-east-1" \
  --parameter-overrides  ParameterKey=VpcName,ParameterValue=Primary


aws cloudformation create-stack-instances --stack-set-name MyStackSet \
  --accounts your-aws-account-id \
  --regions "us-west-1" \
  --parameter-overrides  ParameterKey=VpcName,ParameterValue=Secondary
```

## Delete Stack Set and Stack Instances

### First

```shell
aws cloudformation delete-stack-instances \
    --stack-set-name MyStackSet \
    --accounts your-aws-account-id \
    --regions us-east-1 us-west-1 \
    --no-retain-stacks
```

### Then

```shell
aws cloudformation delete-stack-set \
    --stack-set-name UD
```
