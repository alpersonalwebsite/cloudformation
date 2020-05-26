
# Highly Available and resilient RDS


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
    --parameters  ParameterKey=VpcName,ParameterValue=Primary
```

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

```shell
aws --region us-west-1 cloudformation create-stack --stack-name MyStackSet-DB --template-body file://rds-secondary.yaml
```

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

```shell
mysql --host=rds_endpoint --user=myname --password=password mydb
```

mysql --host=mydbinstance.cdyj3fol1p4g.us-east-1.rds.amazonaws.com --user=dbuser --password=****** mydb


mysql --host=mydbinstancereplica.czjqr4qz6vgt.us-west-1.rds.amazonaws.com --user=dbuser --password=****** mydb

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