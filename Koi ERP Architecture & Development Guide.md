# 🏗️ Koi ERP Architecture & Development Guide

## 📌 Overview

**Koi ERP** adalah sistem enterprise modular yang dirancang untuk mengelola berbagai domain bisnis seperti organisasi, produk, penjualan, SDM, dan CRM. Sistem ini dibangun dengan pendekatan **Modular Monolith + Vertical Slice Architecture**, dan siap untuk dipisah menjadi **microservices** jika dibutuhkan.

---

## 🧱 Modular Structure

### 🔹 Solution: `Koi.sln`

```plaintext
Koi.sln
├── Koi.Janaru/
│   ├── OrganizationServices/
│   ├── ProductServices/
├── Koi.CRM/
├── Koi.HR/
│   ├── EmployeeService/
│   ├── PayrollService/
│   ├── AttendanceService/
├── Shared/
└── Tests/
```

### 🔹 Penjelasan Modular

| Service                    | Domain yang Ditangani                          |
|---------------------------|------------------------------------------------|
| `OrganizationServices`    | Company, Site, Location                        |
| `ProductServices`         | Product, ProductType, Category, Class          |
| `CRM`                     | Lead, Contact, Opportunity, Pipeline           |
| `HR.EmployeeService`      | Data karyawan, kontrak, posisi                 |
| `HR.PayrollService`       | Gaji, slip, pajak, tunjangan                   |
| `HR.AttendanceService`    | Absensi, shift, jadwal, integrasi mesin        |

---

## 🧠 Architectural Principles

- **Vertical Slice per Use Case**: Setiap fitur memiliki folder `Create`, `Edit`, `Delete`, `GetById` dengan Command, Handler, Validator, Response.
- **Shared Kernel**: Entity base seperti `BaseEntity`, `IAuditable`, dan `ValueObject` disimpan di `Shared.Kernel`.
- **Shared Contracts**: DTO lintas service disimpan di `Shared.Contracts`, mendukung REST/gRPC.
- **Scalable Deployment**: Setiap service bisa di-deploy mandiri, cocok untuk containerization dan CI/CD.

---

## 🛠️ Development Guide

### 🔹 Teknologi Utama
- ASP.NET Core 10
- MediatR (CQRS)
- FluentValidation
- Entity Framework Core
- AutoMapper
- REST/gRPC (optional)
- GitHub + GitHub Actions (CI/CD)

### 🔹 Struktur per Service


```plaintext
OrganizationService/
├── Features/
│   ├── Company/
│   │   ├── Commands/
│   │   │   ├── CreateCompany/
│   │   │   ├── EditCompany/
│   │   │   ├── DeleteCompany/
│   │   ├── Queries/
│   │   │   ├── GetCompanyById/
│   │   ├── Shared/
│   │   │   ├── CompanyEntity.cs
│   │   │   ├── ICompanyRepository.cs
│   │   └── CompanyController.cs
│   ├── Site/
│   │   ├── Commands/
│   │   ├── Queries/
│   │   ├── Shared/
│   │   └── SiteController.cs
├── Application/
├── Domain/
├── Infrastructure/
├── Shared/
├── WebApi/
└── Tests/
```

```plaintext
ProductService/
├── Features/
│   ├── Product/
│   │   ├── Commands/
│   │   ├── Queries/
│   │   ├── Shared/
│   │   └── ProductController.cs
│   ├── ProductType/
│   ├── Category/
│   ├── Class/
├── Application/
├── Domain/
├── Infrastructure/
├── Shared/
├── WebApi/
└── Tests/
```

### 🔹 Slice Example: `CreateProduct`

```plaintext
CreateProduct/
├── CreateProductCommand.cs
├── CreateProductHandler.cs
├── CreateProductValidator.cs
├── CreateProductResponse.cs
```

---

## 👥 For Management

### ✅ Keuntungan Arsitektur Ini

| Aspek                  | Manfaat Bisnis                                      |
|------------------------|-----------------------------------------------------|
| **Modular & Terpisah** | Mudah dikembangkan dan dipelihara per domain        |
| **Scalable**           | Siap untuk pertumbuhan pengguna dan fitur           |
| **Tim Efisien**        | Developer bisa bekerja paralel tanpa konflik        |
| **Microservice-ready** | Bisa dipisah ke service mandiri saat dibutuhkan     |
| **Maintainable**       | Struktur jelas, mudah onboarding dan dokumentasi    |

### 📈 Potensi Ekspansi

- Tambah service baru: Finance, Inventory, Accounting
- Integrasi eksternal: e-Faktur, e-Payment, HRIS, CRM tools
- Cloud-native deployment: Docker, Kubernetes, Azure/AWS

---

## 📦 Deployment & CI/CD

- Setiap service bisa di-deploy sebagai container
- Gunakan GitHub Actions untuk build, test, dan deploy otomatis
- Gunakan versioning per service untuk tracking perubahan

---

## 📚 Dokumentasi Tambahan

- `README.md` per service → menjelaskan domain dan endpoint
- `ARCHITECTURE.md` → menjelaskan struktur dan prinsip desain
- `CONTRIBUTING.md` → panduan kontribusi untuk developer baru

