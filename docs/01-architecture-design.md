# B. Desain Arsitektur Infrastruktur

> **Dokumen ini** menjelaskan cara berpikir saya dalam menyusun fondasi environment yang layak dipertahankan saat sistem tumbuh — bukan sekadar konfigurasi yang "bisa jalan".

---

## 1. Asumsi Desain dan Konteks

Skenario yang digunakan sebagai basis dokumen ini:

- Aplikasi web multi-tier (API backend + frontend SPA + background worker)
- Traffic: ~50.000 DAU dengan peak 8x baseline saat event promosi
- Database: PostgreSQL primary dengan kebutuhan PITR (Point-in-Time Recovery)
- Deployment target: AWS (primary cloud), dengan portability awareness
- Tim operasional: 2–4 engineer; tidak ada dedicated NOC
- Constraint eksplisit: **tidak ada scheduled downtime** untuk release

Asumsi-asumsi ini menentukan setiap keputusan desain yang mengikutinya. Infrastruktur tanpa konteks adalah config tanpa makna.

---

## 2. Environment Separation

### Prinsip Dasar

Tiga environment wajib ada dengan **isolasi yang nyata** — bukan sekadar perbedaan nama atau prefix config:

```
Development  →  Staging  →  Production
    ↑               ↑              ↑
 Per-engineer   Shared team   Live traffic
 ephemeral      persistent     persistent
 low-fidelity   high-fidelity  source of truth
```

**Kesalahan umum yang saya hindari:**
- Staging menggunakan production database dengan data yang di-mask. Ini bukan staging, ini adalah production dengan false sense of safety.
- Development langsung connect ke staging services. Ini menciptakan dependency noise yang menyebabkan flaky tests dan false incidents.
- Production dan staging di-deploy ke AWS account yang sama. Blast radius dari misconfiguration IAM atau accidental `terraform destroy` menjadi tidak terbatas.

### Account Strategy

```
AWS Organization
├── Management Account (billing, SCPs only)
├── Development Account
│   └── Per-feature ephemeral environments (via Terraform workspaces)
├── Staging Account
│   └── Persistent staging environment (mirrors production topology)
└── Production Account
    └── Production environment (hardened, restricted access)
```

Pemisahan per-AWS-account — bukan per-VPC dalam account yang sama — memberikan isolasi IAM yang sesungguhnya. SCPs (Service Control Policies) di organization level mencegah, misalnya, staging account membuat resources di region yang tidak diizinkan, atau mengubah billing configuration.

---

## 3. Network Architecture

### VPC Design per Environment

```
Production VPC (10.0.0.0/16)
├── Public Subnets (10.0.0.0/24, 10.0.1.0/24) — AZ-a, AZ-b
│   └── Resources: ALB, NAT Gateway, Bastion Host
├── Private App Subnets (10.0.10.0/24, 10.0.11.0/24) — AZ-a, AZ-b
│   └── Resources: ECS Tasks / EC2 App Servers
├── Private Data Subnets (10.0.20.0/24, 10.0.21.0/24) — AZ-a, AZ-b
│   └── Resources: RDS, ElastiCache, MSK
└── Reserved Subnets (10.0.30.0/24, 10.0.31.0/24)
    └── Future use: internal tooling, monitoring agents
```

**Alasan pemisahan tiga tier subnet:**

| Layer | Justifikasi |
|---|---|
| Public | Hanya load balancer dan NAT yang perlu public IP. Tidak ada application server yang boleh memiliki public IP langsung. |
| Private App | Application tier tidak perlu inbound dari internet; hanya dari ALB. |
| Private Data | Database layer tidak boleh reachable dari manapun kecuali application subnet. Security Group sebagai second layer of defense. |

### Security Group Matrix

```
ALB Security Group
  Inbound:  443/tcp from 0.0.0.0/0
            80/tcp from 0.0.0.0/0 (redirect to HTTPS only)
  Outbound: 8080/tcp to App-SG

App Security Group
  Inbound:  8080/tcp from ALB-SG
            22/tcp from Bastion-SG (SSH; ideally replaced with SSM)
  Outbound: 5432/tcp to DB-SG
            6379/tcp to Cache-SG
            443/tcp to 0.0.0.0/0 (untuk AWS API calls via NAT)

DB Security Group
  Inbound:  5432/tcp from App-SG ONLY
  Outbound: (none required)

Bastion Security Group
  Inbound:  22/tcp from Corporate IP range ONLY (bukan 0.0.0.0/0)
  Outbound: 22/tcp to App-SG
```

**Catatan penting:** Bastion host adalah pilihan pragmatis. Preferensi saya adalah **AWS Systems Manager Session Manager** sebagai pengganti bastion — tidak ada port 22 yang perlu dibuka sama sekali, seluruh session tercatat di CloudTrail, dan tidak ada SSH key yang perlu dirotasi secara manual.

### NAT Gateway Strategy

Satu NAT Gateway per AZ untuk high availability. Ini lebih mahal dari single NAT Gateway, tetapi single NAT Gateway adalah single point of failure untuk semua outbound traffic dari private subnets — failure-nya tidak terdeteksi oleh most ALB health checks dan menyebabkan silent degradation yang sulit didiagnosa.

---

## 4. Multi-AZ dan High Availability Design

```
                        Route 53 (Health Check + Failover)
                               │
                        ALB (Multi-AZ)
                       /              \
              AZ-a                    AZ-b
              ────                    ────
         App Servers              App Servers
         (ASG min:2)              (ASG min:2)
              │                        │
         RDS Primary              RDS Standby
         (AZ-a)          ←sync→   (AZ-b, Multi-AZ)
              │
         ElastiCache
         (Cluster Mode)
```

**RDS Multi-AZ** bukan read replica — ini adalah synchronous standby untuk automatic failover. Untuk read-heavy workload, read replicas ditambahkan secara terpisah di private data subnet.

---

## 5. Provisioning Approach: Terraform

### State Management

```hcl
# backend.tf — setiap environment memiliki state bucket terpisah
terraform {
  backend "s3" {
    bucket         = "infra-tfstate-production"   # per-account bucket
    key            = "core/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"       # prevent concurrent apply
    kms_key_id     = "arn:aws:kms:..."            # CMK, bukan default AWS key
  }
}
```

**Kenapa state harus diencrypt dengan CMK?**  
Terraform state mengandung semua resource attributes termasuk initial passwords, connection strings, dan sensitive outputs. Default S3 encryption menggunakan AWS-managed key yang tidak memberikan granular access control. CMK memungkinkan kita me-revoke access ke state file tanpa menghapus bucket.

### Module Structure

```
terraform/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── production/
├── modules/
│   ├── vpc/
│   ├── ecs-cluster/
│   ├── rds/
│   ├── alb/
│   └── monitoring/
└── global/
    ├── iam/
    └── route53/
```

**Prinsip module design:**
- Module harus memiliki interface yang stabil (input/output) dan implementasi yang bisa berubah tanpa breaking consumers.
- Tidak ada resource yang di-create langsung di `environments/` — semua melalui module untuk konsistensi.
- Setiap module memiliki `outputs.tf` yang eksplisit; tidak ada penggunaan data source cross-module yang implisit.

### Terraform Workflow yang Disiplin

```bash
# Development cycle yang benar
terraform fmt -recursive       # enforced via pre-commit hook
terraform validate
tflint --recursive             # catches logical errors beyond syntax
terraform plan -out=tfplan     # output disimpan, bukan di-pipe langsung
# Code review terhadap plan output
terraform apply tfplan         # apply dari saved plan, bukan interactive
```

**Kenapa `terraform plan -out=tfplan` dan bukan langsung `apply`?**  
`terraform apply` tanpa saved plan bisa menghasilkan apply yang berbeda dari plan yang di-review jika ada state drift antara waktu plan dan apply. Saved plan adalah kontrak yang memastikan apa yang di-apply adalah tepat apa yang di-review.

---

## 6. Service Dependencies dan Boundary Operasional

```yaml
# Service dependency map (simplified)
api-service:
  depends_on:
    - postgres (hard dependency — health check must pass)
    - redis (soft dependency — degrades gracefully to no-cache)
    - s3 (soft dependency — file upload degrades, core flow unaffected)
  
worker-service:
  depends_on:
    - postgres (hard)
    - redis/queue (hard — worker has no function without queue)
    - api-service (none — worker consumes queue directly)

frontend-service:
  depends_on:
    - api-service (soft — can serve cached static content)
    - cdn (soft — falls back to direct S3 serving)
```

Mendefinisikan dependency sebagai hard vs. soft menentukan:
1. **Health check behavior**: hard dependency failure = container unhealthy; soft dependency failure = container degraded tapi healthy.
2. **Rollback trigger**: hard dependency failure di production = automatic rollback; soft = alert + manual decision.
3. **Deployment order**: hard dependencies harus healthy sebelum dependent service di-deploy.

---

## 7. Resource Planning dan Cost Awareness

### Baseline Sizing (production)

| Component | Spec | Justifikasi |
|---|---|---|
| ECS Task (API) | 0.5 vCPU / 1GB RAM (min), 2 vCPU / 4GB (max) | Laravel dengan OPcache; memory-bound bukan CPU-bound |
| ECS Task (Worker) | 0.25 vCPU / 512MB RAM | Queue processing; bursty tapi tidak memory-intensive |
| RDS PostgreSQL | db.t4g.medium (2 vCPU, 4GB) starter, db.r6g.large jika WAL pressure terlihat | Graviton instance untuk cost efficiency ~20% vs Intel |
| ElastiCache | cache.t4g.small (cluster mode disabled di awal) | Session + query cache; scale up sebelum cluster mode kecuali genuinely diperlukan |
| ALB | Managed; cost berdasarkan LCU | Tidak ada overhead operasional |

### Cost Guard Rails

- **AWS Budgets** dengan alert di 80% dan 100% monthly estimate
- **Savings Plans** untuk baseline compute setelah 3 bulan usage pattern established (jangan commit Savings Plans di bulan pertama)
- **S3 Intelligent-Tiering** untuk storage yang access pattern-nya tidak dapat diprediksi
- **RDS Storage Autoscaling** enabled, tapi dengan maximum threshold — unbounded autoscaling dapat menyebabkan cost spike yang tidak disadari

---

*Setiap keputusan di atas dapat dipertanyakan dan direvisi berdasarkan constraint spesifik yang belum disebutkan di sini. Arsitektur yang baik adalah arsitektur yang jelas tentang asumsi-asumsinya.*
