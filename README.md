# Koi ERP – Vertical Slice Modular Monolith

Koi ERP adalah modernisasi dari sistem IBS lama menjadi **.NET Modular Monolith** dengan pendekatan **Vertical Slice + Domain-Driven Design (DDD)**.  
Tujuan: modular, scalable, aman, dan siap dievolusi menjadi **microservices** bila dibutuhkan.

---

## 🚀 Tech Stack

- **Backend**:  
  - C# (.NET 13 / ASP.NET Core 10)  
  - CQRS + Vertical Slice + DDD  
- **Database**: PostgreSQL (schema-per-module, EF Core migrations)  
- **Integrasi**: Outbox Pattern + Event Bus (RabbitMQ/Kafka)  
- **Security & IAM**:  
  - **Keycloak** sebagai Identity Provider (OAuth2.0 / OIDC)  
  - JWT + RBAC + contextual access (BranchId, TenantId)  
  - Fine-grained permission di `Koi.Security`  
- **Deployment & Infra**:  
  - Docker, Kubernetes (opsional), IaC (Terraform/Ansible)  
  - CI/CD (GitHub Actions / Azure DevOps)  
- **Observability**:  
  - **.NET Aspire Dashboard** (logs, metrics, traces)  
  - Serilog + Seq, OpenTelemetry + Jaeger, Prometheus + Grafana  

---

## 🔧 Developer Experience (Aspire)

Koi ERP memanfaatkan **.NET Aspire** untuk mempermudah pengembangan dan observability.

- **Aspire Projects**  
  - `Koi.AppHost` → orchestration modul, Postgres, RabbitMQ, Keycloak  
  - `Koi.ServiceDefaults` → shared konfigurasi logging, tracing, metrics  

- **Fitur Utama**  
  - Aspire Dashboard: monitoring logs, traces, metrics antar modul  
  - Service orchestration: jalankan API + DB + broker + IAM sekali klik  
  - Konsistensi konfigurasi: dari dev → staging → prod  

- **Keuntungan**  
  - Setup dev cepat, tidak perlu konfigurasi docker-compose manual  
  - Monitoring built-in sejak tahap development  
  - Memudahkan debugging flow antar modul (Sales → Invoice → AR → GL)  

---

## 🔒 Identity & Access Management (IAM)

- **Provider**: Keycloak  
- **Authentication**:  
  - Semua modul memvalidasi JWT dari Keycloak  
  - Mendukung SSO (Microsoft, Google, LDAP)  
  - Refresh token, session revoke → handled by Keycloak  
- **Authorization**:  
  - RBAC di Keycloak (role global)  
  - Fine-grained permission → `Koi.Security`  
  - Context-based access (TenantId, BranchId)  
- **Integration Flow**:  
  - `Koi.Identity` = adapter + sync service dari Keycloak ke internal DB  
  - Semua request API → middleware JWT → cek role/permission  

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

### Full [Architecture Ducument](https://github.com/NaviaAng/Koi/blob/57a91c127d625ebf6b8d246557450613020eb4db/architecture.md)
### [Cheatsheet Onion Architecture](https://cheatography.com/vikbert/cheat-sheets/onion-architecture-symfony/)




