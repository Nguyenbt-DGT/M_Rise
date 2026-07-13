# [M-RISE] AD/PD - US3 for 13 API | Order 3 – Sync License Eligibility, Grant & Plan Code Information from M-Rise to AMS via Learning Adapter

| Field | Value |
|---|---|
| **Issue Key** | TBD |
| **Type** | Story |
| **Project** | VN M-RISE (VNEL) |
| **Status** | TODO |
| **Priority** | Medium |
| **Assignee** | TBD |
| **Reporter** | TBD |
| **Created** | TBD |
| **Updated** | 2026-07-13 |

---

## 1. User Story Summary

| Item | Description |
|---|---|
| **User Story** | As **AMS System**, I want to receive license-eligibility check, license grant, and plan-code assignment/refresh information from **M-Rise** through **Learning Adapter**, so that AMS can determine whether a candidate/agent qualifies for a license, record the granted **education/license background**, and maintain the agent's **product plan code** entries for downstream e-certificate processing. |
| **Actor** | AMS System, Learning Adapter, M-Rise |
| **Flow** | M-Rise → Learning Adapter → AMS |
| **Screen Demo** | N/A – Backend Integration |
| **Main Objective** | Sync Order 3 license/plan-code data from M-Rise to AMS, using the training-result and education-background data that Order 2 already populated, and prepare data for Order 4 (e-certificate) processing. |

---

## 2. Trigger / Pre-condition / Post-condition

| Type | Description |
|---|---|
| **Trigger** | M-Rise (or Learning Adapter, on M-Rise's behalf) needs to check whether a candidate/agent meets license-grant conditions · A candidate/agent's training/exam result qualifies them for a license and M-Rise finalizes the grant · An agent's product plan code needs to be assigned (new) or refreshed (expiry/status update) on M-Rise · MIT candidate license/education background needs to be updated on M-Rise |
| **Pre-condition** | Order 2 data already exists in AMS where applicable: education background (TAMS_EDUCATION_BACKGROUNDS), candidate/agent master data (tams_agents / tams_candidates) · For `plancode/refresh`: the target (AGT_CD, PLAN_CD) row must already exist in TAMS_AGT_PRODUCT_DTLS, since this API only updates, it does not insert · Learning Adapter service is running · AMS API endpoints are ready to receive requests |
| **Post-condition** | AMS reflects the candidate/agent's license-eligibility flags per education type (UL, ULC, PENSION, etc.) · AMS records the granted license/education background in TAMS_EDUCATION_BACKGROUNDS, updates related AWS e-certificate housekeeping in TWRK_AGT_CERTIFICATES, and conditionally updates tams_agents.GRP_PRD and tams_candidates.MOF_APRV_NUM/MOF_APRV_DT · AMS's TAMS_AGT_PRODUCT_DTLS reflects newly assigned or refreshed plan codes for the agent, with no duplicate rows created on re-sync · Data is ready for Order 4 e-certificate processing · Log is recorded for each API call |

---

## 3. Integration Flow

> **Important Note:**
> M-Rise **does not call AMS directly**.
> All Order 3 data must go through **Learning Adapter**.

---

## 4. Scope

### 4.1 In Scope

| No. | API | Purpose | Trigger |
|---|---|---|---|
| 1 | POST /api/v1/education/conditions/license | Check whether a candidate/agent meets license-grant conditions per education type | Before license grant/plan-code decision is made on M-Rise |
| 2 | POST /api/v1/education/plancode/refresh | Refresh (update expiry month / status) an agent's existing product plan code | When an existing agent-plan-code record's expiry or status needs to be updated on M-Rise |
| 3 | POST /api/v1/education/plancode/assign | Assign a new product plan code to an agent | When a new agent-plan-code assignment is created on M-Rise |
| 4 | POST /api/v1/education/license/grant | Grant a license / education background to a candidate, with related AWS e-certificate and MOF approval housekeeping | When a candidate's license is finalized/granted on M-Rise |
| 5 | POST /api/v1/education/MIT-license/update | Insert an MIT candidate's license/education background record | When an MIT candidate's license/education background is updated on M-Rise |

`conditions/license` is included in Order 3 (rather than Order 2) because it is the eligibility pre-check that feeds directly into this Order's `license/grant` decision — it reads Order-2-populated data but its business purpose belongs to the grant flow, not to attendance/training-result sync.

### 4.2 Out of Scope

| No. | Item | Belong To |
|---|---|---|
| 1 | Sync class template / exam info / class info | Order 1 |
| 2 | Sync attendance result / training result / inform CPA result | Order 2 |
| 3 | Upload / retrieve e-certificate | Order 4 |

---

## 5. Requirement Details

### R.01 – Check License Grant Conditions

| Item | Description |
|---|---|
| **API** | POST /api/v1/education/conditions/license |
| **API Meaning** | Check/query whether a candidate/agent meets the conditions required to be granted a license, per education type |
| **Trigger** | M-Rise (or Learning Adapter) needs to verify license-grant eligibility before proceeding with `license/grant` or plan-code processing |
| **Purpose** | Return eligibility flags (e.g. UL, ULC, PENSION) so M-Rise/downstream logic can decide whether to grant a license for each education type |
| **Method** | POST |
| **Request Type** | Check/query request (despite POST verb, this API is read-only — it does not insert or update AMS data) |
| **Target Tables** | N/A for write — read/reference only against TAMS_EDUCATION_BACKGROUNDS, tams_agents |
| **Main Data Object Table** | TAMS_EDUCATION_BACKGROUNDS (read-only reference for eligibility check) |

#### Main Data Object

| Field | Required | Description |
|---|---|---|
| agentId | No | Agent ID (source-side identifier; no AMS target) |
| candidateCode | No | Candidate code |
| agentCode | No | Agent code |
| agentCompensationProvision | No | List of compensation provisions |
| UL | No | Flag/value used to check UL license condition |
| ULC | No | Flag/value used to check ULC license condition |

#### Sample Payload

```json
{
  "agentId": 1001,
  "candidateCode": "2801182011",
  "agentCode": "BF345",
  "agentCompensationProvision": "01",
  "UL": 1,
  "ULC": 0
}
```

#### Sample Response Payload

```json
[
  {
    "agentId": 1001,
    "candidateCode": "2801182011",
    "agentCode": "BF345",
    "agentCompensationProvision": ["01"],
    "mapEducationTypeCodeWithPermissionGrantLicense": {
      "UL": true,
      "ULC": false,
      "PENSION": true
    }
  }
]
```

#### Field Mapping

| Source Field | Target Table | Target Field |
|---|---|---|
| agentId | N/A | N/A |
| candidateCode | tams_agents | CAN_NUM |
| agentCode | TAMS_EDUCATION_BACKGROUNDS | AGT_CODE |
| agentCompensationProvision | N/A | N/A |
| UL | TAMS_EDUCATION_BACKGROUNDS | EDU_TYP |
| ULC | TAMS_EDUCATION_BACKGROUNDS | EDU_TYP |

> The source sheet (`order3_sheets.txt`, Sheet 2) also contains a second field-mapping block (rows 11–15) referencing `twrk_agt_certificates` (agentCode/licNo/licType/docId → AGT_CD/LIC_NO/LIC_TYP/DOC_ID). BA cross-checked this against Order 4's `eCertificate/save` sheet, which targets the same table with the same fields — confirming the block is copy-paste contamination from that sheet, not part of `conditions/license`'s actual behavior. This story does not model any write to `twrk_agt_certificates` for `conditions/license`; it is built as a pure read/check API.

#### Logic Summary

| Scenario | Expected Logic |
|---|---|
| Request received | Look up the candidate/agent's existing TAMS_EDUCATION_BACKGROUNDS record(s) and related compensation-provision data. For each education type (UL, ULC, PENSION, etc.), evaluate whether license-grant conditions are met and return `true`/`false` in `mapEducationTypeCodeWithPermissionGrantLicense`. |

> Note: this is a check/validation API, not a write API. No INS/UPD/DEL action applies — do not build insert/update logic against TAMS_EDUCATION_BACKGROUNDS for this endpoint.

#### Validation / Error Handling

| Scenario | Expected Behavior | Source Status |
|---|---|---|
| Missing candidateCode/agentCode | Reject request | Proposed — not explicitly specified in source; recommended default, confirm in SIT |
| No matching education background record found | Return eligibility map with all flags `false` (or empty map) rather than erroring | Proposed — inferred from the API's check-style purpose; confirm exact behavior with BE |

---

### R.02 – Refresh Plan Code

| Item | Description |
|---|---|
| **API** | POST /api/v1/education/plancode/refresh |
| **API Meaning** | Update the expiry month (and status) of an agent's existing product plan code record |
| **Trigger** | An agent's existing plan-code record needs its expiry/status refreshed on M-Rise |
| **Purpose** | Keep AMS's agent-plan-code expiry/status current so downstream e-certificate/eligibility logic uses the right plan-code state |
| **Method** | POST |
| **Request Type** | List data request |
| **Target Table** | TAMS_AGT_PRODUCT_DTLS |
| **Main Data Object Table** | TAMS_AGT_PRODUCT_DTLS |

#### Main Data Object

| Field | Required | Description |
|---|---|---|
| agentCode | Yes | Agent code |
| planCode | Yes | Product/plan code |
| expiredMonth | No | New expiry month of the plan code |

#### Sample Payload

```json
[
  {
    "agentCode": "WJ878",
    "planCode": "RUV10",
    "expiredMonth": "202212"
  }
]
```

#### Field Mapping

| Source Field | Target Table | Target Field |
|---|---|---|
| agentCode | TAMS_AGT_PRODUCT_DTLS | AGT_CD |
| planCode | TAMS_AGT_PRODUCT_DTLS | PLAN_CD |
| expiredMonth | TAMS_AGT_PRODUCT_DTLS | EXPR_MTH |

#### Logic Summary

| Action | Expected Logic |
|---|---|
| UPD only | `UPDATE TAMS_AGT_PRODUCT_DTLS SET EXPR_MTH = ..., STAT_CD = ... WHERE AGT_CD = ... AND PLAN_CD = ...` (per the cross-referenced "API infor" logic tab). This API only updates an existing agent-plan-code record; it does **not** insert a new one if the (AGT_CD, PLAN_CD) key does not already exist. |

> **Working assumption — STAT_CD is server-derived (pending confirmation):** the underlying update logic sets `STAT_CD`, but the request schema only documents 3 input fields (`agentCode`, `planCode`, `expiredMonth`) — no `statusCode` input exists in source. Build default: AMS sets `STAT_CD` to a fixed/derived value (e.g. `'A'`/active) as part of the refresh, independent of client input, rather than expecting an undocumented request field. This must be confirmed with the M-Rise API owner in SIT; if it turns out `statusCode` is a real, missing request field, the request schema will need a follow-up change.

#### Validation / Error Handling

| Scenario | Expected Behavior | Source Status |
|---|---|---|
| Missing agentCode or planCode | Reject request | Proposed — not explicitly specified in source; recommended default, confirm in SIT |
| (AGT_CD, PLAN_CD) does not exist in TAMS_AGT_PRODUCT_DTLS | Reject / return not-found error; do not insert a new row (update-only per the cross-referenced SQL) | Proposed — inferred from the update-only SQL logic; confirm exact error contract with BE |

---

### R.03 – Assign Plan Code

| Item | Description |
|---|---|
| **API** | POST /api/v1/education/plancode/assign |
| **API Meaning** | Create a new agent-plan-code assignment record |
| **Trigger** | A new product plan code is assigned to an agent on M-Rise |
| **Purpose** | Record a new agent-plan-code assignment in AMS so downstream e-certificate/eligibility logic recognizes the agent's plan code |
| **Method** | POST |
| **Request Type** | List data request |
| **Target Table** | TAMS_AGT_PRODUCT_DTLS |
| **Main Data Object Table** | TAMS_AGT_PRODUCT_DTLS |

#### Main Data Object

| Field | Required | Description |
|---|---|---|
| agentCode | Yes | Agent code |
| classCode | No | Class code |
| planCode | Yes | Product/plan code |
| effectiveDate | No | Effective date |
| transactionDate | No | Transaction date |
| expiredMonth | No | Expiry month |
| userId | No | Operator/user ID |
| trainingType | No | Training type (e.g. OFFLINE) |
| statusCode | No | Status |

#### Sample Payload

```json
[
  {
    "agentCode": "E0593",
    "classCode": "A0NP100068",
    "planCode": "NPE11",
    "effectiveDate": "2017-08-01",
    "transactionDate": "2017-08-01",
    "expiredMonth": "202303",
    "userId": "NTNNHI",
    "trainingType": "OFFLINE",
    "statusCode": "A"
  }
]
```

#### Sample Response Payload

```json
[
  {
    "identifier": "BF345",
    "classCode": "CLASS001",
    "planCode": "PLAN01",
    "errorCode": "SUCCESS"
  }
]
```

#### Field Mapping

| Source Field | Target Table | Target Field |
|---|---|---|
| agentCode | TAMS_AGT_PRODUCT_DTLS | AGT_CD |
| classCode | TAMS_AGT_PRODUCT_DTLS | CLS_NUM |
| planCode | TAMS_AGT_PRODUCT_DTLS | PLAN_CD |
| effectiveDate | TAMS_AGT_PRODUCT_DTLS | EFF_DT |
| transactionDate | TAMS_AGT_PRODUCT_DTLS | TRXN_DT |
| expiredMonth | TAMS_AGT_PRODUCT_DTLS | EXPR_MTH |
| userId | TAMS_AGT_PRODUCT_DTLS | USER_ID |
| trainingType | TAMS_AGT_PRODUCT_DTLS | TRAIN_TYP |
| statusCode | TAMS_AGT_PRODUCT_DTLS | STAT_CD |

#### Logic Summary

| Action | Expected Logic |
|---|---|
| INS with existence check (per BR-012) | Before inserting, check whether a row already exists for the key (AGT_CD, PLAN_CD) in TAMS_AGT_PRODUCT_DTLS. If it does not exist, `INSERT INTO TAMS_AGT_PRODUCT_DTLS (AGT_CD, CLS_NUM, PLAN_CD, EFF_DT, TRXN_DT, EXPR_MTH, USER_ID, TRAIN_TYP, STAT_CD, ...)`. If it already exists (e.g. a retried/re-sent request), do not insert a duplicate row — treat as already-applied and return success, or update in place; exact idempotent-response contract to be confirmed with BE during dev. The source SQL as sheeted is a bare INSERT with no such check — this existence check is a PO-mandated addition (BR-012), not a literal transcription of the source. |

#### Validation / Error Handling

| Scenario | Expected Behavior | Source Status |
|---|---|---|
| Missing agentCode or planCode | Reject request | Proposed — not explicitly specified in source; recommended default, confirm in SIT |
| Duplicate (agentCode, planCode) re-sync | Existence check blocks duplicate insert (see BR-012); do not create a duplicate row | PO-mandated default — build requirement, not optional |

---

### R.04 – Grant License

| Item | Description |
|---|---|
| **API** | POST /api/v1/education/license/grant |
| **API Meaning** | Grant (insert or update) a candidate's license/education background, with related AWS e-certificate cleanup and conditional agent product-code / MOF-approval updates |
| **Trigger** | A candidate's license is finalized/granted on M-Rise (e.g. after passing required exams/training and meeting conditions returned by `conditions/license`) |
| **Purpose** | Record the granted license/education background in AMS, keep AWS e-certificate records consistent, and apply related agent/candidate updates (product group code, MOF approval decision) |
| **Method** | POST |
| **Request Type** | List data request |
| **Target Tables** | TAMS_EDUCATION_BACKGROUNDS, TWRK_AGT_CERTIFICATES (delete), tams_agents (conditional), tams_candidates (conditional) |
| **Main Data Object Table** | TAMS_EDUCATION_BACKGROUNDS |

#### Main Data Object

| Field | Required | Description |
|---|---|---|
| candidateNumber | Yes | Candidate number |
| educationType | Yes | Education/license type (e.g. MIT, ULC, BHNTCB) |
| educationRemark | No | License number / remark |
| licenseDate | No | License issue date |
| educationEndDate | No | License/education end date |
| statusCode | No | Status |
| agentCode | No | Agent code — used only to delete the corresponding AWS e-certificate record, not to write TAMS_EDUCATION_BACKGROUNDS |
| productCode | No | Group product code — applied to tams_agents.GRP_PRD only if isUpdateProductCode = 1 |
| isUpdateProductCode | No | Flag: if = 1, update tams_agents.GRP_PRD with productCode |

#### Sample Payload

```json
{
  "candidateNumber": "2801735343",
  "educationType": "BHNTCB",
  "educationRemark": "CĐ 1.103.0004203",
  "licenseDate": "2024-04-01T00:00:00Z",
  "educationEndDate": "2024-04-01T00:00:00Z",
  "statusCode": "A",
  "agentCode": "A001",
  "productCode": 31,
  "isUpdateProductCode": 1
}
```

#### Sample Response Payload

```json
[
  {
    "identifier": "LIC001",
    "errorCode": "SUCCESS"
  }
]
```

#### Field Mapping

| Source Field | Target Table | Target Field |
|---|---|---|
| candidateNumber | TAMS_EDUCATION_BACKGROUNDS | CAN_NUM |
| educationType | TAMS_EDUCATION_BACKGROUNDS | EDU_TYP |
| educationRemark | TAMS_EDUCATION_BACKGROUNDS | EDU_RMRK |
| licenseDate | TAMS_EDUCATION_BACKGROUNDS | LICENCE_DT |
| educationEndDate | TAMS_EDUCATION_BACKGROUNDS | EDU_END_DT |
| statusCode | TAMS_EDUCATION_BACKGROUNDS | STAT_CD |
| agentCode | (Logic delete — TWRK_AGT_CERTIFICATES) | AGT_CD |
| productCode | tams_agents | GRP_PRD (conditional on isUpdateProductCode = 1) |
| isUpdateProductCode | N/A (control flag only) | N/A |

#### Logic Summary

| Step | Condition | Expected Logic |
|---|---|---|
| 1 | Always | Update-or-insert into TAMS_EDUCATION_BACKGROUNDS keyed by (CAN_NUM, EDU_TYP): update the existing row if the key exists, otherwise insert a new row (per the cross-referenced "API infor" logic tab). |
| 2 | educationType is MIT/BHNTCB-related or ULC, and `certCateId` = 1 (per cross-referenced logic) | Before writing the new license, copy/back up the existing TAMS_EDUCATION_BACKGROUNDS record to an archived record with EDU_TYP suffixed `_MVL` (moved/versioned), preserving history, then write the new license data into the primary EDU_TYP record. Sourced from the previously-reviewed "API infor" logic-summary tab, not directly visible in the raw sheet dump — BE to re-confirm against the source stored procedure during dev. |
| 3 | agentCode present | Delete the corresponding AWS e-certificate record(s) for that agent in TWRK_AGT_CERTIFICATES (the sheet marks this as "(Logic delete)" against AGT_CD — used only to identify which record to delete, not to write TAMS_EDUCATION_BACKGROUNDS). |
| 4 | isUpdateProductCode = 1 | Update tams_agents.GRP_PRD with productCode. If isUpdateProductCode ≠ 1 (or absent), do not touch tams_agents.GRP_PRD. |
| 5 | certDecNum and/or certDecDate given (per cross-referenced "API infor" logic; not present as explicit fields in this sheet's Main Data Object) | Conditionally update tams_candidates.MOF_APRV_NUM / MOF_APRV_DT, independent of steps 1–4, mirroring the same pattern as Order 2's `training-result/sync` (US2 R.02, step 5). |

> Note: this API does not carry an explicit `action` (INS/UPD/DEL) field; TAMS_EDUCATION_BACKGROUNDS insert-vs-update is decided by (CAN_NUM, EDU_TYP) existence, per Step 1.

#### Validation / Error Handling

| Scenario | Expected Behavior | Source Status |
|---|---|---|
| Missing candidateNumber or educationType | Reject request | Proposed — not explicitly specified in source; recommended default, confirm in SIT |
| Grant fails for any reason (e.g. downstream write failure) | Return `errorCode: FAILED` in the response payload rather than raising an HTTP-level exception — **soft failure via response code**, unlike Order 1/2's hard-reject error pattern | Sourced (`errorCode` field in response payload) |
| Grant succeeds | Return `errorCode: SUCCESS` with `identifier` (license/record reference) | Sourced |

#### Special Logic

| Scenario | Expected Behavior |
|---|---|
| MIT ↔ BHNTCB and ULC license types with certCateId = 1 | Old certificate record is archived under an `_MVL`-suffixed EDU_TYP before the new one is written (backup/versioning), per the cross-referenced "API infor" logic tab. |
| Soft-failure response pattern | Unlike other Order 3 APIs and Order 1/2 APIs generally, `license/grant` reports both success and failure via `errorCode` in a 200-style response body rather than an HTTP error — Learning Adapter/AMS logging must treat `errorCode = FAILED` as a failure case for logging/alerting purposes even though no exception is thrown. |

---

### R.05 – Update MIT License

| Item | Description |
|---|---|
| **API** | POST /api/v1/education/MIT-license/update |
| **API Meaning** | Insert an MIT candidate's license/education background record |
| **Trigger** | An MIT candidate's license/education background is updated/finalized on M-Rise |
| **Purpose** | Record the MIT candidate's license/education background (and related MOF approval fields) in AMS |
| **Method** | POST |
| **Request Type** | List data request |
| **Target Tables** | TAMS_EDUCATION_BACKGROUNDS, tams_candidates |
| **Main Data Object Table** | TAMS_EDUCATION_BACKGROUNDS |

#### Main Data Object

| Field | Required | Description |
|---|---|---|
| canNum | Yes | Candidate number (key) |
| eduTyp | No | Education/license type |
| eduEndDt | No | Education/license end date |
| licenceDt | No | License issue date |
| statCd | No | Status |
| eduRmk | No | Remarks / license number |
| certDecNum | No | MOF approval decision number |
| certDecDate | No | MOF approval decision date |
| certCateId | No | Certificate category/type ID (branch/condition value only — not written to AMS) |

#### Sample Payload

```json
{
  "canNum": "2800038144",
  "eduTyp": "ULC",
  "eduEndDt": "2025-08-01",
  "licenceDt": "2025-08-01",
  "statCd": "A",
  "eduRmk": "CĐ 2.103.0000248",
  "certDecNum": null,
  "certDecDate": "QD001",
  "certCateId": 46201
}
```

#### Field Mapping

| Source Field | Target Table | Target Field |
|---|---|---|
| canNum | TAMS_EDUCATION_BACKGROUNDS | CAN_NUM |
| eduTyp | TAMS_EDUCATION_BACKGROUNDS | EDU_TYP |
| eduEndDt | TAMS_EDUCATION_BACKGROUNDS | EDU_END_DT |
| licenceDt | TAMS_EDUCATION_BACKGROUNDS | LICENCE_DT |
| statCd | TAMS_EDUCATION_BACKGROUNDS | STAT_CD |
| eduRmk | TAMS_EDUCATION_BACKGROUNDS | EDU_RMRK |
| certDecNum | tams_candidates | MOF_APRV_NUM |
| certDecDate | tams_candidates | MOF_APRV_DT |
| certCateId | N/A | N/A (unmapped — condition/branch value only, consistent with `license/grant`'s use of `certCateId = 1` as a branch condition, R.04 Step 2) |

> **PO decision: build against the corrected mapping above, not the literal-as-sourced sheet.** `order3_sheets.txt` Sheet 14, rows 9–11, as literally transcribed, show `certDecNum → N/A`, `certDecDate → tams_candidates.MOF_APRV_NUM`, `certCateId → tams_candidates.MOF_APRV_DT` — a mapping that is internally inconsistent with its own sample values and Vietnamese description column (row 9 described as decision *number*, row 10 as decision *date*, row 11 as certificate *category*), and is almost certainly a one-row shift/data-entry error in the source workbook. BA and BE both independently converged on the same corrected reading, which also mirrors `license/grant`'s clean, unambiguous pattern for the identical target fields (R.04 Step 5: `certDecNum → MOF_APRV_NUM`, `certDecDate → MOF_APRV_DT`). Given these are MOF (financial regulator) approval fields on `tams_candidates` — a compliance-sensitive write — building against the literal, self-contradictory mapping would risk writing approval data to the wrong meaning/field with no compensating control. The corrected mapping is adopted as the build target. **Still log a follow-up to confirm with the workbook owner** during SIT; if the workbook owner confirms the literal mapping was in fact intentional (unlikely given the internal inconsistency), this will need a change request against this story.

#### Logic Summary

| Action | Expected Logic |
|---|---|
| INS with existence check (per BR-012) | Before inserting, check whether a row already exists for the key (CAN_NUM, EDU_TYP) in TAMS_EDUCATION_BACKGROUNDS. If it does not exist, `INSERT INTO TAMS_EDUCATION_BACKGROUNDS (CAN_NUM, EDU_TYP, EDU_END_DT, LICENCE_DT, STAT_CD, EDU_RMRK, ...)`, then conditionally update `tams_candidates.MOF_APRV_NUM` / `MOF_APRV_DT` per the corrected mapping above. If the key already exists (e.g. a retried/re-sent request), do not insert a duplicate row — treat as already-applied and return success, or update in place; exact idempotent-response contract to be confirmed with BE during dev. The source SQL as sheeted is a bare INSERT with no such check — this existence check is a PO-mandated addition (BR-012), same as R.03's `plancode/assign`. |

> Note: this API does not carry an explicit `action` (INS/UPD/DEL) field.

#### Validation / Error Handling

| Scenario | Expected Behavior | Source Status |
|---|---|---|
| Missing canNum | Reject request | Proposed — not explicitly specified in source; recommended default, confirm in SIT |
| Duplicate (canNum, eduTyp) re-sync | Existence check blocks duplicate insert (see BR-012); do not create a duplicate row | PO-mandated default — build requirement, not optional |
| Success | Return `errorCode: SUCCESS` with `identifier` (candidate number) | Sourced |

---

## 6. Business Rules & Validations

| Rule ID | Rule Name | Business Description | Impacted Flow | Example Scenario |
|---|---|---|---|---|
| BR-001 | License-eligibility check is read-only | `conditions/license` never writes to AMS; it only reads TAMS_EDUCATION_BACKGROUNDS-related data and returns an eligibility map. Refer AC1. | conditions/license API | UL condition met → mapEducationTypeCodeWithPermissionGrantLicense.UL = true, no DB write |
| BR-002 | Plan-code refresh is update-only | `plancode/refresh` only updates an existing TAMS_AGT_PRODUCT_DTLS row keyed by (AGT_CD, PLAN_CD); it never inserts. Refer AC2. | plancode/refresh API | (AGT_CD, PLAN_CD) not found → reject, no new row created |
| BR-003 | Plan-code assignment is insert-with-existence-check | `plancode/assign` inserts a new TAMS_AGT_PRODUCT_DTLS row only if (AGT_CD, PLAN_CD) does not already exist — see BR-012 for the mandated existence-check requirement. Refer AC3. | plancode/assign API | Same (agentCode, planCode) re-sent → existence check blocks duplicate insert |
| BR-004 | License-grant is existence-key-based (no action field) | `license/grant` writes TAMS_EDUCATION_BACKGROUNDS by (CAN_NUM, EDU_TYP) existence check — insert if absent, update if present. Refer AC4. | license/grant API | (CAN_NUM, EDU_TYP) exists → update instead of insert |
| BR-005 | MIT/ULC certificate versioning on re-grant | When granting a MIT↔BHNTCB or ULC-type license with certCateId = 1, the prior license record is archived to an `_MVL`-suffixed EDU_TYP before the new one is written, preserving history. Refer AC4. | license/grant API | Re-grant BHNTCB license → old record archived as `..._MVL`, new record written |
| BR-006 | AWS certificate cleanup on grant | Granting a license deletes the corresponding AWS e-certificate record(s) in TWRK_AGT_CERTIFICATES for the given agentCode. Refer AC4. | license/grant API | agentCode = A001 → old AWS cert record for A001 deleted |
| BR-007 | Conditional product-code update | tams_agents.GRP_PRD is updated with productCode only when isUpdateProductCode = 1. Refer AC4. | license/grant API | isUpdateProductCode = 0 → GRP_PRD untouched |
| BR-008 | Soft-failure response pattern for license/grant | `license/grant` reports success/failure via `errorCode` (SUCCESS/FAILED) in the response body rather than an HTTP-level exception; Learning Adapter/AMS logging must still treat FAILED as a failure. Refer AC4, AC5. | license/grant API | errorCode = FAILED → logged as failed, no exception thrown |
| BR-009 | MIT-license update is insert-with-existence-check | `MIT-license/update` inserts a new TAMS_EDUCATION_BACKGROUNDS row only if (CAN_NUM, EDU_TYP) does not already exist — see BR-012 for the mandated existence-check requirement. Refer AC5. | MIT-license/update API | Same (canNum, eduTyp) re-sent → existence check blocks duplicate insert |
| BR-010 | Integration via Learning Adapter only | M-Rise does not call AMS directly. All requests must go through Learning Adapter. Refer AC6. | All 5 APIs | M-Rise → Learning Adapter → AMS |
| BR-011 | Logging required | Each API call must have a log for troubleshooting and SIT/UAT evidence. Refer AC7. | All 5 APIs | Log API name, timestamp, status, error message, reference key |
| BR-012 | Insert-only APIs must have a defensive existence check (mandatory) | `plancode/assign` (R.03) and `MIT-license/update` (R.05) are sourced as bare inserts with no documented existence check, unlike every other write API in Order 1–3 (which is either update-only or existence-key-based). **PO decision: this is a hard build requirement, not an optional/deferred item.** Both endpoints must check for existence of the natural key — (AGT_CD, PLAN_CD) and (CAN_NUM, EDU_TYP) respectively — before insert, and must not create a duplicate row when the same request is re-sent (e.g. after a Learning Adapter retry on timeout). This preserves the same no-duplicate-on-resync guarantee already established in Order 1 (BR-002) and Order 2, applied consistently to Order 3. Rationale: these APIs write candidate-facing license/plan-code data with real compliance and downstream (Order 4 e-certificate) consequences if silently duplicated — the cost of a defensive check is small relative to that risk, and "ship as literally sourced" was rejected because a bare insert is not an acceptable default for this kind of data. | plancode/assign API, MIT-license/update API | Same (agentCode, planCode) or (canNum, eduTyp) re-sent on retry → existence check blocks/updates instead of duplicating |

---

## 7. Impact and Risk

| Area | Impact / Risk |
|---|---|
| Backend | AMS needs to expose 5 APIs to receive requests from Learning Adapter, including the added existence checks for `plancode/assign` and `MIT-license/update` (BR-012) |
| Database | Impact on TAMS_EDUCATION_BACKGROUNDS, TAMS_AGT_PRODUCT_DTLS, TWRK_AGT_CERTIFICATES, tams_agents, tams_candidates |
| Reference Tables | Dependency on TAMS_EDUCATION_BACKGROUNDS (populated in Order 2), tams_agents / tams_candidates master data |
| Integration Risk | If Learning Adapter is down, all Order 3 sync will be blocked, stalling Order 4 e-certificate processing |
| Data Integrity Risk | `plancode/assign` and `MIT-license/update` are insert-only per source with no documented existence check — mitigated by BR-012's mandatory existence check; residual risk is limited to the small chance a DB-level unique constraint conflicts with the application-level check (to be confirmed in dev) |
| Data Quality Risk | `MIT-license/update`'s certDecNum/certDecDate/certCateId field mapping in the source sheet appears shifted by one row; this story builds against the corrected mapping (R.05) — residual risk is the workbook-owner confirmation still being outstanding |
| Validation Risk | `plancode/refresh`'s underlying SQL sets STAT_CD but the sheet documents no statusCode input field — building against the working assumption (server-derived STAT_CD) rather than the documented schema alone; to be confirmed in SIT |
| Downstream Impact | If Order 3 sync fails or writes inconsistent data, Order 4 (e-certificate) will not have correct license/plan-code data to process |

---

## 8. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Performance / Throughput | N/A – Event-driven, not large batch |
| Security / Privacy | N/A within the scope of this US |
| Audit / Logging | Each API call must log: API name, request timestamp, processing status, error message if any, reference key. For `license/grant`, log must capture `errorCode` even though it is returned as a 200-style response, not an exception. |
| Availability / Resilience | Learning Adapter must operate stably to ensure successful sync |
| Idempotency | Re-syncing the same data must not create duplicate records, for all 5 APIs. `conditions/license` is trivially idempotent (read-only). `plancode/refresh` and `license/grant` are idempotent by construction (existence-key-based). `plancode/assign` and `MIT-license/update` achieve this via the mandatory existence check in BR-012. |
| Error Traceability | Error messages must be clear enough for BA/Tester to trace failure root cause during SIT/UAT, including distinguishing `license/grant`'s soft-failure (`errorCode = FAILED`) from hard-reject errors on the other 4 APIs |

---

## 9. Dependencies / References

| Dependency | Description |
|---|---|
| M-Rise | Source system where license-eligibility, license-grant, and plan-code data originate |
| Learning Adapter | Middleware that receives data from M-Rise and calls AMS APIs |
| AMS | Target system where license/education-background and plan-code data is stored |
| TAMS_EDUCATION_BACKGROUNDS | Populated in Order 2 (`training-result/sync`); read by `conditions/license` and written by `license/grant` / `MIT-license/update` in Order 3 |
| TAMS_AGT_PRODUCT_DTLS | Written by `plancode/assign` (insert-with-check) and `plancode/refresh` (update) in Order 3 |
| TWRK_AGT_CERTIFICATES | AWS e-certificate table; old records deleted by `license/grant` as part of the grant flow |
| Order 2 | Provides training result / education background / MOF approval data that Order 3 reads and extends |
| Order 4 | Depends on Order 3's granted license and assigned/refreshed plan code data to process e-certificate issuance |

---

## 13. Acceptance Criteria

### AC1 – Check License Grant Conditions Successfully

**Given** M-Rise (or Learning Adapter) needs to verify a candidate/agent's license-grant eligibility

**When** Learning Adapter calls API: `POST /api/v1/education/conditions/license`

**Then** AMS must:
- Receive the request from Learning Adapter.
- Evaluate eligibility per education type against TAMS_EDUCATION_BACKGROUNDS-related data, without writing any data.
- Return `mapEducationTypeCodeWithPermissionGrantLicense` with a true/false flag per education type (e.g. UL, ULC, PENSION).

**And** this API must not insert, update, or delete any AMS record.

### AC2 – Refresh Plan Code Successfully

**Given** an agent's existing plan-code record needs its expiry/status refreshed on M-Rise

**When** Learning Adapter calls API: `POST /api/v1/education/plancode/refresh`

**Then** AMS must:
- Receive agentCode, planCode, expiredMonth from Learning Adapter.
- Locate the existing TAMS_AGT_PRODUCT_DTLS row by (AGT_CD, PLAN_CD).
- Update EXPR_MTH (and STAT_CD, server-derived per the working assumption — see Comments).

**And** if (AGT_CD, PLAN_CD) does not exist, AMS must not insert a new row.

### AC3 – Assign Plan Code Successfully

**Given** a new product plan code is assigned to an agent on M-Rise

**When** Learning Adapter calls API: `POST /api/v1/education/plancode/assign`

**Then** AMS must:
- Receive agentCode, classCode, planCode, effectiveDate, transactionDate, expiredMonth, userId, trainingType, statusCode from Learning Adapter.
- Check whether (AGT_CD, PLAN_CD) already exists in TAMS_AGT_PRODUCT_DTLS.
- Insert a new row only if it does not already exist.
- Return a response echoing identifier (agentCode), classCode, planCode, plus errorCode.

**And** do not create a duplicate row when the same (agentCode, planCode) is re-sent (BR-012).

### AC4 – Grant License Successfully

**Given** a candidate's license is finalized/granted on M-Rise

**When** Learning Adapter calls API: `POST /api/v1/education/license/grant`

**Then** AMS must:
- Receive the request from Learning Adapter.
- Insert or update TAMS_EDUCATION_BACKGROUNDS by key (CAN_NUM, EDU_TYP) using candidateNumber, educationType, educationRemark, licenseDate, educationEndDate, statusCode.
- If the education type/certCateId conditions apply (MIT↔BHNTCB, ULC, certCateId = 1), archive the prior record to an `_MVL`-suffixed EDU_TYP before writing the new one.
- Delete the corresponding AWS e-certificate record(s) in TWRK_AGT_CERTIFICATES for the given agentCode.
- Update tams_agents.GRP_PRD with productCode only if isUpdateProductCode = 1.
- Conditionally update tams_candidates.MOF_APRV_NUM / MOF_APRV_DT if the equivalent of certDecNum/certDecDate is present.
- Return `errorCode: SUCCESS` or `errorCode: FAILED` in the response, with `identifier`.

**And** a FAILED errorCode must still be logged as a failure even though no HTTP exception is raised.

### AC5 – Update MIT License Successfully

**Given** an MIT candidate's license/education background is updated on M-Rise

**When** Learning Adapter calls API: `POST /api/v1/education/MIT-license/update`

**Then** AMS must:
- Receive canNum, eduTyp, eduEndDt, licenceDt, statCd, eduRmk, certDecNum, certDecDate, certCateId from Learning Adapter.
- Check whether (CAN_NUM, EDU_TYP) already exists in TAMS_EDUCATION_BACKGROUNDS.
- Insert a new row only if it does not already exist, mapping fields per the corrected mapping in R.05 (certDecNum → MOF_APRV_NUM, certDecDate → MOF_APRV_DT, certCateId unmapped).
- Return `errorCode: SUCCESS` with `identifier` (candidate number) on success.

**And** do not create a duplicate row when the same (canNum, eduTyp) is re-sent (BR-012).

### AC6 – Integration Must Go Through Learning Adapter

**Given** M-Rise needs to sync Order 3 data to AMS

**When** one of the five triggers occurs:
- License-eligibility check needed
- Plan code needs refresh
- New plan code assigned
- License finalized/granted
- MIT license/education background updated

**Then**:
- M-Rise sends data through Learning Adapter.
- Learning Adapter calls AMS API.
- AMS does not receive requests directly from M-Rise.

### AC7 – Error Handling and Logging

**Given** Learning Adapter calls AMS API

**When** AMS processes successfully or fails

**Then** the system records a log including:
- API name
- Request timestamp
- Processing status (including `errorCode` for `license/grant`'s soft-failure pattern)
- Error message if any
- Reference key: agentCode / candidateCode / candidateNumber / canNum, planCode

### AC8 – Validation Failure Handling

**Given** request is invalid or referenced Order 2 master data is missing

**When** AMS processes the request

**Then** AMS must return an appropriate error and not write invalid data into the database.

| Scenario | Expected Error / Behavior | Source Status |
|---|---|---|
| Missing candidateCode/agentCode (conditions/license) | Reject request | Proposed, pending confirmation |
| Missing agentCode or planCode (plancode/refresh) | Reject request | Proposed, pending confirmation |
| (AGT_CD, PLAN_CD) not found (plancode/refresh) | Reject / not-found error, no insert | Proposed, pending confirmation |
| Missing agentCode or planCode (plancode/assign) | Reject request | Proposed, pending confirmation |
| Duplicate (agentCode, planCode) re-sync (plancode/assign) | Existence check blocks duplicate insert (BR-012) | Build requirement (PO-mandated) |
| Missing candidateNumber or educationType (license/grant) | Reject request | Proposed, pending confirmation |
| license/grant processing failure | Return errorCode: FAILED (soft failure, no exception) | Sourced |
| Missing canNum (MIT-license/update) | Reject request | Proposed, pending confirmation |
| Duplicate (canNum, eduTyp) re-sync (MIT-license/update) | Existence check blocks duplicate insert (BR-012) | Build requirement (PO-mandated) |

> **Note:** Rows marked "Proposed, pending confirmation" are engineering inferences, not requirements found directly in `order3_sheets.txt` (the sheets for all 5 Order 3 APIs contain no explicit validation/error-handling rows apart from the response `errorCode`/`errorMessage` fields already documented in R.03–R.05). These should be confirmed against actual AMS behavior during SIT and revised if AMS proves to work differently.

---

## Comments

**PO Review** (2026-07-13):
> **Approved.**
> 1. **BR-012 (insert-only duplicate risk on `plancode/assign` / `MIT-license/update`): the existence check is a hard build requirement, not an optional mitigation.** Both endpoints write candidate-facing license/plan-code data with real compliance and downstream (Order 4 e-certificate) consequences if silently duplicated by a Learning Adapter retry. The cost of a defensive existence check is small relative to that risk, and it keeps Order 3 consistent with the no-duplicate-on-resync guarantee already shipped in Order 1 (BR-002) and Order 2. "Ship as literally sourced" is rejected as the default. R.03, R.05, BR-003/009/012, AC3, AC5, and AC8 updated accordingly.
> 2. **`MIT-license/update` field mapping (R.05): build against the corrected mapping**, not the literal-as-sourced sheet. BA and BE independently converged on the same reading (`certDecNum → MOF_APRV_NUM`, `certDecDate → MOF_APRV_DT`, `certCateId` unmapped), it mirrors `license/grant`'s unambiguous pattern for the same target fields, and the literal sheet mapping is internally inconsistent with its own sample values and Vietnamese descriptions. Since this writes MOF approval fields on `tams_candidates`, getting it wrong has real compliance consequence — building against a self-contradictory literal mapping is not an acceptable default here. Workbook-owner confirmation is still logged as an open follow-up but does not block build.
> 3. **`conditions/license` as read-only:** confirmed correct and correctly scoped to Order 3 — it's the eligibility pre-check feeding directly into this Order's `license/grant` decision, not an Order-2-adjacent item. No change.
> 4. **`license/grant`'s soft-failure pattern:** confirmed the AC/validation/NFR sections correctly treat `errorCode: FAILED` as a logged failure without assuming Order 1's hard-reject pattern. No change.
> 5. **Remaining open item, does not block approval:** `plancode/refresh`'s STAT_CD server-derivation is a working assumption pending confirmation with the M-Rise API owner in SIT — logged below, same pattern as US1's open M-Rise-origin-record question and US2's open follow-ups.
> 6. Format/depth checked against US1.md and US2.md — consistent.

**BA** (2026-07-13):
> Drafted from `order3_sheets.txt` (Sheets 2, 8, 9, 13, 14) plus the previously-reviewed "API infor" summary-tab logic for `plancode/refresh`, `plancode/assign`, `license/grant`, and `MIT-license/update`. Two items remain open post-approval, non-blocking:
> - `plancode/refresh` missing `statusCode` input field (R.02): working assumption is STAT_CD is server-derived; confirm with M-Rise API owner in SIT.
> - `license/grant`'s certDecNum/certDecDate → MOF_APRV_NUM/MOF_APRV_DT step (R.04 Step 5) and the MIT/ULC `_MVL` archival special logic are carried over from the previously-reviewed "API infor" summary tab, not directly visible in `order3_sheets.txt`'s `license_grant` sheet — recommend BE re-verify against the actual stored procedure during dev.

**BE** (2026-07-13):
> Field mappings verified row-by-row against `order3_sheets.txt` for all 5 APIs — no missing, renamed, or invented fields found. Write-pattern characterization (read-only / update-only / insert-with-check / existence-key upsert) confirmed correct for all 5. `license/grant`'s soft-failure response (`errorCode` field, Sheet 13 rows 17–18/25–26) is genuinely sourced and correctly reflected throughout. Recommend the existence check for `plancode/assign`/`MIT-license/update` (BR-012) be implemented as an idempotent upsert-with-conflict-handling (return success on a redundant retry rather than a hard reject) to keep Learning Adapter's retry behavior simple — exact response contract to be finalized with BE during dev, not re-litigated at PO level.
