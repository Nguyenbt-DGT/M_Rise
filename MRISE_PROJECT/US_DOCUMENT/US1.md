# [M-RISE] AD/PD - US1 for 13 API | Order 1 – Sync Class Setup Information from M-Rise to AMS via Learning Adapter

| Field | Value |
|---|---|
| **Issue Key** | VNEL-2157 |
| **Type** | Story |
| **Project** | VN M-RISE (VNEL) |
| **Status** | TODO |
| **Priority** | Medium |
| **Assignee** | Nguyen Bien Trung |
| **Reporter** | Nguyen Bien Trung |
| **Created** | 2026-07-09 |
| **Updated** | 2026-07-10 |

---

## 1. User Story Summary

| Item | Description |
|---|---|
| **User Story** | As **AMS System**, I want to receive class setup information from **M-Rise** through **Learning Adapter**, so that AMS can initialize required training data including **class template**, **exam information**, and **class information** for candidate/agent learning management and downstream processing. |
| **Actor** | AMS System, Learning Adapter, M-Rise |
| **Flow** | M-Rise → Learning Adapter → AMS |
| **Screen Demo** | N/A – Backend Integration |
| **Main Objective** | Sync Order 1 setup data from M-Rise to AMS before Order 2, Order 3, and Order 4 processing. |

---

## 2. Trigger / Pre-condition / Post-condition

| Type | Description |
|---|---|
| **Trigger** | Create/activate class template on M-Rise · Successfully import exam code in class · Approve training plan on M-Rise |
| **Pre-condition** | M-Rise has class template / exam / class data ready to sync · Learning Adapter service is running · AMS API endpoints are ready to receive requests · Related master data already exists in AMS if API has condition checks |
| **Post-condition** | AMS successfully receives and stores class template, exam info, class info · Data is ready for Order 2: attendance, training result · Data is ready for downstream flow: license, plan code, e-certificate · Log is recorded for each API call |

---

## 3. Integration Flow

> **Important Note:**
> M-Rise **does not call AMS directly**.
> All Order 1 data must go through **Learning Adapter**.

---

## 4. Scope

### 4.1 In Scope

| No. | API | Purpose | Trigger |
|---|---|---|---|
| 1 | POST /api/v1/education/sync-class-template | Sync active class template to AMS | When a class template is created/activated on M-Rise |
| 2 | POST /api/v1/education/exam-info/sync | Sync exam information to AMS | After exam code is successfully imported in class |
| 3 | POST /api/v1/sync/sync-class | Sync class information to AMS | When a training plan is approved on M-Rise |

### 4.2 Out of Scope

| No. | Item | Belong To |
|---|---|---|
| 1 | Sync attendance result | Order 2 |
| 2 | Sync training result | Order 2 |
| 3 | Grant license | Order 3 |
| 4 | Assign plan code | Order 3 |
| 5 | Refresh plan code | Order 3 |
| 6 | Upload e-certificate | Order 4 |

---

## 5. Requirement Details

### R.01 – Sync Class Template

| Item | Description |
|---|---|
| **API** | POST /api/v1/education/sync-class-template |
| **API Meaning** | Insert / update / delete class template into AMS |
| **Trigger** | M-Rise creates/activates a class template |
| **Purpose** | Sync class template information and related plan codes to AMS |
| **Method** | POST |
| **Request Type** | List data request |
| **Target Tables** | TAMS_TEMPLS, TAMS_TEMPL_PRODUCTS |

#### Main Data Object

| Field | Required | Description |
|---|---|---|
| templateCode | Yes | Template code |
| templateName | No | Template name |
| effectiveDate | No | Effective date |
| endDate | No | End date |
| remarks | No | Remarks |
| statusCode | No | Template status |
| trainingCategoryCode | No | Training category code |
| noRequiredSession | No | Number of required sessions / full session attendance check |
| planCodeList | No | List of related plan codes |
| action | Recommended | Processing action: INS, UPD, DEL |
| createdBy | No | Created by |
| createdDate | No | Created date |
| updatedBy | No | Updated by |
| updatedDate | No | Updated date |

#### Sample Payload

```json
[
  {
    "templateCode": "string",
    "templateName": "string",
    "effectiveDate": "2026-05-12T09:46:24.583Z",
    "endDate": "2026-05-12T09:46:24.583Z",
    "remarks": "string",
    "statusCode": "string",
    "trainingCategoryCode": "string",
    "noRequiredSession": 1073741824,
    "planCodeList": ["string"],
    "action": "string",
    "updatedBy": "string",
    "updatedDate": "2026-05-12T09:46:24.583Z",
    "createdBy": "string",
    "createdDate": "2026-05-12T09:46:24.583Z"
  }
]
```

#### Field Mapping

| Source Field | Target Table | Target Field |
|---|---|---|
| templateCode | TAMS_TEMPLS | TEMPL_CD |
| templateCode | TAMS_TEMPL_PRODUCTS | TEMPL_CD |
| templateName | TAMS_TEMPLS | TEMPL_NM |
| remarks | TAMS_TEMPLS | REMARKS |
| statusCode | TAMS_TEMPLS | STAT_CD |
| effectiveDate | TAMS_TEMPLS | EFF_DT |
| endDate | TAMS_TEMPLS | END_DT |
| trainingCategoryCode | TAMS_TEMPLS | TRAINING_CAT_CD |
| trainingCategoryCode | TAMS_TEMPL_PRODUCTS | TRAINING_CAT_CD |
| noRequiredSession | TAMS_TEMPLS | FULL_SEC_ATTD_CHK |
| createdDate | TAMS_TEMPLS | CREATED_DT |
| createdBy | TAMS_TEMPLS | CREATED_BY |
| updatedDate | TAMS_TEMPLS | UPDATED_DT |
| updatedBy | TAMS_TEMPLS | UPDATED_BY |
| planCodeList[n] | TAMS_TEMPL_PRODUCTS | PLAN_CD |

#### Logic Summary

| Action | Expected Logic |
|---|---|
| INS | If templateCode does not exist, insert into TAMS_TEMPLS, then insert each plan code into TAMS_TEMPL_PRODUCTS |
| UPD | If templateCode already exists, update TAMS_TEMPLS, delete old plan codes in TAMS_TEMPL_PRODUCTS, then insert new plan codes |
| DEL / Other | Delete data in TAMS_TEMPL_PRODUCTS, then delete data in TAMS_TEMPLS |

---

### R.02 – Sync Exam Information

| Item | Description |
|---|---|
| **API** | POST /api/v1/education/exam-info/sync |
| **API Meaning** | Sync exam information data from external system into AMS database |
| **Trigger** | M-Rise successfully imports exam code in class |
| **Purpose** | Sync exam information to AMS to support exam result processing in Order 2 |
| **Method** | POST |
| **Request Type** | List data request |
| **Target Tables** | TAMS_EXAM_DETAILS |
| **Reference / Check Table** | TAMS_EXAMS used to validate EXAM_TYPE_CD |

#### Main Data Object

| Field | Required | Description |
|---|---|---|
| code | Yes | Exam master code |
| examTypeCd | Yes | Exam type code |
| status | No | Exam status |
| examDate | No | Exam date |
| timeInd | No | Time indicator |
| action | Recommended | Processing action: INS, UPD, DEL |
| createdBy | No | Created by |
| createdDate | No | Created date |
| updatedBy | No | Updated by |
| updatedDate | No | Updated date |

#### Sample Payload

```json
[
  {
    "code": "string",
    "examTypeCd": "string",
    "status": "string",
    "examDate": "2026-05-12T08:45:21.095Z",
    "timeInd": "string",
    "action": "string",
    "updatedBy": "string",
    "updatedDate": "2026-05-12T08:45:21.095Z",
    "createdBy": "string",
    "createdDate": "2026-05-12T08:45:21.095Z"
  }
]
```

#### Field Mapping

| Source Field | Target Table | Target Field |
|---|---|---|
| code | TAMS_EXAM_DETAILS | EXAM_MASTER_CD |
| examTypeCd | TAMS_EXAM_DETAILS | EXAM_TYPE_CD |
| status | TAMS_EXAM_DETAILS | STAT_CD |
| examDate | TAMS_EXAM_DETAILS | EXAM_DT |
| timeInd | TAMS_EXAM_DETAILS | TIME_IND |
| createdBy | TAMS_EXAM_DETAILS | CREATED_BY |
| createdDate | TAMS_EXAM_DETAILS | CREATED_DT |
| updatedBy | TAMS_EXAM_DETAILS | UPDATED_BY |
| updatedDate | TAMS_EXAM_DETAILS | UPDATED_DT |

#### Logic Summary

| Action | Expected Logic |
|---|---|
| INS | Check if examTypeCd exists in TAMS_EXAMS. If valid and code does not exist in TAMS_EXAM_DETAILS, then insert |
| UPD | Check if examTypeCd exists in TAMS_EXAMS. If code already exists, then update TAMS_EXAM_DETAILS |
| DEL | Delete record in TAMS_EXAM_DETAILS by code |

#### Validation / Error Handling

| Scenario | Expected Behavior |
|---|---|
| examTypeCd does not exist in TAMS_EXAMS | Reject request and return error: Cannot save because of invalid exam type code |
| Delete exam but exam has relationship with other tables | Reject delete and return error: Cannot delete because of relationship with other tables |
| Other error during delete | Log error and raise error |

---

### R.03 – Sync Class Information

| Item | Description |
|---|---|
| **API** | POST /api/v1/sync/sync-class |
| **API Meaning** | Sync training class data from external system into AMS |
| **Trigger** | Training plan is approved on M-Rise |
| **Purpose** | Sync class information to AMS to initialize training classes for candidate/agent |
| **Method** | POST |
| **Request Type** | List data request |
| **Target Table** | TAMS_CLASSES |
| **Reference / Check Tables** | TAMS_TEMPLS, TAMS_TRAINING_CATS |

#### Main Data Object

| Field | Required | Description |
|---|---|---|
| classCode | Yes | Class code |
| className | No | Class name |
| startDate | No | Start date |
| endDate | No | End date |
| status | No | Class status |
| trainerCode | No | Trainer code |
| branchCode | No | Branch code |
| classType | No | Class type / training category |
| trainingCatVersion | No | Training category version |
| createdDate | No | Created date |
| updatedDate | No | Updated date |
| createdBy | No | Created by |
| updatedBy | No | Updated by |
| action | Recommended | Processing action: INS, UPD, DEL |
| mitType | No | MIT type |
| templateCode | No | Template code |
| lockDataEntry | No | Lock data entry flag |

#### Sample Payload

```json
[
  {
    "classCode": "string",
    "className": "string",
    "startDate": "2026-05-12T09:25:08.922Z",
    "endDate": "2026-05-12T09:25:08.922Z",
    "status": "s",
    "trainerCode": "string",
    "branchCode": "string",
    "classType": "string",
    "trainingCatVersion": "string",
    "createdDate": "2026-05-12T09:25:08.922Z",
    "updatedDate": "2026-05-12T09:25:08.922Z",
    "createdBy": "string",
    "updatedBy": "string",
    "action": "string",
    "mitType": "string",
    "templateCode": "string",
    "lockDataEntry": "string"
  }
]
```

#### Field Mapping

| Source Field | Target Table | Target Field |
|---|---|---|
| classCode | TAMS_CLASSES | CLS_NUM |
| className | TAMS_CLASSES | CLS_NM |
| startDate | TAMS_CLASSES | STRT_DT |
| endDate | TAMS_CLASSES | END_DT |
| status | TAMS_CLASSES | STATUS |
| trainerCode | TAMS_CLASSES | TRAINER_CD |
| branchCode | TAMS_CLASSES | BR_CODE |
| classType | TAMS_CLASSES | CLS_TYP |
| trainingCatVersion | TAMS_CLASSES | TRAINING_CAT_VERSION |
| createdBy | TAMS_CLASSES | CREATED_BY |
| createdDate | TAMS_CLASSES | CREATED_DT |
| updatedBy | TAMS_CLASSES | UPDATED_BY |
| updatedDate | TAMS_CLASSES | UPDATED_DT |
| mitType | TAMS_CLASSES | MIT_TYP |
| templateCode | TAMS_CLASSES | TEMPL_CD |
| lockDataEntry | TAMS_CLASSES | LOCK_DATA_ENTRY |

#### Logic Summary

| Action | Expected Logic |
|---|---|
| INS | If classCode does not exist, insert new class into TAMS_CLASSES |
| UPD | If classCode already exists, update class information in TAMS_CLASSES |
| DEL | If classCode already exists, delete class in TAMS_CLASSES |

#### Validation / Condition Check

| Check | Source | Expected Behavior |
|---|---|---|
| templateCode exists | TAMS_TEMPLS | If not exists and action is not DEL, return error "Template code does not exist in CAS DB" |
| classType exists | TAMS_TRAINING_CATS | If not exists and action is not DEL, return error "Class type does not exist in CAS DB" |

#### Special Logic

| Scenario | Expected Behavior |
|---|---|
| mitType = F when inserting class | AMS creates an additional sub-class with suffix `_C` and mitType = C |
| Re-sync same classCode | Do not create duplicate class, process update on existing class |

---

## 6. Business Rules & Validations

| Rule ID | Rule Name | Business Description | Impacted Flow | Example Scenario |
|---|---|---|---|---|
| BR-001 | Action-based processing | APIs process based on action: INS, UPD, DEL. Refer AC1, AC2, AC3. | All 3 APIs | action = INS → insert new record |
| BR-002 | No duplicate on re-sync | Do not create duplicate data when re-syncing the same template/exam/class code. Refer AC1, AC2, AC3. | All 3 APIs | Re-sync same templateCode → update or replace plan code, do not insert duplicate |
| BR-003 | Integration via Learning Adapter only | M-Rise does not call AMS directly. All requests must go through Learning Adapter. Refer AC4. | All 3 APIs | M-Rise → Learning Adapter → AMS |
| BR-004 | PlanCode list to individual records | Each element in planCodeList must be inserted as a separate record in TAMS_TEMPL_PRODUCTS. Refer AC1. | sync-class-template API | planCodeList has 3 items → insert 3 records |
| BR-005 | Logging required | Each API call must have a log for troubleshooting and SIT/UAT evidence. Refer AC5. | All 3 APIs | Log API name, timestamp, status, error message, reference key |
| BR-006 | Exam type validation | When syncing exam, examTypeCd must exist in TAMS_EXAMS. | exam-info/sync API | examTypeCd invalid → reject request |
| BR-007 | Class dependency validation | When syncing class, templateCode and classType must exist if action is not DEL. | sync-class API | Template does not exist → reject request |
| BR-008 | Delete relationship handling | When deleting exam, if exam has relationship with other tables, deletion is not allowed. | exam-info/sync API | Delete exam being referenced → return relationship error |

---

## 7. Impact and Risk

| Area | Impact / Risk |
|---|---|
| Backend | AMS needs to expose 3 APIs to receive requests from Learning Adapter |
| Database | Impact on TAMS_TEMPLS, TAMS_TEMPL_PRODUCTS, TAMS_EXAM_DETAILS, TAMS_CLASSES |
| Reference Tables | Dependency on TAMS_EXAMS, TAMS_TRAINING_CATS, TAMS_TEMPLS |
| Integration Risk | If Learning Adapter is down, all Order 1 sync will be blocked |
| Data Integrity Risk | If INS/UPD/DEL processing is incorrect, duplicate or data loss may occur |
| Validation Risk | If master data does not exist, API will fail when syncing exam/class |
| Downstream Impact | If Order 1 sync fails, Order 2/3/4 may not have sufficient data for processing |

---

## 8. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Performance / Throughput | N/A – Event-driven, not large batch |
| Security / Privacy | N/A within the scope of this US |
| Audit / Logging | Each API call must log: API name, request timestamp, processing status, error message if any, reference key |
| Availability / Resilience | Learning Adapter must operate stably to ensure successful sync |
| Idempotency | Re-syncing the same data must not create duplicate records |
| Error Traceability | Error messages must be clear enough for BA/Tester to trace failure root cause during SIT/UAT |

---

## 9. Dependencies / References

| Dependency | Description |
|---|---|
| M-Rise | Source system where template, exam, class data originates |
| Learning Adapter | Middleware that receives data from M-Rise and calls AMS APIs |
| AMS | Target system where training setup data is stored |
| TAMS_EXAMS | Master/reference table to validate exam type |
| TAMS_TEMPLS | Master/reference table to validate template when syncing class |
| TAMS_TRAINING_CATS | Master/reference table to validate class type |
| Order 2 | Depends on class/exam data from Order 1 to process attendance and training result |
| Order 3 | Depends on training result/license condition after Order 1 & 2 |
| Order 4 | Depends on results from previous steps to process e-certificate if applicable |

---

## 13. Acceptance Criteria

### AC1 – Sync Class Template Successfully

**Given** a class template is created or activated on M-Rise

**When** Learning Adapter calls API: `POST /api/v1/education/sync-class-template`

**Then** AMS must:
- Receive the request from Learning Adapter.
- Process based on action: INS, UPD, or DEL.
- Record data into the correct tables: TAMS_TEMPLS, TAMS_TEMPL_PRODUCTS
- Insert each element in planCodeList as a separate record.
- Map fields correctly: templateCode → TEMPL_CD, templateName → TEMPL_NM, trainingCategoryCode → TRAINING_CAT_CD, planCodeList[n] → PLAN_CD

**And** do not create duplicate data when re-syncing the same templateCode.

### AC2 – Sync Exam Information Successfully

**Given** exam code has been successfully imported in class on M-Rise

**When** Learning Adapter calls API: `POST /api/v1/education/exam-info/sync`

**Then** AMS must:
- Receive exam information from Learning Adapter.
- Validate examTypeCd exists in TAMS_EXAMS.
- Create new exam if code does not exist in TAMS_EXAM_DETAILS.
- Update exam if code already exists in TAMS_EXAM_DETAILS.
- Map fields correctly: code → EXAM_MASTER_CD, examTypeCd → EXAM_TYPE_CD, status → STAT_CD, examDate → EXAM_DT, timeInd → TIME_IND

**And** do not create duplicate exam for the same code.

**And** exam data is ready for exam result processing in Order 2.

### AC3 – Sync Class Information Successfully

**Given** training plan has been approved on M-Rise

**When** Learning Adapter calls API: `POST /api/v1/sync/sync-class`

**Then** AMS must:
- Receive class information from Learning Adapter.
- Validate templateCode exists in TAMS_TEMPLS if action is not DEL.
- Validate classType exists in TAMS_TRAINING_CATS if action is not DEL.
- Create new class if classCode does not exist.
- Update class if classCode already exists.
- Delete class if action = DEL.
- Map fields correctly: classCode → CLS_NUM, className → CLS_NM, startDate → STRT_DT, endDate → END_DT, templateCode → TEMPL_CD, mitType → MIT_TYP, lockDataEntry → LOCK_DATA_ENTRY

**And** do not create duplicate class when re-syncing the same classCode.

### AC4 – Integration Must Go Through Learning Adapter

**Given** M-Rise needs to sync Order 1 data to AMS

**When** one of the three triggers occurs:
- Create/activate class template
- Import exam code
- Approve training plan

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
- Reference key: templateCode, code / examCode, classCode

### AC6 – Validation Failure Handling

**Given** request is invalid or related master data is missing

**When** AMS processes the request

**Then** AMS must return an appropriate error and not write invalid data into the database.

| Scenario | Expected Error / Behavior |
|---|---|
| Missing templateCode | Reject request |
| Missing code or examTypeCd | Reject request |
| Missing classCode | Reject request |
| Invalid examTypeCd | Cannot save because of invalid exam type code |
| Invalid templateCode when syncing class | Template code does not exist in CAS DB |
| Invalid classType when syncing class | Class type does not exist in CAS DB |
| Delete exam with relationship | Cannot delete because of relationship with other tables |

---

## Comments

**Nguyen Bien Trung** (2026-07-10):
> Review US:
> 1. khi insert xuống AMS → làm sao phân biệt được của system M-Rise. (check với anh Vũ).
> 2.
