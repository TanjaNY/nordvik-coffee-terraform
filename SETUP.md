# Setup Guide - Nordvik Coffee Infrastructure

This guide explains how to set up, deploy, and work with the Nordvik Coffee infrastructure.

## Prerequisites

### Required Tools

- **AWS Account** with administrator access
- **AWS CLI** v2 or later ([Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html))
- **Terraform** v1.5.0 or later ([Download](https://www.terraform.io/downloads))
- **Git** ([Download](https://git-scm.com/downloads))

### AWS Permissions Required

Your AWS user needs permissions for:
- IAM (roles, policies, OIDC providers)
- S3 (buckets, objects)
- DynamoDB (tables)
- Lambda (functions)
- API Gateway (HTTP APIs)
- CloudWatch Logs
- Budgets

## Initial Setup

### 1. Clone the Repository

```bash
git clone https://github.com/TanjaNY/nordvik-coffee-terraform.git
cd nordvik-coffee-terraform
```

### 2. Configure AWS Credentials

```bash
# Login to AWS (requires aws login configured)
aws login

# Verify credentials
aws sts get-caller-identity
```

You should see output like:
```json
{
    "UserId": "AIDAEXAMPLE",
    "Account": "581137949655",
    "Arn": "arn:aws:iam::581137949655:user/developer"
}
```

### 3. Initialize Terraform

```bash
# Initialize Terraform (downloads providers, configures backend)
terraform init

# Verify configuration
terraform validate
```

### 4. Review the Plan

```bash
# See what Terraform will create
terraform plan
```

Review the output carefully. You should see resources like:
- S3 buckets (website, documentation, state)
- DynamoDB table
- Lambda function
- API Gateway
- IAM roles
- Budget alerts

### 5. Deploy Infrastructure

```bash
# Apply the configuration
terraform apply

# Type 'yes' when prompted
```

Deployment takes 2-3 minutes. When complete, you'll see outputs:

```
Outputs:

api_endpoint = "https://xxxxx.execute-api.eu-central-1.amazonaws.com/reviews"
bucket_name = "nordvik-coffee-xxxxxx"
website_url = "http://nordvik-coffee-xxxxxx.s3-website.eu-central-1.amazonaws.com"
```

### 6. Test the Website

Open the `website_url` in your browser. You should see the Nordvik Coffee homepage.

## Working with the Infrastructure

### Making Changes

1. **Edit Terraform files** (e.g., `lambda.tf`, `s3.tf`)
2. **Review changes**: `terraform plan`
3. **Apply changes**: `terraform apply`
4. **Commit to Git**:
   ```bash
   git add .
   git commit -m "Description of changes"
   git push
   ```

### Updating the Website

Edit `website/index.html.tpl`:

```bash
# Make your changes
nano website/index.html.tpl

# Apply
terraform apply

# The website will be automatically updated
```

### Updating the Lambda Function

Edit `lambda_src/lambda_function.py`:

```bash
# Make your changes
nano lambda_src/lambda_function.py

# Apply (Terraform will re-zip and deploy)
terraform apply
```

### Viewing Logs

```bash
# Lambda logs
aws logs tail /aws/lambda/nordvik-coffee-reviews-handler --follow

# View specific time range
aws logs tail /aws/lambda/nordvik-coffee-reviews-handler --since 1h
```

### Checking DynamoDB Data

```bash
# Scan all reviews
aws dynamodb scan --table-name nordvik-coffee-reviews

# Get item count
aws dynamodb describe-table --table-name nordvik-coffee-reviews \
  --query 'Table.ItemCount'
```

### Checking Costs

```bash
# View current budget status
aws budgets describe-budget \
  --account-id 581137949655 \
  --budget-name nordvik-coffee-monthly-budget

# View current month's costs
aws ce get-cost-and-usage \
  --time-period Start=2026-11-01,End=2026-11-30 \
  --granularity MONTHLY \
  --metrics BlendedCost
```

## CI/CD Pipeline

### How It Works

The GitHub Actions workflow automatically runs when you push to `main`:

1. ✅ **Checkout code** from GitHub
2. ✅ **Authenticate** with AWS using OIDC (secure, no stored credentials)
3. ✅ **Initialize** Terraform with remote state from S3
4. ✅ **Validate** Terraform configuration
5. ✅ **Plan** infrastructure changes
6. ℹ️ **Does NOT apply** (plan-only for safety)

### Viewing Workflow Runs

Go to: https://github.com/TanjaNY/nordvik-coffee-terraform/actions

### Understanding the Workflow

The workflow is configured in `.github/workflows/terraform.yml`:

- **On Push to main**: Runs `terraform plan` to show what would change
- **On Pull Request**: Runs plan and posts results as PR comment
- **Manual**: Can be triggered manually from Actions tab

**Important**: The workflow does NOT automatically apply changes. This is intentional for educational safety. To deploy, run `terraform apply` locally.

### Enabling Auto-Deploy (Optional)

To enable automatic deployment:

1. Uncomment the `apply` job in `.github/workflows/terraform.yml`
2. Configure GitHub Environment protection rules for manual approval
3. Push the changes

**Warning**: Only do this if you understand the risks of automatic deployments.

## Terraform State Management

### Remote State in S3

The Terraform state is stored in S3 bucket: `nordvik-coffee-terraform-state`

**Benefits**:
- ✅ Shared state between team members
- ✅ State versioning enabled (can rollback)
- ✅ Encrypted at rest
- ✅ Works with CI/CD

### Viewing State

```bash
# Show current state
terraform show

# List all resources
terraform state list

# Show specific resource
terraform state show aws_s3_bucket.site
```

### State Backup

State is automatically versioned in S3. To download a backup:

```bash
aws s3 cp s3://nordvik-coffee-terraform-state/terraform.tfstate ./backup.tfstate
```

## Troubleshooting

### AWS Credentials Expired

**Problem**: `ExpiredToken` error

**Solution**:
```bash
aws login
```

### Terraform State Locked

**Problem**: "Error acquiring the state lock"

**Solution**:
Wait for other operations to complete, or force unlock (use with caution):
```bash
terraform force-unlock <LOCK_ID>
```

### GitHub Actions OIDC Fails

**Problem**: "Not authorized to perform sts:AssumeRoleWithWebIdentity"

**Solution**:
1. Check IAM role trust policy includes GitHub repository
2. Verify OIDC provider exists
3. Check role ARN in workflow file matches actual role

### Website Returns 403 Forbidden

**Problem**: Cannot access website

**Solution**:
```bash
# Check bucket policy
aws s3api get-bucket-policy --bucket nordvik-coffee-XXXXXX

# Verify public access settings
aws s3api get-public-access-block --bucket nordvik-coffee-XXXXXX
```

### Lambda Function Errors

**Problem**: API returns 500 errors

**Solution**:
```bash
# Check Lambda logs
aws logs tail /aws/lambda/nordvik-coffee-reviews-handler --follow

# Test Lambda directly
aws lambda invoke \
  --function-name nordvik-coffee-reviews-handler \
  --payload '{"httpMethod":"GET","path":"/reviews"}' \
  response.json

# View response
cat response.json
```

### DynamoDB Access Denied

**Problem**: Lambda can't write to DynamoDB

**Solution**:
Check Lambda IAM role has DynamoDB permissions:
```bash
aws iam get-role-policy \
  --role-name nordvik-coffee-reviews-lambda-role \
  --policy-name nordvik-coffee-reviews-dynamodb-access
```

## Clean Up (Destroying Infrastructure)

**Warning**: This will delete ALL resources and data!

```bash
# Review what will be destroyed
terraform plan -destroy

# Destroy all resources
terraform destroy

# Type 'yes' to confirm
```

**Note**: The state bucket must be manually deleted:
```bash
# Empty the bucket first
aws s3 rm s3://nordvik-coffee-terraform-state --recursive

# Delete the bucket
aws s3 rb s3://nordvik-coffee-terraform-state
```

## Best Practices

### 1. Always Review Plans

```bash
terraform plan    # Review before applying
terraform apply   # Apply changes
```

### 2. Use Meaningful Commit Messages

```bash
git commit -m "Add validation to review rating field"
# Not: "fix stuff"
```

### 3. Test Locally Before Pushing

```bash
terraform validate  # Check syntax
terraform fmt       # Format code
terraform plan      # Review changes
```

### 4. Monitor Costs

- Check AWS Budget alerts (sent to ta.nyberg@gmail.com)
- Review Cost Explorer monthly
- Current budget: $75/month

### 5. Back Up Important Data

- DynamoDB has Point-in-Time Recovery (35 days)
- Terraform state is versioned in S3
- Git repository is your source of truth

### 6. Document Changes

- Update README.md for major changes
- Add comments to complex Terraform code
- Document any manual steps in SETUP.md

## Additional Resources

- [Terraform AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Project Documentation Hub](https://nordvik-coffee-project-docs.s3.eu-central-1.amazonaws.com/docs/index.html)

## Getting Help

1. **Check logs**: CloudWatch Logs for Lambda errors
2. **Review state**: `terraform show` to see current infrastructure
3. **Verify AWS**: Use AWS Console to inspect resources
4. **Check workflow**: GitHub Actions logs show deployment issues

## For Students

This infrastructure demonstrates:
- **Infrastructure as Code** with Terraform
- **Serverless architecture** on AWS
- **CI/CD pipelines** with GitHub Actions
- **Security best practices** (OIDC, IAM roles)
- **Cost management** with AWS Budgets
- **State management** with remote backends

Each component is documented and can be studied independently. Experiment, break things, and learn!

---

**Last Updated**: November 2026  
**Repository**: https://github.com/TanjaNY/nordvik-coffee-terraform  
**Maintainer**: Tanja Nyberg
