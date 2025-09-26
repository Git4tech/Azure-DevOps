
**Azure-DevOps Project**

Demonstrates Terraform-based infrastructure provisioning and CI/CD automation for Azure.
This repository provides a clean, modular starting point for building and deploying Azure infrastructure using Terraform, automating deployments with GitHub Actions, and following DevOps best practices.

**Table of Contents**
- [Overview](#overview)
- [Architecture](#architecture)
- [Screenshots](#screenshots)
- [Prerequisites](#prerequisites)
- [Quick Start (Step-by-Step)](#quick-start-step-by-step)
- [Repository Structure](#repository-structure)
- [Terraform Usage](#terraform-usage)
- [GitHub Actions / CI-CD](#github-actions--ci-cd)
- [Best Practices & Recommendations](#best-practices--recommendations)
- [Cleanup](#cleanup)
- [Contributing](#contributing)
- [License](#license)

**Overview**

This repo shows how to manage Azure infrastructure as code using **Terraform (HCL)** and how to automate plan/apply flows using **GitHub Actions**. It is suitable as a learning resource or as a starter template for a Landing Zone / environment bootstrapping workflow.

Key focus areas:
- Remote Terraform state management (Azure Storage backend)
- Modular Terraform code and reusable modules
- CI/CD automation for infrastructure changes
- Secure credential handling via GitHub Secrets
- Example workload deployment (AKS / App Services) integration points


**Typical components in this repo's architecture:**
- Azure Resource Groups for environment separation (management / platform / workloads)
- Azure Storage Account & Container for Terraform remote state
- Hub-and-Spoke VNets for networking isolation
- AKS cluster (workloads) connected to spoke network
- RBAC & Policies applied at subscription/management-group level
- CI/CD pipeline (GitHub Actions) for terraform plan/apply


**Prerequisites**

- Azure subscription (Owner or Contributor role on target subscription)
- [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli) configured (`az login`)
- [Terraform 1.x](https://www.terraform.io/downloads)
- Git and a GitHub account
- Optional but recommended: [gh CLI](https://cli.github.com/) for managing secrets
- Optional: Docker (for AKS workload testing) and kubectl (for interacting with AKS)


**Quick Start (Step-by-Step)**

> These steps assume this repository contains a `00-bootstrap` directory (or similar) that creates the Terraform backend. Adapt names as needed.

1. **Clone the repo**
```bash
git clone https://github.com/Git4tech/Azure-DevOps.git
cd Azure-DevOps
```

2. **Log in to Azure**
```bash
az login
az account set --subscription "<YOUR_SUBSCRIPTION_ID>"
```

3. **(Optional) Create resource group and storage account for Terraform backend**
If your repo is configured to use an Azure backend, ensure the storage account/container exist. Example:
```bash
az group create --name rg-terraform-state --location eastus
az storage account create --name tfstate<yourunique> --resource-group rg-terraform-state --location eastus --sku Standard_LRS --kind StorageV2
az storage container create --name tfstate --account-name tfstate<yourunique>
```

4. **Create a service principal for CI/CD**
```bash
az ad sp create-for-rbac --name "http://lz-deployer" --role "Contributor" --scopes "/subscriptions/<SUBSCRIPTION_ID>" --sdk-auth > sp-lz-deployer.json
```
**Keep this JSON secret**. Add it to GitHub repository secrets as `AZURE_CREDENTIALS`.

With `gh`:
```bash
gh auth login
gh secret set AZURE_CREDENTIALS --body "$(cat sp-lz-deployer.json)"
```

5. **Initialize Terraform (example: 00-bootstrap)**
```bash
cd 00-bootstrap
terraform init -backend-config="resource_group_name=rg-terraform-state" -backend-config="storage_account_name=tfstate<yourunique>" -backend-config="container_name=tfstate" -backend-config="key=bootstrap.tfstate"
terraform plan -out tfplan
terraform apply tfplan
```

6. **Develop modules & features**
- Add modules under `modules/` for resource groups, networking, AKS, etc.
- Use workspaces or separate state keys per environment (dev/stage/prod).

7. **Push changes & open a PR**
- Create a feature branch, push, and open a Pull Request. The GitHub Actions workflow should run `terraform fmt`, `terraform validate`, and `terraform plan`.


**Repository Structure**

```
.
├── .github/
│   └── workflows/         # GitHub Actions workflows (plan/apply)
├── 00-bootstrap/          # (Optional) create backend resources (RG + Storage)
├── modules/               # Reusable Terraform modules
├── 01-platform/           # Management & platform resources
├── 02-networking/         # Networking (hub/spoke) module
├── 03-aks/                # AKS cluster module / sample app
├── scripts/               # Helper scripts (create-sp.sh, az-login.sh)
├── images/                # Placeholder screenshots & diagrams
└── README.md
```

**Terraform Usage & Workflow**

**Local workflow**
```bash
terraform init
terraform fmt
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

**Best practice tips**
- Use `terraform fmt` and `terraform validate` in CI.
- Use `tflint` and `checkov` or `tfsec` for policy/linting checks.
- Pin `azurerm` provider versions in `required_providers`.
- Use workspaces or a dedicated state key per environment.


**GitHub Actions / CI-CD**

This repo should include workflows that:
- Run on PRs: `terraform fmt`, `terraform validate`, `terraform plan` (store plan as artifact)
- Run on merge to protected branch `main`: manual approval or run `terraform apply` in a controlled manner

**Secrets needed**
- `AZURE_CREDENTIALS` — Service Principal JSON (SDK auth)
- `ARM_SUBSCRIPTION_ID`, `ARM_TENANT_ID` (optional if not using SDK auth)

**Sample job flow**
1. Checkout
2. Azure Login (`azure/login@v1`) using `AZURE_CREDENTIALS`
3. Setup Terraform (`hashicorp/setup-terraform@v2`)
4. Terraform init/plan/validate
5. Store plan artifact & comment on PR


**Best Practices & Recommendations**

- Keep your Terraform state secure and backed up (use Azure Storage with versioning if needed).
- Apply Role-Based Access Control (RBAC) and least privilege for both Azure and Kubernetes (AKS).
- Use modules to avoid duplicate code and simplify maintenance.
- Require PR reviews and protect `main` branch. Use `plan` artifacts for auditability.
- Document your architecture and keep diagrams up to date (save exported PNG/SVG in `docs/`).


**Cleanup**

To avoid costs, destroy resources when not needed:
```bash
cd 00-bootstrap
terraform destroy -auto-approve
# then for other modules
cd ../01-platform
terraform destroy -auto-approve
```
Also revoke the Service Principal if no longer used:
```bash
az ad sp delete --id http://lz-deployer
```


**Contributing**

Contributions are welcome. Please:
1. Fork the repo
2. Create a feature branch
3. Follow Terraform formatting & linting
4. Open a Pull Request describing changes and testing performed


**License**

This repository is provided under the **MIT License**. See the [LICENSE](LICENSE) file for details.
