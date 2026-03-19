# DevOps / Infrastructure — Senior Proof-of-Work Submission

> **Kandidat:** Rakha  
> **Posisi:** Senior DevOps / Infrastructure Engineer  
> **Repository ini** adalah proof-of-work profesional yang mendemonstrasikan cara berpikir arsitektural, penilaian risiko operasional, dan pengambilan keputusan infrastruktur pada level senior — bukan kumpulan tutorial atau portofolio generik.

---

## Navigasi Submission

| Area | Dokumen |
|---|---|
| **A. Executive Overview & Trade-off** | README.md (dokumen ini) |
| **B. Infrastructure Architecture Design** | [docs/01-architecture-design.md](docs/01-architecture-design.md) |
| **C. Deployment Flow & CI/CD** | [cicd/deployment-flow.md](cicd/deployment-flow.md) · [cicd/.github/workflows/production-release.yml](cicd/.github/workflows/production-release.yml) |
| **D. Containerization & Consistency** | [containerization/Dockerfile](containerization/Dockerfile) · [containerization/consistency-strategy.md](containerization/consistency-strategy.md) |
| **E. Monitoring, Logging & Observability** | [observability/monitoring-incident-response.md](observability/monitoring-incident-response.md) |
| **F. Security, Access Control & Secrets** | [security/security-baseline.md](security/security-baseline.md) |
| **G. Backup, Restore & DR** | [operations/backup-dr-policy.md](operations/backup-dr-policy.md) |
| **H. Capacity, Performance & Cost** | [operations/capacity-performance.md](operations/capacity-performance.md) |

---

## A. Executive Overview

### Problem Statement: Migrasi Monolith ke Containerized Microservices dengan Zero-Downtime

Sistem yang saya hadapi dalam konteks ini adalah aplikasi web monolitik berbasis Laravel yang melayani ~50.000 daily active users, di-deploy ke dua VPS bare-metal secara manual via SSH. Tidak ada environment separation yang jelas antara staging dan production. Deployment dilakukan dengan `git pull` langsung di server production. Tidak ada rollback path yang terdefinisi. Logging tersebar di file lokal tanpa aggregation. Secret di-hardcode dalam `.env` yang di-commit ke repo internal.

Kondisi ini bukan kondisi yang "gagal" — sistem masih berjalan. Tapi ini adalah sistem yang **fragile by design**: ketika volume transaksi tumbuh 3x dalam 6 bulan ke depan, atau ketika satu engineer resign membawa pengetahuan operasional di kepalanya, risiko collapse meningkat secara eksponensial.

**Mandat saya dalam skenario ini:**  
Membawa sistem dari kondisi fragile menuju **operationally resilient** — tanpa menghentikan layanan, dan tanpa membuang fondasi yang sudah ada jika masih layak dipertahankan.

---

### Cara Saya Membaca Problem Operasional

Sebelum menyentuh satu konfigurasi pun, saya melakukan **operational risk mapping** dengan tiga lensa:

**1. Blast Radius Assessment**  
Apa yang terjadi jika deployment gagal sekarang? Apakah ada cara rollback? Berapa lama downtime yang dapat ditoleransi? Jawaban atas pertanyaan ini menentukan urutan prioritas perubahan — bukan keinginan untuk menggunakan teknologi terbaru.

**2. Operational Knowledge Distribution**  
Seberapa banyak pengetahuan operasional yang hanya ada di kepala satu orang? Infrastructure yang tidak terdokumentasi dan tidak ter-codified adalah single point of failure yang paling berbahaya, karena ia tidak terlihat di monitoring manapun.

**3. Drift Surface**  
Di mana konfigurasi antara development, staging, dan production mulai berbeda tanpa disadari? Environment drift adalah akar dari mayoritas "works on my machine" incidents di production.

---

### Prinsip yang Saya Gunakan dalam Menata Environment Production

**Infrastructure as Code, bukan Infrastructure as Memory**  
Setiap resource harus dapat di-*reproduce* dari kode, bukan dari ingatan. Ini bukan sekadar best practice — ini adalah persyaratan minimum agar on-call rotation bisa berjalan dengan manusia yang berbeda.

**Defense in Depth untuk Security**  
Security bukan satu layer yang ditambahkan di akhir. Ia harus ada di setiap level: image scanning saat build, secret management di runtime, network boundary di infrastructure layer, dan audit log di semua access point.

**Observability sebagai First-Class Citizen**  
Dashboard yang hanya menampilkan CPU usage bukan observability. Observability yang berguna menjawab pertanyaan: *"Mengapa pengguna di region X mengalami latency tinggi sejak 14:32 tadi?"* Ini memerlukan distributed tracing, structured logging, dan alerting yang dikalibrasi terhadap symptom user-facing, bukan hanya resource metrics.

**Reliability Budget, bukan Zero-Downtime Absolutism**  
Mengejar 100% uptime adalah posisi yang tidak rasional secara operasional — ia menciptakan fear of change yang justru memperlambat perbaikan sistem. Pendekatan yang tepat adalah mendefinisikan SLO (Service Level Objective) yang realistis (misalnya 99.9% monthly uptime = ~43 menit downtime per bulan yang dapat ditoleransi), lalu membuat keputusan engineering berdasarkan error budget tersebut.

---

### Menyeimbangkan Kecepatan Delivery dengan Reliability dan Keamanan Jangka Panjang

Ketegangan antara *move fast* dan *stay stable* adalah ketegangan yang nyata, dan engineer yang mengklaim tidak ada trade-off di sini biasanya belum pernah menangani incident production yang serius.

Posisi saya: **kecepatan delivery yang berkelanjutan hanya bisa dicapai melalui reliability yang kokoh**, bukan melalui kompromi terhadapnya. Tim yang memiliki rollback yang solid, CI/CD yang matang, dan monitoring yang berguna justru bisa deploy lebih sering dengan confidence lebih tinggi — karena mereka tahu bahwa jika sesuatu salah, mereka akan tahu dalam menit, bukan jam, dan bisa kembali ke kondisi stabil dalam hitungan menit.

---

## I. Pertimbangan Teknis dan Trade-off

Bagian ini mendokumentasikan keputusan arsitektural utama, alternatif yang dipertimbangkan, dan alasan di balik pilihan yang diambil.

---

### Trade-off 1: Managed Kubernetes (EKS/GKE) vs. Self-Hosted K3s vs. Docker Compose + Systemd

**Konteks:** Saat memilih platform runtime untuk containerized workload.

| Opsi | Kelebihan | Kekurangan | Kapan Tepat |
|---|---|---|---|
| **EKS/GKE** | Zero operational overhead untuk control plane; built-in integrations (IAM, LB, storage); battle-tested HA | Biaya tinggi; vendor lock-in; kompleksitas networking (CNI, IRSA) perlu pemahaman dalam | Tim ≥5 engineer; workload yang genuinely membutuhkan auto-scaling dan multi-tenancy |
| **K3s self-hosted** | Ringan; full K8s API compatibility; cocok untuk on-prem/VPS | Kamu bertanggung jawab atas HA control plane, etcd backup, upgrade cycle | Tim kecil dengan budget terbatas tapi butuh K8s primitives |
| **Docker Compose + Systemd** | Operasional sederhana; mudah di-debug; tidak ada abstraksi berlebihan | Tidak ada built-in orchestration; scaling manual; tidak ada self-healing primitif | Monolith atau aplikasi dengan komponen terbatas; tim yang tidak memiliki K8s expertise |

**Keputusan yang saya ambil:** Untuk skenario migrasi ini, saya merekomendasikan **Docker Compose + Nginx (reverse proxy) di fase pertama**, dengan path eksplisit menuju K3s atau EKS di fase kedua setelah workload patterns dipahami. Alasan: over-engineering dengan full K8s di tahap awal migrasi dari monolith menciptakan dua masalah sekaligus — kompleksitas operasional yang belum siap ditangani tim, dan debugging yang lebih sulit karena terlalu banyak abstraksi baru diperkenalkan bersamaan.

---

### Trade-off 2: Blue/Green Deployment vs. Canary Release vs. Rolling Update

**Konteks:** Memilih deployment strategy untuk production.

| Strategi | Rollback Speed | Resource Cost | Complexity | Risk Exposure |
|---|---|---|---|---|
| **Blue/Green** | Instan (DNS/LB switch) | 2x resource saat deployment | Medium | Rendah — traffic berpindah atomik |
| **Canary** | Bertahap (perlu traffic shifting logic) | Minimal | Tinggi | Rendah — exposure bertahap, isolable |
| **Rolling Update** | Lambat (perlu rollback deployment) | Minimal | Rendah | Medium — partial version in-flight |

**Keputusan:** Blue/Green untuk services stateless dengan clear health check endpoint. Canary untuk perubahan yang menyentuh shared state (database schema, cache key format) di mana kita perlu memvalidasi dengan sebagian kecil traffic sebelum full rollout. Rolling update hanya untuk non-critical services internal dengan SLO rendah.

---

### Trade-off 3: HashiCorp Vault vs. AWS Secrets Manager vs. Environment Variables di CI/CD

**Konteks:** Secret management untuk aplikasi yang berjalan di multi-environment.

| Opsi | Audit Capability | Rotation Support | Operational Overhead | Portability |
|---|---|---|---|---|
| **HashiCorp Vault** | Excellent (full audit log) | Native (dynamic secrets) | Tinggi (cluster management) | Tinggi (cloud-agnostic) |
| **AWS Secrets Manager** | Good (CloudTrail integration) | Native (Lambda rotation) | Rendah (managed) | Rendah (AWS lock-in) |
| **ENV vars di CI/CD** | Minimal | Manual | Minimal | Tinggi | 

**Keputusan:** AWS Secrets Manager untuk production jika sudah all-in di AWS ekosistem, dengan IAM role-based access (IRSA untuk EKS). HashiCorp Vault jika multi-cloud atau on-prem requirement ada. ENV vars di CI/CD hanya untuk non-sensitive build-time configuration — **tidak pernah** untuk credentials, API keys, atau database passwords.

---

### Trade-off 4: Centralized Logging (ELK) vs. Managed (Datadog/New Relic) vs. Loki + Grafana

**Konteks:** Log aggregation dan observability stack.

| Opsi | Kelebihan | Kekurangan |
|---|---|---|
| **ELK Stack** | Powerful querying; full control; no per-log cost | Operasional berat; Elasticsearch memory-intensive; butuh dedicated team |
| **Datadog/New Relic** | Turnkey; excellent APM integration; low ops overhead | Biaya tinggi di scale; data residency concerns; vendor lock-in |
| **Loki + Grafana + Tempo** | Cost-efficient (label-based index, not full-text index); CNCF native; integrates cleanly dengan Prometheus | Query language (LogQL) learning curve; less powerful full-text search |

**Keputusan:** Loki + Grafana + Prometheus untuk greenfield atau budget-constrained production. ELK hanya jika ada kebutuhan full-text search yang kompleks dan tim yang mampu mengoperasikan cluster Elasticsearch. Datadog untuk tim yang prioritasnya MTTR rendah dan cost bukan constraint utama.

---

*Repository ini disusun sebagai dokumen kerja profesional. Setiap keputusan teknis memiliki alasan yang dapat dipertanggungjawabkan, bukan sekadar preferensi tool.*
