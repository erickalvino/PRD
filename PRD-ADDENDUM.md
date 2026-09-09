# PRODUCT REQUIREMENT DOCUMENT — ADDENDUM (REVISI DOKUMEN INDUK)
## Modul Deployment & Maintenance — Snipe-IT v8.6.3
## Status: FINAL — keputusan sudah dikonfirmasi user

> **Tujuan:** Menambahkan perbaikan yang TIDAK mengubah dokumen induk yang sudah dikerjakan agen AI
> (`PRD-DEPLOY-SERVICE-REVISI.md`). Seluruh perubahan di dokumen ini diimplementasikan sebagai **lapisan tambahan**
> melalui migration baru, FormRequest lanjutan, service patch, dan view patch.
>
> **Catatan kepemilikan dokumen:** Setiap blok berisi komentar `// erickalvino-ADDENDUM-XX`.
> Nomor module tetap menggunakan kode `MODULE_DEPLOY` dan `MODULE_MAINT`.

---

# 1. RINGKASAN PERUBAHAN ADDENDUM

| ID | Modul | Perubahan | Alasan |
|---|---|---|---|
| AD-01 | Deploy & Maint | Tambah `request_date` | Mencatat tanggal permintaan/pengajuan |
| AD-02 | Deploy & Maint | Tambah kolom tanggal transisi (`issued_at`, `started_at`, `ready_at`, `completed_at`, `cancelled_at`) | Memperbaiki `issued_at` yang sudah dipakai di view namun belum ada di migration; mendukung SLA/TCO |
| AD-03 | Deploy | Tambah `source_location_id` di `asset_deployment_allocated_items` | Menentukan stok item dipotong dari lokasi penyimpanan mana |
| AD-04 | Deploy | Tambah `source_location_id` di `asset_deployment_reservations` | Ledger reservasi mencerminkan lokasi sumber saat reserve/release/consume |
| AD-05 | Deploy | Validasi stok per lokasi sumber | Mencegah stok dicek/dipotong di lokasi yang salah |
| AD-06 | Deploy | Index dashboard `(company_id, status, request_date)` | Filter laporan berbasis tanggal |
| AD-07 | Maint | Index dashboard `(company_id, status, request_date)` | Filter laporan berbasis tanggal |

---

# 2. KEPUTUSAN KONFIRMASI (FINAL — SUDAH DISETUJUI USER)

## 2.1 Lokasi penyimpanan & validasi stok — KONFIRMASI: **OPSI A**
- ✅ Memakai **Opsi A**:
  - `source_location_id` di `allocated_items` wajib untuk `COMPONENT` dan `ACCESSORY`, `NULL` untuk `LICENSE`.
  - Validasi: `source_location_id` harus sama dengan `components.location_id` / `accessories.location_id`.
  - Ketersediaan qty tetap dihitung dari `components.qty` / `accessories.qty` (skema Snipe-IT v8.6.3).
  - Default bila user tidak memilih: `deployment.location_id`.
- ❌ **Opsi B ditunda** (tidak diimplementasikan sekarang):
  - Tabel `custom_item_location_stock` (`item_type`, `item_id`, `location_id`, `qty`) dapat ditambahkan di fase 2 bila kebutuhan multi-gudang nyata.

## 2.2 `request_date` — KONFIRMASI: **NULLABLE & BOLEH MUNDUR**
- ✅ `request_date` `nullable`, boleh mundur (tidak dibatasi `<= today`), tidak wajib diisi.
- ✅ Fallback tampilan: `request_date ?? created_at->toDateString()`.
- ✅ Tidak ada validasi `before_or_equal:today`.

## 2.3 Default lokasi penyimpanan — KONFIRMASI: **`deployment.location_id`**
- ✅ Bila `source_location_id` tidak dipilih pada `COMPONENT`/`ACCESSORY`, sistem memakai `deployment.location_id`.
- ✅ Untuk `LICENSE` selalu `NULL`.

## 2.4 Kelengkapan tanggal transisi — KONFIRMASI: **TAMBAH SEMUA**
- ✅ Tambah `request_date`, `issued_at`, `started_at`, `ready_at`, `completed_at`, `cancelled_at` pada `asset_deployments` dan `custom_maintenances`.
- ✅ Semua kolom baru `nullable` agar data lama aman.
- ✅ Fallback tampilan: `request_date ?? created_at->toDateString()`.

## 2.5 Backward compatibility
- ✅ Semua kolom tanggal baru dibuat `nullable`.
- ✅ Data lama tanpa `request_date`/`issued_at`/dst tidak error; view memakai fallback (`created_at`/`-`).

---

# 3. PERUBAHAN MIGRATION (ADDENDUM)

## 3.1 Migration `add_request_date_and_dates_to_asset_deployments_table`
```php
<?php
// erickalvino-ADDENDUM-AD-01/02: Tambah request_date + tanggal transisi deployment
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('asset_deployments', function (Blueprint $table) {
            if (! Schema::hasColumn('asset_deployments', 'request_date')) {
                $table->date('request_date')->nullable()->after('location_id');
            }
            if (! Schema::hasColumn('asset_deployments', 'issued_at')) {
                $table->timestamp('issued_at')->nullable()->after('request_date');
            }
            if (! Schema::hasColumn('asset_deployments', 'started_at')) {
                $table->timestamp('started_at')->nullable()->after('issued_at');
            }
            if (! Schema::hasColumn('asset_deployments', 'ready_at')) {
                $table->timestamp('ready_at')->nullable()->after('started_at');
            }
            if (! Schema::hasColumn('asset_deployments', 'completed_at')) {
                $table->timestamp('completed_at')->nullable()->after('ready_at');
            }
            if (! Schema::hasColumn('asset_deployments', 'cancelled_at')) {
                $table->timestamp('cancelled_at')->nullable()->after('completed_at');
            }

            $table->index(['company_id', 'status', 'request_date'], 'idx_deploy_company_status_request');
        });
    }

    public function down(): void
    {
        Schema::table('asset_deployments', function (Blueprint $table) {
            $table->dropIndex('idx_deploy_company_status_request');
            $table->dropColumn([
                'request_date', 'issued_at', 'started_at',
                'ready_at', 'completed_at', 'cancelled_at',
            ]);
        });
    }
};
```

## 3.2 Migration `add_request_date_and_dates_to_custom_maintenances_table`
```php
<?php
// erickalvino-ADDENDUM-AD-01/02: Tambah request_date + tanggal transisi maintenance
return new class extends Migration
{
    public function up(): void
    {
        Schema::table('custom_maintenances', function (Blueprint $table) {
            if (! Schema::hasColumn('custom_maintenances', 'request_date')) {
                $table->date('request_date')->nullable()->after('asset_id');
            }
            if (! Schema::hasColumn('custom_maintenances', 'issued_at')) {
                $table->timestamp('issued_at')->nullable()->after('request_date');
            }
            if (! Schema::hasColumn('custom_maintenances', 'started_at')) {
                $table->timestamp('started_at')->nullable()->after('issued_at');
            }
            if (! Schema::hasColumn('custom_maintenances', 'ready_at')) {
                $table->timestamp('ready_at')->nullable()->after('started_at');
            }
            if (! Schema::hasColumn('custom_maintenances', 'completed_at')) {
                $table->timestamp('completed_at')->nullable()->after('ready_at');
            }
            if (! Schema::hasColumn('custom_maintenances', 'cancelled_at')) {
                $table->timestamp('cancelled_at')->nullable()->after('completed_at');
            }

            $table->index(['company_id', 'status', 'request_date'], 'idx_maint_company_status_request');
        });
    }

    public function down(): void
    {
        Schema::table('custom_maintenances', function (Blueprint $table) {
            $table->dropIndex('idx_maint_company_status_request');
            $table->dropColumn([
                'request_date', 'issued_at', 'started_at',
                'ready_at', 'completed_at', 'cancelled_at',
            ]);
        });
    }
};
```

## 3.3 Migration `add_source_location_to_asset_deployment_allocated_items_table`
```php
<?php
// erickalvino-ADDENDUM-AD-03: Tambah lokasi penyimpanan sumber pada item alokasi
return new class extends Migration
{
    public function up(): void
    {
        Schema::table('asset_deployment_allocated_items', function (Blueprint $table) {
            if (! Schema::hasColumn('asset_deployment_allocated_items', 'source_location_id')) {
                $table->unsignedBigInteger('source_location_id')->nullable()->after('qty');
                $table->index('source_location_id', 'idx_alloc_source_location');
                $table->foreign('source_location_id', 'fk_alloc_source_location')
                      ->references('id')->on('locations')->onDelete('restrict');
            }
        });
    }

    public function down(): void
    {
        Schema::table('asset_deployment_allocated_items', function (Blueprint $table) {
            $table->dropForeign('fk_alloc_source_location');
            $table->dropIndex('idx_alloc_source_location');
            $table->dropColumn('source_location_id');
        });
    }
};
```

## 3.4 Migration `add_source_location_to_asset_deployment_reservations_table`
```php
<?php
// erickalvino-ADDENDUM-AD-04: Tambah lokasi penyimpanan sumber pada ledger reservasi
return new class extends Migration
{
    public function up(): void
    {
        Schema::table('asset_deployment_reservations', function (Blueprint $table) {
            if (! Schema::hasColumn('asset_deployment_reservations', 'source_location_id')) {
                $table->unsignedBigInteger('source_location_id')->nullable()->after('location_id');
                $table->index('source_location_id', 'idx_reserve_source_location');
                $table->foreign('source_location_id', 'fk_reserve_source_location')
                      ->references('id')->on('locations')->onDelete('restrict');
            }
        });
    }

    public function down(): void
    {
        Schema::table('asset_deployment_reservations', function (Blueprint $table) {
            $table->dropForeign('fk_reserve_source_location');
            $table->dropIndex('idx_reserve_source_location');
            $table->dropColumn('source_location_id');
        });
    }
};
```

### Aturan pengisian `source_location_id`

| `item_type` | `source_location_id` | Keterangan |
|---|---|---|
| `COMPONENT` | wajib (atau default `deployment.location_id`) | stok dipotong dari gudang komponen |
| `ACCESSORY` | wajib (atau default `deployment.location_id`) | stok dipotong dari gudang aksesoris |
| `LICENSE` | `NULL` | lisensi tidak memiliki lokasi penyimpanan |

---

# 4. PERUBAHAN MODEL

## 4.1 `AssetDeployment.php`
Tambahkan `request_date`, tanggal transisi, dan accessor fallback.

```php
// erickalvino-ADDENDUM-AD-01/02: Field tanggal + accessor request_date
protected $fillable = [
    'asset_id', 'company_id', 'location_id', 'target_department_id',
    'assigned_technician_id', 'status', 'cancellation_notes',
    'created_by', 'document_number', 'reservation_token_id',
    // ADDENDUM
    'request_date', 'issued_at', 'started_at',
    'ready_at', 'completed_at', 'cancelled_at',
];

protected $casts = [
    'asset_id' => 'integer', 'company_id' => 'integer', 'location_id' => 'integer',
    'target_department_id' => 'integer', 'assigned_technician_id' => 'integer',
    'created_by' => 'integer', 'reservation_token_id' => 'integer',
    // ADDENDUM
    'request_date' => 'date', 'issued_at' => 'datetime',
    'started_at' => 'datetime', 'ready_at' => 'datetime',
    'completed_at' => 'datetime', 'cancelled_at' => 'datetime',
];

public function getRequestDateAttribute($value): ?string
{
    return $value ?: optional($this->created_at)->toDateString();
}
```

## 4.2 `CustomMaintenance.php`
Sama seperti 4.1, tambahkan `request_date` + tanggal transisi dan accessor fallback.

## 4.3 `AssetDeploymentAllocatedItem.php`
```php
// erickalvino-ADDENDUM-AD-03: Relasi lokasi penyimpanan sumber
protected $fillable = [
    'deployment_id', 'item_type', 'item_id', 'qty',
    'source_location_id', // ADDENDUM
    'notes', 'created_by',
];

public function sourceLocation(): BelongsTo
{
    return $this->belongsTo(\App\Models\Location::class, 'source_location_id');
}
```

## 4.4 `AssetDeploymentReservation.php`
```php
// erickalvino-ADDENDUM-AD-04: Relasi lokasi sumber
protected $fillable = [
    'deployment_id', 'item_type', 'item_id', 'qty',
    'location_id', 'source_location_id', // ADDENDUM
    'status', 'reserved_at', 'released_at', 'consumed_at', 'created_by',
];

public function sourceLocation(): BelongsTo
{
    return $this->belongsTo(\App\Models\Location::class, 'source_location_id');
}
```

---

# 5. PERUBAHAN FORM REQUEST

## 5.1 `DeploymentStoreRequest` — tambah `request_date` + `source_location_id`

```php
public function rules(): array
{
    return [
        'asset_id'               => ['required', 'integer', 'exists:assets,id'],
        'target_department_id'   => ['required', 'integer', 'exists:departments,id'],
        'assigned_technician_id' => ['nullable', 'integer', 'exists:users,id'],
        'request_date'           => ['nullable', 'date'], // ADDENDUM — nullable, boleh mundur
        'allocated_items'        => ['nullable', 'array'],
        'allocated_items.*.type' => ['required_with:allocated_items', 'in:COMPONENT,ACCESSORY,LICENSE'],
        'allocated_items.*.id'   => ['required_with:allocated_items', 'integer'],
        'allocated_items.*.qty'  => ['required_with:allocated_items', 'integer', 'min:1'],
        // ADDENDUM: lokasi penyimpanan sumber
        'allocated_items.*.source_location_id' => [
            'nullable', 'integer', 'exists:locations,id',
            function ($attribute, $value, $fail) {
                // LICENSE -> null; COMPONENT/ACCESSORY -> dirutekan via Service/Controller
            },
        ],
    ];
}
```

**Catatan logika** (di controller/service, bukan hanya validasi row):
- `item_type === 'LICENSE'` → `source_location_id = null`.
- `item_type === 'COMPONENT' || 'ACCESSORY'` → jika kosong gunakan `deployment.location_id`.
- Bila **Opsi A** dipilih → wajib sama dengan `components.location_id` / `accessories.location_id`; bila tidak cocok → `ValidationException`.

## 5.2 `MaintenanceStoreRequest` — tambah `request_date`
```php
'request_date' => ['nullable', 'date'], // ADDENDUM — nullable, boleh mundur
```

---

# 6. PERUBAHAN SERVICE

## 6.1 `StockReservationService::reserve()` — gunakan lokasi sumber

(Di bawah ini pola **Opsi A**; bila **Opsi B** disetujui, ganti dengan query ke `custom_item_location_stock`.)

```php
// erickalvino-ADDENDUM-AD-03/04/05: reserve per source_location_id
public function reserve(AssetDeployment $deployment): void
{
    foreach ($deployment->allocatedItems as $item) {
        $sourceLocationId = $item->source_location_id ?: $deployment->location_id;

        if ($item->item_type !== 'LICENSE') {
            $this->validateSourceLocation($item, $sourceLocationId);
        }

        $this->lockStockRow($item->item_type, $item->item_id);

        $qty = $this->availableQty($item->item_type, $item->item_id);
        if ($qty < $item->qty) {
            throw new RuntimeException(
                "Stok {$item->item_type} id {$item->item_id} di lokasi {$sourceLocationId} tidak cukup (tersedia {$qty}, diperlukan {$item->qty})."
            );
        }

        AssetDeploymentReservation::updateOrCreate(
            [
                'deployment_id' => $deployment->id,
                'item_type'     => $item->item_type,
                'item_id'       => $item->item_id,
            ],
            [
                'qty'               => $item->qty,
                'location_id'       => $deployment->location_id,
                'source_location_id'=> $sourceLocationId, // ADDENDUM
                'status'            => 'RESERVED',
                'created_by'        => auth()->id(),
                'released_at'       => null,
                'consumed_at'       => null,
            ]
        );
    }
}

private function validateSourceLocation($item, int $sourceLocationId): void
{
    $model = match ($item->item_type) {
        'COMPONENT' => \App\Models\Component::findOrFail($item->item_id),
        'ACCESSORY' => \App\Models\Accessory::findOrFail($item->item_id),
        default     => null,
    };

    if ($model && $model->location_id && $model->location_id != $sourceLocationId) {
        throw new RuntimeException(
            "Item {$item->item_type} id {$item->item_id} hanya tersedia di lokasi {$model->location_id}, bukan {$sourceLocationId}."
        );
    }
}
```

## 6.2 `StockReservationService::consume()` / `release()`
- Konsisten memakai `source_location_id` yang tersimpan di `asset_deployment_reservations`.
- `release()` mengembalikan ketersediaan ke `source_location_id` (untuk Opsi B: `increment` ke ledger lokasi tsb).
- `consume()` tetap menandai `CONSUMED` pada baris reservasi yang sama.

---

# 7. PERUBAHAN CONTROLLER

## 7.1 `AssetDeploymentsController::store()`
```php
// erickalvino-ADDENDUM-AD-01/03: set request_date + source_location_id
$deployment = AssetDeployment::create([
    // ...
    'request_date' => $request->date('request_date') ?? now()->toDateString(), // ADDENDUM
]);

foreach ((array) $request->input('allocated_items', []) as $item) {
    $deployment->allocatedItems()->create([
        'item_type'          => $item['type'],
        'item_id'            => $item['id'],
        'qty'                => $item['qty'],
        'source_location_id' => $item['type'] === 'LICENSE'
            ? null
            : ($item['source_location_id'] ?? $deployment->location_id),
        'created_by'         => auth()->id(),
    ]);
}
```

## 7.2 Tambah pengisian tanggal transisi di endpoint
```php
// erickalvino-ADDENDUM-AD-02: isi tanggal transisi
releaseToPending()  -> $deployment->issued_at = now();
startProgress()     -> $deployment->started_at = now();
markReady()         -> $deployment->ready_at = now();
complete()          -> $deployment->completed_at = now();
cancelDeployment()  -> $deployment->cancelled_at = now();
```

## 7.3 `CustomMaintenancesController` (pola sama)
```php
releaseToPendingChecking() -> $maintenance->issued_at = now();
acceptDiagnosis()          -> $maintenance->started_at = now();
readyToReturn()            -> $maintenance->ready_at = now();
complete()                 -> $maintenance->completed_at = now();
rejectToUnrepairable()     -> $maintenance->completed_at = now();
cancelMaintenance()        -> $maintenance->cancelled_at = now();
```

---

# 8. PERUBAHAN VIEW & PRESENTER

## 8.1 Show Deployment — tampilkan `request_date`
Use `Helper::getFormattedDateObject($deployment->request_date, 'date')`, bukan `format()` hardcode.

```blade
<x-info-element title="{{ trans('custom.field.request_date') ?? 'Tanggal Pengajuan' }}" icon_type="calendar">
    {{ Helper::getFormattedDateObject($deployment->request_date, 'date')['formatted'] ?? '-' }}
</x-info-element>
```

## 8.2 Show Maintenance — tampilkan `request_date`
Pola sama.

## 8.3 Index — kolom `request_date` di Presenter `dataTableLayout()`
```php
['field' => 'request_date', 'title' => trans('custom.field.request_date'),
 'searchable' => true, 'sortable' => true, 'switchable' => true,
 'visible' => true, 'formatter' => 'dateDisplayFormatter'],
```

## 8.4 Create/Edit Deployment — dropdown/select lokasi penyimpanan sumber per baris item
- Untuk `COMPONENT` / `ACCESSORY`: `<select class="form-control select2" name="allocated_items[i][source_location_id]">` berisi data `locations`.
- Untuk `LICENSE`: tampilkan `-` (tidak memilih lokasi).
- `@include` nilai dari `$storageLocations` (lokasi yang dikirim controller, bukan hardcoded).

## 8.5 Manifest show — tambah kolom "Lokasi Penyimpanan"
- `$item->sourceLocation?->name ?? '-'`.
- LICENSE → `-`.

---

# 9. PERUBAHAN API TRANSFORMER

```php
// erickalvino-ADDENDUM-AD-03: expose lokasi sumber
'source_location' => [
    'id'   => $item->sourceLocation?->id,
    'name' => e($item->sourceLocation?->name ?? '-'),
],
```

---

# 10. PERUBAHAN TEST / UJI MANUAL

| ID | Skenario | Expected |
|---|---|---|
| AD-TC-01 | Buat deployment, request_date kosong | System fallback ke `created_at` |
| AD-TC-02 | `request_date` lebih lama/lebih baru dari hari ini | Diterima (nullable & boleh mundur) |
| AD-TC-03 | COMPONENT dipilih, `source_location_id` beda dengan `components.location_id` | Ditolak Opsi A |
| AD-TC-04 | LICENSE dipilih | `source_location_id` di-reservasi `NULL`, lisensi tetap diambil via `license_seats` |
| AD-TC-05 | Void setelah `issued_at` | `cancelled_at` terisi, reservasi `RELEASED` |
| AD-TC-06 | Complete setelah transisi | `completed_at` terisi |
| AD-TC-07 | Dashboard filter `request_date` | Data muncul sesuai rentang tanggal |
| AD-TC-08 | Data lama tanpa `request_date` | Tidak error; view menampilkan fallback `created_at` |

---

# 11. CHECKLIST SEBELUM DIBERIKAN KE AGEN IMPLEMENTASI

- [x] Konfirmasi Opsi A — validasi stok ke `components.location_id` / `accessories.location_id`.
- [x] Konfirmasi `request_date` nullable & boleh mundur (tanpa `<= today`).
- [x] Pastikan migration `request_date`/`issued_at` dst dibuat **nullable**.
- [x] Pastikan transisi mengisi tanggal transisi (`issued_at`, `started_at`, `ready_at`, `completed_at`, `cancelled_at`).
- [x] Pastikan kolom `source_location_id` pada `allocated_items` & `reservations` di-refresh pada langkah implementasi.
- [x] Pastikan view show/index/create/edit menampilkan lokasi sumber & request_date.
- [x] Pastikan test AD-TC-01..08 dijalankan.

---

# ✅ KEPUTUSAN FINAL (KONFIRMASI USER)

| No | Pertanyaan | Keputusan final |
|---|---|---|
| 1 | Validasi stok per lokasi | **Opsi A** — validasi ke `components.location_id` / `accessories.location_id` |
| 2 | `request_date` constraint | **Nullable + boleh mundur**; tidak dibatasi `<= today`; fallback `created_at` |
| 3 | Default lokasi saat tidak dipilih | **`deployment.location_id`** untuk COMPONENT/ACCESSORY; LICENSE selalu `NULL` |
| 4 | Kelengkapan tanggal transisi | **Tambah semua**: `request_date`, `issued_at`, `started_at`, `ready_at`, `completed_at`, `cancelled_at` |

> ✅ **Addendum siap diberikan ke agen implementasi.** Tidak ada keputusan terbuka lagi untuk perubahan ini.

---

# 12. CARA MENERAPKAN KE DOKUMEN INDUK (MAPPING PER BAGIAN PRD)

> Instruksi untuk agen implementasi: **JANGAN menulis ulang seluruh `PRD-DEPLOY-SERVICE-REVISI.md`.** Terapkan hanya potongan yang tercantum di bawah ini, lalu biarkan bagian lain tetap seperti aslinya.

## 12.1 Ringkasan perubahan per bagian

| Bagian PRD induk | Perubahan |
|---|---|
| Bagian 5 (Skema Database) | Tambah migration baru: `request_date` + tanggal transisi + `source_location_id` |
| Bagian 6 (Model) | Update `$fillable` / `$casts` / accessor / relasi di `AssetDeployment`, `CustomMaintenance`, `AssetDeploymentAllocatedItem`, `AssetDeploymentReservation` |
| Bagian 8 (Form Request) | Update `DeploymentStoreRequest` & `MaintenanceStoreRequest` |
| Bagian 9 (Service) | Update `StockReservationService::reserve()` per `source_location_id` |
| Bagian 10 (Controller) | Set `request_date` + set `issued_at/started_at/ready_at/completed_at/cancelled_at` pada transisi |
| Bagian 12 (Views) | Show/create/edit/index tampilkan lokasi sumber & `request_date` |
| Bagian 14 (Presenter & Transformer) | Tambah kolom `source_location` & `request_date` di `dataTableLayout`/transformer |

## 12.2 Detail mapping per file implementasi

| PRD induk | Elemen | Tambahan addendum |
|---|---|---|
| Bagian 5.1 `asset_deployments` | Kolom | `request_date`, `issued_at`, `started_at`, `ready_at`, `completed_at`, `cancelled_at` |
| Bagian 5.5 `custom_maintenances` | Kolom | `request_date`, `issued_at`, `started_at`, `ready_at`, `completed_at`, `cancelled_at` |
| Bagian 5.3 `asset_deployment_allocated_items` | Kolom | `source_location_id` (FK `locations`, nullable) |
| Bagian 5.2 `asset_deployment_reservations` | Kolom | `source_location_id` (FK `locations`, nullable) |
| Bagian 6.1 `AssetDeployment` | Property | `$fillable`, `$casts`, accessor `getRequestDateAttribute()` |
| Bagian 6.2 `CustomMaintenance` | Property | `$fillable`, `$casts`, accessor `getRequestDateAttribute()` |
| Bagian 6.3 `AssetDeploymentReservation` | Property | `$fillable`, `sourceLocation()` |
| Bagian 6.3 `AssetDeploymentAllocatedItem` | Property | `$fillable`, `sourceLocation()` |
| Bagian 8.1 `DeploymentStoreRequest` | Rules | `request_date` → `nullable|date`; `allocated_items.*.source_location_id` → `nullable|exists:locations,id` (+ validasi Opsi A di service) |
| Bagian 8.2 `MaintenanceStoreRequest` | Rules | `request_date` → `nullable|date` |
| Bagian 9.1 `StockReservationService` | Methods | `reserve()` pakai source lokasi; `validateSourceLocation()`; `release()/consume()` pakai `source_location_id` |
| Bagian 10.2 `AssetDeploymentsController` | store/transisi | set `request_date`; set `issued_at/started_at/ready_at/completed_at/cancelled_at` |
| Bagian 10.3 `CustomMaintenancesController` | store/transisi | set `request_date`; set `issued_at/started_at/ready_at/completed_at/cancelled_at` |
| Bagian 12.7 `asset-deployments/show.blade.php` | View | info-element `request_date`; manifest kolom Lokasi Penyimpanan; tanggal transisi |
| Bagian 12.8 `custom-maintenances/show.blade.php` | View | info-element `request_date`; tanggal transisi |
| Bagian 12.3 / 12.9 index & create/edit | View | dropdown/select "Lokasi Penyimpanan"; kolom index `request_date` |
| Bagian 14.1 Presenter | `dataTableLayout()` | kolom `request_date` |
| Bagian 14.2 Transformer | Response | field `request_date`, `source_location` |
| Bagian 10 / 12 (jalan transisi) | View/Controller | Aksi transisi mengisi `issued_at/started_at/ready_at/completed_at/cancelled_at` |

## 12.3 Filename migration yang harus ditambahkan (urutan diterapkan)

```bash
# Deployment & Maintenance: tanggal dokumen
php artisan make:migration add_request_date_and_dates_to_asset_deployments_table
php artisan make:migration add_request_date_and_dates_to_custom_maintenances_table

# Deployment: lokasi penyimpanan sumber
php artisan make:migration add_source_location_to_asset_deployment_allocated_items_table
php artisan make:migration add_source_location_to_asset_deployment_reservations_table
```

## 12.4 Urutan kerja ada di agen implementasi

1. Terapkan migration (4 file) → `php artisan migrate`.
2. Update Model (4 file) → tambah property + accessor + relasi.
3. Update FormRequest (2 file) → tambah `request_date` dan `source_location_id`.
4. Update Service (`StockReservationService`) → `reserve()` per source lokasi + `validateSourceLocation()`.
5. Update Controller (`AssetDeploymentsController`, `CustomMaintenancesController`) → set `request_date` + tanggal transisi.
6. Update View (show/create/edit/index) → tampilkan lokasi sumber & `request_date`.
7. Update Presenter & Transformer → kolom `request_date` & `source_location`.
8. Update Dashboard → filter `request_date` (tambahan index `idx_*_company_status_request`).
9. Jalankan test `AD-TC-01..08`.

## 12.5 Jangan ubah bagian berikut (di luar scope addendum)

- State machine deployment & maintenance (Bagian 3).
- Policy/Gate (Bagian 7) — tidak berubah karena perubahan ini murni data transaksi.
- Alur reservasi semantik (`RESERVED/CONSUMED/RELEASED`) — tetap sama; hanya ditambah lokasi sumber.
- Pivot resmi Snipe-IT v8.6.3 (Bagian 5.8) — tidak berubah.
- Module Loan (`PRD-PEMINJAMAN-REVISI.md`) — **tidak tersentuh** addendum ini.
