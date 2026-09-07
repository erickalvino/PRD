# WIREFRAME FLOW — MODUL PEMINJAMAN ASET INTERNAL (ASSET LOAN & REQUEST)

> Sumber: `PRD-PEMINJAMAN-REVISI.md` (Revisi 2.1)
> Platform: Snipe-IT v8.6.3
> Tujuan: mengelola peminjaman aset ber-TAG dan item komposit (accessory / component / license) lintas cabang
> Semua transisi status = **POST + CSRF + Policy/Gate + `lockForUpdate()`**, dicatat ke `custom_module_logs` (`module_context = 'LOAN'`).

---

## 0. PANORAMA MODUL LOAN

```
                          ┌────────────────────────────────────────┐
                          │          SNIPE-IT v8.6.3 (core)         │
                          │  assets / accessories / components /    │
                          │  licenses / license_seats / users /     │
                          │  locations / status_labels              │
                          └────────────────┬───────────────────────┘
                                           │
                        ┌──────────────────┴──────────────────┐
                        │        MODUL PEMINJAMAN INTERNAL     │
                        │        custom_asset_loans            │
                        │  ├─ custom_asset_loan_items          │
                        │  ├─ custom_asset_loan_reservations   │
                        │  └─ custom_loan_compensations        │
                        └──────────────────┬───────────────────┘
                                           │
                  ┌────────────────────────┴────────────────────┐
                  │                                              │
        DRAFT → PENDING_APPROVAL → ON_LOAN → PENDING_RETURN_QC  │
                  │                                              │
                  │                         ┌────────────────────────┐
                  │                         │  RETURNED              │
                  │                         │  (MATCH+GOOD / comp)   │
                  │                         └────────────────────────┘
                  │                                              │
                  └──────────→ CANCELLED  (stok di-rollback)     ┘

Item yang dapat dipinjam:
  - asset      (aset ber-TAG, unik, status aset diubah ke "Dipinjam")
  - accessory  (aksesoris, via accessories_checkout)
  - component  (komponen, qty dari components.qty)
  - license    (seat lisensi, via license_seats)
Item komposit: accessory/component/license dapat dikaitkan ke `parent_asset_id`
               (aset induk yang membawa item tersebut).
```

---

## 1. STATE MACHINE FINAL

```
                        ┌─────────┐
                        │  DRAFT  │   (Creator/FA)
                        └────┬────┘
        ┌────────────────────┼──────────────────────┐
        │                    │                      │
   ubah/hapus            release                 void
   (Creator-only)        (Creator/FA)            (Creator/FA)
        │                    │                      │
        │                    ▼                      ▼
        │          ┌──────────────────┐      ┌───────────┐
        │          │ PENDING_APPROVAL │      │ CANCELLED │  (terminal)
        │          └────────┬─────────┘      └───────────┘
        │                   │
        │               approve
        │                   │  (Supervisor/Approve)
        │                   ▼
        │          ┌──────────────────┐
        │          │     ON_LOAN      │
        │          └────────┬─────────┘
        │                   │  return received (FA/IT)
        │                   ▼
        │          ┌──────────────────────┐
        │          │ PENDING_RETURN_QC    │
        │          └──────┬───────────────┘
        │                 │  completeReturn (FA/IT / QC)
        │                 │
        │        ┌────────┴──────────────┐
        │        │                       │
        │   all MATCH & GOOD       ada MISSING/EXCHANGED
        │        │                       │
        │        ▼                       ▼
        │  ┌─────────┐        ┌──────────────────────┐
        │  │ RETURNED│        │ PENDING_COMPENSATION │
        │  │ (stok   │        └──────────┬───────────┘
        │  │  restock)│                   │  issue & approve BA
        │  └─────────┘                   │  (FA/Supervisor)
        │                                ▼
        │                       ┌───────────────┐
        │                       │   RETURNED    │
        │                       └───────────────┘
        │
        └── ON_LOAN / PENDING_RETURN_QC / PENDING_COMPENSATION
                     │  void (FA/Supervisor)
                     ▼
              ┌───────────┐
              │ CANCELLED │  (terminal; stok di-rollback)
              └───────────┘
```

---

## 2. FLOW LENGKAP MODUL LOAN

### STEP A — CREATE / DRAFT (`customLoans.store`)

```
┌───────────────────────────────────────────────────────────────────────────────┐
│  Input form                                                                   │
│  ├─ borrower_id                (user peminjam)                                │
│  ├─ planned_checkout_date      (rencana keluar)                               │
│  ├─ planned_return_date        (rencana kembali)                              │
│  └─ items[]:                                                                  │
│       ├─ item_type : asset | accessory | component | license                 │
│       ├─ asset_id / parent_asset_id / accessory_id / component_id / license_id│
│       └─ qty_borrowed                                                         │
│                                                                               │
│  Controller → DB::transaction {                                               │
│    ├─ BranchResolver::fromAssetOrOperator(borrower, operator)                 │
│    │     → company_id + location_id (historical lock)                         │
│    ├─ create custom_asset_loans(status=DRAFT)                                  │
│    ├─ create custom_asset_loan_items[]                                        │
│    │     qty_returned=0, return_check_status=MATCH, condition=GOOD            │
│    └─ log(custom_module_logs: DRAFT)                                          │
│  }                                                                            │
└────────────────────────────────┬──────────────────────────────────────────────┘
                                 │ status = DRAFT (creator-only save/edit)
                                 ▼
```

### STEP B — RELEASE / AJUKAN (`customLoans.release`)

```
┌───────────────────────────────────────────────────────────────────────────────┐
│  Policy: edit + creator check (Creator/FA)                                    │
│  DB::transaction {                                                            │
│    ├─ assertNoDoubleBooking()                                                 │
│    │     → asset TAG tidak boleh ada di dokumen aktif lain                    │
│    │       (PENDING_APPROVAL / ON_LOAN / PENDING_RETURN_QC /                  │
│    │        PENDING_COMPENSATION)                                             │
│    ├─ LoanReservationService->reserve(loan)                                   │
│    │    ├─ lockForUpdate() components / accessories / license_seats           │
│    │    ├─ cek available Qty >= qty_borrowed                                  │
│    │    └─ custom_asset_loan_reservations → status = RESERVED                 │
│    ├─ DocumentNumberService->nextLoan() → nomor dokumen LP                    │
│    ├─ status = PENDING_APPROVAL                                               │
│    └─ log(DRAFT → PENDING_APPROVAL)                                           │
│  }                                                                            │
└────────────────────────────────┬──────────────────────────────────────────────┘
                                 ▼
```

### STEP C — APPROVE → ON_LOAN (`customLoans.approve-loan`)

```
┌───────────────────────────────────────────────────────────────────────────────┐
│  Policy: approve (Supervisor/Approve) — hanya dari PENDING_APPROVAL           │
│  DB::transaction {                                                            │
│    ├─ LoanReservationService->consume(loan)                                   │
│    │    ├─ asset      → markAssetOnLoan: assets.status_id = loanStatusId()    │
│    │    ├─ accessory  → accessories.qty - qty; accessories_checkout           │
│    │    ├─ component  → components.qty - qty                                  │
│    │    └─ license    → LicenseSeatHandler (seat ter-assign)                  │
│    │    └─ reservation status = CONSUMED, consumed_at=now()                   │
│    ├─ status = ON_LOAN, actual_checkout_date = now(), approved_by = user      │
│    └─ log(PENDING_APPROVAL → ON_LOAN)                                         │
│  }                                                                            │
└────────────────────────────────┬──────────────────────────────────────────────┘
                                 │ barang keluar
                                 ▼
```

### STEP D — RETURN RECEIVED (`customLoans.return-received`)

```
┌───────────────────────────────────────────────────────────────────────────────┐
│  Policy: edit (FA/IT) — hanya dari ON_LOAN                                    │
│  DB::transaction {                                                            │
│    ├─ status = PENDING_RETURN_QC (QC wajib; tidak bisa langsung RETURNED)     │
│    └─ log(ON_LOAN → PENDING_RETURN_QC)                                        │
│  }                                                                            │
└────────────────────────────────┬──────────────────────────────────────────────┘
                                 ▼
```

### STEP E — COMPLETE RETURN / QC (`customLoans.complete-return`)

```
┌───────────────────────────────────────────────────────────────────────────────┐
│  Policy: edit (FA/IT QC) — hanya dari PENDING_RETURN_QC                       │
│  Input per item: qty_returned, return_check_status (MATCH/MISSING/EXCHANGED),  │
│                  condition_at_return (GOOD/BROKEN), final_status_label_id,     │
│                  notes                                                         │
│                                                                               │
│  DB::transaction {                                                            │
│    ├─ loop items: update hasil QC                                              │
│    ├─ jika ada MISSING / EXCHANGED →                                          │
│    │    loop items:                                                           │
│    │      create custom_loan_compensations(status=DRAFT)                      │
│    │      missing_qty = qty_borrowed - qty_returned                           │
│    │    status = PENDING_COMPENSATION                                          │
│    │    log(PENDING_RETURN_QC → PENDING_COMPENSATION)                          │
│    │    redirect → halaman kompensasi                                          │
│    ├─ jika semua MATCH & GOOD →                                               │
│    │    LoanReservationService->restock(loan)                                 │
│    │      → increment component/accessory, release seat lisensi,              │
│    │        asset status kembali ke initial_status_label_id                    │
│    │      → reservation status = RESTOCKED, restocked_at=now()                │
│    │    status = RETURNED, actual_return_date = now()                         │
│    │    log(PENDING_RETURN_QC → RETURNED)                                     │
│    └─ jika ada BROKEN (MATCH) →                                               │
│         → status = RETURNED                                                     │
│         → stok tidak di-restock; komponen/aset dipindah ke karantina           │
│  }                                                                            │
└────────────────────────────────┬──────────────────────────────────────────────┘
                                 │
                 ┌───────────────┴────────────────┐
                 ▼                                ▼
        ┌────────────────────┐          ┌──────────────────────────┐
        │  RETURNED (normal) │          │ PENDING_COMPENSATION     │
        └────────────────────┘          │  (MISSING/EXCHANGED)     │
                                        └────────────┬─────────────┘
                                                     ▼
```

### STEP F — COMPENSATION (`customLoans.compensation`)

```
┌───────────────────────────────────────────────────────────────────────────────┐
│  Policy: approve (FA/Supervisor) — hanya dari PENDING_COMPENSATION            │
│  Input: amount, currency, notes                                               │
│  DB::transaction {                                                            │
│    ├─ update semua compensation:                                              │
│    │    amount, currency, ba_number = BAG/<tahun>/<seq>                       │
│    │    status = ISSUED                                                       │
│    ├─ LoanReservationService->restock(loan)                                   │
│    │    (kembalikan stok untuk item MATCH; item MISSING/EXCHANGED tidak       │
│    │     di-restock karena sudah diganti rugi)                                │
│    ├─ status = RETURNED, actual_return_date = now()                           │
│    └─ log(PENDING_COMPENSATION → RETURNED)                                    │
│  }                                                                            │
└────────────────────────────────┬──────────────────────────────────────────────┘
                                 ▼
                       ┌──────────────────┐
                       │  RETURNED (END)  │
                       └──────────────────┘
```

---

## 3. CABANG VOID / CANCEL (`customLoans.cancel`)

```
DRAFT / PENDING_APPROVAL / ON_LOAN / PENDING_RETURN_QC / PENDING_COMPENSATION
              │  Policy: void (FA/Supervisor) — VoidRequest
              ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│  Pola:                                                                        │
│    releasedAlready = (status == ON_LOAN || status == PENDING_RETURN_QC)       │
│  DB::transaction {                                                            │
│    ├─ status = CANCELLED                                                      │
│    ├─ cancellation_notes (WAJIB >= 15 karakter)                               │
│    ├─ jika releasedAlready:                                                   │
│    │    LoanReservationService->rollbackStock(loan)                           │
│    │      → asset status restored ke initial_status_label_id                   │
│    │      → component/accessory qty + qty_borrowed                             │
│    │      → license seat dilepas                                               │
│    │      → reservation status = RELEASED, released_at=now()                   │
│    └─ jika masih RESERVED (DRAFT/PENDING_APPROVAL):                           │
│         LoanReservationService->release(loan)                                  │
│           → reservation status = RELEASED                                      │
│    └─ log(old → CANCELLED)                                                    │
│  }                                                                            │
└────────────────────────────────┬──────────────────────────────────────────────┘
                                 ▼
                       ┌─────────────┐
                       │ CANCELLED   │  (terminal; stok dikembalikan/dilepas)
                       └─────────────┘
```

---

## 4. SIDE-EFFECT STOK & ITEM PER FASE

| Fase | Item | Efek ke Snipe-IT core |
|---|---|---|
| `release` (RESERVE) | asset | tidak diubah qty; cek double-booking |
| `release` (RESERVE) | component/accessory | lock row; cek ketersediaan; `RESERVED` |
| `release` (RESERVE) | license | cek seat bebas; `RESERVED` |
| `approve` (CONSUME) | asset | `assets.status_id` → status "Dipinjam" |
| `approve` (CONSUME) | component | `components.qty` - qty |
| `approve` (CONSUME) | accessory | `accessories.qty` - qty; `accessories_checkout` |
| `approve` (CONSUME) | license | `license_seats` ter-assign (user/aset) |
| `return` (RESTOCK) | asset | status kembali ke `initial_status_label_id` |
| `return` (RESTOCK) | component/accessory | qty + qty_returned (hanya MATCH+GOOD) |
| `return` (RESTOCK) | license | seat dilepas (hanya MATCH+GOOD) |
| `BROKEN` (MATCH) | component/asset | tidak restock; pindah ke karantina |
| `MISSING/EXCHANGED` | semua | tidak restock; buat kompensasi |
| `CANCELLED` | semua | rollback / release sesuai status saat void |

---

## 5. ALUR PAGE / UI MODUL LOAN

```
INDEX  (Archetype A)  custom-loans/index
  └─ x-table.custom-loans (route api.customLoans.index)
       ├─ kolom: document_number, branch_location, borrower, planned_dates,
       │         status, actions
       └─ tombol create di header_right (@can create)

CREATE (Archetype D)  custom-loans/create
  └─ x-box custom_loan
       ├─ borrower, planned_checkout_date, planned_return_date
       └─ items repeater (asset / accessory / component / license + parent_asset)

SHOW   (Archetype B)  custom-loans/show
  ├─ x-well: no dokumen, cabang, rencana, peminjam, status
  ├─ manifest item: satu tabel (Tipe, Item, Qty, Status Kembali, Kondisi, Catatan)
  └─ box-footer: release | approve-loan | return-received | complete-return |
                 compensation | cancel

EDIT   (Archetype D)  custom-loans/edit  (hanya DRAFT)
EDIT-CHECKING (Archetype D)  custom-loans/edit-checking
  ├─ input QC per item: qty_returned, return_check_status, condition_at_return,
  │                     final_status_label_id, notes
  └─ submit → customLoans.complete-return

VOID-MODAL
  └─ cancellation_notes (min 15 karakter)
```

---

## 6. POIN NON-NEGOTIABLE (SESUAI PRD)

1. Tanpa hardcoded ID/code/status label — semua dari `config/custom.php`, `.env`, `BranchResolver`, `LoanStatusResolver`.
2. `company_id` + `location_id` = historical lock; **tidak diubah** setelah dokumen keluar dari `DRAFT`.
3. Stok di-*reserve* saat `PENDING_APPROVAL`, di-*consume* saat `ON_LOAN`, di-*restock* saat `RETURNED` (MATCH+GOOD).
4. Aset ber-TAG ganda dicek dengan `assertNoDoubleBooking`.
5. QC wajib lewat `PENDING_RETURN_QC`; tidak bisa langsung `RETURNED` dari `ON_LOAN`.
6. `MISSING`/`EXCHANGED` memblokir penutupan sampai kompensasi `ISSUED`.
7. Void wajib `cancellation_notes` >= 15 karakter.
8. Semua transisi memakai POST + CSRF + Policy + `lockForUpdate()` + idempotent.
9. Semua transisi tercatat di `custom_module_logs` dengan `module_context = 'LOAN'`.
