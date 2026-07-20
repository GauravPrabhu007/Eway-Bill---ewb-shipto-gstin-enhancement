# Function Specification / Delta Code Change Document

## IRN e-Way Bill — ExpShipDtls / DispDtls / ShipTo Gstin·TrdNm — **RP2 (RD2)**

| Attribute | Value |
|-----------|--------|
| **Document type** | Technical FS + Exact Delta |
| **System** | **RP2 / RD2** (SAP ECC / NetWeaver 7.31) |
| **Reference advisory** | GSTN Advisory §4 — EWB by IRN; Prod **1-Aug-2026** |
| **Non-IRN reference** | CD:**8089362** / TR:**RD2K9A5LAN** |
| **Aligned with** | RD4 design (`FS_Delta_IRN_EWB_ShipToGSTIN_RD4.md`) + structure-level params |
| **ABAP rules** | `ABAP Rules - 02-04-2026` |
| **Document date** | 20.07.2026 |
| **MCP baseline** | `user-sap_mcp_rd2` — `Z_SCM_IRN_EWB_GENERATION`, `ZSCM_EXP_SHIP_DTLS_EWB`, `ZCL_EWB_API=>GENERATE_IRN_EWB` |
| **Supersedes** | Prior content of this file (15.07.2026 — Non-IRN param reuse only) |

---

## 0. Design summary (confirmed)

### 0.1 Four IRN params (`ZLOG_EXEC_VAR`) — all gated by `REMARKS = trans_ind`

| NAME | Controls | When ACTIVE for `trans_ind` |
|------|----------|------------------------------|
| **`ZSCE_EWB_IRN_API_EXPSHPDTLS`** | Entire JSON object **`ExpShipDtls`** | Populate & send ExpShipDtls |
| **`ZSCE_EWB_IRN_API_DISPDLTS`** | Entire JSON object **`DispDtls`** | Populate & send DispDtls |
| **`ZSCE_EWB_IRN_API_SHIPTO_GSTIN`** | Field **`ExpShipDtls.Gstin`** | Set Gstin ← `sh_gstn` (**only if** ExpShipDtls param also active) |
| **`ZSCE_EWB_IRN_API_SHIPTO_TRDNAME`** | Field **`ExpShipDtls.TrdNm`** | Set TrdNm ← `to_other_party_name` (**only if** ExpShipDtls param also active) |

### 0.2 Dependency (mandatory rule)

```text
ExpShipDtls param INACTIVE / not maintained for trans_ind
    → do NOT send ExpShipDtls at all
    → Gstin / TrdNm NOT sent even if SHIPTO_* params are ACTIVE for that trans_ind

ExpShipDtls param ACTIVE for trans_ind
    → send ExpShipDtls (Addr/Loc/Pin/Stcd as today)
    → Gstin  only if ZSCE_EWB_IRN_API_SHIPTO_GSTIN   ACTIVE for same trans_ind
    → TrdNm  only if ZSCE_EWB_IRN_API_SHIPTO_TRDNAME ACTIVE for same trans_ind
```

Same pattern for DispDtls: structure param gates the whole object; no nested ShipTo fields under DispDtls.

### 0.3 Isolation from Non-IRN

| Path | Params |
|------|--------|
| Non-IRN | `ZSCE_EWB_API_SHIPTO_GSTIN` / `ZSCE_EWB_API_SHIPTO_TRDNAME` |
| **IRN** | `ZSCE_EWB_IRN_API_*` (four names above) |

---

## 1. RD2 AS-IS (MCP 20.07.2026)

| Object | Status |
|--------|--------|
| `ZSCM_EXP_SHIP_DTLS_EWB` | Addr/Loc/Pin/Stcd only — **no** `_gstin` / `_trd_nm` |
| `ZSCM_EXP_DISP_DTLS_EWB` | Nm/Addr/Loc/Pin/Stcd — OK |
| `Z_SCM_IRN_EWB_GENERATION` | Always maps both DispDtls + ExpShipDtls; no ShipTo Gstin; SELECT only STATE + MULTIMODE |
| `GENERATE_IRN_EWB` | `serialize( compress = abap_false )` then `NULL-VALUE` → `null` |
| `ZSCE_EWB_HDR` | `SHIP_TO_GSTIN` / `SHIP_TO_TRADENAME` columns exist |

---

## 2. AS-IS vs TO-BE

| Item | AS-IS | TO-BE |
|------|-------|-------|
| ExpShipDtls in JSON | Always sent | Only if `ZSCE_EWB_IRN_API_EXPSHPDTLS` ACTIVE for `trans_ind` |
| DispDtls in JSON | Always sent | Only if `ZSCE_EWB_IRN_API_DISPDLTS` ACTIVE for `trans_ind` |
| ExpShipDtls.Gstin | Not in DDIC / not sent | If ExpShip **and** SHIPTO_GSTIN params ACTIVE → `sh_gstn` |
| ExpShipDtls.TrdNm | Not in DDIC / not sent | If ExpShip **and** SHIPTO_TRDNAME params ACTIVE → `to_other_party_name` |
| HDR ship_to_* | Not on IRN path | Update when Gstin/TrdNm mapped |

---

## 3. Impacted objects

| # | Object | Action |
|---|--------|--------|
| 1 | `ZSCM_EXP_SHIP_DTLS_EWB` | Add `_gstin`, `_trd_nm` |
| 2 | `Z_SCM_IRN_EWB_GENERATION` | Param SELECT; conditional Disp/ExpShip mapping; ShipTo fields; HDR |
| 3 | `ZCL_EWB_API=>GENERATE_IRN_EWB` | Omit empty nested structures from JSON (see §6) |
| 4 | `ZLOG_EXEC_VAR` | Create 4 IRN param names × `trans_ind` 1–4 as needed |
| 5 | Non-IRN / BOT / INV_DTL | **None** |

---

## 4. DDIC — `ZSCM_EXP_SHIP_DTLS_EWB`

**AS-IS:** `_addr1`, `_addr2`, `_loc`, `_pin`, `_stcd`  
**TO-BE add:**

| Component | Type | JSON |
|-----------|------|------|
| `_gstin` | `ZEWB_STRING` | `Gstin` |
| `_trd_nm` | `ZEWB_STRING` | `TrdNm` |

Activate; ensure `ZSCM_IRN_EWB_CREATE` consistent.

---

## 5. Exact delta — FM `Z_SCM_IRN_EWB_GENERATION`

**FUGR:** `ZSCM_IRN_EWB` | **NW 7.31**

### 5.1 Constants / DATA

```abap
" BEGIN: Cursor Generated Code
" BEGIN CURSOR CODE ON 20.07.2026 CD:<CHANGE_DOC> TR:<RD2_TRANSPORT> AUTHOR:<USER>
" Purpose: IRN ExpShip/Disp structure gates + ExpShipDtls.Gstin/TrdNm (GSTN Adv §4)
CONSTANTS: lc_irn_expship      TYPE rvari_vnam VALUE 'ZSCE_EWB_IRN_API_EXPSHPDTLS',
           lc_irn_disp         TYPE rvari_vnam VALUE 'ZSCE_EWB_IRN_API_DISPDLTS',
           lc_irn_shipto_gstin TYPE rvari_vnam VALUE 'ZSCE_EWB_IRN_API_SHIPTO_GSTIN',
           lc_irn_shipto_trdnm TYPE rvari_vnam VALUE 'ZSCE_EWB_IRN_API_SHIPTO_TRDNAME'.

DATA: lv_expship_active    TYPE abap_bool,
      lv_disp_active       TYPE abap_bool,
      lv_ship_gstin_active TYPE abap_bool,
      lv_ship_togstin      TYPE string,
      lv_to_tradename      TYPE string,
      lv_trans_ind         TYPE char1.
" END CURSOR CODE ON 20.07.2026 CD:<CHANGE_DOC> TR:<RD2_TRANSPORT> AUTHOR:<USER>
" END: Cursor Generated Code
```

### 5.2 Extend SELECT

**AS-IS:**

```abap
WHERE name IN ('ZSCE_EWB_STATE_API', 'ZSCM_EWB_IRN_MULTIMODE')
  AND active = abap_true.
```

**TO-BE:**

```abap
" BEGIN: Cursor Generated Code
SELECT name
       numb
       remarks
       ewb_uom_d
       active
  FROM zlog_exec_var
  INTO TABLE lt_zlog_exec_var
  WHERE name IN ('ZSCE_EWB_STATE_API',
                 'ZSCM_EWB_IRN_MULTIMODE',
                 'ZSCE_EWB_IRN_API_EXPSHPDTLS',
                 'ZSCE_EWB_IRN_API_DISPDLTS',
                 'ZSCE_EWB_IRN_API_SHIPTO_GSTIN',
                 'ZSCE_EWB_IRN_API_SHIPTO_TRDNAME')
    AND active = abap_true.
IF sy-subrc = 0.
  SORT lt_zlog_exec_var BY ewb_uom_d.  " state READ BINARY SEARCH
ENDIF.
" END: Cursor Generated Code
```

### 5.3 Replace unconditional Disp / ExpShip mapping

Keep vehicle / IRN / distance / multimode logic as today.

**After** `lv_trans_ind` is available:

```abap
lv_trans_ind = im_input_data-ewb_create-trans_ind.
```

**Structure-param checks** (after state table load; before or instead of current Disp/Exp fill):

```abap
" BEGIN: Cursor Generated Code
" BEGIN CURSOR CODE ON 20.07.2026 CD:<CHANGE_DOC> TR:<RD2_TRANSPORT> AUTHOR:<USER>
CLEAR: lv_expship_active,
       lv_disp_active,
       lv_ship_gstin_active,
       lv_ship_togstin,
       lv_to_tradename.

SORT lt_zlog_exec_var BY name remarks.

READ TABLE lt_zlog_exec_var TRANSPORTING NO FIELDS
  WITH KEY name    = lc_irn_disp
           remarks = lv_trans_ind
  BINARY SEARCH.
IF sy-subrc = 0.
  lv_disp_active = abap_true.
ENDIF.

READ TABLE lt_zlog_exec_var TRANSPORTING NO FIELDS
  WITH KEY name    = lc_irn_expship
           remarks = lv_trans_ind
  BINARY SEARCH.
IF sy-subrc = 0.
  lv_expship_active = abap_true.
ENDIF.
" END CURSOR CODE ON 20.07.2026 CD:<CHANGE_DOC> TR:<RD2_TRANSPORT> AUTHOR:<USER>
" END: Cursor Generated Code
```

**DispDtls — only if `lv_disp_active`:**

```abap
" BEGIN: Cursor Generated Code
CLEAR lw_input-_disp_dtls.
IF lv_disp_active = abap_true.
  " existing mapping (from_address1/2, from_place, from_other_party_name, from_pin, from_state→stcd)
  lw_input-_disp_dtls-_addr1 = im_input_data-ewb_create-from_address1.
  ...
ENDIF.
" END: Cursor Generated Code
```

**ExpShipDtls — only if `lv_expship_active`:**

```abap
" BEGIN: Cursor Generated Code
CLEAR lw_input-_exp_ship_dtls.
IF lv_expship_active = abap_true.
  " existing Addr/Loc/Pin/Stcd mapping (to_address*, to_place, to_pin, to_state→stcd)

  " Gstin — nested under ExpShip only
  READ TABLE lt_zlog_exec_var TRANSPORTING NO FIELDS
    WITH KEY name    = lc_irn_shipto_gstin
             remarks = lv_trans_ind
    BINARY SEARCH.
  IF sy-subrc = 0.
    lv_ship_gstin_active = abap_true.
    lv_ship_togstin = im_input_data-ewb_create-sh_gstn.  " no to_gstin fallback
    lw_input-_exp_ship_dtls-_gstin = lv_ship_togstin.
  ENDIF.

  " TrdNm — nested under ExpShip only
  READ TABLE lt_zlog_exec_var TRANSPORTING NO FIELDS
    WITH KEY name    = lc_irn_shipto_trdnm
             remarks = lv_trans_ind
    BINARY SEARCH.
  IF sy-subrc = 0.
    lv_to_tradename = im_input_data-ewb_create-to_other_party_name.
    lw_input-_exp_ship_dtls-_trd_nm = lv_to_tradename.
  ENDIF.
ENDIF.
" If lv_expship_active = false: structure stays CLEAR → omit from JSON (§6)
" ShipTo params alone must NOT populate ExpShipDtls
" END: Cursor Generated Code
```

**SORT note:** Run `SORT BY ewb_uom_d` for state READs **inside** the active Disp/ExpShip blocks (or keep a copy `lt_zlog_state`). Then `SORT BY name remarks` for param READs. Avoid mixing sort keys.

### 5.4 HDR persist (when Gstin/TrdNm mapped)

On success UPDATE (extend existing):

```abap
ship_to_gstin     = lv_ship_togstin
ship_to_tradename = lv_to_tradename
```

Only meaningful when ExpShip + ShipTo params were active.

### 5.5 Optional guard

If `lv_expship_active` and `lv_ship_gstin_active` and `lv_ship_togstin` initial → SAP E / `5001` before API.

---

## 6. Exact delta — `ZCL_EWB_API=>GENERATE_IRN_EWB` (omit empty structures)

**Problem:** Today `compress = abap_false`, so a CLEARed nested structure still serializes as an empty/zero-filled object and appears in JSON.

**TO-BE (preferred):**

```abap
" BEGIN: Cursor Generated Code
" BEGIN CURSOR CODE ON 20.07.2026 CD:<CHANGE_DOC> TR:<RD2_TRANSPORT> AUTHOR:<USER>
zui_cl_json=>serialize(
  EXPORTING
    data        = im_input
    compress    = abap_true   " skip initial / empty components incl. nested DispDtls/ExpShipDtls
    pretty_name = zui_cl_json=>pretty_mode-camel_case
  RECEIVING
    r_json      = lw_input_json ).
" END CURSOR CODE ON 20.07.2026 CD:<CHANGE_DOC> TR:<RD2_TRANSPORT> AUTHOR:<USER>
" END: Cursor Generated Code
```

Keep existing:

```abap
REPLACE ALL OCCURRENCES OF '"NULL-VALUE"' IN lw_input_json WITH 'null'.
```

**Why safe with NULL-VALUE:** intentional nulls are non-initial strings (`NULL-VALUE`), so they are **not** compressed away; they still become JSON `null`.

**Regression:** Confirm other blank fields that relied on serializing as `""` still meet Jio GST expectations (most optional blanks already use `NULL-VALUE`).

**Fallback** (if compress change rejected): after serialize, strip empty `DispDtls` / `ExpShipDtls` objects via controlled REPLACE — less preferred.

---

## 7. Config — `ZLOG_EXEC_VAR` (RP2)

### 7.1 Go-live recommendation (advisory: Gstin mandatory when ExpShip sent)

For each `REMARKS` = `1` / `2` / `3` / `4` (as required by business):

| NAME | ACTIVE | REMARKS |
|------|--------|---------|
| `ZSCE_EWB_IRN_API_EXPSHPDTLS` | `X` | `1`…`4` |
| `ZSCE_EWB_IRN_API_DISPDLTS` | `X` | `1`…`4` (or subset per scenario design) |
| `ZSCE_EWB_IRN_API_SHIPTO_GSTIN` | `X` | `1`…`4` |
| `ZSCE_EWB_IRN_API_SHIPTO_TRDNAME` | `X` | `1`…`4` (optional) |

### 7.2 Flexibility examples

| Goal | Config action |
|------|----------------|
| No ExpShip (hence no Gstin) for Regular | Deactivate ExpShipDtls Remarks=`1` |
| ExpShip without Gstin for one type | ExpShip ACTIVE; SHIPTO_GSTIN inactive for that Remarks |
| SHIPTO_GSTIN ACTIVE but ExpShip inactive | **Gstin not sent** (structure gate wins) |
| No DispDtls for scenario 1 | Deactivate DispDtls Remarks=`1` |

---

## 8. Behavior flow

```text
trans_ind = ewb_create-trans_ind

IF DispDtls param ACTIVE for trans_ind
  → fill DispDtls
ELSE
  → CLEAR DispDtls  → omitted from JSON

IF ExpShipDtls param ACTIVE for trans_ind
  → fill Addr/Loc/Pin/Stcd
  → IF SHIPTO_GSTIN param ACTIVE → Gstin = sh_gstn
  → IF SHIPTO_TRDNAME param ACTIVE → TrdNm = to_other_party_name
ELSE
  → CLEAR ExpShipDtls  → omitted from JSON
  → ignore SHIPTO_* params for this call
```

---

## 9. Tests (RP2)

| ID | Setup | Expected |
|----|--------|----------|
| AT-001 | ExpShip+Disp+SHIPTO_GSTIN ACTIVE for `trans_ind=4` | Both structures; Gstin = sh_gstn |
| AT-002 | ExpShip inactive; SHIPTO_GSTIN ACTIVE | **No** ExpShipDtls; **no** Gstin |
| AT-003 | ExpShip ACTIVE; SHIPTO_GSTIN inactive | ExpShip without Gstin |
| AT-004 | DispDtls inactive | **No** DispDtls in JSON |
| AT-005 | ExpShip ACTIVE; TrdNm ACTIVE | TrdNm present |
| AT-006 | `trans_ind=1` vs `2` different param rows | Independent activation |
| AT-007 | Non-IRN regression | Unchanged CD:8089362 |
| AT-008 | compress=true + NULL-VALUE fields | Still JSON `null` where expected |

---

## 10. Transport checklist

- [ ] `ZSCM_EXP_SHIP_DTLS_EWB` (`_gstin`, `_trd_nm`)
- [ ] `Z_SCM_IRN_EWB_GENERATION`
- [ ] `ZCL_EWB_API` method `GENERATE_IRN_EWB` (compress / omit)
- [ ] `ZLOG_EXEC_VAR` — four IRN param names × required `trans_ind`
- [ ] SCI/SLIN; sandbox; Prod before **1-Aug-2026**

---

## 11. Implementation sequence

1. DDIC ExpShip `_gstin` / `_trd_nm` → activate.  
2. FM: SELECT + structure gates + ShipTo nested mapping + HDR.  
3. `GENERATE_IRN_EWB`: `compress = abap_true` (or fallback omit).  
4. Config rows for go-live.  
5. AT-001…008.

---

## 12. Out of scope

Non-IRN params · IRN+EWB together (`ShipDtls.Gstin` on e-Invoice) · URP redesign · Voluntary EWB Closure · RD4 transport (apply same design there separately if not already)

---

**End of document (RP2 / RD2)**
