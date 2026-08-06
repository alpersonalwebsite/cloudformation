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

## TLS, access logs and the AdminCidr parameter

`server.yml` takes a `CertificateArn`. Leave it empty, which is the default in
`server-parameters.json`, and you get the HTTP-only listener this template originally
had, so the example deploys with no certificate to hand. Pass an ACM certificate ARN in
the same region and the same stack instead adds a listener on 443 with
`SslPolicy: ELBSecurityPolicy-TLS-1-2-2017-01` (no TLS 1.0 or 1.1), turns port 80 into a
`HTTP_301` redirect to it, and attaches the listener rule to the HTTPS listener:

```terminal
aws cloudformation create-stack --stack-name HighlyAvailableWebAppServer --template-body file://server.yml \
  --parameters file://server-parameters.json \
    ParameterKey=CertificateArn,ParameterValue=arn:aws:acm:us-east-1:123456789012:certificate/... \
  --capabilities CAPABILITY_IAM
```

Either way the load balancer writes access logs to a bucket the stack creates (SSE-S3,
public access blocked, 90 day expiry) and sets
`routing.http.drop_invalid_header_fields.enabled`, so headers that are not valid HTTP are
dropped rather than handed to the instances.

The EBS volume in the launch configuration is encrypted. Encryption can only be set when
a volume is created, so an existing stack needs replacing rather than updating.

The web servers now accept HTTP only from the load balancer's security group. They sit in
private subnets, so the previous `0.0.0.0/0` ingress rule was both broader than needed and
misleading about how traffic reaches them.
