# Aurora Serverless: PostgreSQL

## Before you deploy

Two parameters have no usable default, and one secret has to exist first.

**The database credentials.** They are not stack parameters. The template reads them from
Secrets Manager at deploy time, so they never appear in the template, the stack events or
a `describe-stacks` response. Create the secret with the name you pass as `DBSecretName`
(`aurora/master` in the committed parameter file):

```shell
aws secretsmanager create-secret --name aurora/master \
  --secret-string "{\"username\":\"yourdbuser\",\"password\":\"$(openssl rand -base64 24)\"}"
```

**`AdminCidr`.** The client security group's SSH rule used to be open to `0.0.0.0/0`. It
now points at this parameter, whose `AllowedPattern` rejects `0.0.0.0/0` and anything
broader than `/16`. The committed value is `203.0.113.10/32`, from the range RFC 5737
reserves for documentation; replace it with your own address or VPN range.

**`VpcID`** is typed `AWS::EC2::VPC::Id` rather than a string, so the console offers a
dropdown of real VPCs and a typo is rejected at validate time instead of failing halfway
through a create.

## Creating AURORA stack
```shell
aws cloudformation create-stack --stack-name aurora-postgresql-sls --template-body file://aurora-serverless.yml --parameters file://aurora-serverless-parameters.json
```

## Updating AURORA stack
```shell
aws cloudformation update-stack --stack-name aurora-postgresql-sls  --template-body file://aurora-serverless.yml --parameters file://aurora-serverless-parameters.json
```

## Deleting AURORA stack

Example:
```shell
aws cloudformation delete-stack --stack-name aurora-postgresql-sls
```

### Notes

1. To list all of the available engine versions for `aurora-postgresql`, use the following command:
```shell
aws rds describe-db-engine-versions --engine aurora-postgresql --query "DBEngineVersions[].EngineVersion"
```

2. `EngineVersion` is still `10.7`, the version this template was written against.
   RDS no longer accepts it for new clusters, so a deploy today fails until you raise it
   to something the command above returns. `cfn-lint` says the same thing as `W3690`.
   It is left as it was on purpose: this repo shows what the template looked like, not a
   version bumped years later.

3. Aurora Serverless v1, which is the `EngineMode` here, does not support IAM database
   authentication, which is why `EnableIAMDatabaseAuthentication` is absent even though
   `checkov` asks for it (`CKV_AWS_162`).