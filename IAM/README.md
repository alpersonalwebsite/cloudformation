# Create User, Group and some Policies

## Create USER stack (user and basic policies)

First, let's create a `user`.
For this example we will follow the logic of: we have *users*, then, we create *groups*.

### Before you deploy: the console password

`user.yml` does not take the password as a parameter. It reads it from Secrets Manager at
deploy time, so it never appears in the template, the stack events or a `describe-stacks`
response. Create the secret first, with the name you pass as `UserSecretName`
(`iam/initial-console-password` by default):

```shell
aws secretsmanager create-secret --name iam/initial-console-password \
  --secret-string "{\"password\":\"$(openssl rand -base64 18)\"}"
```

The user is created with `PasswordResetRequired: true`, so that value only has to survive
the first login.

With `CFN` you can do the following...

```terminal
aws cloudformation create-stack --stack-name YourCompanyUsers --template-body file://user.yml \
  --parameters file://user-parameters.json \
  --capabilities CAPABILITY_NAMED_IAM
```

`user.yml` creates its own group and puts the user in it, so the self-service policy is
attached to a group rather than straight to the user. With one user the two are
equivalent; with five, per-user attachments are how people end up unable to answer "who
can do this?" without reading every user in turn. The group in `group.yml` below is a
separate one, for project permissions.

Once the `stack` is deployed, the user should log-in into the [AWS console](https://your-company.signin.aws.amazon.com/console) and [Create access key](https://console.aws.amazon.com/iam/home?#/users/your.user?section=security_credentials)


*Note:* The template already forces a password change on first login.

### What the OwnAccessKeysAndMFA policy does and does not allow

It is scoped to `arn:aws:iam::<account>:user/${aws:username}` and
`:mfa/${aws:username}`, so a holder can rotate their own access keys and enrol their own
MFA device and nothing else. `${aws:username}` is an IAM policy variable, not a
CloudFormation one, which is why the template writes it as `${!aws:username}`: the `!`
escapes it from `Fn::Sub` so it reaches IAM as a literal.

Read the resource ARNs before reusing this. With `Resource: '*'`, which is what the
policy had, `iam:CreateAccessKey` on every user means anyone holding it could mint
credentials for an administrator and use them.


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