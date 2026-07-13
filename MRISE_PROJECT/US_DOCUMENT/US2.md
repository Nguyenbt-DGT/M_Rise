# [M-RISE] AD/PD - US2 for 13 API | Order 2 – Sync Attendance, Exam Result & Training Result Information from M-Rise to AMS via Learning Adapter

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
| **User Story** | As **AMS System**, I want to receive candidate exam attendance/result and training result information from **M-Rise** through **Learning Adapter**, so that AMS can record **exam attendance & result**, **class training result**, **candidate result**, **education/license background**, **MOF approval decision**, and **CPA-result-sent status** for downstream license, plan code, and e-certificate processing. |
| **Actor** | AMS System, Learning Adapter, M-Rise |
| **Flow** | M-Rise → Learning Adapter → AMS |
| **Screen Demo** | N/A – Backend Integration |
| **Main Objective** | Sync Order 2 candidate performance/result data from M-Rise to AMS, using the class/exam/template shell that Order 1 already set up, and prepare data for Order 3/Order 4 processing. |

---

## 2. Trigger / Pre-condition / Post-condition

| Type | Description |
|---|---|
| **Trigger** | Candidate exam attendance and result are recorded/finalized on M-Rise · Candidate training/class result (incl. MIT, education background, MOF approval) is finalized on M-Rise · Candidate result has been forwarded to the AP (CPA) system and M-Rise needs to inform AMS of that status |
| **Pre-condition** | Order 1 data already exists in AMS: class (TAMS_CLASSES), exam (TAMS_EXAM_DETAILS / TAMS_EXAMS), template (TAMS_TEMPLS) · Candidate/agent master data already exists in AMS if referenced · For `inform-cpa-result/sync` specifically: `training-result/sync` must already have run successfully for the same (classCode, candidateNum), since that is what creates the TAMS_CANDIDATE_RSLTS row this API updates · Learning Adapter service is running · AMS API endpoints are ready to receive requests |
| **Post-condition** | AMS successfully receives and stores exam attendance/result, training/class result, candidate result, education background, and MOF approval decision · AMS reflects whether a candidate result has been sent to the AP (CPA) system · Data is ready for Order 3 (license/plan code) and Order 4 (e-certificate) processing · Log is recorded for each API call |

---

## 3. Integration Flow

> **Important Note:**
> M-Rise **does not call AMS directly**.
> All Order 2 data must go through **Learning Adapter**.

---

## 4. Scope

### 4.1 In Scope

| No. | API | Purpose | Trigger |
|---|---|---|---|
| 1 | POST /api/v1/education/exam-result/sync | Sync candidate exam attendance and exam result to AMS | When a candidate's exam attendance/result is recorded on M-Rise |
| 2 | POST /api/v1/education/training-result/sync | Sync candidate training/class result, candidate result, education background, and MOF approval decision to AMS | When a candidate's training/class result is finalized on M-Rise (incl. MIT completion, LDP license, MOF approval) |
| 3 | POST /api/v1/education/inform-cpa-result/sync | Inform AMS that a candidate result has been sent to the AP (CPA) system | When M-Rise sends a candidate result to the AP (CPA) system and needs to notify AMS of that status |

> **Scope note (PO):** `inform-cpa-result/sync` is confirmed in Order 2, not Order 3. It only sets a "sent to AP" flag on `TAMS_CANDIDATE_RSLTS`, and that row is created within Order 2 itself by `training-result/sync` (MIT path) — not by Order 3's license/plan-code grant. There is no dependency on Order 3. It is sequenced *after* `training-result/sync` within Order 2 (see Pre-condition and BR-010).

### 4.2 Out of Scope

| No. | Item | Belong To |
|---|---|---|
| 1 | Grant license | Order 3 |
| 2 | Assign plan code | Order 3 |
| 3 | Refresh plan code | Order 3 |
| 4 | License condition check | Order 3 |
| 5 | MIT license update | Order 3 |
| 6 | Upload / retrieve e-certificate | Order 4 |

---

## 5. Requirement Details

### R.01 – Sync Exam Attendance & Result

| Item | Description |
|---|---|
| **API** | POST /api/v1/education/exam-result/sync |
| **API Meaning** | Insert or update candidate exam attendance registration and exam result into AMS |
| **Trigger** | Candidate exam attendance and/or exam result is recorded on M-Rise |
| **Purpose** | Sync candidate attendance and exam score/result to AMS so exam outcome is available for license/plan-code processing in Order 3 |
| **Method** | POST |
| **Request Type** | List data request |
| **Target Tables** | TAMS_EXAM_ATTD_REG, TAMS_EXAM_RSLTS |
| **Main Data Object Table** | TAMS_EXAM_RSLTS (exam result), TAMS_EXAM_ATTD_REG (attendance registration) |

#### Main Data Object

| Field | Required | Description |
|---|---|---|
| examCode | Yes | Exam master code |
| classCode | Yes | Class code |
| examTypeCode | Yes | Exam type code |
| canNum | Yes | Candidate number |
| candidateCode | No | Candidate code |
| examNo | No | Exam attempt number |
| attd | No | Attendance / attendance type |
| examScore | No | Exam score |
| examAttd | No | Exam attendance flag (present at exam or not) |
| createdBy | No | Created by |
| createdDate | No | Created date |
| updatedBy | No | Updated by |
| updatedDate | No | Updated date |

#### Sample Payload

```json
[
  {
    "examCode": "E0UL500170_EX_CKH",
    "classCode": "E0UL500170",
    "examTypeCode": "PR_FINAL_EX_CKH",
    "canNum": "2800982951",
    "candidateCode": "KW751",
    "examNo": 1,
    "attd": "FULL",
    "examScore": 20,
    "examAttd": "Y",
    "createdBy": "NPTHUY",
    "createdDate": "2026-05-12T08:45:21.095Z",
    "updatedBy": "NPTHUY",
    "updatedDate": "2026-05-12T08:45:21.095Z"
  }
]
```

#### Field Mapping

| Source Field | Target Table | Target Field |
|---|---|---|
| examCode | TAMS_EXAM_RSLTS | EXAM_MASTER_CD |
| examCode | TAMS_EXAM_ATTD_REG | EXAM_MASTER_CD |
| classCode | TAMS_EXAM_RSLTS | CLASS_CD |
| classCode | TAMS_EXAM_ATTD_REG | CLASS_CD |
| examTypeCode | TAMS_EXAM_RSLTS | EXAM_TYPE_CD |
| examTypeCode | TAMS_EXAM_ATTD_REG | EXAM_TYPE_CD |
| canNum | TAMS_EXAM_RSLTS | CAN_NUM |
| canNum | TAMS_EXAM_ATTD_REG | CAN_NUM |
| candidateCode | TAMS_EXAM_RSLTS | CAN_CD |
| candidateCode | TAMS_EXAM_ATTD_REG | CAN_CD |
| examNo | TAMS_EXAM_RSLTS | EXAM_NO |
| examNo | TAMS_EXAM_ATTD_REG | EXAM_NO |
| attd | TAMS_EXAM_RSLTS | ATTD_TYP |
| attd | TAMS_EXAM_ATTD_REG | ATTD |
| examScore | TAMS_EXAM_RSLTS | GRADE |
| examAttd | TAMS_EXAM_RSLTS | EXAM_ATTD |
| createdBy | TAMS_EXAM_RSLTS | CREATED_BY |
| createdDate | TAMS_EXAM_RSLTS | CREATED_DT |
| updatedBy | TAMS_EXAM_RSLTS | UPDATED_BY |
| updatedDate | TAMS_EXAM_RSLTS | UPDATED_DT |

> **⚠ Source discrepancy flag (canNum → TAMS_EXAM_ATTD_REG.CAN_NUM):** `order2_sheets.txt` Sheet 11, row 19 (FieldNo 17) literally maps source field **`candidateCode`** → `TAMS_EXAM_ATTD_REG.CAN_NUM`, not `canNum`. This draft's table above instead shows `canNum → TAMS_EXAM_ATTD_REG.CAN_NUM` and `candidateCode → TAMS_EXAM_ATTD_REG.CAN_CD`, normalizing the mapping to be consistent with `TAMS_EXAM_RSLTS`'s unambiguous `canNum → CAN_NUM` mapping in the same sheet and with the natural key `(EXAM_MASTER_CD, CLASS_CD, CAN_NUM)` used throughout the background brief. This normalization is almost certainly correct (most likely a labeling typo in the source row), but it has **not** been confirmed against the raw source or the AMS DDL. **Must be confirmed with the source-system/workbook owner before build.**

#### Logic Summary

| Scenario | Expected Logic |
|---|---|
| Record absent | Check TAMS_EXAM_ATTD_REG by natural key (EXAM_MASTER_CD, CLASS_CD, CAN_NUM). If absent, insert into TAMS_EXAM_ATTD_REG and insert into TAMS_EXAM_RSLTS |
| Record present (re-sync) | If the (EXAM_MASTER_CD, CLASS_CD, CAN_NUM) key already exists, update the existing row in TAMS_EXAM_ATTD_REG and update the corresponding row in TAMS_EXAM_RSLTS |

> Note: this API does not carry an explicit `action` (INS/UPD/DEL) field. Insert-vs-update is decided purely by existence check on the natural key. Delete is not supported by this API.

#### Validation / Error Handling

| Scenario | Expected Behavior | Source Status |
|---|---|---|
| Missing examCode, classCode, examTypeCode, or canNum | Reject request | Proposed — not explicitly specified in source; approved as default behavior for build, confirm against actual AMS behavior in SIT |
| classCode does not exist in TAMS_CLASSES (Order 1 data missing) | Reject request and return error: class not found | Proposed — not explicitly specified in source; approved as default behavior for build, confirm against actual AMS behavior in SIT |
| examCode/examTypeCode does not exist in TAMS_EXAM_DETAILS / TAMS_EXAMS (Order 1 data missing) | Reject request and return error: exam not found | Proposed — not explicitly specified in source; approved as default behavior for build, confirm against actual AMS behavior in SIT |

> **Note on source status:** `order2_sheets.txt` (Sheet 11) contains no validation/error-handling rows for this API — unlike Order 1's sheets, which had explicit reject-text SQL (e.g. `exam-info/sync`'s "Cannot save because of invalid exam type code"). The rows above are an engineering inference (Order 2 result rows logically shouldn't attach to non-existent Order 1 shells), not a directly sourced requirement. PO has approved reject-on-missing-parent as the default build behavior (see Comments) because rejecting a bad-data write is safer than silently accepting an orphan result row, and it gives SIT/UAT a concrete scenario to test. If AMS is later found to accept orphan rows by design, this is a defect against this story, not a blocker to starting build.

---

### R.02 – Sync Training Result

| Item | Description |
|---|---|
| **API** | POST /api/v1/education/training-result/sync |
| **API Meaning** | Insert or update candidate class/training result, candidate result, education background, and MOF approval decision into AMS |
| **Trigger** | Candidate training/class result is finalized on M-Rise (including MIT completion, LDP license eligibility, or MOF approval decision) |
| **Purpose** | Sync candidate class result and, where applicable, candidate result / education-license background / MOF approval decision to AMS so downstream license and plan-code processing (Order 3) has the data it needs |
| **Method** | POST |
| **Request Type** | Object request with nested sub-objects: `agentTrainingResult`, `tamsCandidateResult`, `tamsEduBackGround` |
| **Target Tables** | TAMS_TRAINING_RSLTS, TAMS_CANDIDATE_RSLTS, TAMS_EDUCATION_BACKGROUNDS, TAMS_CANDIDATES |
| **Main Data Object Table** | TAMS_TRAINING_RSLTS (primary record for the API call) |

#### Main Data Object

| Field | Required | Description |
|---|---|---|
| classCode | Yes | Class code |
| candidateCode | No | Agent code |
| candidateNum | Yes | Candidate number |
| trainingCategoryCode | No | Training category code |
| transactionDate | No | Transaction date |
| status | No | Status |
| result | No | Training result (e.g. PASS/FAIL) |
| effectiveDate | No | Effective date |
| remarks | No | Remarks |
| activeIndicator | No | Active flag |
| isMIT | Yes (business flag) | Whether this result is for an MIT (Management-in-Training) candidate; drives which sub-objects apply |
| hasLDPLicense | No (business flag) | Whether the (non-MIT) candidate qualifies for an LDP license/education background record |
| canNum | No | Candidate number (candidate result sub-object) |
| classCd | No | Class code (candidate result sub-object) |
| clsMark | No | Class mark/score |
| productMark | No | Product mark/score |
| clsRslt | No | Class result flag |
| eduTyp | No | Education/license type |
| eduEndDt | No | Education/license end date |
| licenceDt | No | License issue date |
| statCd | No | Education background status |
| eduRmk | No | Education background remarks |
| certDecNum | No | MOF approval decision number |
| certDecDate | No | MOF approval decision date |
| updatedBy | No | Updated by |
| updatedDate | No | Updated date |

#### Sample Payload

```json
{
  "agentTrainingResult": {
    "classCode": "CLASS001",
    "candidateCode": "BF345",
    "candidateNum": "2801182011",
    "trainingCategoryCode": "MIT",
    "transactionDate": "2026-06-28T00:00:00Z",
    "status": "A",
    "result": "PASS",
    "effectiveDate": "2026-06-28T00:00:00Z",
    "remarks": "OK",
    "activeIndicator": "Y",
    "updatedBy": "system",
    "updatedDate": "2026-06-28T00:00:00Z"
  },
  "isMIT": true,
  "hasLDPLicense": false,
  "tamsCandidateResult": {
    "canNum": "2801182011",
    "classCd": "CLASS001",
    "clsMark": 80,
    "productMark": 85,
    "clsRslt": "Y"
  },
  "tamsEduBackGround": {
    "canNum": "2801182011",
    "eduTyp": "MIT",
    "eduEndDt": "2026-06-30T00:00:00Z",
    "licenceDt": "2026-06-30T00:00:00Z",
    "statCd": "A",
    "eduRmk": "LIC-001"
  },
  "certDecNum": "QD001",
  "certDecDate": "2026-06-30T00:00:00Z"
}
```

#### Field Mapping

| Source Field | Target Table | Target Field |
|---|---|---|
| classCode | TAMS_TRAINING_RSLTS | CLASS_CD |
| candidateCode | TAMS_TRAINING_RSLTS | AGT_CD |
| candidateNum | TAMS_TRAINING_RSLTS | CAN_NUM |
| trainingCategoryCode | TAMS_TRAINING_RSLTS | TRAINING_CAT_CD |
| transactionDate | TAMS_TRAINING_RSLTS | TRXN_DT |
| status | TAMS_TRAINING_RSLTS | STAT_CD |
| result | TAMS_TRAINING_RSLTS | RSLT |
| effectiveDate | TAMS_TRAINING_RSLTS | EFF_DT |
| remarks | TAMS_TRAINING_RSLTS | RMRK |
| updatedBy | TAMS_TRAINING_RSLTS | UPDATED_BY |
| updatedDate | TAMS_TRAINING_RSLTS | UPDATED_DT |
| activeIndicator | TAMS_TRAINING_RSLTS | ACTIVE_IND |
| canNum | TAMS_CANDIDATE_RSLTS | CAN_NUM |
| classCd | TAMS_CANDIDATE_RSLTS | CLS_NUM |
| clsMark | TAMS_CANDIDATE_RSLTS | CLS_MARK |
| productMark | TAMS_CANDIDATE_RSLTS | PRODUCT_MARK |
| clsRslt | TAMS_CANDIDATE_RSLTS | CLS_RSLT |
| canNum | TAMS_EDUCATION_BACKGROUNDS | CAN_NUM |
| eduTyp | TAMS_EDUCATION_BACKGROUNDS | EDU_TYP |
| eduEndDt | TAMS_EDUCATION_BACKGROUNDS | EDU_END_DT |
| licenceDt | TAMS_EDUCATION_BACKGROUNDS | LICENCE_DT |
| statCd | TAMS_EDUCATION_BACKGROUNDS | STAT_CD |
| eduRmk | TAMS_EDUCATION_BACKGROUNDS | EDU_RMRK |
| certDecNum | TAMS_CANDIDATES | MOF_APRV_NUM |
| certDecDate | TAMS_CANDIDATES | MOF_APRV_DT |

> Note: `order2_sheets.txt` Sheet 12 source uses lowercase table names (`tams_training_rslts`, etc.); this document standardizes on uppercase throughout, consistent with US1.md's convention. These resolve to the same physical tables.

#### Logic Summary

| Step | Condition | Expected Logic |
|---|---|---|
| 1 | Always | Check TAMS_TRAINING_RSLTS by natural key (CLASS_CD, CAN_NUM). If absent, insert `agentTrainingResult` into TAMS_TRAINING_RSLTS; if present, update it. |
| 2 | isMIT = true | Update TAMS_CANDIDATE_RSLTS (`tamsCandidateResult`) by natural key (CAN_NUM, CLS_NUM). Also insert into TAMS_EDUCATION_BACKGROUNDS (`tamsEduBackGround`) — ignore insert if a duplicate on (CAN_NUM, EDU_TYP) already exists — to issue the education/license credential. |
| 3 | isMIT = false AND hasLDPLicense = true | Insert into TAMS_EDUCATION_BACKGROUNDS (`tamsEduBackGround`) — ignore insert if a duplicate on (CAN_NUM, EDU_TYP) already exists. TAMS_CANDIDATE_RSLTS is not updated in this path. |
| 4 | isMIT = false AND hasLDPLicense = false | Only step 1 (TAMS_TRAINING_RSLTS) applies; no candidate-result or education-background write. |
| 5 | certDecNum and/or certDecDate present | Update TAMS_CANDIDATES.MOF_APRV_NUM / MOF_APRV_DT for the candidate (MOF approval decision), independent of steps 2–4. |

> Note: this API does not carry an explicit `action` (INS/UPD/DEL) field either. Each target table's insert-vs-update behavior is decided by its own natural-key existence check as described above. Delete is not supported by this API.

#### Validation / Error Handling

| Scenario | Expected Behavior | Source Status |
|---|---|---|
| Missing classCode or candidateNum | Reject request | Proposed — not explicitly specified in source; approved as default behavior for build, confirm against actual AMS behavior in SIT |
| classCode does not exist in TAMS_CLASSES (Order 1 data missing) | Reject request and return error: class not found | Proposed — not explicitly specified in source; approved as default behavior for build, confirm against actual AMS behavior in SIT |
| isMIT = true but tamsCandidateResult or tamsEduBackGround missing | **Reject the whole request (fail-fast)** — do not partial-write TAMS_TRAINING_RSLTS while leaving the candidate-result/education-background records silently out of sync | Undetermined by source; BE recommended reject-whole-request. **PO decision (2026-07-13): approved as the default build behavior.** Revisit only if M-Rise's client team confirms a legitimate phased/partial-submission use case — none is known today. |
| isMIT = false and hasLDPLicense = false, with tamsCandidateResult/tamsEduBackGround present but empty/null | Treat as "nothing to do" for those sub-objects (do not reject as malformed) | Proposed default; **open follow-up** — confirm with M-Rise API client owner whether these sub-objects are omitted entirely or sent null/empty in this case (see Comments) |
| Duplicate insert attempt on TAMS_EDUCATION_BACKGROUNDS (same CAN_NUM + EDU_TYP) | Ignore duplicate, do not error, do not overwrite existing record | Sourced from background brief |

> **Note on source status:** As with R.01, `order2_sheets.txt` (Sheet 12) contains no validation/error-handling rows for `training-result/sync`. The classCode/candidateNum-missing and classCode-not-found rows above are an engineering inference, approved by PO as default build behavior for the same reasons as R.01.

---

### R.03 – Inform CPA Result

| Item | Description |
|---|---|
| **API** | POST /api/v1/education/inform-cpa-result/sync |
| **API Meaning** | Update AMS to mark that a candidate's result has been sent to the AP (CPA) system |
| **Trigger** | M-Rise sends a candidate's result to the AP (CPA) system and needs to inform AMS of that status |
| **Purpose** | Keep AMS's candidate result record in sync with whether the result has already been reported to the AP (CPA) system, avoiding duplicate downstream reporting |
| **Method** | POST |
| **Request Type** | List data request |
| **Target Tables** | TAMS_CANDIDATE_RSLTS |
| **Main Data Object Table** | TAMS_CANDIDATE_RSLTS |
| **Sequencing Dependency** | Must run after `training-result/sync` has created the TAMS_CANDIDATE_RSLTS row for the same (classCode, candidateNum) — see BR-010 |

#### Main Data Object

| Field | Required | Description |
|---|---|---|
| classCode | Yes | Class code |
| candidateNum | Yes | Candidate number |
| informCpaResultId | No | Source record ID, used only for caller correlation (echoed back in response, not stored) |

#### Sample Payload

```json
[
  {
    "classCode": "HWT4200050",
    "candidateNum": "2801150919",
    "informCpaResultId": 1
  }
]
```

#### Sample Response Payload

```json
[
  {
    "informCpaResultId": 1,
    "classCode": "CLASS001",
    "candidateNum": "2801182011",
    "errorCode": "SUCCESS",
    "errorMessage": "SUCCESS"
  }
]
```

#### Field Mapping

| Source Field | Target Table | Target Field |
|---|---|---|
| classCode | TAMS_CANDIDATE_RSLTS | CLS_NUM |
| candidateNum | TAMS_CANDIDATE_RSLTS | CAN_NUM |
| informCpaResultId | N/A | N/A (echo-only, not persisted) |

> **Open follow-up:** the actual "sent to AP" status/flag column on TAMS_CANDIDATE_RSLTS is not named in the source workbook (assumed here to be a "SEND_TO_AP"-style flag based on the API's name/purpose). Requires a DDL/data-dictionary lookup before dev estimation — see Comments.

#### Logic Summary

| Scenario | Expected Logic |
|---|---|
| Matching record exists | Locate TAMS_CANDIDATE_RSLTS row by key (CLS_NUM, CAN_NUM) and set its "sent to AP" status flag. Return `informCpaResultId`, `classCode`, `candidateNum`, `errorCode`, `errorMessage` in the response. |
| Matching record does not exist | Return `errorCode`/`errorMessage` indicating failure (record not found); do not insert a new row |

#### Validation / Error Handling

| Scenario | Expected Behavior |
|---|---|
| Missing classCode or candidateNum | Reject request |
| (CLS_NUM, CAN_NUM) not found in TAMS_CANDIDATE_RSLTS | Return errorCode/errorMessage indicating failure, echoing informCpaResultId/classCode/candidateNum for caller correlation |

#### Special Logic

| Scenario | Expected Behavior |
|---|---|
| Re-sync same classCode + candidateNum | Idempotent — re-applying the "sent to AP" status must not error or duplicate; result is the same flag state |
| `informCpaResultId` reuse | `informCpaResultId` is never used for lookup, matching, or deduplication — it is not validated for uniqueness. The actual processing/idempotency key is `(classCode, candidateNum)` only; implementers should not add a uniqueness constraint or dedup cache keyed on `informCpaResultId` |

---

## 6. Business Rules & Validations

| Rule ID | Rule Name | Business Description | Impacted Flow | Example Scenario |
|---|---|---|---|---|
| BR-001 | Existence-key-based processing (no action field) | Unlike Order 1 APIs, the 3 Order 2 APIs have no explicit `action` (INS/UPD/DEL) field. Insert-vs-update is decided by a natural-key existence check per target table. Refer AC1, AC2. | exam-result/sync, training-result/sync | (EXAM_MASTER_CD, CLASS_CD, CAN_NUM) exists → update instead of insert |
| BR-002 | No duplicate on re-sync | Do not create duplicate data when re-syncing the same exam/training/candidate result. Refer AC1, AC2, AC3. | All 3 APIs | Re-sync same (CLASS_CD, CAN_NUM) → update existing row, do not insert duplicate |
| BR-003 | Integration via Learning Adapter only | M-Rise does not call AMS directly. All requests must go through Learning Adapter. Refer AC4. | All 3 APIs | M-Rise → Learning Adapter → AMS |
| BR-004 | Order 1 dependency (proposed, approved for build) | exam-result/sync and training-result/sync require the referenced class (and, for exam-result, exam) to already exist in AMS from Order 1. This is an engineering inference — the source workbook has no validation/error rows confirming AMS enforces this — but PO has approved reject-on-missing-parent as the default build behavior. Refer AC1, AC2. | exam-result/sync, training-result/sync | classCode not found in TAMS_CLASSES → reject request |
| BR-005 | Logging required | Each API call must have a log for troubleshooting and SIT/UAT evidence. Refer AC5. | All 3 APIs | Log API name, timestamp, status, error message, reference key |
| BR-006 | MIT-conditional write logic | Whether TAMS_CANDIDATE_RSLTS and TAMS_EDUCATION_BACKGROUNDS are written depends on the `isMIT` / `hasLDPLicense` flags in training-result/sync. Refer AC2. | training-result/sync API | isMIT = true → update candidate result + insert education background; isMIT = false & hasLDPLicense = true → insert education background only |
| BR-007 | Ignore-duplicate insert on education background | Insert into TAMS_EDUCATION_BACKGROUNDS must ignore (not overwrite, not error) if a duplicate (CAN_NUM, EDU_TYP) already exists. Refer AC2. | training-result/sync API | Re-sync same CAN_NUM + EDU_TYP → ignore, no duplicate row |
| BR-008 | MOF approval decision is independent of MIT path | certDecNum/certDecDate update TAMS_CANDIDATES.MOF_APRV_NUM/MOF_APRV_DT whenever present, regardless of isMIT/hasLDPLicense. Refer AC2. | training-result/sync API | certDecNum present, isMIT = false → still updates TAMS_CANDIDATES |
| BR-009 | CPA-informed status is idempotent | Re-sending the same inform-cpa-result for the same (classCode, candidateNum) must not error and must leave the status in the same "sent" state. Refer AC3. | inform-cpa-result/sync API | Re-sync same classCode+candidateNum → same result, no duplicate/error |
| BR-010 | inform-cpa-result/sync sequencing dependency | inform-cpa-result/sync updates a row on TAMS_CANDIDATE_RSLTS that is only created within Order 2 by training-result/sync (isMIT = true path). It must be called after training-result/sync has succeeded for the same (classCode, candidateNum); it has no dependency on Order 3. Refer AC3. | inform-cpa-result/sync API | inform-cpa-result/sync called before training-result/sync for a candidate → record-not-found error |
| BR-011 | MIT fail-fast on incomplete payload | If isMIT = true and tamsCandidateResult or tamsEduBackGround is missing from the payload, reject the whole request rather than partially writing TAMS_TRAINING_RSLTS. Refer AC2, AC6. | training-result/sync API | isMIT = true, tamsEduBackGround missing → reject request, no partial write |

---

## 7. Impact and Risk

| Area | Impact / Risk |
|---|---|
| Backend | AMS needs to expose 3 APIs to receive requests from Learning Adapter |
| Database | Impact on TAMS_EXAM_ATTD_REG, TAMS_EXAM_RSLTS, TAMS_TRAINING_RSLTS, TAMS_CANDIDATE_RSLTS, TAMS_EDUCATION_BACKGROUNDS, TAMS_CANDIDATES |
| Reference Tables | Dependency on TAMS_CLASSES, TAMS_EXAM_DETAILS, TAMS_EXAMS (all populated by Order 1) |
| Integration Risk | If Learning Adapter is down, all Order 2 sync will be blocked, stalling Order 3 license/plan-code processing |
| Data Integrity Risk | Because there is no `action` field, an incorrect existence-key check (wrong natural key) could silently overwrite the wrong row or create duplicates |
| Validation Risk | Order 1 class/exam existence checks and the MIT fail-fast rule (BR-004, BR-011) are engineering inferences approved by PO as default build behavior, not confirmed source requirements — verify against actual AMS behavior in SIT |
| Downstream Impact | If Order 2 sync fails, Order 3 (license/plan code) and Order 4 (e-certificate) will not have sufficient result data to process |

---

## 8. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Performance / Throughput | N/A – Event-driven, not large batch |
| Security / Privacy | N/A within the scope of this US |
| Audit / Logging | Each API call must log: API name, request timestamp, processing status, error message if any, reference key |
| Availability / Resilience | Learning Adapter must operate stably to ensure successful sync |
| Idempotency | Re-syncing the same result data must not create duplicate records or change the outcome on repeated calls |
| Error Traceability | Error messages must be clear enough for BA/Tester to trace failure root cause during SIT/UAT |

---

## 9. Dependencies / References

| Dependency | Description |
|---|---|
| M-Rise | Source system where exam attendance/result, training result, and CPA-informed status originate |
| Learning Adapter | Middleware that receives data from M-Rise and calls AMS APIs |
| AMS | Target system where candidate result data is stored |
| TAMS_CLASSES | Reference table from Order 1, expected to exist before exam-result/sync and training-result/sync can process (proposed check, approved for build) |
| TAMS_EXAM_DETAILS / TAMS_EXAMS | Reference tables from Order 1, expected to exist before exam-result/sync can process (proposed check, approved for build) |
| Order 1 | Provides class/exam/template shell data that Order 2 result data attaches to |
| training-result/sync (within Order 2) | inform-cpa-result/sync depends on training-result/sync having already created the TAMS_CANDIDATE_RSLTS row for the same (classCode, candidateNum) — see BR-010 |
| Order 3 | Depends on training result / education background / MOF approval data from Order 2 to process license and plan code |
| Order 4 | Depends on final result data (directly or via Order 3) to process e-certificate if applicable |

---

## 13. Acceptance Criteria

### AC1 – Sync Exam Attendance & Result Successfully

**Given** a candidate's exam attendance/result is recorded on M-Rise, and the referenced class/exam already exist in AMS from Order 1

**When** Learning Adapter calls API: `POST /api/v1/education/exam-result/sync`

**Then** AMS must:
- Receive the request from Learning Adapter.
- Check existence of (EXAM_MASTER_CD, CLASS_CD, CAN_NUM) in TAMS_EXAM_ATTD_REG.
- Insert into TAMS_EXAM_ATTD_REG and TAMS_EXAM_RSLTS if the key does not exist.
- Update TAMS_EXAM_ATTD_REG and TAMS_EXAM_RSLTS if the key already exists.
- Map fields correctly: examCode → EXAM_MASTER_CD, classCode → CLASS_CD, examTypeCode → EXAM_TYPE_CD, canNum → CAN_NUM, examScore → GRADE, attd → ATTD_TYP / ATTD

**And** do not create duplicate data when re-syncing the same (examCode, classCode, canNum).

### AC2 – Sync Training Result Successfully

**Given** a candidate's training/class result is finalized on M-Rise

**When** Learning Adapter calls API: `POST /api/v1/education/training-result/sync`

**Then** AMS must:
- Receive the request from Learning Adapter.
- Insert or update TAMS_TRAINING_RSLTS by key (CLASS_CD, CAN_NUM) using the `agentTrainingResult` object.
- If `isMIT` = true, update TAMS_CANDIDATE_RSLTS by key (CAN_NUM, CLS_NUM) and insert into TAMS_EDUCATION_BACKGROUNDS (ignore if duplicate on CAN_NUM + EDU_TYP). If `tamsCandidateResult` or `tamsEduBackGround` is missing, reject the whole request (BR-011) instead of partial-writing.
- If `isMIT` = false and `hasLDPLicense` = true, insert into TAMS_EDUCATION_BACKGROUNDS only (ignore if duplicate).
- If `certDecNum`/`certDecDate` are present, update TAMS_CANDIDATES.MOF_APRV_NUM / MOF_APRV_DT.
- Map fields correctly: classCode → CLASS_CD, candidateNum → CAN_NUM, clsMark → CLS_MARK, eduTyp → EDU_TYP, certDecNum → MOF_APRV_NUM

**And** do not create duplicate education-background records for the same (canNum, eduTyp).

**And** result data is ready for license/plan-code processing in Order 3.

### AC3 – Inform CPA Result Successfully

**Given** a candidate's result has been sent to the AP (CPA) system on M-Rise, and `training-result/sync` has already created the TAMS_CANDIDATE_RSLTS row for that (classCode, candidateNum)

**When** Learning Adapter calls API: `POST /api/v1/education/inform-cpa-result/sync`

**Then** AMS must:
- Receive classCode, candidateNum, and informCpaResultId from Learning Adapter.
- Locate the matching TAMS_CANDIDATE_RSLTS row by (CLS_NUM, CAN_NUM) and set its "sent to AP" status.
- Return a response echoing informCpaResultId, classCode, candidateNum, plus errorCode/errorMessage.

**And** re-sending the same classCode + candidateNum must not error and must not change the outcome.

### AC4 – Integration Must Go Through Learning Adapter

**Given** M-Rise needs to sync Order 2 data to AMS

**When** one of the three triggers occurs:
- Candidate exam attendance/result recorded
- Candidate training/class result finalized
- Candidate result sent to AP (CPA) system

**Then**:
- M-Rise sends data through Learning Adapter.
- Learning Adapter calls AMS API.
- AMS does not receive requests directly from M-Rise.

### AC5 – Error Handling and Logging

**Given** Learning Adapter calls AMS API

**When** AMS processes successfully or fails

**Then** the system records a log including:
- API name
- Request timestamp
- Processing status
- Error message if any
- Reference key: classCode, canNum / candidateNum, examCode (where applicable)

### AC6 – Validation Failure Handling

**Given** request is invalid or referenced Order 1 master data is missing

**When** AMS processes the request

**Then** AMS must return an appropriate error and not write invalid data into the database.

| Scenario | Expected Error / Behavior | Source Status |
|---|---|---|
| Missing examCode, classCode, examTypeCode, or canNum | Reject request | Proposed, approved for build |
| Missing classCode or candidateNum (training-result/sync) | Reject request | Proposed, approved for build |
| Missing classCode or candidateNum (inform-cpa-result/sync) | Reject request | Sourced |
| classCode not found in TAMS_CLASSES (Order 1 missing) | Reject request, class not found | Proposed, approved for build |
| examCode/examTypeCode not found in TAMS_EXAM_DETAILS / TAMS_EXAMS (Order 1 missing) | Reject request, exam not found | Proposed, approved for build |
| isMIT = true but tamsCandidateResult or tamsEduBackGround missing | Reject whole request (fail-fast), no partial write | Proposed, approved for build (BR-011) |
| (classCode, candidateNum) not found in TAMS_CANDIDATE_RSLTS (inform-cpa-result/sync) | Return errorCode/errorMessage indicating failure | Sourced (Sheet 7 response-echo mapping incl. errorCode/errorMessage) |
| Duplicate insert attempt on TAMS_EDUCATION_BACKGROUNDS | Ignore, no error, no duplicate | Sourced from background brief |

> **Note:** Rows marked "Proposed, approved for build" are engineering inferences, not requirements found directly in `order2_sheets.txt` (Sheets 7, 11, 12 contain no validation/error rows for exam-result/sync or training-result/sync). PO has approved them as the default build behavior for SIT/UAT testability and to avoid silent bad-data writes; they should be confirmed against actual AMS behavior during SIT and revised if AMS proves to work differently.

---

## Comments

**PO Review** (2026-07-13):
> **Approved.**
> 1. **Scope (Order 2 vs. Order 3 for inform-cpa-result/sync):** Confirmed in Order 2. It only sets a status flag on TAMS_CANDIDATE_RSLTS, and that row is created within Order 2 itself (training-result/sync, MIT path) — not by Order 3's license/plan-code grant. Keeping it in Order 2 avoids an unnecessary cross-Order dependency. Added an explicit sequencing pre-condition (BR-010): inform-cpa-result/sync must run after training-result/sync for the same (classCode, candidateNum).
> 2. **Proposed validation rules (classCode/exam existence checks, missing-required-field checks):** kept in the story, explicitly marked "Proposed." SIT/UAT needs concrete, testable pass/fail scenarios, and rejecting an orphan result row is a safer default than silently writing one against a non-existent Order 1 shell. If AMS turns out not to enforce this, that's a defect to fix against this story, not a reason to withhold approval today.
> 3. **isMIT = true with a missing sub-object:** adopted BE's fail-fast recommendation (reject whole request) as the default build behavior (BR-011). Partial writes that leave an MIT candidate's license/education record silently out of sync are worse than a rejected sync that can be retried. Revisit if M-Rise's client team later confirms a legitimate phased-submission use case.
> 4. **Remaining open items** (exact "sent to AP" column name; whether empty sub-objects are omitted vs. sent null when isMIT/hasLDPLicense are both false) do not block approval — logged as follow-ups below, same as US1 shipped with Nguyen Bien Trung's open M-Rise-origin-record question.
> 5. Format/depth checked against US1.md — consistent.

**BE** (2026-07-13):
> Field mappings and MIT/LDP/MOF conditional logic verified clean against `order2_sheets.txt` Sheets 7, 11, 12. Two follow-up action items, non-blocking:
> - Confirm the actual "sent to AP" flag/column on TAMS_CANDIDATE_RSLTS against the DDL/data dictionary (workbook doesn't name it) — needed before dev estimation for R.03.
> - Confirm with the M-Rise API client owner whether `tamsCandidateResult`/`tamsEduBackGround` are omitted entirely or sent null/empty when isMIT = false and hasLDPLicense = false — affects request-schema validation.

**BA** (2026-07-13):
> The `candidateCode`/`canNum` → `TAMS_EXAM_ATTD_REG.CAN_NUM` mapping in R.01 is a probable source-sheet labeling typo (Sheet 11, row 19 literally reads `candidateCode`); normalized in this draft for consistency with the natural key, but must be confirmed against the AMS DDL or workbook owner before build.
