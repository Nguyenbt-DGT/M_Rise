# [M-RISE] AD/PD - US4 for 13 API | Order 4 – Sync/Retrieve E-Certificate Information between M-Rise and AMS via Learning Adapter

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
| **Updated** | 2026-07-14 |

---

## 1. User Story Summary

| Item | Description |
|---|---|
| **User Story** | As **AMS System**, I want to receive an agent's e-certificate document reference from **M-Rise** through **Learning Adapter** and record it, so that AMS maintains a housekeeping record of the issued e-certificate — **and** as **Learning Adapter (on behalf of M-Rise)**, I want to retrieve an agent/candidate's e-certificate eligibility and delivery-configuration information from **AMS**, so that M-Rise can generate or (re-)deliver the correct e-certificate document to the right recipient. |
| **Actor** | AMS System, Learning Adapter, M-Rise |
| **Flow** | `eCertificate/save`: M-Rise → Learning Adapter → AMS (write) · `eCertificate` (GET): Learning Adapter → AMS → Learning Adapter/M-Rise (read) |
| **Screen Demo** | N/A – Backend Integration |
| **Main Objective** | Close out the 13-API sync scope: persist the e-certificate document reference produced after Order 3's license grant, and expose a read endpoint so M-Rise/Learning Adapter can pull the agent's certificate-eligibility and delivery-configuration data needed to generate/send that certificate. |

> **PO Review note:** Order 4 is the first (and only) Order with one write API and one read API in the same User Story. The dual-clause framing above ("AMS receives..." for `eCertificate/save`, "Learning Adapter retrieves..." for `eCertificate` GET) is kept deliberately, rather than forced into Order 1–3's uniform "AMS receives" voice — see **Comments** for the reasoning.

---

## 2. Trigger / Pre-condition / Post-condition

| Type | Description |
|---|---|
| **Trigger** | An agent's e-certificate document is generated/uploaded on M-Rise after license grant (Order 3), triggering `eCertificate/save` · M-Rise/Learning Adapter needs to look up an agent's e-certificate eligibility and delivery configuration (e.g. to display, generate, or re-send a certificate), triggering `eCertificate` (GET) |
| **Pre-condition** | Order 2 has recorded a PASS training result for the agent (`tams_training_rslts.rslt = 'PASS'`) · Order 3 has granted an active license/education-background record for the agent (`tams_education_backgrounds.stat_cd = 'A'`) and the agent is active (`tams_agents.stat_cd = '01'`) — **both APIs implicitly depend on this Order 2 + Order 3 state; see Business Rules BR-006** · For `eCertificate/save`: no existing active `VIDI-CERTIFICATE` record already matches the same (AGT_CD, LIC_NO, LIC_TYP, DOC_ID) with STATUS = 'FINISH' · Learning Adapter service is running · AMS API endpoints are ready to receive requests |
| **Post-condition** | `eCertificate/save`: AMS records a new e-certificate housekeeping row in TWRK_AGT_CERTIFICATES (STATUS = 'FINISH', AGT_CERTIFICATES_TYP = 'VIDI-CERTIFICATE') sourced from the agent/candidate/education/training-result data, or rejects with `returnCode = '406'` (duplicate) / `returnCode = '404'` (agent not eligible) · `eCertificate` (GET): Learning Adapter/M-Rise receives the agent's education-background/certificate data (and, per the underlying query, delivery-configuration data — see R.02 and the response-contract gap noted below) needed to generate or deliver the certificate · Log is recorded for each API call |

---

## 3. Integration Flow

> **Important Note:**
> M-Rise **does not call AMS directly**.
> All Order 4 data must go through **Learning Adapter**, in both directions — `eCertificate/save` (M-Rise → Learning Adapter → AMS) and `eCertificate` (GET) (Learning Adapter → AMS, response relayed back toward M-Rise).

---

## 4. Scope

### 4.1 In Scope

| No. | API | Purpose | Trigger |
|---|---|---|---|
| 1 | POST /api/v1/eCertificate/save | Save/upload an agent's e-certificate document reference into AMS housekeeping table | When M-Rise generates/uploads an e-certificate document for an agent after license grant |
| 2 | GET /api/v1/eCertificate | Retrieve an agent/candidate's e-certificate eligibility (education background) and certificate delivery-configuration information | When M-Rise/Learning Adapter needs to display, generate, or (re-)deliver an agent's e-certificate |

### 4.2 Out of Scope

| No. | Item | Belong To |
|---|---|---|
| 1 | Sync class template / exam info / class info | Order 1 |
| 2 | Sync attendance result / training result / inform CPA result | Order 2 |
| 3 | Check license conditions / grant license / assign / refresh plan code | Order 3 |
| 4 | Actual generation or transmission of the certificate file/document (PDF rendering, email/SMS delivery) | Outside 13-API scope — `eCertificate` (GET) only returns the data needed for M-Rise to perform generation/delivery, it does not perform it |

---

## 5. Requirement Details

### R.01 – Save E-Certificate

| Item | Description |
|---|---|
| **API** | POST /api/v1/eCertificate/save |
| **API Meaning** | Insert an agent's e-certificate document reference into AMS's e-certificate housekeeping table, after validating the agent has no existing active matching certificate |
| **Trigger** | M-Rise generates/uploads an e-certificate document for an agent (typically after Order 3's `license/grant` has completed) |
| **Purpose** | Record which document (DOC_ID) was issued as the agent's e-certificate for a given license, for housekeeping/audit and to support later retrieval via `eCertificate` (GET) |
| **Method** | POST |
| **Request Type** | Single-object request |
| **Target Table** | TWRK_AGT_CERTIFICATES |
| **Reference / Check Tables** | tams_agents (stat_cd = '01'), tams_candidates, tams_education_backgrounds (stat_cd = 'A'), tams_training_rslts (rslt = 'PASS') — read to source additional fields and to gate the insert |

#### Main Data Object

| Field | Required | Description |
|---|---|---|
| agentCode | Yes | Agent code (sourced from tams_agents.agt_code) |
| licNo | Yes | License number (input payload; corresponds to tams_education_backgrounds.edu_rmrk) |
| licType | Yes | License type, e.g. ULC or MIT (input payload; corresponds to tams_education_backgrounds.edu_typ) |
| docId | Yes | Document ID of the uploaded e-certificate file (input payload) |

#### Sample Payload

```json
{
  "agentCode": "34583",
  "licNo": "129",
  "licType": "ULC",
  "docId": "AT-FILES-AT-RCL-689c1680f61a1302c38d8b19"
}
```

#### Sample Response Payload

```json
{
  "returnCode": "0000",
  "message": "Successfully"
}
```

> **Response shape note:** this API's response is a simple `{returnCode, message}` pair — not Order 3's `errorCode: SUCCESS/FAILED` soft-failure pattern (R.04/R.05 of US3), and not a list/array-per-item response like Orders 1–3's batch APIs. `returnCode` doubles as both success (`"0000"`) and business-rejection (`"406"` / `"404"`, see Business Rules below) signaling, closer to an HTTP-status-code convention embedded in the body.

#### Field Mapping

| Source Field | Target Table | Target Field |
|---|---|---|
| agentCode | TWRK_AGT_CERTIFICATES | AGT_CD |
| licNo | TWRK_AGT_CERTIFICATES | LIC_NO |
| licType | TWRK_AGT_CERTIFICATES | LIC_TYP |
| docId | TWRK_AGT_CERTIFICATES | DOC_ID |
| (derived via JOIN, not in request payload) | TWRK_AGT_CERTIFICATES | LOC_CODE, AGT_NM, AGT_DOB, AGT_ADDR, EXAM_DT, LIC_DT, LIC_EXP_DT, TRXN_DT |
| (system-set) | TWRK_AGT_CERTIFICATES | STATUS = 'FINISH', AGT_CERTIFICATES_TYP = 'VIDI-CERTIFICATE' |

#### Logic Summary

> **Sourcing caveat (inline):** Steps 1 and 3 below — the existence-check SQL, the JOIN field list (LOC_CODE/AGT_NM/AGT_DOB/AGT_ADDR/EXAM_DT/LIC_DT/LIC_EXP_DT/TRXN_DT), and the eligibility filters (`stat_cd='01'`/`stat_cd='A'`/`rslt='PASS'`) are sourced from the previously-reviewed "API infor" logic-summary tab, **not** directly visible in `order4_sheets.txt`'s raw sheet dump (which only contains the 4 request fields and the `{returnCode, message}` response shape for this API). Same sourcing pattern US3 used for `MIT-license/update`'s MOF fields and `_MVL` archival logic. **BE to re-confirm against the actual stored procedure/DDL during dev before this is treated as final.**

| Step | Condition | Expected Logic |
|---|---|---|
| 1 | Always | Existence check: `SELECT COUNT(*) FROM twrk_agt_certificates WHERE agt_cd = :agentCode AND lic_no = :licNo AND lic_typ = :licType AND lic_stat_cd = 'A' AND status = 'FINISH' AND doc_id = :docId AND agt_certificates_typ = 'VIDI-CERTIFICATE'`. |
| 2 | Count > 0 (active matching cert already exists) | Reject the request. Return `returnCode = '406'`, `message = "License No {licNo} has existed."` Do not insert. |
| 3 | Count = 0 | Insert a new row into TWRK_AGT_CERTIFICATES using AGT_CD/LIC_NO/LIC_TYP/DOC_ID from the payload, plus LOC_CODE, AGT_NM, AGT_DOB, AGT_ADDR, EXAM_DT, LIC_DT, LIC_EXP_DT, TRXN_DT sourced via a JOIN across tams_agents (stat_cd = '01'), tams_candidates, tams_education_backgrounds (stat_cd = 'A'), and tams_training_rslts (rslt = 'PASS'), with STATUS = 'FINISH' and AGT_CERTIFICATES_TYP = 'VIDI-CERTIFICATE'. Return `returnCode = '0000'`, `message = "Successfully"`. |
| 4 | JOIN in Step 3 finds no eligible row (agent not stat_cd='01', or no active education background, or no PASS training result) | Reject the request rather than inserting a row with null derived fields. Return `returnCode = '404'`, `message = "Agent not eligible for e-certificate."` Do not insert. **PO-assigned code — see Business Rules and Comments.** |

> Note: this API carries no explicit `action` (INS/UPD/DEL) field. It is insert-only, gated by the existence check in Step 1 — architecturally similar to Order 3's mandated BR-012 existence-check pattern (`plancode/assign`, `MIT-license/update`), except here the check is already present in the sourced SQL rather than a PO-mandated addition.

> **⚠️ Proposed-derived-value flag: `LIC_STAT_CD`.** Step 1's existence check filters on `lic_stat_cd = 'A'`, but `LIC_STAT_CD` is not introduced anywhere else in this document — the Field Mapping table above only names `STATUS = 'FINISH'` as a system-set field on `TWRK_AGT_CERTIFICATES`. **Working assumption, pending confirmation: `LIC_STAT_CD` is a fixed/derived column on `TWRK_AGT_CERTIFICATES` (value `'A'` = active), set automatically at insert time alongside `STATUS`/`AGT_CERTIFICATES_TYP`, not a client-supplied field** — same pattern as US3's `STAT_CD` derivation gap on `plancode/refresh`. This is a **non-blocking, logged follow-up**: build against the working assumption; BE to confirm against the actual stored procedure/DDL during dev and revise if wrong.

#### Validation / Error Handling

| Scenario | Expected Behavior | Source Status |
|---|---|---|
| Active matching cert already exists (agt_cd, lic_no, lic_typ, doc_id, status='FINISH') | Reject with `returnCode = '406'`, `message = "License No X has existed."` | Sourced ("API infor" logic tab) |
| Missing agentCode, licNo, licType, or docId | Reject request | Proposed — not explicitly specified in source; recommended default, confirm in SIT |
| Agent/education/training-result JOIN finds no matching active row (agent not stat_cd='01', or no active education background, or no PASS training result) | Reject with `returnCode = '404'`, `message = "Agent not eligible for e-certificate."` — a **distinct code from `406`** so Learning Adapter's retry/alerting logic can tell "already exists, don't retry" apart from "not eligible, don't retry ever." **PO-assigned; confirm with M-Rise API owner in SIT, revise if a project-standard code convention says otherwise.** | Proposed — inferred; PO-assigned code pending SIT confirmation |

#### Special Logic

| Scenario | Expected Behavior |
|---|---|
| Cross-Order dependency (implicit) | The insert's sourcing JOIN silently requires the agent to already have a PASS training result (Order 2) and an active license/education-background record (Order 3). If either is missing, the JOIN returns no row and the insert has nothing to source from `LOC_CODE`/`AGT_NM`/etc. This is not stated as an explicit pre-check in the source SQL — it is a side effect of the JOIN — but functions as a de facto eligibility gate. See BR-006. |

---

### R.02 – Retrieve E-Certificate Information

| Item | Description |
|---|---|
| **API** | GET /api/v1/eCertificate |
| **API Meaning** | Read-only lookup, by licNo + agtCode, of an agent/candidate's e-certificate-relevant education-background data and the certificate's delivery configuration |
| **Trigger** | M-Rise/Learning Adapter needs to look up an agent's certificate eligibility/education data and delivery configuration (e.g. to generate a new certificate, re-display it, or re-send it) |
| **Purpose** | Supply M-Rise with the agent/education data and delivery configuration needed to generate or (re-)deliver an e-certificate |
| **Method** | GET |
| **Request Type** | Query-parameter request (licNo, agtCode) |
| **Target Tables** | N/A for write — read-only |
| **Reference / Query Tables** | tams_agents JOIN tams_education_backgrounds JOIN tams_training_rslts (same PASS/active filters as R.01: agent stat_cd='01', education stat_cd='A', training rslt='PASS') for agent/education data · `cas.tdelivery_settings_dtls` / `cas.tdoc_settings` for certificate delivery configuration (recipient type, delivery type, source path, delivery props) |

#### Main Data Object (Request)

| Field | Required | Description |
|---|---|---|
| licNo | Yes | License number |
| agtCode | Yes | Agent code |

#### Sample Request

```
GET /api/v1/eCertificate?licNo=123456&agtCode=A001
```

#### Sample Response Payload (as sourced)

```json
{
  "returnCode": "0000",
  "message": "Successfully",
  "tamsEducationBackgrounds": [
    {
      "agtCode": "BF345",
      "canNum": "2801182011",
      "eduTyp": "MIT",
      "eduRmrk": "LIC-001"
    }
  ]
}
```

#### Field Mapping (query fields, per sheet)

| Source Field | Source Table | Source Field Name |
|---|---|---|
| licNo (request) | tams_education_backgrounds | edu_rmrk |
| agtCode (request) | tams_agents | agt_code |
| locCode (response, per sheet detail rows) | tams_agents | loc_code |
| candidateNo / canNum (response, Sheet 6 row 9) | tams_agents | can_num |
| agentName (response) | tams_agents | agt_nm |
| eduType (response) | tams_education_backgrounds | edu_typ |
| licenseNo (response) | tams_education_backgrounds | edu_rmrk |
| eduEndDate (response) | tams_education_backgrounds | edu_end_dt |
| licenseDate (response) | tams_education_backgrounds | licence_dt |
| eduStatus (response) | tams_education_backgrounds | stat_cd |

> **Placement note on `candidateNo`/`canNum`:** Sheet 6's row 9 sits inside the sheet's "Response data" block (which begins at row 6, header `H: Response data`), alongside locCode/agentName/eduType/etc. (rows 7–15), and is **not** grouped with the two request-parameter rows (licNo row 4, agtCode row 5). The sample response JSON also already carries `"canNum": "2801182011"` inside the `tamsEducationBackgrounds` array (see Sample Response Payload above). On that basis this is placed as a **response** field, not added to the Main Data Object (Request) table. The sheet additionally marks it `O: Key join`, meaning it is likely used internally to correlate the response row back to `tams_agents`/`tams_candidates` rather than being a purely cosmetic display field — a join-key omission risk, not just a display-field omission risk, if it were ever dropped from the mapping.

#### ⚠️ Response-contract documentation gap — scoped pre-build blocker

The sheet's field-mapping detail rows (locCode, agentName, eduType, licenseNo, eduEndDate, licenseDate, eduStatus) describe a **richer response** than the sheet's own sample JSON payload, which only shows `{returnCode, message, tamsEducationBackgrounds: [{agtCode, canNum, eduTyp, eduRmrk}]}`. Neither the detail rows nor the sample payload include any of the `cas.tdelivery_settings_dtls` / `cas.tdoc_settings` delivery-configuration fields (recipient type, delivery type, source path, delivery props) that the underlying query logic ("API infor" tab) explicitly fetches, and **no field list for these two tables exists anywhere in `order4_sheets.txt`** — not even in abbreviated form, confirmed by direct sheet check.

**PO ruling:** this story does **not** fabricate a delivery-settings schema — without real column names any such list would be invented, not inferred, and risks becoming a de facto contract that FE/Learning Adapter build against and later have to break. This is treated as a **scoped pre-build blocker on the delivery-settings portion of R.02's response only**: that sub-piece cannot be implemented until the delivery-settings table DDL or an updated "API infor" tab entry is obtained from the workbook owner. It does **not** block story approval, and it does **not** block the rest of R.02 — the education-background lookup portion is fully specified and can proceed to build independently. This mirrors how US3 shipped with the `STAT_CD`-derivation gap as a non-blocking, logged follow-up: the difference here is narrower — this gap blocks build of one specific response sub-piece until the DDL arrives, while the rest of the API is buildable now. PO to be looped in once the DDL is requested.

#### Logic Summary

| Step | Expected Logic |
|---|---|
| 1 | Look up tams_agents JOIN tams_education_backgrounds JOIN tams_training_rslts by agtCode + licNo, filtered to agent stat_cd = '01', education stat_cd = 'A', training rslt = 'PASS' (same eligibility filter as R.01's insert JOIN). |
| 2 | Separately query `cas.tdelivery_settings_dtls` / `cas.tdoc_settings` for the certificate's delivery configuration (recipient type, delivery type, source path, delivery props), used to determine how/where to generate or send the certificate. **Blocked pending DDL — see gap note above.** |
| 3 | Return `returnCode`, `message`, and the agent/education data (`tamsEducationBackgrounds` array per the sourced sample) — and, once Step 2's DDL is confirmed, the delivery-configuration data in its true shape. |

> Note: this is a read-only API. No INS/UPD/DEL action applies — do not build write logic for this endpoint.

#### Validation / Error Handling

| Scenario | Expected Behavior | Source Status |
|---|---|---|
| Missing licNo or agtCode | Reject request | Proposed — not explicitly specified in source; recommended default, confirm in SIT |
| No matching agent/education/training-result row (not eligible per PASS/active filters) | Return empty `tamsEducationBackgrounds` (and no delivery-settings data) rather than erroring | Proposed — inferred from the API's lookup/query-style purpose; confirm exact behavior with BE |
| Delivery-settings lookup finds no configuration for the doc type | Behavior undocumented — blocked pending DDL, see response-contract gap above | Proposed — must be confirmed with BE once schema is available |

---

## 6. Business Rules & Validations

| Rule ID | Rule Name | Business Description | Impacted Flow | Example Scenario |
|---|---|---|---|---|
| BR-001 | Save is insert-only with a sourced existence check | `eCertificate/save` inserts a new TWRK_AGT_CERTIFICATES row only if no active matching (AGT_CD, LIC_NO, LIC_TYP, DOC_ID, STATUS='FINISH') VIDI-CERTIFICATE record already exists. Refer AC1. | eCertificate/save API | Same (agentCode, licNo, licType, docId) re-sent while an active cert exists → rejected with returnCode 406, no duplicate insert |
| BR-002 | Non-standard return-code response pattern | `eCertificate/save` uses a `{returnCode, message}` response where returnCode carries success (`0000`) and two distinct business-rejection meanings (`406` duplicate, `404` not eligible) — distinct from Order 1's list/action-based responses and Order 3's `errorCode: SUCCESS/FAILED` soft-failure pattern. Refer AC1, AC4. | eCertificate/save API | returnCode = 406 or 404 → treated as a business rejection, not a system error |
| BR-003 | Retrieval is read-only | `eCertificate` (GET) never writes to AMS; it only queries agent/education-background/training-result and delivery-settings data. Refer AC2. | eCertificate (GET) API | Query for agtCode/licNo → returns data, no DB write |
| BR-004 | Integration via Learning Adapter only | M-Rise does not call AMS directly, in either the save (write) or GET (read) direction. All requests must go through Learning Adapter. Refer AC3. | Both APIs | M-Rise ↔ Learning Adapter ↔ AMS |
| BR-005 | Logging required | Each API call — including the GET — must have a log for troubleshooting and SIT/UAT evidence. Refer AC4. | Both APIs | Log API name, timestamp, status, error message, reference key |
| BR-006 | Implicit cross-Order eligibility dependency | Both `eCertificate/save`'s sourcing JOIN and `eCertificate` (GET)'s lookup JOIN require the agent to have a PASS training result (Order 2, tams_training_rslts.rslt='PASS') and an active license/education-background record (Order 3, tams_education_backgrounds.stat_cd='A', tams_agents.stat_cd='01'). Neither API states this as an explicit precondition/error message in the source — it is a side effect of the JOIN filters — but it functions as a de facto eligibility gate carried over from prior Orders. **The two APIs deliberately handle the same gate-failure condition differently: GET (R.02) silently returns an empty result, while save (R.01) explicitly rejects with `returnCode = '404'` — this asymmetry is intentional, not an inconsistency: a query with nothing to return should not error, but a write with nothing to source from should not insert null housekeeping fields.** Refer Pre-condition (Section 2), AC1, AC2. | Both APIs | Agent has no PASS training result → JOIN returns no row → save rejects with 404, GET returns empty education data |
| BR-007 | Response contract incompleteness (GET) — scoped pre-build blocker | `eCertificate` (GET)'s sourced sample response does not include the delivery-configuration data (`cas.tdelivery_settings_dtls`/`cas.tdoc_settings`) that its own underlying query logic fetches, and no field list for those tables exists in source. This is a **non-blocking-for-approval, blocking-for-build (on that sub-piece only)** gap: the education-background portion of R.02 proceeds now; the delivery-settings portion is blocked until the workbook owner/BE supply the DDL. Refer AC2, Comments. | eCertificate (GET) API | Response omits recipient type / delivery type / source path fields until DDL is confirmed |
| BR-008 | Agent-not-eligible save rejection uses a distinct return code | `eCertificate/save`'s eligibility-JOIN-empty case rejects with `returnCode = '404'` and message `"Agent not eligible for e-certificate."`, distinct from `406`'s duplicate-cert case, so Learning Adapter can distinguish "already exists" from "not eligible" by code alone. Refer R.01 Step 4, AC1, AC5. | eCertificate/save API | Agent not stat_cd='01' / no active license / no PASS result → returnCode 404, no insert |

---

## 7. Impact and Risk

| Area | Impact / Risk |
|---|---|
| Backend | AMS needs to expose 2 APIs to Learning Adapter: 1 write (`eCertificate/save`) and 1 read (`eCertificate` GET) — the first bidirectional-flow Order in this integration |
| Database | Impact on TWRK_AGT_CERTIFICATES (write); read-only impact on tams_agents, tams_candidates, tams_education_backgrounds, tams_training_rslts, cas.tdelivery_settings_dtls, cas.tdoc_settings |
| Reference Tables | Dependency on Order 2's tams_training_rslts (PASS results) and Order 3's tams_education_backgrounds (active license/education records) |
| Integration Risk | If Learning Adapter is down, both e-certificate save and retrieval will be blocked — save blocks certificate housekeeping, GET blocks certificate generation/delivery, both being end-of-flow / customer-facing outcomes |
| Data Integrity Risk | The 406/404 existence and eligibility checks in `eCertificate/save` prevent duplicate rows and null-field inserts; both codes are PO-confirmed but should be reconfirmed against actual AMS behavior in SIT |
| Response-Contract Risk | `eCertificate` (GET)'s delivery-configuration fields are undocumented in source — build of that sub-piece is blocked until DDL is obtained; BE must clarify before FE/M-Rise integration work proceeds against that portion of the contract |
| Downstream Impact | This is the final Order (4 of 4) — failures here affect the customer-facing e-certificate outcome directly, with no further Order to compensate |

---

## 8. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Performance / Throughput | N/A – Event-driven / on-demand query, not large batch |
| Security / Privacy | `eCertificate` (GET) exposes agent PII (agtCode, agentName, DOB/address per R.01's sourced fields) and license data — access should be restricted to Learning Adapter/M-Rise via the same integration-path enforcement as all other Orders (BR-004); no additional data-masking requirement identified in source, flag for PO/Security review |
| Audit / Logging | Each API call must log: API name, request timestamp, processing status, error message if any, reference key (agentCode, licNo) — for the GET this includes read/query outcomes, not just writes |
| Availability / Resilience | Learning Adapter must operate stably to ensure successful save and retrieval |
| Idempotency | `eCertificate/save` is idempotent-by-rejection: re-sending the same (agentCode, licNo, licType, docId) while an active match exists is blocked via returnCode 406, not silently duplicated. `eCertificate` (GET) is trivially idempotent (read-only). |
| Error Traceability | Error messages must be clear enough for BA/Tester to trace failure root cause during SIT/UAT, distinguishing `eCertificate/save`'s 406 (duplicate) and 404 (not eligible) business-rejections from other failure modes |

---

## 9. Dependencies / References

| Dependency | Description |
|---|---|
| M-Rise | Source system where the e-certificate document is generated/uploaded, and consumer of the GET's data for certificate generation/delivery |
| Learning Adapter | Middleware that receives data from M-Rise and calls AMS APIs, in both directions |
| AMS | Target/source system: stores the e-certificate housekeeping record (TWRK_AGT_CERTIFICATES) and serves eligibility/delivery-configuration data |
| Order 2 | Provides tams_training_rslts PASS result, implicitly required by both Order 4 APIs' JOIN filters (BR-006) |
| Order 3 | Provides tams_education_backgrounds active license/education record, implicitly required by both Order 4 APIs' JOIN filters (BR-006) |
| cas.tdelivery_settings_dtls / cas.tdoc_settings | Delivery-configuration reference tables queried by `eCertificate` (GET); response-contract gap noted in R.02/BR-007 |

---

## 13. Acceptance Criteria

### AC1 – Save E-Certificate Successfully

**Given** M-Rise generates/uploads an e-certificate document for an agent

**When** Learning Adapter calls API: `POST /api/v1/eCertificate/save`

**Then** AMS must:
- Receive agentCode, licNo, licType, docId from Learning Adapter.
- Check whether an active matching (AGT_CD, LIC_NO, LIC_TYP, DOC_ID, STATUS='FINISH', AGT_CERTIFICATES_TYP='VIDI-CERTIFICATE') record already exists in TWRK_AGT_CERTIFICATES.
- If it exists, reject with `returnCode = '406'`, `message = "License No {licNo} has existed."`, and not insert.
- If the agent is not eligible (JOIN across tams_agents/tams_candidates/tams_education_backgrounds/tams_training_rslts finds no active row), reject with `returnCode = '404'`, `message = "Agent not eligible for e-certificate."`, and not insert.
- If it does not exist and the agent is eligible, insert a new row sourcing LOC_CODE, AGT_NM, AGT_DOB, AGT_ADDR, EXAM_DT, LIC_DT, LIC_EXP_DT, TRXN_DT via JOIN across tams_agents (stat_cd='01'), tams_candidates, tams_education_backgrounds (stat_cd='A'), tams_training_rslts (rslt='PASS'), with STATUS='FINISH' and AGT_CERTIFICATES_TYP='VIDI-CERTIFICATE'.
- Return `returnCode = '0000'`, `message = "Successfully"` on success.

**And** do not create a duplicate active certificate row when the same (agentCode, licNo, licType, docId) is re-sent while an active match exists (BR-001).

### AC2 – Retrieve E-Certificate Information Successfully

**Given** M-Rise/Learning Adapter needs an agent's e-certificate eligibility and delivery-configuration data

**When** Learning Adapter calls API: `GET /api/v1/eCertificate?licNo=...&agtCode=...`

**Then** AMS must:
- Look up tams_agents JOIN tams_education_backgrounds JOIN tams_training_rslts by agtCode + licNo, filtered to agent stat_cd='01', education stat_cd='A', training rslt='PASS'.
- Query cas.tdelivery_settings_dtls / cas.tdoc_settings for the certificate's delivery configuration (build blocked on this portion pending DDL — see BR-007).
- Return `returnCode`, `message`, and the agent/education data (per the sourced sample: `tamsEducationBackgrounds[{agtCode, canNum, eduTyp, eduRmrk}]`).

**And** this API must not insert, update, or delete any AMS record.

**And** the exact shape of the delivery-configuration data in the response remains a scoped pre-build blocker pending workbook-owner/BE DDL confirmation (BR-007) — the education-background portion of this AC is not blocked.

### AC3 – Integration Must Go Through Learning Adapter

**Given** M-Rise needs to save or retrieve Order 4 e-certificate data

**When** one of the two triggers occurs:
- E-certificate document generated/uploaded on M-Rise (save)
- E-certificate eligibility/delivery-configuration lookup needed (GET)

**Then**:
- M-Rise/Learning Adapter and AMS communicate only through Learning Adapter.
- AMS does not receive requests directly from M-Rise, and M-Rise does not query AMS directly.

### AC4 – Error Handling and Logging

**Given** Learning Adapter calls AMS API (save or GET)

**When** AMS processes successfully or fails

**Then** the system records a log including:
- API name
- Request timestamp
- Processing status (including `returnCode` for `eCertificate/save`'s 406/404 business-rejection patterns)
- Error message if any
- Reference key: agentCode, licNo

### AC5 – Validation Failure Handling

**Given** request is invalid or the referenced agent is not eligible (no PASS training result / no active license record)

**When** AMS processes the request

**Then** AMS must return an appropriate error/empty result and not write invalid data into the database.

| Scenario | Expected Error / Behavior | Source Status |
|---|---|---|
| Active matching cert already exists (eCertificate/save) | returnCode = 406, "License No X has existed." | Sourced |
| Missing agentCode/licNo/licType/docId (eCertificate/save) | Reject request | Proposed, pending confirmation |
| Agent not eligible (no PASS result / no active license) at save time | Reject with `returnCode = '404'`, `message = "Agent not eligible for e-certificate."` — distinct from `406` | PO-assigned, pending SIT confirmation |
| Missing licNo/agtCode (eCertificate GET) | Reject request | Proposed, pending confirmation |
| No matching eligible agent (eCertificate GET) | Return empty tamsEducationBackgrounds rather than erroring | Proposed, pending confirmation |

> **Note:** Rows marked "Proposed, pending confirmation" are engineering inferences, not requirements found directly in `order4_sheets.txt` or the cross-referenced "API infor" logic tab. These should be confirmed against actual AMS behavior during SIT and revised if AMS proves to work differently.

---

## Comments

**PO Review** (2026-07-14):
> **Approved.**
> 1. **User Story framing (Section 1): keep the dual-clause framing.** Order 4 is the only Order with one write and one read direction in the same story; forcing the GET into "AMS wants to receive" language would misdescribe what the endpoint does purely for surface-level consistency with Orders 1–3. BE confirmed this is a documentation-only question with no contract/testing impact — each API already has its own independent R.0x section, schema, Logic Summary, and AC, so nothing about build/test/versioning depends on the narrative voice. Accuracy wins over uniformity here. No change requested.
> 2. **Delivery-settings response-schema gap (R.02/BR-007): non-blocking for story approval, scoped pre-build blocker for that sub-piece only.** No field list for `cas.tdelivery_settings_dtls`/`cas.tdoc_settings` exists anywhere in source — BA/BE correctly declined to fabricate one. This is a "can't build this specific sub-piece without it" gap, not a "nice to have more detail" gap, but it does not block the rest of R.02 (education-background lookup is fully specified and buildable now) and does not block the whole API or the story. Same non-blocking-follow-up treatment as US3's STAT_CD gap, scoped more narrowly. PO to be looped in when the DDL is requested from the workbook owner.
> 3. **Two-rejection-code design for `eCertificate/save`: right shape, code assigned now instead of left TBD.** Two distinct rejection cases (duplicate-cert vs. agent-not-eligible) genuinely need two distinct codes so Learning Adapter's retry/alerting logic isn't forced to overload one meaning — BA/BE's design is correct. This is a product-level naming call PO can make without engineering sign-off: **agent-not-eligible is assigned `returnCode = '404'`**, paired with message `"Agent not eligible for e-certificate."` Reflected directly in R.01 Step 4, Validation/Error Handling, BR-008, AC1, and AC5. Still logged as pending SIT confirmation against actual AMS/M-Rise API-owner convention, same confirmation posture as the sourced `406` code.
> 4. **`LIC_STAT_CD` proposed-derived-value flag: ships as non-blocking, logged follow-up**, same pattern as US3's `STAT_CD` derivation gap on `plancode/refresh`. Build against the working assumption (fixed/derived `'A'` value, not client-supplied); BE to confirm against the stored procedure/DDL during dev.
> 5. **Format/depth and 13-API cross-check:** matched to US1/US2/US3's structure and precedent for non-standard response shapes and "proposed, pending confirmation" rows. Cross-checked API count across all 4 Orders: Order 1 = 3 APIs, Order 2 = 3 APIs, Order 3 = 5 APIs, Order 4 = 2 APIs → **13 of 13 APIs accounted for, no gaps or overlaps.** This closes out the 13-API sync backlog.
> 6. **Remaining open items, do not block approval:** (a) `LIC_STAT_CD` derivation (R.01) — confirm with BE/DDL in dev; (b) delivery-settings response schema (R.02/BR-007) — blocked on workbook-owner DDL for that sub-piece only; (c) `returnCode = '404'` for agent-not-eligible (R.01/BR-008) — confirm against AMS/M-Rise API-owner convention in SIT; (d) R.01's Steps 1/3 SQL/JOIN sourcing from the "API infor" tab — BE to re-confirm against the actual stored procedure during dev, same posture as US3's cross-referenced logic.

**BA** (2026-07-13):
> Drafted from `order4_sheets.txt` (Sheet 3 `Save_eCertificate`, Sheet 6 `eCertificate_GET`) plus the previously-reviewed "API infor" logic-summary tab for both APIs. Added the missing `candidateNo`/`canNum` field mapping and the `LIC_STAT_CD` flag per BE review. Declined to fabricate a delivery-settings schema since no source data defines it — flagged as a scoped pre-build blocker instead of a guessed field list. Format/depth matched to US1.md/US3.md's structure and precedent for non-standard response shapes and "proposed, pending confirmation" validation rows.

**BE** (2026-07-13):
> Field mappings verified against `order4_sheets.txt` for both APIs — no other missing, renamed, or invented fields beyond the `candidateNo`/`canNum` correction. `eCertificate/save`'s `{returnCode, message}` pattern is a genuine third response shape in this integration (distinct from Order 3's `errorCode` soft-failure pattern) — correctly characterized. BR-006's GET-silent-empty vs. save-explicit-reject asymmetry is the technically correct choice (an insert with null housekeeping fields is a data-integrity problem an empty GET result never is) — confirmed, not a documentation inconsistency. Recommended the agent-not-eligible case use a return code distinct from `406`; PO assigned `404` (Comments item 3). Recommended treating the delivery-settings gap as a scoped pre-build blocker rather than inventing a schema; PO concurred (Comments item 2).
