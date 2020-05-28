# Create User, Group and some Policies

## Create USER stack (user and basic policies)

First, let's create a `user`.
For this example we will follow the logic of: we have *users*, then, we create *groups*. 

With `CFN` you can do the following...

```terminal
aws cloudformation create-stack --stack-name YourCompanyUsers --template-body file://user.yml \
  --parameters file://user-parameters.json \
  --capabilities CAPABILITY_NAMED_IAM
```

Once the `stack` is deployed, the user should log-in into the [AWS console](https://your-company.signin.aws.amazon.com/console) and [Create access key](https://console.aws.amazon.com/iam/home?#/users/your.user?section=security_credentials)


*Note:* You can force the user to have to change her password on first login.


However, at the time of creating users, I usually recommend the `cli` (aka, take the previous template as an example).

## Create the GROUP stack

In this template, we are going to create a `group`, a non-managed policy with some basic permissions to interact with a repository in `ECR` and, assign the `user` we created previously to this group.

```terminal
aws cloudformation create-stack --stack-name YourCompanyProject --template-body file://group.yml \
  --parameters file://group-parameters.json \
  --capabilities CAPABILITY_NAMED_IAM
```

## Deleting stacks

Example:
```terminal
aws cloudformation delete-stack --stack-name YourCompanyUsers
```