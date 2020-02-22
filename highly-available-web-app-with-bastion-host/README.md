# Highly Available Web App with Bastion Host

## Creating NETWORK stack
```terminal
aws cloudformation create-stack --stack-name HighAvailableWebAppWithBastionHostNetwork --template-body file://network.yml  --parameters file://network-parameters.json 
```

## Creating SERVER stack
```terminal
aws cloudformation create-stack --stack-name HighAvailableWebAppWithBastionHostServer --template-body file://server.yml  --parameters file://server-parameters.json --capabilities CAPABILITY_IAM
```

## Deleting stacks
```terminal
aws cloudformation delete-stack --stack-name HighAvailableWebAppWithBastionHostNetwork

aws cloudformation delete-stack --stack-name HighAvailableWebAppWithBastionHostServer
```