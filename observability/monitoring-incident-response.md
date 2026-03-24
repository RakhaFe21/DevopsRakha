# E. Monitoring, Logging, dan Incident Visibility

> Observability yang baik bukan tentang berapa banyak data yang dikumpulkan — ini tentang **seberapa cepat kita bisa menjawab pertanyaan yang tepat saat production bermasalah**. Dashboard yang penuh dengan grafik tapi tidak bisa menjawab "kenapa user X tidak bisa checkout sejak pukul 14:32?" adalah dashboard dekorasi.

---

## 1. Stack Observability

![enter image description here](https://res.cloudinary.com/djyvswx7e/image/upload/v1774320716/Screenshot_2026-03-24_095106_lgfr6b.png)

**Mengapa Loki bukan ELK:**  
Loki menggunakan label-based indexing (bukan full-text indexing seperti Elasticsearch). Ini berarti biaya storage dan compute jauh lebih rendah pada volume log yang besar. Trade-off: LogQL tidak sefleksibel Elasticsearch query language untuk full-text search. Untuk use case ini (structured application logs, query berdasarkan trace ID atau user ID), Loki lebih dari cukup.

---

## 2. Golden Signals: Empat Sinyal yang Paling Penting

Framework Google SRE mendefinisikan empat golden signals. Ini adalah minimum yang harus ada sebelum metric lainnya:

### Latency (Latensi)

```yaml
# Prometheus recording rule — pre-compute P95/P99 untuk efisiensi query
groups:
  - name: latency_rules
    rules:
      - record: job:http_request_duration_seconds:p95
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (job, le)
          )
      
      - record: job:http_request_duration_seconds:p99
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (job, le)
          )
```

**Alert: P95 latency melebihi SLO**
```yaml
- alert: HighP95Latency
  expr: job:http_request_duration_seconds:p95 > 0.8
  for: 5m
  labels:
    severity: warning
    team: backend
  annotations:
    summary: "P95 latency {{ $value | humanizeDuration }} exceeds 800ms SLO"
    runbook: "https://runbooks.internal/high-latency"
    # Setiap alert WAJIB memiliki runbook URL. Alert tanpa runbook
    # hanya menciptakan panic, bukan resolusi.

- alert: CriticalP99Latency
  expr: job:http_request_duration_seconds:p99 > 2.0
  for: 2m
  labels:
    severity: critical
    team: backend
```

**Catatan penting tentang latency measurement:**  
Ukur latency dari perspektif ALB (access log), bukan hanya dari dalam aplikasi. Latency yang diukur dalam aplikasi tidak menangkap network overhead, TLS handshake time, atau queue time di load balancer. Perbedaan antara keduanya adalah signal penting.

### Traffic (Volume Request)

```yaml
- record: job:http_requests_total:rate5m
  expr: sum(rate(http_requests_total[5m])) by (job, endpoint, status_code)
```

Traffic yang tiba-tiba turun drastis adalah signal yang sama pentingnya dengan traffic yang naik. Drop 80% dalam 2 menit bisa berarti: CDN misconfiguration, DNS failure, load balancer health check mulai gagal, atau (worst case) DDoS protection yang terlalu agresif memblok legitimate users.

```yaml
- alert: UnexpectedTrafficDrop
  expr: |
    (
      sum(rate(http_requests_total[5m])) 
      / 
      sum(rate(http_requests_total[5m] offset 1h))
    ) < 0.3
  for: 3m
  labels:
    severity: critical
  annotations:
    summary: "Traffic dropped to {{ $value | humanizePercentage }} of 1-hour-ago baseline"
```

### Errors (Error Rate)

```yaml
- record: job:http_error_rate:rate5m
  expr: |
    sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (job)
    /
    sum(rate(http_requests_total[5m])) by (job)

- alert: HighErrorRate
  expr: job:http_error_rate:rate5m > 0.01  # > 1% error rate
  for: 2m
  labels:
    severity: warning

- alert: CriticalErrorRate  
  expr: job:http_error_rate:rate5m > 0.05  # > 5% error rate
  for: 1m
  labels:
    severity: critical
```

**Jangan aggregate semua 5xx menjadi satu metric.** 502 Bad Gateway dari ALB (berarti backend tidak bisa dijangkau) sangat berbeda dari 500 Internal Server Error dari aplikasi (berarti code error). Pisahkan dimensi `source` (alb vs application) di label.

### Saturation (Saturasi Resource)

```yaml
# CPU Saturation — lebih berguna untuk trend daripada spike
- alert: HighCPUSaturation
  expr: |
    avg(rate(container_cpu_usage_seconds_total{container="myapp-api"}[5m])) 
    / 
    avg(container_spec_cpu_quota{container="myapp-api"} / 100000) > 0.8
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "CPU usage at {{ $value | humanizePercentage }} of limit for 10 minutes"

# Memory — memory pressure sebelum OOM kill
- alert: HighMemoryPressure
  expr: |
    container_memory_working_set_bytes{container="myapp-api"}
    /
    container_spec_memory_limit_bytes{container="myapp-api"} > 0.85
  for: 5m
  labels:
    severity: warning

# Database connection pool exhaustion — sering diabaikan tapi sangat berbahaya
- alert: DBConnectionPoolExhaustion
  expr: |
    pg_stat_database_numbackends{datname="myapp_production"}
    /
    pg_settings_max_connections > 0.8
  for: 3m
  labels:
    severity: critical
  annotations:
    summary: "PostgreSQL connection pool at {{ $value | humanizePercentage }}"
    # Pool exhaustion menyebabkan connection timeout yang terlihat sebagai
    # 500 errors di aplikasi, bukan sebagai database error.
```

---

## 3. Log Aggregation Strategy

### Structured Logging: Bukan Sekadar Format

Semua log harus berupa JSON dengan field yang konsisten:

```json
{
  "timestamp": "2024-06-15T14:32:01.234Z",
  "level": "error",
  "trace_id": "abc123def456",
  "span_id": "789xyz",
  "user_id": "usr_98765",
  "request_id": "req_abc123",
  "method": "POST",
  "path": "/api/orders",
  "duration_ms": 234,
  "status_code": 500,
  "error": "SQLSTATE[HY000]: General error: 1205 Lock wait timeout exceeded",
  "service": "myapp-api",
  "version": "abc1234def5",
  "environment": "production"
}
```

**`trace_id` adalah field paling penting** — ini yang menghubungkan log request yang sama di semua service. Tanpa trace_id, menginvestigasi request yang melewati lebih dari satu service adalah pekerjaan detektif manual yang memakan waktu berjam-jam.

### Log Level Discipline

```php
// Laravel: konfigurasi log level per environment
// config/logging.php

'channels' => [
    'stderr' => [
        'driver' => 'monolog',
        'handler' => StreamHandler::class,
        'with' => ['stream' => 'php://stderr'],
        'level' => env('LOG_LEVEL', 'error'),
        // Production default: error — hanya log yang benar-benar perlu ditindaklanjuti
        // Staging default: warning — sedikit lebih verbose untuk debugging
        // Development: debug — everything
    ],
],
```

**Jangan log DEBUG atau INFO di production** kecuali sedang dalam active debugging session yang terdefinisi (dan dimatikan kembali setelah selesai). Volume log yang terlalu tinggi menyembunyikan signal penting dalam noise dan meningkatkan biaya storage.

### Log Retention Policy

```yaml
# CloudWatch Log Group — per service
retention_days:
  production_application: 30  # Cukup untuk investigasi incident dan audit
  production_access_log: 90   # ALB access log: diperlukan untuk security audit
  staging: 7                  # Staging tidak perlu history panjang
  
# Cost awareness: 1GB CloudWatch Logs ingestion = ~$0.50/bulan
# 1GB CloudWatch Logs storage (after first month) = ~$0.03/GB/bulan
# Retention policy yang tepat bisa mengurangi biaya log 60-80%
```

---

## 4. Distributed Tracing

Untuk sistem dengan multiple services (API, Worker, scheduled jobs), distributed tracing adalah perbedaan antara "ada error di satu service" dan "inilah root cause dari error tersebut".

```php
// Menggunakan OpenTelemetry PHP SDK
// Trace context di-propagate antar service via HTTP headers (W3C Trace Context)

$tracer = Globals::tracerProvider()->getTracer('myapp-api');

$span = $tracer->spanBuilder('process-order')
    ->setSpanKind(SpanKind::KIND_INTERNAL)
    ->startSpan();

$span->setAttribute('order.id', $orderId);
$span->setAttribute('user.id', $userId);

try {
    // Business logic
    $result = $this->orderService->process($orderId);
    $span->setStatus(StatusCode::STATUS_OK);
} catch (\Exception $e) {
    $span->setStatus(StatusCode::STATUS_ERROR, $e->getMessage());
    $span->recordException($e);
    throw $e;
} finally {
    $span->end();
}
```

---

## 5. Dashboard Design: Pertanyaan yang Harus Dijawab

Setiap dashboard dirancang untuk menjawab pertanyaan spesifik, bukan untuk menampilkan semua metric yang tersedia.

### Dashboard 1: "Apakah Production Sehat Sekarang?" (SRE Overview)

Pertanyaan yang harus terjawab dalam 30 detik:
- Apakah semua service UP?
- Apakah error rate di bawah threshold?
- Apakah latency P95 dalam SLO?
- Apakah ada active alert?

### Dashboard 2: "Apa yang Terjadi Saat Incident X?"

Pertanyaan yang harus terjawab untuk incident investigation:
- Kapan masalah mulai terjadi? (timeline correlation)
- Service mana yang pertama menunjukkan anomali?
- Deployment atau config change apa yang terjadi sebelum incident?
- User mana yang terdampak dan seberapa luas?

### Dashboard 3: "Apakah Kita Mendekati Kapasitas Batas?" (Capacity Planning)

Pertanyaan untuk capacity review mingguan:
- Tren CPU dan memory usage selama 4 minggu terakhir
- Database connection pool utilization trend
- Storage growth rate
- Cache hit rate trend

---

## 6. Alerting: Filosofi yang Mengurangi Alert Fatigue

**Alert yang jelek lebih berbahaya dari tidak ada alert sama sekali** — karena alert yang terlalu sering false positive menyebabkan tim mulai mengabaikan alert sama sekali, termasuk yang genuine.

```
Prinsip alerting yang saya terapkan:

1. Alert pada SYMPTOM user-facing, bukan pada cause internal
   ✅ "Error rate > 2% selama 5 menit"
   ❌ "CPU > 80%" (CPU tinggi tidak selalu menyebabkan user impact)

2. Alert harus actionable — setiap alert memiliki runbook
   ✅ Alert + runbook URL
   ❌ Alert yang hanya bisa dijawab "tunggu sampai turun sendiri"

3. Severity yang tepat
   critical → bangunkan on-call engineer sekarang
   warning  → perlu perhatian, tapi tidak harus malam ini
   info     → untuk review dan trending, tidak perlu aksi immediate

4. Alert must have a clear owner
   Setiap alert memiliki label `team:` yang jelas
   Tidak ada alert yang "everyone's problem" = no one's problem
```

---

*Observability yang baik diukur bukan dari jumlah tools atau metric yang dikumpulkan, melainkan dari seberapa cepat tim bisa diagnosa dan merespons masalah production. MTTD (Mean Time to Detect) dan MTTR (Mean Time to Recover) adalah metric yang benar-benar penting.*
