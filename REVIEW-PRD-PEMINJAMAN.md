# Review & Rekomendasi — PRD-PEMINJAMAN.md

> File yang direview: `PRD-PEMINJAMAN.md` — Modul Peminjaman Aset Internal (Asset Loan & Request)
> Target platform yang sudah dikonfirmasi: **Snipe-IT v8.6.3** (`grokability/snipe-it`, PHP ^8.2, Laravel ^12)
> Status: review sebelum revisi/release.

---

## 1. Ringkasan Eksekutif

PRD ini sudah punya fondasi bagus:
- State machine 6 status (`DRAFT → PENDING_APPROVAL → ON_LOAN → PENDING_RETURN_QC → RETURNED / CANCELLED`).
- Konsep `parent_asset_id` untuk paket komposit (Laptop + RAM + Charger + Lisensi).
- Guard anti-double-booking, QC return, dan ganti rugi karyawan.
- Historical lock `company_id`/`location_id`.

Namun, seperti PRD Deployment sebelumnya, sebagian besar masih berbentuk **snippet kode hipotetis**, bukan spesifikasi siap eksekusi, dan masih memakai **banyak hardcoded**. Ada juga **kontradiksi internal** yang cukup krusial: business rule mengatakan *“blockir penutupan kalau MISSING/EXCHANGED dan wajib BA ganti rugi”*, tetapi implementasi `completeReturn()` justru **selalu menutup dokumen sebagai RETURNED**.

---

## 2. CRITICAL — Harus dibereskan sebelum coding/produksi

### 2.1 `completeReturn()` melanggar aturan ganti rugi sendiri
- Bagian 1.4: *“jika MISSING/EXCHANGED → sistem memblokir penutupan langsung, memotong stok global, dan memaksa penerbitan Berita Acara Ganti Rugi Finansial.”*
- Bagian 12.2: `completeReturn()` **tanpa cek MISSING/EXCHANGED** langsung men-set `$loan->status = 'RETURNED'`.

**Rekomendasi:** Tambahkan validator blocking sebelum penutupan: bila ada item `return_check_status = MISSING/EXCHANGED`, set status dokumen ke `PENDING_COMPENSATION` (status baru) atau buat `custom_loan_compensations`, lalu baru tutup setelah BA diterbitkan.

### 2.2 Transisi `ON_LOAN → PENDING_RETURN_QC` tidak ada
- State machine menyebut `ON_LOAN` → barang balik → `PENDING_RETURN_QC`.
- Tidak ada method/route/aksi yang memindahkan status ke `PENDING_RETURN_QC`.
- Di Transformer, menu `ON_LOAN` langsung arahkan ke `edit-checking`, padahal form `complete-return` hanya boleh diproses saat `PENDING_RETURN_QC`.

**Rekomendasi:** Tambahkan method `markReturnReceived()` dengan route `customLoans.return-received` (POST), atau uraikan bahwa tombol “Proses Barang Kembali” memicu transisi ke `PENDING_RETURN_QC` sebelum form QC.

### 2.3 Tidak ada method, route, dan form untuk `CANCEL`
- Business rule menyatakan pembatalan `CANCELLED` dan modal.
- Transformer menampilkan tombol `cancel-loan-trigger`.
- Tidak ada `cancelLoan()` / route `customLoans.cancel` / `VoidRequest` / auto-release stock.

**Rekomendasi:** Lengkapi `cancelLoan()` + `VoidRequest` + route POST + mekanisme release reservation.

### 2.4 Masalah hardcoded yang masih banyak
| Lokasi | Kode | Masalah |
|---|---|---|
| `store()` | `company_id => $operator->company_id ?? 1` | Fallback `1` = hardcode ID |
| `releaseToPendingApproval()` | `$loan->location_id == 2 ? 'BDG' : ...` | Hardcode kode cabang |
| `releaseToOnLoan()` | `config('custom.loan_status_label_id', 5)` | Fallback ID `5` hardcode |
| `completeReturn()` | `config("custom.branches.{$branchLocationId}") ?? config("custom.branches.1")` | Fallback branch `1` hardcode |
| `completeReturn()` | `asset_id => $branchConfig['asset_id']` + komentar `991/992/993` | Hardcode komentar ID karantina |
| `completeReturn()` | `config('custom.quarantine_status_id', 1)` | Fallback ID `1` hardcode |
| `edit-checking` | `<option value="2">Ready to Deploy</option>` / `<option value="1">Archived</option>` | Hardcode status label |

**Rekomendasi:** Terapkan pola yang sama dengan PRD Deployment revisi: `BranchResolver::get($locationId)`, `config('custom.loan_status_label_id')` tanpa default ID, dan `QuarantineService` terpusat.

### 2.5 Form HTML & FormRequest tidak sinkron
- JS mengirim `items[rowIndex][catalog_id]`.
- `LoanStoreRequest` memvalidasi `items.*.asset_id` / `accessory_id` / `component_id` / `license_id`.

Akibat: **form selalu gagal validasi** untuk item selain asset, dan server tidak tahu nilai `catalog_id`.

**Rekomendasi:** Ganti JS mengirim nama field eksplisit per tipe (mis. `input` name = `items[i][asset_id]` ketika tipe asset), atau buat satu field `catalog_type` + `catalog_id` lalu map di FormRequest/Controller.

### 2.6 `module_context = 'MAINTENANCE'` salah secara semantik
- Semua kode log loan memakai `CustomModuleLog::create(['module_context' => 'MAINTENANCE'])`.
- Di PRD Deployment, enum `custom_module_logs.module_context` hanya `['DEPLOYMENT', 'MAINTENANCE']`.

Akibat: log peminjaman tercatat sebagai maintenance, merusak laporan audit & dashboard.

**Rekomendasi:** Tambahkan enum `'LOAN'` ke `custom_module_logs.module_context`, atau buat tabel log terpisah `custom_loan_logs`, dan perbarui semua service yang menulis log.

---

## 3. HIGH — Wajib diperbaiki sebelum merge kode

### 3.1 Tabel pivot salah: `component_assets` vs `components_assets`
- Bagian 12.2 memakai `DB::table('component_assets')`.
- Snipe-IT v8.6.3 memakai `components_assets` (sesuai table/relation `Asset::components()`).

**Rekomendasi:** Gunakan nama tabel resmi `components_assets` dan verifikasi kolom (`component_id`, `asset_id`, `assigned_to`, `user_id`, `note`).

### 3.2 License masih “placeholder”
- Bagian 11.2 `releaseToOnLoan()` untuk license hanya berisi komentar `// Integrasi internal dengan trigger core seat allocation`.
- Tidak ada seat allocation/release pada return.

**Rekomendasi:** Pakai `license_seats` (kolom `assigned_to`/`asset_id`) seperti di PRD Deployment revisi, dan buat handler assign/release seat.

### 3.3 Tidak ada tabel reservasi / anti race condition
- `releaseToOnLoan()` melakukan `decrement('qty')` langsung, tanpa `lockForUpdate()`.
- Double-booking guard hanya mengecek `asset_id` untuk item `asset`, **tidak mengecek accessories/components/licenses**.
- Tidak ada tabel reservation sehingga dua dokumen bisa meminjam komponen yang sama bersamaan.

**Rekomendasi:** Tambah tabel `custom_asset_loan_reservations` (seperti `asset_deployment_reservations`), atau kolom `reserved_qty`; gunakan `lockForUpdate()`.

### 3.4 Status aset berubah ke “Dipinjam” belum konfigurasi
- Bagian 1.3 menyebut “menyerap konfigurasi `.env`”.
- Tidak ada definisi `custom.loan_status_label_id` di config/env.

**Rekomendasi:** Tambahkan `CUSTOM_LOAN_STATUS_LABEL_ID` di `.env` & `config/custom.php`, lalu `LoanStatusResolver::get()` dengan validasi boot.

### 3.5 `parent_asset_id` tidak benar-benar recursive
- Migrasi: `parent_asset_id` FK → `assets.id`.
- Model: `parentAsset()` → `belongsTo(Asset)`.
- Tapi dokumen menyebutnya “recursive tree structure”.

Secara struktur, ini sebenarnya **hubungan item anak → aset induk**, bukan recursive flat-table ke baris item sendiri. Ini tidak salah secara fungsi, tapi nama & pengertian tidak akurat, dan membatasi relasi (mis. anak dari anak tidak didukung).

**Rekomendasi:** Pilih salah satu:
- (A) Bila memang ingin recursive tree: `parent_id` → `custom_asset_loan_items.id` (self-referencing).
- (B) Bila hanya item bawaan aset: ganti istilah menjadi `composite_parent_asset_id` agar jelas.

### 3.6 `edit-checking.blade.php` truncated / syntax rusak
- Bagian 15.2 berakhir secara acak: `@endforeach{{-- ... --}}Kembali Eksekusi...`.
- Hilang closing `</form>`, `</div>`, `</div>`, `</div>`, `@stop`.
- Ada `<option value="2">Ready to Deploy</option>` / `<option value="1">Archived</option>` = hardcode.

**Rekomendasi:** Tulis ulang view penuh, lalu load dropdown `status_labels` dari controller.

### 3.7 Index view & route belum ada
- PRD menyebut `view('custom-loans.index')`, tapi tidak ada `index.blade.php` untuk loan.
- Tidak ada bagian routing `routes/web.php` & `routes/api.php`.
- Route name di transformer tidak konsisten (`customLoans.release`, `customLoans.releaseToOnLoan`, `customLoans.approve-loan`, `customLoans.complete-return`, `customLoans.print-pdf`).

**Rekomendasi:** Tambahkan bagian routing final (semua transisi POST) + index view + konsisten route name.

### 3.8 Permission/Policies tidak di-register
- `CustomLoanPolicy` belum di-register di `AuthServiceProvider`.
- Permission keys (`customloans.view`, `.create`, `.edit`, `.approve`, dll.) belum dijelaskan di bagian permission setup.
- `$user->hasAccess('customloans.*')` perlu dicek apakah format permission v8.6.3 memang begitu.

**Rekomendasi:** Tambah registrasi policy + permission matrix + cara seed permission.

### 3.9 `trans()` dengan `??` tidak berfungsi
- Semua `trans('custom.xxx') ?? 'fallback'` tidak akan memakai fallback karena `trans()` mengembalikan key string, bukan null.

**Rekomendasi:** Gunakan `Lang::has()` seperti yang sudah dirapikan di PRD Deployment revisi, atau buat file translasi lengkap.

---

## 4. MEDIUM — Penyempurnaan kualitas

### 4.1 Missing Presenter
- Model `CustomAssetLoan` menunjuk `App\Presenters\CustomAssetLoanPresenter`.
- PRD mengaggap `present()->statusLabel()` di transformer, **tapi tidak ada spesifikasi presenternya**.

### 4.2 `company_id` / `location_id` dari operator berpotensi salah
- `store()` mengunci lokasi dari `$operator->location_id`.
- Kalau aset dipinjamkan dari cabang lain (mis. operator Bandung, aset dari Jakarta), historical lock-nya jadi Bandung walaupun aset berada di Jakarta.

**Rekomendasi:** Definisi aturan eksplisit: apakah lokasi historis diambil dari aset, operator, atau borrower. Kalau berbasis aset, gunakan `BranchResolver::fromAsset($asset)` seperti PRD Deployment.

### 4.3 Model relation `company` belum ada di snippet model
- Section 4.2 `CustomAssetLoan` belum punya relasi `company()`, `location()`, `borrower()`, `creator()`, `items()`.
- Relasi itu hanya ada di Bagian 6 sebagai “injeksi”. Praktiknya harus dijadikan satu file model final.

### 4.4 File translasi belum ada
- `custom.module_loan`, `custom.action.*`, `custom.print_loan_document`, `custom.module.management_view`, dsb.
- Nama namespace tidak konsisten (`custom.module_loan` vs `custom.module.loan_creation`).

### 4.5 Sidebar injection bergantung modul lain
- `sidebar-injection.blade.php` me-`@can` ke `AssetDeployment` dan `CustomMaintenance`. Kalau modul itu belum terpasang, view error.
- Lebih baik pisahkan blok menu per modul atau pastikan dependency ada.

### 4.6 API `companyContext()` belum dipastikan
- `CustomAssetLoan::companyContext()` dipanggil di API.
- Perlu verifikasi apakah pola `Company::scopeCompanyables($query)` ada di v8.6.3. Kalau tidak, gunakan scope `where('company_id', auth()->user()->company_id)` seperti revisi deployment.

### 4.7 Print PDF kurang detail & hardcoded literal
- `$loan->company->name ?? 'PT. CORPORATE LOGISTICS ENTERPRISE'` diisi literal nama perusahaan.
- `$loan->location->name ?? 'Kantor Pusat Jakarta'` hardcoded.
- Tidak ada `@page` page-break control pada `signature-container`.

### 4.8 Feature test tidak valid sebagai PHP
- Banyak `this->` tanpa `$this`.
- Banyak `\\(loanActive->id` escaped salah.
- `$loan = CustomAssetLoan::create(...)` lalu `$this->actingAs(operator)->get(...)` — `$operator` tidak pakai `$`.
- `route('customLoans.release', ...)` GET — kalau diubah POST, test ikut berubah.
- `Accessory::factory()->create(['qty'=>50])` perlu disesuaikan dengan skema factory v8.6.3.
- `User::factory()->create(['permissions'=>'{"superuser":1}'])` perlu dicek format permissions v8.6.3.

---

## 5. LOW — Kebersihan & konsistensi dokumen

1. **Copy-paste header salah** — bagian 2–20 berjudul “MODUL DEPLOYMENT ASET & PERBAIKAN…” padahal isinya Modul Peminjaman.
2. **Nomor part tidak konsisten** — “PART 1 DARI 10”, lalu “PART 11 DARI 20”.
3. Nama file migrasi memakai `2026_07_23_100001` sementara PRD Deployment memakai `2026_09_06_00000X`. Perlu satu skema penomoran.
4. Variabel `$item['catalog_id']` vs field validasi tidak sinkron.
5. Banyak `trans('...') ??` yang tidak berfungsi.
6. Belum ada spesifikasi non-functional (concurrency, audit retention, backup, idempotency, notifikasi email).
7. Belum ada spec `custom_loan_compensations` (BA ganti rugi) walau rule menyebutnya.
8. Konsep “recursive” tidak tepat secara terminologi (lihat 3.5).
9. Tidak ada skema menu/dashboard untuk modul loan.
10. Tidak ada acceptance criteria Given/When/Then versi QA.

---

## 6. Checklist artefak yang sebaiknya ditambahkan

| Artefak | Status |
|---|---|
| `config/custom.php` (komponen loan) + `.env` | ❌ belum ada |
| `BranchResolver::fromAsset()` | ❌ perlu |
| `LoanStatusResolver` (label “Dipinjam”) | ❌ belum ada |
| Tabel `custom_asset_loan_reservations` | ❌ belum ada |
| Tabel `custom_loan_compensations` | ❌ belum ada |
| `custom_module_logs.module_context = 'LOAN'` | ❌ perlu migrasi |
| Method `markReturnReceived()` | ❌ belum ada |
| Method `cancelLoan()` + `VoidRequest` | ❌ belum ada |
| Route final (web + api) | ❌ belum ada |
| `CustomAssetLoanPresenter` | ❌ belum ada |
| `index.blade.php` | ❌ belum ada |
| Fix `edit-checking.blade.php` | ❌ belum benar |
| Policy registration + permission matrix | ❌ belum ada |
| File translasi `lang/*/custom.php` (loan) | ❌ belum ada |
| Factory + test valid PHP | ❌ masih rusak |

---

## 7. Rekomendasi Urutan Tindakan

### Tahap 1 — Arsitketur
1. Buat `config/custom.php` + `.env` (loan status label, default company/location, branches).
2. Tambah `custom_asset_loan_reservations`.
3. Tambah `custom_loan_compensations`.
4. Perbaiki `module_context` enum untuk `LOAN`.

### Tahap 2 — Alur end-to-end
5. Tambah transisi `ON_LOAN → PENDING_RETURN_QC`.
6. Tambah `cancelLoan()` + auto-release reservation.
7. Selesaikan `completeReturn()` dengan blocking MISSING/EXCHANGED.
8. Perbaiki form `create` (sinkronisasi field) dan `edit-checking`.
9. Lengkapi routing, presenter, index, translasi.

### Tahap 3 — Snipe-IT v8.6.3 alignment
10. Ganti `component_assets` → `components_assets`.
11. Implementasikan license seat assign/release via `license_seats`.
12. Pakai `lockForUpdate()` + reservation.
13. Hapus semua hardcoded branch ID / status ID.

### Tahap 4 — Testing
14. Tulis ulang feature test sesuai state machine POST (bukan GET).
15. Tambah test anti-double-booking untuk component/accessory/license.
16. Tambah test compensation blocking.

---

## 8. Kesimpulan

`PRD-PEMINJAMAN.md` layak dijadikan **fondasi**, tetapi belum siap dieksekusi sebagai PRD final dalam konteks Snipe-IT v8.6.3. Prinsip dan workflow-nya kuat; yang perlu diperbaiki adalah:

- **menghilangkan hardcoded** (sama seperti PRD Deployment),
- **menyelesaikan alur yang dijanjikan** (return-to-QC, cancel, ganti rugi),
- **menyelaraskan dengan skema v8.6.3** (`components_assets`, `license_seats`, permission), dan
- **menambah artefak pendukung** (config, reservation, policy, routing, view, presenter, translasi, test).

Kalau disetujui, saya bisa langsung menyusun **PRD-PEMINJAMAN-REVISI.md** yang menutup seluruh poin di atas, disesuaikan dengan `PRD-DEPLOY-SERVICE-REVISI.md`.
