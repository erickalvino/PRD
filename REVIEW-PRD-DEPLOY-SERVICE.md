# Review & Rekomendasi Penyempurnaan — PRD-DEPLOY-SERVICE.md

> File yang direview: `PRD-DEPLOY-SERVICE.md` (40 bagian, ±4.183 baris)
> Tujuan: memetakan apa yang sudah bagus dan apa yang perlu disempurnakan sebelum kode di-inject ke Snipe-IT produksi.

---

## 1. Ringkasan Eksekutif

PRD ini sudah **sangat kuat secara konsep**: state machine `DRAFT`, `CANCELLED`, historis `company_id`/`location_id`, audit trail, dan karantina multi-cabang adalah desain yang benar. Namun, sebagian besar isinya masih berupa **sketsa kode** (code snippet), bukan **spesifikasi yang bisa langsung dieksekusi**.

Masalah terbesar ada 4:

1. **Artefak krusial tidak pernah didefinisikan** — `config/custom.php`, `.env`, Policy/Gate, file translasi `custom.php`, factory, view Deployment, dan endpoint transisi lain.
2. **State machine lebih lengkap daripada implementasi** — banyak status yang dijelaskan tapi tidak punya controller/route.
3. **Kontradiksi aturan internal** — PRD melarang *hardcoded*, tapi banyak contoh kode pakai angka mati (`location_id == 2`, `branches.1`, komentar 991/992/993).
4. **Logika stok belum nyata** — "reservation engine" dan "auto-release stock" hanya ditulis sebagai komentar, tidak ada tabel `reserved_qty` atau tabel reservasi.

---

## 2. CRITICAL — Harus dibereskan sebelum coding/produksi

### 2.1 Konfigurasi multi-cabang tidak ada spesifikasinya
- PRD berkali-kali menyebut `config/custom.php` dan `.env`, tapi **tidak pernah mendefinisikan isi dan struktur kunci-kuncinya**.
- Yang tidak terdefinisi:
  - `custom.branches.*` (branch_code, branch_name, asset_id karantina, dsb.)
  - `custom.quarantine_status_id`
  - Variabel `.env` untuk cabang/karantina
- Akibatnya implementasi akan jatuh ke **fallback hardcoded** yang justru dilarang.

### 2.2 Pelanggaran aturan "tanpa hardcoded ID"
Contoh kontradiksi di dalam dokumen:
- Bagian 26: `($maintenance->location_id == 2) ? 'BDG' : (($maintenance->location_id == 3) ? 'SBY' : 'JKT');`
- Bagian 21: `?? config("custom.branches.1")`
- Bagian 23: komentar `// Otomatis mengarah ke 991, 992, atau 993`
- Bagian 40 TC-04: ekspektasi `ID:992`

**Rekomendasi:** Ganti semua tebak-kira ID/code dengan satu sumber konfigurasi + resolver cabang (`$branchConfig = config("custom.branches.{$locationId}")`), lalu buat **config sample** dan **validator config** saat boot.

### 2.3 Reservation / Auto-Release Stock belum diimplementasikan
- Bagian 21 hanya menulis komentar: `// Kunci stok secara logis ... via log transaksi`.
- Tidak ada:
  - kolom `reserved_qty` pada `components` / `accessories` / `licenses`
  - tabel reservasi (`asset_deployment_reservations`)
  - mekanisme `lockForUpdate()` untuk mencegah race condition
- Bagian 22 "auto-release" tidak benar-benar melepas reservasi, hanya menulis log.

**Rekomendasi:** Tambahkan tabel reservasi + kolom `reserved_qty` atau gunakan `checkouts`/`action_logs` Snipe-IT, lalu perbarui release → reserve, cancel → release, complete → decrement.

### 2.4 Policy & Gate tidak pernah dibuat
- `authorize('view', AssetDeployment::class)` dan `Gate::allows('asset-deployment.create')` dipakai di banyak tempat.
- Tapi **tidak ada** `AssetDeploymentPolicy`, `CustomMaintenancePolicy`, `AuthServiceProvider` registration, ataupun permission matrix per role.

**Rekomendasi:** Tambahkan bagian spesifikasi Policy: `view`, `create`, `edit`, `delete`, `print`, plus matriks role (Fixed Asset, IT, FA, Superadmin).

### 2.5 Transisi status state machine tidak lengkap
Status yang **disebut di enum/migrasi tapi tanpa route/controller**:

**Deployment:**
- `DRAFT → PENDING_HANDOVER` (ada)
- `PENDING_HANDOVER → IN_PROGRESS` → tidak ada
- `IN_PROGRESS → READY_TO_RETURN` → tidak ada
- `READY_TO_RETURN → COMPLETED` (ada)

**Maintenance:**
- `DRAFT → PENDING_CHECKING` (ada)
- `PENDING_CHECKING → UNDER_DIAGNOSIS` → tidak ada
- `UNDER_DIAGNOSIS → IN_SERVICE_INTERNAL / OUT_TO_VENDOR` → tidak ada
- `OUT_TO_VENDOR → POST_SERVICE_QC` → tidak ada
- `POST_SERVICE_QC → READY_TO_RETURN` → tidak ada
- `READY_TO_RETURN → COMPLETED / UNREPAIRABLE` (ada)

**Rekomendasi:** Definisikan metode + POST route untuk setiap transisi, atau hapus status yang tidak dibutuhkan. Untuk alur sementara, jangan buat behavior yang mengandalkan direct edit status.

### 2.6 View Deployment tidak ada sama sekali
- Controller `AssetDeploymentsController@index` menyebut `asset-deployments.index`.
- Tapi PRD hanya menyediakan view untuk `custom-maintenances` (Bagian 37, 38, 39).

**Rekomendasi:** Tambahkan `index.blade.php`, `create.blade.php`, `edit.blade.php`, dan `partial/bootstrap-table` untuk module Deployment.

---

## 3. HIGH — Perbaikan wajib sebelum merge kode

### 3.1 Form `create.blade` tidak lengkap dibanding FormRequest
Form maintenance (Bagian 38) hanya punya:
- `asset_id`, `maintenance_type`, `is_warranty_claim`, `issue_description`

Padahal `MaintenanceStoreRequest` butuh:
- `supplier_id` (wajib jika EXTERNAL)
- `estimated_completion_date` (wajib jika EXTERNAL)
- `assigned_technician_id`
- `cannibal_items.*`

Ini akan membuat **form selalu gagal validasi**.

**Rekomendasi:** Sejajarkan form dengan rules, termasuk field vendor conditional + cannibal item repeater + validation error display.

### 3.2 Route name tidak konsisten
- Register: `assetDeployments.printPdf` / `customMaintenances.printPdf`
- Dipakai: `assetDeployments.print-pdf` / `customMaintenances.print-pdf` / `customMaintenances.print-sticker`

**Rekomendasi:** Seragamkan seluruh route name dan/atau ubah pemakaian di transformer.

### 3.3 State-changing action memakai HTTP GET
- `customMaintenances.release` dan `assetDeployments.release` di-register dengan `GET`.
- `release` mengubah status → **tidak aman** untuk GET (bisa di-trigger crawler/CSRF).

**Rekomendasi:** Ubah rilis/penerbitan menjadi `POST` (atau tetap GET hanya untuk halaman konfirmasi, lalu POST) dan tambahkan CSRF.

### 3.4 `trans()` fallback (`??`) tidak berfungsi
Semua kode memakai pola:
```php
trans('custom.status_draft') ?? 'DRAFT'
```
Padahal Laravel `trans()` **mengembalikan string key** bila key tidak ada (`'custom.status_draft'`), bukan `null`. Jadi fallback tidak pernah aktif.

**Rekomendasi:** Gunakan helper `Lang::has()` atau langsung hardcode teks fallback, atau buat file translasi lengkap.

### 3.5 File translasi `custom` tidak ada
Banyak key dipakai, formatnya tidak konsisten:
- `custom.status_draft`
- `custom.status.pending_checking`
- `custom.status_cancelled`
- `custom.tooltips.*`
- `custom.action.*`

**Rekomendasi:** Buat `lang/en/custom.php` dan `lang/id/custom.php` dengan namespace/naming konsisten (semua pakai `custom.status.*`, atau semua `custom.status_*`).

### 3.6 `config/custom.php` & `.env` belum ada di PRD
Tambahkan contoh penuh sebagai artefak terpisah, misalnya:
```php
// config/custom.php
return [
    'quarantine_status_id' => env('CUSTOM_QUARANTINE_STATUS_ID'),
    'branches' => [
        1 => ['code'=>'JKT','name'=>'Jakarta','asset_id'=>env('CUSTOM_QUARANTINE_JKT')],
        2 => ['code'=>'BDG','name'=>'Bandung','asset_id'=>env('CUSTOM_QUARANTINE_BDG')],
        3 => ['code'=>'SBY','name'=>'Surabaya','asset_id'=>env('CUSTOM_QUARANTINE_SBY')],
    ],
];
```
Jangan pakai fallback `branches.1` yang menyiratkan data statis.

---

## 4. MEDIUM — Penyempurnaan kualitas & keamanan data

### 4.1 Foreign Key tidak lengkap
- `asset_id`, `company_id`, `location_id`, `target_department_id`, `assigned_technician_id`, `created_by` di tabel deployment/maintenance **hanya indexed, tanpa FK**.
- `donor_asset_id`, `item_id`, `user_id` di tabel anak juga belum di-FK.

**Rekomendasi:** Gunakan `unsignedBigInteger` konsisten + `foreign(...)->references('id')->on(...)->onDelete('restrict')` sesuai aturan audit.

### 4.2 `item_id` untuk unregistered donor tidak konsisten
Bagian 6 & 14 menyatakan `is_unregistered_donor=1` berarti komponen **belum ada di katalog**, tetapi:
- `item_id` di migrasi bersifat `NOT NULL`
- `complete()` melakukan `Component::findOrFail($item->item_id)` lalu `attach()` ke aset target

Padahal jika tidak terdaftar, seharusnya tidak ada ID komponen yang valid untuk di-attach.

**Rekomendasi:** Untuk unregistered, simpan nama/deskripsi part (kolom `part_name`, `part_spec`) dan buat *note-only* log, tanpa menempel ke tabel `components` core.

### 4.3 Logika accessories ke `assets` salah
Bagian 23:
```php
$accessory->users()->attach($asset->id, ...)
```
Ini menaruh **ID aset** ke pivot `accessories_users`, padahal aksesoris Snipe-IT di-assign ke **user**, bukan ke asset.

**Rekomendasi:** Tentukan model relasi yang benar (biasanya assign ke user/assignee), atau buat mapping custom untuk `asset` dan tempatkan di tabel pivot sendiri.

### 4.4 License tidak pernah diproses
- `allocated_items` mendukung `LICENSE`
- `complete()` deployment hanya memproses `COMPONENT` dan `ACCESSORY`
- `release_license_seat` di `detached_items` tidak pernah dieksekusi

**Rekomendasi:** Tambahkan alur license (assign seat, release seat) di `complete()` dan `cancelDeployment()`.

### 4.5 Karantina tidak sinkron dengan konsep aset virtual
Bagian 23 menulis langsung ke tabel `component_assets` (perlu verifikasi nama asli: Snipe-IT biasanya `components_assets`), tapi:
- tidak menandai komponen tersebut masuk/meninggalkan stok global dengan benar
- tidak memindahkan lokasi/status aset donor
- `rejectToUnrepairable` (Bagian 29) hanya mengubah `status_id` aset, **tidak menempatkannya** ke aset virtual/id karantina dinamis

**Rekomendasi:** Satu desain karantina terpusat untuk seluruh modul: quarantine location + quarantine status + virtual asset ID + pivot part, dipakai dimanapun tanpa hardcoded.

### 4.6 Tidak ada mekanisme anti-duplikat transaksi aktif
Tidak dicegah satu asset punya 2+ tiket aktif (deployment/maintenance) di status berjalan.

**Rekomendasi:** Tambahkan validasi unik per `asset_id` + status aktif, atau kebijakan eksplisit bahwa itu diperbolehkan.

### 4.7 Kurangnya `lockForUpdate()` untuk concurrency
Pengecekan stok, pembuatan nomor `SJP`, dan pengurangan qty tidak aman untuk request paralel (race condition).

**Rekomendasi:** Gunakan `DB::transaction` + `lockForUpdate` pada baris stock/asset saat reserve/complete/cancel.

---

## 5. LOW — Kebersihan dokumen & mantainability

1. **Banyak snippet rusak/typo**
   - `STRINK_PAD_LEFT` → `STR_PAD_LEFT`
   - `\\(id)` / `\\(deployment)` — escaped variable tidak konsisten
   - `releaseToPending` bagian ACCESSORY/LICENSE hanya placeholder "Validasi serupa..."
2. **Pola komentar pelacakan tidak konsisten**
   - Aturan → baris #1 tanpa spasi. Nyatanya banyak file diawali `@extends`/`<!DOCTYPE>`/`namespace` dulu.
3. **Belum ada spec non-functional**
   - Performa, retention log, backup, SLA, notifikasi email, RBAC matrix, retry/idempotency.
4. **Belum ada acceptance criteria Given/When/Then**
   - Table manual test (TC-01..04) bagus, tapi belum ada criterion otomatis yang bisa dipakai QA.
5. **Target platform tidak ditentukan**
   - Sebutkan versi Snipe-IT (misal v6.5), Laravel/PHP version, DB engine, dan kondisi test environment.
6. **Belum ada status `VOID` sebagai enum terpisah**
   - PRD memakai `CANCELLED` dan `VOID` bergantian; pilih satu istilah agar konsisten.
7. **Belum ada dashboard modul**
   - Ada comment tag `MODULE_DASH`, tapi bagian dashboard/TCO/SLA tidak pernah ada artefaknya.
8. **QR code pakai layanan eksternal**
   - `https://qrserver.com...` adalah dependensi eksternal; tidak disarankan untuk internal device.
   - Gunakan `endroid/qr-code` atau library local; jangan tampilkan URL internal via QR eksternal.
9. **Blade `$asset->location->name ?? '...'`** bisa error bila `location` null di PHP <8; pakai `?->`.
10. **Test file tidak lengkap**
    - Tidak ada test module Deployment.
    - Factory `Asset`, `User`, `CustomMaintenance` belum didefinisikan.
    - `User::factory()->create(['permissions'=>'{"superuser":1}'])` perlu dicek format Snipe-IT.

---

## 6. Check List Implementasi yang Sebaiknya Ditambahkan ke PRD

| Artefak | Status |
| --- | --- |
| `config/custom.php` + `.env` sample | ❌ belum ada |
| `lang/en/custom.php`, `lang/id/custom.php` | ❌ belum ada |
| `AssetDeploymentPolicy`, `CustomMaintenancePolicy` | ❌ belum ada |
| View `asset-deployments/*` | ❌ belum ada |
| View maintenance `edit`, `show` | ❌ sebagian |
| Route + controller transisi status lengkap | ❌ sebagian |
| Tabel reservasi / `reserved_qty` | ❌ belum ada |
| `Component`, `Accessory`, `License` handling lengkap | ❌ sebagian |
| Karantina terpusat tanpa hardcoded | ❌ belum konsisten |
| Factory + feature test deployment | ❌ belum ada |
| Acceptance criteria Given/When/Then | ❌ belum ada |
| Spesifikasi versi platform | ❌ belum ada |
| Retrofit road map ke Snipe-IT versi target | ❌ belum ada |

---

## 7. Rekomendasi Urutan Tindakan (Action Plan)

### Tahap 1 — Menutup gap arsitektur
1. Definisikan `config/custom.php`, `.env`, resolver cabang, dan karantina terpusat.
2. Rancang tabel reservasi stok (`reserved_qty`) + relasi.
3. Rancang Policy & permission matrix.
4. Tentukan versi Snipe-IT/Laravel/PHP target.

### Tahap 2 — Melengkapi alur end-to-end
5. Tambahkan semua endpoint transisi status (POST) dan method controller.
6. Lengkapi view Deployment + view Maintenance edit/show.
7. Selesaikan handling Component/Accessory/License, termasuk unregistered donor.
8. Tambahkan file translasi lengkap.

### Tahap 3 — Uji & hardening
9. Tambahkan automated feature test untuk kedua modul.
10. Tambahkan acceptance criteria non-functional (concurrency, audit retention, RBAC).
11. Perbaiki seluruh snippet yang mengandung hardcoded ID/code.
12. Validasi semua tabel/pivot terhadap skema Snipe-IT target.

---

## 8. Kesimpulan

PRD-DEPLOY-SERVICE.md **layak dijadikan fondasi**, tetapi **belum siap dieksekusi sebagai "REVISI FINAL"**. Ia lebih tepat disebut *blueprint v1*: 40 bagian sudah mencakup hampir semua ide, tetapi masih berisi banyak implementasi hipotetis yang belum terbukti cocok dengan skema asli Snipe-IT dan belum memenuhi dua janji utamanya:
- **"tanpa hardcoded ID"** → banyak yang masih hardcoded
- **"auto-release stock"** → belum ada mekanisme reservasi nyata

Jika dua hal di atas dibereskan dan semua artefak (config, policy, view, translation, test) ditambahkan, dokumen ini akan benar-benar menjadi spesifikasi tingkat enterprise.
