# Code Change Document
## E-Way Bill Enhancement — shipToGSTIN & shipToTradeName (Approach 2)

| Field | Value |
|-------|-------|
| **Document ID** | CCD-EWB-SHIPTO-001 |
| **System** | SAP RD2 |
| **Related FS** | FS-EWB-SHIPTO-001 |
| **Version** | 1.0 |
| **Date** | 04-Jun-2026 |

---

## 1. Change summary

| # | Object | Change type | Description |
|---|--------|-------------|-------------|
| 1 | `ZST_EWAY_BILL_GENERATION_REQ` | Dictionary | Add `SHIP_TO_GSTIN`, `SHIP_TO_TRADE_NAME` |
| 2 | `ZCL_EWB_API` | ABAP Class | Update `GENERATE_E_WAY_BILL` — param logic, legacy comment, serialize |
| 3 | `ZLOG_EXEC_VAR` | Config | Add rows for two new param names |

**Approach:** Two separate params; **omit** fields when param does not match (`compress = abap_true`).

---

## 2. Prerequisites / activation order

1. Activate `ZST_EWAY_BILL_GENERATION_REQ`
2. Apply and activate `ZCL_EWB_API`
3. Maintain `ZLOG_EXEC_VAR` configuration
4. UAT test with API log verification

---

## 3. Dictionary change — `ZST_EWAY_BILL_GENERATION_REQ`

**Transaction:** SE11 → Display → Change

### 3.1 Add components

| Component | Data element / type | Length | Short text |
|-----------|---------------------|--------|------------|
| `SHIP_TO_GSTIN` | CHAR (or `ZEWB_STRING`) | 15 | Ship-To GSTIN for Jio API |
| `SHIP_TO_TRADE_NAME` | CHAR | 100 | Ship-To Trade Name for Jio API |

### 3.2 Keep unchanged (do not delete)

| Component | Note |
|-----------|------|
| `DF_GSTIN` | Retained on structure; cleared / not assigned for API |
| `SH_GSTN` | Retained on structure; cleared / not assigned for API |

### 3.3 Expected JSON mapping (via `zui_cl_json` `pretty_mode-low_case`)

| ABAP component | Expected JSON key |
|----------------|-------------------|
| `SHIP_TO_GSTIN` | `shipToGSTIN` (verify casing in UAT) |
| `SHIP_TO_TRADE_NAME` | `shipToTradeName` |

---

## 4. Configuration change — `ZLOG_EXEC_VAR`

### 4.1 Param: `ZSCE_EWB_API_SHIPTO_GSTIN`

| MANDT | NAME | NUMB | REMARKS | ACTIVE |
|-------|------|------|---------|--------|
| *client* | ZSCE_EWB_API_SHIPTO_GSTIN | 1 | 2 | X |
| *client* | ZSCE_EWB_API_SHIPTO_GSTIN | 2 | 4 | X |

### 4.2 Param: `ZSCE_EWB_API_SHIPTO_TRDNAME`

| MANDT | NAME | NUMB | REMARKS | ACTIVE |
|-------|------|------|---------|--------|
| *client* | ZSCE_EWB_API_SHIPTO_TRDNAME | 1 | 2 | X |
| *client* | ZSCE_EWB_API_SHIPTO_TRDNAME | 2 | 4 | X |

---

## 5. ABAP change — `ZCL_EWB_API` / `GENERATE_E_WAY_BILL`

**ADT path:** `ZCL_EWB_API` → Methods → `GENERATE_E_WAY_BILL`

---

### 5.1 PATCH A — New declarations (after Hiral constants ~line 2084)

**Locate:**

```abap
  CONSTANTS : c_name(20) TYPE c VALUE 'ZSCM_EWB_NON_IRN_UAT ',
              c_name1(20) TYPE c VALUE 'ZSCM_EWB_DISP_GSTIN',
              c_name2(20) TYPE c VALUE 'ZSCM_EWB_SHIP_GSTIN',
              c_name3(20) TYPE c VALUE 'ZSCM_EWB_ERR_CODE'.
```

**Add immediately after:**

```abap
  CONSTANTS: lc_shipto_gstin_trantype   TYPE rvari_vnam VALUE 'ZSCE_EWB_API_SHIPTO_GSTIN',
             lc_shipto_trdname_trantype TYPE rvari_vnam VALUE 'ZSCE_EWB_API_SHIPTO_TRDNAME'.

  TYPES: BEGIN OF lty_shipto_trantype,
           remarks TYPE textr,
         END OF lty_shipto_trantype.

  DATA: lt_shipto_gstin_trantype   TYPE STANDARD TABLE OF lty_shipto_trantype,
        lt_shipto_trdname_trantype TYPE STANDARD TABLE OF lty_shipto_trantype,
        lv_ship_gstin_active       TYPE abap_bool,
        lv_ship_trdname_active     TYPE abap_bool.
```

---

### 5.2 PATCH B — Replace legacy header mapping (~lines 2459–2462)

**REMOVE (delta — old code):**

```diff
         lw_header-transactiontype = lw_input_data-trans_ind.
         lw_header-subsupplydesc = lw_input_data-sub_supl_desc.
-        lw_header-df_gstin = lw_input_data-df_gstin.
-        lw_header-sh_gstn  = lw_input_data-sh_gstn.
```

**ADD (delta — new code):**

```diff
         lw_header-transactiontype = lw_input_data-trans_ind.
         lw_header-subsupplydesc = lw_input_data-sub_supl_desc.

+* BOC - shipToGSTIN / shipToTradeName (Jio GST schema, Approach 2 - separate params)
+*       Omit from JSON when param does not match (do not assign field)
+        CLEAR: lv_ship_gstin_active,
+               lv_ship_trdname_active,
+               lw_header-ship_to_gstin,
+               lw_header-ship_to_trade_name,
+               lw_header-df_gstin,
+               lw_header-sh_gstn.
+
+        REFRESH lt_shipto_gstin_trantype.
+        SELECT remarks
+          FROM zlog_exec_var
+          INTO TABLE lt_shipto_gstin_trantype
+          WHERE name   = lc_shipto_gstin_trantype
+            AND active = abap_true.
+        READ TABLE lt_shipto_gstin_trantype TRANSPORTING NO FIELDS
+          WITH KEY remarks = lw_header-transactiontype.
+        IF sy-subrc = 0.
+          lv_ship_gstin_active = abap_true.
+          lw_header-ship_to_gstin = lw_input_data-sh_gstn.
+        ENDIF.
+
+        REFRESH lt_shipto_trdname_trantype.
+        SELECT remarks
+          FROM zlog_exec_var
+          INTO TABLE lt_shipto_trdname_trantype
+          WHERE name   = lc_shipto_trdname_trantype
+            AND active = abap_true.
+        READ TABLE lt_shipto_trdname_trantype TRANSPORTING NO FIELDS
+          WITH KEY remarks = lw_header-transactiontype.
+        IF sy-subrc = 0.
+          lv_ship_trdname_active = abap_true.
+          lw_header-ship_to_trade_name = lw_input_data-to_other_party_name.
+        ENDIF.
+
+*       lw_header-df_gstin = lw_input_data-df_gstin. " Legacy - not in Jio GST API schema
+*       lw_header-sh_gstn  = lw_input_data-sh_gstn.  " Legacy - not in Jio GST API schema
+* EOC - shipToGSTIN / shipToTradeName
```

---

### 5.3 PATCH C — Hiral UAT block (~lines 2667–2680)

**REMOVE (delta — old code):**

```diff
             IF lw_header-transactiontype = '1' AND LW_HEADER-ACTTOSTATECODE = LW_HEADER-TOSTATECODE.
               lw_header-fromgstin = lw_log_exec_var1-remarks.
               lw_header-togstin = lw_log_exec_var1-remarks.
-              lw_header-df_gstin = lw_log_exec_var1-remarks.
-              lw_header-sh_gstn  = lw_log_exec_var1-remarks.
+              IF lv_ship_gstin_active = abap_true.
+                lw_header-ship_to_gstin = lw_log_exec_var1-remarks.
+              ENDIF.
             ELSEIF lw_header-transactiontype = '2' OR lw_header-transactiontype = '3'.
               lw_header-fromgstin = lw_log_exec_var1-remarks.
-              lw_header-df_gstin = lw_log_exec_var1-remarks.
               CLEAR : lw_log_exec_var2.
               READ TABLE lt_log_exec_var INTO lw_log_exec_var2 WITH  KEY  name = c_name2.
               IF sy-subrc = 0.
                 lw_header-togstin = lw_log_exec_var2-remarks.
-                lw_header-sh_gstn  = lw_log_exec_var2-remarks.
+                IF lv_ship_gstin_active = abap_true.
+                  lw_header-ship_to_gstin = lw_log_exec_var2-remarks.
+                ENDIF.
               ENDIF.
             ENDIF.
```

**Full replacement block (for copy-paste):**

```abap
            IF lw_header-transactiontype = '1' AND lw_header-acttostatecode = lw_header-tostatecode.
              lw_header-fromgstin = lw_log_exec_var1-remarks.
              lw_header-togstin   = lw_log_exec_var1-remarks.
*             lw_header-df_gstin = lw_log_exec_var1-remarks. " Legacy - not in Jio GST API schema
*             lw_header-sh_gstn  = lw_log_exec_var1-remarks.  " Legacy - not in Jio GST API schema
              IF lv_ship_gstin_active = abap_true.
                lw_header-ship_to_gstin = lw_log_exec_var1-remarks.
              ENDIF.
            ELSEIF lw_header-transactiontype = '2' OR lw_header-transactiontype = '3'.
              lw_header-fromgstin = lw_log_exec_var1-remarks.
*             lw_header-df_gstin = lw_log_exec_var1-remarks. " Legacy - not in Jio GST API schema
              CLEAR lw_log_exec_var2.
              READ TABLE lt_log_exec_var INTO lw_log_exec_var2 WITH KEY name = c_name2.
              IF sy-subrc = 0.
                lw_header-togstin = lw_log_exec_var2-remarks.
*               lw_header-sh_gstn = lw_log_exec_var2-remarks. " Legacy - not in Jio GST API schema
                IF lv_ship_gstin_active = abap_true.
                  lw_header-ship_to_gstin = lw_log_exec_var2-remarks.
                ENDIF.
              ENDIF.
            ENDIF.
```

---

### 5.4 PATCH D — JSON serialize (~lines 2689–2697)

**REMOVE (delta — old code):**

```diff
         CLEAR lv_json.
         zui_cl_json=>serialize(
           EXPORTING
             data             =  lw_header
-            compress         =  abap_false
+            compress         =  abap_true
             pretty_name      =  zui_cl_json=>pretty_mode-low_case
           RECEIVING
             r_json           =  lv_json
           ).
```

**ADD before serialize:**

```diff
+* Ensure legacy fields omitted from Jio GST API payload
+        CLEAR: lw_header-df_gstin,
+               lw_header-sh_gstn.
+
         CLEAR lv_json.
```

**ADD after serialize (optional — UAT casing fix):**

```abap
        REPLACE ALL OCCURRENCES OF '"shipToGstin"' IN lv_json WITH '"shipToGSTIN"'.
```

---

## 6. Complete delta summary

| Location | Lines (approx.) | Action |
|----------|-----------------|--------|
| Declarations | ~2084 | ADD constants, types, data |
| Header mapping | ~2459–2462 | REPLACE — param logic + comment legacy |
| Hiral UAT block | ~2667–2680 | REPLACE — comment legacy, ship_to_gstin override |
| Pre-serialize | ~2689 | ADD CLEAR legacy fields |
| Serialize | ~2694 | CHANGE `compress` to `abap_true` |
| Post-serialize | ~2698 | ADD optional JSON casing replace |

---

## 7. Objects not changed

| Object | Reason |
|--------|--------|
| `Z_SCE_EWB_INV_DTL_FETCH` | Data already fetched |
| `Z_SCE_EWB_BOT_CREATE_OB` | Mapping already provides `sh_gstn`, `to_other_party_name` |
| `ZSCM_EWAY_REST_API_V3` | Not used for generate payload |
| `SUBMIT_REQUEST` | Generic HTTP — no change |
| `ZCO_IEWAY_BILL_SERVICE` | BOT inactive |

---

## 8. Syntax check and test checklist

| # | Check | Expected |
|---|-------|----------|
| 1 | Activate `ZST_EWAY_BILL_GENERATION_REQ` | No errors |
| 2 | Activate `ZCL_EWB_API` | No syntax errors |
| 3 | EWB create — trans type 2 | `shipToGSTIN` + `shipToTradeName` in decrypted JSON |
| 4 | EWB create — trans type 1 | Both fields **absent** |
| 5 | Any trans type | No `dfGstin` / `shGstn` keys |
| 6 | UAT param active, trans type 2 | `shipToGSTIN` = override value |
| 7 | GSTIN param only for type 4 | Only `shipToGSTIN` present |

---

## 9. Rollback plan

| Step | Action |
|------|--------|
| 1 | Revert transport for `ZCL_EWB_API` |
| 2 | Revert `ZST_EWAY_BILL_GENERATION_REQ` (or leave new fields unused) |
| 3 | Deactivate / delete new `ZLOG_EXEC_VAR` rows |

---

## 10. Transport request details

| Field | Value |
|-------|-------|
| Transport request | *(to be assigned by ABAP team)* |
| Package | `ZLOG` |
| Workbench objects | `ZST_EWAY_BILL_GENERATION_REQ`, `ZCL_EWB_API` |
| Customizing | `ZLOG_EXEC_VAR` entries (manual / BC Set if used) |

---

*End of Code Change Document*
