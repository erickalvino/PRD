# ==============================================================================
# PRODUCT REQUIREMENT DOCUMENT (PRD) — REVISI 2.0 — DRAFT UNTUK REVIEW
# SISTEM: MODUL DEPLOYMENT ASET & PERBAIKAN MULTI-CABANG TINGKAT ENTERPRISE
# PLATFORM: SNIPE-IT (TARGET VERSION: v6.x / v7.x — VERIFIKASI DI TIM IMPLEMENTASI)
# TANDA TANGAN KODE: erickalvino
# ==============================================================================

# ------------------------------------------------------------------------------
# 📌 BAGIAN 0: METADATA REVISI & DAFTAR PERUBAHAN ATAS PRD v1
# ------------------------------------------------------------------------------

## 📄 0.1 Dokumen yang di-revisi
- **Sumber:** `PRD-DEPLOY-SERVICE.md` (Revisi Final v1, 40 bagian)
- **Hasil review:** `REVIEW-PRD-DEPLOY-SERVICE.md`
- **Tujuan revisi:** Menghilangkan semua pendekatan `hardcoded`, melengkapi artefak yang hilang, dan menutup celah teknis sebelum kode di-inject ke Snipe-IT produksi.

## 📄 0.2 Daftar perbaikan utama (traceability ke review)

| ID | Permasalahan di v1 | Perbaikan di v2 |
|---|---|---|
| R-01 | `config/custom.php` & `.env` tidak didefinisikan | Ditambahkan spesifikasi config + env + resolver service |
| R-02 | Hardcoded location code / ID (`location_id == 2`, `branches.1`, 991/992/993) | Diubah 100% ke `BranchResolver` + config + env, tanpa fallback ID mati |
| R-03 | Reservation / Auto-Release tidak nyata | Ditambahkan tabel reservasi + `StockReservationService` + `StockEngine` |
| R-04 | Policy & Gate tidak ada | Ditambahkan `AssetDeploymentPolicy`, `CustomMaintenancePolicy`, permission matrix |
| R-05 | State machine tidak lengkap | Ditambahkan seluruh transition endpoint & method |
| R-06 | View Deployment tidak ada | Ditambahkan spesifikasi view deployment lengkap |
| R-07 | Form maintenance tidak sinkron dengan FormRequest | Form request & view disatukan di spec |
| R-08 | Route name tidak konsisten | Route name dinormalkan (`*print-pdf`, `*print-sticker`) |
| R-09 | State-change via HTTP GET | Semua transisi status pakai `POST` + CSRF |
| R-10 | `trans() ??` tidak berfungsi | Dihapus, semua menggunakan `Lang::has()` atau literal |
| R-11 | File translasi `custom` tidak ada | Ditambahkan skeleton `lang/*/custom.php` |
| R-12 | FK tidak lengkap | Ditambahkan FK lengkap + `unsignedBigInteger` konsisten |
| R-13 | `item_id` unregistered donor tidak konsisten | `item_id` dibuat nullable + kolom `part_name`/`part_spec` |
| R-14 | Accessory di-assign ke asset (salah) | Ditambahkan pivot `asset_accessory_installs` / kebijakan assignee yang benar |
| R-15 | License tidak pernah diproses | Ditambahkan `LicenseSeatHandler` untuk allocate/release seat |
| R-16 | Karantina tidak sinkron dengan config | Ditambahkan `QuarantineService` terpusat |
| R-17 | Tidak ada concurrency control | `lockForUpdate()` + unique constraint + idempotency key |
| R-18 | Test tidak lengkap | Ditambahkan factory + feature test deployment & maintenance |

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 1: RINGKASAN EKSEKUTIF, RUANG LINGKUP & ASUMSI
# ------------------------------------------------------------------------------

## 📄 1.1 Ringkasan Eksekutif

Sistem ini merupakan **dua modul kustom terintegrasi** yang berjalan di atas platform **Snipe-IT**:

1. **Modul Asset Deployment & Setup**
2. **Modul Custom Maintenance & Life-Cycle Evaluation**
3. **Modul Dashboard & Reporting** *(pelengkap untuk TCO/SLA/monitoring cabang)*

Perubahan fundamental pada revisi 2.0:
- Seluruh referensi cabang, gudang karantina, status karantina, dan kode dokumen **tidak lagi hardcoded**.
- **Tidak ada lagi fallback ke `branches.1`**. Jika konfigurasi cabang tidak ditemukan, sistem **gagal dengan pesan yang jelas**, bukan memakai data salah.
- Stok yang dikunci ketika dokumen terbit disimpan di **tabel reservasi khusus**, bukan hanya komentar di kode.
- Aktor, role, dan permission diatur melalui **Laravel Policy + Gate**, tidak lagi hanya `Auth::user()->isSuperUser()`.

## 📄 1.2 Ruang Lingkup

- 1 Kantor Pusat + 2 Kantor Cabang (default: Jakarta, Bandung, Surabaya — **harus melalui config, bukan asumsi di kode**).
- Alur: buat draft → terbitkan → proses → QC → serah terima → selesai / rusak total / batal.
- Tanpa menghilangkan fitur native Snipe-IT; modul kustom hanya menambah lapisan workflow & transaksi.

## 📄 1.3 Asumsi Teknis (WAJIB DITEGASKAN SEBELUM IMPLEMENTASI)

| Asumsi | Nilai |
|---|---|
| Snipe-IT | v6.x / v7.x (pilih satu, verifikasi skema) |
| PHP | 8.1+ |
| Laravel | versi mengikuti Snipe-IT target |
| DB | MySQL 8.0 / MariaDB 10.6 |
| Tabel core Snipe-IT | `assets`, `users`, `companies`, `locations`, `departments`, `suppliers`, `status_labels`, `components`, `accessories`, `licenses`, `license_seats`, `checkouts`, `action_logs` |
| Pivot core | `components_assets`, `accessories_users`, `license_seats`, `checkouts` — **nama kolom divergensi harus diverifikasi di versi target** |

> ⚠️ **Keputusan wajib:** sebelum implementasi, tim harus menjalankan `composer require` di instalasi Snipe-IT target dan mencocokkan seluruh nama tabel/kolom pivot yang dipakai. Bagian ini tidak boleh dianggap final tanpa verifikasi.

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 2: ATURAN BISNIS YANG TIDAK BOLEH DILANGGAR
# ------------------------------------------------------------------------------

## 📄 2.1 Aturan Global (Golden Rules)

1. **Tidak ada hardcoded ID/code.** Semua ID, kode cabang, status karantina, dan URL internal keluar dari config / env / database.
2. **Historical Data Lock.** Kolom `company_id` dan `location_id` selalu disimpan fisik di tabel transaksi dan tidak boleh diubah setelah transaksi dibuat dari `DRAFT`.
3. **Creator Only (DRAFT).** Draf hanya boleh diubah, diterbitkan, atau dihapus oleh pembuat dokumen, kecuali Superadmin.
4. **Cancellation mandatory reason.** `cancellation_notes` minimal 15 karakter.
5. **Auto-release stock.** Setiap pembatalan transaksi yang sudah disetujui wajib melepas reservasi stok dalam satu transaksi database.
6. **Audit trail.** Setiap transisi status wajib tercatat di `custom_module_logs`.
7. **Multi-tenancy.** Kueri pada kedua modul wajib mengikuti konteks `company` (dan, jika dikonfigurasi, `location`) dari user yang sedang login.
8. **Idempotency.** Endpoint transisi wajib aman dijalankan dua kali (menggunakan transisi state machine + unique constraint / idempotency key).

## 📄 2.2 Peran (Role) Acuan

| Role | Singkatan | Focus |
|---|---|---|
| Super Admin | SUPER | Semua |
| Fixed Asset / GA | FA | Membuat draft, menerbitkan, approve, void, print |
| IT Technician | IT | Diagnosis, servis, QC, kanibalisasi |
| Branch Admin | BR-ADMIN | View + export data cabang sendiri |
| End User | USER | Tidak memiliki akses ke modul ini |

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 3: STATE MACHINE FINAL
# ------------------------------------------------------------------------------

## 📄 3.1 Modul Deployment

```text
[DRAFT]
  ├─ terbitkan (Creator/FA)  --> [PENDING_HANDOVER]
  ├─ ubah/hapus (Creator/FA)
  └─ void (Creator/FA)      --> [CANCELLED]   (auto-release reservation)

[PENDING_HANDOVER] --> start progress  --> [IN_PROGRESS]
[IN_PROGRESS]      --> QC passed       --> [READY_TO_RETURN]
[READY_TO_RETURN]  --> FA approve      --> [COMPLETED]
[READY_TO_RETURN]  --> FA void         --> [CANCELLED]
```
- `COMPLETED` dan `CANCELLED` = terminal (tidak dapat diubah lagi).
- Reservasi stok dimulai saat **terbitkan** dan dilepas saat **void**.
- Pemotongan stok riil (decrement) terjadi saat **COMPLETED**.

## 📄 3.2 Modul Maintenance

```text
[DRAFT]
  ├─ terbitkan (Creator/FA) --> [PENDING_CHECKING]
  ├─ ubah/hapus (Creator/FA)
  └─ void (Creator/FA)      --> [CANCELLED]

[PENDING_CHECKING] --> menerima diagnosis     --> [UNDER_DIAGNOSIS]
[UNDER_DIAGNOSIS]  --> interior service       --> [IN_SERVICE_INTERNAL]
[UNDER_DIAGNOSIS]  --> kirim ke vendor        --> [OUT_TO_VENDOR]
[IN_SERVICE_INTERNAL | OUT_TO_VENDOR] --> QC --> [POST_SERVICE_QC]
[POST_SERVICE_QC]  --> good                  --> [READY_TO_RETURN]
[READY_TO_RETURN]  --> FA approve (good)     --> [COMPLETED]
[READY_TO_RETURN]  --> FA approve (BER)      --> [UNREPAIRABLE]
[selain terminal]  --> FA void               --> [CANCELLED]
```
- `COMPLETED`, `UNREPAIRABLE`, `CANCELLED` = terminal.
- **Registrasi kanibalisasi** di draft berstatus *plan*. Saat `COMPLETED`, sistem memutuskan apakah akan decrement stok (registered) atau hanya log part fisik (unregistered).

## 📄 3.3 Matriks Transisi & Authorization

| Modul | Dari | Aksi | Ke | Aktor | Method Controller | Route |
|---|---|---|---|---|---|---|
| Deploy | DRAFT | release | PENDING_HANDOVER | Creator/FA | `releaseToPending` | `assetDeployments.release` |
| Deploy | PENDING_HANDOVER | start | IN_PROGRESS | IT/FA | `startProgress` | `assetDeployments.start` |
| Deploy | IN_PROGRESS | qc | READY_TO_RETURN | IT/FA | `markReady` | `assetDeployments.ready` |
| Deploy | READY_TO_RETURN | complete | COMPLETED | FA | `complete` | `assetDeployments.complete` |
| Deploy | semua aktif | void | CANCELLED | FA | `cancelDeployment` | `assetDeployments.cancel` |
| Maint | DRAFT | release | PENDING_CHECKING | Creator/FA | `releaseToPendingChecking` | `customMaintenances.release` |
| Maint | PENDING_CHECKING | accept | UNDER_DIAGNOSIS | IT | `acceptDiagnosis` | `customMaintenances.diagnosis` |
| Maint | UNDER_DIAGNOSIS | internal | IN_SERVICE_INTERNAL | IT | `startInternal` | `customMaintenances.internal` |
| Maint | UNDER_DIAGNOSIS | vendor | OUT_TO_VENDOR | FA/IT | `sendToVendor` | `customMaintenances.vendor` |
| Maint | IN_SERVICE_INTERNAL/OUT_TO_VENDOR | qc | POST_SERVICE_QC | IT | `postServiceQc` | `customMaintenances.qc` |
| Maint | POST_SERVICE_QC | ready | READY_TO_RETURN | IT/FA | `readyToReturn` | `customMaintenances.ready` |
| Maint | READY_TO_RETURN | complete | COMPLETED | FA | `complete` | `customMaintenances.complete` |
| Maint | READY_TO_RETURN | unrepairable | UNREPAIRABLE | FA | `rejectToUnrepairable` | `customMaintenances.unrepairable` |
| Maint | semua aktif | void | CANCELLED | FA | `cancelMaintenance` | `customMaintenances.cancel` |

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 4: KONFIGURASI, ENV & RESOLVER CABANG (PENGGAANTI HARCODING)
# ------------------------------------------------------------------------------

## 📄 4.1 `config/custom.php` (spesifikasi penuh)

```php
<?php
// erickalvino-MODULE_CORE: Konfigurasi terpusat multi-cabang kustom — tanpa ID hardcoded
return [
    // Status label yang dipakai untuk mengunci unit rusak total / BER
    // Nilai HARUS diisi dari env, tidak boleh di-hardcode di controller.
    'quarantine_status_id' => (int) env('CUSTOM_QUARANTINE_STATUS_ID', 0),

    // Tabel pivot attachment yang dipakai modul (diverifikasi di Snipe-IT target)
    'pivot' => [
        'component_asset'     => 'components_assets',
        'accessory_install'   => 'asset_accessory_installs',
        'license_seat'        => 'license_seats',
    ],

    // Pemetaan cabang berdasarkan location_id (bukan ID absolut yang di-hardcode,
    // melainkan data konfigurasi yang diisi sesuai database Snipe-IT).
    'branches' => [
        1 => [
            'code'                  => env('CUSTOM_BRANCH_1_CODE', 'JKT'),
            'name'                  => env('CUSTOM_BRANCH_1_NAME', 'Jakarta'),
            'quarantine_asset_id'   => (int) env('CUSTOM_BRANCH_1_QUARANTINE_ASSET_ID', 0),
            'quarantine_location_id'=> (int) env('CUSTOM_BRANCH_1_QUARANTINE_LOCATION_ID', 0),
            'prefix'                => env('CUSTOM_BRANCH_1_PREFIX', 'JKT'),
        ],
        2 => [
            'code'                  => env('CUSTOM_BRANCH_2_CODE', 'BDG'),
            'name'                  => env('CUSTOM_BRANCH_2_NAME', 'Bandung'),
            'quarantine_asset_id'   => (int) env('CUSTOM_BRANCH_2_QUARANTINE_ASSET_ID', 0),
            'quarantine_location_id'=> (int) env('CUSTOM_BRANCH_2_QUARANTINE_LOCATION_ID', 0),
            'prefix'                => env('CUSTOM_BRANCH_2_PREFIX', 'BDG'),
        ],
        3 => [
            'code'                  => env('CUSTOM_BRANCH_3_CODE', 'SBY'),
            'name'                  => env('CUSTOM_BRANCH_3_NAME', 'Surabaya'),
            'quarantine_asset_id'   => (int) env('CUSTOM_BRANCH_3_QUARANTINE_ASSET_ID', 0),
            'quarantine_location_id'=> (int) env('CUSTOM_BRANCH_3_QUARANTINE_LOCATION_ID', 0),
            'prefix'                => env('CUSTOM_BRANCH_3_PREFIX', 'SBY'),
        ],
    ],

    // Panjang minimum alasan wajib
    'mandatory_min_chars' => [
        'cancellation' => 15,
        'unrepairable' => 10,
    ],

    // Prefix nomor dokumen
    'document_prefix' => [
        'maintenance' => 'SJP',
        'deployment'  => 'BAST',
    ],
];
```

## 📄 4.2 `.env` sample

```env
# Status karantina (id status_labels di Snipe-IT, harus diisi)
CUSTOM_QUARANTINE_STATUS_ID=0

# Cabang 1
CUSTOM_BRANCH_1_CODE=JKT
CUSTOM_BRANCH_1_NAME=Jakarta
CUSTOM_BRANCH_1_QUARANTINE_ASSET_ID=0
CUSTOM_BRANCH_1_QUARANTINE_LOCATION_ID=0
CUSTOM_BRANCH_1_PREFIX=JKT

# Cabang 2
CUSTOM_BRANCH_2_CODE=BDG
CUSTOM_BRANCH_2_NAME=Bandung
CUSTOM_BRANCH_2_QUARANTINE_ASSET_ID=0
CUSTOM_BRANCH_2_QUARANTINE_LOCATION_ID=0
CUSTOM_BRANCH_2_PREFIX=BDG

# Cabang 3
CUSTOM_BRANCH_3_CODE=SBY
CUSTOM_BRANCH_3_NAME=Surabaya
CUSTOM_BRANCH_3_QUARANTINE_ASSET_ID=0
CUSTOM_BRANCH_3_QUARANTINE_LOCATION_ID=0
CUSTOM_BRANCH_3_PREFIX=SBY
```

## 📄 4.3 `BranchResolver` Service

```php
<?php
// erickalvino-MODULE_CORE: Branch resolver terpusat — melempar error bila config kurang
namespace App\Services\Custom;

use RuntimeException;

class BranchResolver
{
    public static function get(int $locationId): array
    {
        $config = config("custom.branches.{$locationId}");

        if (! is_array($config)) {
            throw new RuntimeException(
                "Konfigurasi branch untuk location_id {$locationId} tidak ditemukan. ".
                "Mohon lengkapi config/custom.php dan file .env."
            );
        }

        if ((int) $config['quarantine_asset_id'] <= 0) {
            throw new RuntimeException(
                "Quarantine asset untuk lokasi {$locationId} belum dikonfigurasi."
            );
        }

        return $config;
    }

    public static function code(int $locationId): string
    {
        return (string) self::get($locationId)['prefix'];
    }
}
```

## 📄 4.4 Validator config saat boot

```php
// app/Providers/CustomModuleConfigServiceProvider.php
// erickalvino-MODULE_CORE: Validasi config custom saat boot agar konfigurasi kosong langsung terdeteksi
public function boot(): void
{
    $this->validateCustomConfig();
}

private function validateCustomConfig(): void
{
    if ((int) config('custom.quarantine_status_id') <= 0) {
        throw new \RuntimeException('[CUSTOM MODULE] CUSTOM_QUARANTINE_STATUS_ID wajib diisi.');
    }

    foreach (array_keys(config('custom.branches') ?? []) as $locationId) {
        $branch = config("custom.branches.{$locationId}");
        if (! is_array($branch) || empty($branch['prefix'])) {
            throw new \RuntimeException("[CUSTOM MODULE] Konfigurasi branch {$locationId} tidak lengkap.");
        }
    }
}
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 5: SKEMA DATABASE (MIGRASI FINAL)
# ------------------------------------------------------------------------------

> Semua primary key & foreign key memakai `unsignedBigInteger`/`bigInteger unsigned` yang konsisten. Semua kolom FK mengarah ke tabel core Snipe-IT.

## 📄 5.1 `2026_09_06_000001_create_asset_deployments_table.php`

```php
<?php
// erickalvino-MODULE_DEPLOY: Migrasi tabel induk asset_deployments — historical data lock + FK konsisten
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('asset_deployments', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->unsignedBigInteger('asset_id')->index();
            $table->unsignedBigInteger('company_id')->index();
            $table->unsignedBigInteger('location_id')->index();
            $table->unsignedBigInteger('target_department_id')->index();
            $table->unsignedBigInteger('assigned_technician_id')->nullable()->index();

            $table->enum('status', [
                'DRAFT','PENDING_HANDOVER','IN_PROGRESS',
                'READY_TO_RETURN','COMPLETED','CANCELLED',
            ])->default('DRAFT')->index();

            $table->text('cancellation_notes')->nullable();
            $table->unsignedBigInteger('created_by')->index();
            $table->string('document_number', 100)->nullable()->unique();
            $table->unsignedBigInteger('reservation_token_id')->nullable()->index();
            $table->timestamps();
            $table->softDeletes();

            $table->foreign('asset_id', 'fk_deploy_asset')
                  ->references('id')->on('assets')->onDelete('restrict');
            $table->foreign('company_id', 'fk_deploy_company')
                  ->references('id')->on('companies')->onDelete('restrict');
            $table->foreign('location_id', 'fk_deploy_location')
                  ->references('id')->on('locations')->onDelete('restrict');
            $table->foreign('target_department_id', 'fk_deploy_department')
                  ->references('id')->on('departments')->onDelete('restrict');
            $table->foreign('assigned_technician_id', 'fk_deploy_technician')
                  ->references('id')->on('users')->onDelete('set null');
            $table->foreign('created_by', 'fk_deploy_creator')
                  ->references('id')->on('users')->onDelete('restrict');
        });

        Schema::table('asset_deployments', function (Blueprint $table) {
            $table->index(['location_id', 'status'], 'idx_deploy_location_status');
            $table->index(['company_id', 'status'], 'idx_deploy_company_status');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('asset_deployments');
    }
};
```

## 📄 5.2 `2026_09_06_000002_create_asset_deployment_reservations_table.php`

```php
<?php
// erickalvino-MODULE_DEPLOY: Tabel reservasi stok — inti dari auto-release stock yang nyata
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('asset_deployment_reservations', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->unsignedBigInteger('deployment_id')->index();
            $table->enum('item_type', ['COMPONENT', 'ACCESSORY', 'LICENSE'])->index();
            $table->unsignedBigInteger('item_id')->index();
            $table->unsignedInteger('qty')->default(1);
            $table->unsignedBigInteger('location_id')->index();
            $table->enum('status', ['RESERVED', 'RELEASED', 'CONSUMED'])
                  ->default('RESERVED')->index();
            $table->timestamp('reserved_at')->nullable()->useCurrent();
            $table->timestamp('released_at')->nullable();
            $table->timestamp('consumed_at')->nullable();
            $table->unsignedBigInteger('created_by')->index();
            $table->timestamps();

            $table->unique(['deployment_id', 'item_type', 'item_id'], 'uniq_deploy_res_item');

            $table->foreign('deployment_id', 'fk_reserve_deploy')
                  ->references('id')->on('asset_deployments')->onDelete('cascade');
            $table->foreign('location_id', 'fk_reserve_location')
                  ->references('id')->on('locations')->onDelete('restrict');
            $table->foreign('created_by', 'fk_reserve_creator')
                  ->references('id')->on('users')->onDelete('restrict');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('asset_deployment_reservations');
    }
};
```

## 📄 5.3 `2026_09_06_000003_create_asset_deployment_allocated_items_table.php`

```php
<?php
// erickalvino-MODULE_DEPLOY: Tabel rencana alokasi item baru non-TAG
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('asset_deployment_allocated_items', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->unsignedBigInteger('deployment_id')->index();
            $table->enum('item_type', ['COMPONENT', 'ACCESSORY', 'LICENSE'])->index();
            $table->unsignedBigInteger('item_id')->index();
            $table->unsignedInteger('qty')->default(1);
            $table->text('notes')->nullable();
            $table->unsignedBigInteger('created_by')->index();

            $table->unique(['deployment_id', 'item_type', 'item_id'], 'uniq_alloc_item');

            $table->foreign('deployment_id', 'fk_alloc_deploy')
                  ->references('id')->on('asset_deployments')->onDelete('cascade');
            $table->foreign('created_by', 'fk_alloc_creator')
                  ->references('id')->on('users')->onDelete('restrict');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('asset_deployment_allocated_items');
    }
};
```

## 📄 5.4 `2026_09_06_000004_create_asset_deployment_detached_items_table.php`

```php
<?php
// erickalvino-MODULE_DEPLOY: Tabel item lama yang dicopot saat upgrade/refurbishment
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('asset_deployment_detached_items', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->unsignedBigInteger('deployment_id')->index();
            $table->enum('item_type', ['COMPONENT', 'ACCESSORY', 'LICENSE']);
            $table->unsignedBigInteger('item_id')->index();
            $table->unsignedInteger('qty')->default(1);
            $table->enum('condition', ['GOOD', 'BROKEN'])->nullable();
            $table->boolean('release_license_seat')->default(false);
            $table->unsignedBigInteger('quarantine_asset_id')->nullable()->index();
            $table->text('notes')->nullable();
            $table->unsignedBigInteger('created_by')->index();

            $table->foreign('deployment_id', 'fk_detach_deploy')
                  ->references('id')->on('asset_deployments')->onDelete('cascade');
            $table->foreign('quarantine_asset_id', 'fk_detach_quarantine')
                  ->references('id')->on('assets')->onDelete('set null');
            $table->foreign('created_by', 'fk_detach_creator')
                  ->references('id')->on('users')->onDelete('restrict');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('asset_deployment_detached_items');
    }
};
```

## 📄 5.5 `2026_09_06_000005_create_custom_maintenances_table.php`

```php
<?php
// erickalvino-MODULE_MAINT: Tabel induk perbaikan — historical lock + SLA/warranty + TCO
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('custom_maintenances', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->unsignedBigInteger('asset_id')->index();
            $table->unsignedBigInteger('company_id')->index();
            $table->unsignedBigInteger('location_id')->index();
            $table->enum('maintenance_type', ['INTERNAL', 'EXTERNAL']);
            $table->boolean('is_warranty_claim')->default(false);
            $table->enum('warranty_status_at_launch', ['ACTIVE', 'EXPIRED', 'UNKNOWN'])->default('UNKNOWN');
            $table->unsignedBigInteger('supplier_id')->nullable()->index();
            $table->unsignedBigInteger('assigned_technician_id')->nullable()->index();

            $table->enum('status', [
                'DRAFT','PENDING_CHECKING','UNDER_DIAGNOSIS',
                'IN_SERVICE_INTERNAL','OUT_TO_VENDOR','POST_SERVICE_QC',
                'READY_TO_RETURN','COMPLETED','UNREPAIRABLE','CANCELLED',
            ])->default('DRAFT')->index();

            $table->date('estimated_completion_date')->nullable();
            $table->string('document_number', 100)->nullable()->unique();
            $table->text('issue_description');
            $table->text('repair_action')->nullable();
            $table->decimal('repair_cost', 20, 2)->default(0);
            $table->string('currency', 3)->default('IDR');

            $table->enum('unrepairable_reason_code', ['TOTAL_FAILURE', 'SPAREPART_UNAVAILABLE', 'OTHER'])->nullable();
            $table->text('unrepairable_notes')->nullable();
            $table->text('cancellation_notes')->nullable();
            $table->unsignedBigInteger('created_by')->index();
            $table->timestamps();
            $table->softDeletes();

            $table->index(['location_id', 'status'], 'idx_maint_location_status');
            $table->index(['company_id', 'status'], 'idx_maint_company_status');
            $table->index(['asset_id', 'status'], 'idx_maint_asset_status');

            $table->foreign('asset_id', 'fk_maint_asset')->references('id')->on('assets')->onDelete('restrict');
            $table->foreign('company_id', 'fk_maint_company')->references('id')->on('companies')->onDelete('restrict');
            $table->foreign('location_id', 'fk_maint_location')->references('id')->on('locations')->onDelete('restrict');
            $table->foreign('supplier_id', 'fk_maint_supplier')->references('id')->on('suppliers')->onDelete('set null');
            $table->foreign('assigned_technician_id', 'fk_maint_tech')->references('id')->on('users')->onDelete('set null');
            $table->foreign('created_by', 'fk_maint_creator')->references('id')->on('users')->onDelete('restrict');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('custom_maintenances');
    }
};
```

## 📄 5.6 `2026_09_06_000006_create_custom_maintenance_cannibals_table.php`

> **Penting:** `item_id` diubah menjadi **nullable**. Jika `is_unregistered_donor = 1`, kolom `part_name` + `part_spec` wajib diisi, karena komponen belum terdaftar di katalog Snipe-IT.

```php
<?php
// erickalvino-MODULE_MAINT: Tabel log kanibalisasi suku cadang — dukung part unregistered
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('custom_maintenance_cannibals', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->unsignedBigInteger('maintenance_id')->index();
            $table->unsignedBigInteger('donor_asset_id')->index();
            $table->enum('item_type', ['COMPONENT', 'ACCESSORY']);
            $table->unsignedBigInteger('item_id')->nullable()->index();
            $table->string('part_name', 255)->nullable();
            $table->string('part_spec', 500)->nullable();
            $table->boolean('is_unregistered_donor')->default(false);
            $table->unsignedInteger('qty')->default(1);
            $table->enum('status', ['PLANNED', 'APPLIED'])->default('PLANNED')->index();
            $table->unsignedBigInteger('created_by')->index();
            $table->timestamps();

            $table->foreign('maintenance_id', 'fk_cannibal_maint')
                  ->references('id')->on('custom_maintenances')->onDelete('cascade');
            $table->foreign('donor_asset_id', 'fk_cannibal_donor')
                  ->references('id')->on('assets')->onDelete('restrict');
            $table->foreign('created_by', 'fk_cannibal_creator')
                  ->references('id')->on('users')->onDelete('restrict');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('custom_maintenance_cannibals');
    }
};
```

## 📄 5.7 `2026_09_06_000007_create_custom_module_logs_table.php`

```php
<?php
// erickalvino-MODULE_DASH: Tabel audit trail terpusat — insert-only
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('custom_module_logs', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->enum('module_context', ['DEPLOYMENT', 'MAINTENANCE'])->index();
            $table->unsignedBigInteger('record_id')->index();
            $table->string('old_status', 50)->nullable();
            $table->string('new_status', 50)->index();
            $table->text('action_notes')->nullable();
            $table->unsignedBigInteger('user_id')->index();
            $table->text('ip_address')->nullable();
            $table->timestamp('created_at')->useCurrent();

            $table->foreign('user_id', 'fk_log_user')->references('id')->on('users')->onDelete('restrict');
            $table->index(['module_context', 'record_id'], 'idx_log_module_record');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('custom_module_logs');
    }
};
```

## 📄 5.8 Pivot kustom untuk aksesoris & license assignment

```php
<?php
// erickalvino-MODULE_DEPLOY: Pivot kustom aksesoris yang dipasang ke aset (bukan ke user)
return new class extends Migration
{
    public function up(): void
    {
        Schema::create('asset_accessory_installs', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->unsignedBigInteger('asset_id')->index();
            $table->unsignedBigInteger('accessory_id')->index();
            $table->unsignedInteger('qty')->default(1);
            $table->unsignedBigInteger('deployment_id')->nullable()->index();
            $table->unsignedBigInteger('created_by')->index();
            $table->timestamps();

            $table->foreign('asset_id')->references('id')->on('assets')->onDelete('cascade');
            $table->foreign('accessory_id')->references('id')->on('accessories')->onDelete('cascade');
            $table->foreign('deployment_id')->references('id')->on('asset_deployments')->onDelete('set null');
            $table->foreign('created_by')->references('id')->on('users')->onDelete('restrict');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('asset_accessory_installs');
    }
};
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 6: MODEL ELOQUENT & RELASI
# ------------------------------------------------------------------------------

## 📄 6.1 `AssetDeployment.php`

```php
<?php
// erickalvino-MODULE_DEPLOY: Model induk deployment + relasi + scope multi-company
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use App\Models\Traits\Presentable;

class AssetDeployment extends Model
{
    use SoftDeletes;
    use Presentable;

    protected $table = 'asset_deployments';
    protected $presenter = \App\Presenters\AssetDeploymentPresenter::class;

    protected $fillable = [
        'asset_id','company_id','location_id','target_department_id',
        'status','assigned_technician_id','cancellation_notes',
        'created_by','document_number','reservation_token_id',
    ];

    protected $casts = [
        'asset_id' => 'integer','company_id' => 'integer','location_id' => 'integer',
        'target_department_id' => 'integer','assigned_technician_id' => 'integer',
        'created_by' => 'integer','reservation_token_id' => 'integer',
    ];

    public function asset() { return $this->belongsTo(\App\Models\Asset::class, 'asset_id'); }
    public function company() { return $this->belongsTo(\App\Models\Company::class, 'company_id'); }
    public function location() { return $this->belongsTo(\App\Models\Location::class, 'location_id'); }
    public function targetDepartment() { return $this->belongsTo(\App\Models\Department::class, 'target_department_id'); }
    public function technician() { return $this->belongsTo(\App\Models\User::class, 'assigned_technician_id'); }
    public function creator() { return $this->belongsTo(\App\Models\User::class, 'created_by'); }
    public function allocatedItems() { return $this->hasMany(\App\Models\AssetDeploymentAllocatedItem::class, 'deployment_id'); }
    public function detachedItems() { return $this->hasMany(\App\Models\AssetDeploymentDetachedItem::class, 'deployment_id'); }
    public function reservations() { return $this->hasMany(\App\Models\AssetDeploymentReservation::class, 'deployment_id'); }

    public function scopeCompanyContext($query)
    {
        return $query->where(function ($q) {
            $user = auth()->user();
            if (! $user) return $q;
            if ($user->isSuperUser()) return $q;

            if (! empty($user->company_id)) {
                $q->where('company_id', $user->company_id);
            }
            return $q;
        });
    }
}
```

## 📄 6.2 `CustomMaintenance.php`

```php
<?php
// erickalvino-MODULE_MAINT: Model induk maintenance + relasi + warranty helper
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Support\Carbon;
use App\Models\Traits\Presentable;

class CustomMaintenance extends Model
{
    use SoftDeletes;
    use Presentable;

    protected $table = 'custom_maintenances';
    protected $presenter = \App\Presenters\CustomMaintenancePresenter::class;

    protected $fillable = [
        'asset_id','company_id','location_id','maintenance_type',
        'is_warranty_claim','warranty_status_at_launch','supplier_id',
        'assigned_technician_id','status','estimated_completion_date',
        'document_number','issue_description','repair_action','repair_cost',
        'currency','unrepairable_reason_code','unrepairable_notes',
        'cancellation_notes','created_by',
    ];

    protected $casts = [
        'asset_id' => 'integer','company_id' => 'integer','location_id' => 'integer',
        'supplier_id' => 'integer','assigned_technician_id' => 'integer',
        'created_by' => 'integer','repair_cost' => 'decimal:20',
        'is_warranty_claim' => 'boolean','estimated_completion_date' => 'date:Y-m-d',
    ];

    public function asset() { return $this->belongsTo(\App\Models\Asset::class, 'asset_id'); }
    public function company() { return $this->belongsTo(\App\Models\Company::class, 'company_id'); }
    public function location() { return $this->belongsTo(\App\Models\Location::class, 'location_id'); }
    public function supplier() { return $this->belongsTo(\App\Models\Supplier::class, 'supplier_id'); }
    public function technician() { return $this->belongsTo(\App\Models\User::class, 'assigned_technician_id'); }
    public function creator() { return $this->belongsTo(\App\Models\User::class, 'created_by'); }
    public function cannibalItems() { return $this->hasMany(\App\Models\CustomMaintenanceCannibal::class, 'maintenance_id'); }

    public function scopeCompanyContext($query)
    {
        // sama dengan AssetDeployment
        return $query->where(function ($q) {
            $user = auth()->user();
            if (! $user || $user->isSuperUser()) return $q;
            if (! empty($user->company_id)) {
                $q->where('company_id', $user->company_id);
            }
            return $q;
        });
    }

    public function detectWarrantyStatus(): string
    {
        $asset = $this->asset;
        if (! $asset || ! $asset->purchase_date || ! $asset->warranty_months) {
            return 'UNKNOWN';
        }
        $expiry = Carbon::parse($asset->purchase_date)->addMonths((int) $asset->warranty_months);
        return $expiry->isAfter(Carbon::now()) ? 'ACTIVE' : 'EXPIRED';
    }
}
```

## 📄 6.3 `AssetDeploymentReservation.php`

```php
<?php
// erickalvino-MODULE_DEPLOY: Model reservasi stok
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class AssetDeploymentReservation extends Model
{
    protected $table = 'asset_deployment_reservations';
    protected $fillable = [
        'deployment_id','item_type','item_id','qty','location_id',
        'status','reserved_at','released_at','consumed_at','created_by',
    ];
    protected $casts = [
        'deployment_id' => 'integer','item_id' => 'integer','qty' => 'integer',
        'location_id' => 'integer','created_by' => 'integer',
        'reserved_at' => 'datetime','released_at' => 'datetime','consumed_at' => 'datetime',
    ];

    public function deployment() { return $this->belongsTo(\App\Models\AssetDeployment::class); }
}
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 7: POLICY & AUTHORIZATION
# ------------------------------------------------------------------------------

## 📄 7.1 Permission keys

| Modul | Ability | Deskripsi |
|---|---|---|
| Deployment | `asset-deployment.view` | Lihat list |
| Deployment | `asset-deployment.create` | Buat draft |
| Deployment | `asset-deployment.edit` | Edit progres / transisi |
| Deployment | `asset-deployment.approve` | Approve COMPLETED / UNREPAIRABLE |
| Deployment | `asset-deployment.void` | Batalkan / void |
| Deployment | `asset-deployment.print` | Cetak BAST |
| Maintenance | `custom-maintenance.view` | Lihat list |
| Maintenance | `custom-maintenance.create` | Buat draft |
| Maintenance | `custom-maintenance.edit` | Edit progres |
| Maintenance | `custom-maintenance.approve` | Approve |
| Maintenance | `custom-maintenance.void` | Batalkan |
| Maintenance | `custom-maintenance.print` | Cetak surat jalan / stiker |

## 📄 7.2 `AssetDeploymentPolicy.php`

```php
<?php
// erickalvino-MODULE_DEPLOY: Policy untuk modul deployment
namespace App\Policies;

use App\Models\User;
use App\Models\AssetDeployment;

class AssetDeploymentPolicy
{
    public function view(User $user): bool
    {
        return $user->isSuperUser() || $user->hasPermission('asset-deployment.view');
    }

    public function create(User $user): bool
    {
        return $user->isSuperUser() || $user->hasPermission('asset-deployment.create');
    }

    public function edit(User $user, ?AssetDeployment $deployment = null): bool
    {
        if ($deployment && $deployment->status === 'DRAFT') {
            return $user->isSuperUser() || $deployment->created_by === $user->id;
        }
        return $user->isSuperUser() || $user->hasPermission('asset-deployment.edit');
    }

    public function approve(User $user): bool
    {
        return $user->isSuperUser() || $user->hasPermission('asset-deployment.approve');
    }

    public function void(User $user, ?AssetDeployment $deployment = null): bool
    {
        if (! $deployment || in_array($deployment->status, ['COMPLETED', 'CANCELLED'])) {
            return false;
        }
        return $user->isSuperUser() || $user->hasPermission('asset-deployment.void');
    }
}
```

## 📄 7.3 Register Policy di `AuthServiceProvider.php`

```php
// erickalvino-MODULE_DEPLOY: Registrasi policy kustom
protected $policies = [
    \App\Models\AssetDeployment::class => \App\Policies\AssetDeploymentPolicy::class,
    \App\Models\CustomMaintenance::class => \App\Policies\CustomMaintenancePolicy::class,
];
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 8: FORM REQUESTS (VALIDASI FINAL)
# ------------------------------------------------------------------------------

## 📄 8.1 `DeploymentStoreRequest.php`

```php
<?php
// erickalvino-MODULE_DEPLOY: Form request penyimpanan draft deployment
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Support\Facades\Gate;

class DeploymentStoreRequest extends FormRequest
{
    public function authorize(): bool
    {
        return Gate::allows('asset-deployment.create');
    }

    public function rules(): array
    {
        return [
            'asset_id'               => ['required','integer','exists:assets,id'],
            'target_department_id'   => ['required','integer','exists:departments,id'],
            'assigned_technician_id' => ['nullable','integer','exists:users,id'],
            'allocated_items'        => ['nullable','array'],
            'allocated_items.*.type' => ['required_with:allocated_items','in:COMPONENT,ACCESSORY,LICENSE'],
            'allocated_items.*.id'   => ['required_with:allocated_items','integer'],
            'allocated_items.*.qty'  => ['required_with:allocated_items','integer','min:1'],
        ];
    }

    public function messages(): array
    {
        return [
            'asset_id.required' => 'Kolom Asset Target wajib diisi.',
            'asset_id.exists'   => 'Asset TAG tidak terdaftar di sistem.',
            'allocated_items.*.qty.min' => 'Kuantitas minimal 1.',
        ];
    }
}
```

## 📄 8.2 `MaintenanceStoreRequest.php`

```php
<?php
// erickalvino-MODULE_MAINT: Form request penyimpanan draft maintenance
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Support\Facades\Gate;

class MaintenanceStoreRequest extends FormRequest
{
    public function authorize(): bool
    {
        return Gate::allows('custom-maintenance.create');
    }

    public function rules(): array
    {
        return [
            'asset_id'                  => ['required','integer','exists:assets,id'],
            'maintenance_type'          => ['required','in:INTERNAL,EXTERNAL'],
            'is_warranty_claim'         => ['required','boolean'],
            'supplier_id'               => ['required_if:maintenance_type,EXTERNAL','nullable','integer','exists:suppliers,id'],
            'assigned_technician_id'    => ['nullable','integer','exists:users,id'],
            'estimated_completion_date' => ['required_if:maintenance_type,EXTERNAL','nullable','date','after_or_equal:today'],
            'issue_description'         => ['required','string','min:10'],
            'cannibal_items'            => ['nullable','array'],
            'cannibal_items.*.donor_id'   => ['required_with:cannibal_items','integer','exists:assets,id'],
            'cannibal_items.*.type'       => ['required_with:cannibal_items','in:COMPONENT,ACCESSORY'],
            'cannibal_items.*.part_id'    => ['required_if:cannibal_items.*.is_unreg,0','required_with:cannibal_items','integer','exists:components,id'],
            'cannibal_items.*.is_unreg'   => ['required_with:cannibal_items','boolean'],
            'cannibal_items.*.part_name'  => ['required_if:cannibal_items.*.is_unreg,1','nullable','string','max:255'],
            'cannibal_items.*.part_spec'  => ['required_if:cannibal_items.*.is_unreg,1','nullable','string','max:500'],
            'cannibal_items.*.qty'        => ['required_with:cannibal_items','integer','min:1'],
        ];
    }

    public function withValidator($validator): void
    {
        $validator->after(function ($v) {
            if (! $v->errors()->isEmpty()) return;
            foreach ((array) $this->input('cannibal_items', []) as $item) {
                $donor = \App\Models\Asset::find($item['donor_id'] ?? null);
                if ($donor && $donor->status && ! $donor->status->archived) {
                    $v->errors()->add(
                        'cannibal_items',
                        "Donor asset {$donor->asset_tag} harus berstatus archived / UNREPAIRABLE."
                    );
                }
            }
        });
    }

    public function messages(): array
    {
        return [
            'asset_id.required' => 'Kolom Asset Target wajib diisi.',
            'maintenance_type.required' => 'Tipe perbaikan wajib diisi.',
            'supplier_id.required_if' => 'Vendor wajib diisi untuk tipe EXTERNAL.',
            'cannibal_items.*.part_name.required_if' => 'Nama part fisik wajib diisi untuk donor tidak terdaftar.',
            'cannibal_items.*.part_id.required_if' => 'ID komponen wajib diisi untuk donor terdaftar.',
        ];
    }
}
```

## 📄 8.3 `VoidRequest.php`

```php
<?php
// erickalvino-MODULE_CORE: Form request pembatalan/void — reason min 15 karakter
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class VoidRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return [
            'cancellation_notes' => ['required','string','min:15'],
        ];
    }

    public function messages(): array
    {
        return ['cancellation_notes.min' => 'Alasan pembatalan wajib minimal 15 karakter.'];
    }
}
```

## 📄 8.4 `UnrepairableRequest.php`

```php
<?php
// erickalvino-MODULE_MAINT: Form request penutupan BER
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class UnrepairableRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'unrepairable_reason_code' => ['required','in:TOTAL_FAILURE,SPAREPART_UNAVAILABLE,OTHER'],
            'unrepairable_notes'       => ['required','string','min:10'],
        ];
    }
}
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 9: SERVICE & DOMAIN ENGINE
# ------------------------------------------------------------------------------

> Prinsip: **controller tidak memuat logika bisnis.** Semua mutasi stok, karantina, nomor dokumen, dan audit log dibungkus di service.

## 📄 9.1 `StockReservationService.php`

```php
<?php
// erickalvino-MODULE_DEPLOY: Service reservasi stok — inti auto-release & concurrency
namespace App\Services\Custom;

use App\Models\AssetDeployment;
use App\Models\AssetDeploymentReservation;
use Illuminate\Support\Facades\DB;
use RuntimeException;
use Throwable;

class StockReservationService
{
    public function reserve(AssetDeployment $deployment): void
    {
        // operasi harus dalam transaksi dari caller
        foreach ($deployment->allocatedItems as $item) {
            // Lock baris stok agar aman dari race condition
            $this->lockStockRow($item->item_type, $item->item_id);

            $qty = $this->availableQty($item->item_type, $item->item_id);
            if ($qty < $item->qty) {
                throw new RuntimeException(
                    "Stok {$item->item_type} id {$item->item_id} tidak cukup (tersedia {$qty}, diperlukan {$item->qty})."
                );
            }

            AssetDeploymentReservation::updateOrCreate([
                'deployment_id' => $deployment->id,
                'item_type'     => $item->item_type,
                'item_id'       => $item->item_id,
            ], [
                'qty'          => $item->qty,
                'location_id'  => $deployment->location_id,
                'status'       => 'RESERVED',
                'created_by'   => auth()->id(),
                'released_at'  => null,
                'consumed_at'  => null,
            ]);
        }
    }

    public function release(AssetDeployment $deployment): void
    {
        $deployment->reservations()
            ->where('status', 'RESERVED')
            ->update(['status' => 'RELEASED', 'released_at' => now()]);
    }

    public function consume(AssetDeployment $deployment): void
    {
        $deployment->reservations()
            ->where('status', 'RESERVED')
            ->update(['status' => 'CONSUMED', 'consumed_at' => now()]);
    }

    private function lockStockRow(string $type, int $id): void
    {
        DB::table(match ($type) {
            'COMPONENT' => 'components',
            'ACCESSORY' => 'accessories',
            'LICENSE'   => 'licenses',
        })->whereKey($id)->lockForUpdate()->first();
    }

    private function availableQty(string $type, int $id): int
    {
        return match ($type) {
            'COMPONENT' => (int) DB::table('components')->whereKey($id)->value('qty'),
            'ACCESSORY' => (int) DB::table('accessories')->whereKey($id)->value('qty'),
            'LICENSE'   => (int) DB::table('licenses')->whereKey($id)->value('seats'),
            default     => 0,
        };
    }
}
```

## 📄 9.2 `BranchResolver` & `QuarantineService.php`

```php
<?php
// erickalvino-MODULE_MAINT: Service karantina terpusat berbasis config
namespace App\Services\Custom;

class QuarantineService
{
    public function quarantineStatusId(): int
    {
        $id = (int) config('custom.quarantine_status_id');
        if ($id <= 0) {
            throw new \RuntimeException('CUSTOM_QUARANTINE_STATUS_ID belum dikonfigurasi.');
        }
        return $id;
    }

    public function moveAssetToQuarantine($asset, int $locationId): void
    {
        $branch = BranchResolver::get($locationId);
        $status = $this->quarantineStatusId();

        $asset->status_id = $status;
        $asset->location_id = $branch['quarantine_location_id'];
        $asset->save();

        // Catat ke audit core Snipe-IT
        \App\Models\Actionlog::logAction($asset, 'CUSTOM_QUARANTINE', $branch['name']);
    }

    public function attachBrokenComponent($asset, $componentId, int $locationId, string $note): void
    {
        $branch = BranchResolver::get($locationId);
        $pivot = config('custom.pivot.component_asset', 'components_assets');

        \DB::table($pivot)->insert([
            'component_id'   => $componentId,
            'asset_id'       => $branch['quarantine_asset_id'],
            'assigned_to'    => null,
            'user_id'        => auth()->id(),
            'note'           => $note,
            'created_at'     => now(),
            'updated_at'     => now(),
        ]);
    }
}
```

## 📄 9.3 `DocumentNumberService.php`

```php
<?php
// erickalvino-MODULE_MAINT: Pembuatan nomor SJP/BAST non-hardcoded
namespace App\Services\Custom;

class DocumentNumberService
{
    public function next(string $module, int $locationId, int $recordId, string $prefix = null): string
    {
        $branch = BranchResolver::get($locationId);
        $prefix = $prefix ?: config("custom.document_prefix.{$module}", 'SJP');
        $year = date('Y');
        return "{$prefix}/{$branch['prefix']}/{$year}/".str_pad((string) $recordId, 5, '0', STR_PAD_LEFT);
    }
}
```

## 📄 9.4 `LicenseSeatHandler.php`

```php
<?php
// erickalvino-MODULE_DEPLOY: Handler lisensi — allocate/release seat dengan benar
namespace App\Services\Custom;

class LicenseSeatHandler
{
    public function assignSeat($licenseId, $asset): void
    {
        // Implementasi harus disesuaikan dengan skema license_seats / checkouts Snipe-IT target.
        // Wajib menggunakan tabel pivot resmi Snipe-IT, tidak boleh insert manual tanpa verifikasi kolom.
        // Jika di target tidak mendukung seat yang terikat aset, gunakan user assignee asset tsb.
        throw_if_empty_placeholder($licenseId, $asset);
    }

    public function releaseSeat($licenseId, $asset): void
    {
        // Sama: gunakan API/checkout Snipe-IT yang sudah tersedia, bukan update manual.
        throw_placeholder($licenseId, $asset);
    }
}
```

> Catatan: `LicenseSeatHandler` ditulis sebagai **kontrak**. Implementasi final harus divalidasi ke skema Snipe-IT target sebelum coding.

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 10: CONTROLLER (POLA & METHODE FINAL)
# ------------------------------------------------------------------------------

## 📄 10.1 Pola umum transition

```php
private function transition($model, string $newStatus, string $module, string $note): void
{
    $old = $model->status;
    if (! in_array($newStatus, $this->allowedNextStates($model), true)) {
        abort(422, "Transisi {$old} -> {$newStatus} tidak diizinkan.");
    }

    $model->status = $newStatus;
    $model->save();

    \App\Models\CustomModuleLog::create([
        'module_context' => $module,
        'record_id'      => $model->id,
        'old_status'     => $old,
        'new_status'     => $newStatus,
        'action_notes'   => $note,
        'user_id'        => auth()->id(),
        'ip_address'     => request()->ip(),
    ]);
}
```

## 📄 10.2 `AssetDeploymentsController` (ringkas)

```php
<?php
// erickalvino-MODULE_DEPLOY: Controller deployment — semua transisi POST
namespace App\Http\Controllers\Custom;

use App\Http\Controllers\Controller;
use App\Http\Requests\DeploymentStoreRequest;
use App\Http\Requests\VoidRequest;
use App\Models\AssetDeployment;
use App\Services\Custom\StockReservationService;
use App\Services\Custom\QuarantineService;
use Illuminate\Support\Facades\DB;

class AssetDeploymentsController extends Controller
{
    public function store(DeploymentStoreRequest $request)
    {
        $this->authorize('create', AssetDeployment::class);
        return DB::transaction(function () use ($request) {
            $asset = \App\Models\Asset::findOrFail($request->integer('asset_id'));

            if (! $asset->company_id || ! $asset->location_id) {
                abort(422, 'Asset target belum memiliki company/location, tidak bisa dibuat historical lock.');
            }

            $deployment = AssetDeployment::create([
                'asset_id'             => $asset->id,
                'company_id'           => $asset->company_id,
                'location_id'          => $asset->location_id,
                'target_department_id' => $request->integer('target_department_id'),
                'assigned_technician_id' => $request->integer('assigned_technician_id'),
                'status'               => 'DRAFT',
                'created_by'           => auth()->id(),
            ]);

            foreach ((array) $request->input('allocated_items', []) as $item) {
                $deployment->allocatedItems()->create([
                    'item_type' => $item['type'], 'item_id' => $item['id'],
                    'qty' => $item['qty'], 'created_by' => auth()->id(),
                ]);
            }

            $this->log($deployment, 'DEPLOYMENT', 'DRAFT', 'Inisiasi draft deployment.');
            return redirect()->route('assetDeployments.index')->with('success', 'Draft tersimpan.');
        });
    }

    // Semua transisi menggunakan POST + lockForUpdate + idempotency
    public function releaseToPending(int $id)
    {
        $deployment = AssetDeployment::lockForUpdate()->findOrFail($id);
        $this->authorize('edit', $deployment);
        $this->assertCreator($deployment);

        if ($deployment->status !== 'DRAFT') {
            abort(422, 'Hanya DRAFT yang bisa diterbitkan.');
        }

        return DB::transaction(function () use ($deployment) {
            (new StockReservationService)->reserve($deployment);
            (new \App\Services\Custom\DocumentNumberService)->uniqueNoIfNeeded($deployment, 'deployment');
            $deployment->status = 'PENDING_HANDOVER';
            $deployment->save();
            $this->log($deployment, 'DEPLOYMENT', 'DRAFT', 'Tiket diterbitkan & reservasi stok dibuat.');
        });
    }

    public function startProgress(int $id) { /* PENDING_HANDOVER -> IN_PROGRESS */ }
    public function markReady(int $id) { /* IN_PROGRESS -> READY_TO_RETURN */ }

    public function complete(int $id)
    {
        $deployment = AssetDeployment::lockForUpdate()->findOrFail($id);
        $this->authorize('approve', $deployment);
        if ($deployment->status !== 'READY_TO_RETURN') abort(422);

        return DB::transaction(function () use ($deployment) {
            $stock = new StockReservationService();
            $stock->consume($deployment);

            $asset = \App\Models\Asset::findOrFail($deployment->asset_id);

            foreach ($deployment->allocatedItems as $item) {
                match ($item->item_type) {
                    'COMPONENT' => $this->installComponent($asset, $item),
                    'ACCESSORY' => $this->installAccessory($asset, $item),
                    'LICENSE'   => (new \App\Services\Custom\LicenseSeatHandler)->assignSeat($item->item_id, $asset),
                    default => null,
                };
            }

            foreach ($deployment->detachedItems as $item) {
                if ($item->condition === 'GOOD') {
                    $this->returnToStock($item);
                } else {
                    (new QuarantineService)->attachBrokenComponent($asset, $item->item_id, $deployment->location_id, 'Copotan rusak');
                }
            }

            $deployment->status = 'COMPLETED';
            $deployment->save();
            $this->log($deployment, 'DEPLOYMENT', 'READY_TO_RETURN', 'Serah terima selesai.');
        });
    }

    public function cancelDeployment(int $id, VoidRequest $request)
    {
        $deployment = AssetDeployment::lockForUpdate()->findOrFail($id);
        $this->authorize('void', $deployment);

        return DB::transaction(function () use ($deployment, $request) {
            $deployment->status = 'CANCELLED';
            $deployment->cancellation_notes = $request->input('cancellation_notes');
            $deployment->save();
            (new StockReservationService)->release($deployment);
            $this->log($deployment, 'DEPLOYMENT', $deployment->getOriginal('status'), 'VOID: '.$request->input('cancellation_notes'));
        });
    }
}
```

## 📄 10.3 `CustomMaintenancesController` (ringkas)

```php
<?php
// erickalvino-MODULE_MAINT: Controller maintenance
namespace App\Http\Controllers\Custom;

use App\Http\Controllers\Controller;
use App\Http\Requests\MaintenanceStoreRequest;
use App\Http\Requests\UnrepairableRequest;
use App\Http\Requests\VoidRequest;
use App\Models\CustomMaintenance;
use App\Services\Custom\QuarantineService;
use App\Services\Custom\DocumentNumberService;
use Illuminate\Support\Facades\DB;

class CustomMaintenancesController extends Controller
{
    public function store(MaintenanceStoreRequest $request)
    {
        return DB::transaction(function () use ($request) {
            $asset = \App\Models\Asset::findOrFail($request->integer('asset_id'));
            if (! $asset->company_id || ! $asset->location_id) abort(422);

            $maintenance = CustomMaintenance::create([
                'asset_id' => $asset->id,
                'company_id' => $asset->company_id,
                'location_id' => $asset->location_id,
                'maintenance_type' => $request->input('maintenance_type'),
                'is_warranty_claim' => $request->boolean('is_warranty_claim'),
                'warranty_status_at_launch' => (new CustomMaintenance())->detectWarrantyStatus(),
                'supplier_id' => $request->integer('supplier_id'),
                'assigned_technician_id' => $request->integer('assigned_technician_id'),
                'status' => 'DRAFT',
                'estimated_completion_date' => $request->input('estimated_completion_date'),
                'issue_description' => $request->input('issue_description'),
                'created_by' => auth()->id(),
            ]);

            foreach ((array) $request->input('cannibal_items', []) as $item) {
                $maintenance->cannibalItems()->create([
                    'donor_asset_id' => $item['donor_id'],
                    'item_type' => $item['type'],
                    'item_id' => $item['is_unreg'] ? null : $item['part_id'],
                    'part_name' => $item['is_unreg'] ? $item['part_name'] : null,
                    'part_spec' => $item['is_unreg'] ? $item['part_spec'] : null,
                    'is_unregistered_donor' => $item['is_unreg'],
                    'qty' => $item['qty'],
                    'status' => 'PLANNED',
                    'created_by' => auth()->id(),
                ]);
            }

            $this->log($maintenance, 'MAINTENANCE', 'DRAFT', 'Inisiasi draft maintenance.');
        });
    }

    public function releaseToPendingChecking(int $id)
    {
        $maintenance = CustomMaintenance::lockForUpdate()->findOrFail($id);
        $this->authorize('edit', $maintenance);
        $this->assertCreator($maintenance);

        return DB::transaction(function () use ($maintenance) {
            if ($maintenance->maintenance_type === 'EXTERNAL') {
                $maintenance->document_number = (new DocumentNumberService)
                    ->next('maintenance', $maintenance->location_id, $maintenance->id);
            }
            $maintenance->status = 'PENDING_CHECKING';
            $maintenance->save();
            $this->log($maintenance, 'MAINTENANCE', 'DRAFT', 'Laporan diterbitkan.');
        });
    }

    // POST transitions: diagnosis, internal, vendor, qc, ready, complete, unrepairable, cancel

    public function complete(int $id)
    {
        $maintenance = CustomMaintenance::lockForUpdate()->findOrFail($id);
        $this->authorize('approve', $maintenance);
        if ($maintenance->status !== 'READY_TO_RETURN') abort(422);

        return DB::transaction(function () use ($maintenance) {
            $target = \App\Models\Asset::findOrFail($maintenance->asset_id);

            foreach ($maintenance->cannibalItems as $item) {
                if ($item->item_type === 'COMPONENT') {
                    if ($item->is_unregistered_donor) {
                        $this->logUnregisteredInstall($target, $item);
                    } else {
                        (new \App\Services\Custom\StockEngine)->consumeComponent($item->item_id, $item->qty);
                        $this->attachComponent($target, $item->item_id, 'Kanibalisasi resmi');
                    }
                    $item->status = 'APPLIED';
                    $item->save();
                }
            }

            $maintenance->status = 'COMPLETED';
            $maintenance->repair_action = $maintenance->repair_action ?: 'Perbaikan selesai.';
            $maintenance->save();
            $this->log($maintenance, 'MAINTENANCE', 'READY_TO_RETURN', 'Tiket selesai.');
        });
    }

    public function rejectToUnrepairable(int $id, UnrepairableRequest $request)
    {
        $maintenance = CustomMaintenance::lockForUpdate()->findOrFail($id);
        $this->authorize('approve', $maintenance);
        if ($maintenance->status !== 'READY_TO_RETURN') abort(422);

        return DB::transaction(function () use ($maintenance, $request) {
            $asset = \App\Models\Asset::findOrFail($maintenance->asset_id);
            (new QuarantineService)->moveAssetToQuarantine($asset, $maintenance->location_id);

            $maintenance->status = 'UNREPAIRABLE';
            $maintenance->unrepairable_reason_code = $request->input('unrepairable_reason_code');
            $maintenance->unrepairable_notes = $request->input('unrepairable_notes');
            $maintenance->save();
            $this->log($maintenance, 'MAINTENANCE', 'READY_TO_RETURN', 'Unit dinyatakan B.E.R.');
        });
    }
}
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 11: ROUTES FINAL (web + api)
# ------------------------------------------------------------------------------

## 📄 11.1 `routes/web.php` — semua transisi pakai POST

```php
<?php
// erickalvino-MODULE_DEPLOY & MODULE_MAINT: Routes final — transisi status via POST
use App\Http\Controllers\Custom\AssetDeploymentsController;
use App\Http\Controllers\Custom\CustomMaintenancesController;

Route::group(['middleware' => ['auth']], function () {
    Route::prefix('custom/asset-deployments')->name('assetDeployments.')->group(function () {
        Route::get('/', [AssetDeploymentsController::class, 'index'])->name('index');
        Route::get('create', [AssetDeploymentsController::class, 'create'])->name('create');
        Route::post('store', [AssetDeploymentsController::class, 'store'])->name('store');
        Route::get('{id}/edit', [AssetDeploymentsController::class, 'edit'])->name('edit');

        Route::post('{id}/release', [AssetDeploymentsController::class, 'releaseToPending'])->name('release');
        Route::post('{id}/start', [AssetDeploymentsController::class, 'startProgress'])->name('start');
        Route::post('{id}/ready', [AssetDeploymentsController::class, 'markReady'])->name('ready');
        Route::post('{id}/complete', [AssetDeploymentsController::class, 'complete'])->name('complete');
        Route::post('{id}/cancel', [AssetDeploymentsController::class, 'cancelDeployment'])->name('cancel');

        Route::get('{id}/print-pdf', [AssetDeploymentsController::class, 'printPdf'])->name('print-pdf');
    });

    Route::prefix('custom/maintenances')->name('customMaintenances.')->group(function () {
        Route::get('/', [CustomMaintenancesController::class, 'index'])->name('index');
        Route::get('create', [CustomMaintenancesController::class, 'create'])->name('create');
        Route::post('store', [CustomMaintenancesController::class, 'store'])->name('store');
        Route::get('{id}/edit', [CustomMaintenancesController::class, 'edit'])->name('edit');

        Route::post('{id}/release', [CustomMaintenancesController::class, 'releaseToPendingChecking'])->name('release');
        Route::post('{id}/diagnosis', [CustomMaintenancesController::class, 'acceptDiagnosis'])->name('diagnosis');
        Route::post('{id}/internal', [CustomMaintenancesController::class, 'startInternal'])->name('internal');
        Route::post('{id}/vendor', [CustomMaintenancesController::class, 'sendToVendor'])->name('vendor');
        Route::post('{id}/qc', [CustomMaintenancesController::class, 'postServiceQc'])->name('qc');
        Route::post('{id}/ready', [CustomMaintenancesController::class, 'readyToReturn'])->name('ready');
        Route::post('{id}/complete', [CustomMaintenancesController::class, 'complete'])->name('complete');
        Route::post('{id}/unrepairable', [CustomMaintenancesController::class, 'rejectToUnrepairable'])->name('unrepairable');
        Route::post('{id}/cancel', [CustomMaintenancesController::class, 'cancelMaintenance'])->name('cancel');

        Route::get('{id}/print-pdf', [CustomMaintenancesController::class, 'printPdf'])->name('print-pdf');
        Route::get('{id}/print-sticker', [CustomMaintenancesController::class, 'printSticker'])->name('print-sticker');
    });
});
```

## 📄 11.2 `routes/api.php`

```php
<?php
// erickalvino-MODULE_DEPLOY & MODULE_MAINT: Routes API JSON — multi-company context
use App\Http\Controllers\Api\AssetDeploymentsController;
use App\Http\Controllers\Api\CustomMaintenancesController;

Route::group([
    'prefix' => 'v1/custom',
    'middleware' => ['auth:api'],
], function () {
    Route::get('asset-deployments', [AssetDeploymentsController::class, 'index'])->name('api.assetDeployments.index');
    Route::get('maintenances', [CustomMaintenancesController::class, 'index'])->name('api.customMaintenances.index');
});
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 12: VIEWS (SPESIFIKASI)
# ------------------------------------------------------------------------------

## 📄 12.1 View yang wajib ada

| Modul | File |
|---|---|
| Deployment | `resources/views/asset-deployments/index.blade.php` |
| Deployment | `resources/views/asset-deployments/create.blade.php` |
| Deployment | `resources/views/asset-deployments/edit.blade.php` |
| Maintenance | `resources/views/custom-maintenances/index.blade.php` |
| Maintenance | `resources/views/custom-maintenances/create.blade.php` |
| Maintenance | `resources/views/custom-maintenances/edit.blade.php` |
| Prints | `resources/views/custom-maintenances/reports/print-pdf.blade.php` |
| Prints | `resources/views/custom-maintenances/reports/asset-status-sticker.blade.php` |

## 📄 12.2 Aturan view

1. Semua state-changing action memakai **form POST + `@csrf`**.
2. Semua tombol aksi harus di-gate dengan `@can`.
3. Kolom `branch_location` wajib muncul di index kedua modul.
4. Modal VOID harus memvalidasi `cancellation_notes >= 15` di client & server.
5. Untuk transisi antar status, gunakan tombol `POST` per status, **tidak boleh drop-down yang mengubah status via GET**.
6. Gunakan `trans('custom.*')` hanya setelah file translasi dibuat (Lihat Bagian 13).

## 📄 12.3 Contoh form `create.blade.php` Maintenance (field lengkap)

```blade
@extends('layouts/default')

{{-- erickalvino-MODULE_MAINT: Form maintenance lengkap sesuai FormRequest --}}
@section('title') {{ trans('custom.module_maintenance') }} @parent @stop

@section('content')
<div class="row">
  <div class="col-md-8 col-md-offset-2">
    <form class="form-horizontal" method="POST" action="{{ route('customMaintenances.store') }}">
      @csrf
      <div class="box box-default">
        <div class="box-header with-border">
          <h3 class="box-title">{{ trans('custom.form.create_maintenance') }}</h3>
        </div>
        <div class="box-body">

          {{-- Asset Target --}}
          <div class="form-group {{ $errors->has('asset_id') ? 'has-error' : '' }}">
            <label class="col-md-3 control-label">{{ trans('custom.field.asset_target') }}</label>
            <div class="col-md-7">
              <select class="form-control select2" name="asset_id" id="asset_id" required>
                @foreach($assets as $asset)
                  <option value="{{ $asset->id }}" {{ old('asset_id') == $asset->id ? 'selected' : '' }}>[{{ $asset->asset_tag }}] {{ $asset->name }}</option>
                @endforeach
              </select>
              {!! $errors->first('asset_id', '<span class="help-block">:message</span>') !!}
            </div>
          </div>

          {{-- Tipe --}}
          <div class="form-group">
            <label class="col-md-3 control-label">{{ trans('custom.field.maintenance_type') }}</label>
            <div class="col-md-7">
              <label><input type="radio" name="maintenance_type" value="INTERNAL" {{ old('maintenance_type','INTERNAL') === 'INTERNAL' ? 'checked' : '' }}> Internal</label>&nbsp;
              <label><input type="radio" name="maintenance_type" value="EXTERNAL" {{ old('maintenance_type') === 'EXTERNAL' ? 'checked' : '' }}> External</label>
            </div>
          </div>

          {{-- Supplier EXTERNAL --}}
          <div class="form-group {{ $errors->has('supplier_id') ? 'has-error' : '' }}" id="supplier_group">
            <label class="col-md-3 control-label">{{ trans('custom.field.supplier') }}</label>
            <div class="col-md-7">
              <select class="form-control select2" name="supplier_id">
                <option value="">--</option>
                @foreach($suppliers as $supplier)
                  <option value="{{ $supplier->id }}" {{ old('supplier_id') == $supplier->id ? 'selected' : '' }}>{{ $supplier->name }}</option>
                @endforeach
              </select>
              {!! $errors->first('supplier_id', '<span class="help-block">:message</span>') !!}
            </div>
          </div>

          {{-- Estimasi selesai --}}
          <div class="form-group">
            <label class="col-md-3 control-label">{{ trans('custom.field.estimated_date') }}</label>
            <div class="col-md-7">
              <input type="date" class="form-control" name="estimated_completion_date" value="{{ old('estimated_completion_date') }}">
            </div>
          </div>

          {{-- Isi issue --}}
          <div class="form-group {{ $errors->has('issue_description') ? 'has-error' : '' }}">
            <label class="col-md-3 control-label">{{ trans('custom.field.issue_description') }}</label>
            <div class="col-md-7">
              <textarea class="form-control" name="issue_description" rows="4" required>{{ old('issue_description') }}</textarea>
              {!! $errors->first('issue_description', '<span class="help-block">:message</span>') !!}
            </div>
          </div>

          {{-- Repeater kanibalisasi --}}
          <div class="form-group">
            <label class="col-md-3 control-label">{{ trans('custom.field.cannibal_items') }}</label>
            <div class="col-md-7">
              <div id="cannibalRepeater">
                <!-- render row per item; wajib punya donor_id, type, part_id/part_name, is_unreg, qty -->
              </div>
              <button type="button" class="btn btn-default btn-sm" id="addCannibalRow">{{ trans('custom.action.add_cannibal') }}</button>
            </div>
          </div>

        </div>
        <div class="box-footer text-right">
          <button type="submit" class="btn btn-success">{{ trans('general.save') }}</button>
        </div>
      </div>
    </form>
  </div>
</div>
@endsection
```

## 📄 12.4 Modal VOID (dipakai kedua modul)

```blade
<div class="modal fade" id="voidModal">
  <div class="modal-dialog">
    <form id="voidForm" method="POST">
      @csrf
      <div class="modal-content">
        <div class="modal-header"><h4>{{ trans('custom.modal.void_title') }}</h4></div>
        <div class="modal-body">
          <textarea class="form-control" name="cancellation_notes" id="cancellation_notes" rows="4" required></textarea>
          <span id="voidError" class="text-danger" style="display:none;">{{ trans('custom.modal.void_min') }}</span>
        </div>
        <div class="modal-footer">
          <button type="submit" class="btn btn-danger">{{ trans('general.submit') }}</button>
        </div>
      </div>
    </form>
  </div>
</div>
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 13: TRANSLASI (WAJIB ADA)
# ------------------------------------------------------------------------------

## 📄 13.1 Daftar key `lang/en/custom.php` (draft)

```php
<?php
// erickalvino-MODULE_CORE: Translation keys modul custom — English
return [
    'module_deployment' => 'Asset Deployment & Setup',
    'module_maintenance' => 'Custom Maintenance & Service',

    'field' => [
        'asset_target' => 'Asset Target',
        'asset_tag' => 'Asset TAG',
        'asset_name' => 'Asset Name',
        'branch' => 'Branch',
        'maintenance_type' => 'Maintenance Type',
        'supplier' => 'Supplier',
        'estimated_date' => 'Estimated Completion',
        'issue_description' => 'Issue Description',
        'repair_cost' => 'Cost',
        'document_number' => 'Document Number',
        'cannibal_items' => 'Cannibalized Parts',
    ],

    'status' => [
        'draft' => 'DRAFT',
        'pending_handover' => 'Pending Handover',
        'in_progress' => 'In Progress',
        'ready_to_return' => 'Ready to Return',
        'completed' => 'Completed',
        'cancelled' => 'Cancelled (VOID)',
        'pending_checking' => 'Pending Checking',
        'under_diagnosis' => 'Under Diagnosis',
        'in_service_internal' => 'Internal Service',
        'out_to_vendor' => 'Out to Vendor',
        'post_service_qc' => 'Post Service QC',
        'unrepairable' => 'Beyond Economic Repair',
    ],

    'action' => [
        'release' => 'Publish Ticket',
        'start' => 'Start Progress',
        'ready' => 'Mark Ready',
        'complete' => 'Approve Complete',
        'void' => 'Void',
        'diagnosis' => 'Accept Diagnosis',
        'internal' => 'Start Internal Service',
        'vendor' => 'Send to Vendor',
        'qc' => 'Post Service QC',
        'unrepairable' => 'Declare BER',
        'add_cannibal' => 'Add Cannibal Part',
        'print' => 'Print',
    ],

    'modal' => [
        'void_title' => 'Cancel / Void Document',
        'void_min' => 'Reason must be at least 15 characters.',
    ],

    'form' => [
        'create_maintenance' => 'Create Maintenance Ticket',
    ],
];
```

Di file `lang/id/custom.php`, isi nilai bahasa Indonesia dengan **key yang sama persis**.

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 14: PRESENTER & TRANSFORMER (FINAL)
# ------------------------------------------------------------------------------

## 📄 14.1 Presenter tetap dipertahankan
Presenters hanya memformat visual & angka. Semua teks badge dipakai dari translation key yang sudah ada (bukan `trans('custom.status_draft') ?? 'DRAFT'`).

```php
public function statusLabel(): string
{
    $class = match ($this->model->status) {
        'DRAFT' => 'label-default',
        'COMPLETED' => 'label-success',
        'UNREPAIRABLE' => 'label-danger',
        'CANCELLED' => 'label-danger',
        default => 'label-info',
    };

    return '<span class="label '.$class.'">' .
        e(Lang::has('custom.status.'.Str::snake($this->model->status))
            ? trans('custom.status.'.Str::snake($this->model->status))
            : $this->model->status) .
        '</span>';
}
```

## 📄 14.2 Transformer
- Memakai route name final (`assetDeployments.print-pdf`, `customMaintenances.print-pdf`).
- Hanya menampilkan tombol yang diizinkan policy.
- Tidak menghasilkan HTML untuk pembatalan via GET; selalu `data-url` untuk modal POST.

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 15: DASHBOARD & PELAPORAN
# ------------------------------------------------------------------------------

## 📄 15.1 Scope modul dashboard
1. **Total asset per status** per cabang & per company.
2. **Workload**: DRAFT/PENDING/IN_PROGRESS/READY.
3. **TCO**: jumlah repair_cost per asset per tahun per cabang.
4. **SLA**: rata-rata waktu dari DRAFT ke COMPLETED per cabang.
5. **Export CSV** dengan filter company/location/status/date.

## 📄 15.2 Implementasi
- Gunakan query builder dengan `scopeCompanyContext()` pada model.
- Semua filter `location_id` diambil dari route/query **bukan hardcoded**.
- Setiap laporan wajib menyertakan `company_id` & `location_id` historis.

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 16: NON-FUNCTIONAL REQUIREMENTS
# ------------------------------------------------------------------------------

## 📄 16.1 Keamanan
- Semua mutasi status via `POST` + CSRF.
- Semua perubahan lewat Policy/Gate.
- `cancellation_notes` & `unrepairable_notes` wajib diisi minimal sesuai config.
- Audit log membaca `ip_address`.

## 📄 16.2 Concurrency
- Semua transaksi stok dibungkus `DB::transaction` + `lockForUpdate`.
- Unique constraint mencegah reservasi ganda per `deployment_id + item_type + item_id`.

## 📄 16.3 Idempotency
- Endpoint transisi memvalidasi status sekarang; status terminal tidak bisa diubah.
- Jika request duplikat tiba, sistem menolak/return sukses tanpa double-decrement.

## 📄 16.4 Performa
- Composite index `(location_id, status)` & `(company_id, status)` di kedua tabel induk.
- Query dashboard memakai `with()` untuk menghindari N+1.

## 📄 16.5 Audit retention
- `custom_module_logs` tidak pernah di-update / delete secara manual.
- Retention default 5 tahun (bisa diubah via config).

## 📄 16.6 Error handling
- Semua service melempar `RuntimeException` / `ValidationException`.
- Controller menangani exception umum dan melakukan `rollback`.

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 17: PENGUJIAN (AUTOMATED + MANUAL)
# ------------------------------------------------------------------------------

## 📄 17.1 Factory yang wajib ada
- `Database\Factories\AssetDeploymentFactory`
- `Database\Factories\CustomMaintenanceFactory`
- `Database\Factories\AssetDeploymentReservationFactory`
- Pastikan factory `Asset`, `User`, `Component`, `Accessory`, `License` mengikuti skema Snipe-IT target.

## 📄 17.2 Feature test minimal

### `tests/Feature/AssetDeploymentEnterpriseTest.php`
```php
<?php
// erickalvino-MODULE_DEPLOY: Feature test deployment
namespace Tests\Feature;

use Tests\TestCase;
use App\Models\Asset;
use App\Models\AssetDeployment;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

class AssetDeploymentEnterpriseTest extends TestCase
{
    use RefreshDatabase;

    public function test_draft_release_blocked_for_non_creator()
    {
        $creator = User::factory()->create(['permissions' => json_encode(['asset-deployment.create' => 1])]);
        $other = User::factory()->create();

        $asset = Asset::factory()->create(['company_id' => 1, 'location_id' => 1]);
        $deployment = AssetDeployment::factory()->create([
            'asset_id' => $asset->id, 'company_id' => 1, 'location_id' => 1,
            'status' => 'DRAFT', 'created_by' => $creator->id,
        ]);

        $this->actingAs($other)->post(route('assetDeployments.release', $deployment->id))
            ->assertRedirect();

        $this->assertSame('DRAFT', $deployment->fresh()->status);
    }

    public function test_void_requires_15_characters()
    {
        $user = User::factory()->create(['permissions' => json_encode(['asset-deployment.void' => 1])]);
        $asset = Asset::factory()->create(['company_id' => 1, 'location_id' => 1]);
        $deployment = AssetDeployment::factory()->create([
            'asset_id' => $asset->id, 'company_id' => 1, 'location_id' => 1,
            'status' => 'READY_TO_RETURN', 'created_by' => $user->id,
        ]);

        $this->actingAs($user)->post(route('assetDeployments.cancel', $deployment->id), [
            'cancellation_notes' => 'salah input',
        ])->assertSessionHasErrors('cancellation_notes');

        $this->assertNotSame('CANCELLED', $deployment->fresh()->status);
    }

    public function test_complete_decrements_reserved_stock()
    {
        // create component qty 10, reserve via release, then complete and assert qty 8
    }

    public function test_cancel_releases_reserved_stock()
    {
        // reserve via release, void, assert reservation status = RELEASED
    }
}
```

## 📄 17.3 Matriks uji manual (black-box)

| ID | Sub-Sistem | Input | Expected |
|---|---|---|---|
| TC-01 | Creator Only | User pembuat + user lain akses release | User lain ditolak, status tetap DRAFT |
| TC-02 | Mandatory 15 chars | Void dengan alasan <15 | Client & server blok, status tidak berubah |
| TC-03 | Reservation engine | Terbitkan lalu void | Reservation berubah menjadi RELEASED |
| TC-04 | Complete stock | COMPLETED deployment dengan komponen qty 10 | qty berkurang sesuai alokasi |
| TC-05 | Dynamic quarantine | UNREPAIRABLE di cabang Bandung | Aset masuk status quarantine & lokasi dari config |
| TC-06 | License | Alokasi LICENSE saat COMPLETED | Seat ter-assign / terpinjam via handler |
| TC-07 | Accessory | Pasang accessory ke asset | Dicatat di `asset_accessory_installs`, bukan pivot user |
| TC-08 | Multi-company | User company A lihat data company B | Data tidak muncul |
| TC-09 | CSV export | Export dashboard dengan filter cabang | Kolom branch & TCO sesuai |
| TC-10 | Idempotency | Klik complete 2x | Decrement hanya terjadi 1x |

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 18: ROADMAP IMPLEMENTASI & CHECKLIST ARSITEK
# ------------------------------------------------------------------------------

## 📄 18.1 Fase A — Fondasi
- [x] State machine & aturan bisnis
- [ ] Konfigurasi config/env & resolver
- [ ] Migration final
- [ ] Validator config boot
- [ ] Registrasi service provider

## 📄 18.2 Fase B — Domain
- [ ] Models & relations
- [ ] Policy & permission matrix
- [ ] Reservation & stock engine
- [ ] Quarantine service
- [ ] Document number service
- [ ] License & accessory handler
- [ ] Controllers + routes POST

## 📄 18.3 Fase C — UI
- [ ] View deployment (index/create/edit)
- [ ] View maintenance lengkap
- [ ] Modal VOID + validasi client
- [ ] Presenter & transformer final
- [ ] File translasi EN/ID

## 📄 18.4 Fase D — Quality
- [ ] Factories
- [ ] Feature test deployment + maintenance
- [ ] QC manual matrix (TC-01..TC-10)
- [ ] Validasi schema terhadap instalasi Snipe-IT target
- [ ] Rename lokal & dokumentasi release

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 19: PERTANYAAN TERBUKA (SEBELUM CODING FINAL)
# ------------------------------------------------------------------------------

1. **Versi Snipe-IT target** benar-benar v6 / v7 mana? Ini menentukan nama tabel pivot & field.
2. Apakah `company_id` & `location_id` pada Snipe-IT **selalu terisi**? Jika tidak, perlu fallback config `custom.default_company_id` / `custom.default_location_id` yang eksplisit.
3. Apakah aksesoris **boleh** dipasang langsung ke aset Snipe-IT, atau hanya ke user? Ini memengaruhi pivot custom di Bagian 5.8.
4. Apakah license di-assign ke **user** atau bisa ke **asset**? License handler harus memakai API resmi Snipe-IT.
5. Nomor dokumen `SJP`/`BAST` harus **unique per cabang & tahun**, sudah dihandle `document_number` unique. Perlu dipastikan kasus duplicate number.
6. Apakah perlu kolom **asset donor** di maintenance dipindah ke lokasi karantina **sebelum** `CANCELLED`? Perlu kebijakan eksplisit.
7. Apakah ada kebutuhan **notifikasi email** saat status berubah? Belum ada di scope v2.

---

# ==============================================================================
# 🎯 KESIMPULAN REVISI 2.0
# ==============================================================================

Dokumen ini menggantikan pendekatan v1 yang **banyak hardcoded** dengan desain **configuration-driven** dan **service layer**:
- Tidak ada ID/code cabang yang dikunci di kode.
- Stok tidak lagi "dikunci secara komentar"; melainkan melalui **tabel reservasi** & `lockForUpdate`.
- Semua transisi status menggunakan **POST** dan didukung **Policy**.
- Semua artefak pendukung (config, env, policy, view, translasi, factory, test) sudah didefinisikan sebagai deliverable.

**Status:** DRAFT UNTUK REVIEW. Sebelum produksi, konfirmasi Bagian 19 (versi Snipe-IT & kebijakan license/accessory) lalu isi checklist Fase A–D.
