# vps-app
Trying to create a production grade vps hosted app using AWS free tier

## Disposition
- AWS setup
- Local setup
- EC2 setup

## AWS setup
### Create user
[Create your user](https://us-east-1.console.aws.amazon.com/iam/home?region=eu-north-1#/users/create) -> `User name: Me` -> `Next` -> `Next` -> `Create user`

Create a policy requiring MFA which grants access to all ec2 actions. We will use this to easily get the instance IP later.  [Add some inline permissions to your user](https://us-east-1.console.aws.amazon.com/iam/home?region=eu-north-1#/users/details/Me/createPolicy?step=addPermissions) -> `JSON` -> 
```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Action": "ec2:*",
			"Effect": "Allow",
			"Resource": "*",
			"Condition": {
				"Bool": {
					"aws:MultiFactorAuthPresent": "true"
				}
			}
		}
	]
}
```
`Policy name: MFA-EC2`

Let's [setup a MFA device](https://us-east-1.console.aws.amazon.com/iam/home?region=eu-north-1#/users/details/Me/mfa) with `Device name: MyPhone` -> `Authenticator app` -> `Next` -> Finalize setup, then click `Add MFA`.

Returning to your [Security credentials page](https://us-east-1.console.aws.amazon.com/iam/home?region=eu-north-1#/users/details/Me?section=security_credentials), we copy the MFA arn to `%USERPROFILE%\.aws\config`:
```
[profile default]
region = eu-north-1
mfa_serial = arn:aws:iam::03...:mfa/MyPhone
```

Let's also [create our access key to this user](https://us-east-1.console.aws.amazon.com/iam/home?region=eu-north-1#/users/details/Me/create-access-key) -> `Command Line Interface (CLI)` -> `I understand...` -> `Create access key`. Copy the secrets into `%USERPROFILE%\.aws\credentials`:
```
[default]
aws_access_key_id = AK...
aws_secret_access_key = dj...
```

### Create EC2 instance
Go and [create an ec2 instance)[https://eu-north-1.console.aws.amazon.com/ec2/home?region=eu-north-1#LaunchInstances:]. The defaults are fine, but you should remember to add a `Key pair (login)` and allow `HTTPS` and `HTTP` traffic from the internet. You may also want to increase your Root volume to e.g. `30 GiB`, which is included in the free tier

## Local setup
Goal: Be able to SSH into the EC2 instance.

### Install UV
`powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"`
### Install AWSCLI
`msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2-2.0.30.msi`
### Install awsume
`uv tool install awsume`
### Setup SSH
First, we get the IP of the ec2 instance
```sh
awsume
# Enter MFA token: 564720
# Session token will expire at 2024-12-15 09:37:54
aws ec2 describe-instances --instance-ids i-0caec4bd0b4fdf500 --query "Reservations[*].Instances[*].PublicIpAddress" --output text
# 13.48.42.100
```
Copy this to `%USERPROFILE%/.ssh/config`, and use the SSH token you added during EC2 creation 
```
Host ec2
  HostName 13.48.42.100
  User ec2-user
  IdentityFile C:\Users\Christoffer\.ssh\stationary.pem
```
Let's confirm we can connect: `ssh ec2`
```
   ,     #_
   ~\_  ####_        Amazon Linux 2023
  ~~  \_#####\
  ~~     \###|
  ~~       \#/ ___   https://aws.amazon.com/linux/amazon-linux-2023
   ~~       V~' '->
    ~~~         /
      ~~._.   _/
         _/ _/
       _/m/'
Last login: Sat Dec 14 19:15:21 2024 from ...
```
Let's connect via VSCode now: 
```sh
code --install-extension ms-vscode-remote.remote-ssh
code --remote ssh-remote+ec2 /home/ec2-user/
```

## AWS setup
To be continued
