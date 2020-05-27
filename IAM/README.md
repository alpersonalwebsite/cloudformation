# Create User, Group and some policies

## Create USER stack (user and basic policies)

First, let's create a `user` (usually you will have first users, then you would want to 
classify them in groups)

With `CFN` you can do the following...

```terminal
aws cloudformation create-stack --stack-name YourCompanyUsers --template-body file://user.yml \
  --parameters file://user-parameters.json \
  --capabilities CAPABILITY_NAMED_IAM
```

... then, utilize the user should login into the `console` (example: https://your-company.signin.aws.amazon.com/console) and `Create access key`. For this, the user should go to https://console.aws.amazon.com/iam/home?#/users/your.user?section=security_credentials and click on `Create access key`.


*Note:* You can force user to have to change her password on first login.


Yet, for creating `users` I usually recommend the `cli`. The previous template is just an example.


```terminal
aws cloudformation create-stack --stack-name YourCompanyProject --template-body file://project.yml \
  --parameters file://project-parameters.json \
  --capabilities CAPABILITY_NAMED_IAM
```

## Deleting stacks

Example:
```terminal
aws cloudformation delete-stack --stack-name YourCompanyUsers
```