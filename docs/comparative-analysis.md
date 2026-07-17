# A2A-T SDK Comparative Analysis

## 1. Executive Summary

`a2a-t-sdk-python` (import name `a2a_t`) is a standalone, dependency-light Python library that implements one narrow slice of the A2A-T telecom protocol family:
- an LLM-driven pipeline that turns user input into a rendered task prompt (`client/prompt_generation`),
- validates a processed prompt against JSON-schema and semantic rules (`server/prompt_compliance`),
- and drives a stateful multi-round negotiation exchange (`negotiation/`, wired through `client/negotiation` and `server/negotiation`).

It has no HTTP/gRPC transport, no `AgentCard` type, no dependency on the official `a2a-sdk` package, and no agent registry, event, signing, or trust layer. Its own documentation explicitly disclaims responsibility for "Agent HTTP service framework, registry-center client, authentication, or key management capabilities."

The headline conclusion: **`a2a-t-sdk-python` is not, today, a conformant implementation of the official A2A protocol's extension mechanism, and it covers roughly one-and-a-half of the four extension profiles (Task-T partially, Negotiation-T partially) that the reference A2A-T implementation treats as core.** 

Against the official A2A spec, it is missing every wire-level mechanism an extension needs to declare itself (`AgentCard.capabilities.extensions[]`), activate per request (`A2A-Extensions` header / `requested_extensions`), and carry its payload on a real `Message`/`Artifact` object (`metadata[uri]` + `extensions[]`) — its negotiation payload is instead a bespoke, TM-Forum-URL-keyed dict with no relationship to A2A's wire format.

Against the Catalyst A2A-T implementation (`/home/lorenzo/git/A2A-T`):
- it lacks Event-T entirely,
- has no AgentCard/signing/mTLS story,
- no Fabric-T registry/discovery/reputation layer, (OPTIONAL)
- no ASGI/WSGI serving helpers,

Though several of these appear to be intentional scope boundaries (the SDK's own docs disclaim HTTP hosting and registry-client responsibilities) rather than oversights.

**The good news: what exists is well-factored (clear orchestrator/builder pipelines, a documented failure-result convention, a pluggable negotiation state store) and several of the identified gaps are additive — they do not require rearchitecting the existing prompt-generation/compliance/negotiation core, only building an interop layer on top of it.**

## 2. Methodology

This analysis synthesizes six research passes:
1. a read of the official A2A protocol specification (`docs/specification.md`, `specification/a2a.proto`, `docs/topics/extensions.md`) sourced from the `a2aproject/A2A` GitHub repository — the canonical source `a2a-protocol.org` is generated from — after a direct fetch of `a2a-protocol.org` was intercepted by a non-canonical redirect;
2. a direct read of the governance repo at `/home/lorenzo/git/A2A` (`a2a.proto`, ADRs, `docs/topics/extension-and-binding-governance.md`);
3. a source read of the official reference SDK at `/home/lorenzo/git/a2a-python` (`src/a2a/extensions/`, `server/context.py`, `utils/errors.py`) to see how a real implementation operationalizes the spec;
4. a structural audit of this repository (`/home/lorenzo/git/openan/a2a-t-sdk-python`) via `grep`/direct file reads against `src/a2a_t`;
5. a full read of the Catalyst A2A-T implementation at `/home/lorenzo/git/A2A-T` (`a2a_telecom`, `a2a_telecom_sdk`, `a2a_fabric` Python packages, its `pyproject.toml`/`uv.lock`, and its test suite, which passes 510/510);
6. a read of A2A-T's `docs/` and `examples/` trees (developer guide, SDK guide, scenario walkthroughs, REST API reference, BPMN mapping).

All findings below are evidence-backed with `file:line` citations gathered during these passes; every finding supplied in the source research has been carried into this document.

Spec versions consulted:
- the official A2A protocol spec has no single version number in the excerpts read (governed by the `a2aproject/A2A` TSC process, "Experimental" extension-governance track). 
- the Catalyst A2A-T; this repo's own docs cite TM Forum **IG1453A v1.0.0** and **IG1453 v2.0.0**, while the reference A2A-T implementation at `/home/lorenzo/git/A2A-T` cites the newer **IG1453A v1.1.0** and **IG1453 v2.1.0** — a version gap discussed further in §5.15 below.

**Update pass (2026-07-03):** \
This document was subsequently cross-checked against three internal analysis documents covering an earlier snapshot of this same repository at `/home/lorenzo/git/openan-catalyst/a2a-t-sdk/`:
- `CODE_ANALYSIS.md`
- `CODE_COMPARISON.md`
- `PROTOCOL_COMPARISON.md`.

Several of their findings are now stale and were **not** carried forward, because the underlying issues have since been fixed on this repo's `main`: the `>=3.14` Python requirement was already lowered to `>=3.12`; the unused `a2a-sdk`/`fastapi`/`uvicorn`/`sqlalchemy`/`sse-starlette`/`anthropic`/`google-genai`/`google-cloud-modelarmor` dependencies were removed from `pyproject.toml` entirely rather than left dangling; the negotiation URIs were corrected from `github.com/a2aproject/telecommunication/...` to `projects.tmforum.org/a2aproject/telecommunication/...` (commit `492c1af`, "fix(prompt): correct prompt url from github to tmforum"); the DeepSeek-specific LLM adapter was generalized into a provider-agnostic `openai`-compatible client (`llm/providers/openai.py`, `llm/factory.py`); and the previously-reported broken `energy_saving` scenario reference is fixed — the SDK now ships four complete scenarios (`subscribe_incident`, `energy_saving`, `private_line_complaint`, `fault_diagnosis`), each with full slot schemas and templates, not one. The genuinely new material those three documents surfaced and this repo's own analysis had not yet covered — a systematic per-A2A-concept walkthrough of task lifecycle, streaming, push notifications, context threading, authentication, and error-type translation, plus a scenario-coverage comparison — has been re-verified against current HEAD and folded into §4.11-§4.16 and §5.16 below.

## 3. Architecture Comparison

| Dimension | `a2a-t-sdk-python` (this repo) | A2A-T reference (`a2a_telecom` / `a2a_telecom_sdk`) | Official `a2a-python` SDK |
|---|---|---|---|
| **AgentCard support** | None — no `AgentCard`/`AgentExtension`/`AgentCapabilities` type anywhere in `src/` | Two tiers: dict-based `build_agent_card`/`add_extension`/`validate_agent_card` (`a2a_telecom/agent_card.py`), and a real, JWS-signed `a2a.types.AgentCard` proto builder (`a2a_telecom_sdk/agent.py:_build_agent_card`, ~948-1019) | Canonical typed source: `AgentCapabilities`/`AgentExtension` protobuf messages (`a2a_pb2.pyi:235-257`); `find_extension_by_uri(card, uri)` helper (`extensions/common.py:21-27`) |
| **Extension declaration** | None — the three TM-Forum-URL constants in `negotiation/common/constants.py` are never surfaced as `AgentExtension` entries | Declared via `build_agent_extensions()` (`a2a_telecom/extension.py:134-160`) and the real `AgentExtension(uri=..., description=...)` list built in `agent.py:_build_agent_card` | `AgentCapabilities.extensions: repeated AgentExtension` is the sole declaration surface (spec 4.6.1) |
| **Extension activation (request-time)** | None — no HTTP client/server, no header parsing, no `requested_extensions` concept anywhere (`grep` confirms zero hits) | Implemented at the dict level: `get_active_extensions(headers)` / `build_extensions_header(uris)` (`a2a_telecom/extension.py:50-83`); **outbound header is wired** (`fabric_client.py:541`, `async_fabric_client.py:462`), but **inbound header parsing is dead code** — live dispatch instead sniffs `Part.metadata` keys directly (`agent.py:_dispatch_handlers` ~862, `asgi.py:_dispatch` ~151) | Canonical: `A2A-Extensions` HTTP header / gRPC metadata, parsed by `get_requested_extensions()` and exposed as `ServerCallContext.requested_extensions` → `RequestContext.requested_extensions` (`server/context.py:25`, `server/agent_execution/context.py:163-166`); pure set-membership check, no decorator/middleware |
| **Required-extension enforcement** | None — no `required` flag, no error type | `required` field exists on `AgentExtension` payloads but nothing in A2A-T's Python code inspects it | `ExtensionSupportRequiredError` fully wired across JSON-RPC (-32008)/REST(400)/gRPC(FAILED_PRECONDITION) transports (`utils/errors.py:103-106`), but the SDK itself never raises it — that's left to the extension author |
| **Task/Message modeling** | Free-text rendered prompts from scenario/slot/template resources; no typed task object survives generation (`client/prompt_generation/models.py`: `PromptGenerationResult` has only `success`/`prompt_text`/`failure`) | Typed `TelecomTask` dataclass (`task_t.py:117-325`): `target_type`, `target_object`, `environmental_info`, `constraints`, `expected_output`, `interaction_mode`, `intent_3gpp` (3GPP TS 28.312 passthrough), eTOM-aligned `TelecomTaskType` enum | Canonical `Message`/`Part`/`Artifact`/`Task` protobuf messages (`a2a.proto`); `Part` is a `oneof{text\|raw\|url\|data}` |
| **Negotiation** | Bespoke, protocol-level state machine: `NegotiationHandler.start/receive/continue_`, `NegotiationType` (CLARIFICATION/FEASIBILITY/FULFILLMENT/INFORMATION — only INFORMATION wired end-to-end), 8-round safety cap (`MAX_IN_PROGRESS_NEGOTIATION_ROUND`), pluggable `NegotiationStateStore` (in-memory only today). Payload is a raw `dict` keyed by TM-Forum extension URLs (`_build_result_map`, `negotiation_handler.py:200-213`), not a real `Message` | Typed `Negotiation` oneof envelope (`negotiation_t.py`) covering TARGET/FEASIBILITY/INFORMATION, `serialize_negotiation`/`deserialize_negotiation`; composed onto **real** `Message.parts[].metadata[uri]` via `_build_extension_part` (`_helpers.py:70-78`); session tracking via `NegotiationManager` — simple in-memory dict, **no pluggable store** (weaker than this SDK here) | No negotiation concept — this is A2A-T-specific vocabulary layered on top of the base protocol via extensions |
| **Fabric / agent registry & discovery** | None — `grep` for "fabric"/"registry" finds only an unrelated docstring | Two tiers: in-memory `AgentRegistry` (`a2a_fabric/registry.py`, register/update/deregister/discover, 18 tests), and an HTTP+mTLS `FabricClient` (`a2a_telecom_sdk/fabric_client.py`) wrapping a specific remote deployment (`agent-fabric-api.boyle-rc.org`) | Out of scope — registry/discovery is not part of the base A2A spec; left to deployment-specific infrastructure |
| **Event-T (subscription/notification)** | None — zero trace anywhere in `src/` | Full profile: `EventSubscription`/`TelecomEvent`/`NotificationCondition`/`AssociatedObject` dataclasses, `SubscriptionManager`, `match_event` (`event_t.py`, 270 lines) | Not applicable — telecom-specific extension concept |
| **Trust / signing / reputation** | None | AgentCard JWS signing + mTLS `SSLContext` builder, fully implemented and tested (`a2a_telecom_sdk/signing.py`, 208 lines, 16 tests). Reputation is a **read-only** client over externally-computed scores (`reputation.py`, 56 lines) — no scoring algorithm exists anywhere in A2A-T. TrustedFlow/SecurePassport per-message attestation is **spec-only**: proto definitions exist (`a2a_secure_passport.proto`, `a2a_trusted_flow.proto`) but zero Python implementation — it relies on an externally-hosted `stage-api.mef.dev` service | Out of scope for the base extension mechanism; a2a-python provides no signing/reputation primitives either |
| **Transport (HTTP/ASGI/WSGI/gRPC)** | None — pure in-process facades (`A2ATClient`, `A2ATServer` have no network dependency) | `wsgi.py` (239 lines, 16 tests) and `asgi.py` (217 lines, 9 tests) provide genuine, portable inbound dispatch, decoupled from the remote Fabric | Full JSON-RPC/HTTP, gRPC, and REST bindings are the SDK's core purpose; `A2A-Extensions` transmitted as HTTP header or gRPC metadata per binding |
| **Docs / examples maturity** | Two-language developer/user guides (`docs/en`, `docs/zh`) documenting prompt generation, compliance, and information-negotiation only; explicitly disclaims HTTP/registry/auth scope; cites **older** TM Forum spec versions (IG1453A v1.0.0, IG1453 v2.0.0) | Much larger: `developer-guide.md` (primitives), `sdk.md` (628-line facade walkthrough), `docs/scenarios/*` (multi-agent JSON wire-level walkthroughs), `docs/rest-api/*` (Fabric-T HTTP reference), `docs/bpmn-mapping/*` (IG1453B process-mapping spec), 4 runnable SDK examples + `fault_diagnosis_agent` reference agent; cites **newer** spec versions (IG1453A v1.1.0, IG1453 v2.1.0). Marked "Experimental/1.0.0-draft" but functionally mature (v0.8.0, 510 passing tests) | Extensive spec-level docs (`docs/topics/extensions.md`, `docs/specification.md`) but **no example extension implementation** anywhere in the reference SDK repo — the activation/error plumbing is tested, but there's no sample extension to imitate |

## 4. Gaps to Being a Full A2A Protocol Extension Implementation

This section evaluates `a2a-t-sdk-python` strictly against the official A2A specification's extension mechanism (Section 4.6, service parameters, `AgentExtension`/`Message`/`Artifact` shapes) and the reference `a2a-python` SDK's operationalization of it.

### P0 — Blocking conformance gaps

#### 4.1 No AgentCard / AgentExtension declaration surface exists anywhere

**What's missing:** A conformant A2A extension must be declared via `AgentCard.capabilities.extensions[]`, each entry an `AgentExtension{uri, description, required, params}` object.

**Spec/reference evidence:** `a2a.proto`: `AgentCapabilities` (line 411, `repeated AgentExtension extensions = 3`), `AgentExtension` message (lines 423-432, four fields: `uri`, `description`, `required`, `params: google.protobuf.Struct`); spec §4.6.1 "Extension Declaration" (`specification.md:1002`) and §4.4.4 "AgentExtension" (`specification.md:913`).

**Current-code evidence:** `grep` for `AgentCard|AgentExtension|AgentCapabilities` across `src/` returns zero hits. `docs/en/user_guide.md:99` states explicitly: "the SDK is responsible for A2A-T prompt generation, validation, and negotiation, and is not responsible for AgentCard registration, authentication, service hosting, or business execution" (mirrored in `docs/zh/用户指南.md`).

**Governance constraint that should shape any fix:** the A2A governance doc's "SDK Support" guidance (`docs/topics/extension-and-binding-governance.md`) states that SDKs MAY implement extensions, but where they do, extension support MUST be **disabled by default and require explicit opt-in**, and extension support is not required for protocol conformance. Any `AgentExtension`-declaration helper this SDK adds should default to inert/undeclared and require the host application to explicitly opt a URI into its `AgentCard`, matching this posture rather than auto-registering.

**Recommendation:** Add a helper (or documented static `AgentExtension` payload) that a host agent can drop into its own `AgentCard.capabilities.extensions[]`, covering each of the three constants in `negotiation/common/constants.py` with `uri`/`description`/`required`/`params` — or explicitly scope the SDK as "extension logic only, caller owns the AgentCard" in the top-level README rather than only in a translated user guide. Whatever shape this takes, keep it opt-in per the governance guidance above.

#### 4.2 No request-time extension activation/negotiation mechanism (A2A-Extensions header)

**What's missing:** A conformant extension participant must read/write the `A2A-Extensions` HTTP header (or gRPC metadata equivalent) to know which extensions are active for a given request.

**Spec/reference evidence:** Spec §3.2.6 "Service Parameters" (`specification.md:477`), §9.2 "Service Parameter Transmission" (`specification.md:2240`), §14.2.2 "A2A-Extensions Header" (`specification.md:3305`); reference SDK `src/a2a/extensions/common.py` (`HTTP_EXTENSION_HEADER`, `get_requested_extensions`) and `src/a2a/server/context.py` (`ServerCallContext.requested_extensions`).

**Current-code evidence:** `grep -rn 'A2A-Extensions|requested_extensions|grpc|jsonrpc' src/` (excluding `.venv`) returns zero hits. `src/a2a_t/client/a2at_client.py:21-58` confirms `A2ATClient`'s constructor wires only an LLM client and local orchestrators — no transport/network dependency.

**Recommendation:** If this SDK is meant to sit behind an `a2a-python`-based (or any A2A-compliant) transport, document/require the host application to gate calls into `A2ATClient`/`A2ATServer` on `EXT_URI in context.requested_extensions`, and to raise/propagate `ExtensionSupportRequiredError` itself; the SDK currently offers no such integration point.

#### 4.3 Negotiation payload is a bespoke dict, not a Message/Artifact with `extensions[]` + `metadata`

**What's missing:** The spec defines exactly two attachment points for extension data on the wire: `Message.extensions` (repeated string, field 7) + `Message.metadata` (Struct, field 6), and the equivalent pair on `Artifact` (`extensions` field 6, `metadata` field 5).

**Spec/reference evidence:** Spec §4.6.2 "Extension Points" (`specification.md:1077`); `a2a.proto:260-293`.

**Current-code evidence:** `src/a2a_t/negotiation/handling/negotiation_handler.py:200-213` — `_build_result_map` returns `{NEGOTIATION_TEXT_KEY: prompt_text, NEGOTIATION_CONTEXT_KEY: context.to_context(), [TASK_PROMPT_KEY: ...]}` as the literal top-level payload — the extension-URI-shaped constants are used as top-level dict keys, not nested inside a `Message.metadata` map with the URI also listed in `Message.extensions[]`. `pyproject.toml` dependencies block (lines 24-28) lists only `jsonschema`/`python-dotenv`/`openai`; no `Message`, `Artifact`, `Task`, or `Part` type is imported or modeled anywhere in the codebase.

**Related hard limitation to respect when fixing this:** the spec explicitly forbids extensions from changing the definition of core data structures (no new/removed required fields on `Message`/`Task`/`Part`/`Artifact`) or adding new enum values — all extension-specific customization MUST instead live inside the existing generic `metadata` map (`docs/topics/extensions.md`, "Limitations" section). Any fix here must route payload data through `metadata`, never through new fields bolted onto the core proto messages.

**Recommendation:** Change the orchestrator's output to either (a) accept/return actual `a2a-sdk` `Message`/`Artifact` objects and populate `.metadata[uri]` + `.extensions.append(uri)`, or (b) clearly document the current dict as an internal/pre-transport representation that the caller MUST translate before putting it on the A2A wire, and provide a translation helper.

#### 4.4 No dependency on, or type-level interop with, the official `a2a-sdk`

**What's missing:** Any conformant extension needs the canonical types (`AgentExtension`, `AgentCapabilities`, `Message`, `Artifact`, `ServerCallContext`, `ExtensionSupportRequiredError`) that `a2a-python` defines.

**Current-code evidence:** `pyproject.toml:24-28` — `dependencies = [jsonschema, python-dotenv, openai]`, no `a2a-sdk`; `uv.lock` has no `a2a`/`a2a-sdk` package entry. None of the reference SDK types have a corresponding import or reimplementation anywhere in `src/a2a_t`.

**Recommendation:** Either add an optional `a2a-sdk` extra/dependency and a thin adapter module (e.g. `a2a_t.a2a_interop`) that converts `A2ATClient`/`A2ATServer` I/O into/out of `a2a-sdk` `Message`/`AgentCard` objects and wires `ExtensionSupportRequiredError`, or explicitly document in the README that this package is a standalone prompt/negotiation library requiring separate integration work to be used as an A2A extension, and is not a drop-in `a2a-python` plugin.

### P1 — Significant conformance gaps

#### 4.5 Extension URI constants are inconsistent with the spec's URI versioning convention

**What's missing:** Spec §4.6.3 "Extension Versioning and Compatibility" (`specification.md:1137`) states extensions SHOULD embed a version segment in their URI, and a new URI MUST be minted for breaking changes.

**Current-code evidence:** `src/a2a_t/negotiation/common/constants.py:4` — `NEGOTIATION_TEXT_KEY = "https://projects.tmforum.org/a2aproject/telecommunication/extensions/NEGOTIATION-T"` has no version suffix, while `NEGOTIATION_CONTEXT_KEY` and `TASK_PROMPT_KEY` (lines 5-6) both end in `/v1`.

**Related versioning rule with no counterpart in this SDK:** the spec also specifies what a conformant agent must do on a version *mismatch* for a non-required extension — it SHOULD ignore the extension and proceed without it, and it explicitly MUST NOT automatically fall back to a previous/different version of the extension (spec §4.6.3). Since this SDK has no activation mechanism at all (§4.2), it has no code path that could honor or violate this rule today, but it should be designed in from the start once activation is added, rather than defaulting to silent version-fallback behavior.

**Recommendation:** Append a version segment to `NEGOTIATION_TEXT_KEY` (e.g. `.../NEGOTIATION-T/v1`) for consistency, and adopt a documented policy (a new URI per breaking change, no automatic fallback across versions) before any of these constants are treated as stable public extension identifiers.

#### 4.6 Extension URIs use a non-governed namespace, disqualifying them as "official" A2A extensions

**What's missing:** Official/first-party A2A extensions under TSC governance use the `https://a2a-protocol.org/extensions/` URI prefix, live in `a2aproject`-org repos prefixed `ext-`/`experimental-ext-`, and require a TSC graduation vote.

**Spec/reference evidence:** A2A `docs/topics/extension-and-binding-governance.md:19-21,106-115`.

**Current-code evidence:** `src/a2a_t/negotiation/common/constants.py:4-6` — all three URIs are rooted at `https://projects.tmforum.org/a2aproject/telecommunication/extensions/`, a TM-Forum-controlled namespace with no relationship to A2A governance.

**Recommendation:** If "official A2A extension" status is actually being sought, either register through the A2A TSC governance process and adopt the `a2a-protocol.org/extensions/` prefix, or drop the "official" framing and describe these as independently-published, TM-Forum-hosted extension identifiers (which the spec allows without governance).

#### 4.7 No handling of required-extension enforcement (`ExtensionSupportRequiredError`)

**What's missing:** If an `AgentExtension` is declared `required: true` and a client's request doesn't declare support, the agent MUST return `ExtensionSupportRequiredError`.

**Spec/reference evidence:** Spec §3.3.4 "Capability Validation" (`specification.md:576`: "the agent MUST return ExtensionSupportRequiredError"; JSON-RPC code -32008); reference SDK `src/a2a/utils/errors.py:103-106` (`class ExtensionSupportRequiredError(A2AError)`).

**Current-code evidence:** `grep -rn 'ExtensionSupportRequired' src/` — zero hits. No `required: bool` field exists on any of the three constants or negotiation types.

**Recommendation:** If/when request-time activation is added (§4.2 above), also add an `ExtensionSupportRequiredError`-equivalent and a requiredness flag per negotiation type so server orchestrators can enforce it. As with §4.1, keep this disabled-by-default/opt-in per the governance guidance on SDK extension support, since it is not required for protocol conformance.

#### 4.8 Cannot be plugged into an `a2a-python`-based agent server today without hand-written glue code

**What's missing:** Synthesizing 4.1-4.4: to serve one of this SDK's three "extensions" from an `a2a-python` `AgentExecutor` today, an integrator must hand-write: (1) an `AgentCard.capabilities.extensions[]` entry per URI; (2) `A2A-Extensions` header/`requested_extensions` gating logic before calling `A2ATServer`; (3) translation of `_build_result_map`'s raw dict into a real `Message` with `.metadata[uri]` set and `.extensions` populated; (4) their own `ExtensionSupportRequiredError` raising.

**Current-code evidence:** `src/a2a_t/server/a2at_server.py` and `src/a2a_t/client/a2at_client.py` are pure local facades with no transport/type interop code; `docs/en/user_guide.md:99` disclaims AgentCard/hosting/business-execution responsibility without providing an integration recipe; no example/sample directory exists anywhere in the repo; `grep -rn 'AgentExecutor'` across `src/` returns zero hits.

**Recommendation:** Ship a minimal adapter module and/or example (even in `docs/`) showing exactly how to wire `A2ATClient`/`A2ATServer` output into `a2a-python`'s `Message`/`AgentCard`/`ServerCallContext` types, since this is currently entirely unaddressed.

### P2 — Minor / cosmetic conformance gaps

#### 4.9 Terminology collision: SDK's "negotiation" state machine is unrelated to A2A's extension-activation negotiation

**What's missing:** The A2A spec uses "negotiation" informally to describe the client/agent exchange over the `A2A-Extensions` header to determine which extensions are active for a request (`docs/topics/extensions.md:156-169`, "Extension Activation"). This SDK's `negotiation/` module implements a completely different, protocol-level concept: a stateful, multi-round clarification/feasibility/fulfillment/information exchange that is itself the *payload* of an A2A message, not a mechanism for turning A2A extensions on or off.

**Current-code evidence:** `src/a2a_t/negotiation/common/enums.py` / `negotiation/handling/negotiation_handler.py` — `NegotiationType`/`NegotiationStatus`/start-receive-continue_ state machine.

**Recommendation:** Add a short clarifying note in the SDK's README/architecture docs distinguishing "A2A-T negotiation" (this SDK's domain protocol) from "A2A extension activation negotiation" (the spec's per-request header handshake, unimplemented here) to avoid confusing readers evaluating spec conformance.

#### 4.10 No response-side activation echo, and no per-object `extensions[]` population

**What's missing:** Per `docs/topics/extensions.md:168-169`, a conformant agent's response SHOULD echo back an `A2A-Extensions` header listing which extensions were actually activated. Separately, none of the three URI constants are ever appended to an `extensions: list[str]` field on any object.

**Current-code evidence:** `grep -rn 'extensions' src/a2a_t --include=*.py` — only matches inside `common/constants.py`'s own key names; no list/array field named `extensions` exists on any model in `negotiation/common/models.py` or elsewhere.

**Recommendation:** Once the payload is reshaped into a real `Message`/`Artifact` (§4.3), populate its `extensions` list with the relevant URI(s) as a matter of course.

### P1 — Additional protocol-integration gaps (added in update pass)

#### 4.11 No mapping between A2A-T's negotiation state and the A2A `TaskState` lifecycle

**What's missing:** The A2A protocol's task lifecycle is a defined `TaskState` enum — `SUBMITTED`, `WORKING`, `INPUT_REQUIRED`, `COMPLETED`, `FAILED`, `CANCELED`, `REJECTED`, `AUTH_REQUIRED` — that any A2A-aware caller reads to know what state a task is in. This SDK's own negotiation state machine (`NegotiationStatus.IN_PROGRESS`/`AGREED`/`REJECTED`) is a *different, second* state machine with no documented or coded correspondence to `TaskState`.

**Spec/reference evidence:** `src/a2a/types/a2a_pb2.pyi:18-28` (`TaskState` enum: `TASK_STATE_SUBMITTED`/`WORKING`/`COMPLETED`/`FAILED`/`CANCELED`/`INPUT_REQUIRED`/`REJECTED`/`AUTH_REQUIRED`); mirrored in `src/a2a/compat/v0_3/a2a_v0_3.proto:155-176`.

**Current-code evidence:** `grep -rn 'TaskState' src/a2a_t` — zero hits; `src/a2a_t/negotiation/common/enums.py`'s `NegotiationStatus` has no `TaskState`-mapping helper, constant, or docstring anywhere in the codebase.

**Recommendation:** This is downstream of §4.3/§4.4 (no `a2a-sdk` types are used at all), but distinct enough to call out on its own: once the interop adapter proposed in Milestone 1 exists, document (at minimum) a mapping table — `IN_PROGRESS` → `INPUT_REQUIRED`, `AGREED` → `WORKING`/`COMPLETED`, `REJECTED` → `REJECTED`, round-8 force-reject → `REJECTED` — so integrators aren't left to invent this correspondence themselves.

#### 4.12 Custom exception hierarchy has no mapping to A2A JSON-RPC error types

**What's missing:** Beyond `ExtensionSupportRequiredError` (already covered in §4.7), the A2A spec defines a broader error-code surface with fixed HTTP/gRPC/JSON-RPC mappings: `TaskNotFoundError`, `TaskNotCancelableError`, `PushNotificationNotSupportedError`, `UnsupportedOperationError`, `ContentTypeNotSupportedError`.

**Spec/reference evidence:** `src/a2a/utils/errors.py:1-60` — `A2AError` base class plus the five subclasses above, each with a fixed `message` and (further down the file) transport-specific status-code mappings.

**Current-code evidence:** This SDK's own exception types (e.g. `NegotiationInputError`, `NegotiationStateError`, `NegotiationTerminalStateError`, `LLMConfigError`, `LLMRuntimeError`, `PromptResourceNotFoundError`) have no translation layer to any `a2a.utils.errors` type — consistent with §4.4's finding that no `a2a-sdk` symbol is imported anywhere in `src/a2a_t`.

**Recommendation:** When the `a2a_t.a2a_interop` adapter from Milestone 1 is built, add an explicit SDK-exception → `A2AError`-subclass mapping table so a host `AgentExecutor` can translate `a2a_t` failures into spec-correct JSON-RPC error responses instead of letting raw `a2a_t` exceptions propagate uncaught.

### P2 — Additional minor protocol gaps (added in update pass)

#### 4.13 No streaming (SSE) support — negotiation is synchronous request/response only

**Description:** Agents advertising `capabilities.streaming` push incremental `Task`/`TaskStatusUpdateEvent`/`TaskArtifactUpdateEvent` objects through an `EventQueue` (`src/a2a/server/events/event_queue.py`). Every `A2ATClient`/`A2ATServer` negotiation round in this SDK (`start_negotiation`/`receive_negotiation`/`continue_negotiation`) is a single synchronous call returning one dict — there is no concept of incremental/streamed negotiation progress, and no `sse-starlette`-style dependency is present in the current `pyproject.toml` to build one from.

**Current-code evidence:** `grep -rn 'EventQueue\|stream' src/a2a_t` — zero hits.

**Recommendation:** Low priority given the SDK's synchronous, in-process design; only worth adding if/when an interop adapter needs to expose multi-round negotiation as an A2A streaming response rather than one message per round.

#### 4.14 No push notification support

**Description:** A2A defines `tasks/pushNotification/set|get` for webhook-based task-state callbacks, backed by `PushNotificationSender`/`PushNotificationConfigStore` abstractions (`src/a2a/server/tasks/push_notification_sender.py`, `push_notification_config_store.py`). This SDK's negotiation-state transitions (`IN_PROGRESS` → `AGREED`/`REJECTED`) have no callback/webhook mechanism — a caller must poll or hold a live reference to the negotiation to observe its outcome.

**Current-code evidence:** `grep -rn 'push_notification\|webhook' src/a2a_t` — zero hits.

**Recommendation:** Low priority; only relevant once the SDK gains actual transport/serving responsibility, which its own docs currently disclaim (§4.8, §5.3).

#### 4.15 No use of A2A `contextId` — `negotiation_id` is a parallel, disconnected identifier

**Description:** A2A's `Message`/`Task` carry a `contextId` used to group multiple tasks into one logical conversation (`src/a2a/server/agent_execution/context.py:34-77`, `RequestContext.context_id`). This SDK's `NegotiationContext.negotiation_id` (a UUID, `negotiation/common/models.py`) serves an analogous "group these exchanges together" role but is generated and tracked entirely independently, with no read/write relationship to any `contextId`.

**Recommendation:** Note this in the eventual interop adapter's design — either derive `negotiation_id` from an inbound `contextId` when one exists, or explicitly document why the two are intentionally kept separate.

#### 4.16 No authentication/security-scheme declaration surface

**Description:** `AgentCard.security_schemes`/`security_requirements` and `AgentSkill.security_requirements` are the spec's mechanism for an agent to declare required auth (`src/a2a/types/a2a_pb2.pyi:218-219,276-277` — `APIKeySecurityScheme`/`HTTPAuthSecurityScheme`/`OAuth2SecurityScheme`/`OpenIdConnectSecurityScheme`/`MTLSSecurityScheme`); the reference SDK also ships client-side `CredentialService`/`InMemoryContextCredentialStore` helpers (`src/a2a/client/auth/credentials.py`) and a server-side `User`/`UnauthenticatedUser` abstraction (`src/a2a/auth/user.py`). This SDK has no `AgentCard` at all (§4.1) and consequently no security-scheme declaration surface; its own docs explicitly disclaim "authentication...capabilities" as out of scope.

**Recommendation:** Tie to §4.1/§5.2 (AgentCard): if/when an `AgentCard` builder is added, expose `security_schemes`/`security_requirements` as pass-through parameters for the host application to populate. This SDK is not the right place to implement credential storage or user authentication itself.

## 5. Improvements to Adopt from A2A-T

This section evaluates `a2a-t-sdk-python` against the reference A2A-T implementation at `/home/lorenzo/git/A2A-T`, which covers a materially larger surface (four extension profiles vs. this SDK's partial coverage of two).

### P0 — Foundational gaps

#### 5.1 No dependency on the official `a2a-sdk` — cannot construct or consume real A2A protocol objects

**Description:** `a2a-t-sdk-python`'s `pyproject.toml` declares only `jsonschema`, `python-dotenv`, and `openai`. A2A-T's `src/python/pyproject.toml` explicitly depends on `a2a-sdk[signing]>=1.0,<2.0`, and its `uv.lock` resolves `a2a-sdk==1.0.3`, giving it real protobuf-backed `Message`, `Part`, `AgentCard`, `AgentExtension`, `AgentCapabilities` types from `a2a.types.a2a_pb2`. Because `a2a-t-sdk-python` has no such dependency, its negotiation/task payloads (keyed by TM Forum extension URLs, `negotiation/common/constants.py:4-6`) are bespoke transport dicts, not real `Message.parts[].metadata[uri]` structures.

**Recommendation:** Add an official `a2a-sdk` dependency (or a documented adapter layer) and compose negotiation/task payloads into real `Message.parts[].metadata[uri]` rather than bespoke dicts, at least as an optional integration path, so downstream consumers can interoperate with standard A2A infrastructure.

**Evidence:** `a2a-t-sdk-python` `pyproject.toml` `dependencies = [jsonschema>=4.23.0, python-dotenv>=1.1.0, openai]`; A2A-T `src/python/pyproject.toml` `dependencies` includes `a2a-sdk[signing]>=1.0,<2.0`; `uv.lock` resolves `a2a-sdk==1.0.3`.

#### 5.2 No AgentCard concept at all — no capability declaration, no extension registration, no signing

**Description:** `a2a-t-sdk-python` has zero AgentCard type, builder, or signing logic anywhere (`grep -ril 'agentcard\|agent_card' src/` returns no matches). A2A-T provides two tiers: a portable dict-based `build_agent_card`/`build_agent_extensions` (`a2a_telecom/agent_card.py`, 138 lines, adds `capabilities.extensions[]` and a `telecomAgentProfile` metadata block), and a real, JWS-signed `a2a.types.AgentCard` protobuf builder (`a2a_telecom_sdk/agent.py:_build_agent_card`, ~948-1019, builds `AgentCapabilities` with `extensions[]`, `AgentProvider`, skills, then calls `sign_agent_card_proto`/`legacy_sign_agent_card_dict`).

**Recommendation:** Introduce an AgentCard builder (at minimum a dict-level utility mirroring A2A-T's `agent_card.py`) that declares supported negotiation/task extension URIs and telecom profile metadata, with a signed variant once/if the `a2a-sdk` dependency is adopted.

**Evidence:** `grep -ril 'agentcard|agent_card' a2a-t-sdk-python/src/` → no matches; A2A-T `a2a_telecom/agent_card.py`; A2A-T `a2a_telecom_sdk/agent.py:947-1019`.

### P1 — Significant capability gaps

#### 5.3 No ASGI/WSGI inbound dispatch helpers — no way to actually serve an agent over HTTP

**Description:** A2A-T ships `a2a_telecom_sdk/wsgi.py` (239 lines) and `asgi.py` (217 lines), with `test_sdk_wsgi.py` (16 tests) and `test_sdk_asgi.py` (9 tests) that parse inbound envelopes into `Message` protos and dispatch registered handlers. `a2a-t-sdk-python` has no equivalent — `docs/en/developer_guide.md` §1.2, line 12 states explicitly: "The SDK does not provide Agent HTTP service framework, registry-center client, authentication, or key management capabilities; these must be integrated by the business system." This is likely intentional scope-narrowing rather than an oversight.

**Recommendation:** If the SDK intends to stay transport-agnostic, consider shipping a thin, optional ASGI/WSGI adapter module (not core-required) mirroring A2A-T's dispatch abstraction, so users aren't forced to write extension-URI routing from scratch. If genuinely out of scope, state this explicitly as a design decision rather than a gap.

**Evidence:** A2A-T `a2a_telecom_sdk/wsgi.py` (239 lines), `asgi.py` (217 lines); `test_sdk_wsgi.py` (16 tests), `test_sdk_asgi.py` (9 tests); `docs/en/developer_guide.md` §1.2, line 12.

#### 5.4 No fabric/agent-registry/discovery layer — agents cannot register, search, or discover peers

**Description:** A2A-T's `a2a_fabric/registry.py` (147 lines) provides a genuine in-memory `AgentRegistry` with `register`/`update`/`deregister`/`discover` methods, tested with 18 tests (`test_fabric_registry.py`). The SDK layer's `fabric_client.py` wraps this as an HTTP+mTLS client against a deployed Fabric service. `a2a-t-sdk-python` has nothing analogous — `grep`ping `src/` for "fabric" or "registry" turns up only an unrelated docstring mention ("Raised when prompt catalog or registry resolution fails" in `prompt/common/errors.py`), with no actual registry implementation.

**Recommendation:** Port at minimum the in-memory `AgentRegistry` (`a2a_fabric/registry.py`) as a portable, dependency-light reference implementation; treat the remote Fabric HTTP client as optional/out-of-scope unless `a2a-t-sdk-python`'s users target the same or an equivalent deployed registry service.

**Evidence:** A2A-T `a2a_fabric/registry.py` (147 lines, 18 tests in `test_fabric_registry.py`); A2A-T `a2a_telecom_sdk/fabric_client.py` (httpx+mTLS registry client); `grep` across `a2a-t-sdk-python/src` for "fabric"/"registry" finds only one unrelated docstring occurrence.

#### 5.5 No Event-T (event subscription/notification) support at all

**Description:** A2A-T defines a full Event-T extension profile in `a2a_telecom/event_t.py` (270 lines) — `EventSubscription`/`TelecomEvent`/`NotificationCondition`/`AssociatedObject` dataclasses. `a2a-t-sdk-python` has zero trace of this concept (`grep -ril event src/` returns no matches). Event-T is one of A2A-T's three core extension profiles (alongside Task-T and Negotiation-T) for scenarios needing subscription/notification rather than only synchronous request/negotiate/response.

**Recommendation:** If event-driven telecom workflows are in scope for `a2a-t-sdk-python`'s roadmap, port a minimal `EventSubscription`/`TelecomEvent` model and matching logic analogous to `event_t.py`. If out of scope, document the omission explicitly the way HTTP-serving is already disclaimed.

**Evidence:** A2A-T `a2a_telecom/event_t.py` (270 lines: `EventSubscription`, `TelecomEvent`, `NotificationCondition`, `AssociatedObject`); `grep -ril 'event' a2a-t-sdk-python/src/` → zero matches.

#### 5.6 Negotiation is a bespoke prompt/dict pipeline, not typed protocol objects composed onto real Message metadata

**Description:** `a2a-t-sdk-python`'s negotiation payloads are plain dicts keyed by TM Forum extension URLs (`negotiation/common/constants.py:4-6`), built/parsed via `NegotiationContext.to_context()`/`from_context()` (`negotiation/common/models.py:39,63`). A2A-T's `negotiation_t.py` defines typed dataclasses (`Negotiation` envelope, `TargetNegotiation`/`FeasibilityNegotiation`) with `serialize_negotiation`/`deserialize_negotiation` (lines 85/147), and at the SDK layer these are composed into real `Message.parts[].metadata[uri]` Structs via `_build_extension_part` in `a2a_telecom_sdk/_helpers.py` (`md.update({uri: payload}); return Part(text=text, metadata=md)`). A2A-T's negotiation payloads are thus directly wire-compatible with the official A2A message envelope, whereas `a2a-t-sdk-python`'s are an internal convention.

**Naming divergence worth reconciling alongside the typed-model work:** A2A-T's reference `Negotiation` envelope covers exactly three types — TARGET, FEASIBILITY, INFORMATION — while this SDK's `negotiation/types/` defines four — CLARIFICATION, FEASIBILITY, FULFILLMENT, INFORMATION (only INFORMATION is wired end-to-end per this repo's own `CLAUDE.md`). `CLARIFICATION` maps loosely onto the reference's `TARGET` negotiation (both concern pinning down an ambiguous target/scope before proceeding), but `FULFILLMENT` has no counterpart in the reference spec at all. This suggests either independent evolution against a different/newer TM Forum draft, or drift that the team has not yet reconciled with the upstream spec — worth a deliberate decision (rename to match, or document the divergence as intentional) before more code accumulates on the current four-type model.

**Recommendation:** Consider introducing typed negotiation payload models as an intermediate representation, with the existing dict format as one possible serialization target and real `Message.metadata` composition as another once `a2a-sdk` is adopted. As part of that work, resolve the TARGET/CLARIFICATION and FULFILLMENT naming divergence noted above.

**Evidence:** `src/a2a_t/negotiation/common/constants.py:4-6`; `src/a2a_t/negotiation/common/models.py` (`NegotiationContext.from_context` line 39, `to_context` line 63); A2A-T `a2a_telecom/negotiation_t.py` (`serialize_negotiation` line 85, `deserialize_negotiation` line 147); A2A-T `a2a_telecom_sdk/_helpers.py:_build_extension_part`; `src/a2a_t/negotiation/types/` (clarification.py, feasibility.py, fulfillment.py, information.py) vs. A2A-T's TARGET/FEASIBILITY/INFORMATION set.

#### 5.7 Task-T lacks a typed domain object model — prompts are free text, not a structured TelecomTask

**Description:** A2A-T's `a2a_telecom/task_t.py` (325 lines) defines a typed `TelecomTask` dataclass with `target_type`, `target_object`, `environmental_info`, `constraints`, `expected_output`, `interaction_mode` fields (lines 117-130), an eTOM-aligned `TelecomTaskType` enum, and `NetworkDomain`/`NetworkLayer` enums. `a2a-t-sdk-python`'s client-facing `PromptGenerationResult` (`client/prompt_generation/models.py`) only carries `success: bool`, `prompt_text: str | None`, `failure: PromptGenerationFailure | None` — no typed task object survives past generation. This means `a2a-t-sdk-python` cannot programmatically route, filter, or validate on task type/domain/layer the way a consuming agent could with A2A-T's typed model.

**Recommendation:** Consider surfacing a typed intermediate task representation (task type, domain, layer, constraints) alongside the rendered prompt text in `PromptGenerationResult`, so consumers can route/filter programmatically rather than only re-deriving structure from text or slots.

**Evidence:** A2A-T `a2a_telecom/task_t.py:117-130` (`TelecomTask` dataclass fields including `intent_3gpp`); `a2a-t-sdk-python` `src/a2a_t/client/prompt_generation/models.py` (`PromptGenerationResult` has only `success`/`prompt_text`/`failure`).

### P2 — Lower-priority / conditional gaps

#### 5.8 No reputation subsystem — note A2A-T's own reputation is a thin remote-read client, not a scoring engine

**Description:** `a2a-t-sdk-python` has no reputation concept. A2A-T provides `a2a_telecom_sdk/reputation.py` (56 lines) — a read-only helper pulling a reputation object from AgentCard extension metadata — plus REST passthroughs to a remote Fabric service. `test_fabric_reputation.py` has only 3 tests, and they assert URI constants rather than scoring logic. There is no scoring algorithm anywhere in A2A-T's Python code — reputation is computed by an external authority and only read/relayed by the SDK.

**Recommendation:** Low priority; only port if `a2a-t-sdk-python`'s roadmap includes talking to an external reputation/trust authority. Otherwise this gap is likely intentional given the SDK's narrower embedded-library scope.

**Evidence:** A2A-T `a2a_telecom_sdk/reputation.py` (56 lines); `tests/test_fabric_reputation.py` (3 tests, asserting URI constants only).

#### 5.9 No TrustedFlow/SecurePassport per-message attestation — unimplemented in A2A-T itself, so this is a spec gap, not a code-port opportunity

**Description:** A2A-T's `specification/a2a_secure_passport.proto` and `a2a_trusted_flow.proto` define per-message, multi-hop cryptographic attestation, documented in `docs/trusted-layer/`. A `grep` across all of A2A-T's `src/python` for `TrustedFlow`/`SecurePassport`/`secure_passport`/`trusted_flow` found zero matches — this exists only as a proto spec plus reliance on an externally-hosted stage API (`docs/trusted-layer/trusted-layer-onboarding-guide.md` references `https://stage-api.mef.dev/api/v2/fabric/identity` for sign/verify), not as portable Python code.

**Recommendation:** Do not prioritize porting (there is nothing to port); if trusted-flow attestation becomes a requirement, treat it as new spec-implementation work informed by the proto definitions, not a code migration from A2A-T.

**Evidence:** A2A-T `specification/a2a_secure_passport.proto`, `a2a_trusted_flow.proto`; zero Python implementation matches; `docs/trusted-layer/trusted-layer-onboarding-guide.md` references the remote stage API only.

#### 5.10 AgentCard signing (JWS) and mTLS transport security absent

**Description:** A2A-T's `a2a_telecom_sdk/signing.py` (208 lines) implements JWS signing of AgentCards plus a legacy signer, tested by 16 tests (`test_sdk_signing.py`). `a2a-t-sdk-python` has no signing or transport-security code at all, consistent with its documented scope (`developer_guide.md` §1.2 disclaims "authentication, or key management capabilities"). This is likely intentional — the SDK expects the host application to own transport security — but it means any AgentCard support added per §5.2 would ship unsigned unless this is also adopted.

**Recommendation:** Treat as coupled to the AgentCard finding (§5.2): if/when AgentCard support is added, at least document that signing/mTLS is the host application's responsibility, rather than silently having no story for it.

**Evidence:** A2A-T `a2a_telecom_sdk/signing.py` (208 lines, 16 tests); `docs/en/developer_guide.md` §1.2 disclaimer.

#### 5.11 No request/reply correlation tracking for asynchronous task exchanges

**Description:** A2A-T's `a2a_telecom_sdk/tracking.py` implements `TaskTracker` (68 lines) that registers an outgoing `task_id` and resolves a caller's wait via a correlated inbound message, tested by 13 tests (`test_sdk_tracking.py`). `a2a-t-sdk-python`'s negotiation handler (`negotiation/handling/negotiation_handler.py`) is a stateful start/receive/continue_ state machine but has no analogous "send and block for correlated reply" utility.

**Recommendation:** Consider only if/when `a2a-t-sdk-python` adds an actual transport-sending capability; not urgent while the SDK remains prompt/negotiation-pipeline-only.

**Evidence:** A2A-T `a2a_telecom_sdk/tracking.py` (68 lines, 13 tests); `src/a2a_t/negotiation/handling/negotiation_handler.py` has no reply-correlation/wait mechanism.

#### 5.12 No 3GPP Intent / STRUCTURED / HYBRID interaction mode support (TS 28.312)

**Description:** A2A-T's `task_t.py` supports an `intent_3gpp: dict[str, Any] | None` field (line 130, referenced in deserialization at line 226) carrying arbitrary nested JSON for TS 28.312 intent payloads, tested in `test_3gpp_compat.py`. `a2a-t-sdk-python`'s prompt pipeline has no such field or concept anywhere in `src/`. For telecom network-management scenarios needing to interoperate with 3GPP intent-based systems, this is a real capability gap, though it may be intentional given the SDK's LLM-prompt-centric design focus.

**Recommendation:** If `a2a-t-sdk-python`'s users need interoperability with 3GPP TS 28.312 intent-driven systems, add an optional structured-intent field/passthrough alongside the NL prompt; otherwise note this as an intentional scope boundary.

**Evidence:** A2A-T `a2a_telecom/task_t.py:130,226`; `tests/test_3gpp_compat.py`; no 3GPP/intent mentions found in `a2a-t-sdk-python` `src/`.

#### 5.13 No REST API surface / server reference implementation documented or provided

**Description:** A2A-T's `docs/rest-api/` (`overview.md` and `examples.md`) documents a Fabric-T HTTP surface with agent registration, discovery, message send/stream, catalog, and PKI endpoints. `a2a-t-sdk-python` has no REST API documentation or reference server at all — consistent with its stated scope as an embeddable pipeline library. This is very likely intentional scope narrowing.

**Recommendation:** No action needed unless `a2a-t-sdk-python`'s roadmap shifts toward providing a reference server; if so, A2A-T's `docs/rest-api/` endpoint shape is a reasonable model to follow for consistency with the broader A2A-T ecosystem.

**Evidence:** A2A-T `docs/rest-api/overview.md`, `examples.md`; `docs/en/developer_guide.md` §1.2 disclaimer.

#### 5.14 Informational: inbound A2A-Extensions header parsing exists in A2A-T but is dead code — a pitfall to avoid replicating

**Description:** A2A-T's `a2a_telecom/extension.py` defines `get_active_extensions(headers)` (parses the `A2A-Extensions` HTTP header), but the live inbound dispatch path in `agent.py`'s `_dispatch_handlers` and `asgi.py`'s `_dispatch` instead scan each Message `Part`'s metadata dict keys directly. A repo-wide `grep` confirms `get_active_extensions` is never called anywhere outside its own definition. This is not a capability `a2a-t-sdk-python` is missing (it has no HTTP dispatch layer at all) — it's a known correctness/consistency gap in the reference implementation worth flagging so it isn't copied if `a2a-t-sdk-python` ever builds equivalent inbound extension-routing logic.

**Recommendation:** If `a2a-t-sdk-python` ever adds ASGI/WSGI-style inbound dispatch, ensure the `A2A-Extensions` activation header is actually consulted for routing/validation, not just metadata-key sniffing, to avoid inheriting this inconsistency from A2A-T.

**Evidence:** A2A-T `a2a_telecom/extension.py` (`get_active_extensions()` defined); `a2a_telecom_sdk/agent.py` `_dispatch_handlers` and `asgi.py` `_dispatch` scan `Part.metadata` keys directly; `grep -rn get_active_extensions` across `a2a_telecom_sdk/` and `a2a_telecom/` finds only its own definition, zero call sites.

#### 5.15 Documentation cites an older TM Forum spec version than the reference implementation

**Description:** `a2a-t-sdk-python`'s own documentation references TM Forum spec versions **IG1453A v1.0.0** and **IG1453 v2.0.0**, while the reference A2A-T implementation's documentation references the newer **IG1453A v1.1.0** and **IG1453 v2.1.0**. This is a documentation/traceability gap, not a code defect, but it means a reader cross-referencing this SDK's behavior against the currently-cited upstream TM Forum specification documents in this repo's `docs/` folder (`IG1453_Agent_to_Agent_Protocol_for_Telecoms_v2.1.0.docx`, `IG1453A_Structured_Prompt_of_Agent_to_Agent_Protocol_for_Telecoms_A2AT_v1.0.0.docx` — note the filenames themselves are inconsistent: the IG1453 docx is already v2.1.0 while the IG1453A docx is v1.0.0) may be working from language the guides don't reflect, and it is unclear whether the SDK's implementation choices (e.g. the negotiation type-naming divergence in §5.6) stem from targeting the older draft.

**Recommendation:** Reconcile the developer/user guide version references with the spec documents actually shipped in `docs/`, and evaluate whether adopting IG1453A v1.1.0 / IG1453 v2.1.0 (matching the reference implementation) would resolve some of the structural divergences noted elsewhere in this document (particularly §5.6's negotiation-type naming gap), or whether the older version is an intentional pin that should simply be stated explicitly.

**Evidence:** `a2a-t-sdk-python` developer/user guide spec-version citations (IG1453A v1.0.0, IG1453 v2.0.0) vs. A2A-T reference repo documentation citations (IG1453A v1.1.0, IG1453 v2.1.0); `docs/IG1453_Agent_to_Agent_Protocol_for_Telecoms_v2.1.0.docx` and `docs/IG1453A_Structured_Prompt_of_Agent_to_Agent_Protocol_for_Telecoms_A2AT_v1.0.0.docx` in this repo.

#### 5.16 Scenario/domain coverage: 4 scenarios today vs. A2A-T's ~10 eTOM-aligned task types

**Description:** At the time of the original research pass this document was based on internal notes describing a single built-in scenario (`subscribe_incident`). As of the current checkout, `a2a-t-sdk-python` ships **four** complete scenarios with full slot schemas and templates in both `en-US` and `zh-CN`: `subscribe_incident`, `energy_saving`, `private_line_complaint`, `fault_diagnosis` (`package_data/prompt_resources/{slots,templates}/`). A2A-T's `TelecomTaskType` enum (`a2a_telecom/task_t.py:19-31`) now defines **ten** non-`UNSPECIFIED` values — `FAULT_MANAGEMENT`, `FAULT_REPAIR`, `COMPLAINT_HANDLING`, `PERFORMANCE_OPTIMIZATION`, `ENERGY_EFFICIENCY`, `SERVICE_DELIVERY`, `SERVICE_FEASIBILITY`, `SERVICE_VERIFICATION`, `SERVICE_WORKORDER`, `CONFIGURATION_MANAGEMENT` — a taxonomy that has itself grown since earlier internal notes counted seven, meaning A2A-T's own coverage is a moving target, not a fixed bar. The breadth gap has narrowed materially (1→4 scenarios) but a real gap remains; more structurally, this SDK's scenarios stay free-form `scenario_code` strings validated only by presence in `scenarios.json`, not instances of a shared, typed enum the way A2A-T's are — the same structural point as §5.7, now quantified.

**Recommendation:** No urgent action — this is tracking information, not a defect. If further scenarios are added, consider whether they should map onto A2A-T's `TelecomTaskType` taxonomy (per §5.7's typed-task-model recommendation) rather than continuing to grow as independent free-form codes.

**Evidence:** `package_data/prompt_resources/scenarios/{en-US,zh-CN}/scenarios.json` (4 entries); `package_data/prompt_resources/{slots,templates}/subscribe_incident|energy_saving|private_line_complaint|fault_diagnosis/`; A2A-T `src/python/a2a_telecom/task_t.py:19-31` (`TelecomTaskType`, 10 non-`UNSPECIFIED` values).

## 6. Prioritized Roadmap

The findings above cluster into four milestones. They are ordered so that each milestone either unblocks the next or delivers independently shippable value; none requires rearchitecting the existing prompt-generation/compliance/negotiation core.

**Milestone 1 — Real A2A extension conformance (addresses §4.1-4.4, §4.7, §4.11, §4.12)**
Rationale: until this ships, `a2a-t-sdk-python` cannot legitimately be called an A2A protocol extension — it produces no artifact a conformant A2A client or server can consume or activate.
1. Add an optional `a2a-sdk` dependency and an `a2a_t.a2a_interop` adapter module that converts negotiation/prompt-compliance outputs into real `Message`/`Artifact` objects, keying payload under `.metadata[uri]` and appending the URI to `.extensions[]` (§4.3, §4.4).
2. Add an `AgentExtension`-builder helper exposing the three existing extension URIs with `uri`/`description`/`required`/`params`, for host applications to drop into their own `AgentCard.capabilities.extensions[]` (§4.1).
3. Add a documented convention (and, if the SDK ever gains transport awareness, actual code) for checking `requested_extensions` before dispatch, plus an `ExtensionSupportRequiredError`-equivalent for the `required=true` case (§4.2, §4.7).
4. Fix the version-suffix inconsistency on `NEGOTIATION_TEXT_KEY` and document the URI-per-breaking-change policy (§4.5) — a small, immediate fix that costs nothing and removes one inconsistency before external consumers depend on the URIs.
5. Once the adapter from step 1 exists, document the `NegotiationStatus` → `TaskState` mapping and add an exception → `A2AError`-subclass translation table (§4.11, §4.12) — both are direct, low-cost consequences of adding the interop layer rather than separate design work.

Note on lower-priority protocol surfaces: §4.13 (streaming), §4.14 (push notifications), §4.15 (`contextId` threading), and §4.16 (authentication/security schemes) are optional A2A capabilities, not required for baseline extension conformance. Address them opportunistically alongside whichever milestone already touches the same code — e.g. §4.16 naturally pairs with the `AgentCard` work in step 2 above and in Milestone 4 item 1.

**Milestone 2 — Interop documentation and worked examples (addresses §4.6, §4.8, §4.9, §4.10)**
Rationale: once Milestone 1's primitives exist, the highest-leverage remaining work is making them usable — today there is no worked example anywhere (in this repo or in the upstream reference SDK) showing how to wire a prompt/negotiation library into a live `a2a-python` `AgentExecutor`.
1. Ship a minimal end-to-end example (ideally under `docs/` or an `examples/` directory) demonstrating `A2ATClient`/`A2ATServer` wired into `a2a-python`'s `Message`/`AgentCard`/`ServerCallContext` types (§4.8).
2. Add a short "terminology" note distinguishing A2A-T negotiation from A2A extension-activation negotiation, and settle the "official extension" framing question — either pursue TSC governance registration under `a2a-protocol.org/extensions/`, or explicitly describe the TM-Forum namespace as independently published (§4.6, §4.9). While touching docs, reconcile the cited TM Forum spec version numbers with the documents actually shipped in `docs/` (§5.15).
3. Populate response-side `extensions[]` echoing once real `Message` objects exist from Milestone 1 (§4.10) — a small follow-on, not new design work.

**Milestone 3 — Task-T and Event-T parity with A2A-T (addresses §5.5, §5.6, §5.7, §5.12, §5.16)**
Rationale: these close the largest functional gap versus the reference implementation — an entire missing extension profile (Event-T) and a materially thinner data model for the profile that does exist (Task-T) — and are additive to the current pipeline rather than replacing it.
1. Introduce a typed intermediate task representation (task type, domain, layer, constraints, expected output) alongside the existing rendered prompt text in `PromptGenerationResult`, modeled on A2A-T's `TelecomTask` (§5.7).
2. Introduce typed negotiation payload models as an intermediate representation, keeping the current dict format as one serialization target and adding real `Message.metadata` composition as another (this reuses Milestone 1's interop layer) (§5.6).
3. Port a minimal `EventSubscription`/`TelecomEvent`/`SubscriptionManager` model analogous to A2A-T's `event_t.py`, or explicitly document Event-T as an intentional scope exclusion the way HTTP-serving already is (§5.5).
4. If 3GPP TS 28.312 interoperability is a real user requirement, add an optional structured-intent passthrough field; otherwise record the omission as a deliberate scope boundary (§5.12).

**Milestone 4 — Fabric/trust parity, evaluated selectively (addresses §5.2, §5.3, §5.4, §5.8-§5.11, §5.13, §5.14)**
Rationale: this is the largest remaining surface but the least clearly in-scope — the SDK's own docs already disclaim HTTP hosting, registry-client, and key-management responsibilities, so each item here should be a deliberate scope decision, not an assumed gap to close. Sub-items with no existing reference implementation to port (TrustedFlow/SecurePassport, §5.9; reputation scoring, §5.8) should not be prioritized as "ports" since there is nothing to port — they would be net-new design work.
1. Decide, as a product/architecture question, whether AgentCard construction (with or without signing) belongs in this SDK at all (§5.2, §5.10) — this gates everything else in this milestone, since Fabric registration and Event-T dispatch both assume an AgentCard exists.
2. If transport-agnostic serving is ever wanted, port the ASGI/WSGI dispatch helpers as an optional, non-core adapter module — and unlike A2A-T's own implementation, make sure inbound dispatch actually consults the `A2A-Extensions` header rather than only sniffing `Part.metadata` keys (§5.3, §5.14 — avoid replicating A2A-T's known defect).
3. If agent discovery is needed, port only the in-memory `AgentRegistry` reference implementation (§5.4) — the remote `FabricClient` HTTP wrapper is only useful if a compatible deployed Fabric service exists for this SDK's users, which is an open deployment question, not a code gap.
4. Treat reputation (§5.8), TrustedFlow/SecurePassport (§5.9), request/reply tracking (§5.11), and a REST API reference (§5.13) as low-priority, conditional on concrete downstream demand — none of them block conformance or the existing pipeline's usefulness.

## 7. Appendix: File Reference Index

### `a2a-t-sdk-python` (this repo, `/home/lorenzo/git/openan/a2a-t-sdk-python`)
- `pyproject.toml` (dependencies block, lines 24-28) — confirms no `a2a-sdk` dependency
- `uv.lock` — confirms no `a2a`/`a2a-sdk` package resolved
- `src/a2a_t/client/a2at_client.py:21-58` — `A2ATClient` facade, no network/transport dependency
- `src/a2a_t/server/a2at_server.py` — `A2ATServer` facade, pure local orchestration
- `src/a2a_t/client/prompt_generation/models.py` — `PromptGenerationResult` (only `success`/`prompt_text`/`failure`, no typed task object)
- `src/a2a_t/negotiation/common/constants.py:4-6` — `NEGOTIATION_TEXT_KEY`, `NEGOTIATION_CONTEXT_KEY`, `TASK_PROMPT_KEY` (TM Forum extension URLs)
- `src/a2a_t/negotiation/common/models.py:39,63` — `NegotiationContext.from_context`/`to_context`
- `src/a2a_t/negotiation/common/enums.py` — `NegotiationType`/`NegotiationStatus`/`NegotiationRole`
- `src/a2a_t/negotiation/handling/negotiation_handler.py:200-213` — `_build_result_map`, the bespoke dict payload builder; start/receive/continue_ state machine
- `src/a2a_t/negotiation/store/` — `NegotiationStateStore` protocol + `InMemoryNegotiationStateStore`
- `src/a2a_t/prompt/common/errors.py` — sole (unrelated) "registry" docstring hit in the codebase
- `docs/en/developer_guide.md` §1.2 (line 12) — explicit disclaimer of HTTP service framework/registry-center/auth/key-management scope
- `docs/en/user_guide.md:99` — explicit disclaimer of AgentCard registration/hosting/business-execution responsibility
- `docs/zh/用户指南.md` — Chinese-language mirror of the above disclaimer
- `pyproject.toml` (`dependencies`, `requires-python`) — confirms current dependency set (`jsonschema`, `python-dotenv`, `openai` only) and `>=3.12` Python requirement, both narrower/cleaner than the sibling `PROTOCOL_COMPARISON.md`'s snapshot
- `src/a2a_t/llm/factory.py`, `src/a2a_t/llm/providers/openai.py` — current provider-agnostic `LLMClientFactory`/`OpenAIClient`, superseding the DeepSeek-specific adapter described in the sibling `CODE_ANALYSIS.md`
- `package_data/prompt_resources/scenarios/{en-US,zh-CN}/scenarios.json` — 4 current scenarios (§5.16)
- `git log --oneline -- pyproject.toml src/a2a_t/negotiation/common/constants.py` — commit `492c1af` ("fix(prompt): correct prompt url from github to tmforum") confirming the URI-namespace fix postdates the sibling docs

### Official A2A specification (sourced via `a2aproject/A2A` GitHub, mirroring `a2a-protocol.org`)
- `docs/specification.md:913` — §4.4.4 "AgentExtension"
- `docs/specification.md:998-1150` (esp. `1002`, `1077`, `1137`) — §4.6 "Extensions": declaration (4.6.1), extension points on Message/Artifact (4.6.2), versioning/compatibility (4.6.3)
- `docs/specification.md:477-490` — §3.2.6 "Service Parameters"
- `docs/specification.md:570-577` — §3.3.4 "Capability Validation" (MUST return `ExtensionSupportRequiredError`)
- `docs/specification.md:1189` — error code table incl. `ExtensionSupportRequiredError` = JSON-RPC -32008 / FAILED_PRECONDITION / HTTP 400
- `docs/specification.md:2240-2260` — §9.2 "Service Parameter Transmission" (JSON-RPC/HTTP binding MUST use headers)
- `docs/specification.md:3305-3324` — §14.2.2 formal `A2A-Extensions` header registration
- `specification/a2a.proto:357-398` — `AgentCard` message, `capabilities` field (7, REQUIRED)
- `specification/a2a.proto:410-432` — `AgentCapabilities` (`extensions` field 3) and `AgentExtension` (`uri`/`description`/`required`/`params`)
- `specification/a2a.proto:260-277` — `Message` (`extensions` field 7, `metadata` field 6)
- `specification/a2a.proto:279-293` — `Artifact` (`extensions` field 6, `metadata` field 5)
- `specification/a2a.proto:221-242` — `Part` (`metadata` only, no `extensions` field)
- `specification/a2a.proto:167-184` — `Task` (`metadata` only, no `extensions` field)
- `docs/topics/extensions.md` (full file, 281 lines) — Extension Declaration, Extension Activation (3-step negotiation flow incl. response echo), Required Extensions, Limitations (no core structure changes, no new enum values), Implementation Considerations (versioning, discoverability, security), Governance
- `docs/topics/extension-and-binding-governance.md:19-21,106-115` — official extension URI prefix, `ext-`/`experimental-ext-` repo convention, TSC graduation vote, SDK support requirements (disabled-by-default/opt-in)

### A2A governance repo (`/home/lorenzo/git/A2A`)
- `specification/a2a.proto:422-432` — canonical `AgentExtension` message definition
- `specification/a2a.proto:410-420` — `AgentCapabilities` with `repeated AgentExtension extensions = 3`
- `specification/json/README.md:1-20` — `a2a.json` generation pipeline (non-normative build artifact, no extension-specific handling)
- `adrs/adr-001-protojson-serialization.md` — only substantive ADR present; JSON casing/enum rules; no ADR specific to extensions exists
- `README.md` — confirmed no extension-related content at the top level

### Official reference SDK (`/home/lorenzo/git/a2a-python`)
- `src/a2a/extensions/common.py:1-27` — `HTTP_EXTENSION_HEADER`, `get_requested_extensions()`, `find_extension_by_uri()`
- `src/a2a/extensions/__init__.py` — empty file, no re-exports, no registry/plugin system
- `src/a2a/types/a2a_pb2.pyi:235-257` — `AgentCapabilities.extensions` and `AgentExtension` message
- `src/a2a/server/context.py:14-25` — `ServerCallContext.requested_extensions`
- `src/a2a/server/agent_execution/context.py:163-166` — `RequestContext.requested_extensions` property proxy
- `src/a2a/server/routes/common.py:49-72` — `DefaultServerCallContextBuilder`, HTTP header parsing
- `src/a2a/server/request_handlers/grpc_handler.py:43-82` — `DefaultGrpcServerCallContextBuilder`, gRPC metadata parsing
- `src/a2a/client/service_parameters.py:49-64` — `with_a2a_extensions()` client-side helper
- `src/a2a/compat/v0_3/extension_headers.py:1-28`, `context_builders.py:1-73` — legacy `X-A2A-Extensions` header compat shim
- `src/a2a/utils/errors.py:103-106,140-195` — `ExtensionSupportRequiredError` definition and transport error-code mappings
- `src/a2a/server/request_handlers/response_helpers.py:35,61` — `ExtensionSupportRequiredError` wired into JSON-RPC error responses
- `src/a2a/types/a2a_pb2.pyi:18-28` — `TaskState` enum (§4.11)
- `src/a2a/utils/errors.py:1-60` — `A2AError` base class and `TaskNotFoundError`/`TaskNotCancelableError`/`PushNotificationNotSupportedError`/`UnsupportedOperationError`/`ContentTypeNotSupportedError` (§4.12)
- `src/a2a/server/events/event_queue.py` — `EventQueue`, streamable `Event` type alias (§4.13)
- `src/a2a/server/tasks/push_notification_sender.py`, `push_notification_config_store.py` — push notification abstractions (§4.14)
- `src/a2a/server/agent_execution/context.py:34-77` — `RequestContext.context_id` (§4.15)
- `src/a2a/types/a2a_pb2.pyi:218-219,276-277` — `AgentCard.security_schemes`/`security_requirements`, `AgentSkill.security_requirements` (§4.16)
- `src/a2a/client/auth/credentials.py` — `CredentialService`/`InMemoryContextCredentialStore` (§4.16)
- `src/a2a/auth/user.py` — `User`/`UnauthenticatedUser` (§4.16)
- `tests/extensions/test_common.py` — unit tests for header parsing and AgentCard extension lookup
- `tests/integration/test_client_server_integration.py:618-683,930-980` — end-to-end extension header propagation and `ExtensionSupportRequiredError` transport round-trip
- `tck/` — minimal SUT harness only (`sut_agent.py`, `sut_agent_with_vertex_task_store.py`), no extension conformance tests present

### A2A-T reference implementation (`/home/lorenzo/git/A2A-T`)
- `src/python/a2a_telecom/extension.py:16-160` — extension URI constants, `get_active_extensions`/`build_extensions_header`, `parse_extension_metadata`/`set_extension_metadata`, `build_agent_extensions`
- `src/python/a2a_telecom/agent_card.py:1-138` — dict-based `build_agent_card`/`add_extension`/`validate_agent_card`, `TelecomAgentProfile` dataclass
- `src/python/a2a_telecom/task_t.py:117-325` — `TelecomTask` dataclass, shared ProtoJSON-style serialize helpers, `validate_telecom_task`, `route_by_task_type`
- `src/python/a2a_telecom/task_t.py:19-31` — `TelecomTaskType` enum, 10 non-`UNSPECIFIED` values (§5.16)
- `src/python/a2a_telecom/event_t.py:47-270` — `EventSubscription`/`TelecomEvent` dataclasses, `SubscriptionManager`, `match_event`
- `src/python/a2a_telecom/negotiation_t.py:22-240` — `Negotiation` oneof envelope, `serialize_negotiation`, `NegotiationManager` state machine
- `src/python/a2a_telecom/prompt.py:1-396` — `StructuredPrompt`/`Instruction`/`Variable`, template load/render, IG1453A markdown validation
- `src/python/a2a_telecom_sdk/agent.py:22-52,394-402,862-931,948-1019` — imports, `build_agent_card`, `_dispatch_handlers`, `_build_agent_card` (real proto + JWS signing)
- `src/python/a2a_telecom_sdk/asgi.py:24-41,151-209` — imports, `_dispatch` (metadata-key matching, not header-based)
- `src/python/a2a_telecom_sdk/wsgi.py` (239 lines) — WSGI inbound dispatch
- `src/python/a2a_telecom_sdk/_helpers.py:52-104` — `_part_to_proto`, `_build_extension_part`, `_build_message_proto`
- `src/python/a2a_telecom_sdk/fabric_client.py:101,195-205,538-541,588` — sync `FabricClient` (HTTP+mTLS), outbound `A2A-Extensions` header wiring
- `src/python/a2a_telecom_sdk/async_fabric_client.py:60,180,462` — async twin of `fabric_client.py`
- `src/python/a2a_telecom_sdk/signing.py:1-208` — JWS AgentCard signing (LF + legacy pipelines), mTLS `SSLContext` builder
- `src/python/a2a_telecom_sdk/reputation.py:1-56` — `reputation_from_extended_card` (read-only)
- `src/python/a2a_telecom_sdk/tracking.py:1-68` — `TaskTracker`, async reply correlation
- `src/python/a2a_telecom_sdk/errors.py:1-39` — `SDKError`/`FabricHTTPError`/`RegistrationError`/`PromptNotFoundError`/`PromptRevokedError` hierarchy
- `src/python/a2a_telecom_sdk/handlers.py` — `MessageContext`/`Sender` protocol shared by `wsgi.py`/`asgi.py`
- `src/python/a2a_telecom_sdk/async_agent.py:40` — `AsyncTelecomAgent` facade
- `src/python/a2a_fabric/registry.py` — `AgentRegistry` (in-memory register/discover/deregister)
- `src/python/a2a_fabric/catalog.py` — `FabricCatalog` CRUD for resources/tools/LLM locators
- `src/python/a2a_fabric/server.py` — `FabricServer` routing; `FABRIC_REGISTRY_URI`/`FABRIC_CATALOG_URI`/`FABRIC_PROFILE_URI`/`FABRIC_REPUTATION_URI`
- `src/python/a2a_fabric/broker.py` — `BrokerInterface`, `InMemoryBroker` (tested), `KafkaBroker`/`MQTTBroker` (untested stubs)
- `src/python/a2a_fabric/client.py` — legacy, superseded `FabricClient` (message builders only)
- `src/python/a2a_fabric/compat.py` — deprecated agent-api REST shim
- `src/python/pyproject.toml:44-50` — dependencies incl. `a2a-sdk[signing]>=1.0,<2.0`
- `specification/a2a_fabric.proto` — `AgentReputation`/`AgentEvaluation`/`EvaluationCase`/`AgentTrustScore` message definitions (data model only, no Python scoring logic)
- `specification/a2a_secure_passport.proto`, `a2a_trusted_flow.proto` — TrustedFlow/SecurePassport message definitions (no Python implementation found)
- `docs/trusted-layer/trusted-layer-onboarding-guide.md` — TrustedFlow sign/verify against remote `stage-api.mef.dev`
- `docs/fabric/overview.md` — Fabric-T architecture overview, extension URI table
- `docs/developer-guide.md:1-374` — primitives-level walkthrough for Task-T/Event-T/Negotiation-T
- `docs/sdk.md:1-628` — `a2a_telecom_sdk` low-code agent facade walkthrough
- `docs/specification.md:1-377` — normative A2A-T spec, marked Experimental/1.0.0-draft
- `docs/scenarios/fault-management.md:1-263` — multi-agent scenario walkthrough with JSON wire examples
- `docs/rest-api/overview.md:1-118`, `examples.md:1-192` — Fabric-T REST API reference
- `docs/bpmn-mapping/BPMN_DataModel_view.puml`, `examples/bpmn/README.md:1-144` — BPMN-to-A2A-T process mapping artifact
- `examples/fault_diagnosis_agent/agent.py:1-102` — representative minimal Task-T agent
- `examples/core_agent/agent_card.json:1-80` — AgentCard with Fabric-T profile extension metadata
- `CHANGELOG.md:1-70` — v0.8.0 maturity signal (510 passing tests) despite "Experimental" spec status
- `tests/test_fabric_reputation.py` — 3 tests, asserting URI constants only (no scoring logic)
- `tests/test_extension.py`, `test_agent_card.py`, `test_interoperability.py`, `test_3gpp_compat.py`, `test_negotiation_t.py` — behavioral contract tests pinning exact extension/AgentCard/negotiation semantics

### Internal analysis documents cross-referenced (`/home/lorenzo/git/openan-catalyst/a2a-t-sdk/`)

These three documents independently analyzed an earlier snapshot of this same repository. Per the Methodology note in §2, most of their code-level findings were already fixed by the time of this update and were deliberately not carried forward; the genuinely new material they surfaced is now covered by §4.11-§4.16 and §5.16 above.

- `CODE_ANALYSIS.md` — internal architecture walkthrough of `a2a-t-sdk-python` as of an earlier commit; largely superseded by this repo's own `CLAUDE.md` and the current source, but the DeepSeek-adapter and single-scenario descriptions helped identify what has since changed
- `CODE_COMPARISON.md` — an earlier `a2a-t-sdk-python` vs. A2A-T comparison; its wire-format, negotiation-type-naming, and LLM-coupling findings substantially overlap with (and are already covered in more depth by) §3-§5 above; its scenario/domain coverage matrix and broken-`energy_saving`-reference bug report are the source of §5.16's fact-check (the bug is fixed; the breadth gap has narrowed but not closed)
- `PROTOCOL_COMPARISON.md` — an earlier `a2a-t-sdk-python` vs. official `a2a-python` compliance analysis; its Gap-ID/effort-sizing format and per-A2A-concept structure (Agent Cards, Task Lifecycle, Message Protocol, Streaming, Push Notifications, Context Threading, Authentication, Error Handling) is the direct source of §4.11-§4.16; several of its specific findings (Python `>=3.14` requirement, unused `a2a-sdk`/`fastapi`/`uvicorn`/`sqlalchemy` dependencies, `pyproject.toml`/`uv.lock` drift, unversioned `NEGOTIATION_TEXT_KEY` on the `github.com` namespace) are stale and were excluded per the Methodology note
