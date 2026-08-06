# Docker plus ECS on AWS Fargate

## Creating NETWORK stack
```terminal
aws cloudformation create-stack --stack-name DockerECSonFargateNetwork --template-body file://network.yml  --parameters file://network-parameters.json 
```

## Creating ROLE(s) stack
```terminal
aws cloudformation create-stack --stack-name DockerECSonFargateRoles --template-body file://roles.yml --parameters file://roles-parameters.json  --capabilities CAPABILITY_IAM
```

## Creating CLUSTER stack
```terminal
aws cloudformation create-stack --stack-name DockerECSonFargateCluster --template-body file://cluster.yml --parameters file://cluster-parameters.json
```

## Creating CONTAINER stack
```terminal
aws cloudformation create-stack --stack-name DockerECSonFargateContainer --template-body file://container.yml --parameters file://container-parameters.json
```

## Deleting stacks

Example:
```terminal
aws cloudformation delete-stack --stack-name DockerECSonFargateContainer
```

## TLS on the load balancer

`cluster.yml` takes a `CertificateArn`, empty by default. Supply an ACM certificate ARN in
the same region and the cluster stack adds a listener on 443 with
`SslPolicy: ELBSecurityPolicy-TLS-1-2-2017-01`, turns port 80 into a `HTTP_301` redirect
to it, and exports *that* listener. `container.yml` imports `${EnvironmentName}-Listener`,
so its listener rule follows whichever listener is live without any change on your part.

Either way the load balancer writes access logs to a bucket the stack creates and sets
`routing.http.drop_invalid_header_fields.enabled`.

The load balancer's security group used to allow `IpProtocol: -1` from `0.0.0.0/0`, which
is every port and every protocol rather than the HTTP it serves. It now allows 80 and 443
inbound, and bounded TCP outbound to the container targets.

## What the task execution role can do

`roles.yml` used to grant six actions on `Resource: '*'` in a single statement. It is
split three ways now:

| Statement | Resource |
| --- | --- |
| `ecr:GetAuthorizationToken` | `*`, because that action genuinely has no resource ARN |
| the three ECR pull actions | repositories in this account and region |
| `logs:CreateLogStream`, `logs:PutLogEvents` | the `web-app` log group and its streams |

Narrow the ECR statement further to the single repository you deploy from if you know its
name.

Container Insights is enabled on the cluster. It bills as CloudWatch custom metrics, so it
is not free; the log group's retention is 1 day for the same reason.
