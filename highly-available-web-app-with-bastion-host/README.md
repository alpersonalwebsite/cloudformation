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

## TLS, access logs and the AdminCidr parameter

`server.yml` takes a `CertificateArn`. Leave it empty, which is the default in
`server-parameters.json`, and you get the HTTP-only listener this template originally
had, so the example deploys with no certificate to hand. Pass an ACM certificate ARN in
the same region and the same stack instead adds a listener on 443 with
`SslPolicy: ELBSecurityPolicy-TLS-1-2-2017-01` (no TLS 1.0 or 1.1), turns port 80 into a
`HTTP_301` redirect to it, and attaches the listener rule to the HTTPS listener:

```terminal
aws cloudformation create-stack --stack-name HighAvailableWebAppWithBastionHostServer --template-body file://server.yml \
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

`AdminCidr` has no default and is required. It is the CIDR allowed to SSH into the
bastion, and its `AllowedPattern` rejects `0.0.0.0/0` and anything broader than `/16`. A
bastion exists to be the single reachable entry point, which is exactly why leaving its
port 22 open to the Internet (which is what this template used to do) defeats the
purpose. The committed value is `203.0.113.10/32`, from the range RFC 5737 reserves for
documentation; replace it with your own address or VPN range.

The web servers accept HTTP only from the load balancer's security group, and SSH only
from the bastion's.
