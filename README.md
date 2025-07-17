# Open-Athena/ec2

Self-terminating EC2 runner for GitHub Actions with minimal boilerplate.

## Features

- 🚀 Starts EC2 instances on-demand for GitHub Actions jobs
- 🧹 Self-terminates when job completes (no separate stop job needed!)
- 🔑 Uses GitHub OIDC for AWS authentication (no long-lived credentials)
- ⚡ Single reusable workflow call
- 🎯 GPU-optimized AMI by default
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

### 3. Create Approval Label (Optional)

If you want to allow trusted external contributors to run GPU tests:
1. Go to your repository's Issues tab
2. Click Labels → New label
3. Create a label named `gpu` (or your custom name)
4. Maintainers can apply this label to PRs to authorize GPU runs

## Usage

In your workflow:

```yaml
name: GPU Tests

on: [push, pull_request]

jobs:
  ec2:
    uses: Open-Athena/ec2/.github/workflows/runner.yml@v1
    secrets: inherit
    with:
      aws_instance_type: "g6.xlarge"  # Optional, defaults to g6.xlarge

  test:
    needs: ec2
    runs-on: ${{ needs.ec2.outputs.instance }}
    steps:
      - uses: actions/checkout@v4

      - name: Run GPU tests
        run: |
          python -m pytest tests/gpu -v

      # No stop job needed - instance self-terminates!
```

## Configuration

| Input | Description | Default |
|-------|-------------|---------|
| `aws_instance_type` | EC2 instance type | `g6.xlarge` |
| `aws_image_id` | AMI ID | `ami-00096836009b16a22` (Deep Learning AMI) |
| `aws_home_dir` | Home directory path | `/home/ubuntu` |
| `approval_label` | Label name for external PR approval (empty to disable) | `gpu` |
| `shutdown_poll_wait` | Minutes to wait for runner setup before monitoring | `3` |

## How It Works

1. The workflow starts an EC2 instance using the specified configuration
2. A GitHub Actions runner is automatically installed and registered
3. Your job runs on the EC2 instance
4. A systemd service monitors the runner process
5. When the runner stops (job completes), the instance self-terminates automatically

## Security

### Fork Protection
This workflow includes built-in protection against unauthorized EC2 launches from forked repositories. EC2 instances will only start for:
- Direct pushes to your repository
- Manual workflow dispatches
- Pull requests from branches within your repository (not forks)
- Pull requests from forks that have the approval label (default: `gpu`) applied by a maintainer

This prevents unauthorized external contributors from using your AWS resources through pull requests.

### Label-Based Approval for External Contributors
For trusted external contributors, maintainers can approve GPU testing by:
1. Adding the approval label (default: `gpu`) to a pull request
2. The workflow will then run with EC2 instances
3. The label should be removed after the run completes

To customize the label name:
```yaml
jobs:
  ec2:
    uses: Open-Athena/ec2/.github/workflows/runner.yml@v1
    secrets: inherit
    with:
      approval_label: "run-benchmarks"  # Custom label name
```

To disable label-based approval entirely:
```yaml
with:
  approval_label: ""  # Empty string disables external PR approval
```

This follows the same pattern used by scikit-learn and other major open source projects for expensive CI resources.

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

**Note**: The cleanup service waits 3 minutes (configurable via `shutdown_poll_wait`) before starting to monitor the runner process.

### Runner not connecting
- Verify `GH_SA_TOKEN` has correct permissions
- Check security group allows outbound HTTPS
- Ensure the AMI is compatible with GitHub Actions runner

## License

MIT
