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