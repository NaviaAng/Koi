# Koi ERP – Vertical Slice Modular Monolith

Koi ERP adalah hasil re-engineering dari sistem IBS lama menjadi **.NET Modular Monolith** dengan pendekatan **Vertical Slice + Domain-Driven Design (DDD)**.  
Tujuan utama: modernisasi arsitektur, efisiensi tim kecil (2–5 dev), dan kesiapan menuju **microservices** bila kebutuhan bisnis meningkat.

---

## 🚀 Tech Stack

- **Backend**:  
  - C# (.NET 13 / ASP.NET Core 10)  
  - CQRS + Vertical Slice + DDD  
- **Database**: PostgreSQL (schema-per-module, EF Core migrations)  
- **Integrasi**:  
  - Outbox Pattern + Event Bus (RabbitMQ / Kafka)  
  - Staging DB untuk IBS/DMS integration  
- **Security & IAM**:  
  - **Keycloak** sebagai Identity & Access Management (OAuth2.0 / OpenID Connect)  
  - JWT + RBAC + Fine-grained permission  
  - Multi-Tenancy dengan `TenantId` filtering  
- **Deployment & Infra**:  
  - Docker + Kubernetes (opsional)  
  - CI/CD (GitHub Actions / Azure DevOps)  
  - IaC (Terraform / Ansible)  
- **Observability**:  
  - Logging → Serilog + Seq  
  - Tracing → OpenTelemetry + Jaeger  
  - Metrics → Prometheus + Grafana  

---

## 🔒 Identity & Access Management (IAM)

Koi ERP menggunakan **Keycloak** sebagai penyedia IAM utama.  

- **Authentication**  
  - Semua modul `Koi.*` memvalidasi token JWT dari Keycloak  
  - Mendukung SSO (Microsoft, Google, LDAP)  
  - Refresh token & session management handled by Keycloak  

- **Authorization**  
  - Role-Based Access Control (RBAC) dikelola di Keycloak  
  - Fine-grained permission ditangani oleh modul `Koi.Security`  
  - Context-based access (misal: `BranchId`, `TenantId`)  

- **Integration Flow**  
  - `Koi.Identity` berperan sebagai adapter antara Keycloak dan aplikasi  
  - Sinkronisasi user, role, dan branch data ke internal DB  
  - Semua request ke API → validasi JWT di middleware → cek permission di `Koi.Security`  

---

## 📂 Struktur Project

```
Koi.sln
├─ build/ # build scripts, CI/CD pipeline
├─ docker/ # infra dev (Postgres, Jaeger, Seq)
├─ src/
│ ├─ Koi.Bootstrap # ASP.NET Core host (composition root)
│ ├─ Koi.Shared # shared kernel (value object, util)
│ ├─ Koi.Core # CQRS, event bus, infra base
│ ├─ Koi.Integration # integrasi IBS lama, DMS
│ ├─ Koi.Identity # adapter Keycloak + identity sync
│ ├─ Koi.Security # RBAC, fine-grained permission
│ ├─ Koi.Janaru # modul ERP (GL, AR, AP, Purchase, Invoice)
│ ├─ Koi.Workshop # Service & BodyPaint
│ └─ Koi.Sales # Sales order & delivery
└─ tests/ # unit & integration test
```


---

## 🧩 Modul Utama

- **Koi.Janaru.GeneralLedger (GL)** → pusat jurnal & laporan keuangan  
- **Koi.Janaru.AccountReceivable (AR)** → piutang pelanggan  
- **Koi.Janaru.AccountPayable (AP)** → hutang vendor  
- **Koi.Janaru.Purchase** → siklus pembelian  
- **Koi.Janaru.Invoice** → pusat billing, integrasi AR/AP  
- **Koi.Sales** → sales order & delivery  
- **Koi.Workshop** → service & body paint  
- **Koi.Identity** → integrasi & sinkronisasi dengan Keycloak  
- **Koi.Security** → enforcement RBAC & permission  

---

## ✅ Testing

- **Unit Test** → xUnit / NUnit  
- **Integration Test** → TestContainers (Postgres)  
- **Contract Test** → Pact  
- **End-to-End Test** → Playwright / Selenium  
- **Coverage target**: 70%+ business logic  
- Semua test berjalan otomatis via CI/CD  

---

## 📦 Deployment

- **Environment**: Development → Staging → Production  
- **Pipeline**:  
  1. Build & Test  
  2. Security Scan (Trivy, Snyk)  
  3. Deploy ke Staging  
  4. Smoke Test  
  5. Deploy ke Production (Blue/Green, rollback otomatis)  
- **Secrets**: dikelola via HashiCorp Vault / Azure KeyVault  

---

## 📑 Coding Guidelines

- Namespace: `Koi.<Domain>.<Module>`  
- Ikuti SRP (Single Responsibility Principle) per modul slice  
- Error handling pakai `Result<T>` pattern  
- Semua perubahan masuk via Pull Request → minimal 2 reviewer  
- API wajib didokumentasikan dengan Swagger/OpenAPI  

---

## 🗺️ Roadmap (High-Level)

1. **0–3 Bulan** → Setup project, GL module, CI/CD, observability  
2. **3–6 Bulan** → AR & AP integrasi GL  
3. **6–9 Bulan** → Sales & Purchase flow  
4. **9–12 Bulan** → Inventory & Workshop  
5. **12–15 Bulan** → Invoice & approval workflow  
6. **15–18 Bulan** → Optimasi performa + security + Keycloak hardening  
7. **18+ Bulan** → HR & Payroll, BI & Analytics, microservices evolution bila dibutuhkan  

---

## 📌 Catatan

- README ini untuk developer yang kontribusi ke repo.  
- Dokumentasi detail (ADR, integrasi, diagram) tersedia di folder `/docs/`.  
- Untuk setup Keycloak lokal (dev), gunakan `docker-compose` di folder `/docker/`.  

