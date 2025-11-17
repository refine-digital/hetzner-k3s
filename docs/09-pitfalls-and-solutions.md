# Potential Pitfalls and Solutions
**Part:** 9 of 11

## Potential Pitfalls to Avoid

### 1. Network Interface Detection Timing

**Problem:** Cloud-init script may fail if private network interface isn't ready

**hetzner-k3s Solution:** 30 attempts × 10s delay (5 min timeout)

**Your Solution:** Implement same retry logic in cloud-init template

```bash
MAX_ATTEMPTS=30
for i in $(seq 1 $MAX_ATTEMPTS); do
  INTERFACE=$(ip -o link | awk '/mtu (1450|1280)/ {print $2}')
  [ -n "$INTERFACE" ] && break
  sleep 10
done
```

### 2. Firewall Rule Limit

**Problem:** Hetzner Cloud Firewalls have 50-rule hard limit

**hetzner-k3s Solution:** Validates rule count before creation

**Your Solution:**
- Use CIDR ranges instead of individual IPs
- Consolidate custom rules
- Add validation in Terraform

```hcl
locals {
  total_rules = (
    length(var.ssh_allowed_networks) +
    length(var.api_allowed_networks) +
    length(var.custom_firewall_rules) +
    10  # Base rules
  )
}

resource "null_resource" "validate_firewall_rules" {
  count = local.total_rules > 50 ? "ERROR: Firewall rules exceed 50 limit" : 0
}
```

### 3. Concurrent Instance Creation

**Problem:** Creating many instances simultaneously can hit rate limits

**hetzner-k3s Solution:** Semaphore limiting to 10 concurrent creates

**Your Solution:** Use Terraform's parallelism flag

```bash
terraform apply -parallelism=10
```

### 4. etcd Cluster Size

**Problem:** Even number of masters causes split-brain risk

**hetzner-k3s Solution:** Documentation recommends 3, 5, or 7 masters

**Your Solution:** Add validation

```hcl
variable "master_count" {
  type = number
  validation {
    condition     = var.master_count == 1 || var.master_count % 2 == 1
    error_message = "Master count must be 1 or odd number (3, 5, 7) for HA."
  }
}
```

### 5. Kubeconfig Retrieval

**Problem:** Kubeconfig not immediately available after instance creation

**hetzner-k3s Solution:** SSH polling with 20 retries × 5s

**Your Solution:** Use Terraform provisioner with retry

```hcl
resource "null_resource" "get_kubeconfig" {
  depends_on = [hcloud_server.master]

  provisioner "remote-exec" {
    inline = [
      "timeout 120 bash -c 'until [ -f /etc/rancher/k3s/k3s.yaml ]; do sleep 5; done'",
      "cat /etc/rancher/k3s/k3s.yaml"
    ]

    connection {
      type        = "ssh"
      host        = hcloud_server.master.ipv4_address
      user        = "root"
      private_key = file(var.ssh_private_key_path)
    }
  }
}
```

### 6. Load Balancer Readiness

**Problem:** LB created but public IP not assigned immediately

**hetzner-k3s Solution:** Poll until public_ip_address is not nil

**Your Solution:** Terraform handles this automatically, but add explicit depends_on

```hcl
resource "null_resource" "wait_for_lb" {
  depends_on = [hcloud_load_balancer.api]

  provisioner "local-exec" {
    command = "sleep 10"  # LB needs time to initialize
  }
}
```

### 7. State File Management

**Problem:** Accidentally using same state for multiple clients

**Your Solution:**
- Never use default local backend
- Unique S3 key per client: `clients/<client-id>/terraform.tfstate`
- Add safeguards in backend config

```hcl
terraform {
  backend "s3" {
    bucket = "your-terraform-state"
    key    = "clients/${var.client_id}/terraform.tfstate"  # Interpolation not allowed in backend!
    # Solution: Use separate backend.tf per environment
  }
}
```

### 8. Secrets in State

**Problem:** Hetzner tokens, k3s tokens stored in Terraform state

**Your Solution:**
- Encrypt state at rest (S3 encryption, Terraform Cloud)
- Restrict state file access (IAM policies, RBAC)
- Never commit state files to git
- Use `sensitive = true` on outputs

```hcl
output "k3s_token" {
  value     = random_password.k3s_token.result
  sensitive = true
}
```

---

## Navigation

**Previous:** [08-cost-estimation.md](./08-cost-estimation.md)
**Next:** [10-quick-reference.md](./10-quick-reference.md)
**Back to:** [ARCHITECTURE-REVIEW.md](./ARCHITECTURE-REVIEW.md)
