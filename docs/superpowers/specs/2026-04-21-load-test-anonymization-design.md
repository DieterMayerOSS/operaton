# Load Test Scenario Anonymization

**Date:** 2026-04-21  
**Status:** Approved

## Problem

The `qa/load-test` module contains BPMN/DMN process files and Java test code that model a recognizable real-world domain: a credit eligibility check with Brazilian CPF/CNPJ numbers, Portuguese task names, and employment-service API endpoints. The original author requested this scenario be made anonymous before merging into the public repository.

## Approach

Full domain-neutral rename. Replace all domain-specific terminology, IDs, and file names with generic, abstract equivalents (item approval workflow). Preserve structural complexity and functional correctness.

> **XML reference rule:** Any renamed BPMN/DMN element ID requires updating **all** XML references that point to it, including: `processRef`, `flowNodeRef`, `incoming`/`outgoing` text, `sourceRef`/`targetRef`, `default`, `errorRef`, BPMNDI `bpmnElement`, and DMNDI `dmnElementRef`.

> **Gateway semantics:** Do not swap or invert gateway branches during anonymization. Preserve existing routing (e.g., the `true` condition leading to the denial path).

---

## File Renames

| Current | New |
|---|---|
| `credit-eligibility.bpmn` | `item-approval.bpmn` |
| `dmn-policy-age.dmn` | `dmn-rule-a.dmn` |
| `dmn-policy-tenure.dmn` | `dmn-rule-b.dmn` |
| `http-only-process.bpmn` | unchanged (internal URLs updated) |
| `script-only-process.bpmn` | unchanged (internal script data updated) |
| `simple-process.bpmn` | unchanged |
| `pure-js-process.bpmn` | unchanged |

---

## `item-approval.bpmn` Changes (formerly `credit-eligibility.bpmn`)

### Process / Structural

| Current | Replacement |
|---|---|
| Process ID/name: `credit-eligibility` / "Credit Eligibility" | `item-approval` / "Item Approval" |
| Participant: "Credit Eligibility" | "Item Approval" |
| Lane: "Employment Service" | "External Service" |
| Lane: "Credit Policy" | "Approval Policy" |

### Task / Gateway Names and IDs

| Current name (id) | Replacement name (id) |
|---|---|
| "Consulta Dados do Cliente" (`http-task-customer`) | "Fetch Item Details" (`http-task-item`) |
| "Check Termination" (`http-task-termination`) | "Validate Status A" (`http-task-status-a`) |
| "Check Leave" (`http-task-leave`) | "Validate Status B" (`http-task-status-b`) |
| "Fee Policy" (`decision-task-fee`) | "Fee Evaluation" (id unchanged) |
| "Age Policy" (`decision-task-age`) | "Rule A Evaluation" (`decision-task-rule-a`) |
| "Tenure Policy" (`decision-task-tenure`) | "Rule B Evaluation" (`decision-task-rule-b`) |
| "Terminated?" (`gtw-termination`) | "Status A?" (`gtw-status-a`) |
| "On Leave?" (`gtw-leave`) | "Status B?" (`gtw-status-b`) |
| "Age Approved?" (`ex-gtw-age`) | "Rule A Approved?" (`ex-gtw-rule-a`) |
| "Tenure Approved?" (`gtw-tenure`) | "Rule B Approved?" (`gtw-rule-b`) |
| "General Employee?" (`gtw-category`) | "Category Valid?" (id unchanged) |
| "Deny: Termination" (`task-set-denied-termination`) | "Deny: Status A" (`task-set-denied-status-a`) |
| "Deny: Leave" (`task-set-denied-leave`) | "Deny: Status B" (`task-set-denied-status-b`) |
| "Deny: Category" (`task-set-denied-category`) | unchanged |

### Internal Flow ID

| Current | Replacement |
|---|---|
| `Flow_rescisao_no` *(Portuguese: "rescisão")* — also `default=` ref and `Flow_rescisao_no_di` | `Flow_status_a_yes` |

### Variables & Script Values

| Current | Replacement |
|---|---|
| `customerId` | `itemId` |
| `customerResponse` | `itemResponse` |
| `terminationResponse` | `statusAResponse` |
| `leaveResponse` | `statusBResponse` |
| `EMPLOYMENT_API_URL` | `SERVICE_API_URL` |
| `POLICY_TERMINATION` | `RULE_STATUS_A` |
| `POLICY_LEAVE` | `RULE_STATUS_B` |
| `POLICY_CATEGORY` | `RULE_CATEGORY` |
| `FIRED` | `STATUS_A_DENIED` |
| `LEAVE` | `STATUS_B_DENIED` |
| `CATEGORY_NOT_EQUAL_101` | `CATEGORY_REJECTED` |
| Policy value `"ELIGIBILITY"` | `"EVALUATION"` |
| Policy value `"OFFER"` | `"ALTERNATIVE"` |
| Fallback default URL `http://customer-mock:8080` | `http://item-mock:8080` |
| Fallback default URL `http://employment-mock:8080` | `http://service-mock:8080` |
| Portuguese comments | Removed |

### API Paths & JSON Fields

| Current | Replacement |
|---|---|
| `${BASE_URL}/api/customers/${INPUT.prop("customerId").stringValue()}` | `${BASE_URL}/api/items/${INPUT.prop("itemId").stringValue()}` |
| `${EMPLOYMENT_API_URL}/api/employment/termination?id=${S(customerResponse).prop("document").stringValue()}` | `${SERVICE_API_URL}/api/service/status-a?id=${S(itemResponse).prop("entityId").stringValue()}` |
| `${EMPLOYMENT_API_URL}/api/employment/leave?id=${S(customerResponse).prop("document").stringValue()}` | `${SERVICE_API_URL}/api/service/status-b?id=${S(itemResponse).prop("entityId").stringValue()}` |
| `.prop("document")` | `.prop("entityId")` |
| `json.prop("employmentRelationship").prop("category")` *(in Deny: Category script)* | `json.prop("relationship").prop("category")` |
| `S(customerResponse).prop("employmentRelationship").prop("category").numberValue() != 101` *(gateway condition)* | `S(itemResponse).prop("relationship").prop("category").numberValue() != 1` |
| `S(customerResponse).prop("employmentRelationship").prop("toj").numberValue()` *(tenure DMN input)* | `S(itemResponse).prop("relationship").prop("score").numberValue()` |

### Gateway Condition Expressions (full rename)

| Current expression | Replacement expression |
|---|---|
| `${S(terminationResponse).prop("hasTermination").boolValue() == true}` | `${S(statusAResponse).prop("hasStatusA").boolValue() == true}` |
| `${S(leaveResponse).prop("hasLeave").boolValue() == true}` | `${S(statusBResponse).prop("hasStatusB").boolValue() == true}` |

### DMN References

**Rule A (age) — end-to-end data flow:**
- `INPUT` JSON field: `age` → `score` (renamed in Java test and script-only-process)
- BPMN inputOutput source expression: `${S(INPUT).prop("age").numberValue()}` → `${S(INPUT).prop("score").numberValue()}`
- BPMN `<camunda:inputParameter name="age">` → `name="value"`
- DMN `camunda:inputVariable="age"` → `"value"`
- DMN input `<text>age</text>` → `<text>value</text>`
- DMN `label="age"` → `label="value"`

**Rule B (tenure) — end-to-end data flow:**
- Source expression: `${S(itemResponse).prop("relationship").prop("score").numberValue()}`
- BPMN `<camunda:inputParameter name="tenure">` → `name="score"`
- DMN `camunda:inputVariable="tenure"` → `"score"`
- DMN input `<text>tenure</text>` → `<text>score</text>`
- DMN `label="tenure"` → `label="score"`

**Result variable renames:**

| Current | Replacement |
|---|---|
| `camunda:decisionRef="decision-policy-age"` | `camunda:decisionRef="decision-rule-a"` |
| `camunda:resultVariable="POLICY_AGE_RESULT"` | `camunda:resultVariable="RULE_A_RESULT"` |
| `${POLICY_AGE_RESULT.get(...)}` | `${RULE_A_RESULT.get(...)}` |
| `camunda:decisionRef="decision-policy-tenure"` | `camunda:decisionRef="decision-rule-b"` |
| `camunda:resultVariable="POLICY_TENURE_RESULT"` | `camunda:resultVariable="RULE_B_RESULT"` |
| `${POLICY_TENURE_RESULT.get(...)}` | `${RULE_B_RESULT.get(...)}` |

---

## `dmn-rule-a.dmn` Changes (formerly `dmn-policy-age.dmn`)

| Current | Replacement |
|---|---|
| Definition id/name: `dmn-policy-age` / "Policy Age" | `dmn-rule-a` / "Rule A" |
| Decision id/name: `decision-policy-age` / "Policy Age" | `decision-rule-a` / "Rule A" |
| Input label/expression: `age` → `camunda:inputVariable="age"` | `value` → `camunda:inputVariable="value"` |
| `POLICY_NAME` output value: `"POLICY_AGE"` | `"RULE_A"` |
| `POLICY_REASON`: `"UNDER_18"` | `"BELOW_MIN"` |
| `POLICY_REASON`: `"ABOVE_65"` | `"ABOVE_MAX"` |
| `POLICY_REASON`: `"ACCEPTED_AGE"` | `"ACCEPTED"` |
| Thresholds `< 18`, `<= 65`, `> 65` | Kept (abstract once input is renamed) |

---

## `dmn-rule-b.dmn` Changes (formerly `dmn-policy-tenure.dmn`)

| Current | Replacement |
|---|---|
| Definition id/name: `dmn-policy-tenure` / "Policy Tenure" | `dmn-rule-b` / "Rule B" |
| Decision id/name: `decision-policy-tenure` / "Policy Tenure" | `decision-rule-b` / "Rule B" |
| Input label/expression: `tenure` → `camunda:inputVariable="tenure"` | `score` → `camunda:inputVariable="score"` |
| `POLICY_NAME` output value: `"POLICY_TENURE"` | `"RULE_B"` |
| `POLICY_REASON`: `"TENURE_UNDER_3"` | `"SCORE_BELOW_MIN"` |
| `POLICY_REASON`: `"ACCEPTED_TENURE"` | `"SCORE_ACCEPTED"` |
| Thresholds `<= 3`, `> 3` | Kept |

---

## `http-only-process.bpmn` Changes

| Current | Replacement |
|---|---|
| `/api/customers/123` | `/api/items/001` |
| `/api/employment/termination?id=02586611195` | `/api/service/status-a?id=item-001` |
| `/api/employment/leave?id=02586611195` | `/api/service/status-b?id=item-001` |
| `EMPLOYMENT_API_URL` | `SERVICE_API_URL` |

---

## `script-only-process.bpmn` Changes

Script 1 has embedded JSON and variable names that must all be updated:

| Current | Replacement |
|---|---|
| `S('{"policy":"ELEGIBILITY","customerId":"123","amount":50,"age":20}')` | `S('{"policy":"EVALUATION","itemId":"001","amount":50,"score":20}')` |
| `input.prop("customerId").stringValue()` | `input.prop("itemId").stringValue()` |
| `execution.setVariable("customerId", customerId)` | `execution.setVariable("itemId", itemId)` |

---

## `MemoryLeakLoadTest.java` Changes

### Constants & Input

| Current | Replacement |
|---|---|
| `PROCESS_KEY` default `"credit-eligibility"` | `"item-approval"` |
| Input JSON: `"ELIGIBILITY"`, `"customerId"`, `"age"` | `"EVALUATION"`, `"itemId"`, `"score"` |
| `EMPLOYMENT_API_URL` variable | `SERVICE_API_URL` |

### WireMock Stubs

**Stub 1** — item details:
- Path: `/api/customers/123` → `/api/items/001`
- Response body: replace all domain-specific fields:
  - `customerId: "cust-001"` → `itemId: "item-001"`
  - `document: "12345678900"` → `entityId: "entity-001"`
  - `birthDate: "1990-05-15"` → remove
  - `age: 35` → `score: 35`
  - `employmentRelationship` object → `relationship` object:
    - `admissionDate` → `enrollmentDate`
    - `employer.document: "11222333000144"` → `provider.id: "provider-001"`
    - `category: 101` → `category: 1`
    - `toj: 24` → `score: 24` *(note: rename `toj` field to `score`; this is the value read by the Rule B DMN)*
    - `grossSalary`, `netSalary`, `realNetSalary` → `quota`, `baseQuota`, `adjustedQuota`
    - `demissionDate` → `endDate`

**Stub 2** — status-a check:
- Path: `/api/employment/termination` → `/api/service/status-a`
- Query param: `id=12345678900` → `id=entity-001`
- Response body: `{ "hasTermination": false }` → `{ "hasStatusA": false }`
- (Aligns with BPMN condition rename: `hasTermination` → `hasStatusA`)

**Stub 3** — status-b check:
- Path: `/api/employment/leave` → `/api/service/status-b`
- Query param: `id=12345678900` → `id=entity-001`
- Response body: `{ "hasLeave": false }` → `{ "hasStatusB": false }`
- (Aligns with BPMN condition rename: `hasLeave` → `hasStatusB`)

---

## `README.md` Changes

- Update `loadtest.processKey` default from `credit-eligibility` → `item-approval`
- Replace WireMock endpoint descriptions: remove Portuguese ("rescisão check", "afastamento check"), use "status A check", "status B check"
- Replace "DMN decision tables (age, fee, TOJ)" → "DMN decision tables (rule A, fee, rule B)"
- Update "BPMN Process" section description

---

## Out of Scope

- No logic changes (thresholds, flow structure, error handling remain identical)
- No changes to `simple-process.bpmn` or `pure-js-process.bpmn`
- No changes to `pom.xml`, Spring Boot configuration, or test infrastructure
- **`DONE OUT` subprocess bug** — The success-path subprocess (`esp-done`) currently writes `result = "DENIED"` and `elegible = false` (also a typo). This is a pre-existing behavioral bug unrelated to anonymization and is excluded from this change.
