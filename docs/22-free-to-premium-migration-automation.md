# FREE to PREMIUM Tier Migration Automation
**Part:** 22 of Multi-Tenant Deployment Series
**Last Updated:** 2025-11-17

## Table of Contents

1. [Migration Architecture Overview](#1-migration-architecture-overview)
2. [State Machine Design](#2-state-machine-design)
3. [Pre-Migration Phase](#3-pre-migration-phase)
4. [Infrastructure Provisioning](#4-infrastructure-provisioning)
5. [Email Migration Strategy](#5-email-migration-strategy)
6. [DNS Cutover Process](#6-dns-cutover-process)
7. [Application Migration](#7-application-migration)
8. [Rollback Strategy](#8-rollback-strategy)
9. [API Endpoints](#9-api-endpoints)
10. [User Experience Flow](#10-user-experience-flow)
11. [Monitoring & Alerting](#11-monitoring--alerting)
12. [Testing Strategy](#12-testing-strategy)

---

## 1. Migration Architecture Overview

### 1.1 High-Level Flow

```
┌─────────────────────────────────────────────────────────────┐
│                      User Action                             │
│            (Click "Upgrade to PREMIUM" button)              │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Payment Processing                         │
│         (Stripe/PayPal - Confirm subscription)              │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Infrastructure Provisioning                     │
│    - Terraform creates k3s cluster on Hetzner Cloud         │
│    - Deploy base services (DNS, monitoring, ingress)        │
│    - Configure networking and storage                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Data Migration                              │
│    - Email (IMAP sync with imapsync)                        │
│    - DNS records (export → import)                          │
│    - Applications (if any)                                  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   DNS Cutover                                │
│    - Update MX records (gradual with TTL management)        │
│    - Update A/AAAA records for applications                 │
│    - Verify propagation and health                          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Validation & Cleanup                            │
│    - Health checks (email delivery, app access)             │
│    - Decommission old resources (after grace period)        │
│    - User notification (success)                            │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Design Principles

1. **Zero Downtime** - Email and apps remain accessible during migration
2. **Gradual Cutover** - Progressive DNS changes with validation gates
3. **Automatic Rollback** - Instant fallback on failure detection
4. **Progress Transparency** - Real-time updates via WebSocket
5. **Idempotent Operations** - Safe to retry any step
6. **Data Integrity** - No lost emails or data during migration

### 1.3 Timeline Estimates

| Cluster Size | Total Time | User-Facing Downtime |
|--------------|-----------|----------------------|
| **Single CAX11** | 8-12 min | 0 seconds (hot cutover) |
| **3-node HA** | 12-18 min | 0 seconds |
| **With Email Migration** | +30-60 min | 0 seconds (parallel MX) |

---

## 2. State Machine Design

### 2.1 State Diagram

```
               START
                 │
                 ▼
         ┌─── PENDING_PAYMENT
         │        │
         │        ▼
         │  PAYMENT_CONFIRMED
         │        │
         │        ▼
         │  PROVISIONING_INFRASTRUCTURE
         │        │
         │        ├──────►[Terraform Execute]
         │        │                │
         │        │                ▼
         │        │          INFRASTRUCTURE_READY
         │        │                │
         │        ▼                ▼
         │  PREPARING_MIGRATION ───┘
         │        │
         │        ├──────► MIGRATING_EMAIL
         │        │              │
         │        ├──────► MIGRATING_DNS
         │        │              │
         │        └──────► MIGRATING_APPS
         │                       │
         │                       ▼
         │              CUTOVER_IN_PROGRESS
         │                       │
         │                       ▼
         │                  VALIDATING
         │                       │
         │         ┌─────────────┴──────────┐
         │         │                        │
         │         ▼                        ▼
         │    COMPLETED              FAILED ───► ROLLING_BACK
         │         │                                    │
         └─────────┴────────────────────────────────────┘
                   │
                   ▼
                 DONE
```

### 2.2 State Definitions

```typescript
enum MigrationState {
  // Initial states
  PENDING_PAYMENT = 'PENDING_PAYMENT',           // Waiting for payment confirmation
  PAYMENT_CONFIRMED = 'PAYMENT_CONFIRMED',       // Payment received, starting provisioning

  // Provisioning states
  PROVISIONING_INFRASTRUCTURE = 'PROVISIONING_INFRASTRUCTURE',  // Terraform running
  INFRASTRUCTURE_READY = 'INFRASTRUCTURE_READY', // Cluster created, services deployed

  // Migration states
  PREPARING_MIGRATION = 'PREPARING_MIGRATION',   // Preparing data for migration
  MIGRATING_EMAIL = 'MIGRATING_EMAIL',           // IMAP sync in progress
  MIGRATING_DNS = 'MIGRATING_DNS',               // DNS records being migrated
  MIGRATING_APPS = 'MIGRATING_APPS',             // Application data migrating

  // Cutover states
  CUTOVER_IN_PROGRESS = 'CUTOVER_IN_PROGRESS',   // DNS changes being applied
  VALIDATING = 'VALIDATING',                     // Post-cutover health checks

  // Terminal states
  COMPLETED = 'COMPLETED',                       // Migration successful
  FAILED = 'FAILED',                             // Migration failed, manual intervention needed
  ROLLING_BACK = 'ROLLING_BACK',                 // Automatic rollback in progress
  ROLLED_BACK = 'ROLLED_BACK'                    // Successfully rolled back
}
```

### 2.3 Database Schema

```sql
CREATE TABLE migrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id INTEGER NOT NULL,
    state VARCHAR(50) NOT NULL,
    started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    failed_at TIMESTAMP,
    rollback_at TIMESTAMP,

    -- Configuration
    source_tier VARCHAR(20) DEFAULT 'FREE',
    target_tier VARCHAR(20) DEFAULT 'PREMIUM',
    options JSONB,  -- Migration-specific options

    -- Infrastructure details
    cluster_id UUID,
    terraform_state_url TEXT,

    -- Progress tracking
    progress_percent INTEGER DEFAULT 0,
    current_step VARCHAR(100),
    steps_completed JSONB DEFAULT '[]'::jsonb,

    -- Error tracking
    error_message TEXT,
    error_stack TEXT,
    retry_count INTEGER DEFAULT 0,

    -- Rollback info
    rollback_reason TEXT,
    rollback_snapshot_id VARCHAR(255),

    FOREIGN KEY (client_id) REFERENCES clients(id),
    INDEX idx_client (client_id),
    INDEX idx_state (state),
    INDEX idx_started (started_at)
);

CREATE TABLE migration_logs (
    id SERIAL PRIMARY KEY,
    migration_id UUID NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    level VARCHAR(10),  -- DEBUG, INFO, WARN, ERROR
    message TEXT,
    metadata JSONB,
    FOREIGN KEY (migration_id) REFERENCES migrations(id) ON DELETE CASCADE,
    INDEX idx_migration (migration_id),
    INDEX idx_timestamp (timestamp)
);

CREATE TABLE migration_checkpoints (
    id SERIAL PRIMARY KEY,
    migration_id UUID NOT NULL,
    checkpoint_name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    data JSONB,  -- Checkpoint-specific data for rollback
    FOREIGN KEY (migration_id) REFERENCES migrations(id) ON DELETE CASCADE,
    INDEX idx_migration (migration_id)
);
```

---

## 3. Pre-Migration Phase

### 3.1 User Intent Capture

**Frontend (React/Next.js):**
```typescript
// components/UpgradeButton.tsx
import { useState } from 'react';
import { useStripe } from '@stripe/react-stripe-js';

export function UpgradeButton({ clientId }) {
  const [loading, setLoading] = useState(false);
  const stripe = useStripe();

  const handleUpgrade = async () => {
    setLoading(true);

    try {
      // 1. Create migration intent
      const response = await fetch('/api/migrations', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          client_id: clientId,
          target_tier: 'PREMIUM',
          options: {
            migrate_email: true,
            migrate_dns: true,
            cluster_config: {
              master_count: 1,
              worker_count: 0,
              instance_type: 'cax11'
            }
          }
        })
      });

      const { migration_id, payment_intent } = await response.json();

      // 2. Process payment
      const { error } = await stripe.confirmCardPayment(payment_intent.client_secret);

      if (error) {
        throw new Error(error.message);
      }

      // 3. Confirm payment and start migration
      await fetch(`/api/migrations/${migration_id}/confirm-payment`, {
        method: 'POST'
      });

      // 4. Redirect to migration status page
      window.location.href = `/migrations/${migration_id}`;

    } catch (error) {
      alert('Upgrade failed: ' + error.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <button
      onClick={handleUpgrade}
      disabled={loading}
      className="btn-primary"
    >
      {loading ? 'Processing...' : 'Upgrade to PREMIUM'}
    </button>
  );
}
```

### 3.2 Resource Requirement Calculation

```typescript
// services/migration/calculator.ts
interface ResourceRequirements {
  cluster: {
    master_count: number;
    worker_count: number;
    instance_type: string;
  };
  storage: {
    total_gb: number;
    volumes: number;
  };
  cost: {
    monthly_usd: number;
    setup_fee_usd: number;
  };
}

export async function calculateRequirements(
  clientId: number
): Promise<ResourceRequirements> {
  // Get current usage
  const usage = await db.query(`
    SELECT
      COUNT(DISTINCT vd.id) as domain_count,
      COUNT(vu.id) as mailbox_count,
      COALESCE(SUM(qu.bytes_used), 0) as total_email_gb,
      COUNT(DISTINCT a.id) as app_count
    FROM clients c
    LEFT JOIN virtual_domains vd ON vd.client_id = c.id
    LEFT JOIN virtual_users vu ON vu.domain_id = vd.id
    LEFT JOIN quota_usage qu ON qu.user_id = vu.id
    LEFT JOIN applications a ON a.client_id = c.id
    WHERE c.id = $1
  `, [clientId]);

  const { mailbox_count, total_email_gb, app_count } = usage.rows[0];

  // Calculate cluster size
  let instance_type = 'cax11';  // Default: 2 vCPU, 4GB RAM
  let master_count = 1;
  let worker_count = 0;

  if (mailbox_count > 50 || app_count > 5) {
    instance_type = 'cax21';  // 4 vCPU, 8GB RAM
  }

  if (app_count > 10) {
    master_count = 3;  // HA setup
    worker_count = 2;
  }

  // Calculate storage
  const storage_gb = Math.ceil(total_email_gb / 1024 / 1024 / 1024) + 20;  // Email + 20GB buffer

  // Calculate cost
  const PRICING = {
    cax11: 3.85,
    cax21: 12.39,
    storage_per_gb: 0.0476
  };

  const monthly_cost =
    (master_count + worker_count) * PRICING[instance_type] +
    storage_gb * PRICING.storage_per_gb;

  return {
    cluster: {
      master_count,
      worker_count,
      instance_type
    },
    storage: {
      total_gb: storage_gb,
      volumes: 1
    },
    cost: {
      monthly_usd: monthly_cost,
      setup_fee_usd: 0  // No setup fee
    }
  };
}
```

### 3.3 Hetzner Cloud Quota Check

```typescript
// services/hetzner/quota-checker.ts
import { HetznerAPI } from './api';

export async function checkQuota(
  hcloudToken: string,
  requirements: ResourceRequirements
): Promise<{ ok: boolean; message?: string }> {
  const api = new HetznerAPI(hcloudToken);

  // Check server quota
  const servers = await api.listServers();
  const limit = await api.getServerLimit();

  const required_servers = requirements.cluster.master_count + requirements.cluster.worker_count;

  if (servers.length + required_servers > limit) {
    return {
      ok: false,
      message: `Insufficient server quota. Required: ${required_servers}, Available: ${limit - servers.length}`
    };
  }

  // Check volume quota
  const volumes = await api.listVolumes();
  const volumeLimit = await api.getVolumeLimit();

  if (volumes.length + requirements.storage.volumes > volumeLimit) {
    return {
      ok: false,
      message: 'Insufficient volume quota'
    };
  }

  return { ok: true };
}
```

---

## 4. Infrastructure Provisioning

### 4.1 Terraform Automation

```typescript
// services/terraform/executor.ts
import { exec } from 'child_process';
import { promisify } from 'util';
import * as fs from 'fs/promises';

const execAsync = promisify(exec);

export class TerraformExecutor {
  private workdir: string;
  private migrationId: string;

  constructor(migrationId: string) {
    this.migrationId = migrationId;
    this.workdir = `/var/lib/migrations/${migrationId}/terraform`;
  }

  async provision(config: ClusterConfig): Promise<void> {
    // 1. Create working directory
    await fs.mkdir(this.workdir, { recursive: true });

    // 2. Generate Terraform configuration
    await this.generateConfig(config);

    // 3. Initialize Terraform
    await this.log('Initializing Terraform...');
    await execAsync('terraform init', { cwd: this.workdir });

    // 4. Plan
    await this.log('Creating execution plan...');
    const { stdout: planOutput } = await execAsync('terraform plan -out=tfplan', {
      cwd: this.workdir
    });
    await this.log(planOutput);

    // 5. Apply
    await this.log('Applying infrastructure changes...');
    await execAsync('terraform apply -auto-approve tfplan', {
      cwd: this.workdir
    });

    // 6. Get outputs
    const { stdout: outputsJson } = await execAsync('terraform output -json', {
      cwd: this.workdir
    });
    const outputs = JSON.parse(outputsJson);

    // 7. Save outputs to database
    await db.query(`
      UPDATE migrations
      SET
        cluster_id = $1,
        terraform_state_url = $2,
        options = jsonb_set(options, '{outputs}', $3::jsonb)
      WHERE id = $4
    `, [
      outputs.cluster_id.value,
      `s3://terraform-state/migrations/${this.migrationId}/terraform.tfstate`,
      JSON.stringify(outputs),
      this.migrationId
    ]);
  }

  private async generateConfig(config: ClusterConfig): Promise<void> {
    const tfConfig = `
terraform {
  required_version = ">= 1.5.0"

  backend "s3" {
    bucket = "terraform-state"
    key    = "migrations/${this.migrationId}/terraform.tfstate"
    region = "us-east-1"
  }

  required_providers {
    hcloud = {
      source  = "hetznercloud/hcloud"
      version = "~> 1.45.0"
    }
  }
}

provider "hcloud" {
  token = var.hcloud_token
}

module "k3s_cluster" {
  source = "../../modules/hetzner-k3s-cluster"

  cluster_name         = var.cluster_name
  master_count         = ${config.master_count}
  worker_count         = ${config.worker_count}
  instance_type        = "${config.instance_type}"
  location             = "${config.location}"
  private_network_subnet = "10.0.0.0/16"

  ssh_public_key_path  = var.ssh_public_key_path
  ssh_private_key_path = var.ssh_private_key_path

  # Use pre-baked Packer images for fast deployment
  use_packer_image     = true
  k3s_version          = "v1.30.2+k3s2"
}

output "cluster_id" {
  value = module.k3s_cluster.cluster_id
}

output "kubeconfig" {
  value     = module.k3s_cluster.kubeconfig
  sensitive = true
}

output "api_endpoint" {
  value = module.k3s_cluster.api_endpoint
}
`;

    await fs.writeFile(`${this.workdir}/main.tf`, tfConfig);

    // Generate tfvars
    const tfvars = `
hcloud_token         = "${config.hcloud_token}"
cluster_name         = "client-${config.client_id}-premium"
ssh_public_key_path  = "/var/lib/migrations/${this.migrationId}/id_ed25519.pub"
ssh_private_key_path = "/var/lib/migrations/${this.migrationId}/id_ed25519"
`;

    await fs.writeFile(`${this.workdir}/terraform.tfvars`, tfvars);
  }

  private async log(message: string): Promise<void> {
    await db.query(`
      INSERT INTO migration_logs (migration_id, level, message)
      VALUES ($1, 'INFO', $2)
    `, [this.migrationId, message]);

    // Emit WebSocket event
    io.to(`migration:${this.migrationId}`).emit('log', {
      timestamp: new Date(),
      level: 'INFO',
      message
    });
  }
}
```

### 4.2 Post-Provision Setup

```typescript
// services/cluster/initializer.ts
export async function initializeCluster(migrationId: string): Promise<void> {
  const migration = await getMigration(migrationId);
  const kubeconfig = migration.options.outputs.kubeconfig;

  // 1. Install base services
  await kubectl.apply('kube-system', MANIFESTS.metrics_server);
  await kubectl.apply('kube-system', MANIFESTS.hetzner_csi);
  await kubectl.apply('kube-system', MANIFESTS.hetzner_ccm);

  // 2. Install ingress controller (Traefik)
  await helm.install('traefik', 'traefik/traefik', {
    namespace: 'kube-system',
    values: {
      service: {
        type: 'LoadBalancer'
      }
    }
  });

  // 3. Install cert-manager
  await helm.install('cert-manager', 'jetstack/cert-manager', {
    namespace: 'cert-manager',
    createNamespace: true,
    set: {
      installCRDs: true
    }
  });

  // 4. Wait for services to be ready
  await kubectl.wait('deployment', 'traefik', 'kube-system', { timeout: '5m' });
  await kubectl.wait('deployment', 'cert-manager', 'cert-manager', { timeout: '5m' });

  // 5. Create client namespace
  await kubectl.createNamespace(`client-${migration.client_id}`);

  // 6. Update migration state
  await updateMigrationState(migrationId, 'INFRASTRUCTURE_READY');
}
```

---

## 5. Email Migration Strategy

### 5.1 IMAP Sync with Imapsync

```typescript
// services/email/migrator.ts
import { spawn } from 'child_process';

export class EmailMigrator {
  async migrateMailbox(
    sourceHost: string,
    sourceEmail: string,
    sourcePassword: string,
    targetHost: string,
    targetEmail: string,
    targetPassword: string,
    migrationId: string
  ): Promise<void> {
    return new Promise((resolve, reject) => {
      const args = [
        '--host1', sourceHost,
        '--user1', sourceEmail,
        '--password1', sourcePassword,
        '--host2', targetHost,
        '--user2', targetEmail,
        '--password2', targetPassword,
        '--syncinternaldates',
        '--syncacls',
        '--dry',  // First do dry run
        '--justlogin'  // Just test authentication
      ];

      const proc = spawn('imapsync', args);

      let output = '';

      proc.stdout.on('data', (data) => {
        output += data.toString();
        this.logProgress(migrationId, data.toString());
      });

      proc.stderr.on('data', (data) => {
        output += data.toString();
        this.logError(migrationId, data.toString());
      });

      proc.on('close', (code) => {
        if (code === 0) {
          // Dry run successful, now do real sync
          this.performRealSync(sourceHost, sourceEmail, sourcePassword, targetHost, targetEmail, targetPassword, migrationId)
            .then(resolve)
            .catch(reject);
        } else {
          reject(new Error(`imapsync failed with code ${code}: ${output}`));
        }
      });
    });
  }

  private async performRealSync(
    sourceHost: string,
    sourceEmail: string,
    sourcePassword: string,
    targetHost: string,
    targetEmail: string,
    targetPassword: string,
    migrationId: string
  ): Promise<void> {
    return new Promise((resolve, reject) => {
      const args = [
        '--host1', sourceHost,
        '--user1', sourceEmail,
        '--password1', sourcePassword,
        '--host2', targetHost,
        '--user2', targetEmail,
        '--password2', targetPassword,
        '--syncinternaldates',
        '--syncacls',
        '--exclude', 'Trash',  // Don't migrate trash
        '--skipsize',  // Skip identical messages
        '--buffersize', '8192000',  // 8MB buffer for performance
        '--split1', '500',  // Fetch 500 messages at a time
        '--split2', '500'
      ];

      const proc = spawn('imapsync', args);
      let totalMessages = 0;
      let syncedMessages = 0;

      proc.stdout.on('data', (data) => {
        const output = data.toString();
        this.logProgress(migrationId, output);

        // Parse progress
        const match = output.match(/(\d+)\/(\d+) msgs/);
        if (match) {
          syncedMessages = parseInt(match[1]);
          totalMessages = parseInt(match[2]);
          this.updateProgress(migrationId, syncedMessages, totalMessages);
        }
      });

      proc.on('close', (code) => {
        if (code === 0) {
          resolve();
        } else {
          reject(new Error(`Email migration failed with code ${code}`));
        }
      });
    });
  }

  private async updateProgress(migrationId: string, synced: number, total: number): Promise<void> {
    const percent = Math.floor((synced / total) * 100);

    await db.query(`
      UPDATE migrations
      SET progress_percent = $1
      WHERE id = $2
    `, [percent, migrationId]);

    // WebSocket update
    io.to(`migration:${migrationId}`).emit('progress', {
      synced,
      total,
      percent
    });
  }
}
```

### 5.2 Parallel Migration Strategy

For clients with multiple mailboxes, migrate in parallel:

```typescript
// services/email/batch-migrator.ts
import pLimit from 'p-limit';

export async function migrateAllMailboxes(
  clientId: number,
  migrationId: string
): Promise<void> {
  // Get all mailboxes for client
  const mailboxes = await db.query(`
    SELECT vu.email, vu.password
    FROM virtual_users vu
    JOIN virtual_domains vd ON vu.domain_id = vd.id
    WHERE vd.client_id = $1 AND vu.active = true
  `, [clientId]);

  // Limit concurrency to 5 simultaneous syncs
  const limit = pLimit(5);

  const migrator = new EmailMigrator();

  const promises = mailboxes.rows.map((mailbox) =>
    limit(() =>
      migrator.migrateMailbox(
        'mail.refine.digital',
        mailbox.email,
        mailbox.password,
        `mail-client-${clientId}.premium.refine.digital`,
        mailbox.email,
        mailbox.password,  // Password remains the same
        migrationId
      )
    )
  );

  await Promise.all(promises);

  await log(migrationId, 'All mailboxes migrated successfully');
}
```

### 5.3 Zero-Downtime Email Strategy

```
Phase 1: Preparation (T-24h)
├─ Lower MX record TTL to 300s (5 minutes)
├─ Start continuous IMAP sync (every 5 minutes)
└─ Monitor sync for errors

Phase 2: Cutover (T=0)
├─ Stop accepting new mail on old server (SMTP reject with 451)
├─ Final IMAP sync (catch last messages)
├─ Update MX records to point to new cluster
└─ Wait for DNS propagation (5-10 minutes)

Phase 3: Validation (T+10m)
├─ Send test emails to verify delivery
├─ Check IMAP access on new server
└─ Monitor for any delivery failures

Phase 4: Cleanup (T+7d)
├─ Restore old server MX as backup (priority 50)
├─ Keep old server running for 30 days
└─ Final decommission after 30 days
```

---

## 6. DNS Cutover Process

### 6.1 DNS Record Migration

```typescript
// services/dns/migrator.ts
export async function migrateDNSRecords(
  clientId: number,
  oldNameservers: string[],
  newNameservers: string[],
  migrationId: string
): Promise<void> {
  // 1. Export DNS records from old system
  const domains = await getClientDomains(clientId);

  for (const domain of domains) {
    await log(migrationId, `Migrating DNS for ${domain.name}...`);

    // Export records
    const records = await exportDNSRecords(domain.name, oldNameservers[0]);

    // Import to new system (PowerDNS on k3s)
    await importDNSRecords(domain.name, records, newNameservers[0]);

    // Create checkpoint for rollback
    await createCheckpoint(migrationId, `dns_${domain.name}`, {
      old_records: records,
      old_nameservers: oldNameservers
    });
  }

  await log(migrationId, 'DNS records migrated');
}

async function exportDNSRecords(domain: string, nameserver: string): Promise<DNSRecord[]> {
  // Query old DNS server
  const resolver = new dns.Resolver();
  resolver.setServers([nameserver]);

  const records: DNSRecord[] = [];

  // Export A records
  const aRecords = await resolver.resolve4(domain);
  records.push(...aRecords.map(ip => ({ type: 'A', name: '@', value: ip, ttl: 300 })));

  // Export MX records
  const mxRecords = await resolver.resolveMx(domain);
  records.push(...mxRecords.map(mx => ({ type: 'MX', name: '@', value: mx.exchange, priority: mx.priority, ttl: 300 })));

  // Export TXT records (SPF, DKIM, DMARC)
  const txtRecords = await resolver.resolveTxt(domain);
  records.push(...txtRecords.map(txt => ({ type: 'TXT', name: '@', value: txt.join(''), ttl: 300 })));

  return records;
}

async function importDNSRecords(domain: string, records: DNSRecord[], nameserver: string): Promise<void> {
  // Import to PowerDNS API
  const pdnsAPI = new PowerDNSAPI(nameserver);

  await pdnsAPI.createZone(domain);

  for (const record of records) {
    await pdnsAPI.addRecord(domain, record);
  }
}
```

### 6.2 Gradual Cutover Strategy

```typescript
// services/dns/cutover.ts
export async function performDNSCutover(
  migrationId: string,
  domain: string
): Promise<void> {
  const migration = await getMigration(migrationId);
  const newClusterIP = migration.options.outputs.load_balancer_ip;

  // 1. Update A record with low TTL (for quick rollback)
  await updateDNSRecord(domain, {
    type: 'A',
    name: '@',
    value: newClusterIP,
    ttl: 60  // 1 minute
  });

  await log(migrationId, `Updated A record for ${domain} to ${newClusterIP} (TTL: 60s)`);

  // 2. Wait for propagation
  await sleep(60000);  // 1 minute

  // 3. Verify DNS propagation
  const resolvedIP = await dns.resolve4(domain);
  if (resolvedIP[0] !== newClusterIP) {
    throw new Error(`DNS not propagated correctly. Expected: ${newClusterIP}, Got: ${resolvedIP[0]}`);
  }

  await log(migrationId, 'DNS propagated successfully');

  // 4. Run health checks
  const healthy = await runHealthChecks(domain, newClusterIP);
  if (!healthy) {
    throw new Error('Health checks failed after DNS cutover');
  }

  // 5. Increase TTL after validation (optional)
  await updateDNSRecord(domain, {
    type: 'A',
    name: '@',
    value: newClusterIP,
    ttl: 300  // 5 minutes
  });

  await log(migrationId, `DNS cutover completed for ${domain}`);
}
```

---

## 7. Application Migration

### 7.1 Container Migration

```typescript
// services/apps/migrator.ts
export async function migrateApplications(
  clientId: number,
  migrationId: string
): Promise<void> {
  const apps = await getClientApplications(clientId);

  for (const app of apps) {
    await log(migrationId, `Migrating application: ${app.name}`);

    // 1. Export application configuration
    const config = await exportAppConfig(app.id);

    // 2. Migrate database (if any)
    if (app.database) {
      await migrateDatabase(app.database, migrationId);
    }

    // 3. Migrate object storage (if any)
    if (app.object_storage) {
      await migrateObjectStorage(app.object_storage, migrationId);
    }

    // 4. Deploy to new cluster
    await deployToCluster(app, config, migrationId);

    // 5. Run smoke tests
    await runSmokeTests(app, migrationId);
  }
}

async function migrateDatabase(
  database: DatabaseConfig,
  migrationId: string
): Promise<void> {
  const oldHost = database.host;
  const newHost = `postgres.client-${database.client_id}.svc.cluster.local`;

  // 1. Create backup
  await log(migrationId, `Creating database backup: ${database.name}`);
  const backupFile = `/tmp/migration-${migrationId}-${database.name}.sql`;
  await execAsync(`pg_dump -h ${oldHost} -U ${database.user} ${database.name} > ${backupFile}`);

  // 2. Deploy PostgreSQL on new cluster
  await helm.install('postgresql', 'bitnami/postgresql', {
    namespace: `client-${database.client_id}`,
    values: {
      auth: {
        username: database.user,
        password: database.password,
        database: database.name
      }
    }
  });

  // 3. Wait for PostgreSQL to be ready
  await kubectl.wait('statefulset', 'postgresql', `client-${database.client_id}`, { timeout: '5m' });

  // 4. Restore backup
  await log(migrationId, `Restoring database: ${database.name}`);
  await execAsync(`kubectl exec -n client-${database.client_id} postgresql-0 -- psql -U ${database.user} ${database.name} < ${backupFile}`);

  await log(migrationId, `Database migration completed: ${database.name}`);
}
```

---

## 8. Rollback Strategy

### 8.1 Automatic Rollback Triggers

```typescript
// services/migration/rollback.ts
export class RollbackManager {
  async checkRollbackTriggers(migrationId: string): Promise<boolean> {
    const migration = await getMigration(migrationId);

    // Trigger 1: Health checks failed
    if (migration.state === 'VALIDATING') {
      const healthy = await runHealthChecks(migration);
      if (!healthy) {
        await this.initiateRollback(migrationId, 'Health checks failed');
        return true;
      }
    }

    // Trigger 2: High error rate
    const errorRate = await getErrorRate(migrationId);
    if (errorRate > 0.05) {  // 5% error threshold
      await this.initiateRollback(migrationId, `High error rate: ${errorRate * 100}%`);
      return true;
    }

    // Trigger 3: Manual intervention
    const manualRollback = await checkManualRollbackFlag(migrationId);
    if (manualRollback) {
      await this.initiateRollback(migrationId, 'Manual rollback requested');
      return true;
    }

    return false;
  }

  async initiateRollback(migrationId: string, reason: string): Promise<void> {
    await log(migrationId, `ROLLBACK INITIATED: ${reason}`);

    await updateMigrationState(migrationId, 'ROLLING_BACK', {
      rollback_reason: reason
    });

    // Execute rollback steps in reverse order
    await this.rollbackDNS(migrationId);
    await this.rollbackEmail(migrationId);
    await this.rollbackInfrastructure(migrationId);

    await updateMigrationState(migrationId, 'ROLLED_BACK');

    // Notify user
    await notifyUser(migrationId, 'migration_rollback', {
      reason
    });
  }

  private async rollbackDNS(migrationId: string): Promise<void> {
    const checkpoints = await getCheckpoints(migrationId, 'dns_%');

    for (const checkpoint of checkpoints) {
      const { old_records, old_nameservers } = checkpoint.data;

      // Restore old DNS records
      await importDNSRecords(checkpoint.domain, old_records, old_nameservers[0]);
    }

    await log(migrationId, 'DNS rolled back');
  }

  private async rollbackEmail(migrationId: string): Promise<void> {
    // Revert MX records to old mail server
    const domains = await getClientDomains(migration.client_id);

    for (const domain of domains) {
      await updateDNSRecord(domain.name, {
        type: 'MX',
        name: '@',
        value: 'mail.refine.digital',
        priority: 10,
        ttl: 300
      });
    }

    await log(migrationId, 'Email routing rolled back');
  }

  private async rollbackInfrastructure(migrationId: string): Promise<void> {
    const migration = await getMigration(migrationId);

    if (migration.cluster_id) {
      // Destroy cluster via Terraform
      const tf = new TerraformExecutor(migrationId);
      await tf.destroy();

      await log(migrationId, 'Infrastructure destroyed');
    }
  }
}
```

---

## 9. API Endpoints

### 9.1 Migration API

```typescript
// routes/migrations.ts
import express from 'express';
const router = express.Router();

// Create migration
router.post('/api/migrations', authenticate, async (req, res) => {
  const { client_id, target_tier, options } = req.body;

  // Validate client ownership
  if (req.user.client_id !== client_id) {
    return res.status(403).json({ error: 'Forbidden' });
  }

  // Calculate requirements
  const requirements = await calculateRequirements(client_id);

  // Check Hetzner quota
  const quotaCheck = await checkQuota(req.user.hcloud_token, requirements);
  if (!quotaCheck.ok) {
    return res.status(400).json({ error: quotaCheck.message });
  }

  // Create payment intent
  const paymentIntent = await stripe.paymentIntents.create({
    amount: requirements.cost.monthly_usd * 100,  // Convert to cents
    currency: 'usd',
    customer: req.user.stripe_customer_id,
    metadata: {
      client_id,
      migration_type: 'free_to_premium'
    }
  });

  // Create migration record
  const migration = await db.query(`
    INSERT INTO migrations (client_id, state, options)
    VALUES ($1, 'PENDING_PAYMENT', $2)
    RETURNING *
  `, [client_id, JSON.stringify({ requirements, ...options })]);

  res.json({
    migration_id: migration.rows[0].id,
    payment_intent: {
      client_secret: paymentIntent.client_secret
    },
    requirements
  });
});

// Confirm payment and start migration
router.post('/api/migrations/:id/confirm-payment', authenticate, async (req, res) => {
  const { id } = req.params;

  // Verify payment
  const migration = await getMigration(id);
  const paymentIntent = await stripe.paymentIntents.retrieve(migration.payment_intent_id);

  if (paymentIntent.status !== 'succeeded') {
    return res.status(400).json({ error: 'Payment not confirmed' });
  }

  // Update state and start migration
  await updateMigrationState(id, 'PAYMENT_CONFIRMED');

  // Enqueue migration job
  await migrationQueue.add('start-migration', { migration_id: id });

  res.json({ status: 'Migration started' });
});

// Get migration status
router.get('/api/migrations/:id/status', authenticate, async (req, res) => {
  const migration = await getMigration(req.params.id);

  res.json({
    id: migration.id,
    state: migration.state,
    progress_percent: migration.progress_percent,
    current_step: migration.current_step,
    started_at: migration.started_at,
    completed_at: migration.completed_at
  });
});

// Get migration logs
router.get('/api/migrations/:id/logs', authenticate, async (req, res) => {
  const logs = await db.query(`
    SELECT * FROM migration_logs
    WHERE migration_id = $1
    ORDER BY timestamp DESC
    LIMIT 1000
  `, [req.params.id]);

  res.json(logs.rows);
});

// Initiate rollback
router.post('/api/migrations/:id/rollback', authenticate, async (req, res) => {
  const rollbackManager = new RollbackManager();
  await rollbackManager.initiateRollback(req.params.id, 'Manual rollback requested by user');

  res.json({ status: 'Rollback initiated' });
});

export default router;
```

### 9.2 WebSocket Real-Time Updates

```typescript
// websockets/migration.ts
import { Server } from 'socket.io';

export function setupMigrationWebSocket(io: Server) {
  io.on('connection', (socket) => {
    socket.on('subscribe:migration', async (migrationId) => {
      // Verify user has access to this migration
      const migration = await getMigration(migrationId);
      if (socket.user.client_id !== migration.client_id) {
        socket.emit('error', { message: 'Unauthorized' });
        return;
      }

      // Join migration room
      socket.join(`migration:${migrationId}`);

      // Send current state
      socket.emit('migration:state', {
        id: migration.id,
        state: migration.state,
        progress_percent: migration.progress_percent,
        current_step: migration.current_step
      });
    });

    socket.on('unsubscribe:migration', (migrationId) => {
      socket.leave(`migration:${migrationId}`);
    });
  });
}

// Emit events from migration process
export function emitMigrationEvent(migrationId: string, event: string, data: any) {
  io.to(`migration:${migrationId}`).emit(event, data);
}
```

---

## 10. User Experience Flow

### 10.1 Migration Status Page (React)

```typescript
// pages/migrations/[id].tsx
import { useEffect, useState } from 'react';
import { useRouter } from 'next/router';
import { io } from 'socket.io-client';

export default function MigrationStatusPage() {
  const router = useRouter();
  const { id } = router.query;

  const [migration, setMigration] = useState(null);
  const [logs, setLogs] = useState([]);

  useEffect(() => {
    if (!id) return;

    // Fetch initial state
    fetch(`/api/migrations/${id}/status`)
      .then(res => res.json())
      .then(setMigration);

    // Connect to WebSocket
    const socket = io({
      auth: {
        token: localStorage.getItem('auth_token')
      }
    });

    socket.emit('subscribe:migration', id);

    socket.on('migration:state', (data) => {
      setMigration(data);
    });

    socket.on('migration:log', (log) => {
      setLogs(prev => [...prev, log]);
    });

    socket.on('migration:progress', (progress) => {
      setMigration(prev => ({ ...prev, progress_percent: progress.percent }));
    });

    return () => {
      socket.emit('unsubscribe:migration', id);
      socket.disconnect();
    };
  }, [id]);

  if (!migration) {
    return <div>Loading...</div>;
  }

  return (
    <div className="migration-status">
      <h1>Migration to PREMIUM Tier</h1>

      {/* Progress Bar */}
      <div className="progress-bar">
        <div
          className="progress-fill"
          style={{ width: `${migration.progress_percent}%` }}
        />
      </div>
      <p>{migration.progress_percent}% complete</p>

      {/* Current Step */}
      <div className="current-step">
        <h2>Current Step</h2>
        <p>{getStepDescription(migration.state)}</p>
      </div>

      {/* Timeline */}
      <div className="timeline">
        {getTimelineSteps(migration.state).map((step, i) => (
          <div key={i} className={`timeline-step ${step.status}`}>
            <div className="step-icon">{step.icon}</div>
            <div className="step-content">
              <h3>{step.title}</h3>
              <p>{step.description}</p>
            </div>
          </div>
        ))}
      </div>

      {/* Logs */}
      <div className="logs">
        <h2>Migration Logs</h2>
        <div className="log-container">
          {logs.map((log, i) => (
            <div key={i} className={`log-entry log-${log.level.toLowerCase()}`}>
              <span className="log-timestamp">{new Date(log.timestamp).toLocaleTimeString()}</span>
              <span className="log-message">{log.message}</span>
            </div>
          ))}
        </div>
      </div>

      {/* Actions */}
      {migration.state === 'FAILED' && (
        <div className="actions">
          <button onClick={() => handleRollback(id)}>
            Rollback Migration
          </button>
          <button onClick={() => handleRetry(id)}>
            Retry Migration
          </button>
        </div>
      )}

      {migration.state === 'COMPLETED' && (
        <div className="success">
          <h2>Migration Completed Successfully! 🎉</h2>
          <p>Your PREMIUM cluster is now ready.</p>
          <button onClick={() => router.push('/dashboard')}>
            Go to Dashboard
          </button>
        </div>
      )}
    </div>
  );
}

function getStepDescription(state: string): string {
  const descriptions = {
    PENDING_PAYMENT: 'Waiting for payment confirmation',
    PAYMENT_CONFIRMED: 'Payment received, preparing infrastructure',
    PROVISIONING_INFRASTRUCTURE: 'Creating k3s cluster on Hetzner Cloud',
    INFRASTRUCTURE_READY: 'Cluster created, installing base services',
    PREPARING_MIGRATION: 'Preparing data for migration',
    MIGRATING_EMAIL: 'Migrating email mailboxes',
    MIGRATING_DNS: 'Migrating DNS records',
    MIGRATING_APPS: 'Migrating applications',
    CUTOVER_IN_PROGRESS: 'Updating DNS records to point to new cluster',
    VALIDATING: 'Running post-migration health checks',
    COMPLETED: 'Migration completed successfully',
    FAILED: 'Migration failed',
    ROLLING_BACK: 'Rolling back changes',
    ROLLED_BACK: 'Migration rolled back'
  };

  return descriptions[state] || 'Unknown state';
}
```

### 10.2 Mobile-Friendly Notifications

```typescript
// services/notifications/sender.ts
export async function notifyUser(
  clientId: number,
  event: string,
  data: any
): Promise<void> {
  const user = await getUser(clientId);

  // Email notification
  await sendEmail({
    to: user.email,
    subject: getEmailSubject(event),
    template: getEmailTemplate(event),
    data
  });

  // Push notification (if user has app installed)
  if (user.push_token) {
    await sendPushNotification({
      token: user.push_token,
      title: getPushTitle(event),
      body: getPushBody(event, data)
    });
  }

  // SMS notification (for critical events)
  if (isCriticalEvent(event) && user.phone_number) {
    await sendSMS({
      to: user.phone_number,
      message: getSMSMessage(event, data)
    });
  }
}

function getEmailTemplate(event: string): string {
  const templates = {
    migration_started: 'migration-started',
    migration_completed: 'migration-completed',
    migration_failed: 'migration-failed',
    migration_rollback: 'migration-rollback'
  };

  return templates[event] || 'default';
}
```

---

## 11. Monitoring & Alerting

### 11.1 Migration Metrics

```typescript
// services/monitoring/metrics.ts
import { Gauge, Counter, Histogram } from 'prom-client';

export const migrationMetrics = {
  // Current migrations by state
  migrations_by_state: new Gauge({
    name: 'migrations_by_state',
    help: 'Number of migrations in each state',
    labelNames: ['state']
  }),

  // Migration duration
  migration_duration_seconds: new Histogram({
    name: 'migration_duration_seconds',
    help: 'Duration of migrations in seconds',
    labelNames: ['state', 'success'],
    buckets: [60, 300, 600, 1800, 3600]  // 1m, 5m, 10m, 30m, 1h
  }),

  // Migration failures
  migration_failures_total: new Counter({
    name: 'migration_failures_total',
    help: 'Total number of migration failures',
    labelNames: ['reason']
  }),

  // Rollbacks
  migration_rollbacks_total: new Counter({
    name: 'migration_rollbacks_total',
    help: 'Total number of migration rollbacks',
    labelNames: ['reason']
  })
};

// Update metrics periodically
setInterval(async () => {
  const stats = await db.query(`
    SELECT state, COUNT(*) as count
    FROM migrations
    WHERE completed_at IS NULL
    GROUP BY state
  `);

  stats.rows.forEach(row => {
    migrationMetrics.migrations_by_state.set(
      { state: row.state },
      parseInt(row.count)
    );
  });
}, 30000);  // Every 30 seconds
```

### 11.2 Alerting Rules

```yaml
# alerting/migration-rules.yml
groups:
  - name: migrations
    interval: 30s
    rules:
      - alert: MigrationStuck
        expr: |
          migrations_by_state{state!="COMPLETED",state!="FAILED"}
          and on (migration_id)
          (time() - migration_started_timestamp) > 3600
        for: 5m
        annotations:
          summary: "Migration {{ $labels.migration_id }} stuck in {{ $labels.state }}"
          description: "Migration has been in {{ $labels.state }} for over 1 hour"

      - alert: HighMigrationFailureRate
        expr: |
          rate(migration_failures_total[5m]) > 0.1
        for: 10m
        annotations:
          summary: "High migration failure rate"
          description: "More than 10% of migrations are failing"

      - alert: FrequentRollbacks
        expr: |
          rate(migration_rollbacks_total[1h]) > 0.05
        annotations:
          summary: "Frequent migration rollbacks"
          description: "Rollbacks are happening frequently, investigate root cause"
```

---

## 12. Testing Strategy

### 12.1 Integration Tests

```typescript
// tests/integration/migration.test.ts
import { describe, it, expect, beforeAll, afterAll } from '@jest/globals';

describe('FREE to PREMIUM Migration', () => {
  let testClientId: number;
  let migrationId: string;

  beforeAll(async () => {
    // Create test client with sample data
    testClientId = await createTestClient({
      domains: ['test-migration.com'],
      mailboxes: 10,
      email_size_mb: 100
    });
  });

  afterAll(async () => {
    // Cleanup test resources
    await cleanupTestClient(testClientId);
  });

  it('should successfully migrate simple client', async () => {
    // 1. Initiate migration
    const response = await fetch('/api/migrations', {
      method: 'POST',
      body: JSON.stringify({
        client_id: testClientId,
        target_tier: 'PREMIUM'
      })
    });

    const { migration_id } = await response.json();
    migrationId = migration_id;

    // 2. Confirm payment (test mode)
    await confirmTestPayment(migration_id);

    // 3. Wait for completion (max 20 minutes)
    const finalState = await waitForMigrationCompletion(migration_id, 1200000);

    expect(finalState).toBe('COMPLETED');

    // 4. Verify cluster is accessible
    const cluster = await getCluster(migration_id);
    expect(cluster).toBeDefined();
    expect(cluster.status).toBe('ready');

    // 5. Verify email is accessible
    const emailTest = await testEmailDelivery('test@test-migration.com');
    expect(emailTest.delivered).toBe(true);

    // 6. Verify DNS is correct
    const dnsTest = await verifyDNS('test-migration.com');
    expect(dnsTest.pointing_to_new_cluster).toBe(true);
  }, 1200000);  // 20 minute timeout

  it('should rollback on infrastructure failure', async () => {
    // Simulate infrastructure failure
    mockTerraformFailure();

    const response = await fetch('/api/migrations', {
      method: 'POST',
      body: JSON.stringify({
        client_id: testClientId,
        target_tier: 'PREMIUM'
      })
    });

    const { migration_id } = await response.json();
    await confirmTestPayment(migration_id);

    // Wait for rollback
    const finalState = await waitForMigrationCompletion(migration_id, 600000);

    expect(finalState).toBe('ROLLED_BACK');

    // Verify client is still on FREE tier
    const client = await getClient(testClientId);
    expect(client.tier).toBe('FREE');
  });
});
```

### 12.2 Chaos Engineering Tests

```typescript
// tests/chaos/migration-chaos.ts
import { ChaosMonkey } from './chaos-monkey';

describe('Migration Resilience', () => {
  it('should handle network failures during email migration', async () => {
    const chaos = new ChaosMonkey();

    // Start migration
    const migrationId = await startMigration(testClientId);

    // Inject network failure during email migration
    chaos.on('state:MIGRATING_EMAIL', () => {
      chaos.injectNetworkFailure({ duration: '30s', target: 'imapsync' });
    });

    // Migration should retry and complete
    const finalState = await waitForMigrationCompletion(migrationId, 1800000);
    expect(finalState).toBe('COMPLETED');
  });

  it('should handle API rate limits', async () => {
    const chaos = new ChaosMonkey();

    // Start migration
    const migrationId = await startMigration(testClientId);

    // Inject Hetzner API rate limits
    chaos.on('state:PROVISIONING_INFRASTRUCTURE', () => {
      chaos.injectAPIRateLimit({ duration: '60s', api: 'hetzner' });
    });

    // Migration should handle rate limits gracefully
    const finalState = await waitForMigrationCompletion(migrationId, 1200000);
    expect(finalState).toBe('COMPLETED');
  });
});
```

---

## Conclusion

This FREE to PREMIUM migration automation provides:

✅ **Zero-downtime migration** with gradual DNS cutover
✅ **Automatic rollback** on failure detection
✅ **Real-time progress updates** via WebSocket
✅ **Comprehensive testing** with chaos engineering
✅ **Mobile-friendly UX** with push notifications
✅ **Production-ready state machine** with 12 states
✅ **Email migration** with parallel IMAP sync
✅ **Infrastructure automation** via Terraform
✅ **Monitoring and alerting** with Prometheus

**Typical Migration Timeline:**
- Simple cluster (1 node): 8-12 minutes
- HA cluster (3 nodes): 12-18 minutes
- With email migration: +30-60 minutes
- Total downtime: **0 seconds**

**Next Steps:**
1. Implement state machine in production
2. Set up monitoring dashboard
3. Run canary migrations with test clients
4. Document rollback procedures
5. Train support team on migration process

---

**Document Version:** 1.0
**Last Updated:** 2025-11-17
**Part:** 22 of Multi-Tenant Deployment Series

**Previous:** [21-packer-k3s-image-strategy.md](./21-packer-k3s-image-strategy.md)
**Next:** [23-platform-api-architecture.md](./23-platform-api-architecture.md)
