# Delta Code Change Document — E-way Bill Ship-to GSTN Enhancement (D11 / P10)

| Field | Value |
|---|---|
| **Document Type** | Delta Code Change Document |
| **Development Server** | D11 |
| **Target System** | P10 (SAP ECC) |
| **Target Client** | 514 |
| **Module** | Logistics — E-way Bill (Custom) |
| **Reference System** | RD2 (implemented), RD4 (delta document) |
| **Reference Transports (RD2)** | RD2K9A5LAN (DDIC / API structure), RD2K9A5LGZ (ABAP logic / config) |
| **Change ID (RD2)** | CD:8089362 |
| **Author (RD2 reference)** | Omkar More |
| **Document Version** | 1.0 |
| **Date** | 16-Jun-2026 |
| **Scope** | Non-IRN E-way Bill API generation only |
| **Analysis Source** | D11 MCP object inspection (confirmed `ZSCE_EWB_SAPSERVER = P10`, `ZSCE_EWB_SYSID = P10`) |

---

## 1. Executive Summary

This document describes the **delta changes required on P10 (via D11 development)** to implement the Ship-to GSTN enhancement for E-way Bill creation, aligned with the solution already delivered on **RD2** and planned for **RD4**.

Two new fields are introduced:

| # | Field Label | API JSON Key | Table Field (`ZSCE_EWB_HDR`) | Length |
|---|---|---|---|---|
| 1 | Ship To GSTIN | `shiptogstin` | `SHIP_TO_GSTIN` | 18 |
| 2 | Ship To Trade Name | `shiptotradename` | `SHIP_TO_TRADENAME` | 100 |

Fields are populated in the **NIC API JSON payload** and persisted to **`ZSCE_EWB_HDR`** only when the corresponding **`ZLOG_EXEC_VAR`** parameter is **active** for the applicable **transaction type** (`1`, `2`, `3`, or `4`).

**Important:** This enhancement applies **only to the non-IRN path** inside `ZCL_EWB_API=>GENERATE_E_WAY_BILL`. IRN-based E-way Bill generation (`Z_SCM_IRN_EWB_GENERATION`) is **out of scope**.

**P10-specific note:** Unlike RD4, P10 requires an **additional change** to `Z_SCE_EWB_BOT_CREATE_OB` to populate `lw_api_data-ewb_create` before calling `generate_e_way_bill`. Without this block, upstream ship-to values cannot reach the API class (see Section 8 and Section 11).

**API field clarification (RD2):** `df_gstin` and `sh_gstn` must **not** be sent as NIC API JSON keys. Ship GSTIN is exposed via `shiptogstin` / `shiptotradename` when parameters are active. Internal structures (`ZEWB_CREATE`) and table `ZSCE_EWB_HDR` retain `DF_GSTIN` / `SH_GSTN` for internal use — see Section 5.3.

---

## 2. Business Requirement

When Bill-to and Ship-to parties differ, the E-way Bill API must send Ship-to GSTIN and Ship-to Trade Name as separate fields in the generation request, and store them in the E-way Bill header table for audit and reporting.

Activation must be **configurable per transaction type** via `ZLOG_EXEC_VAR`, so business can control which transaction scenarios include these fields without further code changes.

---

## 3. P10 Current State vs Target State

### 3.1 Current State on P10 (as-is — verified via D11 MCP)

| Object | P10 Status |
|---|---|
| `ZCL_EWB_API` | Exists (package `ZLOG`) — **no** ship-to logic (`shiptogstin` / `shiptotradename`) |
| `ZSCE_EWB_HDR` | Exists — has `DF_GSTIN`, `SH_GSTN`; **`SHIP_TO_GSTIN` / `SHIP_TO_TRADENAME` not present** |
| `ZSCM_SHIPTOGSTN_DE` | **Not found** |
| `ZSCM_SHIPTOTRADE_DE` | **Not found** |
| `ZST_EWAY_BILL_GENERATION_REQ` | Exists — includes `df_gstin`, `sh_gstn` components (not used in API JSON on P10 today) |
| `ZST_EWAY_BILL_GENERATION_REQ1` | **Not found** |
| `Z_SCE_EWB_BOT_CREATE_OB` | Exists — maps `to_*` from `sh_*` on `lw_bot_header`; **`lw_api_data-ewb_create` not populated** (gap vs RD4) |
| `Z_SCE_EWB_BOT_CREATE_OB_JW` | Exists — assess for parallel change if used in production |
| `Z_SCE_EWB_BOT_CREATE_OB_FIORI` | Exists — assess for parallel change if Fiori path is active |
| `Z_SCE_EWB_BOT_SERVICE_CALL` | Exists — uses **legacy BOT queue only**; does **not** call `ZCL_EWB_API` |
| `Z_SCE_EWB_DATA_SAVE` | Exists — simpler version; does **not** map ship-to API fields |
| `Z_SCE_EWB_INV_DTL_FETCH` | Exists — **`lty_vbpa` has no `parvw`**; VBPA select uses bill-to role only; **no ship-to (`WE`) block** |
| `ZEWB_CREATE` / `ZSCE_EWB_API_STR` | Exist — have `sh_gstn`, `df_gstin`, `trans_ind`, `to_other_party_name` |
| `ZLOG_EWB_S` | Exists — has `sh_gstnid`, `sh_name1`, etc.; **not populated** by current `INV_DTL_FETCH` |
| `ZLOG_EXEC_VAR` | **`ZSCE_EWB_API_SHIPTO_GSTIN` / `ZSCE_EWB_API_SHIPTO_TRDNAME` not present** |
| `ZLOG_EXEC_VAR` | `ZSCE_EWB_BOT_ACTIVE` = **blank** (API path active) |
| `ZLOG_EXEC_VAR` | `ZSCM_EWB_IRN_ACTIVE` = **X** (IRN path active — many docs may bypass this enhancement) |
| `ZLOG_EXEC_VAR` | `ZSCM_EWB_DISP_GSTIN` / `ZSCM_EWB_SHIP_GSTIN` — **not found** (GSTIN override params absent) |

### 3.2 Target State on P10 (to-be)

| Layer | Target |
|---|---|
| **DDIC** | New data elements, extended `ZSCE_EWB_HDR`, new API structure `ZST_EWAY_BILL_GENERATION_REQ1` |
| **ABAP** | `ZCL_EWB_API` enhanced in non-IRN branch with parameter-driven ship-to mapping |
| **ABAP (upstream)** | `Z_SCE_EWB_INV_DTL_FETCH` — ship-to (`WE`) party fetch for `sh_gstnid`, `sh_name1`, address fields |
| **ABAP (upstream — P10 only)** | `Z_SCE_EWB_BOT_CREATE_OB` — populate `lw_api_data-ewb_create` with `sh_gstn`, `trans_ind`, etc. |
| **Config** | `ZLOG_EXEC_VAR` rows maintained per transaction type in client 514 |
| **API** | JSON payload includes `shiptogstin` and `shiptotradename` when parameters are active |
| **Persistence** | `ZSCE_EWB_HDR-SHIP_TO_GSTIN` and `SHIP_TO_TRADENAME` updated before API call |

---

## 4. Scope

### 4.1 In Scope

- Dictionary objects (data elements, table append, API structure)
- Class `ZCL_EWB_API` — method `GENERATE_E_WAY_BILL` (non-IRN `ELSE` branch only)
- Function `Z_SCE_EWB_INV_DTL_FETCH` — ship-to party data fetch (RD2-aligned core block)
- Function `Z_SCE_EWB_BOT_CREATE_OB` — **`ewb_create` population block** (P10-specific; RD4 already has this)
- Customizing table `ZLOG_EXEC_VAR` — parameter maintenance for P10 client 514
- Unit / integration testing for non-IRN billing documents

### 4.2 Out of Scope

| Item | Reason |
|---|---|
| IRN E-way Bill path (`Z_SCM_IRN_EWB_GENERATION`) | RD2 enhancement explicitly excluded from IRN flow |
| `Z_SCE_EWB_BOT_SERVICE_CALL` | BOT queue only; no NIC JSON API |
| `Z_SCE_EWB_DATA_SAVE` | Ship-to persistence handled in `ZCL_EWB_API` |
| BOT queue path (`ZSCE_EWB_BOT_ACTIVE = X`) | Uses legacy BOT interface, not NIC JSON API |
| Full RD2 `INV_DTL_FETCH` parity (STO/BOE/Nepal/export edge cases) | Port **core ship-to block only**; extended RD2 scenarios in Section 9.6 |
| `Z_SCE_EWB_BOT_CREATE_OB_JW` / `_FIORI` | Out of scope unless confirmed in use — then mirror T3b changes |

---

## 5. Solution Architecture

### 5.1 E-way Bill Creation Flow (P10)

```
Z_SCE_EWB_INV_DTL_FETCH
        │
        ▼
Z_SCE_EWB_BOT_CREATE_OB
        │
        ├──► Z_SCE_EWB_DATA_SAVE  ──►  ZSCE_EWB_HDR (initial insert)
        │
        └──► ZLOG_EXEC_VAR check: ZSCE_EWB_BOT_ACTIVE
                    │
                    ├── Active (X)  ──► BOT Queue (legacy — no ship-to JSON change)
                    │
                    └── Inactive      ──► ZCL_EWB_API=>GENERATE_E_WAY_BILL   [ENTRY POINT]
                                              │
                                              ├── IRN exists + ZSCM_EWB_IRN_ACTIVE = X
                                              │         └──► Z_SCM_IRN_EWB_GENERATION  [OUT OF SCOPE]
                                              │
                                              └── ELSE (Non-IRN path)  [THIS ENHANCEMENT]
                                                        ├── Read ZLOG_EXEC_VAR ship-to params
                                                        ├── Map shiptogstin / shiptotradename
                                                        ├── UPDATE ZSCE_EWB_HDR
                                                        ├── Serialize JSON via ZST_EWAY_BILL_GENERATION_REQ1
                                                        ├── Encrypt (call_encryption)
                                                        └── HTTP POST via submit_request (GENEWAYBILL)
```

> **Note:** `CALL METHOD zcl_ewb_api=>generate_e_way_bill` in `Z_SCE_EWB_BOT_CREATE_OB` is the SAP entry point for the NIC API. The actual HTTP POST to the E-way Bill portal occurs inside the method via `submit_request` with `action = 'GENEWAYBILL'` and URL from `ZEWB_URL_CONFIG`.

### 5.2 Data Mapping (Non-IRN API Input)

| API Output Field | Source (`ZEWB_CREATE` / `im_data-ewb_create`) | Set By |
|---|---|---|
| `shiptogstin` | `sh_gstn` | `Z_SCE_EWB_BOT_CREATE_OB` ← `lw_bot_final_header-sh_gstnid` |
| `shiptotradename` | `to_other_party_name` | `Z_SCE_EWB_BOT_CREATE_OB` ← `lw_bot_header-to_other_party_name` ← `sh_name1` |
| `transactiontype` | `trans_ind` | `Z_SCE_EWB_BOT_CREATE_OB` ← `lw_bot_final_header-trans_ind` |

### 5.3 API JSON Field Changes — `df_gstin` & `sh_gstn`

On **RD2**, `df_gstin` and `sh_gstn` are **not sent** in the NIC API JSON call. P10 must align with this design.

#### 5.3.1 What is NOT in the API JSON

The new serialization structure `ZST_EWAY_BILL_GENERATION_REQ1` adds `shiptogstin` and `shiptotradename` only. The NIC API uses standard keys such as `fromgstin`, `togstin`, and — when ship-to params are active — `shiptogstin`, `shiptotradename`.

| Internal field (`ZEWB_CREATE`) | Former / intended API key | P10 API JSON status | Replacement in API |
|---|---|---|---|
| `df_gstin` | `df_gstin` | **Not sent** — do not map to `lw_header` | Dispatch GSTIN via `fromgstin` / `togstin` logic |
| `sh_gstn` | `sh_gstn` | **Not sent** — do not map to `lw_header` | Ship GSTIN sent as **`shiptogstin`** when `ZSCE_EWB_API_SHIPTO_GSTIN` is active |

#### 5.3.2 P10 current baseline

P10 `ZCL_EWB_API` currently serializes `lw_header` (`ZST_EWAY_BILL_GENERATION_REQ`) and does **not** assign `lw_header-df_gstin` or `lw_header-sh_gstn` in code. The DDIC structure contains these components but they are not populated at runtime — acceptable baseline. Ensure they remain unmapped after enhancement.

#### 5.3.3 What still uses `df_gstin` / `sh_gstn` (internal — not API)

| Layer | Usage | Change required on P10? |
|---|---|---|
| `ZEWB_CREATE` | Populated by `Z_SCE_EWB_BOT_CREATE_OB` | **Yes** — add missing `ewb_create` block (Section 8) |
| `ZSCE_EWB_HDR` | `DF_GSTIN`, `SH_GSTN` columns exist | **No API change** — separate from JSON payload |
| `ZCL_EWB_API` JSON build | Must not serialize `df_gstin`/`sh_gstn` | **Yes** — use `shiptogstin` path via `REQ1` |
| NIC API JSON | Uses `fromgstin`, `togstin`, `shiptogstin`, `shiptotradename` | Align with RD2 |

### 5.4 End-to-End Value Flow (when params are active)

```
VBPA (PARVW = 'WE') + KNA1 + ADRC
        │  Z_SCE_EWB_INV_DTL_FETCH  [NEW on P10]
        ▼
ZLOG_EWB_S header: sh_gstnid, sh_name1, sh_name2/3, sh_post_code1, sh_region
        │  Z_SCE_EWB_BOT_CREATE_OB  [NEW: ewb_create mapping on P10]
        ▼
ZEWB_CREATE: sh_gstn, to_other_party_name, trans_ind
        │  ZCL_EWB_API (param active)  [NEW on P10]
        ▼
API JSON: shiptogstin, shiptotradename  +  ZSCE_EWB_HDR: SHIP_TO_GSTIN, SHIP_TO_TRADENAME
```

| Stage | Field | Source |
|---|---|---|
| 1 — `INV_DTL_FETCH` | `sh_gstnid` | Ship-to customer `KNA1-STCD3` (`VBPA-PARVW = 'WE'`) |
| 1 — `INV_DTL_FETCH` | `sh_name1` | Ship-to address `ADRC-NAME1` |
| 2 — `BOT_CREATE_OB` | `sh_gstn` | `lw_bot_final_header-sh_gstnid` |
| 2 — `BOT_CREATE_OB` | `to_other_party_name` | `lw_bot_header-to_other_party_name` ← `sh_name1` |
| 3 — `ZCL_EWB_API` | `shiptogstin` | `im_data-ewb_create-sh_gstn` (when `ZSCE_EWB_API_SHIPTO_GSTIN` active) |
| 3 — `ZCL_EWB_API` | `shiptotradename` | `im_data-ewb_create-to_other_party_name` (when `ZSCE_EWB_API_SHIPTO_TRDNAME` active) |

> **Note:** Parameter activation controls **inclusion in API JSON** (Stage 3). Stages 1–2 must populate values **before** API call — otherwise fields appear with blank values.

---

## 6. Dictionary Changes (Transport 1 — DDIC)

> Suggested P10 transport description: **P10 — EWB Ship-to GSTN — DDIC**

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

**Activation:** Ensure table adjustment / DB sync is executed in P10 after transport import.

### 6.4 Structure: `ZST_EWAY_BILL_GENERATION_REQ1`

Create new structure based on `ZST_EWAY_BILL_GENERATION_REQ` with two additional components:

| Component | Type | Purpose |
|---|---|---|
| `shiptogstin` | `ZEWB_STRING` | API JSON field — Ship To GSTIN |
| `shiptotradename` | `ZEWB_STRING` | API JSON field — Ship To Trade Name |

> **Note:** P10 currently has `ZST_EWAY_BILL_GENERATION_REQ` only (includes legacy `df_gstin` / `sh_gstn` components). `ZST_EWAY_BILL_GENERATION_REQ1` must be **newly created**. When ship-to params are active, serialize `REQ1` instead of `REQ`.

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

> Suggested P10 transport description: **P10 — EWB Ship-to GSTN — ABAP Logic**

### 7.1 Object Changed

| Object | Type | Method / Area | Package |
|---|---|---|---|
| `ZCL_EWB_API` | Class | `GENERATE_E_WAY_BILL` — non-IRN `ELSE` branch | `ZLOG` |

### 7.2 Prerequisite — Confirm `df_gstin` / `sh_gstn` Not in API JSON

Before applying ship-to blocks, confirm P10 `ZCL_EWB_API` baseline:

- `lw_header-df_gstin` and `lw_header-sh_gstn` assignments are **commented out** or **absent** (P10: currently absent — OK).
- API JSON is built from `ZST_EWAY_BILL_GENERATION_REQ` / `REQ1` only.
- Do **not** add active `lw_header-df_gstin` / `lw_header-sh_gstn` assignments.

---

### 7.3 Change Block 1 — Local Declarations

**Location:** Method `GENERATE_E_WAY_BILL`, local data section (near existing constants)

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

DATA:
  lw_header_1 TYPE zst_eway_bill_generation_req1.
```

---

### 7.4 Change Block 2 — Extend `ZLOG_EXEC_VAR` Selection

**Location:** Existing `SELECT` on `lt_log_exec_var` (non-IRN branch, near `c_name` / `c_name1` / `c_name2` read if present)

**Modify WHERE clause** to include ship-to parameter names:

```abap
SELECT name active remarks
  FROM zlog_exec_var
  INTO TABLE lt_log_exec_var
  WHERE ( name = c_name
       OR name = c_name1
       OR name = c_name2
       OR name = lc_shipto_gstin
       OR name = lc_shipto_trdname )
    AND active = abap_true.
```

> **P10 note:** If `c_name1` / `c_name2` (`ZSCM_EWB_DISP_GSTIN` / `ZSCM_EWB_SHIP_GSTIN`) are not used on P10, the corresponding constants/select entries may not exist. Add ship-to params to whichever `ZLOG_EXEC_VAR` select block is present in the non-IRN branch.

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

**Location:** Existing block using `c_name1` (`ZSCM_EWB_DISP_GSTIN`) and `c_name2` (`ZSCM_EWB_SHIP_GSTIN`) remarks — **if present on P10**

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

> **P10 note:** `ZSCM_EWB_DISP_GSTIN` and `ZSCM_EWB_SHIP_GSTIN` are **not currently maintained** in P10 `ZLOG_EXEC_VAR`. Include this block only if functional team confirms override requirement and config is added. Otherwise skip Section 7.6.

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

> **P10 note:** `ZSCM_EWB_IRN_ACTIVE = X` on client 514. Validate enhancement with billing documents that follow the **non-IRN** path.

---

## 8. ABAP Changes — `Z_SCE_EWB_INV_DTL_FETCH` (Transport 3 — Workbench)

> Suggested P10 transport description: **P10 — EWB Ship-to GSTN — INV_DTL_FETCH**
>
> **Reference:** RD2 `Z_SCE_EWB_INV_DTL_FETCH` — ship-to party block (CD:8089362 context)
>
> **Insertion point on P10:** Immediately **after** the existing bill-to / `gstn_rec` block and **before** `"@Item details"` (after `lw_header-rec_region` assignment).

### 8.1 Purpose

Populate ship-to party fields on `ZLOG_EWB_S` work header so that downstream objects can supply values when `ZLOG_EXEC_VAR` ship-to parameters are active:

| `ZLOG_EWB_S` field | Populated from | Required for |
|---|---|---|
| `sh_gstnid` | `KNA1-STCD3` (ship-to / `WE` partner) | `shiptogstin` via `sh_gstn` |
| `sh_name1` | `ADRC-NAME1` (ship-to address) | `shiptotradename` via `to_other_party_name` |
| `sh_name2`, `sh_name3` | `ADRC-NAME2/3` | `to_address1/2` in API (existing BOT mapping) |
| `sh_city1`, `sh_post_code1`, `sh_region` | `ADRC` ship-to | BOT validation (`sh_post_code1` check) |

**Design principle:** **Add** ship-to logic; **do not modify** the existing `gstn_rec` / `rec_*` (bill-to) block.

### 8.2 P10 Gap vs RD2 (current)

| Item | P10 (as-is) | RD2 (reference) |
|---|---|---|
| `lty_vbpa` structure | `vbeln`, `posnr`, `kunnr`, `adrnr` only | Includes **`parvw`** |
| `VBPA` SELECT | Single role from `ZYTTPARA` / `PARTNER` | **`PARTNER` + `WE`** in range |
| `gstn_rec` assignment | `lw_header-gstn_rec = lw_kna1-stcd3` | Same (bill-to path) |
| Ship-to block | **Missing** | `READ lt_vbpa` with `parvw = 'WE'` → `sh_gstnid`, `sh_name1`, etc. |
| `trans_ind` derivation | **Not set** in core loop | RD2/RD4 derive via config or `Z_SCE_EWB_TRANS_TYPE` |

### 8.3 Change Block 1 — Extend `lty_vbpa` Local Type

**Modify:**

```abap
TYPES: BEGIN OF lty_vbpa,
         vbeln TYPE vbeln,
         posnr TYPE posnr,
         parvw TYPE parvw,    " ADD (RD2-aligned)
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

### 8.5 Change Block 3 — Tighten Bill-to `VBPA` Read

**Change** from ambiguous `vbeln`-only read to role-specific read:

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
"@ Ship-to-party details — BEGIN P10 ship-to enhancement (ref. RD2)
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
"@ Ship-to-party details — END P10 ship-to enhancement
```

### 8.7 Sort / Binary Search Update

After extending `lt_vbpa` with `parvw`, update pre-loop sort:

```abap
SORT lt_vbpa BY vbeln parvw.
```

### 8.8 Optional RD2 Extensions (assess per P10 business need)

| RD2 feature | RD2 purpose | P10 action |
|---|---|---|
| `lt_ship_ref` + `lt_shipto_adrc` | Division-specific ship-to address from `VBPA-ADRNR` | Port if P10 uses same ship-ref config |
| `lw_flag1` / BOE address (`lt_bradd`) | Inbound petcoke BOE address | Port if P10 has BOE scenario |
| `Z_SCE_EWB_TRANS_TYPE` call | Derive `trans_ind` from GSTN comparison | **Assess — likely required on P10** (see Section 10.2) |
| Export / Nepal edge cases | Special GSTN handling | Port only if scenario exists on P10 |

---

## 9. ABAP Changes — `Z_SCE_EWB_BOT_CREATE_OB` (Transport 3b — Workbench)

> Suggested P10 transport description: **P10 — EWB Ship-to GSTN — BOT_CREATE_OB**
>
> **P10-specific:** RD4 document lists this FM as "no change" because RD4 already has the mapping block. **P10 is missing this block entirely** and must add it.

### 9.1 Purpose

Populate `lw_api_data-ewb_create` (`ZEWB_CREATE`) before calling `ZCL_EWB_API=>GENERATE_E_WAY_BILL`. The API class reads `im_data-ewb_create`, not `lw_bot_header` directly.

### 9.2 P10 Gap (verified via D11 MCP)

| Item | P10 (as-is) | Required |
|---|---|---|
| `lw_api_data` declaration | Present (`TYPE zsce_ewb_api_str`) | OK |
| `lw_api_data-ewb_create` population | **Missing** | **Add** |
| `lw_ewb_hdr` / `lw_ewb_item` locals | **Missing** | **Add** |
| `CALL METHOD zcl_ewb_api=>generate_e_way_bill` | Passes empty `lw_api_data` | Fix via mapping block |
| `lw_bot_header-to_gstin` | Mapped from `sh_gstnid` | OK (but `sh_gstnid` blank until T3) |
| `lw_bot_header-to_other_party_name` | Mapped from `sh_name1` | OK (but `sh_name1` blank until T3) |

### 9.3 Change Block 1 — Add Local Declarations

**Location:** Local data section (near existing `lw_api_data` declaration)

```abap
DATA: lw_ewb_hdr  TYPE zewb_create,
      lw_ewb_item TYPE zewb_items,
      lt_ewb_items TYPE zewb_items_tt.
```

### 9.4 Change Block 2 — Populate `ewb_create` Before API Call

**Location:** After `lw_bot_header` / `lw_input_ser-way_bill_create_request` assignment, **before** `ZSCE_EWB_BOT_ACTIVE` check / API call

```abap
"@ Populate ewb_create for API path — BEGIN P10 ship-to enhancement
REFRESH lt_ewb_items.
LOOP AT lt_bot_main_item INTO lw_bot_main_item.
  CLEAR lw_ewb_items.
  MOVE-CORRESPONDING lw_bot_main_item TO lw_ewb_items.
  APPEND lw_ewb_items TO lt_ewb_items.
ENDLOOP.

lw_ewb_item-ewb_item = lt_ewb_items.
lw_ewb_hdr-ewb_items = lw_ewb_item.

MOVE-CORRESPONDING lw_bot_header TO lw_ewb_hdr.
lw_ewb_hdr-trans_ind     = lw_bot_final_header-trans_ind.
lw_ewb_hdr-df_gstin      = lw_bot_final_header-df_gstin.
lw_ewb_hdr-sh_gstn       = lw_bot_final_header-sh_gstnid.
lw_ewb_hdr-sub_supl_desc = lw_bot_final_header-sub_supl_desc.

lw_api_data-ewb_create = lw_ewb_hdr.
CLEAR: lw_ewb_hdr, lt_ewb_items.
"@ Populate ewb_create for API path — END P10 ship-to enhancement
```

### 9.5 Field Mapping Summary (`BOT_CREATE_OB` → `ewb_create` → API)

| `ZEWB_CREATE` field | Source on P10 | API field (param active) |
|---|---|---|
| `sh_gstn` | `lw_bot_final_header-sh_gstnid` | `shiptogstin` |
| `to_other_party_name` | `lw_bot_header-to_other_party_name` ← `sh_name1` | `shiptotradename` |
| `trans_ind` | `lw_bot_final_header-trans_ind` | `transactiontype` (param match key) |
| `df_gstin` | `lw_bot_final_header-df_gstin` | Internal only — not in API JSON |
| `sub_supl_desc` | `lw_bot_final_header-sub_supl_desc` | Existing API field when mapped |

### 9.6 Related Function Modules (assess if in use)

| Function Module | Action |
|---|---|
| `Z_SCE_EWB_BOT_CREATE_OB_JW` | Mirror Section 9.4 if JW path is active on P10 |
| `Z_SCE_EWB_BOT_CREATE_OB_FIORI` | Mirror Section 9.4 if Fiori EWB generation is active |
| `Z_PNC_EWB_OGP_CREATE` | Uses `ZCL_EWB_API` — assess separately for OGP scenarios |

---

## 10. Configuration Changes (Transport 4 — Customizing)

> Suggested P10 transport description: **P10 — EWB Ship-to GSTN — ZLOG_EXEC_VAR Config**

### 10.1 Table: `ZLOG_EXEC_VAR`

Maintain entries in **client 514**:

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

### 10.2 Example Configuration

| MANDT | NAME | ACTIVE | REMARKS | Description |
|---|---|---|---|---|
| 514 | `ZSCE_EWB_API_SHIPTO_GSTIN` | X | 1 | Enable ship-to GSTIN for Trans Type 1 |
| 514 | `ZSCE_EWB_API_SHIPTO_GSTIN` | X | 2 | Enable ship-to GSTIN for Trans Type 2 |
| 514 | `ZSCE_EWB_API_SHIPTO_TRDNAME` | X | 2 | Enable ship-to trade name for Trans Type 2 |

> **Note:** Functional team to confirm which transaction types require activation on P10. Mirror RD2 business rules.

### 10.3 Prerequisite Parameters (Existing — Verify Active)

| Parameter | P10 Status | Purpose |
|---|---|---|
| `ZSCE_EWB_BOT_ACTIVE` | Blank (inactive) | API path used — required for ship-to JSON enhancement |
| `ZSCM_EWB_IRN_ACTIVE` | **X** (active) | Controls IRN vs non-IRN routing — test non-IRN docs explicitly |
| `ZSCM_EWB_DISP_GSTIN` | **Not found** | Dispatch GSTIN override — add only if Section 7.6 required |
| `ZSCM_EWB_SHIP_GSTIN` | **Not found** | Ship GSTIN override — add only if Section 7.6 required |

---

## 11. Dependencies and Risks

### 11.1 Upstream Data

| System | `Z_SCE_EWB_INV_DTL_FETCH` behaviour |
|---|---|
| **RD2** | Derives `sh_gstnid` / `sh_name1` from ship-to (`WE`) partner |
| **P10 (after T3)** | Same core logic ported |
| **P10 (after T3b)** | `ewb_create` populated for API class consumption |

**Residual risk:** Billing documents **without** a `WE` partner on `VBPA` will produce blank ship-to values.

### 11.2 `trans_ind` Derivation

P10 `INV_DTL_FETCH` does **not** currently derive `trans_ind`. If blank, ship-to parameter matching by transaction type (`REMARKS = '1'/'2'/'3'/'4'`) will fail.

**Mitigation:** Port `Z_SCE_EWB_TRANS_TYPE` call (function exists in `ZLOG_EBILL_V3` on P10) or RD4-style derivation in `INV_DTL_FETCH` / `BOT_CREATE_OB` validation block.

### 11.3 IRN Path Dominance

`ZSCM_EWB_IRN_ACTIVE = X` on P10. Documents with valid IRN will use `Z_SCM_IRN_EWB_GENERATION` and **will not** receive ship-to JSON fields from this enhancement.

### 11.4 Authorization

Ensure transport/import user and runtime users have access to updated class, function modules, and new table fields.

---

## 12. Objects — Change Summary Matrix

| # | Object | Change Type | Transport |
|---|---|---|---|
| 1 | `ZSCM_SHIPTOGSTN_DE` | Create | T1 — DDIC |
| 2 | `ZSCM_SHIPTOTRADE_DE` | Create | T1 — DDIC |
| 3 | `ZSCE_EWB_HDR` | Append fields | T1 — DDIC |
| 4 | `ZST_EWAY_BILL_GENERATION_REQ1` | Create | T1 — DDIC |
| 5 | `ZCL_EWB_API` | Modify (`GENERATE_E_WAY_BILL`) | T2 — Workbench |
| 6 | `Z_SCE_EWB_INV_DTL_FETCH` | Modify (ship-to `WE` block) | T3 — Workbench |
| 7 | `Z_SCE_EWB_BOT_CREATE_OB` | Modify (`ewb_create` mapping) | **T3b — Workbench (P10 only)** |
| 8 | `ZLOG_EXEC_VAR` | Config entries (client 514) | T4 — Customizing |
| — | `Z_SCE_EWB_BOT_SERVICE_CALL` | **No change** | — |
| — | `Z_SCE_EWB_DATA_SAVE` | **No change** | — |
| — | `ZEWB_CREATE` | **No change** | — |
| — | `Z_SCM_IRN_EWB_GENERATION` | **No change** | — |
| — | `Z_SCE_EWB_BOT_CREATE_OB_JW` / `_FIORI` | Assess / mirror T3b if in use | Optional |

---

## 13. Transport Sequence (P10 via D11)

| Step | Transport | Contents | Import Order |
|---|---|---|---|
| 1 | **T1** — DDIC | Data elements, table append, structure `REQ1` | First |
| 2 | **T2** — Workbench | `ZCL_EWB_API` changes | Second |
| 3 | **T3** — Workbench | `Z_SCE_EWB_INV_DTL_FETCH` ship-to fetch | Third |
| 4 | **T3b** — Workbench | `Z_SCE_EWB_BOT_CREATE_OB` `ewb_create` mapping | Fourth |
| 5 | **T4** — Config | `ZLOG_EXEC_VAR` entries (client 514) | Fifth (after code) |

**Import notes:**
1. Import DDIC transport and activate all dictionary objects.
2. Execute table adjustment for `ZSCE_EWB_HDR` if prompted.
3. Import `ZCL_EWB_API` workbench transport and activate.
4. Import `Z_SCE_EWB_INV_DTL_FETCH` workbench transport and activate.
5. Import `Z_SCE_EWB_BOT_CREATE_OB` workbench transport and activate.
6. Maintain `ZLOG_EXEC_VAR` in client 514.
7. Syntax check all modified objects.

---

## 14. Test Plan

### 14.1 Pre-requisites

- `ZSCE_EWB_BOT_ACTIVE` = **not active** (P10: currently blank — API path)
- Test billing documents available for each transaction type to be activated
- Non-IRN scenario: IRN not generated OR document does not take IRN path despite `ZSCM_EWB_IRN_ACTIVE = X`
- `trans_ind` populated on test documents (verify after T3/T3b)

### 14.2 Test Scenarios

| TC# | Scenario | Setup | Expected Result |
|---|---|---|---|
| TC-01 | Param inactive | No `ZSCE_EWB_API_SHIPTO_*` rows for trans type | API JSON **excludes** `shiptogstin` / `shiptotradename`; HDR ship-to fields blank |
| TC-02 | GSTIN param active | `ZSCE_EWB_API_SHIPTO_GSTIN` active for trans type `2` | API JSON includes `shiptogstin`; `ZSCE_EWB_HDR-SHIP_TO_GSTIN` updated |
| TC-03 | Trade name param active | `ZSCE_EWB_API_SHIPTO_TRDNAME` active for trans type `2` | API JSON includes `shiptotradename` |
| TC-04 | Both params active | Both params active for same trans type | Both API fields present; both HDR fields updated |
| TC-05 | IRN path | Billing doc with valid IRN; IRN EWB generated | Ship-to fields **not** set via this enhancement |
| TC-06 | Trans type 1 override | Trans type `1`, GSTIN param active, override config present | `shiptogstin` uses override remarks (if Section 7.6 implemented) |
| TC-07 | Trans type 2/3 override | Trans type `2` or `3`, GSTIN param active | `shiptogstin` uses ship GSTIN override (if Section 7.6 implemented) |
| TC-08 | API failure | Force API error after HDR update | `SHIP_TO_GSTIN` / `SHIP_TO_TRADENAME` still persisted (pre-API update design) |
| TC-09 | Regression — BOT path | `ZSCE_EWB_BOT_ACTIVE = X` | BOT queue used; no regression in existing BOT flow |
| TC-10 | No `df_gstin`/`sh_gstn` in API JSON | Ship-to param active; capture API payload | JSON contains `shiptogstin` / `shiptotradename`; JSON does **not** contain `df_gstin` or `sh_gstn` keys |
| TC-11 | `INV_DTL_FETCH` ship-to population | Billing doc with `WE` partner; run fetch only | `ET_SCE_EWB_HEADER-sh_gstnid` = ship-to `KNA1-STCD3`; `sh_name1` = ship-to name |
| TC-12 | Bill-to ≠ ship-to | Doc with different bill-to vs `WE` partners | `gstn_rec` ≠ `sh_gstnid`; both API `togstin` and `shiptogstin` correct when param active |
| TC-13 | Regression — `gstn_rec` unchanged | Standard domestic billing doc | `gstn_rec` and `rec_*` fields same as before T3 transport |
| TC-14 | `ewb_create` population | Debug `BOT_CREATE_OB` before API call | `lw_api_data-ewb_create-sh_gstn` and `to_other_party_name` populated (T3b) |

### 14.3 Verification Points

1. **SE11** — `ZSCE_EWB_HDR` contains `SHIP_TO_GSTIN`, `SHIP_TO_TRADENAME`
2. **SE24** — `ZCL_EWB_API` contains `lc_shipto_gstin` / `lc_shipto_trdname`
3. **SM30** — `ZLOG_EXEC_VAR` entries present for client 514
4. **SE16** — Post-test check of `ZSCE_EWB_HDR` for test `REQ_ID`
5. **API trace / log** — Confirm JSON payload keys `shiptogstin`, `shiptotradename`
6. **Debug `INV_DTL_FETCH`** — Breakpoint after ship-to block; verify `sh_gstnid` / `sh_name1`
7. **Debug `BOT_CREATE_OB`** — Breakpoint before `generate_e_way_bill`; verify `lw_api_data-ewb_create`

---

## 15. Rollback Plan

| Step | Action |
|---|---|
| 1 | Deactivate `ZLOG_EXEC_VAR` entries (`ACTIVE` = blank) — immediate kill switch |
| 2 | Revert workbench transports (`ZCL_EWB_API`, `Z_SCE_EWB_INV_DTL_FETCH`, `Z_SCE_EWB_BOT_CREATE_OB`) |
| 3 | Revert DDIC transport (only if no production data in new fields; table append rollback requires DBA review) |

> **Recommendation:** Use parameter deactivation (`ZLOG_EXEC_VAR`) as the primary rollback mechanism without code revert.

---

## 16. Implementation Checklist

- [ ] Create P10 DDIC transport T1 (data elements, table, structure)
- [ ] Import and activate DDIC in D11 development system
- [ ] Implement `ZCL_EWB_API` changes (non-IRN branch) — transport T2
- [ ] Implement `Z_SCE_EWB_INV_DTL_FETCH` ship-to block (Section 8) — transport T3
- [ ] Implement `Z_SCE_EWB_BOT_CREATE_OB` `ewb_create` mapping (Section 9) — transport T3b
- [ ] Assess / implement `trans_ind` derivation if blank on test documents
- [ ] Assess `Z_SCE_EWB_BOT_CREATE_OB_JW` / `_FIORI` for parallel T3b changes
- [ ] Syntax check and peer review (all modified objects)
- [ ] Maintain `ZLOG_EXEC_VAR` in client 514 — transport T4
- [ ] Execute test plan TC-01 through TC-14
- [ ] Functional sign-off
- [ ] Transport to QA → Production per P10 change management process

---

## 17. Appendix

### A. RD2 Reference Transports

| Transport | Description |
|---|---|
| **RD2K9A5LAN** | Dictionary changes + API structure `ZST_EWAY_BILL_GENERATION_REQ1` + `ZCL_EWB_API` ship-to code blocks |
| **RD2K9A5LGZ** | ABAP logic / mapping / parameter configuration |

### B. Key P10 Function Modules

| Function Module | Function Group | Role | Change |
|---|---|---|---|
| `Z_SCE_EWB_INV_DTL_FETCH` | `ZLOG_EBILL` | Fetch invoice/header data | **Modify** — ship-to `WE` block (Section 8) |
| `Z_SCE_EWB_BOT_CREATE_OB` | `ZLOG_EBILL` | Outbound BOT/API orchestration | **Modify** — `ewb_create` mapping (Section 9) |
| `Z_SCE_EWB_BOT_SERVICE_CALL` | `ZBOT_SERVICE` | Service call wrapper (BOT queue) | No change |
| `Z_SCE_EWB_DATA_SAVE` | `ZLOG_EBILL` | Persist header/items to DB | No change |
| `Z_SCM_IRN_EWB_GENERATION` | `ZSCM_IRN_EWB` | IRN-based EWB | No change (out of scope) |

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

### D. P10 vs RD4 Differences

| Item | RD4 | P10 (D11) |
|---|---|---|
| Target client | 225 | **514** |
| `Z_SCE_EWB_BOT_CREATE_OB` | Already maps `ewb_create` | **Missing — T3b required** |
| `ZSCM_EWB_IRN_ACTIVE` | Verify per client | **X** (active) |
| `ZSCM_EWB_DISP_GSTIN` / `SHIP_GSTIN` | May exist | **Not found** — Section 7.6 optional |
| Development server | RD4 DEV | **D11** |

### E. Document Control

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | 16-Jun-2026 | — | Initial delta document for P10 (D11) based on RD2/RD4 reference and D11 MCP analysis |

---

*End of Document*
