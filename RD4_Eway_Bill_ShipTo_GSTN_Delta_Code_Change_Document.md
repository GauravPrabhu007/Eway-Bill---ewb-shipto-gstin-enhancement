# Delta Code Change Document — E-way Bill Ship-to GSTN Enhancement (RD4)

| Field | Value |
|---|---|
| **Document Type** | Delta Code Change Document |
| **Target System** | RD4 (SAP ECC) |
| **Target Client** | 225 |
| **Module** | Logistics — E-way Bill (Custom) |
| **Reference System** | RD2 (implemented) |
| **Reference Transports (RD2)** | RD2K9A5LAN (DDIC / API structure), RD2K9A5LGZ (ABAP logic / config) |
| **Change ID (RD2)** | CD:8089362 |
| **Author (RD2 reference)** | Omkar More |
| **Document Version** | 1.2 |
| **Date** | 15-Jun-2026 |
| **Scope** | Non-IRN E-way Bill API generation only |

---

## 1. Executive Summary

This document describes the **delta changes required on RD4** to implement the Ship-to GSTN enhancement for E-way Bill creation, aligned with the solution already delivered on **RD2**.

Two new fields are introduced:

| # | Field Label | API JSON Key | Table Field (`ZSCE_EWB_HDR`) | Length |
|---|---|---|---|---|
| 1 | Ship To GSTIN | `shiptogstin` | `SHIP_TO_GSTIN` | 18 |
| 2 | Ship To Trade Name | `shiptotradename` | `SHIP_TO_TRADENAME` | 100 |

Fields are populated in the **NIC API JSON payload** and persisted to **`ZSCE_EWB_HDR`** only when the corresponding **`ZLOG_EXEC_VAR`** parameter is **active** for the applicable **transaction type** (`1`, `2`, `3`, or `4`).

**Important:** This enhancement applies **only to the non-IRN path** inside `ZCL_EWB_API=>GENERATE_E_WAY_BILL`. IRN-based E-way Bill generation (`Z_SCM_IRN_EWB_GENERATION`) is **out of scope**.

**API field clarification (RD2):** `df_gstin` and `sh_gstn` are **not sent** in the NIC API JSON payload. On RD2, the mapping lines to `lw_header-df_gstin` / `lw_header-sh_gstn` are **commented out**, and the API structure `ZST_EWAY_BILL_GENERATION_REQ` does not contain these components. Ship GSTIN is exposed via the new API keys `shiptogstin` / `shiptotradename` instead. Internal structures (`ZEWB_CREATE`) and table `ZSCE_EWB_HDR` may still hold `df_gstin` / `sh_gstn` for internal use — see Section 5.3.

---

## 2. Business Requirement

When Bill-to and Ship-to parties differ, the E-way Bill API must send Ship-to GSTIN and Ship-to Trade Name as separate fields in the generation request, and store them in the E-way Bill header table for audit and reporting.

Activation must be **configurable per transaction type** via `ZLOG_EXEC_VAR`, so business can control which transaction scenarios include these fields without further code changes.

---

## 3. RD4 Current State vs Target State

### 3.1 Current State on RD4 (as-is)

| Object | RD4 Status |
|---|---|
| `ZCL_EWB_API` | Exists (package `ZLOG`) — ship-to logic **not confirmed** (source access restricted) |
| `ZSCE_EWB_HDR` | Exists — `SHIP_TO_GSTIN` / `SHIP_TO_TRADENAME` **not present** (per object search) |
| `ZSCM_SHIPTOGSTN_DE` | **Not found** |
| `ZSCM_SHIPTOTRADE_DE` | **Not found** |
| `ZST_EWAY_BILL_GENERATION_REQ1` | **Not found** (only `ZST_EWAY_BILL_GENERATION_REQ` exists) |
| `Z_SCE_EWB_BOT_CREATE_OB` | Exists — already maps `sh_gstn`, `df_gstin`, `sub_supl_desc`, `trans_ind` to `ZEWB_CREATE` |
| `Z_SCE_EWB_BOT_SERVICE_CALL` | Exists — calls `ZCL_EWB_API=>GENERATE_E_WAY_BILL` (non-IRN API path) |
| `Z_SCE_EWB_DATA_SAVE` | Exists — simpler version; does **not** map ship-to fields (same as RD2 design) |
| `Z_SCE_EWB_INV_DTL_FETCH` | Exists — **does not** derive `sh_gstnid` / `sh_name1` from ship-to (`WE`) party (unlike RD2) |
| `ZLOG_EXEC_VAR` entries | `ZSCE_EWB_API_SHIPTO_GSTIN` / `ZSCE_EWB_API_SHIPTO_TRDNAME` — **to be created** |

### 3.2 Target State on RD4 (to-be)

| Layer | Target |
|---|---|
| **DDIC** | New data elements, extended `ZSCE_EWB_HDR`, new API structure `ZST_EWAY_BILL_GENERATION_REQ1` |
| **ABAP** | `ZCL_EWB_API` enhanced in non-IRN branch with parameter-driven ship-to mapping |
| **ABAP (upstream)** | `Z_SCE_EWB_INV_DTL_FETCH` — ship-to (`WE`) party fetch for `sh_gstnid`, `sh_name1`, address fields |
| **Config** | `ZLOG_EXEC_VAR` rows maintained per transaction type |
| **API** | JSON payload includes `shiptogstin` and `shiptotradename` when parameters are active |
| **Persistence** | `ZSCE_EWB_HDR-SHIP_TO_GSTIN` and `SHIP_TO_TRADENAME` updated before API call |

---

## 4. Scope

### 4.1 In Scope

- Dictionary objects (data elements, table append, API structure)
- Class `ZCL_EWB_API` — method `GENERATE_E_WAY_BILL` (non-IRN `ELSE` branch only)
- Function `Z_SCE_EWB_INV_DTL_FETCH` — ship-to party data fetch (RD2-aligned)
- Customizing table `ZLOG_EXEC_VAR` — parameter maintenance for RD4 client 225
- Unit / integration testing for non-IRN billing documents

### 4.2 Out of Scope

| Item | Reason |
|---|---|
| IRN E-way Bill path (`Z_SCM_IRN_EWB_GENERATION`) | RD2 enhancement explicitly excluded from IRN flow |
| `Z_SCE_EWB_BOT_CREATE_OB` | No RD2 change; already passes `sh_gstn` and `to_other_party_name` |
| `Z_SCE_EWB_BOT_SERVICE_CALL` | No RD2 change |
| `Z_SCE_EWB_DATA_SAVE` | Ship-to persistence handled in `ZCL_EWB_API` on RD2 |
| BOT queue path (`ZSCE_EWB_BOT_ACTIVE = X`) | Uses legacy BOT interface, not NIC JSON API |
| Full RD2 `INV_DTL_FETCH` parity (STO/BOE/Nepal/export edge cases) | Port **core ship-to block only** for RD4; extended RD2 scenarios in Section 8.6 |

---

## 5. Solution Architecture

### 5.1 E-way Bill Creation Flow (RD4)

```
Z_SCE_EWB_INV_DTL_FETCH
        │
        ▼
Z_SCE_EWB_BOT_CREATE_OB  /  Z_SCE_EWB_BOT_SERVICE_CALL
        │
        ├──► Z_SCE_EWB_DATA_SAVE  ──►  ZSCE_EWB_HDR (initial insert)
        │
        └──► ZLOG_EXEC_VAR check: ZSCE_EWB_BOT_ACTIVE
                    │
                    ├── Active (X)  ──► BOT Queue (legacy — no ship-to JSON change)
                    │
                    └── Inactive      ──► ZCL_EWB_API=>GENERATE_E_WAY_BILL
                                              │
                                              ├── IRN exists + ZSCM_EWB_IRN_ACTIVE = X
                                              │         └──► Z_SCM_IRN_EWB_GENERATION  [OUT OF SCOPE]
                                              │
                                              └── ELSE (Non-IRN path)  [THIS ENHANCEMENT]
                                                        ├── Read ZLOG_EXEC_VAR ship-to params
                                                        ├── Map shiptogstin / shiptotradename
                                                        ├── UPDATE ZSCE_EWB_HDR
                                                        └── Serialize JSON via ZST_EWAY_BILL_GENERATION_REQ1
```

### 5.2 Data Mapping (Non-IRN API Input)

| API Output Field | Source (`ZEWB_CREATE` / `im_data-ewb_create`) | Set By (existing RD4 code) |
|---|---|---|
| `shiptogstin` | `sh_gstn` | `Z_SCE_EWB_BOT_CREATE_OB` ← `lw_bot_final_header-sh_gstnid` |
| `shiptotradename` | `to_other_party_name` | `Z_SCE_EWB_BOT_CREATE_OB` ← `lw_bot_final_header-sh_name1` |
| `transactiontype` | `trans_ind` | `Z_SCE_EWB_BOT_CREATE_OB` ← `lw_bot_final_header-trans_ind` |

### 5.3 API JSON Field Changes — `df_gstin` & `sh_gstn` (RD2 Reference)

On **RD2**, `df_gstin` and `sh_gstn` are **removed from the NIC API JSON call**. This is a deliberate part of the RD2K9A5LAN design and must be replicated on RD4.

#### 5.3.1 What is NOT in the API JSON

The API serialization structure `ZST_EWAY_BILL_GENERATION_REQ` (and `ZST_EWAY_BILL_GENERATION_REQ1`) does **not** include `df_gstin` or `sh_gstn` as components. The NIC API uses standard keys such as `fromgstin`, `togstin`, and — when ship-to params are active — `shiptogstin`, `shiptotradename`.

| Internal field (`ZEWB_CREATE`) | Former / intended API key | RD2 API JSON status | Replacement in API |
|---|---|---|---|
| `df_gstin` | `df_gstin` (never in `REQ` structure) | **Not sent** — mapping commented out | Dispatch GSTIN reflected via `fromgstin` / `togstin` override logic (`ZSCM_EWB_DISP_GSTIN`, `ZSCM_EWB_SHIP_GSTIN` params) |
| `sh_gstn` | `sh_gstn` (never in `REQ` structure) | **Not sent** — mapping commented out | Ship GSTIN sent as **`shiptogstin`** when `ZSCE_EWB_API_SHIPTO_GSTIN` is active |

#### 5.3.2 Commented-out code on RD2 (`ZCL_EWB_API`)

The following assignments exist on RD2 but are **commented out** and must **remain commented out** on RD4:

```abap
*        lw_header-df_gstin = lw_input_data-df_gstin.
*        lw_header-sh_gstn  = lw_input_data-sh_gstn.

" In transaction type 1 / 2 / 3 GSTIN override block:
*              lw_header-df_gstin = lw_log_exec_var1-remarks.
*              lw_header-sh_gstn  = lw_log_exec_var1-remarks.
*              lw_header-df_gstin = lw_log_exec_var1-remarks.
*                lw_header-sh_gstn  = lw_log_exec_var2-remarks.
```

Instead, when `lv_ship_gstin_active = abap_true`, ship GSTIN override uses:

```abap
lw_header_1-shiptogstin = lw_log_exec_var1-remarks.  " trans type 1
lw_header_1-shiptogstin = lw_log_exec_var2-remarks.  " trans type 2 / 3
```

#### 5.3.3 What still uses `df_gstin` / `sh_gstn` (internal — not API)

| Layer | `df_gstin` / `sh_gstn` usage | Change required on RD4? |
|---|---|---|
| `ZEWB_CREATE` structure | Fields still exist; populated by `Z_SCE_EWB_BOT_CREATE_OB` | **No** — keep as-is (internal carrier) |
| `ZSCE_EWB_HDR` table | `DF_GSTIN`, `SH_GSTN` columns may exist for DB persistence | **No API change** — separate from JSON payload |
| `ZCL_EWB_API` JSON build | **Not serialized** to NIC API | **Yes** — ensure mappings stay commented; use `shiptogstin` path |
| NIC API JSON | Uses `fromgstin`, `togstin`, `shiptogstin`, `shiptotradename` | Align with RD2 |

#### 5.3.4 RD4 implementation note

During `ZCL_EWB_API` porting, verify on RD4:

1. No active (uncommented) `lw_header-df_gstin` or `lw_header-sh_gstn` assignments exist.
2. `sh_gstn` from `im_data-ewb_create` is consumed **only** as the source for `lw_header_1-shiptogstin` (when param active), not as a direct API JSON key.
3. `df_gstin` is **not** passed to the API JSON; dispatch-related GSTIN continues to flow through `fromgstin` / `togstin` logic.

### 5.4 End-to-End Value Flow (when params are active)

```
VBPA (PARVW = 'WE') + KNA1 + ADRC
        │  Z_SCE_EWB_INV_DTL_FETCH  [NEW on RD4]
        ▼
ZLOG_EWB_S header: sh_gstnid, sh_name1, sh_name2/3, sh_post_code1, sh_region
        │  Z_SCE_EWB_BOT_CREATE_OB  [existing RD4]
        ▼
ZEWB_CREATE: sh_gstn, to_other_party_name
        │  ZCL_EWB_API (param active)  [NEW on RD4]
        ▼
API JSON: shiptogstin, shiptotradename  +  ZSCE_EWB_HDR: SHIP_TO_GSTIN, SHIP_TO_TRADENAME
```

| Stage | Field | Source |
|---|---|---|
| 1 — `INV_DTL_FETCH` | `sh_gstnid` | Ship-to customer `KNA1-STCD3` (`VBPA-PARVW = 'WE'`) |
| 1 — `INV_DTL_FETCH` | `sh_name1` | Ship-to address `ADRC-NAME1` |
| 2 — `BOT_CREATE_OB` | `sh_gstn` | `lw_bot_final_header-sh_gstnid` |
| 2 — `BOT_CREATE_OB` | `to_other_party_name` | `lw_bot_final_header-sh_name1` |
| 3 — `ZCL_EWB_API` | `shiptogstin` | `im_data-ewb_create-sh_gstn` (when `ZSCE_EWB_API_SHIPTO_GSTIN` active) |
| 3 — `ZCL_EWB_API` | `shiptotradename` | `im_data-ewb_create-to_other_party_name` (when `ZSCE_EWB_API_SHIPTO_TRDNAME` active) |

> **Note:** Parameter activation controls **inclusion in API JSON** (Stage 3). Stages 1–2 must populate values **before** API call — otherwise fields appear with blank values.

---

## 6. Dictionary Changes (Transport 1 — DDIC)

> Suggested RD4 transport description: **RD4 — EWB Ship-to GSTN — DDIC**

### 6.1 Data Element: `ZSCM_SHIPTOGSTN_DE`

| Attribute | Value |
|---|---|
| Description | Ship To GSTN |
| Data Type | CHAR |
| Length | 18 |
| Reference | Copy from RD2 |

### 6.2 Data Element: `ZSCM_SHIPTOTRADE_DE`

| Attribute | Value |
|---|---|
| Description | Ship To Trade Name |
| Data Type | CHAR |
| Length | 100 |
| Reference | Copy from RD2 |

### 6.3 Table Append: `ZSCE_EWB_HDR`

Append the following fields after existing fields `DF_GSTIN`, `SH_GSTN`, `SUB_SUPL_DESC`:

| # | Field Name | Data Element | Description |
|---|---|---|---|
| 76 | `SHIP_TO_GSTIN` | `ZSCM_SHIPTOGSTN_DE` | Ship To GSTIN (API + persistence) |
| 77 | `SHIP_TO_TRADENAME` | `ZSCM_SHIPTOTRADE_DE` | Ship To Trade Name (API + persistence) |

**Activation:** Ensure table adjustment / DB sync is executed in RD4 after transport import.

### 6.4 Structure: `ZST_EWAY_BILL_GENERATION_REQ1`

Create new structure based on `ZST_EWAY_BILL_GENERATION_REQ` with two additional components:

| Component | Type | Purpose |
|---|---|---|
| `shiptogstin` | `ZEWB_STRING` | API JSON field — Ship To GSTIN |
| `shiptotradename` | `ZEWB_STRING` | API JSON field — Ship To Trade Name |

> **Note:** RD4 currently has `ZST_EWAY_BILL_GENERATION_REQ` only. `ZST_EWAY_BILL_GENERATION_REQ1` must be **newly created** on RD4.

**Reference structure definition (RD2):**

```abap
define structure zst_eway_bill_generation_req1 {
  " ... all fields from zst_eway_bill_generation_req ...
  shiptogstin     : ZEWB_STRING;
  shiptotradename : ZEWB_STRING;
}
```

---

## 7. ABAP Changes — `ZCL_EWB_API` (Transport 2 — Workbench)

> Suggested RD4 transport description: **RD4 — EWB Ship-to GSTN — ABAP Logic**

### 7.1 Object Changed

| Object | Type | Method / Area | Package |
|---|---|---|---|
| `ZCL_EWB_API` | Class | `GENERATE_E_WAY_BILL` — non-IRN `ELSE` branch | `ZLOG` |

### 7.2 Prerequisite — Confirm `df_gstin` / `sh_gstn` Not in API JSON

Before applying ship-to blocks, confirm RD4 `ZCL_EWB_API` matches RD2 baseline:

- `lw_header-df_gstin` and `lw_header-sh_gstn` assignments are **commented out** (or absent).
- API JSON is built from `ZST_EWAY_BILL_GENERATION_REQ` / `REQ1` only — neither structure contains `df_gstin` or `sh_gstn`.
- If RD4 has **active** `lw_header-df_gstin` / `lw_header-sh_gstn` lines, **comment them out** as part of this change (align with RD2 TR RD2K9A5LAN).

---

### 7.3 Change Block 1 — Local Declarations

**Location:** Method `GENERATE_E_WAY_BILL`, local data section (near existing `c_name`, `c_name1`, `c_name2` constants)

**Add constants:**

```abap
CONSTANTS:
  lc_shipto_gstin   TYPE rvari_vnam VALUE 'ZSCE_EWB_API_SHIPTO_GSTIN',
  lc_shipto_trdname TYPE rvari_vnam VALUE 'ZSCE_EWB_API_SHIPTO_TRDNAME'.
```

**Add variables:**

```abap
DATA:
  lv_ship_gstin_active TYPE abap_bool,
  lv_ship_togstin      TYPE string,
  lv_to_tradename      TYPE string.

" Ensure existing declaration includes:
DATA:
  lw_header_1 TYPE zst_eway_bill_generation_req1.
```

---

### 7.4 Change Block 2 — Extend `ZLOG_EXEC_VAR` Selection

**Location:** Existing `SELECT` on `lt_log_exec_var` (non-IRN branch, near `c_name` / `c_name1` / `c_name2` read)

**Modify WHERE clause** to include ship-to parameter names:

```abap
SELECT name active remarks
  FROM zlog_exec_var
  INTO TABLE lt_log_exec_var
  WHERE ( name = c_name
       OR name = c_name1
       OR name = c_name2
       OR name = lc_shipto_gstin      " <<< ADD
       OR name = lc_shipto_trdname )  " <<< ADD
    AND active = abap_true.
```

---

### 7.5 Change Block 3 — Parameter Check and Field Mapping

**Location:** After assignment of `lw_header-transactiontype` and `lw_header-subsupplydesc`

```abap
CLEAR lv_ship_gstin_active.

READ TABLE lt_log_exec_var TRANSPORTING NO FIELDS
  WITH KEY name    = lc_shipto_gstin
           remarks = lw_header-transactiontype.
IF sy-subrc = 0.
  lw_header_1-shiptogstin = lw_input_data-sh_gstn.
  lv_ship_gstin_active    = abap_true.
ENDIF.

READ TABLE lt_log_exec_var TRANSPORTING NO FIELDS
  WITH KEY name    = lc_shipto_trdname
           remarks = lw_header-transactiontype.
IF sy-subrc = 0.
  lw_header_1-shiptotradename = lw_input_data-to_other_party_name.
ENDIF.
```

**Behaviour:**

| Parameter active for `transactiontype` | Effect |
|---|---|
| `ZSCE_EWB_API_SHIPTO_GSTIN` only | Sets `shiptogstin`; enables HDR update and `REQ1` JSON serialization |
| `ZSCE_EWB_API_SHIPTO_TRDNAME` only | Sets `shiptotradename` only (no HDR update / no `REQ1` switch unless GSTIN param also active) |
| Both active | Both fields populated |

---

### 7.6 Change Block 4 — GSTIN Override for Transaction Types 1 / 2 / 3

**Location:** Existing block using `c_name1` (`ZSCM_EWB_DISP_GSTIN`) and `c_name2` (`ZSCM_EWB_SHIP_GSTIN`) remarks

**Add inside existing transaction type conditions:**

```abap
" Transaction type 1 — when ACTTOSTATECODE = TOSTATECODE
IF lv_ship_gstin_active EQ abap_true.
  lw_header_1-shiptogstin = lw_log_exec_var1-remarks.
ENDIF.

" Transaction type 2 or 3
IF lv_ship_gstin_active = abap_true.
  lw_header_1-shiptogstin = lw_log_exec_var2-remarks.
ENDIF.
```

---

### 7.7 Change Block 5 — Persist to `ZSCE_EWB_HDR` and JSON Serialization

**Location:** Before `zui_cl_json=>serialize`, after `lw_header-itemlist` is populated

**Replace unconditional serialize with:**

```abap
CLEAR lv_json.

IF lv_ship_gstin_active = abap_true.

  CALL FUNCTION 'ENQUEUE_EZSCE_EWB_HDR'
    EXPORTING
      mode_zsce_ewb_hdr = lc_e
      mandt             = sy-mandt
      req_id            = lv_req_id
    EXCEPTIONS
      OTHERS            = 1.

  IF sy-subrc = 0.
    UPDATE zsce_ewb_hdr CLIENT SPECIFIED
      SET ship_to_gstin     = lw_header_1-shiptogstin
          ship_to_tradename = lw_header_1-shiptotradename
          changed_by        = sy-uname
          changed_on        = sy-datum
          changed_tm        = sy-uzeit
      WHERE mandt  = sy-mandt
        AND req_id = lv_req_id.

    CALL FUNCTION 'DEQUEUE_EZSCE_EWB_HDR'
      EXPORTING
        mode_zsce_ewb_hdr = lc_e
        mandt             = sy-mandt
        req_id            = lv_req_id.
  ENDIF.

  CLEAR: lv_ship_togstin, lv_to_tradename.
  lv_ship_togstin = lw_header_1-shiptogstin.
  lv_to_tradename = lw_header_1-shiptotradename.
  MOVE-CORRESPONDING lw_header TO lw_header_1.
  lw_header_1-shiptogstin     = lv_ship_togstin.
  lw_header_1-shiptotradename = lv_to_tradename.

  zui_cl_json=>serialize(
    EXPORTING
      data        = lw_header_1
      compress    = abap_false
      pretty_name = zui_cl_json=>pretty_mode-low_case
    RECEIVING
      r_json      = lv_json ).

ELSE.

  zui_cl_json=>serialize(
    EXPORTING
      data        = lw_header
      compress    = abap_false
      pretty_name = zui_cl_json=>pretty_mode-low_case
    RECEIVING
      r_json      = lv_json ).

ENDIF.
```

---

### 7.8 IRN Gate (No Change — Reference Only)

Ship-to logic must remain inside the **`ELSE`** branch of:

```abap
IF lv_irn IS NOT INITIAL AND lv_irn_ewb_active = abap_true.
  CALL FUNCTION 'Z_SCM_IRN_EWB_GENERATION'
  " ... IRN path — DO NOT add ship-to logic here ...
ELSE.
  " ... non-IRN path — all ship-to changes go here ...
ENDIF.
```

---

## 8. ABAP Changes — `Z_SCE_EWB_INV_DTL_FETCH` (Transport 3 — Workbench)

> Suggested RD4 transport description: **RD4 — EWB Ship-to GSTN — INV_DTL_FETCH**
>
> **Reference:** RD2 `Z_SCE_EWB_INV_DTL_FETCH` — ship-to party block (approx. lines 4225–4388, CD:8089362 context)
>
> **Insertion point on RD4:** Immediately **after** the existing bill-to / `gstn_rec` block and **before** `"@Item details"` (after `lw_header-rec_region` assignment, ~line 1111).

### 8.1 Purpose

Populate ship-to party fields on `ZLOG_EWB_S` work header so that downstream objects can supply values when `ZLOG_EXEC_VAR` ship-to parameters are active:

| `ZLOG_EWB_S` field | Populated from | Required for |
|---|---|---|
| `sh_gstnid` | `KNA1-STCD3` (ship-to / `WE` partner) | `shiptogstin` via `sh_gstn` |
| `sh_name1` | `ADRC-NAME1` (ship-to address) | `shiptotradename` via `to_other_party_name` |
| `sh_name2`, `sh_name3` | `ADRC-NAME2/3` | `to_address1/2` in API (existing BOT mapping) |
| `sh_city1`, `sh_post_code1`, `sh_region` | `ADRC` ship-to | BOT validation (`sh_post_code1` check) |

**Design principle:** **Add** ship-to logic; **do not modify** the existing `gstn_rec` / `rec_*` (bill-to) block.

### 8.2 RD4 Gap vs RD2 (current)

| Item | RD4 (as-is) | RD2 (reference) |
|---|---|---|
| `lty_vbpa` structure | `vbeln`, `posnr`, `kunnr`, `adrnr` only | Includes **`parvw`** |
| `VBPA` SELECT | Single role from `ZYTTPARA` / `PARTNER` | **`PARTNER` + `WE`** in range |
| `gstn_rec` assignment | `lw_header-gstn_rec = lw_kna1-stcd3` | Same (bill-to path) |
| Ship-to block | **Missing** | `READ lt_vbpa` with `parvw = 'WE'` → `sh_gstnid`, `sh_name1`, etc. |

### 8.3 Change Block 1 — Extend `lty_vbpa` Local Type

**Location:** Local types section (near existing `BEGIN OF lty_vbpa`)

**Modify:**

```abap
TYPES: BEGIN OF lty_vbpa,
         vbeln TYPE vbeln,
         posnr TYPE posnr,
         parvw TYPE parvw,    " <<< ADD (RD2-aligned)
         kunnr TYPE kunnr,
         adrnr TYPE adrnr,
       END OF lty_vbpa.
```

**Add local variables (if not present):**

```abap
DATA: lt_parvw_r TYPE RANGE OF parvw,
      lw_parvw_r LIKE LINE OF lt_parvw_r.
```

### 8.4 Change Block 2 — Extend `VBPA` Selection to Include `WE`

**Location:** Existing `"@ Fetch GTSN_REC"` / `VBPA` SELECT block (~lines 673–701 on RD4)

**Replace single-role SELECT with RD2-aligned range logic:**

```abap
"@ Fetch GSTN_REC and ship-to partners
CLEAR: lt_parvw_r[], lv_pravw.

SELECT value1
  FROM zyttspara
  INTO lv_pravw
  WHERE param1      = 'ZSCE_EWB'
    AND param2      = 'PARTNER'
    AND active_flag = abap_true.
ENDSELECT.

IF sy-subrc = 0.
  CALL FUNCTION 'CONVERSION_EXIT_PARVW_INPUT'
    EXPORTING  input  = lv_pravw
    IMPORTING  output = lv_pravw.

  lw_parvw_r-sign   = 'I'.
  lw_parvw_r-option = 'EQ'.
  lw_parvw_r-low    = lv_pravw.
  APPEND lw_parvw_r TO lt_parvw_r.

  lw_parvw_r-low = 'WE'.          " Ship-to party — required for sh_gstnid / sh_name1
  APPEND lw_parvw_r TO lt_parvw_r.
  CLEAR lw_parvw_r.

  SORT lt_parvw_r BY low.
  DELETE lt_parvw_r WHERE low IS INITIAL.
  DELETE ADJACENT DUPLICATES FROM lt_parvw_r COMPARING low.

  lt_vbrk_t = lt_vbrk.
  SORT lt_vbrk_t BY vbeln.
  DELETE ADJACENT DUPLICATES FROM lt_vbrk_t COMPARING vbeln.

  SELECT vbeln posnr parvw kunnr adrnr
    FROM vbpa CLIENT SPECIFIED
    INTO TABLE lt_vbpa
    FOR ALL ENTRIES IN lt_vbrk_t
    WHERE mandt = sy-mandt
      AND vbeln = lt_vbrk_t-vbeln
      AND parvw IN lt_parvw_r.
  IF sy-subrc EQ 0.
    lt_vbpa_temp = lt_vbpa.
    SORT lt_vbpa_temp BY kunnr.
    DELETE ADJACENT DUPLICATES FROM lt_vbpa_temp COMPARING kunnr.
    DELETE lt_vbpa_temp WHERE kunnr IS INITIAL.
  ENDIF.
ENDIF.
```

### 8.5 Change Block 3 — Tighten Bill-to `VBPA` Read (recommended)

**Location:** Existing block ~line 1041 (`READ TABLE lt_vbpa WITH KEY vbeln`)

**Change** from ambiguous `vbeln`-only read to role-specific read (align with configured bill-to / PARTNER role):

```abap
READ TABLE lt_vbpa INTO lw_vbpa
  WITH KEY vbeln = lw_vbrk-vbeln
           parvw = lv_pravw.
```

> If `lv_pravw` is initial, retain fallback: `READ TABLE lt_vbpa WITH KEY vbeln = lw_vbrk-vbeln BINARY SEARCH`.

**Existing line unchanged in intent:**

```abap
lw_header-gstn_rec = lw_kna1-stcd3.   " Bill-to / receiver GSTIN — DO NOT REMOVE
```

### 8.6 Change Block 4 — Add Ship-to Party Block (core RD2 port)

**Location:** After bill-to `rec_*` assignments, **before** `"@Item details"`

```abap
"@ Ship-to-party details — BEGIN RD4 ship-to enhancement (ref. RD2)
READ TABLE lt_vbpa INTO lw_vbpa
  WITH KEY vbeln = lw_vbrk-vbeln
           parvw = 'WE'.
IF sy-subrc EQ 0.
  CLEAR: lv_to_plc, lv_to_party, lv_to_name2, lv_to_name3, lv_to_name4.

  READ TABLE lt_kna1 INTO lw_kna1
    WITH KEY kunnr = lw_vbpa-kunnr BINARY SEARCH.
  IF sy-subrc EQ 0.
    lw_header-sh_gstnid = lw_kna1-stcd3.

    READ TABLE lt_cust_adrc INTO lw_cust_adrc
      WITH KEY addrnumber = lw_kna1-adrnr BINARY SEARCH.
    IF sy-subrc EQ 0.
      lv_to_plc    = lw_cust_adrc-city1.
      lv_to_party  = lw_cust_adrc-name1.
      lv_to_name2  = lw_cust_adrc-name2.
      lv_to_name3  = lw_cust_adrc-name3.
      lv_to_name4  = lw_cust_adrc-name4.

      CALL FUNCTION 'Z_REPLACE_SPL_CHAR'
        EXPORTING im_input = lv_to_plc    lm_replace = ' '
        IMPORTING ex_output = lv_to_plc.
      CALL FUNCTION 'Z_REPLACE_SPL_CHAR'
        EXPORTING im_input = lv_to_party  lm_replace = ''
        IMPORTING ex_output = lv_to_party.
      CALL FUNCTION 'Z_REPLACE_SPL_CHAR'
        EXPORTING im_input = lv_to_name2  lm_replace = ' '
        IMPORTING ex_output = lv_to_name2.
      CALL FUNCTION 'Z_REPLACE_SPL_CHAR'
        EXPORTING im_input = lv_to_name3  lm_replace = ' '
        IMPORTING ex_output = lv_to_name3.
      CALL FUNCTION 'Z_REPLACE_SPL_CHAR'
        EXPORTING im_input = lv_to_name4  lm_replace = ' '
        IMPORTING ex_output = lv_to_name4.

      lw_header-sh_city1       = lv_to_plc.
      lw_header-sh_name1       = lv_to_party.
      lw_header-sh_name2       = lv_to_name2.
      lw_header-sh_name3       = lv_to_name3.
      lw_header-sh_name4       = lv_to_name4.
      lw_header-sh_post_code1  = lw_cust_adrc-post_code1.
      lw_header-sh_region      = lw_cust_adrc-region.
    ENDIF.
  ENDIF.
ENDIF.
"@ Ship-to-party details — END RD4 ship-to enhancement
```

### 8.7 Field Mapping Summary (`INV_DTL_FETCH` → downstream)

| `ZLOG_EWB_S` (output of `INV_DTL_FETCH`) | Source | Next hop (`BOT_CREATE_OB`) | API field (param active) |
|---|---|---|---|
| `sh_gstnid` | `KNA1-STCD3` (`WE`) | `lw_ewb_hdr-sh_gstn` | `shiptogstin` |
| `sh_name1` | `ADRC-NAME1` (`WE`) | `lw_bot_header-to_other_party_name` | `shiptotradename` |
| `sh_name2` / `sh_name3` | `ADRC-NAME2/3` | `to_address1` / `to_address2` | `toaddr1` / `toaddr2` |
| `sh_post_code1` | `ADRC-POST_CODE1` | BOT validation | — |

### 8.8 Optional RD2 Extensions (assess per RD4 business need)

The following RD2 logic is **not** in the core port above. Include only if the scenario exists on RD4:

| RD2 feature | RD2 purpose | RD4 action |
|---|---|---|
| `lt_ship_ref` + `lt_shipto_adrc` | Division-specific ship-to address from `VBPA-ADRNR` | Port if RD4 uses same ship-ref config |
| `lw_flag1` / BOE address (`lt_bradd`) | Inbound petcoke BOE address | Port if RD4 has BOE scenario |
| `Z_SCE_EWB_TRANS_TYPE` call | Derive `trans_ind` from GSTN comparison | Separate transport if `trans_ind` blank on RD4 |
| `df_gstin` from dispatch plant | Dispatch GSTIN on header | Separate transport; not required for ship-to GSTIN |

### 8.9 Impact on Current RD4 Functioning

| Area | Expected impact | Mitigation |
|---|---|---|
| `gstn_rec` / `rec_*` (bill-to) | **No change** if existing block untouched | Do not modify `lw_header-gstn_rec = lw_kna1-stcd3` logic |
| API `togstin` / `totrdname` | **No change** — `ZCL_EWB_API` uses `bill_to_gstin` / `bill_to_other_party_name` | Verified on RD2 |
| `sh_gstnid` / `sh_name1` on header | **Populated** where `WE` partner exists | Enables ship-to params to send real values |
| BOT `sh_post_code1` validation | May allow more documents to pass validation | Positive side effect |
| When bill-to = ship-to | `gstn_rec` and `sh_gstnid` may be **same GSTIN** | Expected; no functional issue |

### 8.10 Sort / Binary Search Update

After extending `lt_vbpa` with `parvw`, update pre-loop sort (~line 860):

```abap
SORT lt_vbpa BY vbeln parvw.
```

---

## 9. Configuration Changes (Transport 4 — Customizing)

> Suggested RD4 transport description: **RD4 — EWB Ship-to GSTN — ZLOG_EXEC_VAR Config**
>
> Alternatively: manual SM30 entry in client 225 (non-transportable if table is client-dependent and maintained directly).

### 9.1 Table: `ZLOG_EXEC_VAR`

Maintain entries in **client 225**:

#### Parameter: Ship To GSTIN

| Field | Value |
|---|---|
| `NAME` | `ZSCE_EWB_API_SHIPTO_GSTIN` |
| `ACTIVE` | `X` |
| `REMARKS` | Transaction type: `1`, `2`, `3`, or `4` (one row per applicable type) |

#### Parameter: Ship To Trade Name

| Field | Value |
|---|---|
| `NAME` | `ZSCE_EWB_API_SHIPTO_TRDNAME` |
| `ACTIVE` | `X` |
| `REMARKS` | Transaction type: `1`, `2`, `3`, or `4` (one row per applicable type) |

### 9.2 Example Configuration

| MANDT | NAME | ACTIVE | REMARKS | Description |
|---|---|---|---|---|
| 225 | `ZSCE_EWB_API_SHIPTO_GSTIN` | X | 1 | Enable ship-to GSTIN for Trans Type 1 |
| 225 | `ZSCE_EWB_API_SHIPTO_GSTIN` | X | 2 | Enable ship-to GSTIN for Trans Type 2 |
| 225 | `ZSCE_EWB_API_SHIPTO_TRDNAME` | X | 2 | Enable ship-to trade name for Trans Type 2 |

> **Note:** Functional team to confirm which transaction types require activation on RD4. Mirror RD2 business rules.

### 9.3 Prerequisite Parameters (Existing — Verify Active)

| Parameter | Purpose |
|---|---|
| `ZSCE_EWB_BOT_ACTIVE` | Must be **inactive** for API path (ship-to applies to API, not BOT queue) |
| `ZSCM_EWB_IRN_ACTIVE` | Controls IRN vs non-IRN routing |
| `ZSCM_EWB_DISP_GSTIN` (`c_name1`) | Dispatch GSTIN override |
| `ZSCM_EWB_SHIP_GSTIN` (`c_name2`) | Ship GSTIN override |

---

## 10. Dependencies and Risks

### 10.1 Upstream Data — Resolved by Section 8

| System | `Z_SCE_EWB_INV_DTL_FETCH` behaviour |
|---|---|
| **RD2** | Derives `sh_gstnid` / `sh_name1` from ship-to (`WE`) partner |
| **RD4 (after Section 8)** | Same core logic ported — `sh_gstnid` from `KNA1-STCD3`, `sh_name1` from `ADRC` |

**Residual risk:** Billing documents **without** a `WE` partner on `VBPA` will still produce blank ship-to values. Validate per transaction type.

### 10.2 `trans_ind` Derivation

If `trans_ind` is blank on RD4 headers, ship-to parameter matching by transaction type (`REMARKS = '1'/'2'/'3'/'4'`) will fail. Assess whether `Z_SCE_EWB_TRANS_TYPE` (or equivalent RD4 logic) must be ported separately.

### 10.3 Authorization

Ensure transport/import user and runtime users have access to updated class and table fields.

---

## 11. Objects — Change Summary Matrix

| # | Object | Change Type | Transport |
|---|---|---|---|
| 1 | `ZSCM_SHIPTOGSTN_DE` | Create | DDIC (T1) |
| 2 | `ZSCM_SHIPTOTRADE_DE` | Create | DDIC (T1) |
| 3 | `ZSCE_EWB_HDR` | Append fields | DDIC (T1) |
| 4 | `ZST_EWAY_BILL_GENERATION_REQ1` | Create | DDIC (T1) |
| 5 | `ZCL_EWB_API` | Modify (`GENERATE_E_WAY_BILL`) | Workbench (T2) |
| 6 | `Z_SCE_EWB_INV_DTL_FETCH` | Modify (ship-to `WE` block) | Workbench (T3) |
| 7 | `ZLOG_EXEC_VAR` | Config entries (client 225) | Customizing (T4) |
| — | `Z_SCE_EWB_BOT_CREATE_OB` | **No change** | — |
| — | `Z_SCE_EWB_BOT_SERVICE_CALL` | **No change** | — |
| — | `Z_SCE_EWB_DATA_SAVE` | **No change** | — |
| — | `ZEWB_CREATE` | **No change** | — |
| — | `Z_SCM_IRN_EWB_GENERATION` | **No change** | — |

---

## 12. Transport Sequence (RD4)

| Step | Transport | Contents | Import Order |
|---|---|---|---|
| 1 | **T1** — DDIC | Data elements, table append, structure `REQ1` | First |
| 2 | **T2** — Workbench | `ZCL_EWB_API` changes | Second |
| 3 | **T3** — Workbench | `Z_SCE_EWB_INV_DTL_FETCH` ship-to fetch | Third |
| 4 | **T4** — Config | `ZLOG_EXEC_VAR` entries | Fourth (after code) |

**Import notes:**
1. Import DDIC transport and activate all dictionary objects.
2. Execute table adjustment for `ZSCE_EWB_HDR` if prompted.
3. Import `ZCL_EWB_API` workbench transport and activate.
4. Import `Z_SCE_EWB_INV_DTL_FETCH` workbench transport and activate.
5. Maintain `ZLOG_EXEC_VAR` in client 225.
6. Syntax check all modified objects.

---

## 13. Test Plan

### 13.1 Pre-requisites

- `ZSCE_EWB_BOT_ACTIVE` = **not active** (force API path)
- Test billing documents available for each transaction type to be activated
- Non-IRN scenario: IRN not generated OR `ZSCM_EWB_IRN_ACTIVE` not applicable

### 13.2 Test Scenarios

| TC# | Scenario | Setup | Expected Result |
|---|---|---|---|
| TC-01 | Param inactive | No `ZSCE_EWB_API_SHIPTO_*` rows for trans type | API JSON **excludes** `shiptogstin` / `shiptotradename`; `ZSCE_EWB_HDR` ship-to fields blank |
| TC-02 | GSTIN param active | `ZSCE_EWB_API_SHIPTO_GSTIN` active for trans type `2` | API JSON includes `shiptogstin`; `ZSCE_EWB_HDR-SHIP_TO_GSTIN` updated |
| TC-03 | Trade name param active | `ZSCE_EWB_API_SHIPTO_TRDNAME` active for trans type `2` | API JSON includes `shiptotradename` |
| TC-04 | Both params active | Both params active for same trans type | Both API fields present; both HDR fields updated |
| TC-05 | IRN path | Billing doc with valid IRN; `ZSCM_EWB_IRN_ACTIVE = X` | IRN EWB generated; ship-to fields **not** set via this enhancement |
| TC-06 | Trans type 1 override | Trans type `1`, GSTIN param active, state codes match | `shiptogstin` uses `ZSCM_EWB_DISP_GSTIN` remarks override |
| TC-07 | Trans type 2/3 override | Trans type `2` or `3`, GSTIN param active | `shiptogstin` uses `ZSCM_EWB_SHIP_GSTIN` remarks override |
| TC-08 | API failure | Force API error after HDR update | `SHIP_TO_GSTIN` / `SHIP_TO_TRADENAME` still persisted (pre-API update design) |
| TC-09 | Regression — BOT path | `ZSCE_EWB_BOT_ACTIVE = X` | BOT queue used; no regression in existing BOT flow |
| TC-10 | No `df_gstin`/`sh_gstn` in API JSON | Ship-to param active; capture API payload | JSON contains `shiptogstin` / `shiptotradename`; JSON does **not** contain `df_gstin` or `sh_gstn` keys |
| TC-11 | `INV_DTL_FETCH` ship-to population | Billing doc with `WE` partner; run fetch only | `ET_SCE_EWB_HEADER-sh_gstnid` = ship-to `KNA1-STCD3`; `sh_name1` = ship-to name |
| TC-12 | Bill-to ≠ ship-to | Doc with different `RE`/`AG` vs `WE` partners | `gstn_rec` ≠ `sh_gstnid`; both API `togstin` and `shiptogstin` correct when param active |
| TC-13 | Regression — `gstn_rec` unchanged | Standard domestic billing doc | `gstn_rec` and `rec_*` fields same as before T3 transport |

### 13.3 Verification Points

1. **SE11** — `ZSCE_EWB_HDR` contains `SHIP_TO_GSTIN`, `SHIP_TO_TRADENAME`
2. **SE24** — `ZCL_EWB_API` contains `lc_shipto_gstin` / `lc_shipto_trdname`
3. **SM30** — `ZLOG_EXEC_VAR` entries present for client 225
4. **SE16** — Post-test check of `ZSCE_EWB_HDR` for test `REQ_ID`
5. **API trace / log** — Confirm JSON payload keys `shiptogstin`, `shiptotradename`
6. **Debug `INV_DTL_FETCH`** — Breakpoint after ship-to block; verify `sh_gstnid` / `sh_name1` for test billing doc

---

## 14. Rollback Plan

| Step | Action |
|---|---|
| 1 | Deactivate `ZLOG_EXEC_VAR` entries (`ACTIVE` = blank) — immediate kill switch |
| 2 | Revert workbench transports (`ZCL_EWB_API`, `Z_SCE_EWB_INV_DTL_FETCH`) |
| 3 | Revert DDIC transport (only if no production data in new fields; table append rollback requires DBA review) |

> **Recommendation:** Use parameter deactivation (`ZLOG_EXEC_VAR`) as the primary rollback mechanism without code revert.

---

## 15. Implementation Checklist

- [ ] Create RD4 DDIC transport T1 (data elements, table, structure)
- [ ] Import and activate DDIC in RD4 development system
- [ ] Copy / implement `ZCL_EWB_API` changes from RD2 (non-IRN branch) — transport T2
- [ ] Implement `Z_SCE_EWB_INV_DTL_FETCH` ship-to block (Section 8) — transport T3
- [ ] Syntax check and peer review (all modified objects)
- [ ] Maintain `ZLOG_EXEC_VAR` in client 225 — transport T4
- [ ] Execute test plan TC-01 through TC-13
- [ ] Functional sign-off
- [ ] Transport to QA → Production per RD4 change management process

---

## 16. Appendix

### A. RD2 Reference Transports

| Transport | Description |
|---|---|
| **RD2K9A5LAN** | Dictionary changes + API structure `ZST_EWAY_BILL_GENERATION_REQ1` + `ZCL_EWB_API` ship-to code blocks |
| **RD2K9A5LGZ** | ABAP logic / mapping / parameter configuration (as per RD2 delivery) |

### B. Key RD4 Function Modules

| Function Module | Function Group | Role | Change |
|---|---|---|---|
| `Z_SCE_EWB_INV_DTL_FETCH` | `ZLOG_EBILL` | Fetch invoice/header data | **Modify** — ship-to `WE` block (Section 8) |
| `Z_SCE_EWB_BOT_CREATE_OB` | `ZLOG_EBILL` | Outbound BOT/API orchestration | No change |
| `Z_SCE_EWB_BOT_SERVICE_CALL` | `ZLOG_EBILL` | Service call wrapper | No change |
| `Z_SCE_EWB_DATA_SAVE` | `ZLOG_EBILL` | Persist header/items to DB | No change |

### C. API JSON Sample (when active)

```json
{
  "supplytype": "O",
  "transactiontype": "2",
  "shiptogstin": "27AAAAA0000A1Z5",
  "shiptotradename": "SHIP TO PARTY NAME PVT LTD",
  "...": "..."
}
```

### D. Document Control

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | 15-Jun-2026 | — | Initial delta document for RD4 based on RD2 implementation analysis |
| 1.1 | 15-Jun-2026 | — | Added Section 5.3 — `df_gstin` / `sh_gstn` removal from API JSON; TC-10; ABAP prerequisite 7.2 |
| 1.2 | 15-Jun-2026 | — | Added Section 8 — `Z_SCE_EWB_INV_DTL_FETCH` ship-to fetch (RD2-aligned); Section 5.4 value flow; T3 transport; TC-11–TC-13 |

---

*End of Document*
