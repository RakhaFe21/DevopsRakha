# F. Security, Access Control, dan Secret Handling

> Security baseline bukan checklist yang dicentang sekali lalu dilupakan. Ia adalah posture yang perlu di-maintain seiring sistem berubah. Setiap deployment baru, setiap engineer baru, setiap service baru adalah potensi permukaan risiko baru yang harus dinilai secara sadar.

---

## 1. Threat Model: Apa yang Kita Lindungi

Sebelum mendefinisikan kontrol, kita harus jelas tentang apa yang dilindungi dan dari siapa:

| Aset | Ancaman Utama | Dampak Jika Terkompromis |
|---|---|---|
| Database production (user data, transaksi) | SQL injection, credential leak, insider threat | Regulatory penalty, reputational damage, data loss |
| Secret management (API keys, DB passwords) | Hardcoded secrets di code, leaked env vars | Full system compromise via credential reuse |
| Container runtime | Privilege escalation, container escape | Host compromise, lateral movement |
| CI/CD pipeline | Supply chain attack, code injection | Malicious code di-deploy ke production |
| Infrastructure config (Terraform state) | State file leak, unauthorized access | Full infrastructure visibility + modification capability |

Tanpa threat model yang jelas, security menjadi collection of controls yang tidak terhubung — expensive tapi tidak efektif.

---

## 2. Server Hardening

### EC2 / VPS Baseline (jika menggunakan EC2 selain ECS managed)

```bash
#!/bin/bash
# server-hardening.sh
# Dijalankan saat instance provisioning via Terraform user_data atau Ansible

# ── SSH Hardening ──────────────────────────────────────────────────────────
# Tujuan: Hanya izinkan SSH dengan key authentication; matikan password auth;
#         batasi login sebagai root.

cat > /etc/ssh/sshd_config.d/99-hardening.conf << 'EOF'
# Disable root login — operator tidak boleh login langsung sebagai root
PermitRootLogin no

# Disable password authentication — SSH key only
PasswordAuthentication no
ChallengeResponseAuthentication no

# Disable empty passwords
PermitEmptyPasswords no

# Use only SSH protocol 2
Protocol 2

# Restrict cipher suites ke yang masih dianggap secure (2024)
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KexAlgorithms curve25519-sha256,diffie-hellman-group16-sha512

# Disconnect idle sessions setelah 5 menit
ClientAliveInterval 300
ClientAliveCountMax 1

# Log all authentication attempts
LogLevel VERBOSE

# Batasi max auth attempts per connection
MaxAuthTries 3

# Disable X11 forwarding — tidak diperlukan untuk server
X11Forwarding no

# Disable TCP forwarding kecuali eksplisit diperlukan
AllowTcpForwarding no

# Whitelist user yang diizinkan SSH (tidak semua OS user)
AllowUsers deployuser
EOF

systemctl restart sshd

# ── System Updates ─────────────────────────────────────────────────────────
apt-get update && apt-get upgrade -y
apt-get install -y unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades
# unattended-upgrades: otomatis apply security patches.
# Ini adalah salah satu mitigasi paling efektif terhadap known CVEs.

# ── Disable Unnecessary Services ───────────────────────────────────────────
systemctl disable --now avahi-daemon 2>/dev/null || true
systemctl disable --now cups 2>/dev/null || true
systemctl disable --now bluetooth 2>/dev/null || true
# Setiap service yang berjalan adalah potensi attack surface.
# "Disable jika tidak diperlukan" adalah prinsip minimum.

# ── Kernel Hardening via sysctl ────────────────────────────────────────────
cat >> /etc/sysctl.d/99-security.conf << 'EOF'
# Disable IP forwarding (ini bukan router)
net.ipv4.ip_forward = 0

# Protect against SYN flood attacks
net.ipv4.tcp_syncookies = 1

# Ignore ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0

# Enable reverse path filtering (prevent IP spoofing)
net.ipv4.conf.all.rp_filter = 1

# Disable source routing
net.ipv4.conf.all.accept_source_route = 0

# Log suspicious packets (martian packets)
net.ipv4.conf.all.log_martians = 1

# Disable core dumps (mencegah sensitive data di-dump ke disk)
fs.suid_dumpable = 0
EOF

sysctl -p /etc/sysctl.d/99-security.conf

# ── Audit Logging ──────────────────────────────────────────────────────────
apt-get install -y auditd
systemctl enable --now auditd

# Audit rules: log semua privilege escalation attempts
cat > /etc/audit/rules.d/99-security.rules << 'EOF'
-w /etc/passwd -p wa -k identity
-w /etc/sudoers -p wa -k privilege-escalation
-w /etc/ssh/sshd_config -p wa -k ssh-config
-a always,exit -F arch=b64 -S execve -F euid=0 -k root-commands
EOF

auditctl -R /etc/audit/rules.d/99-security.rules
```

---

## 3. IAM: Least Privilege di Semua Level

### Prinsip: IAM adalah Batas Terakhir

Security Group mengontrol network. IAM mengontrol API access. Keduanya harus konsisten — "defense in depth" berarti satu layer yang gagal tidak membuka segalanya.

### AWS IAM Role untuk GitHub Actions (OIDC, bukan long-lived keys)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GithubActionsProductionDeploy",
      "Effect": "Allow",
      "Action": [
        "ecs:RegisterTaskDefinition",
        "ecs:UpdateService",
        "ecs:DescribeServices",
        "ecs:DescribeTaskDefinition",
        "iam:PassRole"
      ],
      "Resource": [
        "arn:aws:ecs:ap-southeast-1:123456789:cluster/myapp-production",
        "arn:aws:ecs:ap-southeast-1:123456789:service/myapp-production/*",
        "arn:aws:ecs:ap-southeast-1:123456789:task-definition/myapp-api-production:*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "ap-southeast-1"
          // Scope ke satu region — role ini tidak bisa digunakan di region lain
        }
      }
    }
  ]
}
```

**Apa yang TIDAK boleh ada di role ini:**
- `ecs:DeleteService`
- `ecs:DeleteCluster`
- `ec2:*`
- `rds:*`
- `iam:CreateRole`
- `secretsmanager:GetSecretValue` (pipeline tidak perlu membaca production secrets)

Role yang digunakan oleh CI/CD pipeline harus sekecil mungkin. Compromise pada pipeline tidak boleh berarti full access ke infrastructure.

### Separation of Duties: Siapa Bisa Apa

```
Role Matrix:

                        Read    Deploy  Rollback  Destroy  IAM Admin
Developer               ✅      ✅      ✅        ❌       ❌
DevOps Lead             ✅      ✅      ✅        ✅*      ❌
Security Engineer       ✅      ❌      ❌        ❌       ✅
CI/CD Pipeline (GitHub) ❌†     ✅      ✅        ❌       ❌
On-call Engineer        ✅      ✅      ✅        ❌       ❌

* Destroy harus melalui change request + second approval
† Pipeline tidak bisa "read" production secrets — hanya bisa reference ARN-nya
```

---

## 4. Secret Management

### Hirarki Secret Management

```
Level 1 — Tidak Boleh Digunakan untuk Sensitive Data:
  ❌ Hardcoded di code
  ❌ .env file yang di-commit ke git

Level 2 — Acceptable untuk Non-Production Config:
  ⚠️  GitHub Actions Secrets (environment-scoped)
      Hanya untuk staging dan nilai non-kritis seperti Slack webhook URL

Level 3 — Wajib untuk Production Credentials:
  ✅ AWS Secrets Manager
     Database passwords, API keys, JWT secrets, payment gateway credentials

Level 4 — Untuk Dynamic/Short-Lived Credentials:
  ✅ AWS IAM Roles (IRSA untuk EKS; Instance Profile untuk EC2/ECS)
     AWS SDK calls; tidak ada hardcoded AWS credentials sama sekali
```

### Secret Rotation Policy

```yaml
# Rotation schedule — semua production secrets
database_password:
  rotation_interval: 30_days
  rotation_method: RDS native rotation via Lambda
  
api_keys_external:
  rotation_interval: 90_days
  rotation_method: Manual + automated verification
  
jwt_signing_key:
  rotation_interval: 180_days
  rotation_method: Dual-key strategy (old + new valid simultaneously during transition)
  # Tanpa dual-key strategy, JWT rotation menyebabkan semua active sessions invalid

ssh_keys:
  rotation_interval: 365_days
  rotation_method: New key added → verified → old key removed
```

### Detecting Secret Leak

```yaml
# .github/workflows/secret-scan.yml — dijalankan pada setiap PR
- name: Scan for hardcoded secrets
  uses: trufflesecurity/trufflehog@main
  with:
    path: ./
    base: ${{ github.event.pull_request.base.sha }}
    head: ${{ github.event.pull_request.head.sha }}
    extra_args: --debug --only-verified
  # TruffleHog dengan --only-verified: hanya alert jika secret bisa
  # diverifikasi sebagai aktif (bukan false positive dari test fixtures)
```

---

## 5. Container Security

### Image Scanning dalam Pipeline

Container image bisa memiliki CVEs baik dari base image (Alpine, Ubuntu) maupun dari application dependencies. Scanning harus terjadi di CI, bukan hanya di registry.

```yaml
# Kebijakan CVE di pipeline:
CRITICAL:
  action: Block deployment
  exception: Harus dengan explicit .trivyignore entry + justifikasi + expiry date

HIGH:
  action: Block deployment jika ada fix tersedia
  action: Warning jika unfixed (tidak ada patch dari vendor)
  
MEDIUM/LOW:
  action: Log ke security dashboard, tidak block deployment
  review: Monthly review untuk tren dan akumulasi
```

### Runtime Security

```yaml
# Falco rules untuk detect anomalous container behavior
# Falco berjalan sebagai DaemonSet di K8s; di ECS menggunakan CloudTrail + GuardDuty

- rule: Container Running as Root
  desc: Alert jika container berjalan sebagai root user
  condition: >
    spawned_process and container and
    user.uid=0 and not user_trusted_containers
  output: >
    Container running as root (user=%user.name container=%container.name
    image=%container.image.repository)
  priority: WARNING

- rule: Unexpected Outbound Connection from Container
  desc: Container melakukan koneksi ke IP eksternal yang tidak dikenal
  condition: >
    outbound and container and
    not fd.sip in (allowed_external_ips)
  priority: WARNING
```

---

## 6. Network Security: Firewall dan Exposure Awareness

```hcl
# Terraform — WAF di depan ALB
resource "aws_wafv2_web_acl" "main" {
  name  = "myapp-production-waf"
  scope = "REGIONAL"

  default_action { allow {} }

  # AWS Managed Rule: Common vulnerabilities (OWASP Top 10)
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1
    override_action { none {} }
    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesCommonRuleSet"
      }
    }
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSCommonRules"
      sampled_requests_enabled   = true
    }
  }

  # Rate limiting: 1000 requests per 5 menit per IP
  rule {
    name     = "RateLimitRule"
    priority = 2
    action { block {} }
    statement {
      rate_based_statement {
        limit              = 1000
        aggregate_key_type = "IP"
      }
    }
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimit"
      sampled_requests_enabled   = true
    }
  }
}
```

---

## 7. Security Audit dan Compliance

```bash
# Weekly automated security check
# Dijalankan sebagai scheduled job di CI

# 1. CIS Benchmark check terhadap AWS account configuration
aws securityhub get-findings \
  --filters '{"SeverityLabel": [{"Value": "CRITICAL", "Comparison": "EQUALS"}]}' \
  --query 'Findings[].{Title:Title,Resource:Resources[0].Id}'

# 2. Drift detection: IAM policies yang berubah tanpa melalui Terraform
aws iam list-policies --scope Local --query 'Policies[].PolicyName' | \
  # Compare dengan policies yang terdaftar di Terraform state

# 3. Exposed public S3 buckets check
aws s3api list-buckets --query 'Buckets[].Name' | \
  xargs -I {} aws s3api get-bucket-public-access-block --bucket {}

# 4. Unused IAM access keys (lebih dari 90 hari tidak digunakan)
aws iam generate-credential-report
aws iam get-credential-report --output text | \
  awk -F, 'NR>1 && $10!="N/A" && $10 < "'$(date -d '90 days ago' +%Y-%m-%d)'" {print $1, $10}'
```

---

*Security baseline yang baik bukan yang paling restriktif — melainkan yang paling konsisten diterapkan dan paling mudah di-maintain. Security control yang terlalu ketat yang tidak bisa dioperasikan oleh tim akan di-bypass, menghasilkan false sense of security yang lebih berbahaya.*
