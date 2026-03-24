# D. Containerization dan Consistency Strategy

> Konsistensi runtime bukan tentang memastikan semua environment memiliki Dockerfile yang sama. Ini tentang memastikan bahwa **perilaku aplikasi dapat diprediksi** di setiap environment — dan ketika terjadi perbedaan perilaku, kita tahu *persis* di mana perbedaannya.

---

## 1. Prinsip Dasar: Image sebagai Unit of Deployment

**Satu image, banyak environment.**

Image yang di-deploy ke production adalah **image yang sama** yang sudah lolos test di CI dan di-deploy ke staging — bukan image yang di-build ulang. Ini bukan preferensi, ini adalah syarat minimum untuk reproducibility.

![enter image description here](https://res.cloudinary.com/djyvswx7e/image/upload/v1774320515/Screenshot_2026-03-24_094804_xqsfin.png)

**Apa yang berbeda antar environment bukan imagenya — melainkan konfigurasinya.** Database URL berbeda, log level berbeda, feature flags berbeda — tapi binary yang berjalan adalah identik.

---

## 2. Environment Parity: Menghilangkan "Works on My Machine"

### Sumber Drift yang Paling Umum

| Sumber Drift | Manifestasi | Mitigasi |
|---|---|---|
| PHP versi berbeda di local dan production | `str_contains()` tersedia di PHP 8.0+, bukan 7.4 | Dev menggunakan versi yang sama dengan production via Docker |
| Timezone berbeda | Date calculation berbeda; job scheduling off-by-hours | `TZ=Asia/Jakarta` eksplisit di semua environment |
| Locale dan charset berbeda | String comparison menghasilkan hasil berbeda | Definisikan di PHP config: `mbstring.language = Japanese` → `Neutral` |
| File permission berbeda | Aplikasi bisa tulis file di local, tidak bisa di container | Test dengan user non-root di local development juga |
| OS-level library versi berbeda | Crypto behavior berbeda; SSL handshake berbeda | Semua dependency di-pin di Dockerfile |
| Urutan env variable loading | `.env.local` override `.env` di local, tapi tidak ada di production | Audit env var precedence secara eksplisit |

### Dev Environment dengan Docker Compose

```yaml
# docker-compose.dev.yml
# Catatan: ini BUKAN konfigurasi untuk production.
# Production menggunakan ECS. Ini hanya untuk local development.

version: "3.9"

services:
  api:
    # Menggunakan image yang sama dengan production — bukan image khusus "dev"
    # yang mungkin memiliki extra packages atau konfigurasi berbeda.
    build:
      context: .
      dockerfile: containerization/Dockerfile
      target: production  # Build stage yang sama dengan production
    image: myapp-api:dev-local
    
    volumes:
      # Mount source code untuk hot-reload — tapi HANYA di dev.
      # Di production tidak ada volume mount ke source code.
      - ./:/var/www/html:delegated
      - /var/www/html/vendor  # Preserve vendor dari image, jangan di-override
      - /var/www/html/public/build  # Preserve built assets
    
    environment:
      APP_ENV: local
      APP_DEBUG: "true"
      DB_HOST: postgres
      REDIS_HOST: redis
      # Secrets tidak pernah di-hardcode di compose file.
      # Gunakan .env file (tidak di-commit) atau docker secret.
    
    env_file:
      - .env.local  # Di-gitignore; setiap developer membuat file ini sendiri
    
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    
    user: "1000:1000"  # appuser UID — sama dengan production
    
  postgres:
    image: postgres:15.6-alpine  # Pin exact version, termasuk patch
    environment:
      POSTGRES_DB: myapp_dev
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: devpassword  # Dev-only; tidak pernah digunakan di production
    volumes:
      - postgres_dev_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 5s
      timeout: 5s
      retries: 10
  
  redis:
    image: redis:7.2-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    # maxmemory-policy: production config yang harus di-mirror di dev.
    # Jika production Redis menggunakan allkeys-lru, dev yang tidak
    # mengkonfigurasi ini akan menghasilkan perilaku cache eviction berbeda.
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s

volumes:
  postgres_dev_data:
```

---

## 3. Secret dan Sensitive Config: Separation yang Eksplisit

### Apa yang Boleh Ada di Image vs. Apa yang Tidak Boleh

```
✅ Boleh ada di image:
   - Source code
   - Compiled assets
   - PHP config (php.ini) — tanpa credentials
   - Nginx config — tanpa credentials
   - Default environment variable NAMES (tapi bukan VALUES)

❌ Tidak boleh ada di image:
   - Database credentials
   - API keys (payment gateway, email provider, dsb.)
   - JWT secret keys
   - SSL private keys
   - .env file production
```

**Verifikasi otomatis:** Trivy secret scan di pipeline (lihat Stage 2 di production-release.yml) menangkap credentials yang ter-commit secara tidak sengaja sebelum image di-push ke registry.

### Cara Aplikasi Mendapatkan Secret di Runtime

**Production (ECS + AWS Secrets Manager):**

```json
// ECS Task Definition (relevan bagian saja)
{
  "containerDefinitions": [{
    "name": "myapp-api",
    "secrets": [
      {
        "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:123456789:secret:myapp/production/db-password"
      },
      {
        "name": "REDIS_PASSWORD", 
        "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:123456789:secret:myapp/production/redis-password"
      }
    ],
    "environment": [
      // Non-sensitive config boleh di environment variables biasa
      {"name": "APP_ENV", "value": "production"},
      {"name": "LOG_CHANNEL", "value": "stderr"}
    ]
  }]
}
```

ECS Agent mengambil secret dari Secrets Manager saat container start, dan meng-inject sebagai environment variable ke dalam container. Secret tidak pernah tertulis ke filesystem atau muncul di image layers.

**Rotasi secret tanpa restart container:**  
AWS Secrets Manager mendukung automatic rotation. Untuk menghindari keharusan restart container setiap kali secret dirotasi, aplikasi bisa menggunakan AWS SDK untuk membaca secret langsung dari Secrets Manager pada saat dibutuhkan (bukan hanya saat startup) — khususnya untuk long-lived connections seperti database.

---

## 4. Image Tagging Strategy

```
ECR Repository: myapp-api

Tags yang digunakan:
  :abc1234def5  ← Git SHA (immutable; SELALU digunakan untuk deploy)
  :latest       ← Mutable convenience alias (TIDAK PERNAH digunakan untuk deploy)
  :main-20240615-1423  ← Branch + timestamp (untuk debugging kronologi)

Tags yang TIDAK digunakan:
  :v1.2.3       ← Semantic versioning di container image menciptakan dual
                   versioning system yang sering tidak sinkron dengan git history
```

**Mengapa SHA-based tagging:**
- Deterministic: kita tahu persis commit apa yang menghasilkan image ini
- Immutable: tidak bisa di-overwrite secara tidak sengaja
- Traceable: dari image yang berjalan di production, kita bisa langsung ke exact commit, CI run, dan test results

---

## 5. Image Lifecycle Management

```bash
# ECR Lifecycle Policy — mencegah registry penuh dengan image lama
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 images tagged with SHA (production deployable)",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["sha-"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Remove untagged images after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    }
  ]
}
```

**Catatan penting:** Sebelum ECR Lifecycle Policy menghapus image lama, pastikan tidak ada ECS service yang masih menggunakan image tersebut. Jika rollback diperlukan, kita harus bisa me-reference image lama dari registry. 10 image terakhir memberikan buffer untuk setidaknya 10 deployment rollback window.

---

## 6. Container Runtime Konfigurasi: Security dan Resource Constraints

```hcl
# Terraform — ECS Task Definition security constraints
resource "aws_ecs_task_definition" "api" {
  # ...
  
  container_definitions = jsonencode([{
    name  = "myapp-api"
    image = "${var.ecr_image_uri}"
    
    # Resource constraints — WAJIB di production
    # Tanpa memory limit, satu container yang memory-leak bisa
    # menghabiskan seluruh host memory dan men-crash container lain.
    cpu    = 512   # 0.5 vCPU
    memory = 1024  # Hard limit: container dikill jika melewati ini
    memoryReservation = 768  # Soft limit: container bisa burst ke hard limit
    
    # Security: read-only root filesystem
    # Aplikasi tidak boleh menulis ke root filesystem di runtime.
    # Jika ada yang mencoba (misalnya malware atau misconfigured dependency),
    # ini akan fail secara eksplisit, bukan silent.
    readonlyRootFilesystem = true
    
    # Writable mounts — hanya direktori yang memang perlu ditulis
    mountPoints = [
      {
        sourceVolume  = "app-storage"
        containerPath = "/var/www/html/storage"
        readOnly      = false
      }
    ]
    
    # Non-root user
    user = "1000:1000"
    
    # Tidak ada privileges tambahan
    privileged = false
    
    # Drop semua Linux capabilities; tambahkan hanya yang diperlukan
    linuxParameters = {
      capabilities = {
        drop = ["ALL"]
        add  = []  # Aplikasi web tidak memerlukan capabilities apapun
      }
    }
    
    # Log ke CloudWatch — bukan ke file lokal
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = "/ecs/myapp-api"
        awslogs-region        = "ap-southeast-1"
        awslogs-stream-prefix = "ecs"
      }
    }
  }])
}
```

---

*Konsistensi runtime adalah properti sistem, bukan properti individual container. Setiap keputusan yang didokumentasikan di sini berkontribusi pada kemampuan untuk memprediksi — dan men-debug — perilaku aplikasi di semua environment.*
