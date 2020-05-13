# Aurora Serverless: PostgreSQL

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