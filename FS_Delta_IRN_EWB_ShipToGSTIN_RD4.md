# Function Specification / Delta Code Change Document

## IRN e-Way Bill — ShipToGSTIN / ExpShipDtls.Gstin (Jio GST API) — **RD4**

| Attribute | Value |
|-----------|--------|
| **Document type** | Technical FS + Exact Delta |
| **System** | **RD4** (SAP ECC / NetWeaver 7.31) |
| **Reference advisory** | GSTN Advisory (17-Jun-2026) — Production **1-Aug-2026** |
| **Reference Non-IRN (pattern)** | RD2 CD:**8089362** / TR:**RD2K9A5LAN** (`ShipToGSTIN` / `ShipToTradeName`); RD4 Non-IRN DDIC already present |
| **Sister document** | `FS_Delta_IRN_EWB_ShipToGSTIN.md` (RD2) |
| **ABAP rules** | Folder `ABAP Rules - 02-04-2026` |
| **Document date** | 16.07.2026 |
| **Scope** | IRN-based EWB creation (EWB-by-IRN) on **RD4** only |
| **MCP verification** | Confirmed via **user-sap_mcp_rd4** (structures, FM `Z_SCM_IRN_EWB_GENERATION`, HDR, config) |

---

## 1. Description

GSTN mandates **Ship-to GSTIN** for applicable e-Way Bill scenarios. For **EWB by IRN**, API field **`ExpShipDtls.Gstin`** is mandatory (error **5001** if missing). Optional **`ExpShipDtls.TrdNm`**.

### RD4 AS-IS (confirmed)

| Area | RD4 status |
|------|------------|
| Non-IRN payload structure `ZST_EWAY_BILL_GENERATION_REQ1` | Already has `shiptogstin`, `shiptotradename` |
| Table `ZSCE_EWB_HDR` | Already has `SHIP_TO_GSTIN`, `SHIP_TO_TRADENAME` |
| Data elements `ZSCM_SHIPTOGSTN_DE` / `ZSCM_SHIPTOTRADE_DE` | Already exist |
| Config `ZSCE_EWB_API_SHIPTO_GSTIN` / `ZSCE_EWB_API_SHIPTO_TRDNAME` | **Missing** in RD4 `ZLOG_EXEC_VAR` (full table scanned) |
| Flag `ZSCM_EWB_IRN_ACTIVE` | **Active** (`ACTIVE = X`) |
| IRN structure `ZSCM_EXP_SHIP_DTLS_EWB` | **No** `_gstin` / `_trd_nm` |
| FM `Z_SCM_IRN_EWB_GENERATION` | Maps ExpShip Addr/Loc/Pin/Stcd only — **does not** set Gstin/TrdNm |

**IRN gap on RD4 is the same as RD2:** ExpShipDtls cannot carry Gstin until DDIC + FM mapping are added.

> **Note:** `ZCL_EWB_API` source could not be displayed on RD4 via MCP (ADT 403 — no display authorization). Non-IRN ShipTo **DDIC** is present; Non-IRN **param** rows are missing. Before Prod, confirm in SE24 that Non-IRN mapping (CD:8089362 pattern) exists in `GENERATE_E_WAY_BILL`; if not, transport Non-IRN code separately. This IRN delta does not replace that check.

---

## 2. AS-IS vs TO-BE (API payload — IRN)

| GSTN field | AS-IS (RD4 IRN) | TO-BE (RD4 IRN) | Non-IRN reference |
|------------|-----------------|-----------------|-------------------|
| `ExpShipDtls.Gstin` | Not present / not sent | Mandatory when param active; from `sh_gstn` only (no `to_gstin` fallback) | `ShipToGSTIN` ← `sh_gstn` |
| `ExpShipDtls.TrdNm` | Not present / not sent | Optional when param active; from `to_other_party_name` | `ShipToTradeName` ← `to_other_party_name` |
| `ExpShipDtls` Addr/Loc/Pin/Stcd | Already mapped in FM | No change | N/A |
| Persist `ZSCE_EWB_HDR-SHIP_TO_*` | Not updated on IRN path | Update when ship GSTIN/trdname mapped | HDR columns already exist |

---

## 3. Call chain (impacted — RD4)

```
Z_SCE_EWB_BOT_CREATE_OB          (FUGR ZLOG_EBILL)
  └─ builds ZEWB_CREATE (sh_gstn, to_gstin, bill_to_gstin, to_other_party_name, …)
       └─ ZCL_EWB_API=>GENERATE_E_WAY_BILL
            ├─ [IRN + ZSCM_EWB_IRN_ACTIVE = X]   ← already ACTIVE on RD4
            │     └─ Z_SCM_IRN_EWB_GENERATION     ★ CODE CHANGE (FUGR ZSCM_IRN_EWB)
            │           └─ zcl_ewb_api=>generate_irn_ewb  (serialize — no logic change if DDIC done)
            └─ [Non-IRN] ShipToGSTIN fields exist in DDIC; confirm class mapping + add params
```

Upstream `ZEWB_CREATE` already has `sh_gstn`, `to_gstin`, `bill_to_gstin`, `to_other_party_name`, `trans_ind` on RD4 — **no change**.

---

## 4. Impacted objects summary (RD4)

| # | Object | Type | Impact | Action |
|---|--------|------|--------|--------|
| 1 | `ZSCM_EXP_SHIP_DTLS_EWB` | Structure (DDIC) | **High** | Add `_gstin`, `_trd_nm` |
| 2 | `ZSCM_IRN_EWB_CREATE` | Structure (DDIC) | Indirect | Re-activate / check nested ExpShip |
| 3 | `Z_SCM_IRN_EWB_GENERATION` | Function Module | **High** | Map Gstin/TrdNm; param gate; HDR persist |
| 4 | `ZCL_EWB_API` | Class | **None** for IRN serialize | `GENERATE_IRN_EWB` picks up new components automatically |
| 5 | `ZLOG_EXEC_VAR` | Config | **High on RD4** | **Create** `ZSCE_EWB_API_SHIPTO_GSTIN` / `ZSCE_EWB_API_SHIPTO_TRDNAME` (not present today) |
| 6 | `ZSCE_EWB_HDR` | Transparent table | **None** | Columns already exist |
| 7 | `ZSCM_SHIPTOGSTN_DE` / `ZSCM_SHIPTOTRADE_DE` | Data elements | **None** | Already exist |
| 8 | `Z_SCE_EWB_BOT_CREATE_OB` | Function Module | **None** | Already maps `sh_gstn` |
| 9 | `ZST_EWAY_BILL_GENERATION_REQ1` | Structure | **None** | Non-IRN only; already has ShipTo fields |

---

## 5. Exact delta — Object 1: Structure `ZSCM_EXP_SHIP_DTLS_EWB`

### 5.1 Purpose

Extend ExpShipDtls so `zui_cl_json=>serialize` with `pretty_mode-camel_case` emits:

- `_gstin` → **`Gstin`**
- `_trd_nm` → **`TrdNm`**

### 5.2 AS-IS (RD4 MCP — confirmed)

```
_addr1 : ZEWB_STRING
_addr2 : ZEWB_STRING
_loc   : ZEWB_STRING
_pin   : NUM6
_stcd  : ZEWB_STRING
```

### 5.3 TO-BE (add at end)

| Component | Type | JSON name | Mandatory in API |
|-----------|------|-----------|------------------|
| `_gstin` | `ZEWB_STRING` | `Gstin` | Yes (when EWB-by-IRN / advisory) |
| `_trd_nm` | `ZEWB_STRING` | `TrdNm` | No (optional) |

**DDIC notes:**

- Package: same as existing IRN EWB structures (not `$TMP`).
- Activate structure; ensure `ZSCM_IRN_EWB_CREATE` remains consistent.
- No 7.40+ syntax in dependent code.
- Prefer `ZEWB_STRING` for API JSON structures (consistent with Addr1/Loc).

### 5.4 Transport

Include DDIC + FM in same RD4 transport (related objects together).

---

## 6. Exact delta — Object 3: FM `Z_SCM_IRN_EWB_GENERATION`

**Function group:** `ZSCM_IRN_EWB`  
**ABAP compatibility:** NetWeaver 7.31 — declare all variables upfront; no inline `DATA()`, no `VALUE`/`NEW`/`||`, no `@host` in OpenSQL.

### 6.1 RD4 AS-IS insertion point

Immediately **after** existing ExpShipDtls state mapping, and **before** transporter null handling / auth loop / `generate_irn_ewb`.

RD4 current code ends ExpShip mapping with:

```abap
READ TABLE lt_zlog_exec_var INTO lw_zlog_exec_var WITH KEY ewb_uom_d = im_input_data-ewb_create-to_state
                                                           BINARY SEARCH.
IF sy-subrc = 0.
  lw_input-_exp_ship_dtls-_stcd      = lw_zlog_exec_var-remarks.
ENDIF.
```

Insert the new Gstin/TrdNm block **after** this `ENDIF`.

### 6.2 Additional DATA / CONSTANTS

```abap
" BEGIN: Cursor Generated Code
" BEGIN CURSOR CODE ON 16.07.2026 CD:<CHANGE_DOC> TR:<RD4_TRANSPORT> AUTHOR:<USER>
" Purpose: ExpShipDtls.Gstin / TrdNm for Jio GST EWB-by-IRN on RD4 (GSTN Advisory 17.06.2026)
CONSTANTS: lc_shipto_gstin   TYPE rvari_vnam VALUE 'ZSCE_EWB_API_SHIPTO_GSTIN',
           lc_shipto_trdname TYPE rvari_vnam VALUE 'ZSCE_EWB_API_SHIPTO_TRDNAME'.

DATA: lv_ship_gstin_active TYPE abap_bool,
      lv_ship_togstin      TYPE string,
      lv_to_tradename      TYPE string.
" END CURSOR CODE ON 16.07.2026 CD:<CHANGE_DOC> TR:<RD4_TRANSPORT> AUTHOR:<USER>
" END: Cursor Generated Code
```

> Replace `<CHANGE_DOC>`, `<RD4_TRANSPORT>`, `<USER>` before transport.

### 6.3 Extend existing `ZLOG_EXEC_VAR` SELECT (RD4-specific AS-IS)

**AS-IS on RD4 (confirmed):**

```abap
SELECT name
       numb
       remarks
       ewb_uom_d
  FROM zlog_exec_var
  INTO TABLE lt_zlog_exec_var
  WHERE name = 'ZSCE_EWB_STATE_API'
    AND active = abap_true.
IF sy-subrc = 0.
  SORT lt_zlog_exec_var BY ewb_uom_d.
ENDIF.
```

**TO-BE on RD4:**

```abap
" BEGIN: Cursor Generated Code
" BEGIN CURSOR CODE ON 16.07.2026 CD:<CHANGE_DOC> TR:<RD4_TRANSPORT> AUTHOR:<USER>
SELECT name
       numb
       remarks
       ewb_uom_d
  FROM zlog_exec_var
  INTO TABLE lt_zlog_exec_var
  WHERE name IN ('ZSCE_EWB_STATE_API',
                 'ZSCE_EWB_API_SHIPTO_GSTIN',
                 'ZSCE_EWB_API_SHIPTO_TRDNAME')
    AND active = abap_true.
IF sy-subrc = 0.
  " Keep ewb_uom_d sort for existing state READ (BINARY SEARCH)
  SORT lt_zlog_exec_var BY ewb_uom_d.
ENDIF.
" END CURSOR CODE ON 16.07.2026 CD:<CHANGE_DOC> TR:<RD4_TRANSPORT> AUTHOR:<USER>
" END: Cursor Generated Code
```

**Important (RD4 vs RD2):**

- RD4 AS-IS SELECT is **only** `ZSCE_EWB_STATE_API` (no `ZSCM_EWB_IRN_MULTIMODE` in this FM). Do not add MULTIMODE unless RD4 already uses it elsewhere.
- After loading ShipTo params, **state** READ still uses `WITH KEY ewb_uom_d` + BINARY SEARCH → keep `SORT BY ewb_uom_d` for that path.
- For ShipTo param READ by `name` / `remarks`, use a **second** sorted copy **or** SORT by `name remarks` only for ShipTo reads **after** state mapping is done (see §6.4). Recommended approach in §6.4: SORT by `name remarks` immediately before ShipTo READs (state mapping already completed).

### 6.4 Mapping logic (exact delta block)

Place **after** ExpShip `_stcd` mapping and **after** state READs are finished:

```abap
" BEGIN: Cursor Generated Code
" BEGIN CURSOR CODE ON 16.07.2026 CD:<CHANGE_DOC> TR:<RD4_TRANSPORT> AUTHOR:<USER>
" Map ExpShipDtls.Gstin / TrdNm — mirror Non-IRN CD:8089362 param pattern (RD4)
SORT lt_zlog_exec_var BY name remarks.

CLEAR: lv_ship_gstin_active,
       lv_ship_togstin,
       lv_to_tradename.

" ShipTo GSTIN — only if active param exists with REMARKS = current trans_ind
" (same gate as Non-IRN CD:8089362 — no name-only fallback)
READ TABLE lt_zlog_exec_var
  TRANSPORTING NO FIELDS
  WITH KEY name    = lc_shipto_gstin
           remarks = im_input_data-ewb_create-trans_ind
  BINARY SEARCH.
IF sy-subrc = 0.
  lv_ship_gstin_active = abap_true.
  " Source: sh_gstn only (already determined upstream) — no to_gstin fallback
  lv_ship_togstin = im_input_data-ewb_create-sh_gstn.
  lw_input-_exp_ship_dtls-_gstin = lv_ship_togstin.
ENDIF.

" ShipTo Trade Name (optional) — same REMARKS = trans_ind gate
READ TABLE lt_zlog_exec_var
  TRANSPORTING NO FIELDS
  WITH KEY name    = lc_shipto_trdname
           remarks = im_input_data-ewb_create-trans_ind
  BINARY SEARCH.
IF sy-subrc = 0.
  lv_to_tradename = im_input_data-ewb_create-to_other_party_name.
  lw_input-_exp_ship_dtls-_trd_nm = lv_to_tradename.
ENDIF.

" Persist to ZSCE_EWB_HDR (parity with Non-IRN)
IF lv_ship_gstin_active = abap_true
   AND im_input_data-ewb_create-source_req_id IS NOT INITIAL.

  CALL FUNCTION 'ENQUEUE_EZSCE_EWB_HDR'
    EXPORTING
      mode_zsce_ewb_hdr = lc_e
      mandt             = sy-mandt
      req_id            = im_input_data-ewb_create-source_req_id
    EXCEPTIONS
      foreign_lock      = 1
      system_failure    = 2
      OTHERS            = 3.
  IF sy-subrc = 0.
    UPDATE zsce_ewb_hdr
      SET ship_to_gstin     = lv_ship_togstin
          ship_to_tradename = lv_to_tradename
          changed_by        = sy-uname
          changed_on        = sy-datum
          changed_tm        = sy-uzeit
      WHERE req_id = im_input_data-ewb_create-source_req_id.
    IF sy-subrc = 0.
      COMMIT WORK.
    ELSE.
      ROLLBACK WORK.
    ENDIF.

    CALL FUNCTION 'DEQUEUE_EZSCE_EWB_HDR'
      EXPORTING
        mode_zsce_ewb_hdr = lc_e
        mandt             = sy-mandt
        req_id            = im_input_data-ewb_create-source_req_id.
  ENDIF.
ENDIF.
" END CURSOR CODE ON 16.07.2026 CD:<CHANGE_DOC> TR:<RD4_TRANSPORT> AUTHOR:<USER>
" END: Cursor Generated Code
```

**Notes:**

- `lc_e` already exists in RD4 FM (`CONSTANTS: lc_e TYPE c VALUE 'E'`).
- `SORT BY name remarks` for ShipTo must run **after** state mapping (which needs `SORT BY ewb_uom_d`). Sequence: SELECT → SORT by ewb_uom_d → Disp/Exp state READs → SORT by name remarks → ShipTo block.
- Prefer early HDR persist (before API) to match Non-IRN auditability.

### 6.5 Alternative: fold into existing success UPDATE

If early COMMIT is undesirable, omit persist in §6.4 and extend RD4 success UPDATE:

**AS-IS (RD4):**

```abap
UPDATE zsce_ewb_hdr SET ewb_no      = et_ewb_out-_ewb_no
                        ewb_status  = 'C'
                        bot_status  = 'S'
                        api_indi    = 'X'
                        created_on  = sy-datum
                        created_tm  = sy-uzeit
                        api_ewbvalidityto = et_ewb_out-_ewb_valid_till
                        transactionid = lw_txn_id
                       WHERE req_id = im_input_data-ewb_create-source_req_id.
```

**TO-BE:** add

```abap
ship_to_gstin     = lv_ship_togstin
ship_to_tradename = lv_to_tradename
```

**Recommendation:** persist **before** API (§6.4) for Non-IRN parity.

### 6.6 Optional pre-call validation (GSTN 2323)

```abap
" BEGIN: Cursor Generated Code
IF lv_ship_gstin_active = abap_true
   AND im_input_data-ewb_create-bill_to_gstin IS NOT INITIAL
   AND lv_ship_togstin IS NOT INITIAL
   AND im_input_data-ewb_create-bill_to_gstin = lv_ship_togstin.
  CLEAR lw_return.
  lw_return-type    = 'E'.
  lw_return-log_no  = '2323'.
  lw_return-message = 'Bill-to and Ship-to GSTIN cannot be identical'.
  APPEND lw_return TO et_return.
  RETURN.
ENDIF.
" END: Cursor Generated Code
```

Do **not** hard-code URP without business sign-off.

---

## 7. Exact delta — Object 4: Class `ZCL_EWB_API` (IRN)

| Method | Change for this IRN CR |
|--------|------------------------|
| `GENERATE_E_WAY_BILL` | **No change** for IRN routing (flag already active on RD4) |
| `GENERATE_IRN_EWB` | **No change** — serializes `im_input`; new DDIC fields auto-emitted |
| Non-IRN ShipTo block | **Out of scope** of this IRN CR — verify separately (MCP could not read class) |

**Expected JSON fragment after change:**

```json
"ExpShipDtls": {
  "Addr1": "...",
  "Addr2": "...",
  "Loc": "...",
  "Pin": 400001,
  "Stcd": "27",
  "Gstin": "27AAAAA0000A1Z5",
  "TrdNm": "Ship Party Name"
}
```

---

## 8. Exact delta — Object 5: Config `ZLOG_EXEC_VAR` (**mandatory on RD4**)

RD4 currently has **zero** rows for ShipTo API params. Create entries (same names as Non-IRN / RD2):

| NAME | NUMB | ACTIVE | REMARKS | EWB_UOM_D | Purpose |
|------|------|--------|---------|-----------|---------|
| `ZSCE_EWB_API_SHIPTO_GSTIN` | `0001` | `X` | `1` | (blank) | Enable for Regular (`trans_ind=1`) |
| `ZSCE_EWB_API_SHIPTO_GSTIN` | `0002` | `X` | `2` | (blank) | Enable for B2S (`trans_ind=2`) |
| `ZSCE_EWB_API_SHIPTO_GSTIN` | `0003` | `X` | `3` | (blank) | Enable for Disp-From (`trans_ind=3`) |
| `ZSCE_EWB_API_SHIPTO_GSTIN` | `0004` | `X` | `4` | (blank) | Enable for Combination (`trans_ind=4`) |
| `ZSCE_EWB_API_SHIPTO_TRDNAME` | `0001`–`0004` | `X` | `1`/`2`/`3`/`4` | (blank) | Optional TrdNm per trans_ind |

Do **not** rely on blank `REMARKS` / name-only match — gate is strictly `NAME + REMARKS = trans_ind + ACTIVE` (same as Non-IRN CD:8089362).

**Also required for Non-IRN on RD4:** same params power Non-IRN `ShipToGSTIN` if class mapping is present. Creating them once serves **both** paths.

**Already present on RD4:** `ZSCM_EWB_IRN_ACTIVE` = Active — no change needed for IRN routing flag.

---

## 9. Objects with NO code change (explicit)

| Object | Reason |
|--------|--------|
| `Z_SCE_EWB_BOT_CREATE_OB` | Already supplies `sh_gstn` / related fields via `ZEWB_CREATE` |
| `ZEWB_CREATE` / `ZSCE_EWB_API_STR` | Fields already present on RD4 |
| `ZSCE_EWB_HDR` | Columns already exist; only UPDATE logic in FM changes |
| `ZST_EWAY_BILL_GENERATION_REQ1` | Non-IRN only; ShipTo fields already exist |
| `ZSCM_IRN_EWB_CREATE` | Nested ExpShip picks up structure change after activate |

---

## 10. Behavior (FS)

1. BOT create builds API input including `sh_gstn` / bill-to / ship party name.
2. If IRN exists and `ZSCM_EWB_IRN_ACTIVE` = X (already true on RD4), call `Z_SCM_IRN_EWB_GENERATION`.
3. FM builds `DispDtls` and `ExpShipDtls` (address/state as today).
4. If active `ZSCE_EWB_API_SHIPTO_GSTIN` exists with `REMARKS = trans_ind`, set `ExpShipDtls-Gstin` from `sh_gstn` only. If no matching active param → do **not** pass Gstin.
5. If `ZSCE_EWB_API_SHIPTO_TRDNAME` active, set `ExpShipDtls-TrdNm` from `to_other_party_name`.
6. Persist ship-to values to `ZSCE_EWB_HDR`.
7. Call `generate_irn_ewb` → encrypt → POST Jio GST GENERATE_IRN_EWB.
8. Handle response as today.

---

## 11. Errors (GSTN + SAP)

| Code | Meaning | Handling |
|------|---------|----------|
| 5001 | ExpShipDtls.Gstin mandatory | Addressed by this delta |
| 2323 | Bill-to and Ship-to GSTIN same | Optional §6.6 / data fix |
| 2325 / 4074 | State code mismatch | Existing Stcd mapping — regression |
| 3039 | PIN–State mismatch | Regression |
| 5002 | Ship-to GSTIN (IRN+EWB / ShipDtls) | E-invoice path — **out of scope** |

---

## 12. Success

- IRN EWB create returns EWB number as today.
- Payload contains `ExpShipDtls.Gstin` when param active.
- `ZSCE_EWB_HDR-SHIP_TO_GSTIN` / `SHIP_TO_TRADENAME` populated for IRN requests on RD4.

---

## 13. Edge cases

| Case | Handling |
|------|----------|
| `sh_gstn` empty, param active | Send blank Gstin → GSTN 5001; consider SAP E before API (no `to_gstin` fallback for now) |
| `to_gstin` filled but `sh_gstn` empty | Still use `sh_gstn` only — do **not** fall back to `to_gstin` |
| Param inactive / missing | Current RD4 behavior (will fail 5001 after 1-Aug-2026) — **must create params** |
| Bill-to = Ship-to | GSTN 2323 |
| Export / URP | Business confirm; no hard-code in this delta |
| SORT key conflict | State READ before ShipTo SORT (§6.4 sequence) |

---

## 14. Dependencies

| Dependency | Detail |
|------------|--------|
| External | Jio GST GENERATE_IRN_EWB |
| Config | `ZEINV_PARAM` URL for GENERATE_IRN_EWB |
| Config | **New** RD4 `ZLOG_EXEC_VAR` ShipTo params |
| Non-IRN | Field semantics from CD:8089362; RD4 DDIC already aligned |
| DDIC | Activate `ZSCM_EXP_SHIP_DTLS_EWB` before FM |

---

## 15. RD4 vs RD2 delta differences (quick)

| Topic | RD2 | RD4 |
|-------|-----|-----|
| IRN ExpShip structure gap | Yes | **Same** |
| FM mapping gap | Yes | **Same** |
| ShipTo `ZLOG_EXEC_VAR` params | Present (from Non-IRN) | **Missing — must create** |
| `ZSCM_EWB_IRN_ACTIVE` | Check locally | **Already Active** |
| IRN FM SELECT AS-IS | STATE + MULTIMODE (RD2) | **STATE only** |
| Sister FS | This pattern | This document |

---

## 16. Performance / Security checklist

- [ ] No SELECT in loops
- [ ] SORT + BINARY SEARCH for param / state reads (correct order)
- [ ] Only required columns in SELECT
- [ ] No 7.40+ syntax
- [ ] All variables declared upfront
- [ ] SY-SUBRC after READ / UPDATE / ENQUEUE
- [ ] No hard-coded GSTIN
- [ ] Enqueue before HDR UPDATE
- [ ] Code Inspector / SLIN clean before transport

---

## 17. Tests (RD4)

| ID | Scenario | Expected |
|----|----------|----------|
| AT-RD4-001 | IRN EWB, params ON, distinct Bill-to / Ship-to | Success; JSON has ExpShipDtls.Gstin; HDR updated |
| AT-RD4-002 | IRN EWB, params OFF / missing | Fail or no Gstin — prove param gate |
| AT-RD4-003 | IRN EWB, `sh_gstn` empty, `to_gstin` filled | Gstin blank (no fallback); expect 5001 or SAP E |
| AT-RD4-004 | IRN EWB + TrdNm param ON | ExpShipDtls.TrdNm populated |
| AT-RD4-005 | Bill-to = Ship-to | GSTN 2323 or SAP pre-check |
| AT-RD4-006 | Non-IRN regression | ShipToGSTIN works once params + class mapping confirmed |
| AT-RD4-007 | State/PIN cases | 2325/4074/3039 unchanged |
| AT-RD4-008 | Sandbox Jio GST before 1-Aug-2026 | Pass advisory B2S / combo scenarios |

---

## 18. Out of scope (separate CRs)

1. Voluntary EWB Closure API.
2. IRN + EWB together — `ShipDtls.Gstin` in e-Invoice generation.
3. INV_DTL_FETCH URP / Bill-to≠Ship-to redesign.
4. Non-IRN class mapping transport (if missing on RD4) — verify in SE24 first.

---

## 19. Transport checklist (RD4)

- [ ] `ZSCM_EXP_SHIP_DTLS_EWB` (structure)
- [ ] `Z_SCM_IRN_EWB_GENERATION` (FM / FUGR `ZSCM_IRN_EWB`)
- [ ] `ZLOG_EXEC_VAR` table entries for `ZSCE_EWB_API_SHIPTO_*` (**required** — currently absent)
- [ ] Transport description e.g. `RD4 IRN EWB ExpShipDtls.Gstin GSTN Adv 1-Aug-2026 CD:…`
- [ ] Peer review; SCI/SLIN; sandbox; Prod before **1-Aug-2026**

---

## 20. Implementation sequence (RD4)

1. **Config:** Create `ZSCE_EWB_API_SHIPTO_GSTIN` / `ZSCE_EWB_API_SHIPTO_TRDNAME` in `ZLOG_EXEC_VAR` (also enables Non-IRN if class ready).
2. **DDIC:** Add `_gstin`, `_trd_nm` to `ZSCM_EXP_SHIP_DTLS_EWB` → Activate.
3. **FM:** Extend SELECT + mapping + HDR persist in `Z_SCM_IRN_EWB_GENERATION`.
4. **Verify:** SE24 `ZCL_EWB_API` Non-IRN ShipTo mapping present (MCP auth blocked).
5. Unit / sandbox AT-RD4-001…008.
6. Transport RD4 → Q → Prod before **1-Aug-2026**.

---

## 21. ABAP Rules compliance

Aligned with `ABAP Rules - 02-04-2026`:

| Rule | Applied |
|------|---------|
| `00-main.mdc` | 7.31 syntax; FM change for existing encapsulation |
| `02-naming.mdc` | `lv_` / `lc_` / `lt_` / `lw_` |
| `03-database.mdc` | Field-specific SELECT; SORT + BINARY SEARCH |
| `11-interfaces-api.mdc` | Param-driven API payload |
| `12-documentation.mdc` | Cursor begin/end; CD/TR/author |
| `14-transport.mdc` | DDIC + FM together; RD4 transport |
| `08-testing.mdc` | AT-RD4-xxx |
| `20-code-generation-checklist.mdc` | §16 |

---

## 22. MCP evidence summary (RD4)

| Check | Result |
|-------|--------|
| `ZSCM_EXP_SHIP_DTLS_EWB` | Addr/Loc/Pin/Stcd only — no Gstin |
| `Z_SCM_IRN_EWB_GENERATION` | No Gstin/TrdNm mapping |
| `ZSCM_IRN_EWB_CREATE` | Nested `_exp_ship_dtls` → ExpShip structure |
| `ZST_EWAY_BILL_GENERATION_REQ1` | Has `shiptogstin` / `shiptotradename` |
| `ZSCE_EWB_HDR` | Has `SHIP_TO_GSTIN` / `SHIP_TO_TRADENAME` |
| `ZLOG_EXEC_VAR` ShipTo params | **Not found** (221 rows) |
| `ZSCM_EWB_IRN_ACTIVE` | Active |
| `ZCL_EWB_API` source | **ADT 403** — manual SE24 verification needed |

---

**End of document (RD4)**
