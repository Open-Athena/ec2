# Open-Athena/ec2

Self-terminating EC2 runner for GitHub Actions with minimal boilerplate.

## Features

- 🚀 Starts EC2 instances on-demand for GitHub Actions jobs
- 🧹 Self-terminates when job completes (no separate stop job needed!)
- 🔑 Uses GitHub OIDC for AWS authentication (no long-lived credentials)
- ⚡ Single reusable workflow call
- 🎯 GPU-optimized AMI by default

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

## How It Works

1. The workflow starts an EC2 instance using the specified configuration
2. A GitHub Actions runner is automatically installed and registered
3. Your job runs on the EC2 instance
4. A systemd service monitors the runner process
5. When the runner stops (job completes), the instance self-terminates after 2 minutes

## Troubleshooting

### Instance doesn't terminate
- Check CloudWatch logs for the `github-runner-cleanup` service
- Verify the runner process name matches `Runner.Listener`
- Ensure the instance has permissions to terminate itself

### Runner not connecting
- Verify `GH_SA_TOKEN` has correct permissions
- Check security group allows outbound HTTPS
- Ensure the AMI is compatible with GitHub Actions runner

## License

MIT