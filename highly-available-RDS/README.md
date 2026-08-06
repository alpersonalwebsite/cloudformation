
# Highly Available and resilient RDS

A MySQL primary in one region, a read replica in another, and the VPCs for both deployed
as a StackSet. The point of the walkthrough is the last section: promoting the replica.

## Before you start

**Replace `your-admin-account-id`** everywhere it appears below with your own account id
(`aws sts get-caller-identity --query Account --output text`). It appears in the execution
role, in `create-stack-instances`, and in `delete-stack-instances`.

**Create the database secret.** `rds-primary.yaml` does not take a username or password as
a parameter. It reads both from Secrets Manager at deploy time with a dynamic reference,
so neither value appears in the template, the stack events or a `describe-stacks`
response. Create it with the name you pass as `DBSecretName` (`rds/primary` in the
committed parameter file):

```shell
aws secretsmanager create-secret --name rds/primary \
  --secret-string "{\"username\":\"dbuser\",\"password\":\"$(openssl rand -base64 24)\"}"
```

**Know which parameters tie the stacks together.** `Fn::ImportValue` needs the exporting
stack's names to match:

| Parameter | Default | Has to match |
| --- | --- | --- |
| `VpcName` (both DB stacks) | `Primary` / `Secondary` | the `VpcName` the VPC StackSet instance was created with |
| `SourceDBInstanceID`, `SourceDBRegion` (replica) | `mydbinstance`, `us-east-1` | the primary's `DBInstanceID` and region |
| `MonitoringRoleName` (replica) | `MyDBEnhancedMonitoringRole` | the role `rds-primary.yaml` creates. IAM is global, so the replica reuses it instead of creating a second one |
| `AdminCidr` (VPC) | none, required | your own address or VPN range. The application security group's SSH rule points at it, and its `AllowedPattern` rejects `0.0.0.0/0` |


## Create role AWSCloudFormationStackSetAdministrationRole in the root account

Check first... https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs.html

Grant Self-Managed Permissions
https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs-self-managed.html

```shell
aws cloudformation create-stack --stack-name MyStackSet-AWSCloudFormationStackSetAdministrationRole --template-body file://IAM-root.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

## Create role AWSCloudFormationStackSetExecutionRole in ALL accounts (including root) and MyDBRole

```shell
aws cloudformation create-stack --stack-name MyStackSet-AWSCloudFormationStackSetExecutionRole --template-body file://IAM.yaml \
  --parameters  ParameterKey=AdministratorAccountId,ParameterValue=your-admin-account-id \
  --capabilities CAPABILITY_NAMED_IAM
```

## Create StackSet

```shell
aws cloudformation create-stack-set \
    --stack-set-name MyStackSet \
    --template-body file://vpc.yaml \
    --description "VPC" \
    --parameters  ParameterKey=VpcName,ParameterValue=Primary \
                  ParameterKey=AdminCidr,ParameterValue=203.0.113.10/32
```

`AdminCidr` has no default, so `create-stack-set` needs a value for it here even though
each instance below overrides it from its own parameter file.

## Create instances

```shell
aws cloudformation create-stack-instances --stack-set-name MyStackSet \
  --accounts your-admin-account-id \
  --regions "us-east-1" \
  --parameter-overrides file://vpc-primary-parameters.json


aws cloudformation create-stack-instances --stack-set-name MyStackSet \
  --accounts your-admin-account-id \
  --regions "us-west-1" \
  --parameter-overrides file://vpc-secondary-parameters.json
```

## Create DB (default region, us-east-1)

```shell
aws cloudformation create-stack --stack-name MyStackSet-DB --template-body file://rds-primary.yaml \
  --parameters file://rds-primary-parameters.json \
  --capabilities CAPABILITY_NAMED_IAM
```

## Create DB read replica in other region (us-west-1)

**Note:** Wait until the previous stack finalizes (aka, CREATE_COMPLETE || UPDATE_COMPLETE)

Every parameter in `rds-secondary.yaml` has a default matching the primary above, so this
needs none. Pass `SourceDBInstanceID`, `SourceDBRegion` or `VpcName` if you changed them.
The source used to be hardcoded as
`arn:aws:rds:us-east-1:your-admin-account-id:db:mydbinstance`, which meant the template
could not deploy at all without editing it first.

```shell
aws --region us-west-1 cloudformation create-stack --stack-name MyStackSet-DB --template-body file://rds-secondary.yaml
```

The replica sets no `StorageEncrypted`. A read replica inherits encryption from its source
and setting it alongside `SourceDBInstanceIdentifier` is rejected outright, so the
primary's `StorageEncrypted: true` is what encrypts both.

## Create EC2 client (default region)

```shell
aws cloudformation create-stack --stack-name MyStackSet-EC2 \
  --template-body file://EC2/ec2.yaml \
  --parameters file://EC2/ec2-parameters.json
```

## SSH into the EC2

```shell
ssh -i ~/.ssh/MyStackSetProject.pem ec2-user@EC2-public-IP-or-DNS
```

## Master vs Replica

In the Master DB you will be able to read/write; in the replica, just read. 

If you try to write in the replica you will receive an error like:
```shell
MySQL [mydb]> INSERT INTO friend (name, lastName) VALUES ('Peter', 'Pan');
ERROR 1290 (HY000): The MySQL server is running with the --read-only option so it cannot execute this statement
MySQL [mydb]> CREATE TABLE hello (name VARCHAR(20), lastName VARCHAR(20));
ERROR 1290 (HY000): The MySQL server is running with the --read-only option so it cannot execute this statement
MySQL [mydb]>
```

## MySQL

### Install MySQL client

```shell
sudo yum update
sudo yum -y install mysql
```

### Connect to the DB

Read the credentials back out of the secret rather than retyping them, and use `-p` with no
argument so the password is prompted for instead of landing in your shell history and in
`ps` output for every other user on the box:

```shell
creds=$(aws secretsmanager get-secret-value --secret-id rds/primary \
  --query SecretString --output text)
host=$(aws cloudformation describe-stacks --stack-name MyStackSet-DB \
  --query 'Stacks[0].Outputs[?OutputKey==`DBEndpoint`].OutputValue' --output text)

mysql --host="$host" --user="$(echo "$creds" | python3 -c 'import json,sys;print(json.load(sys.stdin)["username"])')" -p mydb
```

The endpoints look like
`mydbinstance.cdyj3fol1p4g.us-east-1.rds.amazonaws.com` and
`mydbinstancereplica.czjqr4qz6vgt.us-west-1.rds.amazonaws.com`.

### Create a table

```shell
CREATE TABLE friend (name VARCHAR(20), lastName VARCHAR(20));
```

### Retrieves all tables

```shell
SHOW TABLES;
```

### Insert data to the table

```shell
INSERT INTO friend (name, lastName) VALUES ('Peter', 'Pan');
```

### Retrieve data from the table

```shell
SElECT * from friend;
```

## Promote REPLICA 

```shell
aws --region us-west-1 rds promote-read-replica \
    --db-instance-identifier mydbinstancereplica
``` 

Wait until changes take effect (status should go from Modifying to Available)

Now, you should be able to read and **WRITE** in the instance.
Note that its role changed from `replica` to `instance`

```shell
SElECT * from friend;

INSERT INTO friend (name, lastName) VALUES ('Capt.', 'Hook');
```

---

## Delete stack set

### Delete StackSet Instances 
```shell
aws cloudformation delete-stack-instances \
    --stack-set-name MyStackSet \
    --accounts your-admin-account-id \
    --regions us-east-1 us-west-1 \
    --no-retain-stacks
```

### Delete StackSet 

```shell
aws cloudformation delete-stack-set \
    --stack-set-name MyStackSet
```