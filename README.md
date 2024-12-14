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
            "Sid": "AllowDescribeInstancesWithMFA",
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

Returning to your [Security credentials page](https://us-east-1.console.aws.amazon.com/iam/home?region=eu-north-1#/users/details/Me?section=security_credentials), we copy the MFA arn to `~\.aws\config`:
```
[profile default]
region = eu-north-1
mfa_serial = arn:aws:iam::03...:mfa/MyPhone
```

Let's also [create our access key to this user](https://us-east-1.console.aws.amazon.com/iam/home?region=eu-north-1#/users/details/Me/create-access-key) -> `Command Line Interface (CLI)` -> `I understand...` -> `Create access key`. Copy the secrets into `~\.aws\credentials`:
```
[default]
aws_access_key_id = AK...
aws_secret_access_key = dj...
```

### Create EC2 instance
Go and [create an ec2 instance](https://eu-north-1.console.aws.amazon.com/ec2/home?region=eu-north-1#LaunchInstances:). The defaults are fine, but you should remember to add a `Key pair (login)` and allow `HTTPS` and `HTTP` traffic from the internet. You may also want to increase your Root volume to e.g. `30 GiB`, which is included in the free tier. You should remember the instance name for later (`i-0caec4bd0b4fdf500`)

## Local setup
Goal: Be able to SSH into the EC2 instance.

### Install UV
`curl -LsSf https://astral.sh/uv/install.sh | sh`
### Install AWSCLI & Awsume
```sh
uv tool update-shell
uv tool install awscli
uv tool install awsume
uv tool run --from awsume --with setuptools awsume-configure
source ~/.bashrc
```
### Setup SSH
We authenticate to AWS, then get the instance IP. Remember to change `i-0caec4bd0b4fdf500` to the name of your instance and `~/.ssh/stationary` to the path to your key. 
```sh
awsume
# Enter MFA token: 564720
# Session token will expire at 2024-12-15 09:37:54
IP=$(aws ec2 describe-instances --instance-ids i-0caec4bd0b4fdf500 --query "Reservations[*].Instances[*].PublicIpAddress" --output text)
ssh -i ~/.ssh/stationary ec2-user@$IP
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
exit
[ec2-user@ip-... ~]$ logout
Connection to 13.60.252.89 closed.
```
We can add an alias `ec2` to our `.bashrc` using the following command: `echo $'alias ec2=\'awsume && ssh -i ~/.ssh/stationary ec2-user@$(aws ec2 describe-instances --instance-ids i-0caec4bd0b4fdf500 --query "Reservations[*].Instances[*].PublicIpAddress" --output text)\'' >> ~/.bashrc`. After restarting the shell, `ec2` should now drop you directly into the remote.

### Setup VSCode
WARNING: 1GB RAM may be insufficient and crash the machine.

Use the above to create an entry in `~/.ssh/config`
```
Host ec2
  HostName 13.60.252.89
  User ec2-user
  IdentityFile ~\.ssh\stationary
```
In case you're on WSL, you need to run the following in windows
`echo C:\Windows\system32\wsl.exe ssh %* > %USERPROFILE%\.ssh\ssh.bat`
And add the following to your `settings.json`
`"remote.SSH.path": "C:\\Users\\Christoffer\\.ssh\\ssh.bat"`

Let's connect via VSCode now: 
```sh
code --install-extension ms-vscode-remote.remote-ssh
code --folder-uri vscode-remote://ssh-remote+ec2/home/ec2-user
```

## AWS setup
Install [docker in AWS](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-docker.html)
```sh
sudo yum update -y
sudo yum install -y docker
sudo service docker start
sudo usermod -a -G docker ec2-user
```
Restart your SSH connection, and the following should work: `docker ps`

To be continued...
