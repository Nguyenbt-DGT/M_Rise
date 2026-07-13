# SQUAD_TEAM — M-Rise ↔ AMS Integration Squad

This folder defines a lightweight 3-role agent squad for the M-Rise project: **Product Owner (PO)**, **Business Analyst (BA)**, and **Backend Engineer (BE)**. Each role has its own persona brief so it can be invoked independently (e.g. via the Claude Code `Agent` tool) and still converge on the same final artifact: an approved **User Story** under `MRISE_PROJECT/US_DOCUMENT/`.

## Why this exists

M-Rise is replacing two legacy e-learning systems and must sync data into the CAS system (**AMS**) through the **Learning Adapter**. The full scope is 13 APIs grouped into 4 Orders (see `MRISE_PROJECT/REQUIREMENT/Requirement for 13 API.md`), each Order becoming one User Story (`US1.md`…`US4.md`). `US1.md` (Order 1 — Sync Class Setup Information) is the reference template for structure, depth, and tone that every future US should match.

## Shared inputs (all three roles read these first)

| Doc | Path | Purpose |
|---|---|---|
| API detail workbook | `MRISE_PROJECT/API_DOCUMENT/API_M-Rise to_AMS.xlsx` | Field-level contract for each of the 13 APIs (requests, mappings, target CAS/AMS tables) |
| Working file | `MRISE_PROJECT/API_DOCUMENT/Working file for 13 API M_RISE TO AMS.xlsx` | Working/tracking copy of the API set |
| Requirement brief | `MRISE_PROJECT/REQUIREMENT/Requirement for 13 API.md` | States the target: 4 User Stories, one per Order, across the 13 APIs |
| Reference US | `MRISE_PROJECT/US_DOCUMENT/US1.md` | Approved Order 1 story — the format/quality bar for all new stories |

## How the squad discusses

Each role is a separate persona (`PO.md`, `BA.md`, `BE.md` in this folder). They are **not** meant to be merged into one prompt — invoke them as independent agents so each stays in its own lane, then relay outputs between them. A typical cycle for drafting or reviewing a User Story:

1. **BA** reads the API workbook + requirement brief for the target Order, drafts the User Story body (summary, flow, scope, requirement details, field mapping, business rules, AC) following the `US1.md` template.
2. **BE** reviews the BA draft against the raw API contract: checks field mappings, target tables, INS/UPD/DEL logic, validation rules, and flags anything technically ambiguous or infeasible back to BA as open questions.
3. **PO** reviews both for business value, scope boundaries (in/out), priority, and open-question resolution; makes the final call on ambiguous points (e.g. business rules the requirement doc doesn't fully specify) and approves/rejects.
4. If PO raises a blocking question, it routes back to BA (content) or BE (technical) — repeat until PO approves.
5. Approved output is saved to `MRISE_PROJECT/US_DOCUMENT/US<N>.md`.

Each persona file below states its role's responsibilities, what it should read, what it hands off, and to whom.

## Roster

| File | Role | One-line focus |
|---|---|---|
| [PO.md](PO.md) | Product Owner | Business value, scope, prioritization, final approval |
| [BA.md](BA.md) | Business Analyst | Requirement analysis, User Story drafting, field mapping, AC |
| [BE.md](BE.md) | Backend Engineer | Technical feasibility, API/data contract validation, risk flags |
