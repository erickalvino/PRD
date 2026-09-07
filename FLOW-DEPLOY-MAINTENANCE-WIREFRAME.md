# WIREFRAME FLOW — MODUL DEPLOYMENT & MAINTENANCE (SNIPE-IT v8.6.3)

> Sumber: `PRD-DEPLOY-SERVICE-REVISI.md` (Revisi 2.1)
> Modul 1: **Asset Deployment & Setup**
> Modul 2: **Custom Maintenance & Life-Cycle Evaluation**
> Semua transisi status = **POST + CSRF + Policy/Gate + `lockForUpdate()`**, dan dicatat ke `custom_module_logs`.

---

## 0. PANORAMA KEDUA MODUL

```
                        ┌──────────────────────────────────────────────┐
                        │              ASSET (Snipe-IT core)           │
                        │  asset_tag / company_id / location_id /      │
                        │  status_id / assigned_to / qty_stok          │
                        └──────────────┬───────────────┬───────────────┘
                                       │               │
              MAINTENANCE diagnostic   │               │  DEPLOYMENT setup
              (rusak/perlu servis)     │               │  (pasang/serah terima ke dept)
                                       ▼               ▼
                ┌────────────────────┐              ┌──────────────────────┐
                │  MODUL MAINTENANCE │              │  MODUL DEPLOYMENT    │
                │  custom_maintenances│              │  asset_deployments   │
                └─────────┬──────────┘              └──────────┬──────────┘
                          │                                    │
        COMPLETED ────────┼──────────────► asset kembali normal / ke target dept
                          │                                    │
        UNREPAIRABLE ─────┼──────────────► QUARANTINE (karantina BER)
                          │                                    │
                          └────────────────────► bahan untuk replacement berikutnya

Aktor:
  SUPER   = Super Admin            (semua aksi)
  FA      = Fixed Asset / GA       (buat draft, terbitkan, approve/void, print)
  IT      = IT Technician          (diagnosis, servis, QC, kanibalisasi)
  BR-ADMIN= Branch Admin           (view + export data cabang sendiri)
  USER    = End User               (tidak ada akses modul ini)
```

---

## 1. MODUL DEPLOYMENT — STATE MACHINE

```
                 ┌─────────┐
                 │  DRAFT  │  (dibuat oleh Creator/FA)
                 └────┬────┘
      ┌───────────────┼───────────────────────┐
      │               │                       │
   ubah/hapus     terbitkan               void (wajib alasan >=15)
   (Creator/FA)   (Creator/FA)            (Creator/FA)
      │               │                       │
      │               ▼                       ▼
      │      ┌──────────────────┐      ┌───────────┐
      │      │ PENDING_HANDOVER │      │ CANCELLED │  (terminal)
      │      └────────┬─────────┘      └───────────┘
      │               │
      │            start progress
      │               │  (IT/FA)
      │               ▼
      │      ┌──────────────┐
      │      │ IN_PROGRESS  │
      │      └──────┬───────┘
      │             │ QC passed — markReady (IT/FA)
      │             ▼
      │      ┌──────────────────┐
      │      │ READY_TO_RETURN  │
      │      └────┬─────────────┘
      │           │
      │     ┌─────┴─────┐
      │     │           │
      │  FA approve   FA void
      │     │           │
      │     ▼           ▼
      │  ┌─────────┐  ┌───────────┐
      └─►│COMPLETED│  │ CANCELLED │  (keduanya TERMINAL)
         └─────────┘  └───────────┘

Side-effect wajib:
  DRAFT       -> simpan historical company/location dari BranchResolver
  PENDING_HANDOVER -> reserve()  + terbitkan document_number
  COMPLETED   -> consume()  + install item ke asset + proses detached items
  CANCELLED   -> release() + alasan pembatalan
```

---

## 2. MODUL DEPLOYMENT — FLOW LENGKAP (Detail)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STEP A — CREATE / DRAFT (route: assetDeployments.store)                        │
│                                                                                 │
│  Input form                                                                     │
│  ├─ asset_id (TAG aset utama)                                                   │
│  ├─ target_department_id  (bagian/departemen TITIK TUJUAN; dari tabel core      │
│  │                        `departments`, WAJIB diisi user)                      │
│  ├─ assigned_technician_id (nullable, teknisi penanggung jawab)                 │
│  ├─ allocated_items[] : {type: COMPONENT|ACCESSORY|LICENSE, id, qty}            │
│  └─ detached_items[]  : {type, id, qty, condition: GOOD|BROKEN,                 │
│                          release_license_seat, notes}                           │
│                                                                                 │
│  Controller → DB::transaction {                                                 │
│    ├─ BranchResolver::fromAsset(asset)  → company_id, location_id               │
│    │     (asal: asset.company_id / asset.location_id;                            │
│    │      null  → CUSTOM_DEFAULT_*; kedua-duanya null → GAGAL)                   │
│    ├─ create asset_deployments(status=DRAFT, created_by=auth.user)               │
│    ├─ create allocated_items[] + detached_items[]                                │
│    ├─ log(custom_module_logs: DRAFT)                                             │
│  }                                                                              │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │  status = DRAFT  (Creator-only editable)
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STEP B — TERBITKAN (releaseToPending → POST assetDeployments.release)           │
│                                                                                 │
│  Policy: edit + creator check (Creator/FA)                                      │
│  DB::transaction {                                                              │
│    ├─ StockReservationService->reserve(deployment)                              │
│    │    ├─ lockForUpdate() komponen/aksesoris/license_seats                      │
│    │    ├─ cek availableQty >= qty → bila kurang → RuntimeException             │
│    │    └─ tabel asset_deployment_reservations → status=RESERVED                 │
│    ├─ DocumentNumberService->next('deployment', location, id) → BAS/BAST number  │
│    ├─ status = PENDING_HANDOVER                                                  │
│    └─ log(DRAFT → PENDING_HANDOVER)                                             │
│  }                                                                              │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STEP C — START PROGRESS (startProgress → POST assetDeployments.start)           │
│  PENDING_HANDOVER → IN_PROGRESS     (IT/FA)                                      │
│  + log + (reset/update assigned_technician bila perlu)                           │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STEP D — QC / READY (markReady → POST assetDeployments.ready)                   │
│  IN_PROGRESS → READY_TO_RETURN    (IT/FA)                                       │
│  + log (hasil QC / catatan pemasangan)                                          │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STEP E — COMPLETE / SERAH TERIMA (complete → POST assetDeployments.complete)    │
│  Policy: approve (FA) — hanya dari READY_TO_RETURN                              │
│                                                                                 │
│  DB::transaction {                                                              │
│    ├─ StockReservationService->consume()  RESERVED → CONSUMED                   │
│    ├─ untuk tiap allocated_items:                                               │
│    │    COMPONENT → installComponent(asset, item)                               │
│    │    ACCESSORY → AccessoryHandler->assignToAsset(item.id, asset)             │
│    │                (INSERT accessories_checkout assigned_type=Asset)          │
│    │    LICENSE   → LicenseSeatHandler->assignSeatToAsset(item.id, asset)       │
│    │                (license_seats.asset_id=asset.id, assigned_to=user asset)   │
│    ├─ untuk tiap detached_items:                                                │
│    │    GOOD   → returnToStock(item)                                            │
│    │    BROKEN → QuarantineService->attachBrokenComponent(asset, item,...)      │
│    │              → components_assets ke quarantine_asset_id                    │
│    ├─ status = COMPLETED                                                        │
│    └─ log(READY_TO_RETURN → COMPLETED)                                          │
│  }                                                                              │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 ▼
                        ┌───────────────────┐
                        │  COMPLETED (END)  │
                        │  aset terpasang di│
                        │  dept target      │
                        └───────────────────┘
```

---

## 3. MODUL DEPLOYMENT — CABANG VOID (CANCEL)

```
DRAFT / PENDING_HANDOVER / IN_PROGRESS / READY_TO_RETURN
              │  FA — POST assetDeployments.cancel (VoidRequest)
              ▼
┌──────────────────────────────────────────────────────────────┐
│ DB::transaction {                                            │
│   ├─ capture old status                                      │
│   ├─ status = CANCELLED                                      │
│   ├─ cancellation_notes = input (WAJIB >= 15 karakter)       │
│   ├─ StockReservationService->release(deployment)            │
│   │    RESERVED → RELEASED, released_at = now()              │
│   └─ log(old → CANCELLED, note alasan)                       │
│ }                                                            │
└──────────────────────────────┬───────────────────────────────┘
                               ▼
                      ┌─────────────┐
                      │ CANCELLED   │  (terminal; stok kembali tersedia)
                      └─────────────┘
```

---

## 4. MODUL MAINTENANCE — STATE MACHINE

```
                 ┌─────────┐
                 │  DRAFT  │  (dibuat oleh Creator/FA)
                 └────┬────┘
      ┌───────────────┼───────────────────────┐
      │               │                       │
   ubah/hapus     terbitkan               void (alasan >=15)
   (Creator/FA)   (Creator/FA)            (FA)
      │               │                       │
      │               ▼                       ▼
      │      ┌──────────────────┐      ┌───────────┐
      │      │ PENDING_CHECKING │      │ CANCELLED │
      │      └────────┬─────────┘      └───────────┘
      │               │
      │         menerima diagnosis (IT)
      │               ▼
      │      ┌──────────────────┐
      │      │ UNDER_DIAGNOSIS  │
      │      └───────┬──────────┘
      │               │
      │    ┌──────────┴──────────┐
      │    │                     │
      │ internal servis       vendor
      │ (IT)                  (FA/IT)
      │    │                     │
      │    ▼                     ▼
      │ ┌──────────────────┐ ┌──────────────────┐
      │ │IN_SERVICE_INTERNAL│ │ OUT_TO_VENDOR    │
      │ └────────┬─────────┘ └────────┬─────────┘
      │          └──────────┬─────────┘
      │                     │ QC (IT)
      │                     ▼
      │            ┌──────────────────┐
      │            │ POST_SERVICE_QC  │
      │            └────────┬─────────┘
      │                     │ good → readyToReturn (IT/FA)
      │                     ▼
      │            ┌──────────────────┐
      │            │ READY_TO_RETURN  │
      │            └────┬─────────────┘
      │                 │
      │       ┌─────────┴─────────┐
      │       │                   │
      │    FA approve good    FA approve BER
      │       │                   │
      │       ▼                   ▼
      │  ┌─────────┐          ┌───────────────┐
      └─►│COMPLETED│          │ UNREPAIRABLE  │
         └─────────┘          └───────────────┘
                        (keduanya TERMINAL)

Side-effect wajib:
  DRAFT                     -> simpan historical company/location + hitung warranty
  PENDING_CHECKING          -> EXTERNAL: terbitkan document_number (SJP)
  COMPLETED                 -> apply kanibalisasi (registered consume + attach / unregistered log)
  UNREPAIRABLE              -> QuarantineService->moveAssetToQuarantine()
  CANCELLED                 -> alasan pembatalan (tidak ada reservasi stok khusus di modul ini)
```

---

## 5. MODUL MAINTENANCE — FLOW LENGKAP (Detail)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STEP A — CREATE / DRAFT (route: customMaintenances.store)                       │
│                                                                                 │
│  Input form                                                                     │
│  ├─ asset_id (TAG aset yang bermasalah)                                         │
│  ├─ maintenance_type : INTERNAL | EXTERNAL                                      │
│  ├─ is_warranty_claim (bool)                                                    │
│  ├─ supplier_id (nullable → wajib bila EXTERNAL)                                │
│  ├─ assigned_technician_id (nullable)                                           │
│  ├─ estimated_completion_date, issue_description                                │
│  └─ cannibal_items[] : {donor_asset_id, type: COMPONENT|ACCESSORY,              │
│                        id?, part_name/part_spec?, is_unregistered_donor, qty}   │
│                                                                                 │
│  Controller → DB::transaction {                                                 │
│    ├─ BranchResolver::fromAsset(asset) → company_id, location_id                │
│    ├─ create custom_maintenances(                                               │
│    │    status=DRAFT, warranty_status_at_launch='UNKNOWN')                      │
│    ├─ detectWarrantyStatus()  → ACTIVE | EXPIRED | UNKNOWN (setelah model ada)  │
│    ├─ create cannibal_items[]  status=PLANNED                                   │
│    └─ log(DRAFT)                                                                │
│  }                                                                              │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │  status = DRAFT (Creator-only editable)
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STEP B — TERBITKAN (releaseToPendingChecking → POST customMaintenances.release) │
│                                                                                 │
│  Policy: edit + creator check (Creator/FA)                                      │
│  DB::transaction {                                                              │
│    ├─ bila maintenance_type = EXTERNAL:                                          │
│    │      document_number = DocumentNumberService->next('maintenance',...)       │
│    ├─ status = PENDING_CHECKING                                                  │
│    └─ log(DRAFT → PENDING_CHECKING)                                             │
│  }                                                                              │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STEP C — DIAGNOSIS (acceptDiagnosis → POST customMaintenances.diagnosis)        │
│  PENDING_CHECKING → UNDER_DIAGNOSIS     (IT)                                    │
│  + isi diagnosis / repair_action / repair_cost awal                              │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STEP D — PILIH JALUR PERBAIKAN                                                  │
│                                                                                 │
│   UNDER_DIAGNOSIS                                                                │
│        ├─ startInternal (POST customMaintenances.internal)  → IN_SERVICE_INTERNAL│
│        │    (IT, servis dikerjakan internal)                                    │
│        └─ sendToVendor  (POST customMaintenances.vendor)    → OUT_TO_VENDOR      │
│             (FA/IT, kirim ke supplier)                                          │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STEP E — QC (postServiceQc → POST customMaintenances.qc)                        │
│   IN_SERVICE_INTERNAL / OUT_TO_VENDOR → POST_SERVICE_QC    (IT)                 │
│   + hasil QC, repair_action, repair_cost                                        │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STEP F — READY (readyToReturn → POST customMaintenances.ready)                  │
│   POST_SERVICE_QC → READY_TO_RETURN    (IT/FA)                                  │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STEP G — APPROVE FINAL                                                          │
│                                                                                 │
│  READY_TO_RETURN                                                                │
│    ├─ complete (POST customMaintenances.complete)  → COMPLETED  (FA)            │
│    │    DB::transaction {                                                       │
│    │      ├─ apply tiap cannibal_items:                                         │
│    │      │    COMPONENT + registered   → StockEngine->consumeComponent(        │
│    │      │                                item_id, qty) + attachComponent(     │
│    │      │                                target_asset, 'Kanibalisasi resmi')  │
│    │      │    COMPONENT + unregistered → logUnregisteredInstall(target, item)  │
│    │      │    item.status = APPLIED                                            │
│    │      ├─ repair_action default 'Perbaikan selesai.'                         │
│    │      ├─ status = COMPLETED                                                 │
│    │      └─ log(READY_TO_RETURN → COMPLETED)                                   │
│    │    }                                                                       │
│    │                                                                             │
│    └─ rejectToUnrepairable (POST customMaintenances.unrepairable) → UNREPAIRABLE│
│         (FA) — Wajib unrepairable_reason_code + unrepairable_notes             │
│         DB::transaction {                                                       │
│           ├─ QuarantineService->moveAssetToQuarantine(asset, location_id)       │
│           │    ├─ ambil CUSTOM_QUARANTINE_STATUS_ID (status_labels)             │
│           │    ├─ pindahkan ke quarantine_location_id cabang                    │
│           │    └─ tulis Actionlog core Snipe-IT (CUSTOM_QUARANTINE)             │
│           ├─ status = UNREPAIRABLE                                              │
│           └─ log(READY_TO_RETURN → UNREPAIRABLE)                                │
│         }                                                                        │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. MODUL MAINTENANCE — CABANG VOID (CANCEL)

```
DRAFT / PENDING_CHECKING / UNDER_DIAGNOSIS / IN_SERVICE_INTERNAL /
OUT_TO_VENDOR / POST_SERVICE_QC / READY_TO_RETURN
              │  FA — POST customMaintenances.cancel (VoidRequest)
              ▼
┌──────────────────────────────────────────────────────────────┐
│ status = CANCELLED                                           │
│ cancellation_notes (>= 15 karakter)                          │
│ log(old → CANCELLED)                                         │
│ → jika dokumen EXTERNAL sudah terbit, nomor SJP tetap tercatat│
│   tetapi transaksi ditutup tanpa memindahkan aset             │
└──────────────────────────────────────────────────────────────┘
```

---

## 7. SIDE-EFFECT STOK & ITEM (RINGKASAN)

| Module | Fase | Item | Efek |
|---|---|---|---|
| Deployment | Publish (`release`) | COMPONENT / ACCESSORY / LICENSE | `RESERVED` + lock stok; tersedia dikurangi logis |
| Deployment | Complete | COMPONENT | `installComponent()` — pasang ke aset |
| Deployment | Complete | ACCESSORY | `accessories_checkout` (`assigned_type=Asset`) |
| Deployment | Complete | LICENSE | `license_seats.asset_id` + `assigned_to` user aset |
| Deployment | Complete | detached GOOD | kembali ke stok |
| Deployment | Complete | detached BROKEN | masuk `components_assets` pada `quarantine_asset_id` |
| Deployment | Void | semua yang RESERVED | `RELEASED` + `released_at` |
| Maintenance | Complete | cannibal registered | `StockEngine->consumeComponent()` + `attachComponent()` |
| Maintenance | Complete | cannibal unregistered | log part fisik (tanpa stok catalog) |
| Maintenance | UNREPAIRABLE | ASET | pindah ke status/lokasi karantina |

---

## 8. ALUR PAGE / UI PER MODUL

```
MODUL DEPLOYMENT
  INDEX  (A)  asset-deployments/index
       └─ x-table.asset-deployments → presenter dataTableLayout
            ├─ kolom: document_number, branch_location, status, created_at, actions
            ├─ tombol: create (header_right @can)
            └─ formatter: link / status / actions POST
  CREATE (D) asset-deployments/create
       └─ x-box: asset_id, target_department_id, assigned_technician_id,
                 allocated_items repeater, detached_items repeater
  SHOW   (B)  asset-deployments/show
       ├─ well: no dokumen, cabang, status, dept target, teknisi, pembuat
       ├─ manifest: allocated_items + detached_items (satu tabel, kolom Tipe)
       └─ box-footer: release | start | ready | complete | void
  EDIT   (D)  asset-deployments/edit  (hanya DRAFT)

MODUL MAINTENANCE
  INDEX  (A)  custom-maintenances/index
       └─ x-table.custom-maintenances → presenter dataTableLayout
  CREATE (D)  custom-maintenances/create
       └─ x-box: asset_id, maintenance_type, warranty, supplier, techn,
                 issue, cannibal_items repeater
  SHOW   (B)  custom-maintenances/show
       ├─ well: no dokumen, cabang, tipe, garansi, supplier, teknisi, issue
       ├─ manifest: cannibal_items
       └─ box-footer: release | diagnosis | internal/vendor | qc | ready |
                      complete | unrepairable | void
  EDIT   (D)  custom-maintenances/edit  (hanya DRAFT)
```

---

## 9. POIN NON-NEGOTIABLE (SESUAI PRD)

1. Tidak ada hardcoded ID/code cabang — semua lewat `config/custom.php`, `.env`, `BranchResolver`.
2. `company_id` + `location_id` di transaksi = **historical lock** dari aset; `target_department_id` = **input user** (departemen tujuan, dari tabel `departments`).
3. Transisi hanya via POST, tidak pernah GET.
4. Void wajib alasan >= 15 karakter.
5. Stock reservation: RESERVED → RELEASED (void) / CONSUMED (complete).
6. Idempotent + `lockForUpdate()` + unique constraint.
7. Semua transisi tercatat di `custom_module_logs`.
