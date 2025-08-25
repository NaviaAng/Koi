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

