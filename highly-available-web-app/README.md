# Highly Available Web App

## Creating NETWORK stack
```terminal
aws cloudformation create-stack --stack-name HighlyAvailableWebAppNetwork --template-body file://network.yml  --parameters file://network-parameters.json 
```

## Creating SERVER stack
```terminal
aws cloudformation create-stack --stack-name HighlyAvailableWebAppServer --template-body file://server.yml  --parameters file://server-parameters.json --capabilities CAPABILITY_IAM
```

## Deleting stacks
```terminal
aws cloudformation delete-stack --stack-name HighlyAvailableWebAppNetwork

aws cloudformation delete-stack --stack-name HighlyAvailableWebAppServer
```