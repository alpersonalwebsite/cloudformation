# CloudFormation templates

A set of small, self-contained CloudFormation stacks, each one a worked example of a
single piece of AWS infrastructure. Clone it, read the template, deploy it, delete it.

Every directory has its own README with the exact `aws cloudformation` commands and a
matching `*-parameters.json`. The committed parameter files are meant to deploy as they
are, apart from the handful of values called out under [Secrets](#secrets) below.

| Directory | What it builds |
| --- | --- |
| `network/` | The smallest useful template in the repo: one `AWS::EC2::VPC` |
| `ec2-instance-with-cfinit.yml` | A single EC2 instance that installs Apache and PHP through `cfn-init` |
| `highly-available-web-app/` | Autoscaling web tier in private subnets behind an Application Load Balancer |
| `highly-available-web-app-with-bastion-host/` | The same, with a bastion host as the only SSH entry point |
| `fargate/` | ECS on Fargate: network, task execution role, cluster with a load balancer, service |
| `aurora-serverless-postgresql/` | An Aurora Serverless PostgreSQL cluster, plus an EC2 client to reach it |
| `highly-available-RDS/` | A StackSet-deployed VPC in two regions, a MySQL primary and a cross-region read replica |
| `IAM/` | A user, a group, and a self-service policy for access keys and MFA |
| `StackSets/` | The administration and execution roles StackSets needs |

## Conventions shared across the templates

### Secrets

No template takes a password as a parameter. Passwords are read at deploy time from
Secrets Manager with a [dynamic
reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/dynamic-references.html):

```yaml
MasterUserPassword: !Sub '{{resolve:secretsmanager:${DBSecretName}:SecretString:password}}'
```

That matters more than `NoEcho` does. A `NoEcho` parameter is masked in the console, but
you still typed it on a command line, it still sits in your shell history, and it still
travels to CloudFormation. A dynamic reference means the value never leaves Secrets
Manager: it is not in the template, not in the stack events, and not in a
`describe-stacks` response.

Create the secret once before deploying. The name you pass as `DBSecretName` or
`UserSecretName` must match:

```shell
aws secretsmanager create-secret --name rds/primary \
  --secret-string "{\"username\":\"dbuser\",\"password\":\"$(openssl rand -base64 24)\"}"
```

The templates read the `username` and `password` keys out of that JSON body.

### SSH is never open to the Internet

Anything with an SSH rule takes an `AdminCidr` parameter with no default. Its
`AllowedPattern` rejects `0.0.0.0/0` and anything broader than `/16`, so the stack fails
validation rather than quietly building a host the whole Internet can knock on:

```shell
--parameters ParameterKey=AdminCidr,ParameterValue=203.0.113.10/32
```

The committed parameter files use `203.0.113.10/32`, from the range RFC 5737 reserves for
documentation. Replace it with your own address or VPN range.

The standalone `ec2-instance-with-cfinit.yml` at the root has no directory or parameter
file of its own, so pass all three of its parameters:

```shell
aws cloudformation create-stack --stack-name cfinit-demo \
  --template-body file://ec2-instance-with-cfinit.yml \
  --parameters ParameterKey=NameOfService,ParameterValue=cfinit-demo \
    ParameterKey=KeyName,ParameterValue=your-existing-keypair \
    ParameterKey=AdminCidr,ParameterValue=203.0.113.10/32
```

### TLS on the load balancers is opt-in but one parameter away

The three load balancer templates take a `CertificateArn`. Leave it empty, which is the
default, and you get the HTTP-only listener the template originally had, so the example
still deploys for someone with no certificate to hand. Supply an ACM certificate ARN in
the same region and the stack instead:

- adds a listener on 443 with `SslPolicy: ELBSecurityPolicy-TLS-1-2-2017-01`, which
  excludes TLS 1.0 and 1.1,
- turns port 80 into a `HTTP_301` redirect to it,
- attaches the listener rules to the HTTPS listener rather than the HTTP one.

Access logging and `routing.http.drop_invalid_header_fields.enabled` are on either way.
Each load balancer template creates its own log bucket: SSE-S3 encryption, public access
blocked, versioned, current versions expiring at 90 days and noncurrent ones the day
after. SSE-S3 was the only algorithm ALB log delivery accepted when these templates were
written; AWS has since added SSE-KMS with a customer managed key, but AES256 stays the
right default here because it needs no key, no key policy and no bill.

### Dependencies between stacks are parameters, not hardcoded names

Stacks that consume another stack's exports take the exporting stack's name as a
parameter (`EnvironmentName`, `VpcName`, `VPCname` depending on the directory) instead of
hardcoding the prefix. If `Fn::ImportValue` fails at create time with `No export named
... found`, that parameter does not match the stack that produced the export.

## Verifying a change

Both tools are period-appropriate and run against the whole tree:

```shell
pip install cfn-lint checkov

# one file at a time: cfn-lint's glob handling does not take a list of paths
for f in $(find . -name '*.yml' -o -name '*.yaml'); do cfn-lint "$f"; done

checkov -d . --framework cloudformation --compact
```

`cfn-lint` reports no errors and two warnings, both deliberate:

| Warning | Why it stays |
| --- | --- |
| `W3690` aurora-postgresql `10.7` | The version this template was written against. RDS no longer accepts it for new clusters, so raise it when you deploy; the template is left as it was. |
| `W3691` mysql `8.0.17` | Same, for the RDS primary. |

`checkov` reports 213 passed and 8 failed. Each of the 8 is a deliberate decision rather
than an oversight:

| Check | Where | Why it is not "fixed" |
| --- | --- | --- |
| `CKV_AWS_260` (HTTP from `0.0.0.0/0`) | The three load balancer security groups, plus the standalone EC2 example | Accepting HTTP from anywhere is what a public web server is for. With a `CertificateArn` supplied, port 80 only issues a redirect. |
| `CKV_AWS_162` (IAM database auth) | `aurora-serverless-postgresql/aurora-serverless.yml` | Aurora Serverless v1, which is the `EngineMode` this template uses, is widely documented as not supporting IAM database authentication. Not confirmed against a current AWS page, because the v1 documentation was retired when the engine reached end of life, so `EnableIAMDatabaseAuthentication` is left off rather than added on a guess. |
| `CKV_AWS_158` (KMS on the log group) | `fargate/cluster.yml` | Needs a customer-managed KMS key with a `logs.amazonaws.com` key policy. That is a billed resource that survives the stack for at least seven days after deletion, which is the wrong trade for a teaching example. |
| `CKV_AWS_16` (encryption at rest) | `highly-available-RDS/rds-secondary.yaml` | A read replica inherits encryption from its source, and setting `StorageEncrypted` alongside `SourceDBInstanceIdentifier` is rejected outright (`cfn-lint E3020`). The primary sets it. This is a false positive. |
| `CKV_AWS_157` (Multi-AZ) | `highly-available-RDS/rds-secondary.yaml` | The cross-region replica *is* the availability story in that stack, and Multi-AZ on top of it doubles the replica's cost. Set `MultiAZ: true` if you want both. |

The `AWS::S3::Bucket` resources also carry a `Metadata: checkov: skip:` for
`CKV_AWS_18`, with the reason inline: they are the access log buckets, so S3 server
access logging on them either needs a second bucket with the same problem, or points at
itself and logs its own log writes forever.
