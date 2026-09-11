# UPDATE
- [ECS Exec is now available in the AWS Management Console
](https://aws.amazon.com/jp/about-aws/whats-new/2025/09/ecs-exec-aws-management-console/)

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
git clone https://github.com/shueisha-arts-and-digital/sssh.git
cd sssh
./sssh
```

## Usage
```bash
# Select profile, cluster, service, container, and task, and run an interactive sh
./sssh

# Run a one-off command on the container ("--opt value" and "--opt=value" both work)
./sssh --command "php -v"
./sssh --command="php -v"

# The command is wrapped in "/bin/sh -c ...", so pipes work directly,
# and the remote command's exit code becomes the exit code of sssh
./sssh --command "php -v | head -n 1"
./sssh --command 'exit 7' ; echo $?  # => 7

# Run port forwarding
./sssh --remote-host rds.example.com --remote-port 3306 --local-port 13306

# Specify OTP for MFA authentication
./sssh --profile foo-profile --otp 123456

# Help
./sssh --help
```
**Note**: Before running the script, ensure that the AWS CLI profile you use is configured to output in JSON format. Otherwise, the script will crash. Running `aws configure` only sets the `default` profile, so configure the profile you pass to `--profile` explicitly: `aws configure set output json --profile foo-profile`.

**Note**: Since v5.0.0, when `--command` is given, it is wrapped in `/bin/sh -c ...` and the exit code of the remote command becomes the exit code of this script, which makes it usable from CI pipelines and AI agents. For an interactive shell, run without `--command` (the exit code capture would garble interactive prompts, so it is disabled in that case; the exit code is then not reflected, an ECS Exec limitation).

**Note**: ECS Exec runs the remote command on a PTY (pseudo-terminal) inside the SSM agent, so stdout is text-oriented (e.g. `\n` may be translated to `\r\n`). Binary output such as tar/gzip streams is NOT preserved — this is an ECS Exec limitation, not a limitation of `sssh`. Transfer files via S3 or similar instead.

**Note**: Since v5.0.0, all informational messages from sssh (date, Profile/Cluster/Service/Task/Container, the executed command line) go to stderr. stdout carries the remote command's output as well as messages from AWS CLI / Session Manager. Filter the output as needed, for example with `./sssh --command 'php -v' | grep PHP`.

**Note**: Without `pipefail`, `$?` after this pipeline reports `grep`'s exit code. For automation that needs sssh's exit code, run sssh without piping its output.

**Note**: Do not embed passwords or tokens in `--command`. The command line is shown on screen, kept in your local shell history, and recorded in CloudTrail; with ECS Exec logging enabled it may also be stored in CloudWatch Logs / S3.

## Special thanks to contributor
- [leewc](https://github.com/leewc)
- [bamkuhen](https://github.com/bamkuhen)
- [florianfelsing](https://github.com/florianfelsing)
