
# Koi.sln – Blueprint Modular Monolith (ASP.NET Core 10 + Clean Architecture + DDD/CQRS)

> Target: Re-engineering IBS (Sales, Service, Body & Paint, Spareparts, Accounting/Tax) dari WinForm + MySQL 5.0 ke **ASP.NET Core 10 + PostgreSQL** dengan **Modular Monolith**, siap evolve ke microservices.

---

## 1 – Prinsip Desain & Keputusan Arsitektur

-   **Modular Monolith** dengan boundary domain tegas (DDD): tiap modul memiliki `Domain`, `Application`, `Infrastructure`, `Api`.
-   **CQRS**: Query terpisah dari Command; gunakan **MediatR** (atau in-proc mediator **Cortext.Mediator** ) + pipeline behaviors.
-   **Business Logic di Code** (bukan DB). Stored procedure lama dipindah ke **Domain Services / Application Handlers**.
-   **PostgreSQL Centralized** (recommended): **partitioning by period (monthly)** + **subpartition by branch**. Tidak lagi bikin DB baru per bulan.
-   **Outbox Pattern** untuk integrasi/eventual consistency dan audit.
-   **EF Core** + **Dapper** (hybrid): EF untuk write/rich domain, Dapper untuk heavy read/reporting.
-   **Validation**: FluentValidation.
-   **Mapping**: Mapster (lebih ringan dari AutoMapper).
-   **Observability**: Serilog + OpenTelemetry (trace + metrics).
-   **Security**: JWT + Role/Permission per modul.
-   **Namespace**: konsisten `Koi.*` dan **Janaru** khusus domain accounting sesuai konvensi internal.
    
---

## 2 – Struktur Solution (ringkas)

```
Koi.sln
├─ build/                      # skrip build, templates, paket nuspec
│  └─ Directory.Build.props    # central msbuild props (versi, style rules)
├─ docker/                     # docker-compose infra dev
│  ├─ docker-compose.yml       # postgres, pgadmin, jaeger, seq, redis, rabbitmq
│  └─ init/                    # sql init: roles, extensions
├─ aspire/                     # Aspire app host (compose orchestrator)
│  ├─ AppHost/                 # deklarasi services (sql, seq, redis, mq)
│  └─ Observability/           # tracing, metrics, dashboard
├─ src/
│  ├─ Koi.Bootstrap            # ASP.NET Core host (composition root)
│  │  └─ Program.cs            # integrasi Aspire host & modul Koi
│  ├─ Koi.BuildingBlocks       # shared abstractions & cross-cutting
│  │  ├─ Koi.BuildingBlocks.Application
│  │  ├─ Koi.BuildingBlocks.Domain
│  │  ├─ Koi.BuildingBlocks.Infrastructure
│  │  └─ Koi.BuildingBlocks.Presentation
│
│  ├─ Koi.Identity             # authN/authZ, users, roles, permissions
│  │  ├─ Domain | Api
│  │  ├─ Application
│  │  │  ├─ Services/             # ➕ tambahan sub-modul
│  │  │     ├─ UserManagement/    # CRUD user, reset password, profil
│  │  │     ├─ RoleManagement/    # buat role, assign user→role
│  │  │     ├─ PermissionService/ # izin modul: ex. akses Sparepart
│  │  │     └─ TokenService/      # JWT, refresh token, SSO integration
│  │  ├─ Infrastructure
│  │     └─ ExternalProviders/    # ➕ integrasi OAuth2, AzureAD, Google SSO
│
│  ├─ Koi.Service              # bengkel (perawatan & perbaikan)
│  │  ├─ Domain | Application | Infrastructure | Api
│  ├─ Koi.BodyPaint            # body & cat
│  │  ├─ Domain | Application | Infrastructure | Api
│  ├─ Koi.Spareparts           # parts sales & inventory
│  │  ├─ Domain | Application | Infrastructure | Api
│  ├─ Koi.Sales                # penjualan kendaraan (opsional gelombang 2)
│  │  ├─ Domain | Application | Infrastructure | Api
│  ├─ Koi.Janaru.GeneralLedger # accounting: GL
│  │  ├─ Domain | Application | Infrastructure | Api
│  ├─ Koi.Janaru.AccountPayable
│  │  ├─ Domain | Application | Infrastructure | Api
│  ├─ Koi.Janaru.Tax           # pajak (PPN, PPh)
│  │  ├─ Domain | Application | Infrastructure | Api
│  │
│  ├─ Koi.SharedKernel         # seedwork domain (Result, AggregateRoot, dll.)
│  │  ├─ Domain/
│  │  │  ├─ BaseEntity.cs          # entitas dengan Id, CreatedAt, UpdatedAt
│  │  │  ├─ AggregateRoot.cs       # marker untuk aggregate root (DDD)
│  │  │  ├─ ValueObject.cs         # base class untuk value objects
│  │  │  ├─ Result.cs              # pattern Result<T> (success/failure)
│  │  │  └─ DomainEvent.cs         # event domain base class
│  │  │
│  │  ├─ Events/
│  │  │  ├─ IDomainEvent.cs        # interface event
│  │  │  └─ DomainEventDispatcher.cs # mediator pattern
│  │  │
│  │  ├─ Exceptions/
│  │  │  ├─ DomainException.cs     # custom exception domain
│  │  │  └─ ValidationException.cs # validasi rules
│  │  │
│  │  └─ Specifications/
│  │     └─ ISpecification.cs      # pattern specification (filtering rules)
│
│  ├─ Koi.Integration          # adapters ke DMS, IBS legacy, ETL, messaging
│  │  ├─ Messaging/            # RabbitMQ/Azure Service Bus adapters
│  │  ├─ Replication/          # DataReplicationService (IBS legacy → Koi)
│  │  └─ LegacyAdapters/       # konektor ke IBS lama (via DB/API)
│
│  ├─ Koi.Reporting            # readmodels, projections, BI adapters
│  │  ├─ PowerBI/              # Power BI embedded connectors
│  │  └─ DevExpress/           # DevExpress reports migration
│
│  ├─ Koi.Security                  # modul security tambahan (SSO, RBAC granular, audit trail, compliance)
│  │  ├─ Domain                      # inti business rules
│  │  │  ├─ Entities/                # Role.cs, Permission.cs, AuditLog.cs
│  │  │  ├─ ValueObjects/            # PolicyRule.cs, SensitiveField.cs
│  │  │  ├─ Services/                # PolicyEvaluator.cs, AuditBuilder.cs
│  │  │  └─ Events/                  # RoleAssigned.cs, SuspiciousLoginDetected.cs
│  │  ├─ Application                 # use cases (commands, queries, policies)
│  │  │  ├─ Commands/                # AssignRoleCommand.cs, RevokePermissionCommand.cs
│  │  │  ├─ Queries/                 # GetUserPermissionsQuery.cs, GetAuditLogsQuery.cs
│  │  │  └─ Handlers/                # implementasi CommandHandler & QueryHandler
│  │  ├─ Infrastructure              # implementasi teknis
│  │  │  ├─ Repositories/            # RoleRepository.cs, AuditLogRepository.cs
│  │  │  ├─ EFConfigurations/        # RoleConfiguration.cs, AuditLogConfiguration.cs
│  │  │  ├─ Encryption/              # FieldEncryptionProvider.cs
│  │  │  └─ ChangeTracker/           # EF hook capture perubahan data sensitif
│  │  └─ Api                         # expose endpoint
│  │     ├─ Controllers/             # SecurityController.cs, AuditController.cs
│  │     └─ DTOs/                    # AssignRoleDto.cs, AuditLogDto.cs

│
│  └─ Koi.Monitoring           # observability, metrics, health checks
│     ├─ Logging/              # Serilog/Seq sink
│     ├─ Metrics/              # Prometheus/OpenTelemetry
│     └─ HealthChecks/         # service/db health checks
└─ tests/
   ├─ Koi.UnitTests            # test unit (fokus Domain & Application)
   ├─ Koi.IntegrationTests     # test integrasi modul & infra
   ├─ Koi.ContractTests        # api contracts (e.g., OpenAPI, Verify, Snapshots)
   └─ Koi.ReplicationTests     # tes replikasi data IBS → Koi

```

> Modul bisa diaktif/nonaktif via feature flags.
> Jika nanti dipecah ke microservices, boundary sudah siap.

# 📝 Penjelasan per Bagian

### 🔹 `build/`

-   **Tujuan:** standar build seluruh tim.
-   **Kenapa:** Dev tidak perlu setup manual, semua via props/targets.
-   **Isi:** aturan compiler (`LangVersion`, `Nullable`), analyzer rule, template packaging.
    
----------

### 🔹 `docker/`
-   **Tujuan:** infra stateful dijalankan konsisten di semua mesin dev.
-   **Service:**
    -   `postgres`: database utama, dengan init schema.
    -   `redis`: cache/session.
    -   `jaeger`: tracing.
    -   `seq`: structured logging.
    -   `pgadmin`: akses DB via UI.
        
-   **Kenapa init di Docker:** Postgres butuh roles/schema sekali saja saat bootstrap.

----------

### 🔹 `aspire/`

-   **Tujuan:** orkestrasi app .NET (API modul, worker, reporting).
-   **Kelebihan:** Aspire otomatis inject connection string ke modul, integrasi tracing/logging langsung.
-   **File penting:**
    -   `Koi.AppHost/Program.cs` → definisi modul Koi.*.Api + referensi ke Postgres, Redis, Jaeger, Seq.
    -   `Koi.ServiceDefaults/Extensions.cs` → config observability shared.
        
----------

### 🔹 `src/Koi.Bootstrap/`

-   **Tujuan:** host ASP.NET Core utama.
-   **Kenapa:** jadi composition root → load modul, apply middleware global (auth, tracing, error handler).
    
----------

### 🔹 `src/Koi.BuildingBlocks/`

-   **Tujuan:** shared abstractions agar coding standar antar tim.
-   **Isi:**
    -   `Domain`: Entity, AggregateRoot, ValueObject.
    -   `Application`: interface CQRS, `Result<T>`, pipeline behavior.
    -   `Infrastructure`: base repository, unit of work, migrasi DB.
    -   `Presentation`: response standar, API exception filter.
        
----------

### 🔹 `src/Koi.SharedKernel/`

-   **Tujuan:** seedwork domain yang spesifik IBS tapi dipakai lintas modul.    
-   **Contoh:** `Money`, `BranchId`, `Period`.
-   **Kenapa beda dengan BuildingBlocks:** ini domain-driven, bukan abstraksi teknis.
    
----------

### 🔹 `src/Koi.Identity/`, `src/Koi.Service/`, dll

-   **Struktur sama:** Domain, Application, Infrastructure, Api.
-   **Kenapa:** menjaga konsistensi (DDD modular monolith).
-   **Benefit:** kalau nanti mau pecah microservice → gampang, tinggal angkat modul keluar.
    
----------

### 🔹 `src/Koi.Integration/`

-   **Tujuan:** isolasi integrasi eksternal (DMS API, MySQL legacy, ETL, messaging).
-   **Kenapa dipisah:** biar domain bersih, tidak tergantung infra luar.
    
----------

### 🔹 `src/Koi.Reporting/`

-   **Tujuan:** read model projection + endpoint query untuk BI.
-   **Kenapa dipisah:** pattern CQRS → baca & tulis bisa beda struktur.
    
----------

### 🔹 `tests/`

-   **UnitTests:** uji business logic murni (Domain & Application).
-   **IntegrationTests:** test repo DB, messaging, modul end-to-end.
-   **ContractTests:** pastikan API sesuai kontrak (OpenAPI snapshot, Verify).
    
----------

👉 Jadi struktur ini hybrid:

-   **Docker** handle infra stateful.
-   **Aspire** handle modul .NET.
-   **Modular monolith** dengan konsistensi DDD, tapi scalable kalau nanti mau split microservice.

---

## 3 – Lapisan per Modul 

### 3A - Contoh Koi.Sales
```
src/Koi.Sales/
├─ Domain                                      # Lapisan inti bisnis, murni domain logic (tanpa dependensi luar)
│  ├─ Entities                                 # Entitas utama → punya identity (ID) & lifecycle
│  │  ├─ VehicleStock.cs                       # Representasi stok kendaraan di dealer (tersedia, reserved, dsb)
│  │  ├─ VehicleOrder.cs                       # Pesanan kendaraan (customer, unit, tanggal, status)
│  │  ├─ VehicleSale.cs                        # Penjualan kendaraan (harga, metode bayar, invoice, status paid)
│  │  └─ DeliveryOrder.cs                      # Dokumen DO (pengiriman unit setelah penjualan selesai)
│  ├─ ValueObjects                             # Objek nilai → tidak punya identity, equality by value
│  │  ├─ VIN.cs                                # Vehicle Identification Number (unik, validasi format/length)
│  │  └─ OrderNumber.cs                        # Nomor order (format tertentu, auto-generate)
│  ├─ Repositories                             # Kontrak untuk akses data (abstraksi storage)
│  │  ├─ IVehicleStockRepository.cs            # Kontrak akses/manipulasi stok kendaraan
│  │  └─ IVehicleOrderRepository.cs            # Kontrak akses/manipulasi pesanan kendaraan
│  └─ Events                                   # Domain events → menandakan perubahan signifikan
│     ├─ VehicleOrdered.cs                     # Event saat kendaraan berhasil dipesan
│     ├─ VehicleSold.cs                        # Event saat kendaraan berhasil terjual
│     └─ VehicleDelivered.cs                   # Event saat kendaraan berhasil dikirim (DO diterbitkan)

├─ Application                                 # Orkestrasi use case → pakai Domain untuk eksekusi perintah/kueri
│  ├─ Commands                                 # Aksi → mengubah state
│  │  ├─ PlaceOrderCommand.cs                  # Command membuat pesanan kendaraan baru
│  │  ├─ ConfirmSaleCommand.cs                 # Command mengonfirmasi penjualan kendaraan
│  │  └─ IssueDeliveryOrderCommand.cs          # Command menerbitkan DO (Delivery Order)
│  ├─ Queries                                  # Aksi baca → tidak mengubah state
│  │  ├─ GetStockQuery.cs                      # Query ambil daftar stok kendaraan
│  │  └─ GetSalesReportQuery.cs                # Query laporan penjualan (periode tertentu)
│  └─ Services                                 # Service khusus untuk logika bisnis lintas aggregate
│     └─ SalesDomainService.cs                 # Contoh: validasi stok tersedia sebelum pesanan dibuat

├─ Infrastructure                              # Implementasi teknis (DB, persistence, migrations, external IO)
│  ├─ EFConfigurations                         # Mapping Entity Framework Core ke tabel
│  │  └─ VehicleOrderConfiguration.cs          # Konfigurasi tabel VehicleOrder (kolom, relasi, constraint)
│  ├─ Repositories                             # Implementasi dari kontrak Repository (pakai EF Core, Dapper, dsb)
│  │  └─ VehicleOrderRepository.cs             # Implementasi IVehicleOrderRepository (akses database)
│  └─ Migrations                               # Script perubahan database
│     └─ 20250822_InitialSales.cs              # Migrasi awal schema modul Sales

└─ Api                                        # Endpoint HTTP (REST/gRPC/GraphQL) untuk konsumsi client
   ├─ Controllers                              # Controller ASP.NET Core → expose command/query ke luar
   │  └─ SalesController.cs                    # API endpoint (POST /sales/order, GET /sales/report, dsb)
   └─ DTOs                                    # Data Transfer Object → kontrak input/output API
      ├─ PlaceOrderDto.cs                      # DTO request untuk membuat pesanan
      ├─ ConfirmSaleDto.cs                     # DTO request untuk mengonfirmasi penjualan
      └─ DeliveryOrderDto.cs                   # DTO request untuk menerbitkan DO

```

### 3B - Contoh Koi.Spareparts)
```
src/Koi.Spareparts/
├─ Domain/
│  ├─ Entities/ ValueObjects/ Aggregates/
│  ├─ Events/  # domain events
│  ├─ Policies/
│  └─ Services/ # domain services murni
├─ Application/
│  ├─ Abstractions/ (ports: repos, clock, userContext)
│  ├─ Commands/   (CQRS write: CreatePart, ReceiveStock, AdjustStock)
│  ├─ Queries/    (CQRS read: GetPart, SearchStock)
│  ├─ DTOs/
│  ├─ Behaviors/  (logging, validation, tx, outbox)
│  └─ Mappings/
├─ Infrastructure/
│  ├─ Persistence/ EF DbContext, Config, Migrations
│  ├─ Repositories/
│  ├─ Outbox/
│  └─ Services/ external adapters (DMS, files, email)
└─ Api/
   ├─ Controllers/ or MinimalEndpoints
   └─ Contracts/ (request/response)
```

Jadi struktur ini benar-benar backend modular:
Domain = inti aturan bisnis.
Application = jalankan use case.
Infrastructure = detail teknis (DB, repo).
Api = expose ke dunia luar.

---

## 4 – Konvensi Namespace & Paket

-   **Namespaces**: `Koi.<Module>.<Layer>` (contoh: `Koi.Spareparts.Domain`).
-   **Paket NuGet** (suggested):
    -   `MediatR`, `FluentValidation`, `Mapster`, `EFCore.PG`, `EFCore.BulkExtensions`,
    -   `Serilog.AspNetCore`, `OpenTelemetry.Exporter.Otlp`,
    -   `Scrutor` (assembly scanning), `NodaTime`, `Polly`,
    -   `Quartz`, `IdempotentUtils`.

---

## 5 – PostgreSQL Skema & Partitioning

-   **Kolom standar multi-cabang & periode**: `branch_code`, `period_month`, auditing.
-   **Partitioning**: RANGE by `period_month`, LIST by `branch_code`.
-   **Automasi partisi** via Quartz job.
-   **Schema**: `core` (transaksi), `read` (reporting), `intg` (staging integrasi).
-   **Extensi**: `uuid-ossp`, `pg_trgm`, `btree_gin`.
    
---

## 6 – Pola Integrasi & Outbox

-   Setiap transaksi tulis menghasilkan **Domain Event** → disimpan di tabel **Outbox**.
-   **Outbox Dispatcher** (Quartz hosted service) mem-publish event ke Kafka/RabbitMQ atau internal handler.
-   Tujuan: **idempotency** & **eventual consistency** antar modul.
    

Contoh tabel outbox:

```
CREATE TABLE outbox (
  id UUID PRIMARY KEY,
  occurred_on TIMESTAMPTZ NOT NULL,
  type TEXT NOT NULL,
  payload JSONB NOT NULL,
  processed_at TIMESTAMPTZ,
  error TEXT
);
```
---

## 7 – Pipeline Aplikasi (MediatR Behaviors)

1.  **ValidationBehavior** (FluentValidation)    
2.  **AuthorizationBehavior** (cek permission per modul)
3.  **UnitOfWorkBehavior** (EF transaction + outbox)
4.  **Logging/TracingBehavior** (Serilog + OpenTelemetry)
----------

## 8 – Contoh Endpoint & Handler

**Command**: Terima stok parts

```
public sealed record ReceiveStockCommand(
    string BranchCode,
    YearMonth Period,
    string PartNo,
    decimal Qty,
    string Uom,
    string ReferenceNo
) : IRequest<Result<Guid>>;

public sealed class ReceiveStockHandler : IRequestHandler<ReceiveStockCommand, Result<Guid>>
{
    private readonly ISparepartRepository _repo;
    private readonly IUnitOfWork _uow;

    public async Task<Result<Guid>> Handle(ReceiveStockCommand cmd, CancellationToken ct)
    {
        var part = await _repo.GetByPartNoAsync(cmd.PartNo, ct);
        if (part is null) return Result.Failure<Guid>("PartNotFound");

        part.Receive(cmd.BranchCode, cmd.Period, cmd.Qty, cmd.Uom, cmd.ReferenceNo);
        await _uow.SaveChangesAsync(ct);
        return Result.Success(part.Id);
    }
}
```

----------

## 9 – Migrasi dari MySQL 5.0

1.  Inventarisasi semua sproc → mapping ke Application/Domain Services.
2.  Mapping tipe data MySQL → PostgreSQL.
3.  Buat ETL staging (`intg.stg_*`).
4.  Migrasi batch per cabang & periode.
5.  Rekonsiliasi saldo (stock, AR/AP, GL).
6.  Cutover per modul.
7.  Arsip MySQL lama read-only.
    

----------

## 10 – Konvensi Penamaan

-   Entities singular; Tables plural snake_case.    
-   PK: `BIGSERIAL` atau `UUID`.
-   Periode: `period_month DATE = first day of month`.
-   Branch: `CHAR(3)` uppercase.
-   Audit: `created_*`, `updated_*`.
    

----------

## 11 – Bootstrap Host (Koi.Bootstrap)

-   Registrasi modul (MediatR, Validators, DbContexts).
-   Health checks, versioning, feature flags.
-   Middleware: exception, correlation-id, logging.
    

Contoh wiring:

```
builder.Services.AddKoiBuildingBlocks();
builder.Services.AddSparepartsModule(cfg => cfg.UsePostgres(cs));
app.MapSparepartsEndpoints();
```

----------

## 12 – DevOps & Tooling

-   **Docker Compose**: postgres, pgadmin, jaeger, seq.
-   **CI/CD**: build, test, migrate → deploy.    
-   **Feature Flags** untuk rollout bertahap.
    

----------

## 13 – Security & Permission Model

-   **User** → **Roles** → **Permissions** per modul (`Spareparts.Receive`, `Service.WO.Create`).
-   Policy-based auth di endpoint.
-   Cache claims dari token JWT.
    

----------

## 14 – Roadmap Implementasi

1.  Week 1–2: Bootstrap infra & SharedKernel.
2.  Week 3–6: Modul Spareparts.
3.  Week 7–10: Modul Service.
4.  Week 11–14: BodyPaint.
5.  Week 15–18: Janaru GL.
6.  Week 19–20: Reporting, observability.
    
----------

## 15 – Contoh DTO

```
public sealed record ReceiveStockRequest(
  string BranchCode,
  string Period,
  string PartNo,
  decimal Qty,
  string Uom,
  string ReferenceNo
);

public sealed record ReceiveStockResponse(Guid ReceiptId);
```

----------

## 16 – Testing Strategy

-   Unit: domain rules, value objects.
-   Integration: handlers + Postgres container.
-   Contract: API schema tests.
-   Data reconciliation: compare totals old vs new.
    

----------

## 17 – Checklist Cutover Cabang

-   ✅ Network stabil (<150 ms).
-   ✅ User & Role provisioning.
-   ✅ Migrasi master data fix.
-   ✅ ETL transaksi awal & saldo.
-   ✅ Pelatihan user + SOP fallback.
    

----------

## 18 – Naming Accounting (Janaru)

-   `Koi.Janaru.GeneralLedger`
-   `Koi.Janaru.AccountPayable`
-   `Koi.Janaru.AccountReceivable`
-   `Koi.Janaru.Tax`
    

2. Identity vs Security 🔄

Koi.Identity → fokus authN/authZ (users, roles, tokens, SSO).

Koi.Security → fokus cross-cutting: RBAC enforcement, audit, compliance, encryption.
👉 Jangan duplikasi role/permission di dua tempat, semua definisi role/permission tetap di Identity, tapi enforcement & audit di Security.

3. Outbox & Eventing 📤

Tambahkan alur yang jelas:

Domain Event → Outbox Event → Publisher → External Bus

👉 Jadi tidak ada event langsung keluar tanpa lewat outbox.

4. Observability 🔍

Tambahkan 2 hal:

Correlation Id Middleware (tiap request traceable end-to-end).

Alert Rules di Koi.Monitoring (ex. suspicious login, negative stock).

5. Database Strategy 🗄️

Sudah oke partisi by period_month + branch_code.
Tambahkan:

Auto Partition Manager → job scheduler untuk create/drop partisi.

Composite Index → (branch_code, period_month) supaya query reporting cepat.

6. Testing 🔬

Tambahkan:

Property-based Testing untuk domain rules (contoh: validasi perhitungan pajak, posting GL).

7. CI/CD 🚀

Tambahkan:

DB Migration Automation (jalankan migrasi otomatis saat deploy).

Feature Flags untuk rollout bertahap modul baru.

8. Roadmap 🛣️

Spareparts → Service → BodyPaint → GL → Reporting tetap.
Tambahan:

Parallel Track Security (Koi.Identity + Koi.Security) harus dibangun sejak awal.

Integration IBS Replication start dari modul pertama (Spareparts) supaya sinkronisasi data teruji lebih cepat.
----------

## 19 – What’s Next

-   Generate repo template (dotnet new).
-   Build flow Spareparts.Receive end-to-end.
-   Mapper sproc → handlers (top 10 first).
