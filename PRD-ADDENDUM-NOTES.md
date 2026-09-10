# PRODUCT REQUIREMENT DOCUMENT — ADDENDUM NOTES (CATATAN TEKNISI)
## Modul Deployment & Maintenance — Snipe-IT v8.6.3
## Status: FINAL — semua keputusan sudah dikonfirmasi user

> Dokumen ini **terpisah** dari `PRD-ADDENDUM.md` agar bisa dikerjakan **bertahap oleh agen-AI**:
> - Tahap 1: `PRD-ADDENDUM.md` (request_date, tanggal transisi, source_location_id) — sudah FINAL.
> - Tahap 2: dokumen ini (catatan teknisi) — kerjakan SETELAH tahap 1 selesai.
>
> Penomoran kebutuhan di dokumen ini memakai prefiks **ADN-** (Addendum Notes) agar tidak
> bentrok dengan **AD-** di `PRD-ADDENDUM.md`.
>
> **Tidak ada file PRD induk yang diubah** oleh addendum ini.

---

# 1. TUJUAN FITUR

Teknisi yang mengerjakan **Asset Deployment** dan **Custom Maintenance** sering menemukan
kendala di lapangan (aset rusak, spek tidak sesuai, akses lokasi, menunggu sparepart, dll).
Saat ini tidak ada tempat terstruktur untuk mencatatnya, sehingga:

- Informasi kendala hilang di chat pribadi / tidak masuk laporan resmi.
- Manajemen tidak tahu deployment/maintenance mana yang sedang terhambat.
- Tidak ada bukti tertulis kendala sudah ditangani (audit trail).

Solusi: **panel Catatan Teknisi** pada halaman detail deployment & maintenance, dengan
riwayat lengkap (siapa, kapan, tipe, severity) dan status penyelesaian kendala.

---

# 2. KEPUTUSAN YANG SUDAH DIKONFIRMASI USER

## 2.1 Penyimpanan — ✅ SATU TABEL TERSTRUKTUR BERSAMA
- Satu tabel `custom_technician_notes` dipakai bersama modul **DEPLOYMENT** dan **MAINTENANCE**
  (pemisahan via kolom `module` — konsisten dengan pola enum string yang sudah dipakai modul,
  mis. `item_type` pada `asset_deployment_allocated_items`).
- Alasan ditolak: kolom `technician_notes` TEXT tunggal (tanpa riwayat/penulis/status) dan
  dua tabel terpisah (duplikasi struktur).

## 2.2 Nilai enum — ✅ BAHASA INGGRIS (UI DITERJEMAHKAN VIA LANG)
- Nilai di database **selalu bahasa Inggris**, tampilan ke user diterjemahkan dengan
  file lang **`en-US`** dan **`id-ID`** (pola `trans('custom.notes.*')` yang sudah dipakai PRD induk).
- `type`   : `NOTE` | `ISSUE` | `RECOMMENDATION`
- `severity`: `INFO` | `WARNING` | `BLOCKER` (hanya relevan untuk `ISSUE`)
- Status kendala **turunan** (bukan kolom): `resolved_at IS NULL` → `OPEN`, selain itu `RESOLVED`.

## 2.3 Deliverable — ✅ FILE TERPISAH, KERJA BERTAHAP
- Dokumen ini sendiri adalah deliverable-nya; agen implementasi mengerjakannya sebagai
  workstream terpisah setelah `PRD-ADDENDUM.md`.

---

# 3. SKEMA DATABASE

## ADN-01 Migration: `custom_technician_notes`

```php
<?php
// database/migrations/xxxx_xx_xx_xxxxxx_create_custom_technician_notes_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('custom_technician_notes', function (Blueprint $table) {
            $table->increments('id');
            $table->enum('module', ['DEPLOYMENT', 'MAINTENANCE']);   // pemilik catatan
            $table->unsignedBigInteger('module_id')->index();        // id asset_deployments / custom_maintenances

            $table->unsignedBigInteger('technician_id')->nullable()->index(); // penulis catatan (FK users)
            $table->enum('type', ['NOTE', 'ISSUE', 'RECOMMENDATION'])->default('NOTE');
            $table->enum('severity', ['INFO', 'WARNING', 'BLOCKER'])->nullable(); // hanya untuk ISSUE
            $table->text('body');                                    // isi catatan (wajib)

            $table->timestamp('resolved_at')->nullable();            // diisi saat ISSUE selesai ditangani
            $table->text('resolution_note')->nullable();             // cara penyelesaian (wajib bila resolved)

            $table->unsignedBigInteger('created_by')->nullable();    // audit Snipe-IT convention
            $table->timestamps();
            $table->softDeletes();

            // Composite index untuk badge "Kendala Aktif" di index page
            $table->index(['module', 'module_id', 'type', 'resolved_at'], 'idx_tech_notes_badge');

            $table->foreign('technician_id', 'fk_tech_notes_user')
                  ->references('id')->on('users')->onDelete('set null');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('custom_technician_notes');
    }
};
```

**Catatan desain:**
- `module_id` sengaja **tanpa FK relational** ke dua tabel berbeda (polimorfik manual via
  `module`); integritas dijaga di layer service (`assertSubjectExists()`), sama seperti pola
  `item_type`/`item_id` di modul.
- `severity` nullable — hanya diisi bila `type = ISSUE`.
- Tidak ada kolom status fisik; `OPEN`/`RESOLVED` diturunkan dari `resolved_at`.

---

# 4. MODEL

## ADN-02 `App\Models\Custom\TechnicianNote`

```php
<?php
namespace App\Models\Custom;

use App\Models\SnipeModel;
use App\Models\User;
use Illuminate\Database\Eloquent\SoftDeletes;

class TechnicianNote extends SnipeModel
{
    use SoftDeletes;

    protected $table = 'custom_technician_notes';

    protected $fillable = [
        'module', 'module_id', 'technician_id', 'type', 'severity',
        'body', 'resolved_at', 'resolution_note', 'created_by',
    ];

    protected $casts = [
        'resolved_at' => 'datetime',
        'technician_id' => 'integer',
        'module_id'     => 'integer',
    ];

    // Nilai enum bahasa Inggris — tampilan diterjemahkan via lang
    public const MODULE_DEPLOYMENT  = 'DEPLOYMENT';
    public const MODULE_MAINTENANCE = 'MAINTENANCE';

    public const TYPE_NOTE           = 'NOTE';
    public const TYPE_ISSUE          = 'ISSUE';
    public const TYPE_RECOMMENDATION = 'RECOMMENDATION';

    public const SEVERITY_INFO    = 'INFO';
    public const SEVERITY_WARNING = 'WARNING';
    public const SEVERITY_BLOCKER = 'BLOCKER';

    /** Status terminal — catatan menjadi read-only */
    public const DEPLOYMENT_TERMINAL  = ['COMPLETED', 'CANCELLED'];
    public const MAINTENANCE_TERMINAL = ['COMPLETED', 'UNREPAIRABLE', 'CANCELLED'];

    public function technician() { return $this->belongsTo(User::class, 'technician_id'); }

    // ---- Scopes ----
    public function scopeForModule($query, string $module, int $moduleId)
    {
        return $query->where('module', $module)->where('module_id', $moduleId)
                     ->with('technician')->orderBy('created_at', 'desc');
    }

    public function scopeOpenIssues($query)
    {
        return $query->where('type', self::TYPE_ISSUE)->whereNull('resolved_at');
    }

    // ---- Accessor turunan ----
    public function getStatusAttribute(): string   // OPEN | RESOLVED (turunan, bukan kolom)
    {
        return $this->resolved_at === null ? 'OPEN' : 'RESOLVED';
    }

    public function isIssue(): bool   { return $this->type === self::TYPE_ISSUE; }
    public function isOpen(): bool    { return $this->isIssue() && $this->resolved_at === null; }
    public function isBlocker(): bool { return $this->isIssue() && $this->severity === self::SEVERITY_BLOCKER; }
}
```

## ADN-02b Relasi balik (tambahan kecil di 2 model induk)

```php
// AssetDeployment.php  (tambahan $fillable tidak berubah — tabel induk tidak disentuh)
public function technicianNotes() {
    return $this->hasMany(\App\Models\Custom\TechnicianNote::class, 'module_id')
                ->where('module', \App\Models\Custom\TechnicianNote::MODULE_DEPLOYMENT);
}
public function allowsTechnicianNotes(): bool {
    return ! in_array($this->status, \App\Models\Custom\TechnicianNote::DEPLOYMENT_TERMINAL, true);
}

// CustomMaintenance.php — pola identik
public function technicianNotes() { /* ... MODULE_MAINTENANCE ... */ }
public function allowsTechnicianNotes(): bool {
    return ! in_array($this->status, \App\Models\Custom\TechnicianNote::MAINTENANCE_TERMINAL, true);
}
```

---

# 5. FORM REQUEST

## ADN-03 `TechnicianNoteStoreRequest` / `TechnicianNoteResolveRequest`

```php
public function rules(): array
{
    return [
        'type'     => ['required', 'in:NOTE,ISSUE,RECOMMENDATION'],
        'severity' => ['nullable', 'required_if:type,ISSUE', 'in:INFO,WARNING,BLOCKER'],
        'body'     => ['required', 'string', 'max:5000'],
    ];
    // severity wajib hanya bila type = ISSUE; NOTE/RECOMMENDATION tidak pakai severity
}

public function authorize(): bool
{
    // Penulis = teknisi yang ditugaskan pada dokumen induk, ATAU superuser
    $user = auth()->user();
    return $user->isSuperUser()
        || $this->subject->assigned_technician_id === $user->id;
}
```

```php
public function rules(): array // TechnicianNoteResolveRequest
{
    return [
        'resolution_note' => ['required', 'string', 'max:5000'], // wajib saat menutup ISSUE
    ];
}
```

---

# 6. SERVICE

## ADN-04 `App\Services\TechnicianNoteService`

```php
class TechnicianNoteService
{
    /** Posting catatan baru. Melempar ValidationException bila dokumen induk sudah terminal. */
    public function add(AssetDeployment|CustomMaintenance $subject, array $data, User $author): TechnicianNote
    {
        $this->assertWritable($subject);

        return TechnicianNote::create([
            'module'        => $subject instanceof AssetDeployment
                                ? TechnicianNote::MODULE_DEPLOYMENT
                                : TechnicianNote::MODULE_MAINTENANCE,
            'module_id'     => $subject->id,
            'technician_id' => $author->id,
            'type'          => $data['type'],
            'severity'      => $data['type'] === TechnicianNote::TYPE_ISSUE ? $data['severity'] : null,
            'body'          => $data['body'],
            'created_by'    => $author->id,
        ]);
    }

    /** Menutup ISSUE: wajib resolution_note. NOTE/RECOMMENDATION tidak bisa di-resolve. */
    public function resolve(TechnicianNote $note, string $resolutionNote, User $actor): TechnicianNote
    {
        abort_unless($note->isIssue(), 422, 'Only ISSUE can be resolved');
        abort_if($note->isOpen() === false, 422, 'Issue already resolved');

        $note->update([
            'resolved_at'     => now(),
            'resolution_note' => $resolutionNote,
        ]);
        return $note;
    }

    public function activeIssueCount(AssetDeployment|CustomMaintenance $subject): int
    {
        return $subject->technicianNotes()->openIssues()->count();
    }

    public function hasOpenBlocker(AssetDeployment|CustomMaintenance $subject): bool
    {
        return $subject->technicianNotes()
            ->where('severity', TechnicianNote::SEVERITY_BLOCKER)
            ->whereNull('resolved_at')->exists();
    }

    private function assertWritable(AssetDeployment|CustomMaintenance $subject): void
    {
        abort_unless($subject->allowsTechnicianNotes(), 422,
            'Notes are read-only after the document is completed/cancelled');
    }
}
```

---

# 7. CONTROLLER & ROUTES

## ADN-05 `TechnicianNotesController`

```php
// store  : POST  custom/asset-deployments/{deployment}/notes      → name('assetDeployments.notes.store')
// store  : POST  custom/custom-maintenances/{maintenance}/notes   → name('customMaintenances.notes.store')
// resolve: PATCH custom/technician-notes/{note}/resolve           → name('technicianNotes.resolve')

public function store(TechnicianNoteStoreRequest $request, AssetDeployment $deployment)
{
    $this->noteService->add($deployment, $request->validated(), auth()->user());
    return redirect()->back()->with('success', trans('custom.notes.msg.created'));
}

public function resolve(TechnicianNoteResolveRequest $request, TechnicianNote $note)
{
    $this->noteService->resolve($note, $request->input('resolution_note'), auth()->user());
    return redirect()->back()->with('success', trans('custom.notes.msg.resolved'));
}
```

- Route pakai **pattern route existing** modul (group `custom/`, middleware `auth`).
- Setelah simpan: `redirect()->back()` ke halaman show pemilik catatan (deployment/maintenance).

## ADN-05b Guard config di `complete()` (keputusan D-1 — default NON-AKTIF)

Tambahkan di `config/deploy-service.php` (file config modul):

```php
// config/deploy-service.php — tambahan
'block_on_open_blocker' => env('DEPLOY_BLOCK_ON_OPEN_BLOCKER', false),
```

Guard di awal `complete()` kedua controller (deployment & maintenance) — **tidak aktif
secara default** (keputusan D-1 = warning saja):

```php
// erickalvino-ADDENDUM-NOTES ADN-05b: guard opsional, default false
if (config('deploy-service.block_on_open_blocker')
    && app(\App\Services\TechnicianNoteService::class)->hasOpenBlocker($deployment)) {
    throw \Illuminate\Validation\ValidationException::withMessages([
        'technician_notes' => trans('custom.notes.msg.blocked_complete'),
    ]);
}
```

> Karena default `false`, perilaku rilis pertama = **warning saja** (ADN-06c); bila suatu saat
> manajemen ingin memperketat, cukup set env `DEPLOY_BLOCK_ON_OPEN_BLOCKER=true`.

---

# 8. VIEWS

## ADN-06 Partial panel: `resources/views/custom/technician-notes/_panel.blade.php`

Dipanggil dari `asset-deployments/show.blade.php` dan `custom-maintenances/show.blade.php`:
`@include('custom.technician-notes._panel', ['subject' => $deployment])`

```blade
<x-box :header="trans('custom.notes.title')" box_style="default">

    {{-- Form tambah catatan: hanya saat dokumen belum terminal --}}
    @if ($subject->allowsTechnicianNotes())
        <form method="POST" action="{{ $formAction }}">
            @csrf
            <div class="row">
                <div class="col-md-3">
                    <label>{{ trans('custom.notes.field.type') }}</label>
                    <select name="type" class="form-control" id="note_type">
                        <option value="NOTE">{{ trans('custom.notes.type.NOTE') }}</option>
                        <option value="ISSUE">{{ trans('custom.notes.type.ISSUE') }}</option>
                        <option value="RECOMMENDATION">{{ trans('custom.notes.type.RECOMMENDATION') }}</option>
                    </select>
                </div>
                <div class="col-md-3" id="severity_wrap" style="display:none">
                    <label>{{ trans('custom.notes.field.severity') }}</label>
                    <select name="severity" class="form-control">
                        <option value="INFO">{{ trans('custom.notes.severity.INFO') }}</option>
                        <option value="WARNING">{{ trans('custom.notes.severity.WARNING') }}</option>
                        <option value="BLOCKER">{{ trans('custom.notes.severity.BLOCKER') }}</option>
                    </select>
                </div>
                <div class="col-md-12" style="margin-top:8px">
                    <textarea name="body" class="form-control" rows="3" required
                        placeholder="{{ trans('custom.notes.field.body_placeholder') }}"></textarea>
                </div>
            </div>
            <div class="text-right" style="margin-top:8px">
                <button type="submit" class="btn btn-primary">
                    <x-icon type="checkmark" /> {{ trans('general.save') }}
                </button>
            </div>
        </form>
    @else
        <p class="text-muted"><i>{{ trans('custom.notes.msg.readonly') }}</i></p>
    @endif

    {{-- Riwayat catatan --}}
    <table class="table table-striped" style="margin-top:12px">
        <thead>
            <tr>
                <th>{{ trans('general.date') }}</th>
                <th>{{ trans('custom.notes.field.technician') }}</th>
                <th>{{ trans('custom.notes.field.type') }}</th>
                <th>{{ trans('custom.notes.field.severity') }}</th>
                <th>{{ trans('custom.notes.field.body') }}</th>
                <th>{{ trans('custom.notes.field.status') }}</th>
                <th></th>
            </tr>
        </thead>
        <tbody>
            @forelse ($subject->technicianNotes as $note)
                <tr>
                    <td>{{ $note->created_at->format('d-m-Y H:i') }}</td>
                    <td>{{ $note->technician?->getFullNameAttribute() ?? '-' }}</td>
                    <td>{{ trans('custom.notes.type.'.$note->type) }}</td>
                    <td>
                        @if ($note->isIssue())
                            <span class="label label-{{ ['INFO'=>'default','WARNING'=>'warning','BLOCKER'=>'danger'][$note->severity] }}">
                                {{ trans('custom.notes.severity.'.$note->severity) }}
                            </span>
                        @else - @endif
                    </td>
                    <td>{{ $note->body }}</td>
                    <td>
                        @if ($note->isIssue())
                            <span class="label label-{{ $note->isOpen() ? 'danger' : 'success' }}">
                                {{ trans('custom.notes.status.'.$note->status) }}
                            </span>
                        @else - @endif
                    </td>
                    <td>
                        @if ($note->isOpen())
                            <button class="btn btn-xs btn-default" data-toggle="modal"
                                data-target="#resolve-{{ $note->id }}">
                                {{ trans('custom.notes.action.resolve') }}
                            </button>
                            {{-- modal kecil berisi textarea resolution_note + POST/PATCH --}}
                        @endif
                    </td>
                </tr>
                @if ($note->resolved_at)
                    <tr class="text-muted small">
                        <td></td><td colspan="5">
                            ✔ {{ $note->resolved_at->format('d-m-Y H:i') }} — {{ $note->resolution_note }}
                        </td><td></td>
                    </tr>
                @endif
            @empty
                <tr><td colspan="7" class="text-muted">{{ trans('custom.notes.msg.empty') }}</td></tr>
            @endforelse
        </tbody>
    </table>
</x-box>

@push('scripts')
{{-- script kecil: tampilkan severity_wrap hanya bila type = ISSUE --}}
@endpush
```

**Aturan render (kepatuhan Snipe-IT v8.6.3):**
- `<x-box>` TIDAK di-nesting `box-body` manual — slot langsung; form + table di dalam slot.
- Tombol simpan: `<button class="btn btn-primary">` + `<x-icon type="checkmark">`
  (pola `blade/box/footer.blade.php`). Tidak memakai komponen fiktif.
- Semua label via `trans('custom.notes.*')`; **tidak ada teks hardcode** di blade.
- Aksi tulis dibungkus pengecekan `allowsTechnicianNotes()` (bukan state machine baru).

## ADN-06b Badge di halaman index

Transformer (Bagian 14 PRD induk) menambah field:

```php
'active_issues'    => (int) $this->technicianNotes()->openIssues()->count(),
'has_open_blocker' => (bool) app(\App\Services\TechnicianNoteService::class)
                              ->hasOpenBlocker($this),
```

Presenter `dataTableLayout()` menambah kolom (pola kolom existing):

```
[ { field: 'active_issues', title: 'Kendala Aktif', ... } ]
```

Render badge: `0` → abu; `>0` → kuning; `has_open_blocker = true` → **merah**
(contoh: `Kendala Aktif: 2`).

---

# 9. LANGUAGE FILES (en-US & id-ID)

## ADN-07 Tambahan key di `lang/<locale>/custom.php`

```php
// lang/en-US/custom.php  (tambahan group 'notes')
'notes' => [
    'title'  => 'Technician Notes',
    'field' => [
        'type' => 'Type', 'severity' => 'Severity', 'body' => 'Note',
        'technician' => 'Technician', 'status' => 'Status',
        'body_placeholder' => 'Write note, issue, or recommendation…',
    ],
    'type' => [
        'NOTE' => 'Note', 'ISSUE' => 'Issue', 'RECOMMENDATION' => 'Recommendation',
    ],
    'severity' => [
        'INFO' => 'Info', 'WARNING' => 'Warning', 'BLOCKER' => 'Blocker',
    ],
    'status' => [ 'OPEN' => 'Open', 'RESOLVED' => 'Resolved' ],
    'action' => [ 'resolve' => 'Resolve' ],
    'msg' => [
        'created'  => 'Note added.',
        'resolved' => 'Issue resolved.',
        'readonly' => 'Notes are read-only — document is completed/cancelled.',
        'empty'    => 'No notes yet.',
        'open_issues_warning' => 'There are still active issues on this document.',
        'open_issues_detail'  => 'Active issues: :count:blocker Please review the Technician Notes panel before completing.',
        'with_blocker' => ' (including a BLOCKER-severity issue!)',
        'blocked_complete' => 'Completion is blocked while a BLOCKER-severity issue is still open.',
    ],
],

// lang/id-ID/custom.php  (tambahan group 'notes')
'notes' => [
    'title'  => 'Catatan Teknisi',
    'field' => [
        'type' => 'Tipe', 'severity' => 'Tingkat', 'body' => 'Catatan',
        'technician' => 'Teknisi', 'status' => 'Status',
        'body_placeholder' => 'Tulis catatan, kendala, atau rekomendasi…',
    ],
    'type' => [
        'NOTE' => 'Catatan', 'ISSUE' => 'Kendala', 'RECOMMENDATION' => 'Rekomendasi',
    ],
    'severity' => [
        'INFO' => 'Info', 'WARNING' => 'Perhatian', 'BLOCKER' => 'Blokir',
    ],
    'status' => [ 'OPEN' => 'Terbuka', 'RESOLVED' => 'Selesai' ],
    'action' => [ 'resolve' => 'Tutup' ],
    'msg' => [
        'created'  => 'Catatan ditambahkan.',
        'resolved' => 'Kendala selesai ditangani.',
        'readonly' => 'Catatan hanya baca — dokumen sudah selesai/dibatalkan.',
        'empty'    => 'Belum ada catatan.',
    ],
],
```

**Aturan:** nilai enum DB tetap `NOTE/ISSUE/...` (Inggris); terjemahan hanya di layer view
via `trans('custom.notes.type.'.$note->type)` — sesuai keputusan user (lang trans en-US & id-ID).

---

# 10. ATURAN BISNIS (RINGKAS)

| # | Aturan |
|---|---|
| BN-01 | Penulis catatan: **teknisi yang ditugaskan** (`assigned_technician_id`) pada dokumen induk, atau **superuser**. |
| BN-02 | Catatan bisa dibuat selama status dokumen **belum terminal** (deployment: bukan `COMPLETED/CANCELLED`; maintenance: bukan `COMPLETED/UNREPAIRABLE/CANCELLED`). Setelah terminal → panel read-only. |
| BN-03 | `severity` wajib hanya bila `type = ISSUE`; `NOTE`/`RECOMMENDATION` tanpa severity. |
| BN-04 | Hanya `ISSUE` yang bisa di-**resolve**; wajib isi `resolution_note`; resolved bersifat permanen (tidak bisa dibuka lagi — buat ISSUE baru bila berulang). |
| BN-05 | "Kendala aktif" = `type=ISSUE` dengan `resolved_at IS NULL`. Badge index: abu `0`, kuning `>0`, merah bila ada `BLOCKER` aktif. |
| BN-06 | Fitur catatan **tidak mengubah state machine** deployment/maintenance — murni lampiran dokumen (lihat Bagian 12 D-1 untuk opsi memperketat). |
| BN-07 | Nilai enum selalu Inggris di DB; tampilan diterjemahkan via lang `en-US`/`id-ID`. |

---

# 11. TEST CASE MANUAL

| ID | Skenario | Hasil diharapkan |
|---|---|---|
| ADN-TC-01 | Teknisi posting `NOTE` | Muncul di panel + riwayat (nama, tanggal) |
| ADN-TC-02 | Teknisi posting `ISSUE` severity `BLOCKER` | Badge index merah, baris label `Blokir` + `Terbuka` |
| ADN-TC-03 | User lain (bukan teknisi/superuser) POST catatan | 403 |
| ADN-TC-04 | Posting catatan setelah status `COMPLETED` | Ditolak (422), panel read-only |
| ADN-TC-05 | Resolve `ISSUE` tanpa `resolution_note` | Ditolak (validasi wajib) |
| ADN-TC-06 | Resolve `ISSUE` lengkap | Status jadi `Selesai`, catatan resolusi tampil |
| ADN-TC-07 | `RECOMMENDATION` mencoba di-resolve | Ditolak (422 — hanya ISSUE) |
| ADN-TC-08 | Dokumen lama tanpa catatan | Badge `0` abu, panel "Belum ada catatan", tidak error |
| ADN-TC-09 | Ganti locale `id-ID` ↔ `en-US` | Label berubah (`Kendala/Blokir` ↔ `Issue/Blocker`), nilai DB tetap |

---

# 12. KEPUTUSAN KONFIRMASI (FINAL — SUDAH DISETUJUI USER)

> Ketiga keputusan mengikuti **rekomendasi** yang diberikan. Struktur tabel & enum tidak berubah.

## D-1 — Efek `ISSUE` severity `BLOCKER` terhadap `complete()`
- ✅ **(a) Warning saja.** Tombol complete tetap aktif; tampil peringatan kuning
  "Masih ada N kendala aktif" (detail di ADN-06c).
- ✅ Config toggle siap: `config('deploy-service.block_on_open_blocker', false)` — bila kelak
  di-set `true`, `complete()` otomatis menolak selama ada `BLOCKER` aktif **tanpa deploy kode baru**
  (guard sudah ditulis di ADN-05b, tidak aktif secara default).

## D-2 — Lampiran foto pada catatan
- ✅ **(a) Tidak dulu — fase 1 teks saja.** Kolom `attachment_path` TIDAK dibuat sekarang.
- 📌 Roadmap fase 2 (eksplisit ditunda): upload foto per catatan via pola `image-upload` Snipe-IT
  (butuh storage disk + validasi file). Bila diimplementasikan, cukup 1 migration tambah kolom +
  update FormRequest/view — tidak mengubah struktur existing.

## D-3 — Notifikasi ke atasan/admin saat `BLOCKER` dibuat
- ✅ **(a) Tidak dulu — fase 1.** Visibilitas cukup dari badge merah di halaman index (ADN-06b)
  dan banner peringatan di halaman show (ADN-06c).
- 📌 Roadmap fase 2 (eksplisit ditunda): Laravel Notification (mail) ke admin/pembuat dokumen
  saat `ISSUE` severity `BLOCKER` dibuat, mengikuti pola notification Snipe-IT.

## Tabel keputusan final

| No | Pertanyaan | Keputusan final |
|---|---|---|
| 1 | Penyimpanan catatan | **Satu tabel `custom_technician_notes`** bersama DEPLOYMENT & MAINTENANCE |
| 2 | Nilai enum | **Bahasa Inggris** di DB; UI diterjemahkan via lang `en-US` & `id-ID` |
| 3 | D-1: efek `BLOCKER` | **Warning saja** + config toggle `block_on_open_blocker` (default `false`) |
| 4 | D-2: lampiran foto | **Tidak dulu** — fase 2 (teks saja di fase 1) |
| 5 | D-3: notifikasi email | **Tidak dulu** — fase 2 (badge + banner cukup di fase 1) |

> ✅ **Addendum siap diberikan ke agen implementasi tahap 2.** Tidak ada keputusan terbuka lagi.

---

# 13. CHECKLIST IMPLEMENTASI (UNTUK AGEN TAHAP 2)

- [ ] Prasyarat: tahap 1 (`PRD-ADDENDUM.md`) sudah diterapkan & migrate.
- [ ] Buat migration `custom_technician_notes` (ADN-01) → `php artisan migrate`.
- [ ] Buat model `TechnicianNote` + scope + accessor (ADN-02).
- [ ] Tambah relasi `technicianNotes()` + `allowsTechnicianNotes()` di 2 model induk (ADN-02b).
- [ ] Buat 2 FormRequest (ADN-03).
- [ ] Buat `TechnicianNoteService` (ADN-04).
- [ ] Buat `TechnicianNotesController` + 3 routes (ADN-05).
- [ ] Tambah config `block_on_open_blocker` (default `false`) + guard di `complete()` kedua controller (ADN-05b).
- [ ] Buat partial `_panel.blade.php` + include di 2 show view (ADN-06).
- [ ] Tambah kolom badge di transformer/presenter index deployment & maintenance (ADN-06b).
- [ ] Tambah banner warning di atas tombol complete di 2 show view (ADN-06c).
- [ ] Tambah group `notes` di `lang/en-US/custom.php` dan `lang/id-ID/custom.php` (ADN-07).
- [x] Keputusan D-1/D-2/D-3 sudah final (lihat Bagian 12): warning saja / tanpa foto / tanpa notifikasi di fase 1.
- [ ] Jalankan test ADN-TC-01..12.

---

# 14. MAPPING KE PRD INDUK (TANPA MENGUBAH FILE INDUK)

| Bagian PRD induk | Sentuhan addendum ini |
|---|---|
| Bagian 5 (Skema DB) | Tambah 1 tabel baru `custom_technician_notes` — tabel induk TIDAK diubah |
| Bagian 6 (Model) | 1 model baru + 2 method kecil di `AssetDeployment`/`CustomMaintenance` |
| Bagian 7 (Policy/Gate) | Aturan penulis di FormRequest `authorize()` (BN-01) — policy induk tidak berubah |
| Bagian 10 (Controller) | 1 controller baru; tidak menyentuh transisi state machine |
| Bagian 12 (Views) | 1 partial baru + `@include` di 2 show view; kolom badge di 2 index |
| Bagian 14 (Presenter/Transformer) | Tambah 2 field transformer + 1 kolom presenter |
| `lang/*/custom.php` | Tambah group `notes` (en-US, id-ID) |

> Prinsip: addendum ini **murni additive** — tidak ada kolom/status/route lama yang berubah.
> Bila D-1(b) yang dipilih, baru `complete()` controller diberi 1 blok guard tambahan.
yang berubah.
> Bila D-1(b) yang dipilih, baru `complete()` controller diberi 1 blok guard tambahan.
ahan.
