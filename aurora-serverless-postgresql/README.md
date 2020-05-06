# Aurora Serverless: PostgreSQL

## Creating AURORA stack
```shell
aws cloudformation create-stack --stack-name aurora-postgresql-sql --template-body file://aurora-serverless.yml --parameters file://aurora-serverless-parameters.json
```

```
redis-cli -h prod-ec-redis-cocina.seasoned.co -p 6379
```

## Updating AURORA stack
```shell
aws cloudformation update-stack --stack-name aurora-postgresql-sql  --template-body file://aurora-serverless.yml --parameters file://aurora-serverless-parameters.json
```

## Deleting AURORA stack

Example:
```shell
aws cloudformation delete-stack --stack-name aurora-postgresql-sql
```

### Notes

1. To list all of the available engine versions for `aurora-postgresql`, use the following command:
```shell
aws rds describe-db-engine-versions --engine aurora-postgresql --query "DBEngineVersions[].EngineVersion"
```