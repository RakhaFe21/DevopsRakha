
# G. Backup, Restore, dan Recovery Readiness

> Backup yang belum pernah diuji adalah backup yang mungkin tidak berfungsi. Disaster recovery yang hanya ada di dokumen tapi belum pernah dipraktikkan adalah false confidence yang paling berbahaya. Kesiapan recovery diukur dari kemampuan menjawab satu pertanyaan: **"Jika server production hilang sekarang, berapa menit yang dibutuhkan untuk kembali normal?"**

---

## 1. RPO dan RTO: Mendefinisikan Target Sebelum Mendefinisikan Solusi

**RPO (Recovery Point Objective)** — Berapa banyak data yang dapat hilang?  
**RTO (Recovery Time Objective)** — Berapa lama downtime yang dapat ditoleransi?

```
Skenario              RPO Target    RTO Target    Justifikasi
─────────────────────────────────────────────────────────────────────
Database corruption   1 menit       30 menit      Transaksi keuangan tidak boleh hilang
Database server down  5 menit       15 menit      Multi-AZ failover otomatis
Application server    0 (stateless) 5 menit       Container re-spawn dari image
Config/secret loss    0 (IaC)       10 menit      Terraform re-apply
Complete region down  1 jam         4 jam         DR ke secondary region
```
![enter image description here](https://res.cloudinary.com/djyvswx7e/image/upload/v1774321122/Screenshot_2026-03-24_095622_ga5i3x.png)

**Catatan penting:** RPO dan RTO harus disetujui oleh stakeholder bisnis, bukan hanya tim teknis. Mencapai RPO 1 detik dan RTO 0 adalah secara teknis mungkin tapi sangat mahal. Angka di atas adalah hasil trade-off antara cost dan business requirement.

---

## 2. Database Backup Strategy

### PostgreSQL: Multi-Layer Backup

![enter image description here](https://res.cloudinary.com/djyvswx7e/image/upload/v1774321122/Screenshot_2026-03-24_095707_x8jay7.png)

**Konfigurasi RDS Backup:**

```hcl
# Terraform
resource "aws_db_instance" "production" {
  # ...
  
  backup_retention_period = 30
  backup_window           = "18:00-19:00"  # UTC = 01:00-02:00 WIB
  # Backup window dipilih di luar peak hours (midnight lokal)
  
  maintenance_window = "Mon:19:00-Mon:20:00"  # UTC = Selasa 02:00-03:00 WIB
  # Maintenance window harus berbeda dari backup window
  
  # Multi-AZ: automated failover jika primary AZ down
  multi_az = true
  
  # Deletion protection: wajib di production
  # Mencegah `terraform destroy` atau manual deletion secara tidak sengaja
  deletion_protection = true
  
  # Enable Performance Insights untuk query-level visibility
  performance_insights_enabled          = true
  performance_insights_retention_period = 7  # hari; 7 gratis, 731 berbayar
  
  # Enhanced Monitoring untuk OS-level metrics
  monitoring_interval = 30  # detik
  
  # Enable automated minor version upgrade tapi NOT major version
  auto_minor_version_upgrade = true
  allow_major_version_upgrade = false
}
```

### Point-in-Time Recovery: Cara Penggunaan

```bash
# Scenario: Data corruption ditemukan pada pukul 15:45 WIB
# Root cause: buggy migration yang di-deploy pukul 15:30

# Step 1: Identifikasi timestamp sebelum corruption
RESTORE_TIME="2024-06-15T08:29:00Z"  # 15:29 WIB = sebelum deployment

# Step 2: Restore ke new RDS instance (TIDAK ke existing — avoid data loss race)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier myapp-production \
  --target-db-instance-identifier myapp-production-restored-20240615 \
  --restore-time "$RESTORE_TIME" \
  --db-instance-class db.r6g.large \
  --multi-az \
  --no-publicly-accessible

# Step 3: Tunggu instance available (typically 15-30 menit untuk database besar)
aws rds wait db-instance-available \
  --db-instance-identifier myapp-production-restored-20240615

# Step 4: Verifikasi data di restored instance sebelum pointing traffic
# Hubungkan ke instance baru dan verifikasi record terakhir sebelum corruption

# Step 5: Update application config untuk point ke restored instance
# (Via Secrets Manager atau parameter store — tidak perlu reconfig manual di setiap server)

# Step 6: Setelah verified, terminasi original corrupted instance
# JANGAN terminasi sebelum verified — kamu masih mungkin perlu data dari sana
```

---

## 3. Application Config Backup

Semua application config harus di-version control dan dapat di-restore dari infrastructure code:

```
Config Type             Storage         Recovery Method
────────────────────────────────────────────────────────────────
Infrastructure          Git + Terraform  terraform apply
                        state di S3
Application config      Git + Secrets    Secrets Manager restore
                        Manager
Nginx/proxy config      Git              Docker image rebuild
SSL certificates        ACM (managed)    Automatic renewal
SSH authorized keys     Terraform        terraform apply
IAM policies            Terraform        terraform apply
```

**Prinsip:** Tidak boleh ada konfigurasi production yang hanya ada di kepala engineer atau di server tanpa backup. Jika sesuatu tidak ada di Git atau di AWS managed service, itu adalah configuration drift yang perlu segera di-codify.

### Terraform State Backup

```hcl
# S3 backend dengan versioning — terraform state adalah backup dari dirinya sendiri
resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = "infra-tfstate-production"
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Lifecycle: state version lebih dari 90 hari dipindah ke Glacier
resource "aws_s3_bucket_lifecycle_configuration" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  
  rule {
    id     = "tfstate-retention"
    status = "Enabled"
    
    noncurrent_version_transition {
      noncurrent_days = 90
      storage_class   = "GLACIER"
    }
    
    noncurrent_version_expiration {
      noncurrent_days = 365
    }
  }
}
```

---

## 4. Disaster Recovery Runbook

### Skenario 1: Single Container Failure

```
Deteksi: ECS health check gagal
Respons: ECS otomatis spawn replacement task (self-healing)
RTO: < 2 menit
Action dari tim: Monitor bahwa replacement task sehat; investigasi root cause
```

### Skenario 2: EC2 Host / AZ Failure

```
Deteksi: CloudWatch alarm + ECS service events
Respons: ECS reschedule tasks ke healthy host di AZ lain
RTO: < 5 menit
Action dari tim: Verify traffic terdistribusi ke AZ yang tersisa;
                 Jika ASG tidak scale, trigger manual scale-out
```

### Skenario 3: Database Primary Failure

```
Deteksi: RDS Multi-AZ automated failover (triggered oleh RDS)
Respons: RDS promote standby ke primary di AZ lain
RTO: 1-2 menit (automated failover)
Action dari tim: 
  1. Verify aplikasi reconnect ke new primary
     (PHP PDO persistent connection mungkin perlu di-recycle)
  2. Check bahwa connection pool error spike di grafana sudah normal kembali
  3. Investigate root cause di RDS events
  4. RDS akan otomatis provision new standby di AZ sebelumnya
```


## 4. Disaster Recovery Runbook

### Skenario 1: Single Container Failure

**Deteksi:**  ECS health check gagal  
**Respons:**  ECS otomatis spawn replacement task  _(self-healing)_  
**RTO:**  < 2 menit  
**Action tim:**  Monitor replacement task sehat; investigasi root cause di ECS events.

### Skenario 2: EC2 Host / AZ Failure

**Deteksi:**  CloudWatch alarm + ECS service events  
**Respons:**  ECS reschedule tasks ke healthy host di AZ lain  
**RTO:**  < 5 menit  
**Action tim:**  Verify traffic terdistribusi ke AZ yang tersisa; jika ASG tidak scale otomatis, trigger manual scale-out.

### Skenario 3: Database Primary Failure

**Deteksi:**  RDS Multi-AZ automated failover  _(triggered oleh RDS control plane)_  
**Respons:**  RDS promote standby ke primary di AZ lain  
**RTO:**  1–2 menit  _(automated failover)_  
**Action tim:**

1.  Verify aplikasi reconnect ke new primary — PHP PDO persistent connection mungkin perlu di-recycle
2.  Pastikan connection pool error spike di Grafana sudah kembali normal
3.  Investigate root cause di RDS Events tab
4.  RDS otomatis provision new standby di AZ sebelumnya — monitor sampai selesai

### Skenario 4: Complete Region Failure (ap-southeast-1 down)

**RTO Target: 4 jam | RPO Target: 1 jam**
![enter image description here](https://res.cloudinary.com/djyvswx7e/image/upload/v1774321100/Screenshot_2026-03-24_095736_yjpiqd.png)

---

## 5. Restore Testing: Jadwal dan Prosedur

**Backup yang tidak pernah diuji adalah backup yang nilainya tidak diketahui.**

```yaml
# Jadwal restore testing (wajib, bukan opsional)

quarterly_full_restore_test:
  frequency: Setiap 3 bulan
  scope:
    - Restore full database snapshot ke test environment
    - Verify row count dan data integrity
    - Run application integration tests terhadap restored database
    - Document restore time actual (compare dengan RTO target)
    - Create "restore tested" tag pada snapshot
  
monthly_pitr_test:
  frequency: Setiap bulan
  scope:
    - Restore database ke timestamp acak dalam 30 hari terakhir
    - Verify data konsisten
    - Document actual RPO achieved
    
annual_dr_drill:
  frequency: Setiap tahun
  scope:
    - Full DR activation simulation ke secondary region
    - Include traffic failover test (gunakan test subdomain, bukan production)
    - Measure actual RTO vs. target
    - Identify gaps dalam runbook
    - Update runbook berdasarkan findings
```

### Template Hasil Restore Test

```markdown
## Restore Test Report — 2024-Q2

**Date:** 2024-06-15
**Tested by:** [nama engineer]
**Type:** Quarterly Full Restore Test

**Snapshot used:** rds:myapp-production-2024-06-14
**Target instance:** myapp-restore-test-20240615

**Timeline:**
- 09:00: Restore initiated
- 09:23: Instance available (23 menit — dalam target 30 menit)
- 09:25: Data verification started
- 09:35: Integration tests passed

**Data Verification:**
- Row count match: ✅ 2,847,293 rows in orders table
- Latest transaction timestamp: 2024-06-14 23:58:44 (RPO achieved: ~1 menit)
- Spot check 10 random records: ✅ All match source

**Issues Found:**
- N/A

**RTO Achieved:** 23 menit (Target: 30 menit) ✅
**RPO Achieved:** 1 menit (Target: 5 menit) ✅

**Action Items:**
- N/A
```

---

*Recovery readiness bukan seberapa bagus dokumentasinya — melainkan seberapa cepat sistem kembali normal saat dokumentasi itu benar-benar harus digunakan di tengah tekanan incident.*
