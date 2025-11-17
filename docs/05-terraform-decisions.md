# Terraform Conversion: Key Decisions
**Part:** 5 of 11

## Terraform Conversion: Key Decisions

### Decision 1: Cloud-Init Template Management

**Option A: Inline Template (Simpler)**
```hcl
resource "hcloud_server" "master" {
  user_data = templatefile("${path.module}/templates/cloud-init.yaml.tpl", {
    k3s_version = var.k3s_version
    # ... other variables
  })
}
```

**Option B: Template Module (More Flexible)**
```hcl
module "cloud_init" {
  source = "./modules/k3s-cloud-init"
  # ... variables
}

resource "hcloud_server" "master" {
  user_data = module.cloud_init.rendered
}
```

**Recommendation:** Option A for single-node, Option B for complex multi-pool setups

### Decision 2: Multi-Tenancy State Management

**Option A: Separate Workspaces (Not Recommended)**
```bash
terraform workspace new client-a
terraform workspace new client-b
```
❌ Shared state file, risk of cross-contamination

**Option B: Separate Backends (Recommended)**
```hcl
# client-a/backend.tf
terraform {
  backend "s3" {
    bucket = "terraform-state"
    key    = "clients/client-a/prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```
✅ Complete isolation, separate state files

**Recommendation:** Option B with unique S3 keys per client

### Decision 3: Hetzner Token Management

**Option A: Environment Variables (Development)**
```bash
export HCLOUD_TOKEN="client-a-token"
terraform apply
```

**Option B: Terraform Variables (CI/CD)**
```hcl
variable "hcloud_token" {
  type      = string
  sensitive = true
}

provider "hcloud" {
  token = var.hcloud_token
}
```

**Option C: Secret Management (Production)**
- Terraform Cloud: Workspace variables (encrypted)
- Vault: Dynamic secrets with rotation
- GitHub Secrets: For GitHub Actions

**Recommendation:** Option C for production, Option A for local dev

---

**Previous:** [04-technical-details.md](./04-technical-details.md)
**Next:** [06-arm-considerations.md](./06-arm-considerations.md)
