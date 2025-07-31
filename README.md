# Open-Athena/ec2
Self-terminating EC2 GHA runner.

Demo workflows: [Open-Athena/ec2-runner-demo].

## Features

- Starts EC2 instances on-demand for GitHub Actions jobs
- Self-terminates when job completes (no separate stop job needed)
- Uses GitHub OIDC for AWS authentication (no long-lived credentials)
- Single reusable workflow call
- Defaults to [`g4dn.xlarge`] (cheapest EC2 GPU instance we're aware of) and `ami-00096836009b16a22` (`amazon/Deep Learning OSS Nvidia Driver AMI GPU PyTorch 2.4.1 (Ubuntu 22.04) 20250302`)

## Setup

### 1. Configure AWS IAM Role

Here's an example [Pulumi] recipe to create the necessary AWS IAM role:

<details>
<summary>Pulumi example</summary>

Update `ORGS_REPOS_UPDATEME` below with the repos/orgs you want the role to be accessible from:

```python
"""Create AWS_ROLE used to launch EC2 instances in GitHub Actions workflows."""

import pulumi
import pulumi_aws as aws


current = aws.get_caller_identity()

# Create IAM OIDC provider for GitHub Actions
# fingerprint instructions: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc_verify-thumbprint.html
#
# ```console
# $ d=token.actions.githubusercontent.com
# $ openssl s_client -servername $d -showcerts -connect $d:443 </dev/null | \
#   perl -0777 -ne 'print (pop @{[ /-----BEGIN CERTIFICATE-----\n.*?-----END CERTIFICATE-----/gs ]})' | \
#   openssl x509 -fingerprint -sha1 -noout | \
#   sed 's/.*=//' | \
#   tr -d : | \
#   tr '[:upper:]' '[:lower:]'
# 2b18947a6a9fc7764fd8b5fb18a863b0c6dac24f
# ```
github_oidc_provider = aws.iam.OpenIdConnectProvider(
    "github-actions",
    client_id_lists=["sts.amazonaws.com"],
    thumbprint_lists=["2b18947a6a9fc7764fd8b5fb18a863b0c6dac24f"],
    url="https://token.actions.githubusercontent.com",
)

# Recommended policy from gha-runner
# https://github.com/omsf/gha-runner/blob/main/docs/aws.md#prepare-a-policy
ec2_policy = aws.iam.Policy("github-actions-ec2-policy",
    policy="""{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "ec2:RunInstances",
                    "ec2:TerminateInstances",
                    "ec2:DescribeInstances",
                    "ec2:DescribeInstanceStatus",
                    "ec2:DescribeImages",
                    "ec2:CreateTags"
                ],
                "Resource": "*"
            }
        ]
    }"""
)

ORGS_REPOS_UPDATEME = [
   "org1/repo1",
   "org2/*",
]

# Create IAM role that GitHub Actions can assume, one per repo
for index, repo in enumerate(ORGS_REPOS_UPDATEME):
    github_actions_role = aws.iam.Role(f"github-actions-role-{index}",
        assume_role_policy=f"""{{
            "Version": "2012-10-17",
            "Statement": [
                {{
                    "Effect": "Allow",
                    "Principal": {{
                        "Federated": "arn:aws:iam::{current.account_id}:oidc-provider/token.actions.githubusercontent.com"
                    }},
                    "Action": "sts:AssumeRoleWithWebIdentity",
                    "Condition": {{
                        "StringLike": {{
                            "token.actions.githubusercontent.com:sub": "repo:{repo}:*"
                        }}
                    }}
                }}
            ]
        }}"""
    )

    # Attach the custom EC2 policy
    ec2_policy_attachment = aws.iam.RolePolicyAttachment(f"github-actions-ec2-policy-attachment-{index}",
        role=github_actions_role.name,
        policy_arn=ec2_policy.arn
    )

    # Export the role ARN
    pulumi.export(f"github_actions_role_arn_{repo}", github_actions_role.arn)
```
</details>

The role must be able to launch, tag, describe, and shutdown instances, and should be integrated with GitHub's OIDC provider (see [GitHub's documentation][gh oidc] for more info).

### 2. Configure Secrets and Variables

#### Required Secret: `GH_SA_TOKEN`
This workflow requires a GitHub token with admin permissions to the repo it's run within, because the underlying `gha-runner` [calls `/actions/runners/registration-token`][call], whose [docs] state:

> Authenticated users must have admin access to the repository to use this endpoint.

[call]: https://github.com/Open-Athena/gha-runner/blob/v1/src/gha_runner/gh.py#L144-L146
[docs]: https://docs.github.com/en/rest/actions/self-hosted-runners?apiVersion=2022-11-28#create-a-registration-token-for-a-repository

#### Required Variable (or pass as input):
- `AWS_ROLE`: ARN of your AWS IAM role (e.g. `arn:aws:iam::123456789012:role/GitHubActionsRole`)

Set this in your GitHub organization or repository settings by e.g.:

```bash
gh variable set AWS_ROLE --body "arn:aws:iam::123456789012:role/GitHubActionsRole"
```

## Minimal Example

Here's a minimal workflow that exercises a GPU on an EC2 instance:

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

This launches an EC2 instance, runs the `gpu-test` job on it, and automatically terminates when finished.

## Configuration

### Workflow Inputs

| Input                   | Description | Default |
|-------------------------|-------------|---------|
| `aws_role`              | AWS role ARN for EC2 operations | `vars.AWS_ROLE` |
| `ec2_instance_type`     | EC2 instance type | [`g4dn.xlarge`](https://instances.vantage.sh/aws/ec2/g4dn.xlarge) (cheapest GPU instance) |
| `ec2_image_id`          | AMI ID | `ami-00096836009b16a22` (Deep Learning AMI) |
| `ec2_home_dir`          | Home directory path | `/home/ubuntu` |
| `runner_grace_period`   | Seconds to wait before terminating after last job | `30` |
| `ec2_key_name`          | EC2 key pair name for SSH access | - |
| `ec2_security_group_id` | Security group ID (must allow SSH if using) | - |
| `ssh_pubkey`            | Additional SSH public key to authorize | - |

### Environment Variables

Set these as organization or repository variables for defaults:

- `AWS_ROLE` - AWS role ARN for EC2 operations (required if not passed as input)
- `EC2_IMAGE_ID` - Default AMI ID
- `EC2_INSTANCE_TYPE` - Default instance type
- `EC2_HOME_DIR` - Default home directory path (should match the default AMI)
- `EC2_KEY_NAME` - Default SSH key pair name
- `EC2_SECURITY_GROUP_ID` - Default security group ID
- `SSH_PUBKEY` - Default SSH public key to add to instances

Priority: workflow inputs > environment variables > hardcoded defaults

## How It Works

1. The workflow launches an EC2 instance using the specified configuration
2. A GitHub Actions runner is installed and registered with runner hooks
3. Your job(s) run on the EC2 instance
4. After each job completes, the runner waits for the grace period before checking for termination
5. If no new jobs start within the grace period, the instance terminates

## Multi-Job Workflows

This workflow supports running multiple sequential jobs on the same EC2 instance:

```yaml
jobs:
  ec2:
    uses: Open-Athena/ec2/.github/workflows/runner.yml@main
    secrets: inherit
    with:
      runner_grace_period: "120"  # 2 minutes for multi-job workflows

  prepare:
    needs: ec2
    runs-on: ${{ needs.ec2.outputs.instance }}
    steps:
      - run: echo "Setting up environment"

  train:
    needs: [ec2, prepare]
    runs-on: ${{ needs.ec2.outputs.instance }}
    steps:
      - run: echo "Training on GPU"

  evaluate:
    needs: [ec2, train]
    runs-on: ${{ needs.ec2.outputs.instance }}
    steps:
      - run: echo "Evaluating results"
```

The `runner_grace_period` parameter controls how long to wait after a job completes:
- **Default (30s)**: Suitable for single-job workflows
- **60-120s**: Recommended for multi-job workflows
- **Higher values**: For workflows with slow job startup times

## Security

### AWS Authentication
This workflow assumes GitHub OIDC for AWS authentication, eliminating the need for long-lived credentials. Ensure your AWS IAM role is properly configured to trust only your GitHub organization/repository.

### Public repos: set "Require approval for all external contributors", don't approve
If you want to use this action on a public repo, you should restrict external contributors from triggering workflows that can access your `AWS_ROLE` secret.

The best way to do this is to enable "Require approval for all external contributors" under "Actions" → "General".

## SSH Debugging <a id="ssh-debugging"></a>

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

These can also be passed as `inputs` to the workflow.

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

1. **Connect to the instance** using SSH

   (This requires having provided an `ec2_key_name` (or `ssh_pubkey`) and `ec2_security_group`; see [SSH Debugging](#ssh-debugging) above).

2. **Check the termination hook logs**:
   ```bash
   sudo cat /var/log/runner-termination-hook.log
   ```

3. **Verify the runner hooks are configured**:
   ```bash
   cat ~/actions-runner/.env
   ```

4. **Check termination behavior**:
   ```bash
   aws ec2 describe-instance-attribute \
     --instance-id $(ec2-metadata --instance-id | cut -d " " -f 2) \
     --attribute instanceInitiatedShutdownBehavior \
     --region us-east-1
   ```

**Note**: The instance uses GitHub's native runner hooks feature. The `ACTIONS_RUNNER_HOOK_JOB_COMPLETED` environment variable is configured to run a termination script immediately after each job completes. This ensures reliable and immediate cleanup without the need for polling or monitoring services.

[`g4dn.xlarge`]: https://instances.vantage.sh/aws/ec2/g4dn.xlarge
[Pulumi]: https://www.pulumi.com
[Open-Athena/ec2-runner-demo]: https://github.com/Open-Athena/ec2-runner-demo
[aws-actions/configure-aws-credentials@v4]: https://github.com/aws-actions/configure-aws-credentials/tree/v4
[gh oidc]: https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
