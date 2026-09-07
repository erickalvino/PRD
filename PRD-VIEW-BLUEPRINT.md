# PRODUCT REQUIREMENT DOCUMENT (PRD) - BLUEPRINT
## Kustom Modul Snipe-IT (erickalvino)
## VIEW BLUEPRINT: STANDAR TEMPLATE UI/UX UNTUK SEMUA MODUL KUSTOM
## ARSITEKTUR: LARAVEL BLADE + ADMINLTE 2 + KOMPONEN BLADE SNIPE-IT (100% SNIPE-IT COMPATIBLE)

---

## 1. PENDAHULUAN & RUANG LINGKUP

Dokumen ini adalah **standar tunggal (blueprint)** untuk membangun atau memperbaiki semua halaman view (index / show / edit / create) pada modul kustom Snipe-IT. Tujuan akhirnya: setiap halaman baru maupun lama memiliki tampilan yang konsisten dengan AdminLTE 2 dan pola Blade component Snipe-IT v6+, mendukung dark mode, ramah aksesibilitas, dan mudah dipelihara.

### 1.1 Latar Belakang Masalah

Beberapa modul kustom ditulis dengan gaya yang berbeda-beda:

| Modul | Gaya saat ini |
|-------|---------------|
| `asset-mutation-requests` (RTM) | Show memakai `well` + grid inline + 4 sub-box terpisah |
| `asset-mutations` (MTS) | Show memakai `table table-condensed` label:value 2 kolom |
| `assetmutation/bast` (BAST) | Show memakai `x-container` + `x-box` + `box-footer` |
| `material-issuances` (MI) | Show memakai `x-container` + `x-box` + inline style padat |
| `purchase-requests`, `procurement` | Show memakai `table table-striped` label:value 1 kolom |
| `asset-disposal`, `stocktake` | Show memakai pola berbeda lagi |

Akibatnya:

1. Halaman yang semestinya "dokumen transaksional" (RTM/MTS/BAST/MI) tampil tidak seragam.
2. Banyak `inline style` hard-coded (`rgba(0,0,0,0.02)`, `rgba(255,255,255,0.05)`) yang **rusak di dark mode** dan tak terlihat kontras di light mode.
3. `@section('page_title')` sering didefinisikan padahal **layout induk tidak pernah me-render blok ini** (hanya `title`, `header_right`, `content`).
4. Tanggal di-render hard-code `format('d M Y H:i')` tanpa `Helper::getFormattedDateObject()`, sehingga tidak ter-lokalisasi.
5. Tidak ada breadcrumb pada modul kustom → navigasi kembali hilang.
6. Status badge di-render lewat `if/ternary` bersarang yang sulit dirawat dan mudah lupa menangani status baru (contoh: `Logistics_Rejected`).

### 1.2 Tujuan

1. Menetapkan **pola arsitektur view wajib** untuk seluruh modul kustom (index/show/create/edit).
2. Memaksa pemakaian **komponen Blade Snipe-IT** (`x-container`, `x-box`, `x-well`, `x-tabs`, `x-info-element`, `x-info-panel`, `x-icon`, `x-copy-to-clipboard`, `x-table.*`) dan melarang `inline style` untuk keperluan layout/warna.
3. Menetapkan **4 archetype halaman** agar implementor tinggal memilih: `INDEX`, `SHOW-DOKUMEN`, `SHOW-MASTER`, `FORM`.
4. Menyediakan **template kode kanonik** (copy-paste ready) + **checklist kepatuhan** untuk review.
5. Menyediakan **matriks audit** modul existing beserta prioritas remediasi.

### 1.3 Ruang Lingkup

Berlaku untuk seluruh view web (bukan API, bukan email, bukan print khusus):

- Modul transaksional: RTM (`asset-mutation-requests`), MTS (`asset-mutations`), BAST (`assetmutation/bast`), MI (`material-issuances`), PR (`purchase-requests`), RP (`procurement`).
- Modul master/siklus: `asset-disposal`, `stocktake`, modul baru lain yang mengikuti pola yang sama.
- Tidak mengubah: halaman core Snipe-IT (`hardware`, `users`, `licenses`, dst), template email, halaman kiosk mandiri.

### 1.4 Definisi Istilah

| Istilah | Arti |
|---------|------|
| Archetype | Pola kerangka halaman baku (INDEX / SHOW-DOKUMEN / SHOW-MASTER / FORM) |
| Manifest | Blok tabel berisi daftar item/detail dokumen (header-detail pattern) |
| Info-column | Kolom metadata `label: value` memakai `x-well` / `x-info-element` |
| `trans('key') ?? 'fallback'` | Pola wajib: selalu sediakan fallback string ketika kunci belum ada |

---

## 2. ATURAN WAJIB (HARD RULES)

### 2.1 Komentar Pelacakan Kustom

Setiap blok kode baru/modifikasi wajib diawali komentar dengan pola:

```php
{{-- KUSTOM erickalvino-{KODE}: Deskripsi singkat --}}
```

Contoh: `{{-- KUSTOM erickalvino-VB-01: Show detail dokumen RTM memakai archetype SHOW-DOKUMEN --}}`

### 2.2 Bahasa Tampilan

- Seluruh label memakai `trans('...')` yang didefinisikan minimal di `resources/lang/id-ID/general.php`.
- Setiap pemakaian `trans()` **wajib** diikuti `?? 'Fallback String'` agar tidak blank saat kunci belum ada.
- Tanggal tampilan memakai locale via `Helper::getFormattedDateObject()`.

### 2.3 Tema, Dark Mode, dan Aksesibilitas

1. **Dilarang** `style="..."` untuk warna, background, border, dan layout. Gunakan kelas AdminLTE/Snipe-IT (`well`, `callout`, `label`, `bg-*`, `text-*`, `table`, `box`) atau komponen Blade.
2. Satu-satunya `style` yang diperbolehkan: utilitas spacing minor (`style="margin-top: 5px;"`) bila tidak ada kelas setara, dan `style` dinamis berbasis data (warna tag/status).
3. Wajib `role="alert"` + `aria-live` pada callout penting (pola `hardware/view.blade.php`).
4. Tombol ikon wajib punya `title`/tooltip dan teks `sr-only` bila tidak ada teks.

---

## 3. ARSITEKTUR LAYER VIEW

### 3.1 Layout Induk

Semua halaman memakai `@extends('layouts/default')`. Layout ini menyediakan CSS variable `light-dark()` (dark mode), sidebar, header, dan kontainer breadcrumb.

### 3.2 Blok Section yang Tersedia

| Section | Fungsi | Wajib? |
|---------|--------|--------|
| `@section('title0')` | Judul kustom 1 (opsional, untuk judul bertingkat) | Tidak |
| `@section('title')` | Judul halaman — dirender di `<h1 class="pagetitle">` bila breadcrumb kosong | Ya |
| `@section('header_right')` | Tombol aksi utama, di kanan atas header halaman | Ya (bila ada aksi) |
| `@section('content')` | Isi halaman | Ya |
| `@section('moar_scripts')` | JavaScript tambahan (bootstrap-table, formatter) | Tidak |
| `@stack('css')` | CSS per-halaman | Tidak |

> Catatan: `@section('page_title')` **tidak pernah di-render** oleh `layouts/default.blade.php`. Dilarang memakai blok ini.

### 3.3 Breadcrumb

- Modul kustom wajib mendaftarkan breadcrumb `assetMutationRequests.show`, `assetMutations.show`, dst. di `routes/web.php` (pola route resource core).
- Bila breadcrumb belum terdaftar, halaman tetap harus menyediakan tombol `← Kembali` (`route('{module}.index')`) di `header_right`.

---

## 4. INVENTARIS KOMPONEN BLADE SNIPE-IT (WAJIB PAKAI)

| Komponen | Fungsi | Props penting |
|----------|--------|---------------|
| `<x-container>` | Wrapper `row` + `col` | `columns` (1 = otomatis `col-md-12`), `class` |
| `<x-page-column>` | Kolom grid | `class` (`col-md-8`, `col-md-4`, ...) |
| `<x-box>` | Kartu panel AdminLTE | `name`, `box_style` (`default`/`success`/dll), `header` (string judul), `route` (render `box-footer` otomatis), `customfooter` (slot footer kustom) |
| `<x-tabs>` | Tab container | slot `tabnav` + `tabpanes` |
| `<x-tabs.nav-item>` | Tab header | `name`, `label`, `count`, `icon_type`, `tooltip` |
| `<x-tabs.pane>` | Isi tab | `name`, slot `content` |
| `<x-well>` | Kolom metadata | slot isi label:value |
| `<x-info-element>` | Satu baris `label: value` | `title` (label), `icon_type`, `icon_color` |
| `<x-info-panel>` | Panel metadata + tombol aksi bawah | `infoPanelObj`, `buttons`, `before_list`, `after_list` |
| `<x-icon>` | Ikon FontAwesome via `Icon::icon($type)` | `type`, `class`, `title` |
| `<x-copy-to-clipboard>` | Tombol salin nilai | `copy_what` |
| `<x-table.index>` | Bootstrap Table (index) | `presenter` (JSON `dataTableLayout()`), `api_url`, `buttons`, `export_filename`, `sort_field`, `sort_order` |
| `<x-table.{module}>` | Tabel khusus modul (wrapper presenter + formatter) | `route`, `columns`, dll |
| `<x-notifications>` | Notifikasi halaman | — |
| `<x-button.info-panel-toggle>` | Tombol lihat semua info (collapsible) | — |

Contoh pemakaian terverifikasi di codebase:

```blade
{{-- hardware/view.blade.php & material-issuances/show.blade.php --}}
<x-container>
    <x-box name="material_issuance" box_style="default">
        <div class="row">
            <div class="col-md-8">
                <x-well> ... </x-well>
            </div>
            <div class="col-md-4">
                <x-well> ... </x-well>
            </div>
        </div>
    </x-box>
</x-container>
```

---

## 5. POLA HALAMAN (PAGE ARCHETYPES)

| Archetype | Rasio kolom | Dipakai untuk | Contoh |
|-----------|-------------|---------------|-------|
| **A. INDEX** | 1 kolom penuh | Daftar + filter + bulk action | `asset-mutation-requests/index` |
| **B. SHOW-DOKUMEN** | 1 kolom (`col-md-12`) | Detail dokumen transaksional: info-column + manifest + footer aksi | RTM, MTS, BAST, MI, PR |
| **C. SHOW-MASTER** | 2 kolom (`col-md-8` + `col-md-4`) opsional + `x-tabs` | Detail master yang punya banyak sub-tab (aktivitas, riwayat, dst) | `hardware/view.blade.php`, `users` core |
| **D. FORM** | 1 kolom (`col-md-12`) + wrapper box | Create/Edit | `material-issuances/edit` |

### 5.1 Archetype A — INDEX

```blade
{{-- KUSTOM erickalvino-VB-02: Archetype INDEX --}}
@extends('layouts/default')

@section('title0') {{ trans('general.asset_mutation_requests', [], 2) ?? 'Permintaan Mutasi' }} @stop
@section('title') @yield('title0') @parent @stop

@section('content')
    <x-container>
        <x-box name="asset_mutation_requests">
            <x-table.asset-mutation-requests :route="route('api.assetMutationRequests.index')" />
        </x-box>
    </x-container>
@stop

@section('moar_scripts')
    @include('partials.bootstrap-table')
@stop
```

Aturan INDEX:

1. Kolom tabel didefinisikan **sekali** di `App\Presenters\{Module}Presenter::dataTableLayout()` (format bootstrap-table JSON), bukan di JS view.
2. `x-table.{module}` bertugas me-render `<x-table.index :presenter="...Presenter::dataTableLayout()" :api_url="..." />` + mendaftarkan formatter milik modul.
3. Formatter status / tanggal / aksi CUKUP satu fungsi per jenis, tidak boleh duplikat label-badge di tiap view (lihat 6.3).
4. Tombol "Buat Baru" lewat `header_right` dengan `@can('create')`.
5. Endpoint API: `route('api.{module}.index')`.

### 5.2 Archetype B — SHOW-DOKUMEN (WAJIB UNTUK MODUL TRANSAKSIONAL)

Kerangka wireframe (tekstual):

```
┌──────────────────────────────────────────────────────────────┐
│ content-header: Breadcrumb > judul       [← Kembali] [Edit]   │
├──────────────────────────────────────────────────────────────┤
│ box.box-default (customfooter)                               │
│   box-header.with-border                                     │
│     box-title: <x-icon> Judul Dokumen   [label status]       │
│   box-body                                                  │
│     row                                                     │
│       well (col-md-12)                                      │
│         info-element: No. Dokumen      (copy-to-clipboard)  │
│         info-element: Tanggal Pengajuan / Tanggal Mutasi     │
│         info-element: Status                               │
│         info-element: Pengaju / Pemohon (link user)         │
│         info-element: Unit Asal / Tujuan (link location)    │
│         info-element: Catatan                              │
│       callout (bila perlu): alasan penolakan / peringatan   │
│     row                                                     │
│       box-title: Manifest (Jumlah: n)                       │
│       table.table.table-striped (satu tabel, kolom Tipe)    │
│   box-footer (customfooter)                                 │
│     [Tombol workflow sesuai @can & status]                  │
└──────────────────────────────────────────────────────────────┘
```

> Posisi aksi workflow **wajib di `box-footer`** (contextual action), bukan ditumpuk `pull-right` di header. Header hanya untuk aksi global: kembali + edit.

### 5.3 Archetype C — SHOW-MASTER (DUA KOLOM + TAB)

```
┌──────────────────────────────────────────────────────────────┐
│ box-header: Judul            [label status]                  │
├──────────────────────────────┬───────────────────────────────┤
│ col-md-8                      │ col-md-4                      │
│  x-tabs                       │  x-well (side info)           │
│   tab 1: Info Umum …          │  info-element…                │
│   tab 2: Aktivitas …          │                               │
└──────────────────────────────┴───────────────────────────────┘
```

Contoh: `hardware/view.blade.php`, `material-issuances/show.blade.php` (untuk keputusan "kolom apa saja yang masuk tab" lihat 6.2).

### 5.4 Archetype D — FORM (CREATE/EDIT)

1. Wrapper `@section('content')` + `<x-container>` + `<x-box>`.
2. `box-body` berisi kartu field per bagian; bagian memakai `col` + kelas form `form-control`.
3. Tombol simpan/batal di `box-footer` (`<x-save-button>`, `<x-cancel-button>`).
4. Setiap `<select>`/input relasi pastikan punya `autocomplete`/search (pola form asset).

---
## 6. ATURAN DETAIL BLUEPRINT SHOW

### 6.1 Delineasi Kolom Metadata

Kolom metadata dokumen dibagi **3 kelompok logis**; kelompok yang ruang lingkupnya pendek memakai `x-info-panel`, kelompok yang wajib tampil selalu memakai `x-well` + `x-info-element`.

| Kelompok | Isi | Kolom |
|----------|-----|-------|
| **Identitas Dokumen** | Status, No. Dokumen (copy), Tanggal transaksi (`request_date` / `mutation_date` / `handover_date`), User Pembuat (link) | wajib tampil |
| **Pihak & Rute** | Pengaju / Penerima / Pemohon (link), Lokasi Asal / Tujuan (link), Unit Bisnis, Kategori | x-well kanan |
| **Keterangan** | Catatan, Alasan Penolakan, Frekuensi tambahan | x-info-panel / callout |

Semua kolom yang model-nya benar-benar ada **wajib di-render**, tidak boleh disembunyikan tanpa alasan (contoh: `request_date` ada di model `AssetMutationRequest` tapi tidak ditampilkan di show — ini defisiten).

### 6.2 Pilihan Satu Kolom vs Dua Kolom + Tab

- **Satu kolom (`col-md-12`)**: jumlah metadata <= 10 field → pakai Archetype B.
- **Dua kolom + tab**: ada >= 2 sub-listing (misal manifest + riwayat aktivitas), atau metadata > 14 field → pakai Archetype C (`hardware/view.blade.php`).

### 6.3 Status Badge — SATU SUMBER KEBENARAN

Status **harus** dipetakan lewat satu array helper tunggal per modul (di Presenter atau view-partial `_status.blade.php`), **bukan** `if/ternary` bersarang yang disalin-salin antar view. Pemetaan wajib lengkap sehingga status baru tidak jatuh ke `label-default` diam-diam.

Peta warna baku (pola `_logistics_tracker_cell` / formatter index):

| Kelas | Status |
|-------|--------|
| `label-success` `bg-green` | `Approved`, `Processed`, `Completed`, `Returned`, `Received`, `CheckedOut` |
| `label-warning` `bg-yellow` | `Draft`, `Pending`, `PartiallyReturned`, `Pending Approval` |
| `label-info` `bg-aqua` | `In Progress`, `Submitted`, `Pending Logistics` |
| `label-danger` `bg-red` | `Rejected`, `Cancelled`, `Logistics_Rejected`, `Declined`, `Lost/Stolen` |
| `label-default` `bg-gray` | `Archived`, `Void`, status tak dikenal |

> Contoh bug historis: `Logistics_Rejected` jatuh ke `label-warning` padahal harus `label-danger`.

Dua tempat konfigurasi yang wajib sinkron:

1. **Presenter blob**: property/array `statusBadges()` memuat `['Logistics_Rejected' => 'danger', ...]`.
2. **JS formatter index**: memakai array yang sama (loop array, bukan switch manual).

### 6.4 Manifest Item (Header-Detail)

1. Seluruh item dirender dalam **satu tabel** `table.table.table-striped` di dalam `table-responsive`.
2. Jika item punya tipe campuran (`accessory` / `component` / `license` / `asset`), tambahkan **kolom `Tipe`** dengan `label` badge, bukan memecah jadi sub-box per tipe.
3. Kolom minimum manifest: `#`, `Tipe`, `Item` (link), `Jumlah` (badge), `Asal/Target` (tautan relasi), `Catatan`.
4. Empty state: satu baris `<td colspan="n">` dengan `callout callout-info` ("Tidak ada item terdaftar."), bukan `div` dashed bercustom-style.
5. Jumlah item tampilkan di `box-title` manifest (`{{ trans('general.items') ?? 'Item' }} ({{ $doc->items->count() }})`).

### 6.5 Aksi & Authorization

| Jenis Aksi | Lokasi | Aturan |
|------------|--------|--------|
| Navigasi global (`← Kembali`, `Edit`) | `header_right` | Selalu tampil |
| Workflow transisi status (Setujui / Tolak / Proses / Batalkan) | `box-footer` | Dibungkus `@can('update', $doc)` / `@can('approve')` dan digate state saat ini di controller |
| Aksi destruktif (Hapus) | `box-footer` | `@can('delete')`, wajib konfirmasi modal |
| Aksi kontekstual baris (lihat detail item) | baris tabel | `btn-default btn-sm` + tooltip |

Pola tombol box-footer (rtm/mts): tombol kiri = action primer, tombol kanan = secondary, urut: Cancel (default) → Submit (primary) → Approve (success) → Reject (danger, buka modal).

### 6.6 Modal Konfirmasi

- Satu modal per dokumen (`#rejectModal`, `#approveModal`, `#deleteModal`) di bagian akhir `box-body`.
- Modal memakai `x-confirm` style class AdminLTE (`modal`, `modal-dialog`, `modal-content`), field `textarea` untuk alasan, `hidden _token`.
- Form reject/kirim memakai `--form` dengan `data-action`; submit via JS `confirm` bawaan bila tanpa alasan.

---

## 7. KONVENSI RENDERING DATA

### 7.1 Format Tanggal

Wajib memakai `Helper::getFormattedDateObject($value)` (return `{ date, formatted }`) — memastikan locale, zona waktu, dan format konsisten dengan halaman core.

```blade
@php $requestDate = Helper::getFormattedDateObject($doc->request_date, 'date'); @endphp
<x-info-element title="{{ trans('general.request_date') ?? 'Tanggal Pengajuan' }}">
    <span data-value="{{ $requestDate['date'] }}">{{ $requestDate['formatted'] }}</span>
</x-info-element>
```

Dilarang: `$doc->created_at->format('d M Y H:i')` hard-code di view (kecuali print/export yang memang butuh format tetap).

### 7.2 Nilai Kosong

- Nilai nullable ditampilkan `-` (tidak blank, tidak `null`).
- Relasi yang null ditampilkan `-` (contoh: `$doc->approvedBy?->getFullNameAttribute() ?? '-'`).

### 7.3 Relasi & Link

| Relasi | Tampilan |
|--------|----------|
| User / pemohon | Link `route('users.show', ...)` |
| Aset (`hardware`) | Link `route('hardware.show', ...)` + `asset_tag` tebal |
| Lokasi | Link `route('locations.show', ...)` |
| Dokumen induk (RTM → MTS → BAST) | Link antar-route show modul bersangkutan |
| Nilai tanpa detail | Teks polos |

Atribut yang punya tooltip (`data-tooltip` + `data-title`) wajib memakai helper `<x-icon>` / pola `hardware/view.blade.php`.

### 7.4 Translasi & Fallback

Setiap label wajib: `trans('general.{key}') ?? 'Bahasa Indonesia'`. Ini mencegah halaman kosong di env dengan kunci lang belum sync.

---
## 8. TEMPLATE KODE KANONIK

### 8.1 Template INDEX (Archetype A — final)

```blade
{{-- KUSTOM erickalvino-VB-03: Archetype INDEX --}}
@extends('layouts/default')

@section('title0')
    {{ trans('general.{module_title}') ?? '{Nama Modul}' }}
@stop

@section('title')
    @yield('title0') @parent
@stop

@section('header_right')
    @can('create', \App\Models\{Model}::class)
        <a href="{{ route('{module}.create') }}" class="btn btn-primary pull-right">
            <x-icon type="new" />
            {{ trans('general.create') ?? 'Buat Baru' }}
        </a>
    @endcan
@stop

@section('content')
    <x-container>
        <x-box name="{module_plural}">
            <x-table.{module} :route="route('api.{module}.index')" />
        </x-box>
    </x-container>
@stop

@section('moar_scripts')
    @include('partials.bootstrap-table')
@stop
```

### 8.2 Template SHOW-DOKUMEN (Archetype B — final, 1 kolom)

```blade
{{-- KUSTOM erickalvino-VB-04: Archetype SHOW-DOKUMEN (1 kolom) --}}
@extends('layouts/default')

@section('title')
    {{ $doc->request_number ?? trans('general.{module_detail}') ?? 'Detail' }}
@stop

@section('header_right')
    <a href="{{ route('{module}.index') }}" class="btn btn-default">
        <i class="fa fa-arrow-left"></i>
        {{ trans('general.back') ?? 'Kembali' }}
    </a>
    @can('update', $doc)
        <a href="{{ route('{module}.edit', $doc->id) }}" class="btn btn-primary">
            <x-icon type="edit" />
            {{ trans('general.edit') ?? 'Edit' }}
        </a>
    @endcan
@stop

@section('content')
    <x-container>
        <x-box name="{module_singular}" box_style="default">
            {{-- box-header: judul + status --}}
            <div class="box-header with-border">
                <h2 class="box-title">
                    <x-icon type="file" />
                    {{ trans('general.{module_title}') ?? 'Detail' }}: {{ $doc->request_number ?? $doc->id }}
                </h2>
                <span class="label {{ $doc->statusBadgeClass() }} pull-right">{{ $doc->status }}</span>
            </div>

            <div class="box-body">
                {{-- Info-column --}}
                <div class="row">
                    <div class="col-md-12">
                        <x-well>
                            <x-info-element title="{{ trans('general.request_number') ?? 'No. Dokumen' }}" icon_type="file">
                                {{ $doc->request_number ?? '-' }}
                                <x-copy-to-clipboard :copy_what="$doc->request_number ?? ''" />
                            </x-info-element>
                            @php $reqDate = Helper::getFormattedDateObject($doc->request_date, 'date'); @endphp
                            <x-info-element title="{{ trans('general.request_date') ?? 'Tanggal Pengajuan' }}" icon_type="calendar">
                                {{ $reqDate['formatted'] ?? '-' }}
                            </x-info-element>
                            <x-info-element title="{{ trans('general.status') ?? 'Status' }}" icon_type="status">
                                <span class="label {{ $doc->statusBadgeClass() }}">{{ $doc->status ?? '-' }}</span>
                            </x-info-element>
                            @if ($doc->requestedBy)
                                <x-info-element title="{{ trans('general.requested_by') ?? 'Pengaju' }}" icon_type="user">
                                    <a href="{{ route('users.show', $doc->requested_by) }}">{{ $doc->requestedBy->getFullNameAttribute() }}</a>
                                </x-info-element>
                            @endif
                            <x-info-element title="{{ trans('general.notes') ?? 'Catatan' }}" icon_type="note">
                                {{ $doc->notes ?: '-' }}
                            </x-info-element>
                        </x-well>
                    </div>
                </div>
---
{{-- Callout rejection/peringatan --}}
                @if ($doc->status === 'Rejected' && $doc->rejection_note)
                    <div class="callout callout-danger" role="alert" aria-live="assertive">
                        <h4><i class="fa fa-ban"></i> {{ trans('general.rejection_note') ?? 'Alasan Penolakan' }}</h4>
                        <p>{{ $doc->rejection_note }}</p>
                    </div>
                @endif

                {{-- Manifest --}}
                <div class="row">
                    <div class="col-md-12">
                        <h3 class="box-title">
                            {{ trans('general.items') ?? 'Item' }} ({{ $doc->items->count() }})
                        </h3>
                        <div class="table-responsive">
                            <table class="table table-striped">
                                <thead>
                                    <tr>
                                        <th>#</th>
                                        <th>{{ trans('general.item_type') ?? 'Tipe' }}</th>
                                        <th>{{ trans('general.item') ?? 'Item' }}</th>
                                        <th class="text-center">{{ trans('general.quantity') ?? 'Jumlah' }}</th>
                                        <th>{{ trans('general.target_asset') ?? 'Asal/Tujuan' }}</th>
                                        <th>{{ trans('general.notes') ?? 'Catatan' }}</th>
                                    </tr>
                                </thead>
                                <tbody>
                                @forelse ($doc->items as $item)
                                    <tr>
                                        <td>{{ $loop->iteration }}</td>
                                        <td><span class="label label-info">{{ ucfirst($item->item_type) }}</span></td>
                                        <td><strong>{{ $item->asset->asset_tag ?? $item->item_name ?? '-' }}</strong></td>
                                        <td class="text-center"><span class="label label-success">{{ $item->qty }}</span></td>
                                        <td>{{ $item->targetAsset?->asset_tag ?? '-' }}</td>
                                        <td>{{ $item->notes ?: '-' }}</td>
                                    </tr>
                                @empty
                                    <tr>
                                        <td colspan="6">
                                            <div class="callout callout-info">{{ trans('general.no_results') ?? 'Tidak ada item terdaftar.' }}</div>
                                        </td>
                                    </tr>
                                @endforelse
                                </tbody>
                            </table>
                        </div>
                    </div>
                </div>

                {{-- Modal reject --}}
                <div id="rejectModal" class="modal fade" role="dialog">
                    <div class="modal-dialog">
                        <div class="modal-content">
                            <form action="{{ route('{module}.reject', $doc->id) }}" method="POST">
                                @csrf
                                <div class="modal-header">
                                    <button type="button" class="close" data-dismiss="modal">&times;</button>
                                    <h4 class="modal-title">{{ trans('general.reject') ?? 'Tolak' }}</h4>
                                </div>
                                <div class="modal-body">
                                    <label for="rejection_note">{{ trans('general.rejection_note') ?? 'Alasan Penolakan' }}</label>
                                    <textarea name="rejection_note" id="rejection_note" class="form-control" rows="3" required></textarea>
                                </div>
                                <div class="modal-footer">
                                    <button type="button" class="btn btn-default" data-dismiss="modal">{{ trans('general.close') ?? 'Tutup' }}</button>
                                    <button type="submit" class="btn btn-danger">{{ trans('general.reject') ?? 'Tolak' }}</button>
                                </div>
                            </form>
                        </div>
                    </div>
                </div>
            </div>{{-- /.box-body --}}

            {{-- box-footer: workflow actions --}}
            <div class="box-footer">
                @if ($doc->status === 'Draft')
                    <a href="{{ route('{module}.edit', $doc->id) }}" class="btn btn-default">{{ trans('general.edit') ?? 'Edit' }}</a>
                    @can('approve', $doc)
                        <form action="{{ route('{module}.approve', $doc->id) }}" method="POST" class="pull-left" style="margin-left: 5px;">
                            @csrf
                            <button class="btn btn-success" type="submit" onclick="return confirm('{{ trans('general.{module}_approve_confirm') ?? 'Setujui?' }}')">
                                <i class="fa fa-check"></i> {{ trans('general.approve') ?? 'Setujui' }}
                            </button>
                        </form>
                    @endcan
                    <button class="btn btn-danger" data-toggle="modal" data-target="#rejectModal">
                        <i class="fa fa-times"></i> {{ trans('general.reject') ?? 'Tolak' }}
                    </button>
                @endif
            </div>
        </x-box>
    </x-container>
@stop
```

> Catatan: `$doc->statusBadgeClass()` adalah accessor pada Model yang membungkus array status (satu sumber kebenaran, lihat 6.3). Implementasi array wajib menangkap `Logistics_Rejected` sebagai `danger`.

---
### 8.3 Template SHOW-MASTER (Archetype C — dua kolom + tab)

```blade
{{-- KUSTOM erickalvino-VB-05: Archetype SHOW-MASTER (2 kolom + tab) --}}
@extends('layouts/default')

@section('title') {{ $model->name ?? 'Detail' }} @stop

@section('header_right')
    <a href="{{ route('{module}.index') }}" class="btn btn-default">
        <i class="fa fa-arrow-left"></i> {{ trans('general.back') ?? 'Kembali' }}
    </a>
@stop

@section('content')
    <x-container>
        <x-box name="{module_singular}" box_style="default">
            <div class="box-header with-border">
                <h2 class="box-title"><x-icon type="file" /> {{ $model->name ?? '-' }}</h2>
                <span class="label {{ $model->statusBadgeClass() }} pull-right">{{ $model->status ?? '-' }}</span>
            </div>
            <div class="box-body">
                <div class="row">
                    <div class="col-md-8">
                        <x-tabs>
                            <x-slot name="tabnav">
                                <x-tabs.nav-item name="info" label="Info Umum" icon_type="info" />
                                <x-tabs.nav-item name="activity" label="Aktivitas" :count="$model->activities->count()" icon_type="clock" />
                            </x-slot>
                            <x-slot name="tabpanes">
                                <x-tabs.pane name="info">
                                    {{-- info-elements utama --}}
                                </x-tabs.pane>
                                <x-tabs.pane name="activity">
                                    {{-- table aktivitas / timeline --}}
                                </x-tabs.pane>
                            </x-slot>
                        </x-tabs>
                    </div>
                    <div class="col-md-4">
                        <x-well>
                            <x-info-element title="{{ trans('general.status') ?? 'Status' }}" icon_type="status">
                                <span class="label {{ $model->statusBadgeClass() }}">{{ $model->status ?? '-' }}</span>
                            </x-info-element>
                            <x-info-element title="{{ trans('general.created_at') ?? 'Dibuat' }}" icon_type="calendar">
                                {{ Helper::getFormattedDateObject($model->created_at, 'datetime')['formatted'] ?? '-' }}
                            </x-info-element>
                            {{-- metadata tambahan --}}
                        </x-well>
                    </div>
                </div>
            </div>
            <div class="box-footer">
                {{-- contextual workflow actions --}}
            </div>
        </x-box>
    </x-container>
@stop
```

---

## 9. CHECKLIST KEPATUHAN (CONFORMANCE CHECKLIST)

Pakai untuk review setiap view (index/show/edit) sebelum dianggap selesai:

### Struktur Umum
- [ ] `@extends('layouts/default')`
- [ ] Tidak ada `@section('page_title')`
- [ ] `@section('title')` terisi; `title0` opsional
- [ ] `header_right` berisi tombol kembali/edit yang relevan
- [ ] Konten dibungkus `<x-container>` + `<x-box>`
- [ ] Komentar `{{-- KUSTOM erickalvino-{KODE}: ... --}}` di tiap blok

### UI/UX
- [ ] Nol `inline style` warna/background/border (dark mode aman)
- [ ] Breadcrumb terdaftar, atau tersedia tombol kembali
- [ ] Status memakai satu sumber array (accessor `statusBadgeClass()` / formatter), `Logistics_Rejected` → `danger`
- [ ] Label memakai `trans('...') ?? 'fallback'`
- [ ] Tanggal via `Helper::getFormattedDateObject()`, bukan `format()` hard-code
- [ ] Nilai nullable tampil `-`
- [ ] Semua `@can` ada; aksi workflow di `box-footer`
- [ ] Manifest satu tabel + kolom Tipe; empty state `callout`
- [ ] Modal reject/delete pakai `modal` + `@csrf`
- [ ] Ikon pakai `<x-icon>` / `fa` + tooltip; atribut penting punya `role="alert"`

### Index
- [ ] Kolom dari `{Module}Presenter::dataTableLayout()`, formatter dimiliki modul (x-table wrapper)
- [ ] `@include('partials.bootstrap-table')`
- [ ] API url `route('api.{module}.index')`

---

## 10. MATRIKS AUDIT MODUL EXISTING

Status saat dokumen ini ditulis (August 2026):

| Modul | Index | Show | Masalah prioritas |
|-------|-------|------|-------------------|
| RTM `asset-mutation-requests` | ✅ (x-table + formatter) | ⚠️ raw div + inline | no breadcrumb; status map `Logistics_Rejected` salah; `request_date` tak tampil; aksi di header; 4 sub-box pecah |
| MTS `asset-mutations` | ✅ | ⚠️ table label:value | tak ada manifest; aksi di header |
| BAST `assetmutation/bast` | ✅ | ✅ pola box | sinkronisasi callout + manifest tab |
| MI `material-issuances` | ✅ | ⚠️ inline style padat | dark mode rusak; manifest pakai inline style; tak memakai `x-well` |
| PR `purchase-requests` | ✅ | ⚠️ table 1 kolom | deliniasi kolom tak sesuai blueprint |
| RP `procurement` | ✅ | ⚠️ | cek dark mode; manifest |
| `asset-disposal` | ✅ | ⚠️ | cek archetype B |
| `stocktake` | ✅ + kiosk | ⚠️ | cek archetype C (ada kiosk terpisah) |

Prioritas remediasi: **RTM → MTS → MI → PR** (dokumen transaksional paling sering dipakai).

---

## 11. DEFINISI SELESAI (DoD)

Sebuah view dinyatakan selesai bila:

1. Lolos seluruh checklist bagian 9.
2. Tampil benar di **light mode dan dark mode** (verifikasi manual, tidak ada kotak gelap/terang tak kontras).
3. Tidak ada warning/error Blade; tidak ada query N+1 baru (pakai Eager Loading relasi yang di-render).
4. Status baru otomatis aman: menambah nilai enum **tanpa edit view** tidak menghasilkan `label-default` yang salah wajar atau halaman blank.
5. Kode view di-review memakai archetype yang dipilih dan tidak copy-paste inline style lintas modul.

---
**Lampiran wajib dibaca**: `resources/views/hardware/view.blade.php` (Archetype C core), `resources/views/material-issuances/show.blade.php` (Archetype B campuran), `resources/views/blade/{box,well,info-element,info-panel,tabs,table,container}.blade.php` (API komponen).
