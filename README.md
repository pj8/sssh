# sssh
## About
- Bash script to run ecs-exec on Amazon ECS Fargate containers.

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

# Help
./sssh --help
```

## Prerequisites
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [Session Manager plugin for the AWS CLI](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)
- [jq](https://stedolan.github.io/jq/download/)
- [peco](https://github.com/peco/peco#installation)

## Special thanks to contributor
- [leewc](https://github.com/leewc)
- [bamkuhen](https://github.com/bamkuhen)
