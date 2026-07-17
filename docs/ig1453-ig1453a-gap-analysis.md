# IG1453 / IG1453A Structured-Prompt & Extension Compliance Gap Analysis

## 1. Executive Summary

TM Forum **IG1453 "Agent to Agent Protocol for Telecoms" v2.1.0** defines four interoperable A2A extension profiles — **Task-T** (task description/execution), **Notification-T** (event subscription and delivery), **Negotiation-T** (multi-turn task-term negotiation), and **Authorization-T** (operation-permission review/approval) — each with a normative data model, a canonical set of URIs, and reference Markdown prompt templates. **IG1453A "Structured Prompt of Agent to Agent Protocol for Telecoms A2AT" v1.0.0** defines the syntax layer those four profiles share: an `## InstructionName` heading syntax, a `{{variable}}` placeholder syntax, formatting rules for lists/links/paragraphs, and a six-section "Basic Template" (`Task Type` / `Task Description` / `Target Object` / `Task Context` / `Constraints` / `Expected Output`).

Checked against these two documents directly (not against the A2A protocol generically, and not against the A2A-T reference implementation — see `docs/comparative-analysis.md` for those), `a2a-t-sdk-python` implements **one of the four extension profiles, partially and with several naming/URI deviations from the spec text**, and implements **none of IG1453A's syntax rules as enforced, machine-checkable constraints** — the SDK's templates are free-form Markdown/Python `str.format_map()` substitution with no validation against the spec's Instruction vocabulary. Concretely:

- **Task-T**: partially implemented. The client generates a rendered Markdown body, but three of four current scenario templates rename the spec's `Target Object` to a spec-undefined `Task Target` (任务目标) attribute, one scenario (`fault_diagnosis`) omits the `Constraints` section entirely, and every wire payload is wrapped in a YAML front-matter block (`scenario_code`/`language`/`description`) that IG1453A's syntax does not define — a strictly spec-conformant Task-T parser expecting the document to open with `## Task Type` would fail to parse this SDK's output.
- **Notification-T**: not implemented as a distinct extension at all. No `Notification-T` URI, no subscribe/notify duality, no event-delivery code path exists anywhere in `src/`. One scenario (`subscribe_incident`) happens to reuse Notification-T's exact attribute vocabulary (Subscription Description/Notification Subject/Notification Conditions/Event Data Reporting Format) in its template text, but it is routed through the generic Task-T-shaped prompt-generation/negotiation pipeline under `TASK_PROMPT_KEY`, not a `Notification-T` extension URI, and produces no actual asynchronous notification delivery.
- **Negotiation-T**: implemented with a materially different type taxonomy than the spec defines. The spec defines exactly three negotiation forms — **Target Negotiation**, **Feasibility Negotiation**, **Information Negotiation** — each with a specific attribute set. The SDK implements four types — `CLARIFICATION`, `FEASIBILITY`, `FULFILLMENT`, `INFORMATION` — of which `FULFILLMENT` has no spec counterpart at all, `CLARIFICATION` is an unofficial rename of `Target Negotiation`, and three of the four types (`CLARIFICATION`/`FEASIBILITY`/`FULFILLMENT`) are behavioral no-ops that inherit an identity-function prompt renderer, never invoke an LLM, and never consult the type-specific `system.md`/`user.md` prompt resource files that are shipped for them.
- **Authorization-T**: not implemented at all. Zero trace anywhere in `src/` — no operation-permission data model, no configuration/modification/dynamic-authorization templates, no approval workflow.
- **IG1453A structured-prompt syntax**: the `{{variable}}` substitution mechanics are implemented (`TaskPromptRenderer`), but the Instruction-heading vocabulary, the mandatory/optional attribute rules, and the Basic Template's six-section shape are not validated or enforced anywhere in code — template authors are free to (and currently do) deviate from the spec's attribute names with no compliance check catching it.

None of this contradicts `docs/comparative-analysis.md`'s conclusions — it sharpens them with normative spec-text evidence and identifies several new, code-verifiable findings that document did not have grounds to make (the exact spec-vs-code URI mismatches, the extra `Task Target` attribute, the missing `Constraints` section, the dead negotiation-type prompt resources, and the complete absence of Notification-T and Authorization-T as distinct capabilities).

## 2. Methodology & Source Documents

This analysis is grounded in a direct reading of:

- `docs/IG1453_Agent_to_Agent_Protocol_for_Telecoms_v2.1.0.docx` — the full document body, extracted via `python-docx` (the document's content is wrapped in a single `w:sdt` content control, which required a recursive block walker rather than the library's default `Document.paragraphs`/`Document.tables` accessors, which return empty for this file).
- `docs/IG1453A_Structured_Prompt_of_Agent_to_Agent_Protocol_for_Telecoms_A2AT_v1.0.0.docx` — same extraction method.
- The current `a2a-t-sdk-python` source tree (`src/a2a_t/`) and shipped resources (`package_data/prompt_resources/`), read directly — not from any prior analysis document's characterization of the code.

Findings are cited by spec section heading (as they appear in the source `.docx`, since the documents carry no stable page numbers usable outside a Word renderer) and by `file:line` for code. This document does not re-derive findings already established against the A2A-T reference implementation or the base A2A protocol in `docs/comparative-analysis.md`; where a finding here overlaps conceptually with that document (e.g. Event-T absence, negotiation-type naming), it is because the two independent evidence sources — the reference implementation there, the normative spec text here — happen to agree, not because this analysis assumes that document's conclusions.

## 3. Spec Overview

### 3.1 IG1453's four extension profiles

| Profile | URI root (as written in spec examples) | Purpose |
|---|---|---|
| **Task-T** | `.../extensions/TASK-T/NL/v1` (NL form); `.../extensions/TASK-T/3GPP_Intent/v19.0` (3GPP form) | Tasking and execution for telecom intents |
| **Notification-T** | `.../extensions/Notification-T/NL/v1` | Event subscription, filtering, and delivery |
| **Negotiation-T** | `.../extensions/Negotiation-T/NL/v1` | Multi-turn proposal negotiation for task terms |
| **Authorization-T** | `.../extensions/ Authorization-T/NL/v1` (stray leading space present in the spec text itself) | Review/approval of high-risk operations and dynamic permission application |

*(IG1453 §"Protocol description"; note the spec text itself is inconsistent in URI casing — `TASK-T` is all-caps while `Notification-T`/`Negotiation-T`/`Authorization-T` use title case. This inconsistency is in the source document, not introduced by this analysis.)*

IG1453A (v1.0.0, the older/narrower of the two documents) only defines the syntax layer for three of these — Task-T, "Event-T" (IG1453A's own name for what IG1453 v2.1.0 calls Notification-T), and Negotiation-T (IG1453A §"Introduction to A2A-T"). Authorization-T is new in IG1453 v2.1.0 and has no corresponding syntax-layer treatment in IG1453A v1.0.0 — its templates in IG1453 follow the same `##`-heading convention by example but aren't covered by IG1453A's formal syntax rules.

### 3.2 IG1453A's syntax layer

- **Instruction syntax**: `## InstructionName` — a level-2 Markdown heading whose text is the exact instruction name (IG1453A §"Syntax of Instruction"). The spec's own prose says the name should follow `##` "without any spaces or other identifiers," but every worked example in both documents shows a space (`## Task Description`, not `##Task Description`) — a wording/example inconsistency in the spec itself, noted here so it isn't mistaken for a code defect if a future implementation follows the examples (the practical convention) over the literal prose rule.
- **Variable syntax**: `{{VariableName}}` double-curly-brace placeholders (IG1453A §"Syntax of Variables").
- **Formatting rules**: numbered lists (`1.`, `2.`, ...) for `Expected Output` enumerations, dash-prefixed unordered lists, blank-line-separated paragraphs, and `[text](url)` links (IG1453A §"Syntax of Formatting Rules", §"Syntax of other commonly used text formats").
- **Basic Template** (IG1453A §"Basic Template"): six sections in this order — `Task Type`, `Task Description`, `Target Object`, `Task Context`, `Constraints`, `Expected Output`. Per the Essential Instruction Index Table (IG1453A §"Essential Instruction of A2A-T"), only `Task Description` is mandatory; the other five are optional.

## 4. Task-T Compliance

### 4.1 Data model attribute mapping

| Spec attribute (IG1453 §"Data Model" / IG1453A §"Essential Instruction of A2A-T") | Mandatory? | Current SDK equivalent | Status |
|---|---|---|---|
| `Task Description` | Mandatory | Present in all four scenario templates as `## Task Description` / `## 任务描述` | ✅ Present, correct name |
| `Task Type` | Optional | Present in 3 of 4 templates as `## 任务类型(Task Type)` (`fault_diagnosis`, `energy_saving`, `private_line_complaint`); absent from `subscribe_incident` (which uses the Notification-T vocabulary instead, see §5) | ⚠️ Present but heading text deviates from spec (bilingual gloss, not the bare instruction name) |
| `Target Object` | Optional | **Not present under this name in any template.** `energy_saving`/`private_line_complaint`/`fault_diagnosis` use `## 任务对象(Task Object)` for what the spec's examples call `Target Object`, and additionally define a spec-undefined `## 任务目标(Task Target)` section | ❌ Renamed and duplicated — see §4.2 |
| `Task Context` | Optional | Present in all four templates as `## Task Context` / `## 任务上下文` | ✅ Present, correct name |
| `Constraints` | Optional | Present in `energy_saving`; **absent entirely from `fault_diagnosis`** | ⚠️ Inconsistent — see §4.3 |
| `Expected Output` | Optional | Present in all four templates | ✅ Present, correct name |

**Evidence:** `package_data/prompt_resources/templates/{energy_saving,fault_diagnosis,private_line_complaint,subscribe_incident}/zh-CN/template.md`; IG1453 §"Task-T data model attributes"; IG1453A §"Essential Instruction Index Table".

### 4.2 Extra, spec-undefined `Task Target` (任务目标) attribute

**What's wrong:** `fault_diagnosis`, `energy_saving`, and `private_line_complaint` templates all define a `## 任务目标(Task Target)` section that has no counterpart anywhere in IG1453's Task-T data model or IG1453A's Essential Instruction set. Its actual content in each template describes what the spec's `Task Description` and `Target Object` attributes already cover (e.g. `fault_diagnosis`'s "任务目标" content is "说明对故障诊断任务的具体要求" — "state the specific requirements of the fault-diagnosis task," which is `Task Description`'s job per spec; `energy_saving`'s "任务目标" holds the energy/rate targets, closer to what the spec's `Target Object` covers for other scenarios).

**Current-code evidence:** `package_data/prompt_resources/templates/fault_diagnosis/zh-CN/template.md:9-16`, `energy_saving/zh-CN/template.md:7-13`, `private_line_complaint/zh-CN/template.md:7-9`; corresponding `slot.json` files key on `"任务目标"` as a top-level required property (e.g. `package_data/prompt_resources/slots/fault_diagnosis/zh-CN/slot.json`, `required: ["任务目标", "任务上下文"]`).

**Recommendation:** Reconcile each scenario's `任务目标` content into the correct spec attribute (`Task Description` or `Target Object` depending on scenario) rather than carrying a sixth, non-normative attribute alongside the spec's five optional ones — or, if the SDK team has a deliberate reason to keep a distinct "task target/goal" concept, document it explicitly as an SDK-specific extension attribute rather than presenting it silently alongside spec-named sections.

### 4.3 `fault_diagnosis` scenario omits `Constraints` entirely

**What's wrong:** IG1453A's Basic Template includes `Constraints` as one of six sections; three of the SDK's four scenario templates include it (`energy_saving` has `## 约束条件(Constraints)`; `subscribe_incident`'s Notification-T-shaped template doesn't need it since Constraints isn't a Notification-T attribute; `private_line_complaint` omits it too). `fault_diagnosis` — the one scenario whose template most closely follows the Task-T Basic Template shape otherwise — has no `Constraints` section or slot at all.

**Current-code evidence:** `package_data/prompt_resources/templates/fault_diagnosis/zh-CN/template.md` (sections present: 任务类型, 任务描述, 任务目标, 任务上下文, 预期输出 — no constraints section); `package_data/prompt_resources/slots/fault_diagnosis/zh-CN/slot.json` (no `约束条件`/`Constraints` property).

**Recommendation:** Since `Constraints` is optional per spec, this is not itself a violation — but it is an unexplained inconsistency across scenarios that share the same underlying Task-T shape. Either add a `Constraints` section (task priority is explicitly relevant for fault diagnosis — the spec's own `Fault Repair` example template includes it) or note in template documentation why it was intentionally left out for this scenario.

### 4.4 Wire payload is wrapped in a non-spec YAML front-matter block

**What's wrong:** IG1453A's syntax defines the structured prompt as beginning directly with `## Task Type` (or whichever the first Instruction is) — there is no front-matter or preamble concept anywhere in either document. This SDK's `format_task_prompt()` unconditionally prepends a three-field YAML block before the rendered body:

```
---
scenario_code: subscribe_incident
language: en-US
description: ...
---

## Task Type
...
```

A consumer built strictly to IG1453A's syntax rules — expecting the document to open with an Instruction heading — would fail to locate `Task Type`/`Task Description` at the start of the document, or would need SDK-specific knowledge of this front-matter convention to skip past it.

**Current-code evidence:** `src/a2a_t/prompt/common/task_prompt_format.py:9-19` (`format_task_prompt`, unconditional `---\nscenario_code: ...\n---\n\n` prefix); `src/a2a_t/prompt/common/task_prompt_format.py:22-33` (`parse_task_prompt_metadata` requires the prompt to start with `---\n` or raises `TaskPromptFormatError`).

**Recommendation:** If the front matter is needed for the SDK's own internal routing (scenario/language/version resolution), keep it, but clearly document that the front-matter-wrapped document is an SDK-internal representation, and provide (or document) a strip-front-matter step before the body is treated as a spec-conformant IG1453A structured prompt on the wire — analogous to how `docs/comparative-analysis.md` §4.3 already recommends a `Message`/`Artifact` translation layer for A2A protocol conformance generally.

### 4.5 Task-T extension URI is not used anywhere, and the SDK's own URI constant doesn't match the spec's

**What's wrong:** IG1453 defines the Task-T NL extension URI as `https://projects.tmforum.org/a2aproject/telecommunication/extensions/TASK-T/NL/v1` (all-caps `TASK-T`, with an `/NL/v1` suffix distinguishing the natural-language form from the 3GPP-intent form). The SDK's closest constant, `TASK_PROMPT_KEY`, is `https://projects.tmforum.org/a2aproject/telecommunication/extensions/Task-T/v1` — different casing (`Task-T` vs `TASK-T`) and missing the `/NL` form-discriminator segment entirely (so there is no way to express "this payload uses the 3GPP-intent form" per spec, since the SDK has no code path for 3GPP intents at all — consistent with `docs/comparative-analysis.md` §5.12's finding).

**Current-code evidence:** `src/a2a_t/negotiation/common/constants.py:6` — `TASK_PROMPT_KEY = "https://projects.tmforum.org/a2aproject/telecommunication/extensions/Task-T/v1"`; IG1453 §"Telecom Task Extension" / §"Overview" (`URI: "https://projects.tmforum.org/a2aproject/telecommunication/extensions/TASK-T/NL/v1"`).

**Recommendation:** Align the constant's exact string (casing and `/NL/v1` suffix) with the spec text if byte-for-byte URI compatibility with other IG1453 implementations is a goal; if the current value is an intentional SDK-specific URI rather than a claim of spec conformance, say so explicitly rather than leaving the near-miss unexplained.

## 5. Notification-T Compliance

### 5.1 No distinct Notification-T extension exists in code

**What's missing:** IG1453's Notification-T defines five attributes — `Subscription Description` (mandatory-shaped, used by the "producer agent" to describe subscription/notification requirements), `Notification Subject` (optional), `Notification Conditions` (optional), `Event Data Reporting Format` (optional), and `Expected Output` — plus a dedicated URI (`.../Notification-T/NL/v1`) and an asynchronous subscribe-now/notify-later interaction pattern (IG1453 §"Network Event Subscription and Notification", §"Notification-T Data Model").

**Current-code evidence:** `grep -rli "notification-t\|event-t\|subscription" src/a2a_t/` returns zero matches. There is no `Notification-T`-namespaced URI constant anywhere in `src/a2a_t/negotiation/common/constants.py` (which defines only `NEGOTIATION_TEXT_KEY`, `NEGOTIATION_CONTEXT_KEY`, `TASK_PROMPT_KEY`). No code path delivers an asynchronous notification back to a subscriber; the SDK's request/response model is entirely synchronous (`A2ATClient.generate_task_prompt()` → `A2ATServer.check_task_prompt()` → negotiation rounds → final agreed prompt).

**Recommendation:** This is the single largest structural gap versus IG1453 (also identified independently, via the A2A-T reference implementation, in `docs/comparative-analysis.md` §5.5). If Notification-T support is in scope, it needs: (a) a `NOTIFICATION_T_KEY` extension URI constant, (b) a data model with the five attributes above, (c) an asynchronous delivery mechanism (which in turn depends on the transport/AgentExecutor work `docs/comparative-analysis.md` Milestone 1 already recommends), and (d) prompt templates per the three examples the spec provides (incident, energy-efficiency, network-quality).

### 5.2 `subscribe_incident`'s template content matches Notification-T's vocabulary exactly, but is transmitted as if it were Task-T

**What's notable:** Unlike the other three scenarios, `subscribe_incident`'s template uses `## 订阅描述` (Subscription Description), `## 通知主题` (Notification Subject), `## 订阅条件` (~Notification Conditions), `## 上报通知数据格式` (Event Data Reporting Format), `## 预期输出` (Expected Output) — a close, correctly-named Chinese translation of IG1453's actual Notification-T attribute set, not Task-T's. This is a genuine positive: whoever authored this template evidently modeled it directly on the Notification-T data model.

However, this scenario is generated by the same `PromptGenerationOrchestrator` as the other three, validated by the same `PromptComplianceOrchestrator`, and — if it reaches negotiation — carried under `TASK_PROMPT_KEY` (the Task-T wire key) exactly like a fault-diagnosis or energy-saving task, with no `Notification-T`-specific URI, no subscription-lifecycle state (active/failed/canceled), and no code that ever pushes a notification back to the subscriber when a matching incident actually occurs. The scenario name (`subscribe_incident`) and prompt content promise event-subscription semantics that the transport and runtime never deliver.

**Current-code evidence:** `package_data/prompt_resources/templates/subscribe_incident/zh-CN/template.md` (Notification-T-shaped content); `src/a2a_t/negotiation/handling/negotiation_handler.py:200-213` (`_build_result_map` — the same `TASK_PROMPT_KEY`-keyed payload builder used for every scenario, including this one); `src/a2a_t/negotiation/common/constants.py` (no `Notification-T`-specific key exists to route this scenario through instead).

**Recommendation:** Once a distinct Notification-T extension exists (§5.1), migrate `subscribe_incident` to it — the template content is already spec-aligned and would need little rework, unlike the other three scenarios' Task-T deviations (§4.2-§4.3).

## 6. Negotiation-T Compliance

### 6.1 Type taxonomy mismatch

| Spec type (IG1453 §"Task Negotiation Extension" / §"Overview") | Purpose | Current SDK type | Status |
|---|---|---|---|
| **Target Negotiation** | Disambiguate the task objective ("Objective Confirmation"/"Original Objective"/"Objective Recognition"/"Follow-Up Information" attributes) | `CLARIFICATION` | ⚠️ Conceptually corresponds, but renamed with no spec-derived name anywhere in code or docs |
| **Feasibility Negotiation** | Resolve resource/timing conflicts ("Feasibility Confirmation"/"Task Information"/"Constraints"/"Output Format" attributes) | `FEASIBILITY` | ✅ Name matches |
| **Information Negotiation** | Supplement missing execution-time information ("Information Supplement Request"/"Required Information Items"/"Data Return Confirmation"/"Data Request Rejection"/"Alternative Solution Proposal" attributes) | `INFORMATION` | ✅ Name matches, and is the only type with real logic (§6.3) |
| *(no spec counterpart)* | — | `FULFILLMENT` | ❌ Not defined anywhere in IG1453 or IG1453A |

**Evidence:** IG1453 §"Task Negotiation Extension" ("This document defines 3 commonly used task negotiation templates in the ICT field including, Target Negotiation, feasibility negotiation, and information negotiation"); `src/a2a_t/negotiation/common/enums.py` (`NegotiationType`: `CLARIFICATION`/`FEASIBILITY`/`FULFILLMENT`/`INFORMATION`); `src/a2a_t/negotiation/types/{clarification,feasibility,fulfillment,information}.py`.

This sharpens `docs/comparative-analysis.md` §5.6's finding (which compared against the A2A-T reference implementation's `TARGET`/`FEASIBILITY`/`INFORMATION` set) with the normative source: the reference implementation's naming is not an independent design choice, it is the spec's actual vocabulary. `CLARIFICATION` is this SDK's own coinage with no textual basis in either document, and `FULFILLMENT` appears to be invented outright — there is no "fulfillment negotiation" concept in IG1453 v2.1.0 at all.

### 6.2 Negotiation-T extension URI mismatch and missing form discriminator

**What's wrong:** the spec's Negotiation-T URI is `https://projects.tmforum.org/a2aproject/telecommunication/extensions/Negotiation-T/NL/v1` (title case, `/NL/v1` suffix). The SDK's `NEGOTIATION_TEXT_KEY` is `https://projects.tmforum.org/a2aproject/telecommunication/extensions/NEGOTIATION-T` — all-caps, no version segment at all (this exact inconsistency was already flagged relative to the SDK's own other two constants in `docs/comparative-analysis.md` §4.5; here it's additionally confirmed to not match the spec's casing or the `/NL/v1` suffix either way). The SDK's `NEGOTIATION_CONTEXT_KEY` (`.../DATA-NEGOTIATION-T/v1`) has no counterpart in the spec text at all — the spec carries negotiation state as attributes within the single Negotiation-T payload (e.g. round/status are not called out as a separately-keyed "data negotiation" object anywhere in IG1453), so this appears to be an SDK-invented URI rather than a spec-derived one.

**Evidence:** IG1453 §"Task Negotiation Extension" ("reference the URI: https://projects.tmforum.org/a2aproject/telecommunication/extensions/Negotiation-T/NL/v1"); `src/a2a_t/negotiation/common/constants.py:4-5`.

**Recommendation:** Same as §4.5 — either align the constants to the spec's exact strings, or explicitly document that these are SDK-internal URIs not intended as literal IG1453 URI reuse.

### 6.3 Three of four negotiation types are behavioral no-ops with dead prompt resources

**What's wrong:** `CLARIFICATION`, `FEASIBILITY`, and `FULFILLMENT` are empty subclasses of `BaseNegotiationType` that add no logic (`class ClarificationNegotiationType(BaseNegotiationType): pass`, identically for the other two). `BaseNegotiationType.process_received_message()` always returns `need_response=True` while `IN_PROGRESS`, doing no analysis of the message content at all — it never calls an LLM, never applies the type-specific attribute set from §6.1's table (e.g. never asks for "Objective Recognition" for a `CLARIFICATION`/Target negotiation, never asks for "Feasibility Confirmation" for a `FEASIBILITY` negotiation).

More specifically: `NegotiationPromptRenderer.render()` (the method every type's `render_start_prompt`/`render_continue_prompt` ultimately calls) is a literal identity function — `return message` — regardless of `negotiation_type`. Yet the package ships four complete, populated, bilingual (`en-US`/`zh-CN`) prompt-resource directories at `package_data/prompt_resources/prompts/{clarification,feasibility,fulfillment,information}_negotiation/` with `system.md`/`user.md` files. None of these files are read by `NegotiationPromptRenderer` or by `NegotiationHandler` — only `InformationNegotiationType` has real logic, and it doesn't use these prompt files either; it delegates to `PromptComplianceOrchestrator.check()` instead (a completely different code path, reused from the Task-T pipeline).

**Current-code evidence:** `src/a2a_t/negotiation/types/clarification.py`, `feasibility.py`, `fulfillment.py` (each a 2-line pass-through class); `src/a2a_t/negotiation/types/base.py:14-31` (`process_received_message` default: unconditional `need_response=True`); `src/a2a_t/negotiation/rendering/negotiation_prompt_renderer.py:8-10` (`render()` returns `message` unchanged, `negotiation_type` parameter unused); `package_data/prompt_resources/prompts/clarification_negotiation/en-US/user.md`, `feasibility_negotiation/en-US/user.md`, `fulfillment_negotiation/en-US/user.md` (all populated, all unreferenced by any Python code — confirmed via `grep -rn "clarification_negotiation\|feasibility_negotiation\|fulfillment_negotiation" src/a2a_t/`, which finds no loader call sites for these three specific prompt-resource directories, unlike `information_negotiation` which is likewise unloaded by the renderer but at least has `InformationNegotiationType`'s own real logic backing it).

**Recommendation:** This is more severe than `docs/comparative-analysis.md`'s existing framing of these as "stubs" — the negotiation *type* dispatch exists and is exercised by `NegotiationHandler`, but the *content* is entirely generic regardless of which of the four types is active, and three pre-written, translated prompt-resource directories are dead code. Either wire `NegotiationPromptRenderer` (or a type-specific renderer) to actually use `system.md`/`user.md` per negotiation type and implement the type-specific attribute sets from §6.1, or remove the unused prompt-resource directories so they don't imply functionality that isn't there.

## 7. Authorization-T Compliance

### 7.1 No trace of Authorization-T anywhere in the codebase

**What's missing:** IG1453 defines Authorization-T with two attribute groups — (1) `Task Description`/`Operation List`/`Expected Output` for configuring or modifying an operation-permission policy, and (2) `Task Description`/`Operation permission application list`/`Expected Output` for a dynamic, in-execution authorization request — plus three reference templates (`Configuration Operation Permission Policy`, `Configuration Modify Operation Policy`, `Dynamic Authorization Apply`) and a dedicated URI (IG1453 §"Security Extension", §"Authorization-T Data Model"). It exists specifically to address OWASP LLM-application threat categories T1/T5/T6/T15 for O&M operations (IG1453 §"Overview" under Security Extension).

**Current-code evidence:** `grep -rli "authorization-t\|operation.*permission\|dynamic.*authoriz" src/a2a_t/` returns zero matches. No `AUTHORIZATION_T_KEY`-style constant exists. No approval/permission data model, no operation-list type, no "allow/deny" response shape exists anywhere in `src/a2a_t/`.

**Recommendation:** This is a wholesale-missing extension profile, not a partial-implementation gap — there is nothing partially built to extend. Given the spec frames Authorization-T as addressing specific security threat categories for high-risk O&M operations, and this SDK's `SafetyGuardrailFactory`/`noop` guardrail (`docs/comparative-analysis.md` mentions this only in passing) is the closest existing concept, treat Authorization-T as net-new design work: (1) define the `Operation Permission Object` shape (operation name, operation type, authorization policy, validity period, allowed time windows, trigger conditions — per IG1453 §"Authorization-T Data Model"), (2) implement the three reference templates, (3) implement an approval-result response shape (`Result: Allow/Deny`, expiration, credential, approver).

## 8. IG1453A Structured-Prompt Syntax Compliance

### 8.1 `{{variable}}` substitution: implemented and correctly matches the spec's syntax

**What's right:** IG1453A defines variables as `{{VariableName}}` double-curly-brace placeholders (§"Syntax of Variables"). `TaskPromptRenderer` correctly recognizes this exact syntax and substitutes it with extracted slot values.

**Current-code evidence:** `src/a2a_t/prompt/task_rendering/task_prompt_renderer.py:8` — `_DOUBLE_BRACED_SLOT_PATTERN = re.compile(r"\{\{([^{}]+)\}\}")`, converted to single-brace `str.format_map()` syntax internally (line 29) before substitution. This is a genuine point of conformance worth preserving as the rest of this section's gaps are addressed.

### 8.2 No enforcement of the Instruction-heading vocabulary or the Essential Instruction Index Table

**What's missing:** IG1453A defines a specific, named set of Instructions (`Task Description`, `Task Type`, `Target Object`, `Task Context`, `Constraints`, `Expected Output`) with mandatory/optional annotations per §3.2 above. Nothing in the current codebase validates that a template's `##`-headings match this vocabulary, that the one mandatory attribute (`Task Description`) is present, or that unrecognized/renamed headings (like §4.2's `Task Target`) are flagged.

**Current-code evidence:** `src/a2a_t/prompt/task_rendering/task_prompt_renderer.py` — the renderer's `_SECTION_HEADING_PATTERN` (`^\s*##+\s+\S`) only detects that *some* level-2-or-deeper heading exists at a line, not which heading; it never compares heading text against a known vocabulary. `src/a2a_t/prompt/validation/json_schema_slot_validator.py` validates slot *values* (via `slot.json`'s JSON Schema) but has no concept of section/Instruction names at all — slot validation and section-heading conformance are two entirely separate, unconnected checks.

**Recommendation:** Add a lightweight structural linter (could run at CI time against the shipped `template.md` resources, or at runtime as an optional strict-mode check) that parses a template's `##` headings and flags: headings outside the Essential Instruction vocabulary (or a documented scenario-specific extension list), a missing `Task Description`, and — if adopted — enforces ordering matching the Basic Template. This would have caught §4.2 and §4.3 before they shipped.

### 8.3 Formatting-rules syntax (ordered/unordered lists, links, paragraphs) is unvalidated free text

**What's missing:** IG1453A §"Syntax of Formatting Rules" and §"Syntax of other commonly used text formats" define specific list/link/paragraph conventions primarily so that an `Expected Output` section's structure is machine-parseable by the receiving agent. The current SDK treats the entire template body (including `Expected Output`) as an opaque string that is rendered once (variable substitution) and then handed to an LLM for re-extraction on the server side (`PromptComplianceOrchestrator`) — there's no code that parses or validates conformance to the ordered-list/unordered-list/link syntax rules themselves.

**Current-code evidence:** `src/a2a_t/prompt/task_rendering/task_prompt_renderer.py` has no list/link parsing logic; `package_data/prompt_resources/templates/*/zh-CN/template.md` — spot-checking `Expected Output` sections shows a mix of plain numbered prose (`1、订阅结果...` in `subscribe_incident`, using a Chinese full-width comma-period `、` list marker, not the spec's `1.`-with-period ASCII convention) and unmarked prose (`energy_saving`'s `Expected Output` uses `###`-level sub-headings for "意图报告基本信息"/"意图报告详细信息" rather than the spec's flat numbered-list convention).

**Recommendation:** Lower priority than §8.2 (structural Instruction-name conformance matters more for interop than output-formatting-style conformance), but worth normalizing `Expected Output` sections to the spec's `1.`/`-`/`[text](url)` conventions for consistency, particularly since `subscribe_incident`'s `、` list-marker convention is not ASCII-portable and could confuse a non-Chinese-aware parser.

### 8.4 Basic Template's six-section shape is not offered as a literal fallback template

**What's missing:** IG1453A §"Basic Template" defines a generic six-section template explicitly intended for "abstract or unfamiliar tasks, such as cross-business scenarios that appear for the first time" — a fallback for scenarios with no dedicated best-practice template yet. This SDK's scenario-recognition pipeline (`ScenarioResolutionOrchestrator`) requires a `scenario_code` to match an entry in `scenarios.json`; there is no generic/fallback scenario or template that implements IG1453A's Basic Template for unmatched inputs — an unrecognized scenario fails with `SCENARIO_PARSE_FAILED` rather than falling back to the spec's intended abstract-task template.

**Current-code evidence:** `src/a2a_t/prompt/analysis/scenario_resolution_orchestrator.py` (resolves against the fixed `scenarios.json` list, no fallback path); CLAUDE.md's documented failure codes confirm `SCENARIO_PARSE_FAILED` is a terminal failure, not a fallback trigger.

**Recommendation:** Consider adding the spec's literal Basic Template (`Task Type`/`Task Description`/`Target Object`/`Task Context`/`Constraints`/`Expected Output`, all slots free-text) as a genuine fallback scenario, activated when scenario recognition doesn't match a known `scenario_code`, rather than only supporting the fixed, closed set of four current scenarios.

## 9. Prioritized Missing-Feature List

### P0 — Entire extension profiles or spec-required behaviors absent

1. **Notification-T is not implemented as a distinct extension** (§5.1) — no URI, no data model, no async delivery. `subscribe_incident` simulates its vocabulary but not its transport or lifecycle.
2. **Authorization-T is not implemented at all** (§7.1) — zero code, zero data model, zero templates.
3. **Negotiation-T's type taxonomy doesn't match the spec** (§6.1) — `FULFILLMENT` has no spec basis; `CLARIFICATION` is an unofficial rename of `Target Negotiation`; three of four types have no real behavior despite shipped, unused prompt resources (§6.3).
4. **None of the SDK's three extension URI constants match the spec's exact URI strings** (§4.5, §6.2) — casing and version-segment mismatches on every one, plus one URI (`DATA-NEGOTIATION-T/v1`) with no spec basis at all.

### P1 — Partial implementations with concrete, fixable deviations

5. **`Target Object` is renamed to a spec-undefined `Task Target`, duplicated across three scenario templates** (§4.2).
6. **`fault_diagnosis` omits the `Constraints` section** that its sibling scenarios and the spec's own Fault Repair example include (§4.3).
7. **Wire payloads are wrapped in non-spec YAML front matter**, breaking strict IG1453A structured-prompt parsing at the document boundary (§4.4).
8. **No structural validation of the Instruction-heading vocabulary** — nothing in the codebase would catch findings 5 or 6 automatically (§8.2).

### P2 — Consistency and completeness gaps

9. **`Expected Output` formatting conventions are inconsistent** across scenarios and don't uniformly follow IG1453A's list-marker syntax (§8.3).
10. **No fallback implementation of IG1453A's Basic Template** for scenarios outside the fixed four (§8.4).
11. **All four scenario templates and slot schemas are `zh-CN`-only** — despite the SDK's documented default language being `en-US` (`A2AT_LANGUAGE=en-US` per `package_data/env.example`) and despite `en-US` prompt resources existing for scenario-recognition/slot-extraction/negotiation LLM prompts, there is no `en-US` `template.md` or `slot.json` for any of the four scenarios (`find package_data/prompt_resources/{templates,slots} -name template.md -o -name slot.json` returns only `zh-CN` paths). Since `CLAUDE.md` documents client-side language fallback as falling back *to* `en-US` when a requested language's resource is missing, a default-configured `en-US` deployment would fail to find any scenario template or slot schema at all, with no `zh-CN`-ward fallback to catch it.

## 10. Appendix: File Reference Index

### Source specification documents (`docs/`)
- `IG1453_Agent_to_Agent_Protocol_for_Telecoms_v2.1.0.docx` §"Protocol description", §"Telecom Task Extension", §"Network Event Subscription and Notification", §"Task Negotiation Extension", §"Security Extension" — the four extension profiles' data models, URIs, and reference templates
- `IG1453A_Structured_Prompt_of_Agent_to_Agent_Protocol_for_Telecoms_A2AT_v1.0.0.docx` §"Syntax and Common Ontology of A2A-T Structured Prompt", §"Structured Prompt Template" — the syntax layer (Instruction/Variable/Formatting-Rules syntax, Basic Template)

### `a2a-t-sdk-python` (this repo)
- `src/a2a_t/prompt/common/task_prompt_format.py:9-33` — `format_task_prompt`/`parse_task_prompt_metadata`, the non-spec YAML front-matter wrapper (§4.4)
- `src/a2a_t/prompt/task_rendering/task_prompt_renderer.py` — `{{variable}}` substitution (§8.1, conformant) and heading-agnostic section handling (§8.2, non-conformant)
- `src/a2a_t/prompt/validation/json_schema_slot_validator.py` — slot-value validation, no Instruction-name validation
- `src/a2a_t/prompt/analysis/scenario_resolution_orchestrator.py` — fixed `scenarios.json` matching, no Basic Template fallback (§8.4)
- `src/a2a_t/negotiation/common/constants.py` — the three extension URI constants, none matching spec strings exactly (§4.5, §6.2)
- `src/a2a_t/negotiation/common/enums.py` — `NegotiationType` (`CLARIFICATION`/`FEASIBILITY`/`FULFILLMENT`/`INFORMATION`) vs. spec's `Target`/`Feasibility`/`Information` (§6.1)
- `src/a2a_t/negotiation/types/{base,clarification,feasibility,fulfillment,information}.py` — type dispatch and (mostly absent) behavior (§6.3)
- `src/a2a_t/negotiation/rendering/negotiation_prompt_renderer.py:8-10` — identity-function renderer, unused by any type-specific content (§6.3)
- `src/a2a_t/negotiation/handling/negotiation_handler.py:200-213` — `_build_result_map`, the single `TASK_PROMPT_KEY`-keyed payload builder used for every scenario including `subscribe_incident` (§5.2)
- `package_data/prompt_resources/templates/{subscribe_incident,energy_saving,private_line_complaint,fault_diagnosis}/zh-CN/template.md` — the four current scenario templates (§4.1-§4.3, §5.2, §8.3)
- `package_data/prompt_resources/slots/{subscribe_incident,energy_saving,private_line_complaint,fault_diagnosis}/zh-CN/slot.json` — slot schemas, `zh-CN`-only (§9 finding 11)
- `package_data/prompt_resources/prompts/{clarification,feasibility,fulfillment,information}_negotiation/{en-US,zh-CN}/{system,user}.md` — shipped, mostly-dead negotiation-type prompt resources (§6.3)
- `package_data/prompt_resources/scenarios/{en-US,zh-CN}/scenarios.json` — the fixed four-scenario list
- `package_data/env.example` — `A2AT_LANGUAGE=en-US` default, contradicted by `zh-CN`-only template/slot resources (§9 finding 11)
- `CLAUDE.md` — documented client-side-only language fallback behavior, referenced in §9 finding 11
