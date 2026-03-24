
# C. Deployment Flow dan CI/CD

> Deployment bukan sekadar "menjalankan kode baru di server". Deployment adalah proses terkontrol yang harus dapat diulang, dapat diaudit, aman untuk dijalankan kapanpun, dan memiliki escape hatch yang jelas saat sesuatu tidak berjalan sesuai rencana.

---

## 1. Filosofi Deployment

**Pertanyaan yang harus terjawab sebelum setiap deployment:**

1. Apakah semua test gate lulus? (bukan sekadar unit test — integration test juga)
2. Apakah image yang akan di-deploy sudah discan dan tidak memiliki critical vulnerability?
3. Apakah ada database migration? Jika ya, apakah migration bersifat backward-compatible?
4. Apakah ada dependency service yang sedang degraded saat ini?
5. Jika deployment ini gagal, apa rollback path-nya dan berapa lama?

Jika pertanyaan nomor 5 tidak bisa dijawab dengan cepat, deployment harus ditunda.

---

## 2. Deployment Strategy: Blue/Green

### Mengapa Blue/Green untuk Skenario Ini

Sistem yang dimigrasikan dari monolith menggunakan stateful session (cookie-based auth, tidak stateless JWT). Ini berarti rolling update akan menyebabkan sebagian request diarahkan ke version baru sementara sebagian lagi masih ke version lama — user yang sedang login bisa mengalami session invalidation atau inconsistent state.

Blue/Green menghilangkan masalah ini: seluruh traffic berpindah atomik dari environment lama (Blue) ke environment baru (Green) melalui satu ALB target group swap.

```
Before Deployment:
  ALB → Target Group Blue (v1.2.3) ← 100% traffic

During Deployment:
  ALB → Target Group Blue (v1.2.3) ← 100% traffic (tidak terganggu)
  Green environment (v1.2.4) spin up, warm up, health check pass

After Swap:
  ALB → Target Group Green (v1.2.4) ← 100% traffic
  Blue environment tetap berjalan (standby rollback)

Rollback jika diperlukan (< 2 menit):
  ALB → Target Group Blue (v1.2.3) ← 100% traffic (swap kembali)
  Green environment terminated
```

### Syarat untuk Blue/Green yang Aman

1. **Database migrations harus backward-compatible** — Green (v1.2.4) harus bisa berjalan dengan schema yang sama dengan Blue (v1.2.3). Expand-and-contract pattern: pertama tambahkan kolom baru (expand), lalu setelah semua traffic di Green, hapus kolom lama (contract) di deployment berikutnya.

2. **Health check endpoint `/health` harus substantif** — bukan hanya HTTP 200, tapi verifikasi koneksi ke database dan cache. Green tidak boleh menerima live traffic sebelum semua dependency-nya healthy.

3. **Session state harus tersimpan di Redis**, bukan di application server memory — ini adalah persyaratan arsitektural untuk Blue/Green yang efektif.

---

## 3. Pipeline Stages

![enter image description here](https://res.cloudinary.com/djyvswx7e/image/upload/v1774320303/Screenshot_2026-03-24_094446_tvx99t.png)

---

## 4. Rollback Awareness

### Rollback Harus Lebih Mudah dari Deploy

Ini bukan sekadar slogan. Rollback yang sulit menyebabkan tim menunda rollback saat seharusnya rollback — dan downtime memanjang akibat sunk cost fallacy ("kita sudah deploy, sayang kalau di-rollback").

**Rollback mechanisms by layer:**

| Layer | Rollback Method | RTO |
|---|---|---|
| Application (container) | ALB target group swap ke Blue | < 2 menit |
| Database migration | Hanya expand (tidak pernah destructive di satu deployment) | N/A — backward-compatible |
| Config/Secrets | Versi eksplisit di Secrets Manager, rollback ke previous version | < 1 menit |
| Infrastructure (Terraform) | `terraform apply` dari previous state file | 5-15 menit |

### Kapan Rollback, Kapan Forward Fix

- **Rollback segera** jika: error rate naik > 5% dalam 5 menit post-deploy, atau P95 latency naik > 2x baseline.
- **Forward fix** hanya jika: issue terisolasi pada feature non-critical yang bisa di-feature-flag, dan rollback akan menyebabkan data loss.

---

## 5. Deployment Window Policy

```yaml
# Deployment ke production HANYA diizinkan pada:
allowed_windows:
  - Mon-Thu: 09:00 - 16:00 WIB
  - Fri: 09:00 - 13:00 WIB  # sebelum long weekend risk

blocked_periods:
  - Jumat 13:00 - Senin 09:00
  - H-3 dan H+1 setiap major event (promosi, campaign, live event)
  - Saat ada active incident (obvious, tapi perlu dikodifikasikan)
```

Policy ini bukan bureaucracy — ini adalah perlindungan. Engineer yang on-call pada Jumat malam dengan deployment yang baru saja dilakukan memiliki cognitive load yang jauh lebih tinggi, dan quality keputusan mereka saat incident terdegradasi.

---

*File YAML pipeline ada di [.github/workflows/production-release.yml](../.github/workflows/production-release.yml)*
