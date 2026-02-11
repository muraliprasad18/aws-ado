# Azure DevOps → AWS via OIDC (Terraform + Pipelines)

This repository bootstraps a secure **Azure DevOps → AWS federation using OpenID Connect (OIDC)** with **no long-lived AWS keys**.

## What you get
- **AWS**: IAM OIDC provider for Azure DevOps + a locked‑down IAM role (trust is bound to one ADO service connection via `sub`).
- **Azure DevOps**: An **AWS** service connection configured with **`use_oidc = true`** via the Azure DevOps Terraform provider (requires the **AWS Toolkit for Azure DevOps** extension).
- **Pipelines**: A minimal verification pipeline and a 3‑stage Terraform pipeline (validate → plan → apply with approvals).
- **(Optional)** Terraform backend bootstrap (S3 + DynamoDB) for state.

## Prerequisites
1. **Install AWS Toolkit for Azure DevOps** in your ADO organization (enables AWS service connection & tasks).
2. **Find your Azure DevOps Organization GUID (ORG_GUID)**, then construct the issuer: `https://vstoken.dev.azure.com/<ORG_GUID>`.
   - Quickest: run the `azure-pipelines-verify.yml` once and copy the issuer printed in task logs; or open `https://dev.azure.com/<org>/_apis/connectionData` and read `instanceId`.
3. An ADO PAT (or SPN) with rights to create service connections and manage pipelines.

## Structure
```
.
├─ bootstrap/            # (Optional) S3 + DynamoDB for Terraform backend
├─ aws/                  # AWS: OIDC provider + IAM role + example policy
├─ ado/                  # ADO: AWS service connection (OIDC) via TF
├─ checks/               # Terraform assertions to validate config
└─ pipelines/            # ADO YAML pipelines
```

## Quick start

### 0) (Optional) Backend
```bash
cd bootstrap
cp terraform.tfvars.example terraform.tfvars
# edit bucket/table names
terraform init && terraform apply -auto-approve
```

### 1) AWS side (OIDC + role)
```bash
cd ../aws
cp terraform.tfvars.example terraform.tfvars
# fill: ado_org_guid, ado_org_name, ado_project_name, ado_service_connection_name
terraform init && terraform apply -auto-approve
# Capture output: ado_role_arn
```

### 2) ADO side (service connection)
```bash
cd ../ado
terraform init
terraform apply -auto-approve       -var="azure_devops_org_url=https://dev.azure.com/<org>"       -var="azure_devops_pat=<PAT>"       -var="ado_project_id=<PROJECT_GUID>"       -var="ado_service_connection_name=aws-oidc"       -var="aws_role_arn=<FROM_AWS_OUTPUT>"
```

### 3) Validate in a pipeline
- Run `pipelines/azure-pipelines-verify.yml` in your project → should print an **assumed-role ARN**.
- Use `pipelines/azure-pipelines-terraform.yml` for gated deployments.

## Multi-account
Replicate the **aws** module with provider aliases and create one ADO service connection per AWS account.

## Push to GitHub
```bash
git init
git add .
git commit -m "ado->aws via oidc"
gh repo create <owner>/<repo> --public --source=. --remote=origin --push
# or manually add origin and push
```

## License
MIT
