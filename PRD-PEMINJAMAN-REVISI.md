# ==============================================================================
# PRODUCT REQUIREMENT DOCUMENT (PRD) — REVISI 2.1
# MODUL PEMINJAMAN ASET INTERNAL (ASSET LOAN & REQUEST) — MULTI-CABANG ENTERPRISE
# PLATFORM: SNIPE-IT (TARGET VERSION: v8.6.3 — https://github.com/grokability/snipe-it/tree/v8.6.3)
# TANDA TANGAN KODE: erickalvino
# ==============================================================================

# ------------------------------------------------------------------------------
# 📌 BAGIAN 0: METADATA REVISI & TRACEABILITY TERHADAP REVIEW
# ------------------------------------------------------------------------------

## 📄 0.1 Dokumen yang di-revisi
- **Sumber:** `PRD-PEMINJAMAN.md` (revisi lama, 20 bagian)
- **Hasil review:** `REVIEW-PRD-PEMINJAMAN.md`
- **Versi target:** `grokability/snipe-it` tag `v8.6.3` (PHP `^8.2`, Laravel `^12.0`)
- **Keputusan pengguna yang sudah dikonfirmasi:**
  - Accessories & licenses dapat di-assign ke **user maupun asset**.
  - `company_id` / `location_id` pada aset Snipe-IT **tidak selalu terisi (nullable)**.

## 📄 0.2 Daftar perbaikan utama (traceability ke review)

| ID | Masalah di PRD lama | Perbaikan di PRD revisi ini |
|---|---|---|
| R-01 | `completeReturn()` tidak memblokir `MISSING/EXCHANGED` | Ditambahkan status `PENDING_COMPENSATION` + tabel `custom_loan_compensations` |
| R-02 | Tidak ada transisi `ON_LOAN → PENDING_RETURN_QC` | Ditambahkan `markReturnReceived()` + route POST |
| R-03 | Tidak ada `CANCEL` | Ditambahkan `cancelLoan()` + `VoidRequest` + auto-release |
| R-04 | Banyak hardcoded ID/code | Dihapus; memakai `BranchResolver`, config `.env`, dan service terpusat |
| R-05 | Form tidak sinkron dengan FormRequest | Field `items.*.<type>_id` diseragamkan dengan validasi |
| R-06 | `module_context = 'MAINTENANCE'` untuk log loan | Ditambahkan enum `LOAN` pada `custom_module_logs` |
| R-07 | Pivot `component_assets` salah | Diganti `components_assets` sesuai v8.6.3 |
| R-08 | License placeholder | Ditambahkan `LicenseSeatHandler` berbasis `license_seats` |
| R-09 | Tidak ada reservasi / race condition | Ditambahkan tabel `custom_asset_loan_reservations` + `lockForUpdate()` |
| R-10 | Status "Dipinjam" hardcoded | Ditambahkan `CUSTOM_LOAN_STATUS_LABEL_ID` + `LoanStatusResolver` |
| R-11 | `edit-checking.blade.php` rusak | Ditulis ulang lengkap tanpa hardcode status |
| R-12 | Routing/view/presenter belum ada | Ditambahkan routing final + index view + presenter |
| R-13 | Policy belum terdaftar | Ditambahkan policy + registrasi + permission matrix |
| R-14 | `parent_asset_id` salah disebut recursive | Didefinisikan sebagai **komposit anak aset induk** (bukan self-reference) |
| R-15 | Test tidak valid | Ditulis ulang test menggunakan `$this`, POST route, dan skema v8.6.3 |

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 1: RINGKASAN EKSEKUTIF, RUANG LINGKUP & ASUMSI
# ------------------------------------------------------------------------------

## 📄 1.1 Ringkasan Eksekutif

Modul Peminjaman Aset Internal berfungsi untuk mengelola peminjaman aset ber-TAG dan item komposit (aksesori, komponen, lisensi) lintas cabang dengan **kontrol birokrasi** dan **pelacakan fisik berangkat/pulang**.

Perubahan fundamental pada revisi ini:
1. **Zero hardcoded** — seluruh cabang, status karantina, status "Dipinjam", dan kode dokumen berasal dari config/`.env` atau database.
2. **Stok tidak langsung dipotong saat PENDING_APPROVAL** — stok di-*reserve*, lalu dipotong/dikonsumsi saat `ON_LOAN`, dan dikembalikan saat `RETURNED` (MATCH + GOOD).
3. **Ganti rugi menjadi alur resmi** — `MISSING/EXCHANGED` memindahkan dokumen ke `PENDING_COMPENSATION`, bukan langsung menutup `RETURNED`.
4. **Seluruh log memakai `module_context = 'LOAN'`** agar tidak tercampur dengan maintenance.
5. **Disesuaikan dengan skema Snipe-IT v8.6.3** — `components_assets`, `accessories_checkout`, `license_seats`. Tidak membuat tabel pivot kustom untuk aksesoris/lisensi kecuali untuk kebutuhan logistik internal.

## 📄 1.2 Ruang Lingkup

- 1 Kantor Pusat + 2 Kantor Cabang (default Jakarta, Bandung, Surabaya — melalui `config/custom.php`, bukan hardcoded).
- Item yang dapat dipinjam: `asset`, `accessory`, `component`, `license`.
- Paket komposit: item dapat memiliki `parent_asset_id` yang menunjuk ke aset induk (mis. laptop) untuk mencatat silsilah bawaan.

## 📄 1.3 Asumsi Teknis (TERKONFIRMASI PADA v8.6.3)

| Asumsi | Nilai |
|---|---|
| Repo & tag | `grokability/snipe-it` tag `v8.6.3` |
| PHP / Laravel | `^8.2` / `^12.0` |
| Asset `company_id` / `location_id` | **nullable**; modul memakai `CUSTOM_DEFAULT_*` bila kosong |
| Component pivot | `components_assets` |
| Accessory checkout | `accessories_checkout` (`assigned_to` + `assigned_type` = User / Asset / Location) |
| License seats | `license_seats` (`assigned_to` = user, `asset_id` = asset) |
| Log audit custom | `custom_module_logs.module_context` perlu menambah nilai `'LOAN'` |

## 📄 1.4 Konvensi penamaan
- Route name: `customLoans.*`
- URL prefix: `/custom/loans`
- Comments: `// erickalvino-MODULE_LOAN: ...`

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 2: ATURAN BISNIS (GOLDEN RULES)
# ------------------------------------------------------------------------------

1. **No hardcoded.** Semua ID/code/status label keluar dari config/env/database.
2. **Historical lock.** `company_id` & `location_id` disimpan fisik di induk & item, tidak diubah setelah dokumen keluar dari `DRAFT`.
3. **Creator Only (DRAFT).** Draf hanya bisa diubah/rilis/dihapus oleh pembuat dokumen (atau Superadmin).
4. **Reservasi kunci stok saat PENDING_APPROVAL**, **konsumsi saat ON_LOAN**, **restock saat RETURNED (MATCH+GOOD)**.
5. **Double booking guard.** Aset ber-TAG tidak boleh berada di lebih dari satu dokumen aktif (`PENDING_APPROVAL`, `ON_LOAN`, `PENDING_RETURN_QC`).
6. **QC wajib.** Dokumen tidak bisa langsung `RETURNED` dari `ON_LOAN`; wajib melewati `PENDING_RETURN_QC`.
7. **Compensation block.** Item `MISSING` / `EXCHANGED` memblokir penutupan hingga `custom_loan_compensations` diterbitkan & disetujui.
8. **Cancellation reason.** `cancellation_notes` minimal 15 karakter.
9. **Audit trail.** Setiap transisi status dicatat di `custom_module_logs` dengan `module_context = 'LOAN'`.
10. **Idempotency & concurrency.** Semua transaksi stok memakai `DB::transaction` + `lockForUpdate()`.

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 3: STATE MACHINE FINAL
# ------------------------------------------------------------------------------

## 📄 3.1 Diagram status

```text
[DRAFT] --release--> [PENDING_APPROVAL] --approve--> [ON_LOAN]
   |                       |                              |
   |  (creator edit/void)  |    (auto-release)            | return received
   |                       |                              v
   |                       +--void--> [CANCELLED]   [PENDING_RETURN_QC]
   |                                                         |
   |                                                         +-- all MATCH & GOOD --> [RETURNED]
   |                                                         +-- any MISSING/EXCHANGED --> [PENDING_COMPENSATION]
   |                                                         +-- BROKEN --> [RETURNED] (stok tidak dikembalikan)
   |                                                                              |
   |                                                         [PENDING_COMPENSATION] --BA issued-> [RETURNED]
   |
[ON_LOAN / PENDING_RETURN_QC / PENDING_COMPENSATION] --void--> [CANCELLED]
```

## 📄 3.2 Matriks transisi & authorization

| Dari status | Aksi | Ke status | Aktor | Method | Route |
|---|---|---|---|---|---|
| DRAFT | release | PENDING_APPROVAL | Creator/FA | `releaseToPendingApproval` | `customLoans.release` |
| PENDING_APPROVAL | approve | ON_LOAN | Supervisor/Approve | `approveToOnLoan` | `customLoans.approve-loan` |
| ON_LOAN | return received | PENDING_RETURN_QC | FA/IT | `markReturnReceived` | `customLoans.return-received` |
| PENDING_RETURN_QC | complete | RETURNED | FA/IT (QC) | `completeReturn` | `customLoans.complete-return` |
| PENDING_RETURN_QC | compensation | PENDING_COMPENSATION | FA (auto) | (dipicu `completeReturn`) | — |
| PENDING_COMPENSATION | issue & approve | RETURNED | FA/Supervisor | `issueCompensation` | `customLoans.compensation` |
| semua non-terminal | void | CANCELLED | FA/Supervisor | `cancelLoan` | `customLoans.cancel` |

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 4: KONFIGURASI & ENV (PENGGANTI HARCODING)
# ------------------------------------------------------------------------------

## 📄 4.1 `config/custom.php` — tambahan modul loan

```php
<?php
// erickalvino-MODULE_LOAN: Konfigurasi peminjaman tanpa ID hardcoded
return [
    // Default bila company/location pada asset atau operator kosong
    'default_company_id' => (int) env('CUSTOM_DEFAULT_COMPANY_ID', 0),
    'default_location_id'=> (int) env('CUSTOM_DEFAULT_LOCATION_ID', 0),

    // Status "Dipinjam" pada assets saat ON_LOAN
    'loan_status_label_id' => (int) env('CUSTOM_LOAN_STATUS_LABEL_ID', 0),

    // Status karantina rusak (dipakai bila kondisi return BROKEN)
    'quarantine_status_id' => (int) env('CUSTOM_QUARANTINE_STATUS_ID', 0),

    // Prefix dokumen peminjaman
    'document_prefix' => ['loan' => 'LP'],

    // Cabang (lihat PRD-DEPLOY-SERVICE-REVISI.md; disini hanya merujuk)
    // 'branches' => [...],
];
```

## 📄 4.2 `.env` sample

```env
CUSTOM_DEFAULT_COMPANY_ID=0
CUSTOM_DEFAULT_LOCATION_ID=0
CUSTOM_LOAN_STATUS_LABEL_ID=0
CUSTOM_QUARANTINE_STATUS_ID=0
```

## 📄 4.3 `BranchResolver` (loan-aware)

```php
<?php
// erickalvino-MODULE_LOAN: Resolver lokasi & default untuk modul loan
namespace App\Services\Custom;

use RuntimeException;

class BranchResolver
{
    public static function get(int $locationId): array
    {
        $config = config("custom.branches.{$locationId}");
        if (! is_array($config)) {
            throw new RuntimeException("Konfigurasi cabang location_id {$locationId} tidak ditemukan.");
        }
        return $config;
    }

    public static function fromAssetOrOperator($asset, $operator): array
    {
        $companyId = (int) ($asset->company_id ?? ($operator->company_id ?? 0));
        $locationId = (int) ($asset->location_id ?? ($operator->location_id ?? 0));

        if ($companyId <= 0) {
            $companyId = (int) config('custom.default_company_id', 0);
        }
        if ($locationId <= 0) {
            $locationId = (int) config('custom.default_location_id', 0);
        }

        if ($companyId <= 0) {
            throw new RuntimeException('CUSTOM_DEFAULT_COMPANY_ID belum diisi dan aset/operator tidak memiliki company_id.');
        }
        if ($locationId <= 0) {
            throw new RuntimeException('CUSTOM_DEFAULT_LOCATION_ID belum diisi dan aset/operator tidak memiliki location_id.');
        }

        self::get($locationId);
        return ['company_id' => $companyId, 'location_id' => $locationId];
    }
}
```

## 📄 4.4 `LoanStatusResolver`

```php
<?php
// erickalvino-MODULE_LOAN: Resolver status "Dipinjam" pada asset
namespace App\Services\Custom;

class LoanStatusResolver
{
    public static function loanStatusId(): int
    {
        $id = (int) config('custom.loan_status_label_id', 0);
        if ($id <= 0) {
            throw new \RuntimeException('CUSTOM_LOAN_STATUS_LABEL_ID wajib diisi.');
        }
        return $id;
    }
}
```

## 📄 4.5 Validator config saat boot

```php
// di dalam service provider
private function validateCustomConfig(): void
{
    if ((int) config('custom.default_company_id') <= 0) throw new \RuntimeException('[LOAN] CUSTOM_DEFAULT_COMPANY_ID wajib diisi.');
    if ((int) config('custom.default_location_id') <= 0) throw new \RuntimeException('[LOAN] CUSTOM_DEFAULT_LOCATION_ID wajib diisi.');
    if ((int) config('custom.loan_status_label_id') <= 0) throw new \RuntimeException('[LOAN] CUSTOM_LOAN_STATUS_LABEL_ID wajib diisi.');
    if ((int) config('custom.quarantine_status_id') <= 0) throw new \RuntimeException('[LOAN] CUSTOM_QUARANTINE_STATUS_ID wajib diisi.');
}
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 5: SKEMA DATABASE (MIGRASI FINAL)
# ------------------------------------------------------------------------------

## 📄 5.1 `2026_09_06_200001_create_custom_asset_loans_table.php`

```php
<?php
// erickalvino-MODULE_LOAN: Tabel induk peminjaman
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('custom_asset_loans', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('document_number', 100)->nullable()->unique();
            $table->unsignedBigInteger('company_id')->index();
            $table->unsignedBigInteger('location_id')->index();
            $table->unsignedBigInteger('borrower_id')->index();
            $table->unsignedBigInteger('created_by')->index();
            $table->unsignedBigInteger('approved_by')->nullable()->index();

            $table->enum('status', [
                'DRAFT','PENDING_APPROVAL','ON_LOAN',
                'PENDING_RETURN_QC','PENDING_COMPENSATION',
                'RETURNED','CANCELLED',
            ])->default('DRAFT')->index();

            $table->date('planned_checkout_date');
            $table->date('planned_return_date');
            $table->dateTime('actual_checkout_date')->nullable();
            $table->dateTime('actual_return_date')->nullable();
            $table->text('cancellation_notes')->nullable();
            $table->unsignedBigInteger('reservation_token_id')->nullable()->index();
            $table->timestamps();
            $table->softDeletes();

            $table->index(['location_id', 'status'], 'idx_loan_location_status');
            $table->index(['company_id', 'status'], 'idx_loan_company_status');

            $table->foreign('company_id', 'fk_loan_company')->references('id')->on('companies')->onDelete('restrict');
            $table->foreign('location_id', 'fk_loan_location')->references('id')->on('locations')->onDelete('restrict');
            $table->foreign('borrower_id', 'fk_loan_borrower')->references('id')->on('users')->onDelete('restrict');
            $table->foreign('created_by', 'fk_loan_creator')->references('id')->on('users')->onDelete('restrict');
            $table->foreign('approved_by', 'fk_loan_approver')->references('id')->on('users')->onDelete('set null');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('custom_asset_loans');
    }
};
```

## 📄 5.2 `2026_09_06_200002_create_custom_asset_loan_items_table.php`

```php
<?php
// erickalvino-MODULE_LOAN: Tabel item komposit peminjaman
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('custom_asset_loan_items', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->unsignedBigInteger('asset_loan_id')->index();
            $table->unsignedBigInteger('company_id')->index();
            $table->unsignedBigInteger('location_id')->index();

            $table->enum('item_type', ['asset', 'accessory', 'component', 'license'])->index();
            $table->unsignedBigInteger('asset_id')->nullable()->index();
            $table->unsignedBigInteger('parent_asset_id')->nullable()->index() // aset induk komposit
                ->comment('Aset TAG induk tempat accessory/component/license menempel.');
            $table->unsignedBigInteger('accessory_id')->nullable()->index();
            $table->unsignedBigInteger('component_id')->nullable()->index();
            $table->unsignedBigInteger('license_id')->nullable()->index();

            $table->unsignedInteger('qty_borrowed')->default(1);
            $table->unsignedInteger('qty_returned')->default(0);
            $table->unsignedBigInteger('initial_status_label_id')->nullable()->index();
            $table->unsignedBigInteger('final_status_label_id')->nullable()->index();
            $table->enum('return_check_status', ['MATCH', 'MISSING', 'EXCHANGED'])->default('MATCH')->index();
            $table->enum('condition_at_return', ['GOOD', 'BROKEN'])->default('GOOD')->index();
            $table->text('notes')->nullable();
            $table->timestamps();

            $table->index(['asset_loan_id', 'item_type'], 'idx_loan_item_type');

            $table->foreign('asset_loan_id', 'fk_loan_item_loan')->references('id')->on('custom_asset_loans')->onDelete('cascade');
            $table->foreign('asset_id', 'fk_loan_item_asset')->references('id')->on('assets')->onDelete('restrict');
            $table->foreign('parent_asset_id', 'fk_loan_item_parent')->references('id')->on('assets')->onDelete('restrict');
            $table->foreign('accessory_id', 'fk_loan_item_accessory')->references('id')->on('accessories')->onDelete('restrict');
            $table->foreign('component_id', 'fk_loan_item_component')->references('id')->on('components')->onDelete('restrict');
            $table->foreign('license_id', 'fk_loan_item_license')->references('id')->on('licenses')->onDelete('restrict');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('custom_asset_loan_items');
    }
};
```

## 📄 5.3 `2026_09_06_200003_create_custom_asset_loan_reservations_table.php`

```php
<?php
// erickalvino-MODULE_LOAN: Tabel reservasi stok peminjaman
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('custom_asset_loan_reservations', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->unsignedBigInteger('asset_loan_id')->index();
            $table->enum('item_type', ['asset', 'accessory', 'component', 'license'])->index();
            $table->unsignedBigInteger('item_id')->index();
            $table->unsignedInteger('qty')->default(1);
            $table->unsignedBigInteger('location_id')->index();
            $table->enum('status', ['RESERVED', 'CONSUMED', 'RELEASED', 'RESTOCKED'])->default('RESERVED')->index();
            $table->timestamp('reserved_at')->nullable()->useCurrent();
            $table->timestamp('consumed_at')->nullable();
            $table->timestamp('released_at')->nullable();
            $table->timestamp('restocked_at')->nullable();
            $table->unsignedBigInteger('created_by')->index();
            $table->timestamps();

            $table->unique(['asset_loan_id', 'item_type', 'item_id'], 'uniq_loan_reservation_item');

            $table->foreign('asset_loan_id', 'fk_loan_res_loan')->references('id')->on('custom_asset_loans')->onDelete('cascade');
            $table->foreign('location_id', 'fk_loan_res_location')->references('id')->on('locations')->onDelete('restrict');
            $table->foreign('created_by', 'fk_loan_res_creator')->references('id')->on('users')->onDelete('restrict');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('custom_asset_loan_reservations');
    }
};
```

## 📄 5.4 `2026_09_06_200004_create_custom_loan_compensations_table.php`

```php
<?php
// erickalvino-MODULE_LOAN: Tabel berita acara ganti rugi
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('custom_loan_compensations', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->unsignedBigInteger('asset_loan_id')->index();
            $table->unsignedBigInteger('loan_item_id')->nullable()->index();
            $table->enum('item_type', ['asset', 'accessory', 'component', 'license'])->index();
            $table->unsignedBigInteger('item_id')->nullable()->index();
            $table->unsignedInteger('missing_qty')->default(1);
            $table->decimal('amount', 20, 2)->default(0);
            $table->string('currency', 3)->default('IDR');
            $table->enum('status', ['DRAFT', 'ISSUED', 'SETTLED', 'DISPUTED'])->default('DRAFT')->index();
            $table->string('ba_number', 100)->nullable()->unique();
            $table->text('notes')->nullable();
            $table->unsignedBigInteger('created_by')->index();
            $table->timestamps();
            $table->softDeletes();

            $table->foreign('asset_loan_id', 'fk_comp_loan')->references('id')->on('custom_asset_loans')->onDelete('cascade');
            $table->foreign('created_by', 'fk_comp_creator')->references('id')->on('users')->onDelete('restrict');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('custom_loan_compensations');
    }
};
```

## 📄 5.5 Migrasi tambahan `module_context` pada `custom_module_logs`

```php
<?php
// erickalvino-MODULE_LOAN: Tambahkan konteks LOAN ke custom_module_logs
use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;

return new class extends Migration
{
    public function up(): void
    {
        // MySQL / MariaDB
        DB::statement("ALTER TABLE custom_module_logs MODIFY module_context ENUM('DEPLOYMENT','MAINTENANCE','LOAN') NOT NULL");
    }

    public function down(): void
    {
        DB::statement("ALTER TABLE custom_module_logs MODIFY module_context ENUM('DEPLOYMENT','MAINTENANCE') NOT NULL");
    }
};
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 6: MODEL & RELASI
# ------------------------------------------------------------------------------

## 📄 6.1 `CustomAssetLoan.php`

```php
<?php
// erickalvino-MODULE_LOAN: Model induk peminjaman
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use App\Models\Traits\Presentable;

class CustomAssetLoan extends Model
{
    use SoftDeletes;
    use Presentable;

    protected $table = 'custom_asset_loans';
    protected $presenter = \App\Presenters\CustomAssetLoanPresenter::class;

    protected $fillable = [
        'document_number','company_id','location_id','borrower_id','created_by','approved_by',
        'status','planned_checkout_date','planned_return_date',
        'actual_checkout_date','actual_return_date','cancellation_notes','reservation_token_id',
    ];

    protected $casts = [
        'company_id'=>'integer','location_id'=>'integer','borrower_id'=>'integer',
        'created_by'=>'integer','approved_by'=>'integer',
        'planned_checkout_date'=>'date:Y-m-d','planned_return_date'=>'date:Y-m-d',
        'actual_checkout_date'=>'datetime','actual_return_date'=>'datetime',
    ];

    public function company() { return $this->belongsTo(\App\Models\Company::class, 'company_id'); }
    public function location() { return $this->belongsTo(\App\Models\Location::class, 'location_id'); }
    public function borrower() { return $this->belongsTo(\App\Models\User::class, 'borrower_id'); }
    public function creator() { return $this->belongsTo(\App\Models\User::class, 'created_by'); }
    public function approver() { return $this->belongsTo(\App\Models\User::class, 'approved_by'); }
    public function items() { return $this->hasMany(\App\Models\CustomAssetLoanItem::class, 'asset_loan_id'); }
    public function reservations() { return $this->hasMany(\App\Models\CustomAssetLoanReservation::class, 'asset_loan_id'); }
    public function compensations() { return $this->hasMany(\App\Models\CustomLoanCompensation::class, 'asset_loan_id'); }

    public function scopeCompanyContext($query)
    {
        return $query->where(function ($q) {
            $user = auth()->user();
            if (! $user || $user->isSuperUser()) return $q;
            if (! empty($user->company_id)) $q->where('company_id', $user->company_id);
            return $q;
        });
    }
}
```

## 📄 6.2 `CustomAssetLoanItem.php`

```php
<?php
// erickalvino-MODULE_LOAN: Model item komposit peminjaman
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class CustomAssetLoanItem extends Model
{
    protected $table = 'custom_asset_loan_items';

    protected $fillable = [
        'asset_loan_id','company_id','location_id','item_type',
        'asset_id','parent_asset_id','accessory_id','component_id','license_id',
        'qty_borrowed','qty_returned','initial_status_label_id','final_status_label_id',
        'return_check_status','condition_at_return','notes',
    ];

    protected $casts = [
        'asset_loan_id'=>'integer','company_id'=>'integer','location_id'=>'integer',
        'asset_id'=>'integer','parent_asset_id'=>'integer','accessory_id'=>'integer',
        'component_id'=>'integer','license_id'=>'integer','qty_borrowed'=>'integer',
        'qty_returned'=>'integer','initial_status_label_id'=>'integer','final_status_label_id'=>'integer',
    ];

    public function loan() { return $this->belongsTo(\App\Models\CustomAssetLoan::class, 'asset_loan_id'); }
    public function asset() { return $this->belongsTo(\App\Models\Asset::class, 'asset_id'); }
    public function parentAsset() { return $this->belongsTo(\App\Models\Asset::class, 'parent_asset_id'); }
    public function accessory() { return $this->belongsTo(\App\Models\Accessory::class, 'accessory_id'); }
    public function component() { return $this->belongsTo(\App\Models\Component::class, 'component_id'); }
    public function license() { return $this->belongsTo(\App\Models\License::class, 'license_id'); }
}
```

## 📄 6.3 `CustomAssetLoanReservation.php` & `CustomLoanCompensation.php`

```php
<?php
// erickalvino-MODULE_LOAN: Model reservasi & kompensasi
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class CustomAssetLoanReservation extends Model
{
    protected $table = 'custom_asset_loan_reservations';
    protected $fillable = [
        'asset_loan_id','item_type','item_id','qty','location_id','status',
        'reserved_at','consumed_at','released_at','restocked_at','created_by',
    ];
    protected $casts = [
        'asset_loan_id'=>'integer','item_id'=>'integer','qty'=>'integer',
        'location_id'=>'integer','created_by'=>'integer',
        'reserved_at'=>'datetime','consumed_at'=>'datetime',
        'released_at'=>'datetime','restocked_at'=>'datetime',
    ];
    public function loan() { return $this->belongsTo(\App\Models\CustomAssetLoan::class, 'asset_loan_id'); }
}

class CustomLoanCompensation extends Model
{
    use \Illuminate\Database\Eloquent\SoftDeletes;
    protected $table = 'custom_loan_compensations';
    protected $fillable = [
        'asset_loan_id','loan_item_id','item_type','item_id','missing_qty',
        'amount','currency','status','ba_number','notes','created_by',
    ];
    protected $casts = [
        'asset_loan_id'=>'integer','loan_item_id'=>'integer','item_id'=>'integer',
        'missing_qty'=>'integer','amount'=>'decimal:20','created_by'=>'integer',
    ];
    public function loan() { return $this->belongsTo(\App\Models\CustomAssetLoan::class, 'asset_loan_id'); }
}
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 7: POLICY & PERMISSION MATRIX
# ------------------------------------------------------------------------------

## 📄 7.1 Permission keys

| Ability | Deskripsi |
|---|---|
| `custom-loan.view` | Lihat list peminjaman |
| `custom-loan.create` | Buat draft |
| `custom-loan.edit` | Edit progres / draft (Creator Only saat DRAFT) |
| `custom-loan.approve` | Approve ke ON_LOAN |
| `custom-loan.void` | Batalkan dokumen |
| `custom-loan.print` | Cetak surat jalan |

## 📄 7.2 `CustomLoanPolicy.php`

```php
<?php
// erickalvino-MODULE_LOAN: Policy peminjaman
namespace App\Policies;

use App\Models\User;
use App\Models\CustomAssetLoan;
use Illuminate\Auth\Access\HandlesAuthorization;

class CustomLoanPolicy
{
    use HandlesAuthorization;

    public function before(User $user, $ability)
    {
        if ($user->isSuperUser()) return true;
    }

    public function view(User $user): bool
    {
        return $user->hasPermission('custom-loan.view');
    }

    public function create(User $user): bool
    {
        return $user->hasPermission('custom-loan.create');
    }

    public function edit(User $user, CustomAssetLoan $loan): bool
    {
        if ($loan->status === 'DRAFT') {
            return $user->id === $loan->created_by;
        }
        return $user->hasPermission('custom-loan.edit')
            && in_array($loan->status, ['PENDING_APPROVAL','ON_LOAN','PENDING_RETURN_QC','PENDING_COMPENSATION']);
    }

    public function approve(User $user, CustomAssetLoan $loan): bool
    {
        return $user->hasPermission('custom-loan.approve') && $loan->status === 'PENDING_APPROVAL';
    }

    public function void(User $user, CustomAssetLoan $loan): bool
    {
        return $user->hasPermission('custom-loan.void')
            && ! in_array($loan->status, ['RETURNED','CANCELLED']);
    }

    public function delete(User $user, CustomAssetLoan $loan): bool
    {
        return $loan->status === 'DRAFT' && $user->id === $loan->created_by;
    }
}
```

## 📄 7.3 Registrasi di `AuthServiceProvider`

```php
protected $policies = [
    \App\Models\CustomAssetLoan::class => \App\Policies\CustomLoanPolicy::class,
];
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 8: FORM REQUESTS
# ------------------------------------------------------------------------------

## 📄 8.1 `LoanStoreRequest.php`

```php
<?php
// erickalvino-MODULE_LOAN: Validasi pembuatan draft peminjaman
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Support\Facades\Gate;

class LoanStoreRequest extends FormRequest
{
    public function authorize(): bool { return Gate::allows('custom-loan.create'); }

    public function rules(): array
    {
        return [
            'borrower_id' => ['required','integer','exists:users,id'],
            'planned_checkout_date' => ['required','date','after_or_equal:today'],
            'planned_return_date'   => ['required','date','after_or_equal:planned_checkout_date'],

            'items'                 => ['required','array','min:1'],
            'items.*.item_type'     => ['required','in:asset,accessory,component,license'],
            'items.*.asset_id'      => ['required_if:items.*.item_type,asset','nullable','integer','exists:assets,id'],
            'items.*.parent_asset_id'=> ['nullable','integer','exists:assets,id'],
            'items.*.accessory_id'  => ['required_if:items.*.item_type,accessory','nullable','integer','exists:accessories,id'],
            'items.*.component_id'  => ['required_if:items.*.item_type,component','nullable','integer','exists:components,id'],
            'items.*.license_id'    => ['required_if:items.*.item_type,license','nullable','integer','exists:licenses,id'],
            'items.*.qty_borrowed'  => ['required','integer','min:1'],
        ];
    }

    public function messages(): array
    {
        return [
            'borrower_id.required' => 'Karyawan peminjam wajib dipilih.',
            'items.required' => 'Minimal satu item wajib dipilih.',
            'items.*.qty_borrowed.min' => 'Kuantitas minimal 1.',
        ];
    }
}
```

## 📄 8.2 `LoanReturnQcRequest.php`

```php
<?php
// erickalvino-MODULE_LOAN: Validasi hasil QC pengembalian
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class LoanReturnQcRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return [
            'items_check' => ['required','array'],
            'items_check.*.qty_returned' => ['required','integer','min:0'],
            'items_check.*.return_check_status' => ['required','in:MATCH,MISSING,EXCHANGED'],
            'items_check.*.condition_at_return' => ['required','in:GOOD,BROKEN'],
            'items_check.*.final_status_label_id' => ['nullable','integer','exists:status_labels,id'],
            'items_check.*.notes' => ['nullable','string'],
        ];
    }

    public function withValidator($validator): void
    {
        $validator->after(function ($v) {
            if ($v->errors()->isNotEmpty()) return;
            foreach ((array) $this->input('items_check', []) as $id => $data) {
                if ((int) $data['qty_returned'] > 0 && $data['condition_at_return'] === 'BROKEN') {
                    $v->errors()->add('items_check.'.$id.'.qty_returned', 'Item BROKEN tidak boleh dihitung kembali sebagai stok bagus.');
                }
            }
        });
    }
}
```

## 📄 8.3 `LoanVoidRequest.php`

```php
<?php
// erickalvino-MODULE_LOAN: Validasi pembatalan
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class LoanVoidRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return ['cancellation_notes' => ['required','string','min:15']];
    }
}
```

## 📄 8.4 `LoanCompensationRequest.php`

```php
<?php
// erickalvino-MODULE_LOAN: Validasi penerbitan BA ganti rugi
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class LoanCompensationRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'amount' => ['required','numeric','min:0'],
            'currency' => ['required','string','size:3'],
            'notes' => ['nullable','string','min:5'],
        ];
    }
}
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 9: SERVICES
# ------------------------------------------------------------------------------

## 📄 9.1 `LoanReservationService.php`

```php
<?php
// erickalvino-MODULE_LOAN: Reservasi stok peminjaman
namespace App\Services\Custom;

use App\Models\CustomAssetLoan;
use App\Models\CustomAssetLoanReservation;
use Illuminate\Support\Facades\DB;
use RuntimeException;

class LoanReservationService
{
    public function reserve(CustomAssetLoan $loan): void
    {
        foreach ($loan->items as $item) {
            $this->lockRow($item->item_type, $this->itemId($item));
            $available = $this->availableQty($item->item_type, $this->itemId($item));
            if ($available < $item->qty_borrowed) {
                throw new RuntimeException("Stok {$item->item_type} id {$this->itemId($item)} tidak mencukupi.");
            }

            CustomAssetLoanReservation::updateOrCreate(
                ['asset_loan_id'=>$loan->id,'item_type'=>$item->item_type,'item_id'=>$this->itemId($item)],
                ['qty'=>$item->qty_borrowed,'location_id'=>$loan->location_id,'status'=>'RESERVED','created_by'=>auth()->id()]
            );
        }
    }

    public function consume(CustomAssetLoan $loan): void
    {
        foreach ($loan->items as $item) {
            if ($item->item_type === 'asset') {
                $this->markAssetOnLoan($item);
            } else {
                $this->decrement($item->item_type, $this->itemId($item), $item->qty_borrowed);
            }
            $loan->reservations()->where('item_type',$item->item_type)->where('item_id',$this->itemId($item))
                ->update(['status'=>'CONSUMED','consumed_at'=>now()]);
        }
    }

    public function release(CustomAssetLoan $loan): void
    {
        $loan->reservations()->where('status','RESERVED')->update(['status'=>'RELEASED','released_at'=>now()]);
    }

    public function restock(CustomAssetLoan $loan): void
    {
        foreach ($loan->items as $item) {
            if ($item->return_check_status === 'MATCH' && $item->condition_at_return === 'GOOD' && $item->item_type !== 'asset') {
                $this->increment($item->item_type, $this->itemId($item), $item->qty_returned);
            }
            $loan->reservations()->where('item_type',$item->item_type)->where('item_id',$this->itemId($item))
                ->update(['status'=>'RESTOCKED','restocked_at'=>now()]);
        }
    }

    public function rollbackStock(CustomAssetLoan $loan): void
    {
        foreach ($loan->items as $item) {
            if ($item->item_type === 'asset') {
                (new QuarantineService)->restoreAssetStatus($item->asset_id, $item->initial_status_label_id);
            } else {
                $this->increment($item->item_type, $this->itemId($item), $item->qty_borrowed);
            }
            $loan->reservations()->where('item_type',$item->item_type)->where('item_id',$this->itemId($item))
                ->update(['status'=>'RELEASED','released_at'=>now()]);
        }
    }

    private function itemId($item): int
    {
        return match ($item->item_type) {
            'asset' => (int) $item->asset_id,
            'accessory' => (int) $item->accessory_id,
            'component' => (int) $item->component_id,
            'license' => (int) $item->license_id,
            default => 0,
        };
    }

    private function lockRow(string $type, int $id): void
    {
        $table = match($type) {
            'component' => 'components',
            'accessory' => 'accessories',
            'license' => 'licenses',
            default => null,
        };
        if ($table) DB::table($table)->whereKey($id)->lockForUpdate()->first();
    }

    private function availableQty(string $type, int $id): int
    {
        return match($type) {
            'component' => (int) DB::table('components')->whereKey($id)->value('qty'),
            'accessory' => (int) DB::table('accessories')->whereKey($id)->value('qty'),
            'license' => (int) DB::table('license_seats')->where('license_id',$id)->whereNull('assigned_to')->whereNull('asset_id')->count(),
            default => 999999, // asset unik: tidak dihitung qty massal
        };
    }

    private function decrement(string $type, int $id, int $qty): void
    {
        match($type) {
            'component' => DB::table('components')->whereKey($id)->decrement('qty', $qty),
            'accessory' => DB::table('accessories')->whereKey($id)->decrement('qty', $qty),
            'license' => null, // handled by LicenseSeatHandler
            default => null,
        };
    }

    private function increment(string $type, int $id, int $qty): void
    {
        match($type) {
            'component' => DB::table('components')->whereKey($id)->increment('qty', $qty),
            'accessory' => DB::table('accessories')->whereKey($id)->increment('qty', $qty),
            'license' => null,
            default => null,
        };
    }

    private function markAssetOnLoan($item): void
    {
        $asset = DB::table('assets')->whereKey($item->asset_id)->lockForUpdate()->first();
        DB::table('assets')->where('id', $item->asset_id)->update([
            'status_id' => \App\Services\Custom\LoanStatusResolver::loanStatusId(),
        ]);
    }
}
```

## 📄 9.2 `AccessoryHandler.php` (v8.6.3)

```php
<?php
// erickalvino-MODULE_LOAN: Handler aksesoris via accessories_checkout
namespace App\Services\Custom;

use App\Models\AccessoryCheckout;
use App\Models\Asset;
use App\Models\User;

class AccessoryHandler
{
    public function assignToUser(int $accessoryId, int $userId, ?string $note = null): void
    {
        AccessoryCheckout::create([
            'accessory_id'=>$accessoryId, 'assigned_to'=>$userId,
            'assigned_type'=>User::class, 'note'=>$note, 'created_by'=>auth()->id(),
        ]);
    }

    public function assignToAsset(int $accessoryId, Asset $asset, ?string $note = null): void
    {
        AccessoryCheckout::create([
            'accessory_id'=>$accessoryId, 'assigned_to'=>$asset->id,
            'assigned_type'=>Asset::class, 'note'=>$note, 'created_by'=>auth()->id(),
        ]);
    }

    public function releaseFromAsset(int $accessoryId, int $assetId): void
    {
        AccessoryCheckout::where('accessory_id',$accessoryId)
            ->where('assigned_type', Asset::class)
            ->where('assigned_to', $assetId)
            ->delete();
    }
}
```

## 📄 9.3 `LicenseSeatHandler.php` (v8.6.3)

```php
<?php
// erickalvino-MODULE_LOAN: Handler lisensi via license_seats
namespace App\Services\Custom;

use App\Models\LicenseSeat;
use App\Models\Asset;
use Illuminate\Support\Facades\DB;

class LicenseSeatHandler
{
    public function assignToAsset(int $licenseId, Asset $asset): void
    {
        $seat = LicenseSeat::where('license_id',$licenseId)
            ->whereNull('assigned_to')->whereNull('asset_id')
            ->lockForUpdate()->first();
        if (! $seat) abort(422, 'Tidak ada seat lisensi tersedia.');
        $seat->asset_id = $asset->id;
        $seat->assigned_to = $asset->assigned_to ?? null;
        $seat->save();
    }

    public function assignToUser(int $licenseId, int $userId): void
    {
        $seat = LicenseSeat::where('license_id',$licenseId)
            ->whereNull('assigned_to')->whereNull('asset_id')
            ->lockForUpdate()->first();
        if (! $seat) abort(422, 'Tidak ada seat lisensi tersedia.');
        $seat->assigned_to = $userId;
        $seat->save();
    }

    public function release(int $licenseId, ?int $userId = null, ?int $assetId = null): void
    {
        $q = LicenseSeat::where('license_id',$licenseId);
        if ($userId) $q->where('assigned_to',$userId);
        if ($assetId) $q->where('asset_id',$assetId);
        $q->update(['assigned_to'=>null,'asset_id'=>null]);
    }
}
```

## 📄 9.4 `QuarantineService.php`

```php
<?php
// erickalvino-MODULE_LOAN: Karantina aset/komponen rusak
namespace App\Services\Custom;

class QuarantineService
{
    public function quarantineAsset(int $assetId, int $locationId): void
    {
        $branch = BranchResolver::get($locationId);
        $asset = \App\Models\Asset::findOrFail($assetId);
        $asset->status_id = config('custom.quarantine_status_id');
        $asset->location_id = $branch['quarantine_location_id'];
        $asset->save();
    }

    public function restoreAssetStatus(int $assetId, ?int $statusLabelId): void
    {
        if ($statusLabelId) {
            \App\Models\Asset::whereKey($assetId)->update(['status_id'=>$statusLabelId]);
        }
    }

    public function quarantineBrokenComponent(int $componentId, int $locationId, string $note): void
    {
        $branch = BranchResolver::get($locationId);
        \DB::table('components_assets')->insert([
            'component_id'=>$componentId,
            'asset_id'=>$branch['quarantine_asset_id'],
            'assigned_to'=>null,
            'user_id'=>auth()->id(),
            'note'=>$note,
            'created_at'=>now(),
            'updated_at'=>now(),
        ]);
    }
}
```

## 📄 9.5 `DocumentNumberService.php`

```php
<?php
// erickalvino-MODULE_LOAN: Nomor dokumen LP non-hardcoded
namespace App\Services\Custom;

class DocumentNumberService
{
    public function nextLoan(int $locationId, int $recordId): string
    {
        $branch = BranchResolver::get($locationId);
        $prefix = config('custom.document_prefix.loan', 'LP');
        return $prefix.'/'.$branch['prefix'].'/'.date('Y').'/'.str_pad((string)$recordId,5,'0',STR_PAD_LEFT);
    }
}
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 10: CONTROLLER
# ------------------------------------------------------------------------------

## 📄 10.1 `CustomAssetLoansController.php` (ringkas final)

```php
<?php
// erickalvino-MODULE_LOAN: Controller peminjaman — semua transisi via POST
namespace App\Http\Controllers\Custom;

use App\Http\Controllers\Controller;
use App\Http\Requests\LoanStoreRequest;
use App\Http\Requests\LoanReturnQcRequest;
use App\Http\Requests\LoanVoidRequest;
use App\Http\Requests\LoanCompensationRequest;
use App\Models\CustomAssetLoan;
use App\Services\Custom\BranchResolver;
use App\Services\Custom\LoanReservationService;
use App\Services\Custom\DocumentNumberService;
use App\Services\Custom\LicenseSeatHandler;
use App\Services\Custom\AccessoryHandler;
use App\Services\Custom\QuarantineService;
use Illuminate\Support\Facades\DB;

class CustomAssetLoansController extends Controller
{
    public function store(LoanStoreRequest $request)
    {
        $this->authorize('create', CustomAssetLoan::class);

        return DB::transaction(function () use ($request) {
            $operator = auth()->user();
            $borrower = \App\Models\User::findOrFail($request->integer('borrower_id'));
            $context = BranchResolver::fromAssetOrOperator($borrower, $operator);

            $loan = CustomAssetLoan::create([
                'company_id'=>$context['company_id'],
                'location_id'=>$context['location_id'],
                'borrower_id'=>$borrower->id,
                'created_by'=>$operator->id,
                'status'=>'DRAFT',
                'planned_checkout_date'=>$request->date('planned_checkout_date'),
                'planned_return_date'=>$request->date('planned_return_date'),
            ]);

            foreach ($request->input('items') as $item) {
                $loan->items()->create([
                    'company_id'=>$context['company_id'],
                    'location_id'=>$context['location_id'],
                    'item_type'=>$item['item_type'],
                    'asset_id'=>$item['asset_id'] ?? null,
                    'parent_asset_id'=>$item['parent_asset_id'] ?? null,
                    'accessory_id'=>$item['accessory_id'] ?? null,
                    'component_id'=>$item['component_id'] ?? null,
                    'license_id'=>$item['license_id'] ?? null,
                    'qty_borrowed'=>$item['qty_borrowed'],
                    'qty_returned'=>0,
                    'return_check_status'=>'MATCH',
                    'condition_at_return'=>'GOOD',
                ]);
            }

            $this->log($loan, 'DRAFT', 'Inisiasi draft peminjaman.');
            return redirect()->route('customLoans.index')->with('success','Draft tersimpan.');
        });
    }

    public function releaseToPendingApproval(int $id)
    {
        $loan = CustomAssetLoan::with('items')->lockForUpdate()->findOrFail($id);
        $this->authorize('edit', $loan);
        if ($loan->status !== 'DRAFT') abort(422);

        return DB::transaction(function () use ($loan) {
            $this->assertNoDoubleBooking($loan);
            (new LoanReservationService)->reserve($loan);
            $loan->document_number = (new DocumentNumberService)->nextLoan($loan->location_id, $loan->id);
            $loan->status = 'PENDING_APPROVAL';
            $loan->save();
            $this->log($loan, 'PENDING_APPROVAL', 'Pengajuan diajukan & stok direservasi.');
        });
    }

    public function approveToOnLoan(int $id)
    {
        $loan = CustomAssetLoan::with('items')->lockForUpdate()->findOrFail($id);
        $this->authorize('approve', $loan);
        if ($loan->status !== 'PENDING_APPROVAL') abort(422);

        return DB::transaction(function () use ($loan) {
            (new LoanReservationService)->consume($loan);
            $loan->status = 'ON_LOAN';
            $loan->actual_checkout_date = now();
            $loan->approved_by = auth()->id();
            $loan->save();
            $this->log($loan, 'ON_LOAN', 'Dokumen disetujui & stok terpotong.');
        });
    }

    public function markReturnReceived(int $id)
    {
        $loan = CustomAssetLoan::with('items')->lockForUpdate()->findOrFail($id);
        $this->authorize('edit', $loan);
        if ($loan->status !== 'ON_LOAN') abort(422);

        return DB::transaction(function () use ($loan) {
            $loan->status = 'PENDING_RETURN_QC';
            $loan->save();
            $this->log($loan, 'PENDING_RETURN_QC', 'Barang kembali, menunggu QC.');
        });
    }

    public function completeReturn(int $id, LoanReturnQcRequest $request)
    {
        $loan = CustomAssetLoan::with('items')->lockForUpdate()->findOrFail($id);
        $this->authorize('edit', $loan);
        if ($loan->status !== 'PENDING_RETURN_QC') abort(422);

        return DB::transaction(function () use ($loan, $request) {
            $hasCompensation = false;

            foreach ($loan->items as $item) {
                $data = $request->input("items_check.{$item->id}");
                $item->qty_returned = (int) $data['qty_returned'];
                $item->return_check_status = $data['return_check_status'];
                $item->condition_at_return = $data['condition_at_return'];
                $item->final_status_label_id = $data['final_status_label_id'] ?? null;
                $item->notes = $data['notes'] ?? null;
                $item->save();

                if (in_array($item->return_check_status, ['MISSING','EXCHANGED'])) {
                    $hasCompensation = true;
                    $loan->compensations()->create([
                        'loan_item_id'=>$item->id,
                        'item_type'=>$item->item_type,
                        'item_id'=>$this->itemId($item),
                        'missing_qty'=>max(1, $item->qty_borrowed - $item->qty_returned),
                        'status'=>'DRAFT',
                        'created_by'=>auth()->id(),
                    ]);
                }
            }

            if ($hasCompensation) {
                $loan->status = 'PENDING_COMPENSATION';
                $loan->save();
                $this->log($loan, 'PENDING_COMPENSATION', 'Ada item MISSING/EXCHANGED; menunggu BA ganti rugi.');
                return redirect()->route('customLoans.compensation', $loan->id);
            }

            (new LoanReservationService)->restock($loan);
            $loan->status = 'RETURNED';
            $loan->actual_return_date = now();
            $loan->save();
            $this->log($loan, 'RETURNED', 'Dokumen ditutup setelah QC.');
            return redirect()->route('customLoans.index')->with('success','Pengembalian selesai.');
        });
    }

    public function issueCompensation(int $id, LoanCompensationRequest $request)
    {
        $loan = CustomAssetLoan::with('items')->lockForUpdate()->findOrFail($id);
        $this->authorize('approve', $loan);
        if ($loan->status !== 'PENDING_COMPENSATION') abort(422);

        return DB::transaction(function () use ($loan, $request) {
            $loan->compensations()->each(function ($comp) use ($request) {
                $comp->amount = $request->input('amount');
                $comp->currency = $request->input('currency');
                $comp->ba_number = 'BAG/'.date('Y').'/'.str_pad($comp->id,5,'0',STR_PAD_LEFT);
                $comp->status = 'ISSUED';
                $comp->notes = $request->input('notes');
                $comp->save();
            });

            (new LoanReservationService)->restock($loan);
            $loan->status = 'RETURNED';
            $loan->actual_return_date = now();
            $loan->save();
            $this->log($loan, 'RETURNED', 'BA ganti rugi diterbitkan & dokumen ditutup.');
        });
    }

    public function cancelLoan(int $id, LoanVoidRequest $request)
    {
        $loan = CustomAssetLoan::with('items')->lockForUpdate()->findOrFail($id);
        $this->authorize('void', $loan);

        return DB::transaction(function () use ($loan, $request) {
            $releasedAlready = $loan->status === 'ON_LOAN' || $loan->status === 'PENDING_RETURN_QC';
            $loan->status = 'CANCELLED';
            $loan->cancellation_notes = $request->input('cancellation_notes');
            $loan->save();

            if ($releasedAlready) {
                (new LoanReservationService)->rollbackStock($loan);
            } else {
                (new LoanReservationService)->release($loan);
            }
            $this->log($loan, 'CANCELLED', 'VOID: '.$request->input('cancellation_notes'));
        });
    }

    private function assertNoDoubleBooking(CustomAssetLoan $loan): void
    {
        foreach ($loan->items->where('item_type','asset') as $item) {
            $conflict = \DB::table('custom_asset_loan_items')
                ->join('custom_asset_loans','custom_asset_loan_items.asset_loan_id','=','custom_asset_loans.id')
                ->where('custom_asset_loan_items.asset_id',$item->asset_id)
                ->whereIn('custom_asset_loans.status',['PENDING_APPROVAL','ON_LOAN','PENDING_RETURN_QC','PENDING_COMPENSATION'])
                ->where('custom_asset_loans.id','!=',$loan->id)
                ->first();
            if ($conflict) abort(422, 'Asset TAG sedang terkunci dokumen lain.');
        }
    }

    private function itemId($item): int
    {
        return match($item->item_type) {
            'asset' => (int) $item->asset_id,
            'accessory' => (int) $item->accessory_id,
            'component' => (int) $item->component_id,
            'license' => (int) $item->license_id,
            default => 0,
        };
    }

    private function log(CustomAssetLoan $loan, string $new, string $note): void
    {
        \App\Models\CustomModuleLog::create([
            'module_context'=>'LOAN','record_id'=>$loan->id,
            'old_status'=>$loan->getOriginal('status'),'new_status'=>$new,
            'action_notes'=>$note,'user_id'=>auth()->id(),'ip_address'=>request()->ip(),
        ]);
    }
}
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 11: ROUTES
# ------------------------------------------------------------------------------

## 📄 11.1 `routes/web.php`

```php
<?php
// erickalvino-MODULE_LOAN: Routes peminjaman — semua transisi POST
use App\Http\Controllers\Custom\CustomAssetLoansController;

Route::group(['middleware'=>['auth']], function () {
    Route::prefix('custom/loans')->name('customLoans.')->group(function () {
        Route::get('/', [CustomAssetLoansController::class,'index'])->name('index');
        Route::get('create', [CustomAssetLoansController::class,'create'])->name('create');
        Route::post('store', [CustomAssetLoansController::class,'store'])->name('store');
        Route::get('{id}/edit', [CustomAssetLoansController::class,'edit'])->name('edit');

        Route::post('{id}/release', [CustomAssetLoansController::class,'releaseToPendingApproval'])->name('release');
        Route::post('{id}/approve-loan', [CustomAssetLoansController::class,'approveToOnLoan'])->name('approve-loan');
        Route::post('{id}/return-received', [CustomAssetLoansController::class,'markReturnReceived'])->name('return-received');
        Route::post('{id}/complete-return', [CustomAssetLoansController::class,'completeReturn'])->name('complete-return');
        Route::post('{id}/compensation', [CustomAssetLoansController::class,'issueCompensation'])->name('compensation');
        Route::post('{id}/cancel', [CustomAssetLoansController::class,'cancelLoan'])->name('cancel');

        Route::get('{id}/print-pdf', [CustomAssetLoansController::class,'printPdf'])->name('print-pdf');
    });
});
```

## 📄 11.2 `routes/api.php`

```php
<?php
// erickalvino-MODULE_LOAN: API JSON peminjaman
use App\Http\Controllers\Api\CustomAssetLoansController;

Route::group(['prefix'=>'v1/custom','middleware'=>['auth:api']], function () {
    Route::get('loans', [CustomAssetLoansController::class,'index'])->name('api.customLoans.index');
});
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 12: VIEWS (SPESIFIKASI)
# ------------------------------------------------------------------------------

## 📄 12.1 Daftar view wajib

| File | Fungsi |
|---|---|
| `custom-loans/index.blade.php` | Tabel Bootstrap Bootstrap |
| `custom-loans/create.blade.php` | Form draft |
| `custom-loans/edit-checking.blade.php` | QC pengembalian |
| `custom-loans/edit.blade.php` | Edit draft |
| `custom-loans/reports/loan-handover-pdf.blade.php` | Cetak surat jalan |

## 📄 12.2 Aturan view
- Semua state-changing memakai form `POST` + `@csrf`.
- Semua dropdown status label di-load dari controller (`$statusLabels`), **bukan hardcoded option**.
- Item form memakai field eksplisit sesuai tipe:
  - `items[i][asset_id]`
  - `items[i][accessory_id]`
  - `items[i][component_id]`
  - `items[i][license_id]`
  - `items[i][parent_asset_id]` (untuk aksesori/komponen/license bawaan aset induk)
- `index` wajib menampilkan kolom `document_number`, `branch_location`, `borrower`, `planned_dates`, `status`, `actions`.

## 📄 12.3 `index.blade.php` (skeleton)

```blade
@extends('layouts/default')
@section('title')
    {{ trans('custom.module_loan') }} @parent
@stop
@section('content')
<div class="row">
  <div class="col-md-12">
    <div class="box box-default">
      <div class="box-header with-border">
        <h3 class="box-title"><i class="fa fa-exchange"></i> {{ trans('custom.module_loan') }}</h3>
        <div class="box-tools pull-right">
          @can('create', \App\Models\CustomAssetLoan::class)
            <a href="{{ route('customLoans.create') }}" class="btn btn-primary btn-sm"><i class="fa fa-plus"></i> {{ trans('general.create') }}</a>
          @endcan
        </div>
      </div>
      <div class="box-body">
        @include('partials.bootstrap-table', [
          'url' => route('api.customLoans.index'),
          'id' => 'customLoansTable',
          'cookie' => 'customLoansTableCookie',
          'columns' => [
            ['field'=>'document_number','title'=>trans('custom.field.document_number'),'sortable'=>true],
            ['field'=>'branch_location','title'=>trans('custom.field.branch'),'sortable'=>true],
            ['field'=>'borrower','title'=>trans('general.user'),'sortable'=>true],
            ['field'=>'planned_dates','title'=>trans('custom.field.planned_dates'),'sortable'=>true],
            ['field'=>'status','title'=>trans('general.status'),'sortable'=>true],
            ['field'=>'actions','title'=>trans('general.actions'),'sortable'=>false,'searchable'=>false],
          ],
        ])
      </div>
    </div>
  </div>
</div>
@include('custom-loans.partials.void-modal')
@stop
```

## 📄 12.4 `edit-checking.blade.php` (QC form — tanpa hardcode status)

```blade
@extends('layouts/default')
@section('title')
    {{ trans('custom.title.return_qc') }} - {{ $loan->document_number }} @parent
@stop
@section('content')
<form method="POST" action="{{ route('customLoans.complete-return', $loan->id) }}">
  @csrf
  <div class="box box-default">
    <div class="box-header">
      <h3 class="box-title">{{ trans('custom.title.return_qc') }}</h3>
    </div>
    <div class="box-body">
      <table class="table table-bordered">
        <thead>
          <tr>
            <th>Item</th><th>Qty Pinjam</th><th>Qty Kembali</th>
            <th>Status</th><th>Kondisi</th><th>Status Akhir</th><th>Catatan</th>
          </tr>
        </thead>
        <tbody>
          @foreach($loan->items as $item)
            <tr>
              <td>{{ $item->item_type }} - {{ $item->asset?->asset_tag ?? $item->component?->name ?? $item->accessory?->name ?? $item->license?->name }}</td>
              <td>{{ $item->qty_borrowed }}</td>
              <td>
                <input type="number" class="form-control" name="items_check[{{ $item->id }}][qty_returned]"
                       value="{{ $item->qty_borrowed }}" min="0" max="{{ $item->qty_borrowed }}" required>
              </td>
              <td>
                <select class="form-control" name="items_check[{{ $item->id }}][return_check_status]" required>
                  <option value="MATCH">MATCH</option>
                  <option value="MISSING">MISSING</option>
                  <option value="EXCHANGED">EXCHANGED</option>
                </select>
              </td>
              <td>
                <select class="form-control" name="items_check[{{ $item->id }}][condition_at_return]" required>
                  <option value="GOOD">GOOD</option>
                  <option value="BROKEN">BROKEN</option>
                </select>
              </td>
              <td>
                @if($item->item_type === 'asset')
                  <select class="form-control select2" name="items_check[{{ $item->id }}][final_status_label_id]">
                    @foreach($statusLabels as $label)
                      <option value="{{ $label->id }}">{{ $label->name }}</option>
                    @endforeach
                  </select>
                @else
                  <span class="text-muted">Auto</span>
                @endif
              </td>
              <td><input class="form-control" name="items_check[{{ $item->id }}][notes]"></td>
            </tr>
          @endforeach
        </tbody>
      </table>
    </div>
    <div class="box-footer text-right">
      <button type="submit" class="btn btn-success">{{ trans('general.submit') }}</button>
    </div>
  </div>
</form>
@stop
```

## 📄 12.5 `void-modal.blade.php`

```blade
<div class="modal fade" id="cancelLoanModal">
  <div class="modal-dialog">
    <form id="cancelLoanForm" method="POST">
      @csrf
      <div class="modal-content">
        <div class="modal-header"><h4>{{ trans('custom.modal.void_title') }}</h4></div>
        <div class="modal-body">
          <textarea class="form-control" name="cancellation_notes" rows="4" required></textarea>
        </div>
        <div class="modal-footer"><button type="submit" class="btn btn-danger">{{ trans('general.submit') }}</button></div>
      </div>
    </form>
  </div>
</div>
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 13: PRESENTER & TRANSFORMER
# ------------------------------------------------------------------------------

## 📄 13.1 `CustomAssetLoanPresenter.php`

```php
<?php
// erickalvino-MODULE_LOAN: Presenter status peminjaman
namespace App\Presenters;

use App\Presenters\Presenter;
use Illuminate\Support\Facades\Lang;
use Illuminate\Support\Str;

class CustomAssetLoanPresenter extends Presenter
{
    public function statusLabel(): string
    {
        $class = match($this->model->status) {
            'DRAFT' => 'label-default',
            'PENDING_APPROVAL' => 'label-warning',
            'ON_LOAN' => 'label-info',
            'PENDING_RETURN_QC' => 'label-primary',
            'PENDING_COMPENSATION' => 'label-danger',
            'RETURNED' => 'label-success',
            'CANCELLED' => 'label-danger',
            default => 'label-default',
        };
        $key = 'custom.status.'.Str::snake($this->model->status);
        $text = Lang::has($key) ? trans($key) : $this->model->status;
        return '<span class="label '.$class.'">'.e($text).'</span>';
    }

    public function documentNumber(): string
    {
        return $this->model->document_number
            ? '<code class="text-bold">'.e($this->model->document_number).'</code>'
            : '<span class="text-muted"><i>DRAFT</i></span>';
    }
}
```

## 📄 13.2 `CustomAssetLoansTransformer.php`

```php
<?php
// erickalvino-MODULE_LOAN: Transformer list peminjaman
namespace App\Transformers;

use App\Models\CustomAssetLoan;
use Illuminate\Support\Facades\Gate;

class CustomAssetLoansTransformer
{
    public function transformCustomAssetLoan(CustomAssetLoan $loan): array
    {
        return [
            'id' => (int) $loan->id,
            'document_number' => $loan->present()->documentNumber(),
            'branch_location' => $loan->location?->name ?? '-',
            'borrower' => $loan->borrower ? '<a href="'.route('users.show',$loan->borrower_id).'">'.e($loan->borrower->first_name.' '.$loan->borrower->last_name).'</a>' : '-',
            'planned_dates' => date('d/m/Y', strtotime($loan->planned_checkout_date)).' s.d '.date('d/m/Y', strtotime($loan->planned_return_date)),
            'actual_checkout' => $loan->actual_checkout_date ? date('d/m/Y H:i', strtotime($loan->actual_checkout_date)) : '-',
            'actual_return' => $loan->actual_return_date ? date('d/m/Y H:i', strtotime($loan->actual_return_date)) : '-',
            'status' => $loan->present()->statusLabel(),
            'actions' => $this->generateActionButtons($loan),
        ];
    }

    private function generateActionButtons(CustomAssetLoan $loan): string
    {
        $adminUserId = auth()->id();
        $html = '<div class="btn-group pull-right"><button class="btn btn-default btn-sm dropdown-toggle" data-toggle="dropdown">'.e(__('general.actions')).' <span class="caret"></span></button><ul class="dropdown-menu pull-right">';

        if ($loan->status === 'DRAFT' && ($loan->created_by === $adminUserId || auth()->user()->isSuperUser())) {
            $html .= '<li><a href="'.route('customLoans.release',$loan->id).'" data-method="post" data-csrf="'.csrf_token().'">'.__('custom.action.release').'</a></li>';
        }
        if ($loan->status === 'PENDING_APPROVAL' && Gate::allows('approve',$loan)) {
            $html .= '<li><a href="'.route('customLoans.approve-loan',$loan->id).'" data-method="post" data-csrf="'.csrf_token().'">'.__('custom.action.approve').'</a></li>';
        }
        if ($loan->status === 'ON_LOAN' && Gate::allows('edit',$loan)) {
            $html .= '<li><a href="'.route('customLoans.return-received',$loan->id).'" data-method="post" data-csrf="'.csrf_token().'">'.__('custom.action.return_process').'</a></li>';
        }
        if ($loan->status === 'PENDING_RETURN_QC' && Gate::allows('edit',$loan)) {
            $html .= '<li><a href="'.route('customLoans.edit-checking',$loan->id).'">'.__('custom.action.qc_verify').'</a></li>';
        }
        if (in_array($loan->status,['PENDING_APPROVAL','ON_LOAN','PENDING_RETURN_QC']) && Gate::allows('void',$loan)) {
            $html .= '<li><a href="#" class="cancel-loan-trigger" data-id="'.$loan->id.'" data-toggle="modal" data-target="#cancelLoanModal">'.__('custom.action.void').'</a></li>';
        }
        if (! in_array($loan->status,['DRAFT','CANCELLED']) && Gate::allows('custom-loan.print')) {
            $html .= '<li><a href="'.route('customLoans.print-pdf',$loan->id).'" target="_blank">'.__('custom.action.print').'</a></li>';
        }
        $html .= '</ul></div>';
        return $html;
    }
}
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 14: TRANSLASI
# ------------------------------------------------------------------------------

`lang/en/custom.php` & `lang/id/custom.php` harus memuat:

```php
'module_loan' => 'Asset Loan & Request',
'field' => [
    'document_number' => 'Document Number',
    'branch' => 'Branch',
    'planned_dates' => 'Planned Dates',
],
'status' => [
    'draft' => 'DRAFT',
    'pending_approval' => 'Pending Approval',
    'on_loan' => 'On Loan',
    'pending_return_qc' => 'Pending Return QC',
    'pending_compensation' => 'Pending Compensation',
    'returned' => 'Returned',
    'cancelled' => 'Cancelled (VOID)',
],
'action' => [
    'release' => 'Submit for Approval',
    'approve' => 'Approve (On Loan)',
    'return_process' => 'Process Return',
    'qc_verify' => 'Verify QC',
    'void' => 'Void',
    'print' => 'Print',
],
'modal' => ['void_title' => 'Cancel / Void Loan Document'],
'title' => ['return_qc' => 'Return QC'],
```

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 15: NON-FUNCTIONAL REQUIREMENTS
# ------------------------------------------------------------------------------

| Aspek | Requirement |
|---|---|
| Concurrency | `DB::transaction` + `lockForUpdate()` di reserve/consume/restock/void |
| Idempotency | State machine menolak transisi dari status yang salah; status terminal tidak berubah |
| Security | Semua transisi POST + CSRF + Policy |
| Audit | Semua status transition dicatat `module_context = LOAN` |
| Multi-tenant | scopeCompanyContext() pada query index/API |
| Retention | `custom_module_logs` insert-only |
| Error handling | Service melempar exception → rollback transaksi |
| Performa | Composite index `(location_id,status)` dan `(company_id,status)` |

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 16: PENGUJIAN
# ------------------------------------------------------------------------------

## 📄 16.1 Factory
- `Database\Factories\CustomAssetLoanFactory`
- `Database\Factories\CustomAssetLoanItemFactory`
- `Database\Factories\CustomAssetLoanReservationFactory`
- `Database\Factories\CustomLoanCompensationFactory`

## 📄 16.2 Feature test contoh

```php
<?php
// erickalvino-MODULE_LOAN: Feature test modul peminjaman
namespace Tests\Feature;

use Tests\TestCase;
use App\Models\User;
use App\Models\Asset;
use App\Models\Accessory;
use App\Models\CustomAssetLoan;
use App\Models\CustomAssetLoanItem;
use Illuminate\Foundation\Testing\RefreshDatabase;

class CustomAssetLoanEnterpriseTest extends TestCase
{
    use RefreshDatabase;

    public function test_double_booking_is_blocked()
    {
        $operator = User::factory()->create(['company_id'=>1,'location_id'=>1]);
        $borrower = User::factory()->create();
        $asset = Asset::factory()->create(['company_id'=>1,'location_id'=>1]);

        $active = CustomAssetLoan::factory()->create([
            'company_id'=>1,'location_id'=>1,'borrower_id'=>$borrower->id,
            'status'=>'ON_LOAN','created_by'=>$operator->id,
        ]);
        CustomAssetLoanItem::factory()->create(['asset_loan_id'=>$active->id,'item_type'=>'asset','asset_id'=>$asset->id,'qty_borrowed'=>1]);

        $draft = CustomAssetLoan::factory()->create([
            'company_id'=>1,'location_id'=>1,'borrower_id'=>$borrower->id,
            'status'=>'DRAFT','created_by'=>$operator->id,
        ]);
        CustomAssetLoanItem::factory()->create(['asset_loan_id'=>$draft->id,'item_type'=>'asset','asset_id'=>$asset->id,'qty_borrowed'=>1]);

        $this->actingAs($operator)
            ->post(route('customLoans.release',$draft->id))
            ->assertRedirect();

        $this->assertEquals('DRAFT', $draft->fresh()->status);
    }

    public function test_accessory_stock_decrements_on_approve()
    {
        $operator = User::factory()->create(['company_id'=>1,'location_id'=>1,'permissions'=>json_encode(['custom-loan.approve'=>1])]);
        $borrower = User::factory()->create();
        $accessory = Accessory::factory()->create(['qty'=>50,'location_id'=>1]);

        $loan = CustomAssetLoan::factory()->create([
            'company_id'=>1,'location_id'=>1,'borrower_id'=>$borrower->id,
            'status'=>'PENDING_APPROVAL','created_by'=>$operator->id,
        ]);
        CustomAssetLoanItem::factory()->create(['asset_loan_id'=>$loan->id,'item_type'=>'accessory','accessory_id'=>$accessory->id,'qty_borrowed'=>5]);

        $this->actingAs($operator)->post(route('customLoans.approve-loan',$loan->id));

        $this->assertEquals(45, $accessory->fresh()->qty);
        $this->assertEquals('ON_LOAN', $loan->fresh()->status);
    }

    public function test_return_with_missing_goes_to_compensation()
    {
        // buat loan PENDING_RETURN_QC + item MISSING, maka status menjadi PENDING_COMPENSATION
        $response = $this->actingAs($operator)
            ->post(route('customLoans.complete-return',$loan->id), [
                'items_check' => [$item->id => [
                    'qty_returned'=>0,
                    'return_check_status'=>'MISSING',
                    'condition_at_return'=>'GOOD',
                ]],
            ]);
        $this->assertEquals('PENDING_COMPENSATION', $loan->fresh()->status);
    }
}
```

## 📄 16.3 Matriks uji manual

| ID | Skenario | Expected |
|---|---|---|
| TC-01 | Double booking asset | Release kedua ditolak |
| TC-02 | Stok aksesoris | Dicatat reservation; decrement saat ON_LOAN |
| TC-03 | Return utuh | Status RETURNED + stock increment |
| TC-04 | Return MISSING/EXCHANGED | Status PENDING_COMPENSATION + blok tutup |
| TC-05 | Compensation issued | BA number dibuat + status RETURNED |
| TC-06 | BROKEN component | Tidak restock, masuk karantina |
| TC-07 | Cancel sebelum ON_LOAN | Reservation released, stock tidak berubah |
| TC-08 | Cancel setelah ON_LOAN | Rollback stock + status asset awal |
| TC-09 | License | Seat assigned saat ON_LOAN, release saat return |
| TC-10 | Accessory | Dicatat `accessories_checkout` assigned_type Asset / User |

---

# ------------------------------------------------------------------------------
# 📌 BAGIAN 17: ROADMAP IMPLEMENTASI
# ------------------------------------------------------------------------------

| Tahap | Deliverable |
|---|---|
| A | Config/env, BranchResolver, LoanStatusResolver, migration final |
| B | Models, Policy, Reservation/License/Accessory/Quarantine service, Controller, Routes |
| C | Views (index/create/edit-checking/print), Presenter, Transformer, Translation |
| D | Factory, Feature test, QC manual TC-01..TC-10, verifikasi skema v8.6.3 |

---

# ==============================================================================
# 🎯 KESIMPULAN REVISI
# ==============================================================================

Dokumen ini menyederhanakan dan menguatkan `PRD-PEMINJAMAN.md` lama:
- **Zero hardcoded**: semua cabang/status/pivot mengikuti Snipe-IT v8.6.3 & config.
- **Alur lengkap**: release → approve → return-received → QC → RETURNED / PENDING_COMPENSATION → RETURNED, plus CANCELLED.
- **Stok aman**: `custom_asset_loan_reservations` + `lockForUpdate()`.
- **Ganti rugi formal**: `custom_loan_compensations`.
- **Log audit benar**: `module_context = 'LOAN'`.

Status: **SIAP UNTUK REVIEW TIM IMPLEMENTASI / BUILD KOMPONEN KODE**.
