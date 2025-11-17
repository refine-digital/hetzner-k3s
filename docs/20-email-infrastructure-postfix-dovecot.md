# Email Infrastructure: Postfix + Dovecot for Multi-Tenant Platform
**Part:** 20 of Multi-Tenant Deployment Series
**Last Updated:** 2025-11-17

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Postfix Configuration](#2-postfix-configuration)
3. [Dovecot Configuration](#3-dovecot-configuration)
4. [Database Schema](#4-database-schema)
5. [Multi-Tenancy Patterns](#5-multi-tenancy-patterns)
6. [Security Configuration](#6-security-configuration)
7. [API Integration](#7-api-integration)
8. [Automation & Provisioning](#8-automation--provisioning)
9. [Scaling Considerations](#9-scaling-considerations)
10. [Migration to PREMIUM](#10-migration-to-premium)
11. [Monitoring & Troubleshooting](#11-monitoring--troubleshooting)

---

## 1. Architecture Overview

### 1.1 System Components

```
┌─────────────────────────────────────────────────────────────┐
│                     Internet (SMTP/IMAP)                     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    Load Balancer (HAProxy)                   │
│              SMTP: 25, 587, 465 | IMAP: 993, 143            │
└─────────────────────────────────────────────────────────────┘
                            │
          ┌─────────────────┴─────────────────┐
          ▼                                   ▼
┌──────────────────────┐           ┌──────────────────────┐
│   Postfix Servers    │           │  Dovecot Servers     │
│   (MX, Submission)   │◄─────────►│  (IMAP/POP3/LMTP)   │
│                      │   LMTP    │                      │
│  - Virtual domains   │           │  - Virtual users     │
│  - Anti-spam         │           │  - Maildir storage   │
│  - DKIM signing      │           │  - Quota management  │
│  - TLS termination   │           │  - Sieve filtering   │
└──────────────────────┘           └──────────────────────┘
          │                                   │
          └─────────────────┬─────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              PostgreSQL Database (Virtual Config)            │
│  - virtual_domains                                           │
│  - virtual_users (with password hashes)                      │
│  - virtual_aliases                                           │
│  - quotas                                                    │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Shared Storage (NFS/GlusterFS)              │
│               /var/mail/vmail/{domain}/{user}/               │
│                      Maildir format                          │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **MTA** | Postfix 3.7+ | SMTP server, mail routing |
| **MDA** | Dovecot 2.3+ | IMAP/POP3/LMTP delivery |
| **Database** | PostgreSQL 14+ | Virtual users/domains |
| **Storage** | Maildir on NFS | Email storage |
| **Anti-Spam** | Rspamd | Spam filtering, DKIM |
| **Anti-Virus** | ClamAV | Virus scanning |
| **Webmail** | Roundcube/SnappyMail | Optional web interface |
| **DNS** | PowerDNS | MX/SPF/DKIM records |

### 1.3 Virtual Mailbox Architecture

**Key Concept:** All mailboxes are virtual (database-backed), not system users.

- **Virtual Domains:** Each client can have multiple domains
- **Virtual Users:** email@domain.com → database entry → mailbox path
- **Maildir Format:** One file per message, IMAP-friendly
- **Quota:** Per-user storage limits enforced by Dovecot

---

## 2. Postfix Configuration

### 2.1 Main Configuration (`/etc/postfix/main.cf`)

```ini
# Basic settings
myhostname = mail.refine.digital
myorigin = $myhostname
mydestination = localhost
mynetworks = 127.0.0.0/8, 10.0.0.0/8

# Virtual mailbox settings
virtual_mailbox_domains = pgsql:/etc/postfix/pgsql-virtual-mailbox-domains.cf
virtual_mailbox_maps = pgsql:/etc/postfix/pgsql-virtual-mailbox-maps.cf
virtual_alias_maps = pgsql:/etc/postfix/pgsql-virtual-alias-maps.cf

# Delivery to Dovecot via LMTP
virtual_transport = lmtp:unix:private/dovecot-lmtp

# TLS settings
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.refine.digital/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mail.refine.digital/privkey.pem
smtpd_tls_security_level = may
smtpd_tls_auth_only = yes
smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtpd_tls_ciphers = high
smtpd_tls_exclude_ciphers = aNULL, MD5, DES, 3DES, DES-CBC3-SHA, RC4-SHA, AES256-SHA, AES128-SHA
smtpd_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtpd_tls_mandatory_ciphers = high
smtpd_tls_received_header = yes
smtpd_tls_session_cache_timeout = 3600s
tls_random_source = dev:/dev/urandom

# Client TLS
smtp_tls_security_level = may
smtp_tls_loglevel = 1
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

# SASL authentication (via Dovecot)
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_authenticated_header = yes

# Restrictions
smtpd_relay_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination

smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_non_fqdn_recipient,
    reject_unknown_recipient_domain,
    reject_unauth_destination,
    check_policy_service unix:private/policyd-spf,
    check_policy_service inet:127.0.0.1:11332

smtpd_sender_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_non_fqdn_sender,
    reject_unknown_sender_domain

smtpd_helo_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_non_fqdn_helo_hostname,
    reject_invalid_helo_hostname,
    reject_unknown_helo_hostname

# Rate limiting per domain (via policyd)
smtpd_client_restrictions = check_policy_service unix:private/policyd

# Message size limit (50MB)
message_size_limit = 52428800

# Mailbox size limit (10GB default)
mailbox_size_limit = 0

# Rspamd integration
smtpd_milters = inet:127.0.0.1:11332
non_smtpd_milters = inet:127.0.0.1:11332
milter_default_action = accept
milter_protocol = 6

# Queue settings
maximal_queue_lifetime = 5d
bounce_queue_lifetime = 5d
```

### 2.2 Master Configuration (`/etc/postfix/master.cf`)

```ini
# SMTP (port 25) - receiving mail from internet
smtp      inet  n       -       y       -       -       smtpd

# Submission (port 587) - authenticated users
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_sasl_type=dovecot
  -o smtpd_sasl_path=private/auth
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING

# Submissions (port 465) - authenticated users with implicit TLS
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_sasl_type=dovecot
  -o smtpd_sasl_path=private/auth
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING

# Policyd for rate limiting
policyd  unix  -       n       n       -       -       spawn
  user=policyd argv=/usr/bin/policyd-rate-limit
```

### 2.3 PostgreSQL Query Files

#### Virtual Domains (`/etc/postfix/pgsql-virtual-mailbox-domains.cf`)

```ini
hosts = 127.0.0.1
user = postfix
password = <strong-password>
dbname = mailserver
query = SELECT 1 FROM virtual_domains WHERE name='%s' AND active = true
```

#### Virtual Mailbox Maps (`/etc/postfix/pgsql-virtual-mailbox-maps.cf`)

```ini
hosts = 127.0.0.1
user = postfix
password = <strong-password>
dbname = mailserver
query = SELECT 1 FROM virtual_users WHERE email='%s' AND active = true
```

#### Virtual Alias Maps (`/etc/postfix/pgsql-virtual-alias-maps.cf`)

```ini
hosts = 127.0.0.1
user = postfix
password = <strong-password>
dbname = mailserver
query = SELECT destination FROM virtual_aliases WHERE source='%s' AND active = true
```

---

## 3. Dovecot Configuration

### 3.1 Main Configuration (`/etc/dovecot/dovecot.conf`)

```ini
# Protocols
protocols = imap pop3 lmtp submission

# Listen on all interfaces
listen = *, ::

# Base directory
base_dir = /var/run/dovecot/

# Instance name
instance_name = dovecot
```

### 3.2 Authentication (`/etc/dovecot/conf.d/10-auth.conf`)

```ini
# Disable system users
disable_plaintext_auth = yes
auth_mechanisms = plain login

# SQL authentication
passdb {
  driver = sql
  args = /etc/dovecot/dovecot-sql.conf.ext
}

userdb {
  driver = sql
  args = /etc/dovecot/dovecot-sql.conf.ext
}
```

### 3.3 SQL Configuration (`/etc/dovecot/dovecot-sql.conf.ext`)

```ini
driver = pgsql
connect = host=127.0.0.1 dbname=mailserver user=dovecot password=<strong-password>

# Password query
password_query = \
  SELECT email as user, password, \
  '/var/mail/vmail/%d/%n' as userdb_home, \
  'maildir:/var/mail/vmail/%d/%n' as userdb_mail, \
  5000 AS userdb_uid, 5000 AS userdb_gid \
  FROM virtual_users \
  WHERE email='%u' AND active = true

# User query
user_query = \
  SELECT '/var/mail/vmail/%d/%n' as home, \
  'maildir:/var/mail/vmail/%d/%n' as mail, \
  5000 AS uid, 5000 AS gid, \
  concat('*:storage=', quota) AS quota_rule \
  FROM virtual_users \
  WHERE email='%u' AND active = true

# Iterate query (for doveadm)
iterate_query = SELECT email as username FROM virtual_users WHERE active = true
```

### 3.4 Mail Location (`/etc/dovecot/conf.d/10-mail.conf`)

```ini
# Maildir format
mail_location = maildir:/var/mail/vmail/%d/%n

# Mailbox locations
namespace inbox {
  type = private
  separator = /
  inbox = yes

  mailbox Drafts {
    auto = subscribe
    special_use = \Drafts
  }
  mailbox Sent {
    auto = subscribe
    special_use = \Sent
  }
  mailbox Trash {
    auto = subscribe
    special_use = \Trash
  }
  mailbox Spam {
    auto = subscribe
    special_use = \Junk
  }
}

# Mail user/group
mail_uid = vmail
mail_gid = vmail
mail_privileged_group = vmail

# Permissions
mail_access_groups = vmail
```

### 3.5 LMTP Service (`/etc/dovecot/conf.d/20-lmtp.conf`)

```ini
protocol lmtp {
  # Postfix delivery
  postmaster_address = postmaster@refine.digital

  # Plugins
  mail_plugins = $mail_plugins sieve quota

  # Log all LMTP commands
  mail_debug = yes
}

service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }
}
```

### 3.6 IMAP/POP3 Services (`/etc/dovecot/conf.d/10-master.conf`)

```ini
service imap-login {
  inet_listener imap {
    port = 143
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }

  # Performance tuning
  process_min_avail = 4
  service_count = 1
}

service pop3-login {
  inet_listener pop3 {
    port = 110
  }
  inet_listener pop3s {
    port = 995
    ssl = yes
  }
}

service submission-login {
  inet_listener submission {
    port = 587
  }
}

service auth {
  # Postfix SASL
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }

  # Auth socket for doveadm
  unix_listener auth-userdb {
    mode = 0600
    user = vmail
  }
}
```

### 3.7 Quota Configuration (`/etc/dovecot/conf.d/90-quota.conf`)

```ini
plugin {
  quota = maildir:User quota
  quota_exceeded_message = Storage quota for this account has been exceeded.

  # Quota warnings
  quota_warning = storage=95%% quota-warning 95 %u
  quota_warning2 = storage=80%% quota-warning 80 %u
}

service quota-warning {
  executable = script /usr/local/bin/quota-warning.sh
  user = vmail
  unix_listener quota-warning {
    user = vmail
  }
}

# Quota warning script
#  #!/bin/bash
# PERCENT=$1
# USER=$2
# cat << EOF | /usr/lib/dovecot/dovecot-lda -d $USER -o "plugin/quota=maildir:User quota:noenforcing"
# From: postmaster@refine.digital
# Subject: Quota warning - $PERCENT% full
#
# Your mailbox is now $PERCENT% full.
# EOF
```

### 3.8 Sieve Filtering (`/etc/dovecot/conf.d/90-sieve.conf`)

```ini
plugin {
  sieve = file:~/sieve;active=~/.dovecot.sieve
  sieve_default = /var/mail/vmail/default.sieve
  sieve_default_name = default
  sieve_quota_max_scripts = 10
  sieve_quota_max_storage = 10M
}
```

### 3.9 SSL/TLS Configuration (`/etc/dovecot/conf.d/10-ssl.conf`)

```ini
ssl = required

ssl_cert = </etc/letsencrypt/live/mail.refine.digital/fullchain.pem
ssl_key = </etc/letsencrypt/live/mail.refine.digital/privkey.pem

ssl_min_protocol = TLSv1.2
ssl_cipher_list = ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
ssl_prefer_server_ciphers = yes
ssl_dh = </etc/dovecot/dh.pem
```

---

## 4. Database Schema

### 4.1 PostgreSQL Schema

```sql
-- Create database
CREATE DATABASE mailserver;
\c mailserver;

-- Create tables
CREATE TABLE virtual_domains (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL UNIQUE,
    client_id INTEGER NOT NULL,
    active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    verified_at TIMESTAMP,
    INDEX idx_name (name),
    INDEX idx_client (client_id),
    INDEX idx_active (active)
);

CREATE TABLE virtual_users (
    id SERIAL PRIMARY KEY,
    domain_id INTEGER NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,  -- Argon2 or bcrypt hash
    quota BIGINT DEFAULT 5368709120,  -- 5GB in bytes
    active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login_at TIMESTAMP,
    FOREIGN KEY (domain_id) REFERENCES virtual_domains(id) ON DELETE CASCADE,
    INDEX idx_email (email),
    INDEX idx_domain (domain_id),
    INDEX idx_active (active)
);

CREATE TABLE virtual_aliases (
    id SERIAL PRIMARY KEY,
    domain_id INTEGER NOT NULL,
    source VARCHAR(255) NOT NULL,
    destination TEXT NOT NULL,  -- Can be multiple addresses (comma-separated)
    active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (domain_id) REFERENCES virtual_domains(id) ON DELETE CASCADE,
    INDEX idx_source (source),
    INDEX idx_domain (domain_id)
);

CREATE TABLE quota_usage (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    bytes_used BIGINT DEFAULT 0,
    message_count INTEGER DEFAULT 0,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES virtual_users(id) ON DELETE CASCADE,
    UNIQUE(user_id)
);

-- Function to update quota
CREATE OR REPLACE FUNCTION update_quota_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.last_updated = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER quota_timestamp_trigger
BEFORE UPDATE ON quota_usage
FOR EACH ROW
EXECUTE FUNCTION update_quota_timestamp();

-- Create users for Postfix and Dovecot
CREATE USER postfix WITH PASSWORD '<strong-password>';
GRANT SELECT ON virtual_domains, virtual_users, virtual_aliases TO postfix;

CREATE USER dovecot WITH PASSWORD '<strong-password>';
GRANT SELECT ON virtual_domains, virtual_users, virtual_aliases TO dovecot;
GRANT ALL ON quota_usage TO dovecot;
```

### 4.2 Example Data

```sql
-- Add test domain
INSERT INTO virtual_domains (name, client_id) VALUES ('example.com', 1);

-- Add test user
INSERT INTO virtual_users (domain_id, email, password, quota)
VALUES (1, 'user@example.com', '{ARGON2}$argon2id$v=19$m=65536,t=3,p=4$...', 10737418240);

-- Add alias (forwarding)
INSERT INTO virtual_aliases (domain_id, source, destination)
VALUES (1, 'info@example.com', 'user@example.com');

-- Add catch-all
INSERT INTO virtual_aliases (domain_id, source, destination)
VALUES (1, '@example.com', 'catchall@example.com');
```

---

## 5. Multi-Tenancy Patterns

### 5.1 Isolation Strategy

**Domain-Level Isolation:**
- Each client can have multiple domains
- Domains are isolated in database
- Mailboxes are isolated by domain in filesystem
- No cross-domain aliases allowed (enforced by application)

**Filesystem Layout:**
```
/var/mail/vmail/
├── example.com/
│   ├── user1/
│   │   ├── cur/
│   │   ├── new/
│   │   └── tmp/
│   └── user2/
├── anotherdomain.com/
│   └── admin/
```

### 5.2 Resource Limits

**Per-Client Quotas:**
```sql
-- Client-level quota tracking
CREATE TABLE client_quotas (
    client_id INTEGER PRIMARY KEY,
    total_quota BIGINT NOT NULL,  -- Total bytes allowed
    used_quota BIGINT DEFAULT 0,
    max_domains INTEGER DEFAULT 10,
    max_mailboxes_per_domain INTEGER DEFAULT 100,
    max_aliases_per_domain INTEGER DEFAULT 500,
    max_daily_emails INTEGER DEFAULT 10000
);

-- View for checking client quota
CREATE VIEW client_quota_usage AS
SELECT
    c.client_id,
    c.total_quota,
    COALESCE(SUM(q.bytes_used), 0) as used_quota,
    COUNT(DISTINCT vd.id) as domain_count,
    COUNT(vu.id) as mailbox_count
FROM client_quotas c
LEFT JOIN virtual_domains vd ON vd.client_id = c.client_id
LEFT JOIN virtual_users vu ON vu.domain_id = vd.id
LEFT JOIN quota_usage q ON q.user_id = vu.id
GROUP BY c.client_id, c.total_quota;
```

### 5.3 Rate Limiting

**Postfix Policyd Configuration:**
```ini
# /etc/postfix-policyd/policyd.conf
[quotas]
# Per-sender limits (per day)
user_quota = 500
domain_quota = 5000

# Per-recipient limits
recipient_quota = 1000

# Connection limits
conn_rate = 10/60s   # 10 connections per minute
```

---

## 6. Security Configuration

### 6.1 SPF, DKIM, DMARC Setup

**SPF Record (DNS):**
```
example.com. IN TXT "v=spf1 mx include:_spf.refine.digital ~all"
```

**DKIM with Rspamd:**
```bash
# Generate DKIM key
mkdir -p /var/lib/rspamd/dkim
rspamadm dkim_keygen -b 2048 -s mail -d example.com -k /var/lib/rspamd/dkim/example.com.mail.key > example.com.mail.txt

# DNS record
mail._domainkey.example.com. IN TXT "v=DKIM1; k=rsa; p=MIIBIjANBgkqhki..."
```

**DMARC Record:**
```
_dmarc.example.com. IN TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@refine.digital"
```

### 6.2 Anti-Spam with Rspamd

**Rspamd Configuration (`/etc/rspamd/local.d/worker-proxy.inc`):**
```
bind_socket = "127.0.0.1:11332";
milter = yes;
timeout = 120s;
upstream "local" {
  default = yes;
  self_scan = yes;
}
```

### 6.3 Fail2ban Protection

```ini
# /etc/fail2ban/jail.local
[postfix-auth]
enabled = true
port = smtp,submission,submissions
logpath = /var/log/mail.log

[dovecot]
enabled = true
port = imap,imaps,pop3,pop3s
logpath = /var/log/mail.log
```

---

## 7. API Integration

### 7.1 REST API Endpoints

```javascript
// Domain Management
POST   /api/v1/domains                     // Add domain
GET    /api/v1/domains/:id                 // Get domain
PUT    /api/v1/domains/:id                 // Update domain
DELETE /api/v1/domains/:id                 // Delete domain
POST   /api/v1/domains/:id/verify          // Verify domain ownership
GET    /api/v1/domains/:id/dns-records     // Get required DNS records

// Email Account Management
POST   /api/v1/mailboxes                   // Create mailbox
GET    /api/v1/mailboxes/:id               // Get mailbox
PUT    /api/v1/mailboxes/:id               // Update mailbox
DELETE /api/v1/mailboxes/:id               // Delete mailbox
PUT    /api/v1/mailboxes/:id/password      // Change password
GET    /api/v1/mailboxes/:id/usage         // Get quota usage

// Alias Management
POST   /api/v1/aliases                     // Create alias
GET    /api/v1/aliases/:id                 // Get alias
PUT    /api/v1/aliases/:id                 // Update alias
DELETE /api/v1/aliases/:id                 // Delete alias
```

### 7.2 Example API Implementation (Node.js + Express)

```javascript
const express = require('express');
const { Pool } = require('pg');
const bcrypt = require('bcrypt');
const argon2 = require('argon2');

const pool = new Pool({
  host: 'localhost',
  database: 'mailserver',
  user: 'apiuser',
  password: process.env.DB_PASSWORD
});

// Create mailbox
app.post('/api/v1/mailboxes', async (req, res) => {
  const { domain_id, email, password, quota } = req.body;

  try {
    // Hash password with Argon2
    const passwordHash = await argon2.hash(password, {
      type: argon2.argon2id,
      memoryCost: 65536,
      timeCost: 3,
      parallelism: 4
    });

    const result = await pool.query(
      'INSERT INTO virtual_users (domain_id, email, password, quota) VALUES ($1, $2, $3, $4) RETURNING *',
      [domain_id, email, passwordHash, quota || 5368709120]
    );

    // Create mailbox directory
    const [localPart, domain] = email.split('@');
    await execAsync(`sudo -u vmail mkdir -p /var/mail/vmail/${domain}/${localPart}/{cur,new,tmp}`);

    res.json(result.rows[0]);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get quota usage
app.get('/api/v1/mailboxes/:id/usage', async (req, res) => {
  try {
    const result = await pool.query(`
      SELECT
        vu.email,
        vu.quota,
        COALESCE(qu.bytes_used, 0) as bytes_used,
        COALESCE(qu.message_count, 0) as message_count,
        ROUND((COALESCE(qu.bytes_used, 0)::numeric / vu.quota::numeric) * 100, 2) as usage_percent
      FROM virtual_users vu
      LEFT JOIN quota_usage qu ON qu.user_id = vu.id
      WHERE vu.id = $1
    `, [req.params.id]);

    res.json(result.rows[0]);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

---

## 8. Automation & Provisioning

### 8.1 New Client Onboarding Flow

```
1. Client Signs Up
   ↓
2. API Call: POST /api/clients
   - Create client_id in database
   - Initialize client_quotas
   ↓
3. Client Adds Domain
   - POST /api/domains
   - Generate DNS records (MX, SPF, DKIM, DMARC)
   - Create DKIM keys
   ↓
4. Domain Verification
   - POST /api/domains/:id/verify
   - Check DNS records via API
   - Mark domain as verified
   ↓
5. Create First Mailbox
   - POST /api/mailboxes
   - Create filesystem structure
   - Send welcome email
```

### 8.2 DNS Record Automation

```javascript
// Generate required DNS records
async function generateDNSRecords(domain) {
  // Generate DKIM key if not exists
  const dkimKey = await generateDKIMKey(domain);

  return {
    mx: [
      { priority: 10, value: 'mail.refine.digital.' },
      { priority: 20, value: 'mail2.refine.digital.' }
    ],
    spf: {
      type: 'TXT',
      name: '@',
      value: 'v=spf1 mx include:_spf.refine.digital ~all'
    },
    dkim: {
      type: 'TXT',
      name: 'mail._domainkey',
      value: `v=DKIM1; k=rsa; p=${dkimKey.publicKey}`
    },
    dmarc: {
      type: 'TXT',
      name: '_dmarc',
      value: 'v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@refine.digital'
    }
  };
}
```

### 8.3 Certificate Management

```bash
#!/bin/bash
# certbot-renew.sh

# Renew certificates
certbot renew --deploy-hook /usr/local/bin/reload-mail-services.sh

# reload-mail-services.sh
#!/bin/bash
systemctl reload postfix
systemctl reload dovecot
```

---

## 9. Scaling Considerations

### 9.1 Horizontal Scaling

**Multiple MX Servers:**
```
DNS:
example.com. MX 10 mail1.refine.digital.
example.com. MX 20 mail2.refine.digital.
example.com. MX 30 mail3.refine.digital.
```

**Shared Storage:**
- NFS or GlusterFS for `/var/mail/vmail`
- Database replication (PostgreSQL streaming replication)

**Load Balancing:**
```
HAProxy Configuration:
frontend smtp_frontend
    bind *:25
    default_backend smtp_backend

backend smtp_backend
    balance roundrobin
    server mail1 10.0.1.10:25 check
    server mail2 10.0.1.11:25 check
```

### 9.2 Database Replication

```bash
# Primary server
wal_level = replica
max_wal_senders = 3
wal_keep_size = 1GB

# Replica server
hot_standby = on
```

---

## 10. Migration to PREMIUM

### 10.1 Email Migration Strategy

When a client upgrades to PREMIUM tier:

**Option 1: Keep Email on Shared Infrastructure**
- Simplest: Email stays on FREE tier servers
- Client gets k3s cluster for applications only
- No email migration needed

**Option 2: Migrate Email to Client's Cluster**
- Deploy Postfix+Dovecot on client's k3s cluster
- Use `imapsync` to migrate mailboxes:

```bash
#!/bin/bash
# migrate-mailbox.sh
imapsync \
  --host1 mail.refine.digital \
  --user1 user@example.com \
  --password1 "$OLD_PASSWORD" \
  --host2 mail.example.com \
  --user2 user@example.com \
  --password2 "$NEW_PASSWORD" \
  --delete2 \
  --expunge2
```

**Option 3: Hybrid Approach**
- Gradual migration over 30 days
- Lower MX record TTL to 300s (5 minutes)
- Update MX records to point to client's cluster
- Keep old server as backup MX (priority 50)

---

## 11. Monitoring & Troubleshooting

### 11.1 Monitoring Metrics

**Key Metrics:**
- Queue size: `postqueue -p | tail -1`
- Connection rate: `pflogsumm /var/log/mail.log`
- Disk usage per domain: `du -sh /var/mail/vmail/*`
- Database connections: `pg_stat_activity`

**Prometheus Exporter:**
```yaml
# docker-compose.yml
version: '3'
services:
  postfix-exporter:
    image: kumina/postfix-exporter
    ports:
      - "9154:9154"
    volumes:
      - /var/log:/var/log:ro
```

### 11.2 Common Issues

**Issue: Mail delivery slow**
```bash
# Check queue
postqueue -p

# Check Postfix logs
tail -f /var/log/mail.log | grep "status=deferred"

# Check Dovecot LMTP
doveadm log find
```

**Issue: Authentication failures**
```bash
# Test SASL
testsaslauthd -u user@example.com -p password -s smtp

# Check Dovecot auth
doveadm auth test user@example.com password
```

**Issue: High CPU usage**
```bash
# Check Rspamd
rspamadm configtest
rspamadm control stat

# Optimize Postfix
postconf -e "default_process_limit = 50"
postconf -e "smtpd_client_connection_count_limit = 10"
```

### 11.3 Log Analysis

```bash
# Install pflogsumm
apt-get install pflogsumm

# Generate daily report
pflogsumm -d yesterday /var/log/mail.log

# Key metrics to watch:
# - Recipients by message size
# - Senders by message count
# - Rejection reasons
```

---

## Conclusion

This Postfix + Dovecot architecture provides:

✅ **Scalable multi-tenant email** for 100s-1000s of clients
✅ **Database-backed virtual domains/users** for easy management
✅ **API-driven provisioning** for automation
✅ **Production-ready security** (SPF, DKIM, DMARC, TLS)
✅ **Flexible migration** to PREMIUM tier
✅ **Comprehensive monitoring** and troubleshooting

**Next Steps:**
1. Review [22-free-to-premium-migration-automation.md](./22-free-to-premium-migration-automation.md) for migration details
2. Review [23-platform-api-architecture.md](./23-platform-api-architecture.md) for complete API design
3. Implement monitoring dashboard
4. Set up automated backups

---

**Document Version:** 1.0
**Last Updated:** 2025-11-17
**Part:** 20 of Multi-Tenant Deployment Series

**Next:** [21-packer-k3s-image-strategy.md](./21-packer-k3s-image-strategy.md) ✅ (Already Created)
