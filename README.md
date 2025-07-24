# Open-Athena/ec2
Auto-terminating EC2 GHA runner.

📖 **Demo**: See [ec2-runner-demo](https://github.com/Open-Athena/ec2-runner-demo) for a complete working example.

## Features

- 🚀 Starts EC2 instances on-demand for GitHub Actions jobs
- 🧹 Self-terminates when job completes (no separate stop job needed!)
- 🔑 Uses GitHub OIDC for AWS authentication (no long-lived credentials)
- ⚡ Single reusable workflow call
- 🎯 GPU-optimized AMI by default
- 💰 Defaults to [`g4dn.xlarge`](https://instances.vantage.sh/aws/ec2/g4dn.xlarge) - the cheapest EC2 GPU instance we're aware of
- 🔒 Uses Open-Athena's fork of start-aws-gha-runner for `userdata` support

## Setup

### 1. Configure AWS IAM Role

Your AWS role needs permissions to:
- Launch EC2 instances
- Pass IAM roles
- Manage GitHub Actions runners

The role must trust GitHub's OIDC provider. See [GitHub's documentation](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) for setup.

### 2. Set Organization Secrets

In your GitHub organization settings, create these secrets:
- `AWS_ROLE`: ARN of your AWS IAM role (e.g., `arn:aws:iam::123456789012:role/GitHubActionsRole`)
- `GH_SA_TOKEN`: GitHub token with permissions to manage self-hosted runners


## Minimal Example

Just 16 lines to run GPU tests on EC2:

```yaml
name: Minimal GPU EC2 runner test
on:
  workflow_dispatch:
permissions:
  id-token: write  # Required for AWS OIDC authentication
  contents: read   # Required for actions/checkout
jobs:
  ec2:
    uses: Open-Athena/ec2/.github/workflows/runner.yml@main
    secrets: inherit
  gpu-test:
    needs: ec2
    runs-on: ${{ needs.ec2.outputs.instance }}
    steps:
    - run: nvidia-smi  # g4dn.xlarge!
```

That's it! The EC2 instance starts, runs your job, and automatically terminates when done.

## Full Example

For more control over instance configuration:

```yaml
name: GPU Tests

on: [push, pull_request]

jobs:
  ec2:
    uses: Open-Athena/ec2/.github/workflows/runner.yml@main
    secrets: inherit  # Requires `AWS_ROLE`, `GH_SA_TOKEN`
    with:
      aws_instance_type: "g4dn.xlarge"  # Optional, defaults to g4dn.xlarge

  test:
    needs: ec2
    runs-on: ${{ needs.ec2.outputs.instance }}
    steps:
      - uses: actions/checkout@v4

      - name: Verify GPU availability
        run: nvidia-smi

      # No stop job needed - instance self-terminates!
```

## Configuration

### Workflow Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `aws_instance_type` | EC2 instance type | [`g4dn.xlarge`](https://instances.vantage.sh/aws/ec2/g4dn.xlarge) (cheapest GPU instance) |
| `aws_image_id` | AMI ID | `ami-00096836009b16a22` (Deep Learning AMI) |
| `aws_home_dir` | Home directory path | `/home/ubuntu` |
| `aws_key_name` | EC2 key pair name for SSH access | - |
| `aws_security_group_id` | Security group ID (must allow SSH if using) | - |
| `ssh_pubkey` | Additional SSH public key to authorize | - |
| `shutdown_poll_wait` | Minutes to wait for runner setup before monitoring | `3` |
| `poll_interval` | Seconds between runner process checks | `15` |

### Environment Variables

Set these as organization or repository variables for defaults:

- `EC2_IMAGE_ID` - Default AMI ID
- `EC2_INSTANCE_TYPE` - Default instance type
- `EC2_KEY_NAME` - Default SSH key pair name
- `EC2_SECURITY_GROUP_ID` - Default security group ID

Priority: workflow inputs > environment variables > hardcoded defaults

## How It Works

1. The workflow starts an EC2 instance using the specified configuration
2. A GitHub Actions runner is automatically installed and registered
3. Your job runs on the EC2 instance
4. A systemd service monitors the runner process
5. When the runner stops (job completes), the instance self-terminates automatically

## Security

### AWS Authentication
This workflow uses GitHub OIDC for AWS authentication, eliminating the need for long-lived credentials. Ensure your AWS IAM role is properly configured to trust only your GitHub organization/repository.

### Public repos: set "Require approval for all external contributors", don't approve
If you want to use this action on a public repo, you need to protect against external contributors triggering workflows with access to your `AWS_ROLE` secret.

Recommended settings / protocol:
1. In your repository settings, enable "Require approval for all external contributors" under "Actions" → "General".
2. **Never directly approve workflow runs** from external contributors. Instead:
   - Create a temporary branch from the external contributor's commit.
   - Trigger the workflow on this temporary branch, using a `workflow_dispatch` event (from the web UI or `gh` CLI)
   - Delete the branch when done.

This ensures external contributors never gain persistent workflow execution rights.

## SSH Debugging

### Initial Setup

1. **Create an EC2 key pair**:
```bash
# Create key pair and save private key
aws ec2 create-key-pair --key-name gha --key-type ed25519 \
  | jq -r .KeyMaterial > ~/.ssh/gha.pem
chmod 600 ~/.ssh/gha.pem

# View the public key
ssh-keygen -y -f ~/.ssh/gha.pem
```

2. **Create a security group with SSH access**:
```bash
# Create security group
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name gha-runner-ssh \
  --description "GitHub Actions runner with SSH access" \
  --query 'GroupId' --output text)

# Allow SSH from anywhere (or restrict --cidr to your IP)
aws ec2 authorize-security-group-ingress \
  --group-id $SECURITY_GROUP_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

echo "Security Group ID: $SECURITY_GROUP_ID"
```

3. **Set organization/repository variables**:
```bash
# Set at org level
gh variable set EC2_KEY_NAME --org Open-Athena --body "gha"
gh variable set EC2_SECURITY_GROUP_ID --org Open-Athena --body "$SECURITY_GROUP_ID"

# Or at repo level
gh variable set EC2_KEY_NAME --body "gha"
gh variable set EC2_SECURITY_GROUP_ID --body "$SECURITY_GROUP_ID"
```

### Connecting to Running Instances

1. **Find the latest instance**:
```bash
# Get the most recent running instance with gha# name
INSTANCE_INFO=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=gha#*" \
           "Name=instance-state-name,Values=running" \
  --query 'sort_by(Reservations[].Instances[], &LaunchTime)[-1].[PublicDnsName,InstanceId,Tags[?Key==`Name`].Value|[0]]' \
  --output text)

INSTANCE_DNS=$(echo "$INSTANCE_INFO" | cut -f1)
INSTANCE_ID=$(echo "$INSTANCE_INFO" | cut -f2)
INSTANCE_NAME=$(echo "$INSTANCE_INFO" | cut -f3)

echo "Connecting to $INSTANCE_NAME ($INSTANCE_ID)"
ssh -i ~/.ssh/gha.pem ubuntu@$INSTANCE_DNS
```

2. **Optional: Configure SSH client for easier access**:
```bash
# Add to ~/.ssh/config
Host gha
    User ubuntu
    IdentitiesOnly yes
    IdentityFile ~/.ssh/gha.pem
    StrictHostKeyChecking no
    ForwardAgent yes

# Then connect with:
ssh gha -o HostName=$INSTANCE_DNS
```

### Key Debugging Locations

Once connected to the instance:

```bash
# 1. Self-termination daemon logs (most important!)
sudo tail -f /var/log/github-runner-cleanup.log
# Shows: startup time, poll interval, wait duration, process monitoring

# 2. Daemon status
sudo systemctl status github-runner-cleanup.service
# Shows: if service is running, PID, memory usage

# 3. Runner setup logs
sudo cat /var/log/runner-setup.log
# Shows: SSH key installation, userdata completion

# 4. Full userdata execution log
sudo cat /var/log/cloud-init-output.log
# Shows: complete boot process, runner download/registration

# 5. GitHub Actions runner logs
ls -la /home/ubuntu/_diag/
cat /home/ubuntu/_diag/Runner_*.log
# Shows: runner connection status, job execution

# 6. Check running processes
ps aux | grep -E "Runner|runner"
# Shows: if runner is actively processing a job

# 7. View the cleanup script itself
sudo cat /usr/local/bin/github-runner-cleanup.sh
```

## Troubleshooting

### Instance doesn't terminate

If the instance doesn't terminate after the workflow completes:

1. **Connect to the instance** using AWS Systems Manager:
   ```bash
   aws ssm start-session --target i-xxxxx --region us-east-1
   ```

2. **Check the cleanup service logs**:
   ```bash
   sudo journalctl -u github-runner-cleanup
   sudo cat /var/log/github-runner-cleanup.log
   ```

3. **Verify the service status**:
   ```bash
   sudo systemctl status github-runner-cleanup
   ```

4. **Check termination behavior**:
   ```bash
   aws ec2 describe-instance-attribute \
     --instance-id $(ec2-metadata --instance-id | cut -d " " -f 2) \
     --attribute instanceInitiatedShutdownBehavior \
     --region us-east-1
   ```

**Note**: The cleanup service waits for the runner to start (default 3 minutes, configurable via `shutdown_poll_wait`). It checks every 10 seconds during this period and begins monitoring immediately when the runner is detected. Once monitoring begins, it checks every 15 seconds if the runner is still active. When the job completes and the runner stops, the instance typically terminates within 30-45 seconds.

### Runner not connecting
- Verify `GH_SA_TOKEN` has correct permissions
- Check security group allows outbound HTTPS
- Ensure the AMI is compatible with GitHub Actions runner

## License

MIT
