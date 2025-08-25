# 📑 Dokumen Arsitektur Proyek – Koi ERP (Vertical Slice Modular Monolith)

## Bab 1 – Latar Belakang & Tujuan

### 1.1 Kondisi Sistem Saat Ini
Sistem **IBS (Integrated Business System)** yang digunakan saat ini dibangun dengan teknologi **WinForm** dan **MySQL 5.0**.  
Beberapa keterbatasan utama yang ditemukan adalah:  

- **Teknologi Usang**  
  - WinForm tidak lagi relevan untuk aplikasi modern yang membutuhkan fleksibilitas multi-platform (desktop, web, mobile).  
  - MySQL 5.0 sudah tidak mendapat dukungan resmi, memiliki keterbatasan dalam partisi data, indexing modern, dan kemampuan auditing.  

- **Arsitektur Monolitik Tradisional**  
  - Semua modul bercampur dalam satu codebase besar.  
  - Perubahan kecil di satu modul dapat memengaruhi modul lainnya.  
  - Sulit di-scale secara parsial.  

- **Keterbatasan Integrasi**  
  - Tidak ada API standar.  
  - Integrasi antar sistem dilakukan dengan query langsung atau export/import file, yang rawan error.  

- **Tantangan Operasional**  
  - Maintenance tinggi karena codebase legacy.  
  - Dokumentasi teknis minim.  
  - SDM baru kesulitan mempelajari sistem lama.  
  - Membutuhkan resource tambahan untuk mengatasi double entry akibat keterbatasan integrasi dengan aplikasi eksternal (contoh: DMS).  

---

### 1.2 Kebutuhan Modernisasi
Perusahaan membutuhkan sistem yang mampu:  

- **Fleksibel & Modular**  
  - Setiap domain bisnis (Sales, Spareparts, Workshop, Accounting) dipisahkan menjadi modul yang lebih mudah dikelola.  

- **Mendukung Tim Kecil**  
  - Sistem harus bisa dikembangkan oleh tim 2–5 orang, dengan fokus pada efisiensi, reusability, dan menghindari duplikasi pekerjaan.  

- **Mendukung Integrasi Modern**  
  - Harus menyediakan API (REST/GraphQL) dan event streaming untuk integrasi dengan aplikasi lain.  
  - Harus mendukung komunikasi asynchronous (event-driven).  

- **Menggunakan Database Modern**  
  - PostgreSQL dipilih karena mendukung fitur partisi, indexing canggih, JSONB untuk semi-structured data, serta lebih baik untuk workload ERP.  

- **Mendukung Observability**  
  - Logging, tracing, dan monitoring terintegrasi, agar mudah melakukan troubleshooting.  

---

### 1.3 Tujuan Re-engineering
Tujuan utama proyek ini adalah melakukan transformasi IBS menjadi:  

- **Aplikasi Modular Monolith berbasis ASP.NET Core 10**  
  - Memanfaatkan arsitektur **Vertical Slice** agar setiap domain dipisahkan dengan jelas (contoh: Sales, Inventory, Accounting).  
  - Setiap domain berisi *Domain Layer, Application Layer, Infrastructure Layer, dan Api Layer*.  

- **Arsitektur yang Future-Proof**  
  - Modular Monolith ini akan didesain agar mudah diekstrak menjadi **Microservices** apabila dibutuhkan di masa depan, tanpa perlu rewrite total.  
  - Mendukung skalabilitas bertahap sesuai dengan kebutuhan bisnis.  

- **Database Modern (PostgreSQL)**  
  - Migrasi dari MySQL 5.0 ke PostgreSQL dengan penerapan praktik terbaik (schema per modul, partitioning, audit fields).  

- **Integrasi dengan Sistem Legacy & Eksternal**  
  - Menyediakan lapisan integrasi (`Koi.Integration`) untuk berkomunikasi dengan sistem lama maupun eksternal (contoh: DMS).  
  - Menggunakan **Outbox Pattern** untuk event publishing.  

---

### 1.4 Sasaran
Sasaran yang ingin dicapai dari proyek ini meliputi:  

- **Produktivitas & Efisiensi**  
  - Mengurangi kompleksitas dalam pengembangan dengan modul yang lebih terstruktur.  
  - Mengurangi kebutuhan double entry yang saat ini membebani operasional.  

- **Ketahanan & Skalabilitas**  
  - Sistem dapat bertahan dalam jangka panjang (≥10 tahun).  
  - Mampu di-scale sesuai kebutuhan tanpa harus dilakukan redesign besar.  

- **Kontrol & Kepatuhan Data**  
  - Pencatatan transaksi yang konsisten, terutama di domain akuntansi (General Ledger, Account Receivable, Account Payable).  
  - Audit trail yang jelas untuk mendukung keperluan compliance dan regulasi.  

- **Kesiapan Ekspansi**  
  - Sistem yang dapat mendukung pertumbuhan cabang baru, integrasi dengan sistem DMS, serta perluasan layanan digital di masa depan.  


## Bab 2

 – Prinsip Desain & Keputusan Arsitektur

### 2.1 Pendekatan Arsitektur
Arsitektur yang dipilih adalah **Vertical Slice Modular Monolith** dengan prinsip **DDD (Domain-Driven Design)**.  
Pendekatan ini dipilih karena:  

- **Mendukung modularitas**: setiap domain bisnis berdiri sebagai modul mandiri.  
- **Mudah dikelola oleh tim kecil** (2–5 orang).  
- **Future-proof**: modul dapat diekstrak menjadi microservice di masa depan tanpa refactor besar.  
- **Lebih sederhana daripada langsung microservices** yang membutuhkan infra kompleks (service mesh, distributed tracing, devops overhead).  

---

### 2.2 Prinsip Desain
Beberapa prinsip utama yang diterapkan:  

1. **Single Responsibility per Modul (Vertical Slice)**  
   - Setiap domain dipisahkan: contoh `Koi.Janaru.GeneralLedger`, `Koi.Workshop.Service`.  
   - Modul hanya berfokus pada satu area bisnis.  

2. **Separation of Concerns**  
   - Setiap modul terdiri dari:  
     - **Domain** → Aggregate root, entity, value object.  
     - **Application** → CQRS handler, use case, DTO.  
     - **Infrastructure** → Repository, database config.  
     - **Api** → Controller, endpoint, contract.  

3. **Shared Kernel**  
   - Konsep lintas domain seperti `Money`, `BranchId`, `DocumentNumber`, `Period` diletakkan di `Koi.Shared`.  
   - Mengurangi duplikasi kode.  

4. **Building Blocks Teknis**  
   - `Koi.Core` menyimpan mekanisme teknis seperti:  
     - Mediator pipeline (CQRS).  
     - Event bus / Outbox pattern.  
     - Unit of Work & Repository base.  
     - Error handling standar.  

5. **Integration First**  
   - Menyediakan modul khusus `Koi.Integration` untuk komunikasi dengan IBS lama & DMS.  
   - Menggunakan event-driven (Outbox → Message Broker) agar data sinkron antar sistem.  

6. **Database Per Modul (Schema-per-Module)**  
   - PostgreSQL dipilih dengan strategi `schema` per modul (misal: `janaru.gl`, `janaru.ap`, `workshop.service`).  
   - Memberikan isolasi logis tanpa harus memecah DB secara fisik.  

7. **Observability**  
   - Logging terpusat.  
   - Distributed tracing (OpenTelemetry, Jaeger).  
   - Health check & monitoring bawaan.  

---

### 2.3 Keputusan Arsitektur Utama
- **Arsitektur Utama** → Vertical Slice Modular Monolith.  
- **Bahasa & Framework** → C# (.NET 13 / ASP.NET Core 10).  
- **Database** → PostgreSQL (schema per modul).  
- **Integrasi** → Outbox Pattern + Event Bus.  
- **Autentikasi & Otorisasi** → `Koi.Identity` + `Koi.Security` berbasis JWT + RBAC.  
- **Deployment** → Containerized (Docker), dapat diorkestrasi dengan Kubernetes bila perlu.  

---

### 2.4 Alternatif yang Dipertimbangkan
1. **Langsung Microservices**  
   - ❌ Dibatalkan karena tim kecil → overhead DevOps besar.  
   - Cocok untuk tim >20 developer.  

2. **Monolith Tradisional**  
   - ❌ Tidak dipilih karena rawan jadi *big ball of mud*.  
   - Sulit diekstrak menjadi microservices.  

3. **Modular Monolith dengan Vertical Slice**  
   - ✅ Dipilih karena balance antara **kesederhanaan** dan **future readiness**.  
   - Bisa di-deploy sebagai satu aplikasi, tapi tetap modular.  

---

### 2.5 Prinsip Evolusi ke Microservices
Jika di masa depan ada kebutuhan untuk **scalability** atau **team scaling**, maka modul tertentu dapat dipromosikan menjadi microservice, dengan langkah:  

1. Identifikasi modul yang membutuhkan scale khusus (misalnya `Koi.Janaru.GeneralLedger`).  
2. Ekstrak modul menjadi service terpisah.  
3. Hubungkan melalui API / Message Broker.  
4. Tetap mempertahankan Shared Kernel (`Koi.Shared`) sebagai kontrak umum.  



## Bab 3 – Struktur Project & Modul

### 3.1 Struktur Solusi Utama
Struktur solusi `Koi.sln` dibagi menjadi beberapa kategori utama:  

~~~
Koi.sln  
├─ build/ # script build, pipeline, docker, tools  
├─ docker/ # infra dev (Postgres, Jaeger, Seq, ...)  
│ └─ docker-compose.yml  
├─ src/ # source utama  
│ ├─ Koi.Bootstrap/ # host utama ASP.NET Core (composition root)  
│ ├─ Koi.Shared/ # shared kernel (value object, util common)  
│ ├─ Koi.Core/ # building blocks teknis (CQRS, event bus, infra)  
│ ├─ Koi.Integration/ # adapter integrasi ke IBS lama, DMS, dsb  
│ ├─ Koi.Identity/ # modul identity & security  
│ ├─ Koi.Security/ # modul RBAC, policy, permission  
│ ├─ Koi.Janaru/ # cluster modul ERP (Accounting, Finance)  
│ ├─ Koi.Workshop/ # cluster modul Workshop (Service, Body & Paint)  
│ └─ Koi.Sales/ # cluster modul Sales  
└─ tests/ # unit test & integration test
~~~

### 3.2 Struktur Modul Janaru (ERP – Accounting & Finance)
Contoh detail untuk **Koi.Janaru** (ERP cluster) dengan sub-modul:  

```
Koi.Janaru/  
├─ GeneralLedger/ # modul slice General Ledger  
│ ├─ Domain/ # aggregate root: Journal, Account  
│ ├─ Application/ # CQRS handler, DTO  
│ ├─ Infrastructure/ # EF Core config, repository  
│ └─ Api/ # endpoint/controller  
│  
├─ AccountReceivable/ # modul slice AR  
│ ├─ Domain/ # Customer, InvoiceAR, Payment  
│ ├─ Application/  
│ ├─ Infrastructure/  
│ └─ Api/  
│  
├─ AccountPayable/ # modul slice AP  
│ ├─ Domain/ # Vendor, InvoiceAP, Payment  
│ ├─ Application/  
│ ├─ Infrastructure/  
│ └─ Api/  
│  
├─ Invoice/ # modul slice Invoice umum  
│ ├─ Domain/ # Invoice, TaxInvoice  
│ ├─ Application/  
│ ├─ Infrastructure/  
│ └─ Api/  
│  
└─ Purchase/ # modul slice Purchase  
├─ Domain/ # PurchaseOrder, Supplier  
├─ Application/  
├─ Infrastructure/  
└─ Api/
```

### 3.3 Struktur Modul Workshop
Cluster **Koi.Workshop** menggabungkan Service & Body Paint.  

```
Koi.Workshop/  
├─ Service/  
│ ├─ Domain/ # WorkOrder, Mechanic, JobCard  
│ ├─ Application/  
│ ├─ Infrastructure/  
│ └─ Api/  
│  
└─ BodyPaint/  
├─ Domain/ # RepairOrder, Estimation  
├─ Application/  
├─ Infrastructure/  
└─ Api/
```
### 3.4 Struktur Modul Sales
Cluster **Koi.Sales** menangani penjualan unit.  
```
Koi.Sales/  
├─ Lead/  
│ ├─ Domain/ # Prospect, Lead  
│ ├─ Application/  
│ ├─ Infrastructure/  
│ └─ Api/  
│  
└─ Order/  
├─ Domain/ # SalesOrder, DeliveryOrder  
├─ Application/  
├─ Infrastructure/  
└─ Api/
```
### 3.5 Shared & Core
Untuk mengurangi duplikasi kode, digunakan dua layer utama:  

- **Koi.Shared** → berisi *domain primitive* & *shared kernel*  
  - Value Object: `Money`, `Quantity`, `Period`, `BranchId`  
  - Abstraksi umum: `IAggregateRoot`, `EntityBase`, `DomainEvent`  

- **Koi.Core** → berisi *building blocks teknis*  
  - CQRS handler (MediatR/Medator)  
  - Event Bus (Outbox + Message Broker adapter)  
  - Error handling & result wrapper  
  - Base `DbContext` untuk modul  

---

### 3.6 Testing
Folder `tests/` berisi pengujian untuk setiap modul:  
```
tests/  
├─ Koi.Janaru.Tests/  
│ ├─ GeneralLedgerTests.cs  
│ ├─ AccountReceivableTests.cs  
│ └─ ...  
├─ Koi.Workshop.Tests/  
└─ Koi.Sales.Tests/
```
### 3.7 Evolusi ke Microservices
Dengan struktur ini, **setiap folder modul** dapat di-*extract* menjadi microservice terpisah dengan langkah:  

1. Pisahkan `Api/` dari modul menjadi host service sendiri.  
2. Konfigurasi DB schema menjadi database/service khusus.  
3. Komunikasi antar modul dilakukan via Event Bus (Kafka/RabbitMQ).  

## Bab 4 – Alur Data & Integrasi Antar Modul

### 4.1 Prinsip Integrasi
Integrasi antar modul di Koi ERP mengikuti prinsip berikut:

1. **Event-Driven Communication (Domain Event → Integration Event)**  
   - Setiap modul mengeluarkan *Domain Event* internal.  
   - Jika event perlu dikonsumsi oleh modul lain, diubah menjadi *Integration Event* yang dipublish via **Event Bus**.  
   - Contoh: `InvoiceCreated` → dipublish → dikonsumsi oleh `AccountReceivable`.

2. **Loose Coupling**  
   - Modul tidak saling memanggil langsung ke domain lain.  
   - Komunikasi menggunakan:
     - **Event Bus** (async)  
     - **API contract (DTO)** bila sinkron diperlukan.  

3. **Schema-per-Module**  
   - Setiap modul menyimpan data di schema terpisah.  
   - Modul lain tidak boleh query langsung → hanya melalui event/contract.  

---

### 4.2 Alur Data Utama

#### 4.2.1 Purchase → Account Payable → General Ledger
1. `PurchaseOrder` dibuat di modul **Purchase**.  
2. Vendor mengirimkan invoice → modul **AccountPayable** mencatat `VendorInvoice`.  
3. Event `VendorInvoicePosted` dipublish.  
4. Modul **GeneralLedger** menerima event → membuat jurnal otomatis (`Debit Expense`, `Credit AccountPayable`).  

---

#### 4.2.2 Sales → Invoice → Account Receivable → General Ledger
1. `SalesOrder` dibuat di modul **Sales**.  
2. Modul **Invoice** menghasilkan `CustomerInvoice`.  
3. Event `InvoiceIssued` dipublish.  
4. Modul **AccountReceivable** menerima event → mencatat piutang pelanggan.  
5. Modul **GeneralLedger** menerima event → membuat jurnal (`Debit AccountReceivable`, `Credit Revenue`).  

---

#### 4.2.3 Workshop → Invoice → Account Receivable → General Ledger
1. Modul **Workshop.Service** membuat `WorkOrder`.  
2. Setelah pekerjaan selesai → generate `ServiceInvoice`.  
3. Event `ServiceInvoiceCreated` dipublish.  
4. Modul **AccountReceivable** mencatat piutang.  
5. Modul **GeneralLedger** membuat jurnal (`Debit AR`, `Credit WorkshopRevenue`).  

---

#### 4.2.4 Payment Flow (Customer Payment)
1. Customer melakukan pembayaran.  
2. Modul **AccountReceivable** mencatat `CustomerPayment`.  
3. Event `CustomerPaymentApplied` dipublish.  
4. Modul **GeneralLedger** mencatat jurnal (`Debit Cash`, `Credit AR`).  

---

#### 4.2.5 Payment Flow (Vendor Payment)
1. Finance melakukan pembayaran ke vendor.  
2. Modul **AccountPayable** mencatat `VendorPayment`.  
3. Event `VendorPaymentProcessed` dipublish.  
4. Modul **GeneralLedger** mencatat jurnal (`Debit AP`, `Credit Cash/Bank`).  

---

### 4.3 Diagram Integrasi (Teks)
Sales ----> Invoice ----> AccountReceivable ----> GeneralLedger  
Purchase --> AccountPayable -------------------> GeneralLedger  
Workshop --> Invoice ----> AccountReceivable --> GeneralLedger  
|  
v  
AccountPayable <---- Purchase

### 4.4 Integrasi dengan Sistem Eksternal
Selain antar modul internal, Koi ERP juga terhubung ke sistem eksternal:

- **IBS (Legacy ERP Desktop)**  
  - Data tertentu (misalnya transaksi lama, master data) ditarik via `Koi.Integration`.  
  - Outbox pattern memastikan sinkronisasi tidak merusak integritas data.  

- **DMS (Dealer Management System)**  
  - Data pelanggan, kendaraan, dan transaksi sparepart dikonsumsi via API DMS.  
  - Disimpan di staging DB → diolah → masuk ke modul terkait (`Sales`, `Workshop`, `Sparepart`).  

---

### 4.5 Keputusan Integrasi
- **Event Bus sebagai jalur utama** antar modul.  
- **API Contract sinkron** hanya digunakan jika data perlu *query real-time*.  
- **No direct DB access** antar modul.  
- **Staging DB** untuk integrasi dengan sistem eksternal (DMS, IBS).  

---

### 4.6 Contoh Event Contract
```csharp
// Contoh event untuk Invoice
public record InvoiceIssued(
    Guid InvoiceId,
    string InvoiceNumber,
    Guid CustomerId,
    decimal Amount,
    DateTime IssuedDate
);

// Contoh event untuk Payment
public record CustomerPaymentApplied(
    Guid PaymentId,
    Guid InvoiceId,
    decimal Amount,
    DateTime PaidDate
);
```

## Bab 5 – Detail Modul

### 5.1 Modul General Ledger (GL)
**Fungsi Utama:**
- Menjadi *central ledger* seluruh transaksi keuangan.  
- Menerima event dari modul lain (AR, AP, Invoice, Purchase, Sales, Workshop).  
- Menghasilkan laporan keuangan: Neraca, Laba Rugi, Buku Besar.  

**Entitas Utama:**
- `Account` → Chart of Account (COA)  
- `Journal` → Pencatatan debit/kredit  
- `JournalLine` → detail jurnal  

**Contoh Event Flow:**  
- Event `InvoiceIssued` → Jurnal otomatis (`Debit AR`, `Credit Revenue`).  
- Event `VendorPaymentProcessed` → Jurnal otomatis (`Debit AP`, `Credit Cash`).  

---

### 5.2 Modul Account Receivable (AR)
**Fungsi Utama:**
- Mencatat semua piutang pelanggan.  
- Menangani proses penagihan, pembayaran, alokasi invoice.  
- Sinkron dengan modul Invoice & GL.  

**Entitas Utama:**
- `Customer`  
- `CustomerInvoice`  
- `CustomerPayment`  
- `ARBalance`  

**Contoh Event Flow:**  
- Event `InvoiceIssued` → AR mencatat piutang.  
- Event `CustomerPaymentApplied` → Update status invoice & saldo piutang.  
- Event `CustomerWriteOff` → Posting ke GL.  

---

### 5.3 Modul Account Payable (AP)
**Fungsi Utama:**
- Mencatat utang ke vendor.  
- Mengelola pembayaran vendor & aging report.  
- Integrasi dengan modul Purchase & GL.  

**Entitas Utama:**
- `Vendor`  
- `VendorInvoice`  
- `VendorPayment`  
- `APBalance`  

**Contoh Event Flow:**  
- Event `VendorInvoicePosted` → AP mencatat utang.  
- Event `VendorPaymentProcessed` → Update saldo utang + posting GL.  

---

### 5.4 Modul Invoice
**Fungsi Utama:**
- Mengelola invoice baik dari Sales maupun Workshop.  
- Menjadi sumber utama data AR.  
- Terhubung dengan GL untuk pencatatan pendapatan.  

**Entitas Utama:**
- `Invoice` (umum)  
- `TaxInvoice`  
- `InvoiceLine`  

**Contoh Event Flow:**  
- `SalesOrderCompleted` → Generate `InvoiceIssued`.  
- `ServiceWorkOrderClosed` → Generate `ServiceInvoiceCreated`.  

---

### 5.5 Modul Purchase
**Fungsi Utama:**
- Mengelola pembelian barang/jasa dari vendor.  
- Menyediakan data untuk modul AP & Inventory (jika ada).  

**Entitas Utama:**
- `PurchaseOrder`  
- `PurchaseReceipt`  
- `PurchaseInvoice`  

**Contoh Event Flow:**  
- `PurchaseOrderApproved` → Informasi ke vendor.  
- `PurchaseInvoiceReceived` → Event `VendorInvoicePosted`.  

---

### 5.6 Modul Sales
**Fungsi Utama:**
- Mengelola penjualan unit kendaraan.  
- Terhubung dengan modul Invoice & AR.  

**Entitas Utama:**
- `SalesOrder`  
- `DeliveryOrder`  
- `Customer`  

**Contoh Event Flow:**  
- `SalesOrderConfirmed` → Generate Invoice.  
- `DeliveryOrderIssued` → Update status order.  

---

### 5.7 Modul Workshop (Service & Body Paint)
**Fungsi Utama:**
- Mengelola jasa perbaikan kendaraan (service & body paint).  
- Menghasilkan `ServiceInvoice` untuk AR.  

**Entitas Utama:**
- `WorkOrder`  
- `JobCard`  
- `Mechanic`  
- `RepairOrder` (BodyPaint)  

**Contoh Event Flow:**  
- `WorkOrderClosed` → Generate `ServiceInvoiceCreated`.  
- `RepairCompleted` → Generate invoice ke AR.  

---

### 5.8 Modul Shared & Identity
**Koi.Shared**
- Value Object: `Money`, `Period`, `BranchId`, `Quantity`.  
- Domain primitive: `EntityBase`, `IAggregateRoot`, `DomainEvent`.  

**Koi.Identity**
- User Management.  
- Role & Permission.  
- Authentication & Authorization.  

---

### 5.9 Integrasi Antar Modul
```
| Sumber Modul     | Event Dipublish           | Modul Konsumen        | Efek                        |
|------------------|---------------------------|-----------------------|-----------------------------|
| Invoice          | InvoiceIssued             | AR, GL                | Piutang + Jurnal            |
| AR               | CustomerPaymentApplied    | GL                    | Kas Masuk + Update Piutang  |
| AP               | VendorPaymentProcessed    | GL                    | Kas Keluar + Update Utang   |
| Purchase         | VendorInvoicePosted       | AP, GL                | Utang Vendor + Jurnal       |
| Workshop         | ServiceInvoiceCreated     | AR, GL                | Piutang & Pendapatan        |
| Sales            | SalesOrderConfirmed       | Invoice, AR, GL       | Tagihan + Piutang           |

```
## Bab 6 – Deployment & Infrastruktur

### 6.1 Prinsip Deployment
1. **Modular Monolith → Microservices Ready**
   - Seluruh modul dikembangkan dengan *vertical slice pattern*.  
   - Bisa dijalankan dalam satu **Monolith Deployment** (1 host) atau dipecah menjadi beberapa service jika skala membesar.  

2. **Containerized Deployment**
   - Setiap aplikasi utama (API, Worker, Event Bus) dibungkus dengan **Docker container**.  
   - Infrastruktur bisa dijalankan via **Docker Compose** (dev) atau **Kubernetes** (production).  

3. **Isolated Database per Module**
   - Masing-masing modul memiliki schema DB sendiri.  
   - Jika migrasi ke microservices, schema dapat dipisahkan ke database dedicated tanpa refactor besar.  

---

### 6.2 Lingkungan Deployment
- **Development**
  - Tools: Docker Compose (Postgres, Seq, Jaeger, RabbitMQ/Kafka).  
  - Hot reload API untuk mempercepat iterasi.  

- **Staging**
  - Lingkungan mirip production.  
  - Digunakan untuk UAT (User Acceptance Testing).  
  - Data dummy atau *masked* dari production.  

- **Production**
  - Bisa dijalankan sebagai:
    1. **Single Monolith** → lebih mudah dikelola (1 API host, 1 DB server).  
    2. **Split Microservices** → jika load tinggi, contoh:
       - `Koi.Janaru.GeneralLedger` → service terpisah.  
       - `Koi.Janaru.Workshop` → service terpisah.  
    - Deployment diatur oleh **Kubernetes + Helm Chart**.  

---

### 6.3 Infrastruktur Pendukung
1. **Database**
   - PostgreSQL → database utama.  
   - Schema per modul: `janaru_gl`, `janaru_ar`, `janaru_ap`, dll.  

2. **Event Bus**
   - RabbitMQ / Kafka → komunikasi antar modul (publish/subscribe).  
   - Outbox pattern untuk memastikan *reliability*.  

3. **Observability**
   - **Logging**: Seq / ELK Stack.  
   - **Tracing**: Jaeger (OpenTelemetry).  
   - **Metrics**: Prometheus + Grafana.  

4. **CI/CD**
   - GitHub Actions / GitLab CI untuk pipeline:
     - Build → Test → Lint → Package Docker → Deploy (Staging → Production).  

---

### 6.4 Deployment Model
#### 6.4.1 Monolith Deployment (Awal)
```
[ User ] → [ API Gateway / Load Balancer ] → [ Koi.Janaru (Monolith API) ]
→ [ PostgreSQL (Schema per modul) ]
→ [ RabbitMQ / Kafka ]
```

#### 6.4.2 Microservices Deployment (Opsional)
```
[ User ] → [ API Gateway ] → [ Koi.Janaru.Invoice API ]
→ [ Koi.Janaru.AR API ]
→ [ Koi.Janaru.AP API ]
→ [ Koi.Janaru.GL API ]
→ [ Koi.Workshop API ]
→ [ Identity Service ]
→ [ Shared Services ]
→ [ PostgreSQL per modul ]
→ [ RabbitMQ / Kafka ]
```

### 6.5 Skema CI/CD (Contoh Pipeline)
1. **Push Code ke GitHub/GitLab**  
2. **Build & Test**  
   - `dotnet build` & `dotnet test`  
3. **Docker Build & Push**  
   - `docker build -t registry/koi-janaru:tag .`  
   - Push ke private registry  
4. **Deploy ke Staging**  
   - Auto via Docker Compose atau Kubernetes.  
5. **Manual Approval** → Deploy ke Production.  

---

### 6.6 Keputusan Infrastruktur
- **Start Simple (Monolith Deployment)** untuk tim kecil (2-5 dev).  
- **Kubernetes-ready** sejak awal agar bisa naik skala tanpa refactor besar.  
- **Event-driven** untuk memastikan modul tetap loosely-coupled.  
- **Shared observability stack** supaya monitoring & debugging konsisten.  

---
## Bab 7 – Security & Authorization

### 7.1 Prinsip Dasar
1. **Security by Design**  
   - Semua modul wajib mempertimbangkan aspek keamanan sejak tahap desain.  
   - Menghindari *security bolted-on* di akhir pengembangan.  

2. **Authentication & Authorization Terpusat**  
   - Modul `Koi.Identity` berperan sebagai *centralized identity provider*.  
   - Menggunakan standar **OAuth2.0 / OpenID Connect** untuk integrasi.  

3. **Defense in Depth**  
   - Proteksi berlapis: API Gateway, JWT validation, Role-based Access, Logging & Audit.  

---

### 7.2 Authentication
- **User Identity**
  - Disimpan di `Koi.Identity` (username, email, phone, password hash).  
  - Password hash → **Argon2 / PBKDF2**.  
  - Opsi login via **SSO (Google / Microsoft)** untuk enterprise.  

- **Token Based**
  - Menggunakan **JWT (JSON Web Token)**.  
  - Akses modul lain melalui **Bearer Token**.  
  - Token memiliki `scope` (misal: `sales:read`, `invoice:write`).  

- **Refresh Token**
  - Untuk menjaga sesi jangka panjang.  
  - Disimpan aman di Redis atau DB Identity.  

---

### 7.3 Authorization
1. **Role-Based Access Control (RBAC)**
   - Contoh Role:
     - `Admin` → full access  
     - `Finance` → akses AR/AP/GL  
     - `Sales` → akses modul Sales & Invoice  
     - `Mechanic` → akses modul Workshop  

2. **Permission Granularity**
   - Role dipecah menjadi permission kecil:
     - `invoice.read`, `invoice.create`, `gl.postjournal`, dll.  
   - Role adalah *collection of permissions*.  

3. **Contextual Access**
   - Berdasarkan **Branch / Dealer** (`BranchId`).  
   - Contoh: User Medan hanya bisa melihat transaksi branch Medan.  

---

### 7.4 Data Security
- **At Rest**
  - Data sensitif dienkripsi di database (Postgres + pgcrypto).  
  - Backup terenkripsi.  

- **In Transit**
  - Semua komunikasi antar modul via **HTTPS/TLS**.  
  - Event bus (RabbitMQ/Kafka) menggunakan TLS + SASL authentication.  

- **Audit Trail**
  - Semua perubahan data kritis (GL posting, Invoice issued, Payment processed) dicatat.  
  - Audit log disimpan di modul khusus (`Koi.Audit`).  

---

### 7.5 Multi-Tenancy & Segregasi Data
- **Single Database, Multi-Schema** (default)  
  - Schema per tenant (`tenant1_janaru_gl`, `tenant2_janaru_gl`).  

- **Alternative: Database per Tenant**  
  - Untuk skala besar atau kebutuhan compliance.  

- **Tenant Isolation**
  - Token JWT menyimpan `TenantId`.  
  - Semua query harus memiliki filter `TenantId`.  

---

### 7.6 Security Governance
1. **Security Policy**
   - Minimal password strength.  
   - Rotation policy untuk admin password.  
   - 2FA (Two Factor Authentication) untuk akses kritis.  

2. **Vulnerability Management**
   - Dependency scan otomatis (GitHub Dependabot / Snyk).  
   - Penetration test tahunan.  

3. **Incident Response**
   - Alert dari logging (Seq/ELK).  
   - Notifikasi via Slack/Telegram untuk activity mencurigakan.  

---

### 7.7 Contoh Flow
**Contoh: User Finance melakukan posting jurnal**
1. User login → token JWT dengan `role=Finance`.  
2. User akses endpoint `/api/gl/post-journal`.  
3. Middleware cek JWT:
   - Valid token  
   - Role memiliki permission `gl.postjournal`  
   - Branch sesuai dengan `BranchId` user  
4. Aksi berhasil, event `JournalPosted` dipublish ke GL & modul lain.  
5. Audit log dicatat: user, waktu, aksi, data yang berubah.  

---

## Bab 8 – Data Management & Integration

### 8.1 Prinsip Umum
1. **Single Source of Truth (SSOT)**  
   - Setiap modul bertanggung jawab penuh atas data domain-nya sendiri.  
   - Contoh: Data jurnal → `GeneralLedger`; Data invoice → `Invoice`.  

2. **Loose Coupling via Integration**  
   - Modul tidak langsung query ke modul lain.  
   - Interaksi menggunakan **event bus** (asynchronous) atau **API contracts** (synchronous).  

3. **Data Staging Layer**
   - Untuk integrasi dengan sistem eksternal (misalnya DMS ↔ IBS).  
   - Data dari eksternal masuk ke **Koi.Staging** sebelum diproses ke modul domain.  

---

### 8.2 Database Strategy
- **RDBMS**: PostgreSQL sebagai primary database.  
- **Schema-per-Module**  
  - `janaru_gl`, `janaru_ap`, `janaru_ar`, `janaru_sales`, dll.  
  - Mengurangi risiko konflik antar modul.  
- **Migration Management**
  - Setiap modul punya script migrasi sendiri (pakai `EF Core Migration`).  
  - Versioning via folder `Migrations/`.  

---

### 8.3 Integrasi Antar Modul Internal
1. **Event-Driven (Asynchronous)**
   - Modul publish event domain penting ke Event Bus.  
   - Contoh:
     - `InvoiceIssued` → dikonsumsi oleh `AccountReceivable`.  
     - `PaymentReceived` → dikonsumsi oleh `GeneralLedger`.  

2. **API Contract (Synchronous)**
   - Dipakai jika perlu query cepat (misalnya untuk UI).  
   - Contoh: Modul `Sales` query ke modul `Inventory` untuk ketersediaan stok.  

---

### 8.4 Integrasi Eksternal
1. **Dengan DMS (Dealer Management System)**
   - Data customer, service order, spare part transfer, vehicle stock → masuk via `Koi.Staging`.  
   - Setelah validasi, data diproses masuk ke modul terkait (`Sales`, `Workshop`, `Inventory`).  

2. **Dengan IBS (In-house ERP lama)**
   - Transisi bertahap: IBS menjadi sumber data historis.  
   - Data baru diinput via Koi ERP, IBS hanya sinkronisasi (read-only).  

3. **ETL (Extract-Transform-Load)**
   - Proses batch malam hari (cron job).  
   - Data dari DMS → staging → modul ERP.  
   - Transformasi pakai `Koi.Integration` service.  

---

### 8.5 Data Governance
- **Data Ownership**
  - Modul domain memiliki otoritas penuh atas data-nya.  
  - Tidak boleh ada cross-module table join langsung.  

- **Data Quality**
  - Validasi sebelum masuk ke staging.  
  - Duplicate check untuk customer, invoice, dan transaksi keuangan.  

- **Data Lineage**
  - Semua data dari DMS/IBS diberi metadata asal (`SourceSystem`, `ImportBatchId`).  

---

### 8.6 Master Data Management (MDM)
1. **Customer Master** → dikelola oleh `Sales` & `AR`.  
2. **Vendor Master** → dikelola oleh `AP` & `Purchase`.  
3. **Product / Spare Part Master** → dikelola oleh `Inventory`.  
4. **Chart of Account (COA)** → dikelola oleh `GeneralLedger`.  

---

### 8.7 Data Security
- **Encryption**
  - Data sensitif (NIK, rekening, gaji, harga beli) dienkripsi di database.  
- **Masking**
  - Untuk data non-prod (staging/testing), field sensitif dimasking.  
- **Audit**
  - Setiap perubahan data kritis dicatat di modul `Koi.Audit`.  

---

### 8.8 Contoh Flow Integrasi
**Contoh: Sales → Invoice → AR → GL**
1. Sales modul buat Sales Order → publish event `SalesOrderConfirmed`.  
2. Invoice modul terima event → buat `InvoiceIssued`.  
3. AR modul terima event → catat piutang.  
4. GL modul terima event → buat jurnal debit/kredit.  
5. Semua transaksi memiliki `TransactionId` untuk trace ke seluruh modul.  

---

### 8.9 Tooling
- **Integration Framework**: MassTransit / NServiceBus.  
- **Scheduler**: Hangfire / Quartz.NET untuk ETL.  
- **Data Validation**: FluentValidation untuk batch import.  



## Bab 9 – Testing & Quality Assurance

### 9.1 Prinsip Umum
1. **Shift-Left Testing**  
   - Testing dilakukan sejak tahap awal pengembangan, bukan hanya di akhir.  
   - Unit test wajib tersedia untuk setiap handler/aggregate penting.  

2. **Automation First**  
   - Automasi testing (unit, integration, end-to-end) sebisa mungkin mengurangi dependency pada manual testing.  

3. **Continuous Testing**  
   - Testing menjadi bagian dari pipeline CI/CD.  
   - Build tidak boleh lolos jika test gagal.  

---

### 9.2 Jenis Testing
1. **Unit Testing**
   - Fokus pada kelas/domain logic.  
   - Contoh:  
     - `Journal.Post()` menghasilkan debit=credit.  
     - `Invoice.CalculateTotal()` menghitung PPN dengan benar.  
   - Framework: **xUnit / NUnit**.  

2. **Integration Testing**
   - Menguji interaksi antar modul (misal: Invoice → AR).  
   - Menggunakan database in-memory / testcontainer (Docker PostgreSQL).  
   - Contoh: API `POST /invoice` menghasilkan record di Invoice + Event ter-publish.  

3. **Contract Testing**
   - Menggunakan **Pact** untuk memastikan API antar modul konsisten.  
   - Berguna saat modul dipecah menjadi microservices.  

4. **End-to-End Testing (E2E)**
   - Menggunakan **Playwright / Selenium** untuk simulasi UI.  
   - Contoh: Buat Sales Order → Invoice → Payment → cek hasil di GL.  

5. **Performance Testing**
   - Uji beban modul kritis (Invoice, GL Posting).  
   - Tools: JMeter / k6.  

6. **Security Testing**
   - Penetration test (OWASP Top 10).  
   - Cek token expiration, SQL Injection, XSS.  

---

### 9.3 Quality Gates
- **Code Coverage**
  - Minimal 70% untuk business logic (Domain + Application).  
- **Static Code Analysis**
  - Menggunakan SonarQube / Roslyn Analyzer.  
- **Linting & Style**
  - Menggunakan EditorConfig + StyleCop.  
- **Pull Request Policy**
  - PR wajib lolos build + test sebelum merge.  

---

### 9.4 CI/CD Pipeline Integration
- **Pipeline Step**
  1. Build & Restore dependencies.  
  2. Run Unit Tests.  
  3. Run Integration Tests (dengan TestContainers).  
  4. Static Code Analysis.  
  5. Deploy ke environment Staging (jika lolos).  
  6. Run E2E Tests di Staging.  
  7. Approval → Deploy ke Production.  

- **Branching Strategy**
  - `main` → stable production  
  - `develop` → active development  
  - `feature/*` → per modul/fitur  

---

### 9.5 User Acceptance Testing (UAT)
- **Scope**
  - Dilakukan oleh user bisnis (Finance, Sales, Workshop).  
  - Fokus pada validasi proses bisnis (bukan hanya bug teknis).  

- **Format UAT**
  1. Skenario bisnis ditulis di test case (contoh: "Posting Jurnal Ganda").  
  2. User menjalankan skenario.  
  3. Hasil: `Passed` / `Failed`.  

- **Tools**
  - Bisa menggunakan Azure DevOps Test Plan / TestRail.  

---

### 9.6 Regression Testing
- **Automated Regression Suite**
  - Setiap kali ada perubahan, regression test otomatis jalan.  
  - Fokus pada modul sensitif: GL, Invoice, AR/AP.  

- **Snapshot Testing**
  - Berguna untuk output laporan (PDF/Excel).  
  - Snapshot disimpan dan dibandingkan tiap build.  

---

### 9.7 QA Governance
1. **Definition of Done (DoD)**
   - Fitur dianggap selesai jika:
     - Unit test & integration test lulus.  
     - Code review selesai.  
     - Dokumentasi API tersedia.  
     - UAT dinyatakan `Passed`.  

2. **Bug Lifecycle**
   - Bug dilaporkan → triage → assign → perbaikan → retest → close.  

3. **Test Data Management**
   - Data dummy di DB Test/Staging.  
   - Tidak boleh pakai data real customer.  

---

### 9.8 Contoh Test Flow
**Contoh: Invoice → AR → GL**
1. Developer push code baru `InvoiceIssuedHandler`.  
2. Pipeline jalan:
   - Unit test jalan → lolos.  
   - Integration test buat invoice → cek event AR terima.  
   - Regression test cek GL posting tetap balance.  
3. QA deploy ke Staging → User Finance jalankan UAT.  
4. Jika semua `Passed` → merge ke `main` → deploy ke Production.  



## Bab 10 – Deployment & Operations

### 10.1 Prinsip Umum
1. **Automated Deployment**  
   - Semua deployment dilakukan melalui pipeline CI/CD, tidak manual.  

2. **Immutable Deployment**  
   - Aplikasi dideploy sebagai image/container yang konsisten di semua environment.  

3. **Infrastructure as Code (IaC)**  
   - Konfigurasi server, database, dan network dikelola dengan Terraform/Ansible.  

---

### 10.2 Environment Strategy
- **Development (DEV)**
  - Untuk developer.  
  - Sering direset.  
  - Menggunakan test data dummy.  

- **Staging (UAT)**
  - Mirror dari Production.  
  - Digunakan untuk UAT & pre-release testing.  

- **Production (PROD)**
  - High availability, monitoring, backup aktif.  

---

### 10.3 Deployment Options
1. **On-Premise**
   - Server cluster di data center DIPO.  
   - Pakai Docker Compose / Kubernetes.  

2. **Cloud (Hybrid)**
   - Modul non-kritis di cloud (Azure/AWS).  
   - Modul kritis (Accounting/GL) tetap on-prem untuk kontrol penuh.  

3. **Containerization**
   - Semua modul (`Koi.Janaru.*`) dibungkus menjadi **Docker Image**.  
   - Deployment orchestration via **Kubernetes / AKS / EKS**.  

---

### 10.4 Deployment Pipeline
1. **Build**
   - Compile & restore dependencies.  
   - Generate Docker image.  

2. **Test**
   - Run unit + integration test.  
   - Run security scan (Trivy, Snyk).  

3. **Deploy to Staging**
   - Apply DB migrations.  
   - Deploy container ke Kubernetes namespace `staging`.  

4. **Smoke Test**
   - Cek endpoint health.  
   - Pastikan API responsif.  

5. **Deploy to Production**
   - Manual approval.  
   - Blue/Green deployment → tidak ada downtime.  
   - Rollback otomatis jika gagal.  

---

### 10.5 Configuration Management
- **Environment Variables**
  - Connection string, API key, secrets.  
- **Secret Management**
  - Menggunakan Vault / Azure KeyVault.  
- **Feature Flag**
  - New feature bisa diaktifkan/disable tanpa redeploy.  

---

### 10.6 Monitoring & Observability
1. **Logging**
   - Menggunakan **Serilog + Seq**.  
   - Log structured JSON, mudah ditelusuri.  

2. **Tracing**
   - Distributed tracing dengan **Jaeger / OpenTelemetry**.  
   - Bisa track request dari Sales → Invoice → AR → GL.  

3. **Metrics**
   - Prometheus + Grafana dashboard.  
   - KPI: request/second, error rate, latency, DB performance.  

4. **Alerts**
   - Alert ke MS Teams / Telegram jika:
     - Error rate > 5%.  
     - Latency > 2s.  
     - DB connection pool penuh.  

---

### 10.7 Backup & Recovery
- **Database**
  - Daily full backup.  
  - Transaction log backup tiap 15 menit.  
- **Application**
  - Image versioning di container registry.  
- **Recovery Test**
  - Simulasi restore tiap bulan untuk memastikan prosedur valid.  

---

### 10.8 Security in Operations
- **Zero Trust Access**
  - Hanya akses lewat VPN / bastion host.  
- **RBAC (Role-Based Access Control)**
  - Ops hanya punya hak terbatas (misal: restart service, tidak bisa akses DB langsung).  
- **Audit Trail**
  - Semua deployment & konfigurasi dicatat.  

---

### 10.9 Contoh Workflow
1. Developer merge fitur baru `InvoiceDiscount`.  
2. Pipeline jalan → build → test → deploy ke Staging.  
3. QA jalankan UAT → hasil `Passed`.  
4. Ops approve → deploy ke Production dengan Blue/Green.  
5. Monitoring aktif → alert jika error meningkat.  
6. Jika gagal → rollback otomatis ke versi sebelumnya.  

---

### 10.10 Tools
- **CI/CD** → Azure DevOps, GitHub Actions, GitLab CI.  
- **Container Registry** → Azure ACR / AWS ECR / Docker Hub Private.  
- **Orchestration** → Kubernetes (AKS/EKS) atau Docker Swarm (untuk sederhana).  
- **Monitoring** → Grafana + Prometheus + Seq + Jaeger.  



## Bab 11 – Security & Compliance

### 11.1 Prinsip Umum Keamanan
1. **Defense in Depth**
   - Keamanan diterapkan berlapis: aplikasi → API → database → network → infrastructure.  
2. **Least Privilege**
   - User dan service hanya diberi akses minimum sesuai kebutuhan.  
3. **Zero Trust**
   - Jangan percaya langsung pada internal network. Semua akses diverifikasi.  
4. **Security by Design**
   - Aspek keamanan dipertimbangkan sejak awal desain, bukan ditambahkan belakangan.  

---

### 11.2 Authentication & Authorization
1. **Authentication**
   - Menggunakan **OAuth 2.0 / OpenID Connect (OIDC)**.  
   - **SSO via Keycloak (default) atau Azure AD (opsional)**. 
   - Token berbasis JWT dengan expiry pendek.  

2. **Authorization**
   - RBAC → role Finance, Sales, HR, Workshop.
   - ABAC → granular berdasarkan tenant/cabang.
   - Role & policy dikelola di Keycloak realm.

3. **Service-to-Service Authentication**
   - Antar modul (misalnya Invoice → AR → GL) menggunakan **mTLS** atau signed token.  
   - Antar modul → mTLS atau signed service token dari Keycloak.

---

### 11.3 Data Security
1. **Encryption**
   - Data at-rest → Enkripsi dengan AES-256 (database & file storage).  
   - Data in-transit → HTTPS (TLS 1.2/1.3).  

2. **Database Security**
   - Setiap schema punya role spesifik.  
   - Tidak ada akses `sa`/`superuser` di Production.  
   - Query parameterized → hindari SQL Injection.  

3. **Audit Log**
   - Semua perubahan data sensitif (Invoice, GL, AR/AP) dicatat.  
   - Log berisi: siapa, kapan, perubahan apa.  

---

### 11.4 Secure Development Lifecycle (SDL)
1. **Code Review**
   - Wajib untuk semua pull request.  
   - Fokus pada security flaw (SQL injection, XSS, hardcoded secrets).  

2. **Static Application Security Testing (SAST)**
   - Analisis kode dengan SonarQube, Checkmarx.  

3. **Dynamic Application Security Testing (DAST)**
   - Simulasi serangan runtime dengan OWASP ZAP.  

4. **Dependency Scanning**
   - Scan library pihak ketiga dengan tools seperti **Snyk** atau Dependabot.  

---

### 11.5 Regulatory & Compliance
1. **GDPR / PDPA**
   - Data pribadi (customer, karyawan) dilindungi.  
   - Hak akses, hak untuk dihapus, hak portabilitas data.  

2. **SOX / Financial Audit**
   - Modul Accounting (GL, AR/AP, Invoice) harus mendukung audit trail.  
   - Tidak boleh ada perubahan transaksi tanpa jejak audit.  

3. **ISO 27001**
   - Keamanan informasi → policy & prosedur IT harus konsisten.  

4. **Internal Policy**
   - Password policy (minimal 12 karakter, MFA wajib).  
   - Session timeout otomatis setelah idle 15 menit.  

---

### 11.6 Incident Response
1. **Detection**
   - Monitoring log & alert jika ada pola mencurigakan (misal login gagal berulang).  

2. **Response**
   - Tim security melakukan triage → containment → recovery.  

3. **Post-Mortem**
   - Investigasi penyebab.  
   - Update playbook agar insiden serupa tidak terulang.  

---

### 11.7 Security Testing
1. **Penetration Test**
   - Dilakukan secara berkala (setidaknya 2x setahun).  
   - Fokus: modul finansial & API eksternal.  

2. **Red Team Simulation**
   - Uji serangan internal untuk mengukur kesiapan tim.  

3. **Bug Bounty Program**
   - Insentif bagi external tester yang menemukan celah keamanan.  

---

### 11.8 Contoh Implementasi

- Semua perubahan transaksi → masuk audit log + immutable record di GL.
- Login user → redirect ke **Keycloak SSO** (utama) / SSO (Azure AD)
- Token JWT berisi role `Finance.Admin` → akses API `Invoice`.  
-  User `HR.Manager` hanya bisa approve payroll di modul HR.
-   Semua sesi aktif dapat dimonitor & diputus lewat admin console Keycloak.


## Bab 12 – Data Management & Migration

### 12.1 Prinsip Umum
1. **Single Source of Truth (SSOT)**
   - Setiap domain memiliki sumber data utama yang konsisten (misalnya: General Ledger → sumber resmi laporan keuangan).  

2. **Separation of Concerns**
   - Data operasional (transactional) dipisahkan dari data analitik (reporting/BI).  

3. **Data Quality First**
   - Validasi integritas data wajib sebelum migrasi.  

4. **Auditability**
   - Semua migrasi tercatat, dengan log siapa, kapan, dan data apa yang dimigrasi.  

---

### 12.2 Data Architecture
1. **Operational Database**
   - PostgreSQL / SQL Server untuk transaksi harian (GL, AR, AP, Invoice, Inventory, Sales, Purchase).  

2. **Staging Area**
   - Database perantara untuk impor data dari DMS/IBS.  
   - Hanya data valid yang masuk ke core ERP.  

3. **Data Warehouse (DW)**
   - Untuk reporting & BI.  
   - Disinkronisasi via ETL (Extract, Transform, Load).  

4. **Master Data Management (MDM)**
   - Entitas kunci: Customer, Vendor, Item, Account.  
   - Dijaga konsistensinya di semua modul.  

---

### 12.3 Data Flow
- **Contoh Alur (Sales → Invoice → AR → GL):**
  1. Data transaksi **Sales Order** dibuat di modul Sales.  
  2. Setelah barang terkirim → dibuat **Invoice**.  
  3. Invoice menghasilkan **Account Receivable**.  
  4. Saat pembayaran diterima → jurnal masuk ke **General Ledger**.  

- **Integrasi dengan DMS/IBS**
  - Data Service, Sparepart, Body & Paint → masuk ke **Staging DB**.  
  - Dari staging → divalidasi → masuk ke modul ERP (Invoice, GL, dsb).  

---

### 12.4 Data Migration Strategy
1. **Assessment**
   - Identifikasi tabel & field di sistem lama (IBS, DMS).  
   - Mapping ke struktur baru di Koi ERP.  

2. **Data Cleansing**
   - Hapus duplikat (misalnya customer ganda).  
   - Normalisasi data (alamat, nomor telepon, kode akun).  

3. **Migration Run**
   - Jalankan migrasi per batch (misal per branch / per periode akuntansi).  
   - Gunakan skrip ETL otomatis.  

4. **Validation**
   - Bandingkan balance sheet, trial balance, aging report antara sistem lama dan baru.  

5. **Cut-over Strategy**
   - Big Bang → langsung switch ke ERP baru (risiko tinggi, cepat).  
   - Parallel Run → sistem lama & baru berjalan bersama untuk periode tertentu (aman, tapi mahal).  

---

### 12.5 Data Governance
1. **Ownership**
   - Finance Department → bertanggung jawab atas data akuntansi.  
   - HR Department → bertanggung jawab atas data karyawan.  
   - IT Department → bertanggung jawab atas infrastruktur & keamanan data.  

2. **Data Stewardship**
   - Setiap entitas kunci punya **Data Steward** yang menjaga kualitas data.  

3. **Retention Policy**
   - Transaksi finansial: disimpan minimal 10 tahun (sesuai regulasi).  
   - Log aktivitas: disimpan 1–2 tahun.  

---

### 12.6 Tools & Technology
- **ETL**
  - Apache NiFi, Talend, SSIS, atau skrip Python custom.  
- **Migration Scripts**
  - SQL scripts + version control (Flyway / Liquibase).  
- **Validation**
  - Laporan perbandingan (trial balance, aging AR/AP).  

---

### 12.7 Risiko & Mitigasi
| Risiko | Dampak | Mitigasi |
|--------|---------|----------|
| Data duplikat (customer/vendor) | Laporan tidak akurat | Data cleansing + master data governance |
| Mapping field salah | Transaksi tidak balance | Double validation + UAT finance |
| Downtime saat migrasi | Operasional terganggu | Gunakan parallel run / migrasi bertahap |
| Data sensitif bocor | Pelanggaran compliance | Gunakan enkripsi & masking selama migrasi |

---

### 12.8 Contoh Mapping Field (Invoice → GL)
| Field (IBS)         | Field (ERP)          | Transformation Rule            |
|----------------------|----------------------|--------------------------------|
| `INV_NO`            | `InvoiceNumber`      | Trim + Uppercase               |
| `INV_DATE`          | `InvoiceDate`        | Format ke ISO (YYYY-MM-DD)     |
| `CUSTOMER_CODE`     | `CustomerId`         | Mapping via Master Data        |
| `TOTAL_AMOUNT`      | `Amount`             | Konversi ke currency standar   |
| `GL_ACCOUNT`        | `DebitAccountId`     | Mapping ke Chart of Account    |

---

### 12.9 Contoh Cut-over Timeline
1. **T-30 Hari** → Data cleansing & master data finalisasi.  
2. **T-7 Hari** → Dry-run migrasi & verifikasi laporan.  
3. **T-1 Hari** → Freeze data transaksi baru di sistem lama.  
4. **T-0 (Go-Live)** → Migrasi final, switch ke Koi ERP.  
5. **T+7 Hari** → Monitoring, validasi ulang balance.  

---

## Bab 13 – Testing Strategy

### 13.1 Tujuan Pengujian
- Memastikan kualitas kode dan modul sebelum rilis.  
- Menjamin integrasi antar modul berjalan konsisten (misalnya Sales → Invoice → AR → GL).  
- Mengurangi risiko bug kritikal di produksi.  
- Mendukung Continuous Integration (CI) & Continuous Delivery (CD).  

---

### 13.2 Jenis Pengujian
1. **Unit Testing**
   - Menguji logika kecil (function, class, handler).
   - Contoh: validasi jurnal harus balance (debit = kredit).  

2. **Integration Testing**
   - Menguji interaksi antar modul.
   - Contoh: Sales Order menghasilkan Invoice, lalu posting ke GL.  

3. **API Testing**
   - Menguji endpoint REST/GraphQL.
   - Tools: Postman, Newman, RestSharp.  

4. **End-to-End (E2E) Testing**
   - Menguji alur bisnis dari awal hingga akhir.
   - Contoh: Buat Sales Order → Invoice → Payment → GL Report.  

5. **Performance Testing**
   - Load Test → uji respon sistem untuk 100–500 user concurrent.  
   - Stress Test → uji titik maksimal sistem.  
   - Tools: JMeter, k6.  

6. **User Acceptance Testing (UAT)**
   - Dilakukan oleh user bisnis (Finance, Inventory, HR).  
   - Fokus pada kesesuaian proses dengan kebutuhan operasional.  

---

### 13.3 Testing Scope per Modul
- **General Ledger (GL)**
  - Validasi posting jurnal debit = kredit.
  - Laporan trial balance sesuai data transaksi.  

- **Account Receivable (AR)**
  - Invoice menambah saldo piutang.  
  - Pembayaran mengurangi saldo piutang.  

- **Account Payable (AP)**
  - Purchase Invoice menambah saldo hutang.  
  - Pembayaran mengurangi saldo hutang.  

- **Inventory**
  - Penerimaan barang menambah stok.  
  - Penjualan/pemakaian barang mengurangi stok.  

- **Sales**
  - Sales Order → Invoice ter-generate otomatis.  
  - Diskon & pajak dihitung sesuai aturan.  

---

### 13.4 Testing Automation
1. **Framework**
   - xUnit / NUnit → Unit & Integration Test.  
   - SpecFlow (BDD) → E2E test berbasis scenario.  
   - Cypress / Playwright → UI test.  

2. **Pipeline CI/CD**
   - GitHub Actions / Azure DevOps:  
     - Run unit test setiap commit.  
     - Run integration & API test di branch staging.  
     - Run E2E test di environment QA.  

3. **Coverage**
   - Target 70%+ unit test coverage.  
   - Critical business flow (GL posting, Invoice, Payment) harus 100%.  

---

### 13.5 Test Data Management
- **Test Database**
  - Gunakan PostgreSQL container untuk test environment.  
  - Reset DB sebelum setiap test run.  

- **Data Factory**
  - Buat data dummy otomatis (customer, vendor, item).  
  - Gunakan faker library untuk data random.  

- **Sensitive Data**
  - Jangan gunakan data asli customer/vendor untuk testing.  
  - Jika perlu, gunakan data masking.  

---

### 13.6 Reporting & Monitoring
- **Test Report**
  - Hasil unit & integration test → tersimpan di CI pipeline.  
  - E2E test → laporan HTML dengan screenshot error.  

- **Defect Tracking**
  - Semua bug dicatat di Jira/Azure Boards.  
  - Setiap bug diberi prioritas (Critical, Major, Minor).  

- **Regression Testing**
  - Semua test otomatis dijalankan ulang sebelum release.  

---

### 13.7 Risiko & Mitigasi

| Risiko | Dampak | Mitigasi |
|--------|---------|----------|
| Kurang coverage unit test | Bug muncul di production | Wajib minimal 70% coverage |
| API sering berubah | Test gagal terus | Gunakan contract testing (Pact) |
| Data test tidak konsisten | Test tidak reliable | Gunakan seed data & reset DB |
| Performance bottleneck | User tidak bisa akses saat ramai | Load testing rutin sebelum go-live |

---

### 13.8 Contoh Flow CI/CD dengan Testing
1. **Developer push code ke branch feature**  
   → Run unit test otomatis.  

2. **Merge ke branch develop**  
   → Run integration test & API test.  

3. **Deploy ke staging**  
   → Run E2E test & performance test.  

4. **UAT di staging**  
   → User bisnis validasi proses.  

5. **Deploy ke production**  
   → Hanya setelah semua test & UAT lulus.  

## Bab 14 – Deployment & DevOps

### 14.1 Tujuan Deployment & DevOps
- Menyediakan mekanisme **deploy cepat, aman, dan terukur**.  
- Memastikan **konsistensi environment** antara dev, staging, dan production.  
- Mendukung **Continuous Integration (CI)** dan **Continuous Delivery (CD)**.  
- Mempermudah rollback bila terjadi kegagalan deployment.  

---

### 14.2 Environment Strategy
1. **Local Development**
   - Menggunakan Docker Compose (PostgreSQL, Redis, Message Broker).  
   - Developer bisa menjalankan seluruh modul secara lokal.  

2. **Staging**
   - Environment mirip dengan production.  
   - Tempat integrasi & UAT dilakukan.  

3. **Production**
   - High availability (HA) dengan load balancer.  
   - Monitoring, alerting, logging aktif.  

---

### 14.3 Deployment Pipeline (CI/CD)
1. **Continuous Integration (CI)**
   - Trigger setiap commit/pull request.  
   - Steps:
     - Compile & build project.  
     - Run unit & integration test.  
     - Scan security & dependency.  
     - Generate artifacts (Docker image, NuGet package).  

2. **Continuous Delivery (CD)**
   - Deploy otomatis ke staging.  
   - UAT oleh tim bisnis.  
   - Approval manual sebelum rilis ke production.  

3. **Continuous Deployment (opsional)**
   - Jika tim sudah mature, production bisa auto-deploy setelah semua test lolos.  

---

### 14.4 Infrastruktur Deployment
1. **Containerization**
   - Semua modul di-build ke Docker image.  
   - Naming: `registry.company.com/koi-janaru/<module>:<version>`.  

2. **Orchestration**
   - Kubernetes (k8s) untuk production.  
   - Helm chart untuk konsistensi deploy modul.  

3. **Service Mesh (opsional)**
   - Istio/Linkerd untuk observability, mTLS, traffic control antar modul.  

---

### 14.5 Monitoring & Logging
1. **Monitoring**
   - Prometheus → metrics aplikasi & server.  
   - Grafana → visualisasi dashboard.  

2. **Logging**
   - Serilog + ElasticSearch + Kibana (ELK Stack).  
   - Log berisi trace ID untuk debugging antar modul.  

3. **Tracing**
   - OpenTelemetry + Jaeger untuk distributed tracing.  

---

### 14.6 Backup & Disaster Recovery
1. **Database Backup**
   - Full backup harian + incremental backup setiap jam.  
   - Simpan di lokasi terpisah (off-site/cloud).  

2. **Application Backup**
   - Docker images & Helm chart disimpan di private registry.  
   - Config & secret disimpan di Vault.  

3. **Disaster Recovery Plan**
   - RTO (Recovery Time Objective) = 2 jam.  
   - RPO (Recovery Point Objective) = 15 menit.  

---

### 14.7 Deployment Strategy
1. **Blue-Green Deployment**
   - Dua environment (Blue & Green).  
   - Switch traffic ke versi baru setelah validasi.  

2. **Canary Deployment**
   - Sebagian kecil user diarahkan ke versi baru.  
   - Jika stabil, diperluas ke semua user.  

3. **Rolling Update**
   - Deploy bertahap per instance.  
   - Zero downtime.  

---

### 14.8 Security di Pipeline
- Secret & credential disimpan di **Vault/K8s Secret** (bukan di repo).  
- Signing Docker image untuk mencegah manipulasi.  
- Scan vulnerabilities setiap build.  
- Access control berbasis role (Dev hanya bisa ke staging, Ops ke production).  

---

### 14.9 Contoh Pipeline (GitHub Actions)
```yaml
name: CI-CD

on:
  push:
    branches: [ "main", "develop" ]

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '8.0.x'
      - name: Restore dependencies
        run: dotnet restore
      - name: Build
        run: dotnet build --no-restore
      - name: Test
        run: dotnet test --no-build --verbosity normal

  docker:
    runs-on: ubuntu-latest
    needs: build-test
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker Image
        run: docker build -t registry.company.com/koi-janaru:latest .
      - name: Push to Registry
        run: docker push registry.company.com/koi-janaru:latest
```

### 14.10 Rollback Plan

-   Simpan minimal **2 versi terakhir** di registry.
    
-   Gunakan Helm rollback atau `kubectl rollout undo`.
    
-   Data migrasi harus bisa di-_rollback_ (misalnya via migration script reversible).

---

## Bab 15 – Governance & Best Practices

### 15.1 Tujuan Governance
- Menjamin kualitas kode konsisten di seluruh tim.  
- Meminimalisir risiko teknis jangka panjang (technical debt).  
- Memastikan setiap developer mengikuti pola arsitektur & standar yang telah ditentukan.  
- Mendukung maintainability & scalability jangka panjang.  

---

### 15.2 Coding Standards
1. **Naming Convention**
   - Namespace: `Koi.Janaru.<Module>`  
   - Class: PascalCase → `CustomerService`  
   - Method: PascalCase → `GetCustomerById()`  
   - Variable & field: camelCase → `orderDate`, `_repository`  
   - DTO/Command/Query jelas sesuai CQRS → `CreateInvoiceCommand`, `GetSalesQueryResult`  

2. **Clean Code Principle**
   - SRP (Single Responsibility Principle).  
   - Hindari *God Object* atau *God Service*.  
   - Panjang method maksimal 30 baris.  
   - Unit test wajib untuk domain logic.  

3. **Error Handling**
   - Gunakan `Result<T>` atau `Either<T>` pattern.  
   - Jangan melempar exception tanpa logging.  
   - Semua error yang tampil ke user → friendly message.  

---

### 15.3 Branching Strategy (GitOps Flow)
- **Main** → branch stabil (production).  
- **Develop** → integrasi utama.  
- **Feature/*** → untuk pengembangan fitur spesifik.  
- **Hotfix/*** → perbaikan langsung di production.  

Contoh alur:
1. Developer buat `feature/invoice-discount`.  
2. PR → review → merge ke `develop`.  
3. Setelah UAT lolos, merge ke `main` → auto deploy production.  

---

### 15.4 Code Review Policy
- Semua perubahan harus melalui **Pull Request (PR)**.  
- Minimal 2 reviewer sebelum merge ke `develop`.  
- Checklist review:
  - Kode sesuai naming convention.  
  - Unit test ada dan berjalan.  
  - Tidak ada *hardcode config* / credential.  
  - Dokumentasi update bila ada perubahan API.  

---

### 15.5 Testing Standards
1. **Unit Test**
   - Untuk domain logic.  
   - Coverage minimal 70%.  

2. **Integration Test**
   - Untuk komunikasi antar modul (Sales ↔ Invoice).  

3. **End-to-End Test**
   - Jalankan di staging.  
   - Gunakan Cypress/Playwright untuk UI.  

4. **Performance Test**
   - Minimal 100 TPS untuk modul kritis (Invoice, GL Posting).  

---

### 15.6 Documentation
- **Architecture Decision Record (ADR)** → alasan pemilihan desain.  
- **API Documentation** → OpenAPI/Swagger.  
- **Module Documentation** → setiap modul memiliki README.md.  
- **Change Log** → setiap rilis dicatat di `CHANGELOG.md`.  

---

### 15.7 DevOps Best Practices
- Semua environment dikelola via **Infrastructure as Code (IaC)**.  
- Deployment otomatis, tidak ada manual copy file.  
- Monitoring wajib aktif (Grafana, Prometheus, ELK).  
- Security scanning jadi bagian pipeline CI/CD.  

---

### 15.8 Governance Board
- **Tech Lead** → bertanggung jawab menjaga arsitektur & coding standard.  
- **DevOps Engineer** → memastikan CI/CD konsisten.  
- **Product Owner** → memastikan fitur sesuai kebutuhan bisnis.  
- **Developer** → mengikuti standar & membuat dokumentasi kecil di tiap PR.  

---

### 15.9 Continuous Improvement
- Setiap sprint → lakukan *retrospective*.  
- Evaluasi arsitektur & refactor bila ada bottleneck.  
- Update standar coding bila ditemukan praktik lebih baik.  
- Audit internal tiap kuartal untuk cek konsistensi implementasi.  


## Bab 16 – Roadmap Implementasi

### 16.1 Tujuan Roadmap
- Memberikan panduan strategis untuk implementasi modul Koi ERP.  
- Membagi pekerjaan dalam tahapan realistis agar sesuai kapasitas tim (2–5 orang).  
- Memastikan integrasi berjalan bertahap tanpa mengganggu operasional.  

---

### 16.2 Prinsip Penyusunan Roadmap
1. **Business Value First** → mulai dari modul dengan dampak terbesar bagi operasional.  
2. **Iteratif & Incremental** → rilis kecil, cepat, dan sering.  
3. **Low Coupling** → pilih modul yang relatif mandiri dulu sebelum modul yang saling bergantung.  
4. **Ready for Microservices** → tiap modul dibangun dengan vertical slice agar mudah dipecah kemudian.  

---

### 16.3 Tahap Implementasi

#### Fase 1 – Foundation (0–3 Bulan)
- Setup **Project Structure** (Koi.Janaru + Koi.Shared).  
- Setup CI/CD, monitoring, logging.  
- Implementasi modul **General Ledger (GL)** sebagai core accounting.  
- ADR & dokumentasi standar tim.  

📌 Deliverables:  
- `Koi.Janaru.GeneralLedger` berjalan di staging.  
- Basic chart of account + posting journal.  
- Pipeline build-test-deploy otomatis.  

---

#### Fase 2 – Core Accounting (3–6 Bulan)
- Modul **Accounts Payable (AP)**.  
- Modul **Accounts Receivable (AR)**.  
- Integrasi AP/AR dengan GL.  
- Basic reporting (trial balance, aging report).  

📌 Deliverables:  
- `Koi.Janaru.AccountPayable` + `Koi.Janaru.AccountReceivable`.  
- GL otomatis terupdate dari transaksi AP/AR.  
- Laporan dasar untuk keuangan.  

---

#### Fase 3 – Procurement & Sales (6–9 Bulan)
- Modul **Purchase** (PO, GRN, invoice).  
- Modul **Sales** (SO, delivery, invoice).  
- Integrasi otomatis ke AR/AP/GL.  

📌 Deliverables:  
- `Koi.Janaru.Purchase` + `Koi.Janaru.Sales`.  
- Siklus transaksi end-to-end dari pembelian → pembayaran → posting GL.  
- Siklus transaksi end-to-end dari penjualan → penerimaan → posting GL.  

---

#### Fase 4 – Inventory & Workshop (9–12 Bulan)
- Modul **Inventory** (stock, mutasi, deadstock).  
- Modul **Workshop** (Service + BodyPaint digabung).  
- Integrasi Inventory ↔ Sales & Purchase.  
- Integrasi Workshop ↔ Inventory (spare part usage).  

📌 Deliverables:  
- `Koi.Janaru.Inventory` + `Koi.Workshop`.  
- Laporan ketersediaan stok.  
- Service order dengan konsumsi sparepart otomatis update stock.  

---

#### Fase 5 – Invoice & Extended Features (12–15 Bulan)
- Modul **Invoice** sebagai pusat billing.  
- Integrasi penuh dengan AR/AP.  
- Workflow approval invoice.  

📌 Deliverables:  
- `Koi.Janaru.Invoice`.  
- Invoice terhubung dengan AR/AP.  
- Approval workflow berjalan.  

---

#### Fase 6 – Stabilization & Optimization (15–18 Bulan)
- Optimasi performa query & reporting.  
- Tambah caching layer.  
- Fine-grained security & role-based access.  
- Integrasi monitoring lebih dalam (business KPIs).  

📌 Deliverables:  
- Sistem stabil untuk >500 concurrent users.  
- Dashboard monitoring untuk transaksi & error rate.  
- Role-based access control diterapkan.  

---

### 16.4 Milestone Utama
- **M1**: GL live di staging (3 bulan).  
- **M2**: AP/AR terintegrasi ke GL (6 bulan).  
- **M3**: Sales & Purchase aktif (9 bulan).  
- **M4**: Inventory & Workshop berjalan (12 bulan).  
- **M5**: Invoice & workflow approval (15 bulan).  
- **M6**: Stabil & siap scale (18 bulan).  

---

### 16.5 Risk & Mitigation
- **Risk**: Beban tim kecil (2–5 orang).  
  - **Mitigation**: Fokus pada modul prioritas, gunakan incremental delivery.  
- **Risk**: Kompleksitas integrasi antar modul.  
  - **Mitigation**: Mulai dari core accounting (GL), gunakan vertical slice + event-driven.  
- **Risk**: Resistensi user terhadap sistem baru.  
  - **Mitigation**: UAT di tiap fase, training user sejak awal.  

---

### 16.6 Long-Term Plan (18+ Bulan)
- Pecah modul ke microservices bila load meningkat.  
- Tambahkan modul HR & Payroll (fase berikutnya).  
- Integrasi dengan aplikasi pihak ketiga (DMS, IBS, dsb).  
- Business Intelligence (BI) dengan data warehouse.  

---
## Bab 17 – HR & Payroll Roadmap

### 17.1 Latar Belakang
HR (Human Resource) dan Payroll adalah komponen penting dalam ERP modern.  
Walaupun tahap awal implementasi Koi ERP berfokus pada modul Accounting, Inventory, Sales, Purchase, dan Workshop, HR & Payroll tetap harus ada dalam roadmap jangka panjang agar:
- Manajemen SDM lebih terintegrasi dengan sistem keuangan.  
- Payroll otomatis tercatat di General Ledger.  
- Meminimalisir kesalahan manual dalam penggajian & tunjangan.  

---

### 17.2 Lingkup Modul HR
1. **Master Data Karyawan**
   - Data personal, riwayat pekerjaan, kontrak, status aktif/non-aktif.  
2. **Absensi & Kehadiran**
   - Integrasi mesin fingerprint / aplikasi mobile.  
   - Rekap lembur, cuti, izin, sakit.  
3. **Pengelolaan Cuti**
   - Alokasi cuti tahunan.  
   - Approval workflow.  
4. **Performance Management (Opsional fase awal)**
   - Penilaian kinerja, KPI.  
5. **Training & Development (Opsional fase berikutnya)**

---

### 17.3 Lingkup Modul Payroll
1. **Penggajian**
   - Hitung gaji pokok, tunjangan, potongan.  
   - Siklus bulanan & tahunan.  
2. **Potongan & Pajak**
   - PPh 21 otomatis.  
   - BPJS Kesehatan & Ketenagakerjaan.  
3. **Payroll Cycle**
   - Slip gaji otomatis.  
   - Transfer ke bank via file format standar.  
4. **Integrasi Akuntansi**
   - Posting otomatis ke General Ledger (biaya gaji, kewajiban BPJS, pajak, dll).  

---

### 17.4 Roadmap HR & Payroll

#### Fase 7 – HR Core (18–21 Bulan)
- Implementasi **Master Data Karyawan**.  
- Modul Absensi sederhana (import dari mesin fingerprint).  
- Modul Cuti & approval workflow.  

📌 Deliverables:  
- `Koi.HR` (modul utama).  
- Data karyawan terpusat.  
- Sistem cuti & absensi dasar.  

---

#### Fase 8 – Payroll Core (21–24 Bulan)
- Implementasi **modul Payroll**.  
- Hitung gaji pokok, tunjangan tetap, potongan, BPJS.  
- Slip gaji otomatis (PDF).  
- Integrasi posting ke GL.  

📌 Deliverables:  
- `Koi.Payroll`.  
- Payroll cycle bulanan berjalan otomatis.  
- Posting jurnal otomatis ke GL.  

---

#### Fase 9 – HR Advanced (24–30 Bulan)
- Modul Performance Management (KPI, appraisal).  
- Training & Development tracking.  
- Integrasi penuh dengan data absensi mobile app.  

📌 Deliverables:  
- Evaluasi kinerja karyawan terukur.  
- Training record terdokumentasi.  
- Laporan HR analitik (turnover, absensi, cuti).  

---

### 17.5 Integrasi HR & Payroll
- **HR → Payroll**: Data karyawan, absensi, cuti mempengaruhi perhitungan gaji.  
- **Payroll → GL**: Semua payroll posting otomatis masuk ke General Ledger.  
- **HR → Workshop**: Absensi mekanik → dasar perhitungan insentif.  
- **HR → Project** (opsional): Alokasi tenaga kerja ke proyek tertentu untuk costing.  

---

### 17.6 Risiko & Mitigasi
- **Regulasi Pajak sering berubah**  
  - Mitigasi: Buat modul pajak dengan konfigurasi fleksibel.  
- **Data karyawan sensitif**  
  - Mitigasi: Terapkan enkripsi & akses role-based.  
- **Integrasi absensi hardware**  
  - Mitigasi: Gunakan adapter modul `Koi.Connector.Absensi` agar hardware bisa plug-and-play.  

---

### 17.7 Long-Term Vision
- Self-Service Portal: Karyawan bisa akses slip gaji, cuti, absensi via web/mobile.  
- Integrasi dengan BPJS & pajak via API pemerintah.  
- Payroll multi-entity: mendukung banyak cabang / anak perusahaan.  
- Analytics berbasis AI: prediksi turnover, produktivitas, dan cost per employee.  

---
