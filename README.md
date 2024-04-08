# sssh
## About
- Bash script to run ecs-exec on Amazon ECS Fargate containers.

## Prerequisites
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [Session Manager plugin for the AWS CLI](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)
- [jq](https://stedolan.github.io/jq/download/)
- [peco](https://github.com/peco/peco#installation)

## Install
```bash
git clone https://github.com/pj8/sssh.git
cd sssh
./sssh
```

## Usage
```bash
# Select profile, cluster, service, container, and task, and run sh
./sssh

# Run port forwarding
./sssh --remote-host rds.example.com --remote-port 3306 --local-port 13306

# Specify OTP for MFA authentication
./sssh --profile foo-profile --otp 123456

# Help
./sssh --help
```
**Note**: Before running the script, ensure that your AWS CLI session profile is configured to output in JSON format. Otherwise, the script will crash. You can set the output format as JSON when you run `aws configure sso`.

## Special thanks to contributor
- [leewc](https://github.com/leewc)
- [bamkuhen](https://github.com/bamkuhen)
