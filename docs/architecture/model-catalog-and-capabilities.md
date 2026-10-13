# Model Catalog and Capabilities

## 1. Purpose

This Task 05 artifact freezes conceptual catalog, discovery, evidence, capability,
and availability semantics. It does not define Python/Pydantic schemas, cache
implementation, endpoints, discovery calls, SDK calls, CLI flags, subprocesses,
auth lifecycle, retries, timeouts, cancellation, or error envelopes.

```text
registered provider != discovered model != catalog entry != current availability
```

`ModelCatalog` is a provider-side/application service. It supplies resolved
catalog metadata to the Task 04 router; it never changes the required execution
path `CanonicalRequest -> ProviderRouter -> selected BaseProvider ->
InferenceResult`.

## 2. Scope and Classifications

**Frozen decision** is change-controlled. **Invariant** must hold in every
implementation. **Deferred decision** has an owner and trigger. **Assumption** is
not evidence. **Open question** has a safe current result. **Evidence-backed fact**
is supported only by the cited local reference. **Unverified capability** is not
advertised and cannot satisfy a required capability.

## 3. Inputs and Freeze Boundary

| Input | Accepted SHA-256 | Use here |
| --- | --- | --- |
| Task 01 protocol contract | `7f38d5fd91962d50209bc56e68e29c1c6ea44427003c84cada78e3bfd08b7523` | Manual model IDs and no listing precondition |
| Task 02 canonical contracts | `ac71a6ae853dec879f47fe6b403bb04ecf7648eb959b7b8e4c1cda56826edf4b` | Frozen `ModelInfo`, `ProviderCapabilities`, capability tri-state |
| Task 03 compatibility contract | `30e6d0e6788a4b5fda69225770f332a4119aea6dc3b6d35ac3373e2b981219fb` | Namespaced public identity and complete responses |
| Task 04 routing contract | `fbb9ff3084cdec737f8b76be7b3579291cc4be34d474c8fd07e73b492509bdff` | One binding, selected transport, capability gate |

**Frozen decision:** no Task 02 structural field changes are made. Catalog
metadata is an adjacent internal record, not a changed `ModelInfo` shape and not
a public canonical field. Any need for structural schema revision returns to the
Coordinator for impact review.

## 4. Evidence Honesty

**Evidence-backed fact:** local project references require official OpenAI SDK/
SIWC and Google SDK or `agy` surfaces, prohibit private credential/session access,
and require capability honesty.

**Unverified capability:** upstream versions are unpinned and Task 05 makes no
live SDK, CLI, discovery, auth, or inference probe. It therefore asserts neither
current model availability nor headless `agy` image support. Local references link
official sources but are project constraints, not a snapshot of current upstream
truth.

## 5. ModelCatalog Responsibilities

`ModelCatalog` shall:

1. Aggregate registered-provider baseline, configured entries, and discovery
   observations into evidence-backed catalog records.
2. Preserve namespaced `public_model_id` as the public key.
3. Retain internal owning `ProviderId`, selected transport, auth category, source,
   freshness, discovery state, availability, and capability evidence as metadata.
4. Resolve effective per-request capability and availability for Task 04.
5. Produce a conservative public `/v1/models` projection.
6. Track provenance, conflicts, invalidation, and state transitions without
   retaining credentials, tokens, sessions, or raw sensitive diagnostics.

## 6. ModelCatalog Non-Responsibilities

The catalog shall not parse public requests, select a binding, execute inference,
perform automatic fallback, own OAuth/PKCE/refresh, inspect keyrings/private auth
files, call undocumented APIs, map errors, own retries/deadlines/cancellation, or
make HTTP endpoint decisions. Task 06 owns lifecycle/error mechanics; Task 07 owns
auth lifecycle. Numeric TTLs, storage, and refresh mechanism remain deferred.

## 7. Identity and Ownership

The only public catalog key is namespaced `public_model_id`, for example
`openai:gpt-example` or `google:gemini-example`. A bare slug is never a key,
listing ID, routing identity, or merge key.

The catalog record retains the Task 04 internal `ProviderId` and transport as
non-public metadata. `ModelInfo.model` remains the frozen `ModelIdentity`; neither
the internal `ProviderId` nor SDK/CLI transport is exposed through that shape.

## 8. Relationship to Frozen Shapes

| Frozen shape | Task 05 semantic refinement | Not changed |
| --- | --- | --- |
| `ProviderCapabilities` | Baseline evidence can be provider-scoped and resolved with finer evidence | Fields and tri-state vocabulary |
| `ModelInfo` | `source` truthfully distinguishes configured/discovered/unknown; overrides remain explicit tri-state | Fields, identity, and override scope |
| `ModelCapabilityOverrides` | Per-model evidence supplies only explicit feature overrides | No availability/auth/provider metadata added |
| `ModelIdentity` | Catalog keys and returned inference identity stay public namespaced identity | No internal transport selector |

## 9. Capability Levels

| Level | Meaning | Example |
| --- | --- | --- |
| Provider baseline | Registered provider’s declared default for its selected transport/auth category | CLI images unknown |
| Model override | Explicit model-scoped override to a provider baseline | One model images unsupported |
| Runtime observation | Fresh direct result relevant to a selected model/transport/auth category | Current model discovery says unavailable |

Runtime evidence changes effective truth only within its scope. It never causes a
slug to choose a different provider or transport.

## 10. Capability Vocabulary

Each of `images`, `instructions`, `internal_streaming`, and `structured_output`
is exactly `supported`, `unsupported`, or `unknown`.

| State | Meaning | Required request outcome |
| --- | --- | --- |
| `supported` | Sufficient current evidence supports the feature in scope | May pass the capability gate |
| `unsupported` | Sufficient current evidence says it is unavailable in scope | Reject before execution |
| `unknown` | Evidence is absent, insufficient, stale/conflicted, or out of scope | Reject before execution |

**Invariant:** capability and current availability are independent dimensions.

## 11. Capability Semantics

`images` means preserving ordered canonical image parts through a supported
provider mapping. `instructions` means preserving system/developer instruction
semantics, not merely concatenating text. `internal_streaming` is upstream/private
collection capability and is distinct from public streaming, which Task 03 keeps
unsupported. `structured_output` means native provider enforcement; text JSON
prompting is not native structured output. `auth_categories` describes safe
configuration categories and proves neither authentication health nor model access.

## 12. Evidence Metadata

Each conceptual evidence item answers, without secrets: source, transport, safe
auth category, provider/model scope, source or implementation version when known,
observed_at, evidence kind, freshness, effective state, and reason.

| Rank | Evidence kind | Admissible conclusions | Limitation |
| --- | --- | --- | --- |
| 5 | Direct provider observation | Only the scoped availability/capability actually exercised or authoritatively rejected | Task 06 owns failure mapping; success does not prove unexercised features or discovered identity |
| 4 | Authenticated discovery | Scoped discovered identity, availability, and capabilities explicitly returned by the official selected-provider surface | Can be partial or stale; omitted feature metadata is `unknown`, not unsupported |
| 3 | Official versioned documentation | Explicit capability/constraint for the matching transport, auth category, model/scope, and version | General marketing/model-family text is inadmissible |
| 2 | Configured declaration | Configured identity/intent and explicit local restriction only | Never proves discovery, availability, or positive provider capability |
| 1 | Static project constraint | Architecture/security prohibition or conservative baseline only | Never proves a model exists, is available, or supports a feature |

Evidence is filtered for admissibility before ranking. An inadmissible item cannot
win a merge regardless of freshness; for example, configuration cannot establish
`available` or `supported`, and SDK documentation cannot establish CLI behavior.

## 13. Effective Capability Precedence

Effective capability is deterministic: consider matching scope first (model over
provider), then freshness, then authority/directness. This order makes fresh
scoped contradiction beat stale documentation while keeping direct evidence above
assumptions of the same freshness. Evidence from another transport/auth category
is never a candidate.

The comparator is the ordered tuple `(scope specificity, freshness,
authority rank)`, applied lexicographically after Section 12 admissibility filtering
and before the closed state tie-break below. Authority ranks are the closed integers
5 through 1 in Section 12; higher wins. For equal scope and freshness, direct
provider observation therefore wins over authenticated discovery, which wins over
matching official documentation, configured declaration, and static constraint.
If authority also ties, the state tie-break applies. Scope and freshness are never
inferred from ingestion order.

| Equal-rank evidence | Effective state |
| --- | --- |
| Only `supported` conclusions | `supported` |
| Only `unsupported` conclusions | `unsupported` |
| `supported` conflicts with `unsupported` | `unsupported` |
| A conclusive supported/unsupported item plus an `unknown` absence marker | The conclusive item; `unknown` is not contrary evidence |
| No conclusive item, stale-only evidence, or unresolved provenance conflict | `unknown` |

Fresh model-specific authoritative evidence can override provider baseline; weak
assumptions never override strong evidence. A stale conclusion first loses current
authority and contributes provenance only, so stale `supported` cannot override a
fresh `unknown` or `unsupported`. This resolution is ingestion-order independent.

## 14. Availability States

Availability is non-Boolean and separate from capability:

| State | Meaning | Router/listing consequence |
| --- | --- | --- |
| `available` | Fresh sufficient evidence allows the selected scoped model | Eligible subject to capability gate |
| `unavailable_authoritative` | Fresh authoritative evidence rejects the selected scoped model | Reject before inference; do not list |
| `unevaluated` | No authoritative current availability conclusion | Manual inference may reach selected provider after capability gate |
| `temporarily_unavailable` | Recent transient/provider health observation | Do not list; Task 06 maps request outcome |
| `auth_blocked` | Selected discovery/auth category cannot establish availability | Do not list; manual configured path is governed below |
| `configured_only` | Configuration names model but discovery has not established it | Do not list; manual inference may reach selected provider after capability gate |

## 15. Discovery States

Discovery state is also distinct from availability:

| State | Meaning |
| --- | --- |
| `never_attempted` | No discovery attempt occurred |
| `fresh_success` | Complete successful observation within policy freshness |
| `stale_success` | Last successful observation is no longer fresh |
| `partial_success` | Some reliable results exist but completeness is not established |
| `failed` | Attempt failed; this is not an empty model set |
| `auth_blocked` | Discovery cannot safely proceed under current auth state/category |

## 16. Configured and Discovered Records

A configured model creates a truthful `ModelInfo(source=configured)` candidate,
not discovered availability. Authenticated discovery creates
`ModelInfo(source=discovered)` evidence for the selected provider/transport/auth
scope. A record may contain both origins while preserving the most specific source
explanation in provenance. Discovery failure never deletes configured intent and
never turns into a discovered empty set.

## 17. Merge Key and Collision Policy

Catalog aggregation first partitions by selected internal `ProviderId` and
transport, then derives the one public `public_model_id` projection. A same slug
under `openai:` and `google:` is distinct. The same `google:<slug>` exposed by SDK
and CLI is not two public choices under the current Task 04 policy: only the active
explicit `google` binding contributes an active public projection. Inactive
transport records can exist internally for administration/evidence, but do not
blend or list.

## 18. Deterministic Record Merge

Within one selected provider/transport/auth scope, merge exact same public ID by:

1. Preserve the canonical public ID, opaque slug, and owner metadata.
2. Retain all provenance; never overwrite evidence history silently.
3. Resolve capabilities with Section 13 precedence.
4. Resolve availability with fresh authoritative `unavailable_authoritative`
   winning over stale/configured optimism; transient and auth states remain honest.
5. Prefer discovered presentation metadata only when scoped, fresh, and nonempty.
6. Treat identity inconsistency, cross-transport collision, or impossible source
   claim as an invalid record requiring exclusion from public listing.

No merging uses subscription-plan labels, display names, or bare slugs.

Availability resolution uses the same Section 12 admissibility filter and Section
13 `(scope specificity, freshness, authority rank)` comparator under this
total restrictive order for equal-rank conflicts:

| Rank | Effective availability | Router/listing consequence |
| --- | --- | --- |
| 1 | `unavailable_authoritative` | Reject inference; do not list |
| 2 | `temporarily_unavailable` | Current request fails through Task 06; do not list |
| 3 | `auth_blocked` | Do not list; apply sanitized request-auth readiness below |
| 4 | `available` | May infer and list only with separate fresh discovered identity |
| 5 | `configured_only` | Do not list; selected manual path may validate |
| 6 | `unevaluated` | Do not list; selected manual path may validate |

Fresh higher-scope evidence outranks stale evidence before this tie-break applies.
Equal-rank contradictory evidence resolves to the lower-numbered restrictive
state. Discovery state never outranks availability directly; it determines
whether identity/availability evidence is current enough to participate.

## 19. Transport-Scoped Evidence

Evidence is transport-scoped by default. SDK evidence does not prove CLI behavior,
and CLI evidence does not prove SDK behavior. The same Google slug uses only the
Task 04 active binding. In particular, SDK image evidence never proves CLI image
capability. CLI headless images are unverified here, remain `unknown`, are not
advertised as image-capable, and image requests reject before a subprocess call.

## 20. Authentication Scope

OpenAI evidence is model and auth-path scoped; no all-model or plan-label
assumption is permitted. Google SDK categories are API key, Vertex, and ADC;
their accessible models/capabilities are not assumed identical. Google CLI owns
browser/session/keyring authentication. The catalog neither reads private session
data nor reuses it in the SDK. `auth_blocked` means insufficient discovery evidence,
not an empty catalog, an authoritative model-not-found result, or a safe fallback.
A discovery-auth block does not itself prove whether request-time authentication
is ready; only the selected provider's sanitized auth/applicability boundary may
establish that, with lifecycle/error handling owned by Tasks 07 and 06.
The conceptual readiness signal is closed: `ready`, `not_ready`, or `unknown`.
It contains no token, account identity, session detail, expiry, or keyring data.
`ready` permits normal selected-provider dispatch after other gates; `not_ready`
rejects before inference; `unknown` permits only that selected provider to perform
its supported request-time auth/applicability validation. None permits fallback.

## 21. OpenAI-Specific Constraints

OpenAI internal streaming may be supported while public streaming remains absent.
For SIWC, the selected provider must retain `store=False`, `stream=True`, terminal
completion, and parameter filtering rules. Native structured output is distinct
from text JSON prompting, and preserving a literal system instruction requires a
supported instruction mapping. These are provider/model/auth-path claims only when
matching evidence exists; no global OpenAI model assertion is made.

## 22. Discovery, Freshness, and Invalidation

Discovery is provider-local behind the common abstraction; catalog orchestration
may request it but endpoints and compatibility do not. Refresh/invalidation
triggers include application composition/configuration change, safe auth-state
change, evidence expiry, explicit administrative refresh, authoritative model
rejection, and fresh request-time authoritative truth. Request-time truth may
update or invalidate catalog evidence; it does not introduce a fallback. TTL
numbers, scheduling, persistence, and concurrency are deferred.

## 23. Request-Time Revalidation

Task 04 routing uses catalog truth when fresh authoritative availability exists.
An authoritatively unavailable model rejects before inference. A configured-only or
unevaluated model may be manually inferred through the one selected provider after
the conservative capability gate; discovery/listing is never a prerequisite. The
provider may revalidate applicability at request time. Task 06 owns resulting
error classification and lifecycle, with no circular dependency.

| Request-time observation | Request outcome | Catalog transition | Retained evidence | Invalidated evidence | Listing effect | Fallback |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Terminal successful inference | Success | Set scoped availability `available`; record only capabilities actually exercised | Identity, provenance, and non-conflicting capability evidence | Conflicting current outage/auth/unavailable availability at the same scope | Recompute; list only if fresh discovered identity also exists | None |
| Authoritative model-unavailable result | Failure | Set `unavailable_authoritative` | Identity, capability evidence, and provenance history | Conflicting current availability | Withdraw | None |
| Transient provider/service outage | Failure | Set `temporarily_unavailable` | Identity, capability evidence, and provenance history | Current `available` conclusion | Withdraw | None |
| Sanitized authentication failure | Failure | Set `auth_blocked` | Identity, non-auth capability evidence, and provenance history | Current availability/auth-readiness conclusion | Withdraw | None |
| Authoritative capability rejection | Failure | Set matching scoped capability `unsupported` | Identity, availability, unrelated capabilities, and provenance | Conflicting current conclusion for that capability | Identity listing is unchanged if availability/identity rules pass; feature is not advertised | None |
| Non-authoritative or malformed failure | Failure | No factual catalog transition | All factual catalog evidence | None | Unchanged | None |

Every row targets only the selected provider/transport/auth/model scope. Repeating
the same observation produces the same state.

## 24. Public Listing Projection

`GET /v1/models`, should Task 01’s public surface expose it, projects catalog
records; it is not the catalog itself and is distinct from inference eligibility.
List only entries with sufficient current selected-scope evidence: fresh discovered
identity **and** fresh `available` evidence, plus no authoritative/transient/auth block,
and no required advertised
feature claimed without evidence. Do not list `unevaluated`, `configured_only`,
`auth_blocked`, `temporarily_unavailable`, failed/partial-discovery-only, or stale
records unless a future public contract adds an honest status representation.

Manual Comic Translate inference never requires a prior listing/discovery call.

## 25. Public Listing Matrix

| Evidence/availability | List? | Infer? |
| --- | --- | --- |
| Fresh discovered, available, sufficient feature evidence | Yes | Yes, after request capability gate |
| Configured-only | No | Yes through selected provider after gate |
| Unevaluated | No | Yes through selected provider after gate |
| Stale discovery | No | Same as unevaluated unless fresh authoritative availability exists |
| Partial discovery | No | Only selected manual path may validate applicability |
| Failed discovery | No | Configured/manual selected path remains unevaluated and may validate applicability |
| Auth-blocked | No | `ready` dispatches; `not_ready` rejects; `unknown` permits only selected-provider auth/applicability validation; no fallback |
| Temporarily unavailable | No | Task 06 outcome; no fallback |
| Authoritatively unavailable | No | No |

## 26. Provenance Matrix (Synthetic)

| public_model_id | ProviderId/transport | auth category | source/evidence kind | source version | discovery state | availability | images | observed_at/freshness | effective reason | public list |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `openai:gpt-sample` | `openai-siwc` / Responses | SIWC | authenticated model discovery + model evidence | synthetic `v1` | fresh_success | available | supported | synthetic fresh | Scoped discovery and feature evidence agree | yes |
| `google:gemini-sample` | `google-sdk` / SDK | API key | configured baseline only | unknown | never_attempted | configured_only | unknown | synthetic n/a | Configuration is not capability evidence | no |
| `google:gemini-sample` | `google-cli` / agy | CLI session | unverified headless image claim | unknown | auth_blocked | auth_blocked | unknown | synthetic n/a | No scoped discovery or image evidence | no |
| `openai:retired-sample` | `openai-api` / Responses | API key | authoritative provider rejection | synthetic `v1` | fresh_success | unavailable_authoritative | unknown | synthetic fresh | Fresh model-scoped rejection | no |
| `google:temp-sample` | active selected transport | selected category | transient observation | synthetic `v1` | stale_success | temporarily_unavailable | supported | synthetic fresh runtime | Availability outage is independent of capability | no |

Synthetic values illustrate semantics only; none claims a real model exists.

## 27. Catalog Scenarios

| ID | Boundary | Outcome | Future assertion |
| --- | --- | --- | --- |
| MCV-001 | Same slug under `openai:` and `google:` | Two distinct public IDs | No cross-namespace merge |
| MCV-002 | SDK and CLI expose same Google slug; SDK binding active | SDK record only is active projection | CLI has no effect on list/gate |
| MCV-003 | Same, CLI binding active | CLI record only is active projection | SDK has no effect on list/gate |
| MCV-004 | CLI image evidence absent | `images=unknown`; not image-advertised | Image request has no subprocess call |
| MCV-005 | No auth for discovery | `auth_blocked`, not empty set | No fabricated unavailable models |
| MCV-006 | Provider/service outage is explicitly classified transient | `temporarily_unavailable`, not discovery failure | Listing withdraws entry; request fails with no fallback |
| MCV-007 | Configured undiscovered model | `configured_only`, non-listed | Selected manual inference may run after gate |
| MCV-008 | Fresh model override conflicts with provider default | Fresh scoped restrictive/model evidence wins | Effective state is deterministic |
| MCV-009 | Fresh authoritative unavailable model | Reject before provider inference | No alternate provider call |
| MCV-010 | Stale supported versus fresh unknown | Fresh unknown wins | Required feature rejects |
| MCV-026 | `openai:model-a` discovered and currently available | Eligible for listing and request capability gate | Public ID and scoped evidence are preserved |
| MCV-027 | Google SDK model has fresh versioned image evidence | `images=supported` only for matching SDK/auth/model scope | CLI record for same slug remains unchanged |
| MCV-036 | Discovery attempt fails before yielding reliable records | Discovery=`failed`; configured model remains unevaluated | Not listed; selected manual applicability validation remains possible |
| MCV-037 | Partial discovery includes reliable evidence for requested model | Resolve that model from its scoped evidence | Unobserved configured models remain unevaluated, not unavailable |
| MCV-038 | Equal-rank supported and unsupported capability evidence conflict | `unsupported` wins | Merge result is ingestion-order independent |
| MCV-039 | Equal-rank unknown marker accompanies supported evidence | Supported conclusion remains effective | Unknown absence marker is not contrary evidence |
| MCV-040 | Available conflicts equally with temporary outage | `temporarily_unavailable` wins | No listing or inference fallback |
| MCV-041 | Fresh availability exists but discovered identity is stale | Not publicly listable | Manual selected-provider inference policy remains separate |
| MCV-042 | Request-time success for configured-only model | Availability becomes available; identity source remains configured | Not listed until fresh discovery identity exists |
| MCV-043 | Request-time authoritative model rejection after prior availability | Availability becomes unavailable_authoritative | Prior availability invalidated; listing withdrawn |
| MCV-044 | Auth-blocked discovery with readiness ready/not_ready/unknown | Exact three-way dispatch policy from Section 25 | No credential detail or alternate transport is consulted |
| MCV-046 | SDK and CLI observations use separate evidence stores | Same slug produces isolated scoped records | Neither store read/write changes the other |
| MCV-047 | Equally scoped/fresh documentation says supported while direct observation says unsupported | Rank 5 direct observation wins over rank 3 documentation | Effective capability is unsupported regardless of ingestion order |

## 28. Negative Cases

| ID | Architectural rejection | Future assertion |
| --- | --- | --- |
| MCV-011 | Registration implies discovery | Registered provider alone never lists a model |
| MCV-012 | Discovery failure becomes empty catalog | Failed state retains failure/provenance |
| MCV-013 | Bare slug catalog key | Bare slug cannot be stored/listed/routed |
| MCV-014 | Cross-namespace slug merge | `openai:x` and `google:x` stay separate |
| MCV-015 | SDK capability proves CLI capability | CLI image remains unknown |
| MCV-016 | CLI capability proves SDK capability | SDK evidence remains independently resolved |
| MCV-017 | Plan labels decide availability | Plus/Pro/Ultra have no capability/availability role |
| MCV-018 | Unknown treated as supported | Required capability rejects pre-execution |
| MCV-019 | Capability equals availability | Available model with unknown images still rejects images |
| MCV-020 | Listing eligibility equals inference eligibility | Configured-only manual request can reach selected provider |
| MCV-021 | Discovery is an inference prerequisite | No required prior `/v1/models` call |
| MCV-022 | Automatic SDK/CLI fallback | Alternate transport has zero calls |
| MCV-023 | Cross-auth-category evidence blending | API key evidence does not prove Vertex/ADC/SIWC scope |
| MCV-024 | Stale supported overrides fresh unknown | Fresh unknown rejects required feature |
| MCV-025 | Authoritative unavailable reaches provider | No infer call |
| MCV-028 | Stale evidence is labeled or consumed as fresh | Freshness remains explicit and stale records are not publicly listed |
| MCV-029 | Discovery auth failure becomes model-not-found | State remains `auth_blocked`; no authoritative absence is fabricated |
| MCV-030 | Configured-only model is marked discovered | Source stays configured until scoped discovery evidence exists |
| MCV-031 | Temporary unavailability becomes permanent unsupported capability | Availability and capability states remain independent |
| MCV-032 | Textual JSON prompting proves native structured output | `structured_output` remains unknown/unsupported absent native evidence |
| MCV-033 | Internal provider streaming proves public streaming | Public streaming remains unsupported under Task 03 |
| MCV-034 | CLI keyring/session is inspected to discover models | Discovery is rejected; provider exposes only sanitized supported state |
| MCV-035 | Private undocumented Google backend or fabricated capability satisfies catalog/request | Architecture rejects the evidence and required capability remains unknown |
| MCV-045 | Evidence ingestion order changes merge result | Same scoped evidence set must resolve identically in every order |

## 29. MCI Invariants Registry

| ID | Invariant |
| --- | --- |
| MCI-001 | Registered provider, discovered model, catalog entry, and availability are distinct facts. |
| MCI-002 | Every public catalog key is a namespaced `public_model_id`, never a bare slug. |
| MCI-003 | Catalog internal owner metadata never changes or publicly exposes frozen canonical shape. |
| MCI-004 | Capability is tri-state; unknown is not supported. |
| MCI-005 | Capability and availability are independent. |
| MCI-006 | Required unknown or unsupported capability rejects before execution. |
| MCI-007 | Availability is non-Boolean and authoritative unavailable rejects before inference. |
| MCI-008 | Discovery failure is not an empty set. |
| MCI-009 | Configured-only/unevaluated records are not public-list evidence. |
| MCI-010 | Listing eligibility never becomes an inference prerequisite. |
| MCI-011 | One Task 04 selected binding controls active Google transport. |
| MCI-012 | Transport/auth-scoped evidence is never blended. |
| MCI-013 | SDK image evidence never proves CLI image capability. |
| MCI-014 | No catalog rule infers truth from subscription-plan labels. |
| MCI-015 | No catalog outcome enables fallback or session/credential sharing. |
| MCI-016 | Fresh authoritative model evidence can refine baseline evidence. |
| MCI-017 | Weak/stale support cannot override fresh unknown/unsupported evidence. |
| MCI-018 | Provenance never stores or emits secrets/private session data. |
| MCI-019 | Request-time authoritative truth may invalidate catalog truth. |
| MCI-020 | Compatibility and endpoints remain provider-agnostic/catalog-orchestration-free. |
| MCI-021 | Internal streaming and public streaming are distinct claims. |
| MCI-022 | Native structured output and text JSON prompting are distinct claims. |
| MCI-023 | Evidence conflicts resolve deterministically and restrictively. |
| MCI-024 | No catalog success asserts a current upstream model without scoped evidence. |
| MCI-025 | Capability and availability merge results are deterministic and ingestion-order independent. |
| MCI-026 | Fresh availability alone never makes stale or configured identity publicly listable. |
| MCI-027 | Discovery failure preserves configured manual inference semantics and never fabricates absence. |
| MCI-028 | Evidence kinds have closed admissibility and authority ranks; inadmissible or weaker evidence cannot override stronger scoped truth. |

## 30. MCD Decisions Registry

| ID | Decision | Rationale | Prerequisite/evidence | Downstream impact | Revisit trigger |
| --- | --- | --- | --- | --- | --- |
| MCD-001 | Use public namespaced ID as catalog key | Preserves public identity | CDI-002; PRI-003 | List/router correlation | Public identity revision |
| MCD-002 | Keep ProviderId/transport internal metadata | Avoids consumer auth/transport coupling | PRD-003 | No Task 02 schema edit | Approved schema impact review |
| MCD-003 | Retain tri-state capability | Evidence honesty | CDI-016; PRI-007 | Conservative gate | Canonical revision |
| MCD-004 | Model/provider/runtime evidence levels | Models differ from baselines | Task 02 overrides | Deterministic resolution | Evidence model change |
| MCD-005 | Non-Boolean availability/discovery state machines | Failure and unknown differ | PRI-016/017 | Honest list/inference policy | Lifecycle contract conflict |
| MCD-006 | Transport-scoped Google evidence | SDK/CLI have distinct surfaces/auth | PRI-009/010 | No blended evidence | Official shared contract |
| MCD-007 | Active Task 04 binding determines Google public projection | Duplicate slug is not selector | PRD-002/006 | Coexistence without ambiguity | Approved public selector |
| MCD-008 | Configured-only may manually infer after gate | Preserve no-discovery precondition | PRI-017; CT-REQ-005 | Listing differs from inference | Safer stricter policy proven |
| MCD-009 | Listing requires sufficient current evidence | Prevent stale/unknown advertising | Task 01 optional listing | Conservative `/v1/models` | Public status contract |
| MCD-010 | Restrictive evidence precedence | Never overclaim feature support | CDI-010/016 | Gate resolution | Stronger evidence standard |
| MCD-011 | Request-time truth invalidates evidence | Catalog must not outlive authoritative truth | PRI-017 | Task 06 maps error only | Cache/lifecycle design |
| MCD-012 | Subscription labels are excluded | Labels do not establish model access | MVP 4.8 | Config/UI cannot infer truth | Explicit supported evidence |
| MCD-013 | Capability ties use the closed Section 13 rule | Future tests must observe one state | CDI-010/016 | Deterministic gate inputs | Capability vocabulary revision |
| MCD-014 | Availability ties use the closed Section 18 restrictive order | Availability cannot depend on ingestion order | PRI-016/017 | Deterministic listing/routing inputs | Availability vocabulary revision |
| MCD-015 | Public listing requires fresh discovered identity and fresh availability independently | Runtime success does not prove discoverability | CT-REQ-019; PRD-010 | Manual inference remains decoupled | Public listing contract adds statuses |
| MCD-016 | Request-time observations use the Section 23 transition matrix | Catalog updates must be factual and reproducible | PRD-010 | Task 06 supplies classified observations | Lifecycle evidence model changes |
| MCD-017 | Evidence authority uses the closed Section 12 rank after admissibility filtering | Cross-kind contradictions require one result | Provenance requirements; MCI-023 | Capability and availability merges share one comparator | New authoritative evidence kind approved |

## 31. MCD Deferrals Registry

| ID | Owner | Reason | Revisit trigger |
| --- | --- | --- | --- |
| MCD-D01 | Implementation/configuration | Cache TTL values, persistence, refresh scheduling | Runtime design authorization |
| MCD-D02 | Google specialist + implementation | Exact SDK discovery calls and `agy` flags/output evidence | Versioned official evidence or controlled test |
| MCD-D03 | Task 06 | Error classes, status mapping, retry, timeout, cancellation, lifecycle | Task 06 acceptance |
| MCD-D04 | Task 07 | OAuth/auth status lifecycle and safe auth-state signal | Task 07 design |
| MCD-D05 | Future contract review | Public catalog status fields/transport selection | Consumer-compatible proposal |
| MCD-D06 | Implementation/testing | Exact evidence storage/query model and concurrency | Implementation authorization |

## 32. Assumptions and Open Questions

| Classification | ID | Statement | Safe outcome |
| --- | --- | --- | --- |
| Assumption | MCA-001 | Configuration can identify a safe auth category without a secret | Metadata remains descriptive only |
| Assumption | MCA-002 | A provider can expose sanitized discovery evidence | Otherwise state is unknown/auth-blocked |
| Open question | MCQ-001 | Which exact discovery results are authoritative per upstream/auth mode? | Do not advertise absent scoped proof |
| Open question | MCQ-002 | Can `agy` headless accept images for a versioned CLI? | Images remain unknown and reject |
| Open question | MCQ-003 | Which models overlap across Google SDK auth categories? | Never assume overlap |
| Open question | MCQ-004 | Does a future public model-list contract expose state detail? | Current listing remains conservative |

## 33. Traceability to Task 02

All 18 canonical invariants are preserved: CDI-001 provider canonical boundary;
CDI-002 namespaced identity; CDI-003 pre-invocation rejection; CDI-004/005 ordered
content and image preservation; CDI-006 no text mutation; CDI-007/008 no auth or
provider objects; CDI-009 valid image normalization; CDI-010 explicit capability
failure; CDI-011 terminal results; CDI-012/013 truthful usage/reason; CDI-014
provider mapping after selection; CDI-015 provider-neutral compatibility; CDI-016
unknown is not support; CDI-017 internal/public streaming separation; CDI-018 safe
opaque correlation. Count: 18/18.

## 34. Traceability to Task 04

All provider invariants are preserved: PRI-001 canonical result boundary,
PRI-002 normalized identity, PRI-003 explicit namespace binding, PRI-004 slug does
not select transport/auth, PRI-005 pre-invocation routing failures, PRI-006
registration is not discovery, PRI-007 unknown/unsupported gate, PRI-008 no
fallback, PRI-009 SDK/CLI separation, PRI-010 no Google switch, PRI-011 OpenAI
auth selection isolation, PRI-012 provider isolation, PRI-013 preserved result
identity, PRI-014 explicit dependencies, PRI-015 no Codex, PRI-016 catalog owns
availability, and PRI-017 unavailable versus unevaluated behavior. Count: 17/17.

All Task 04 decisions are consumed: PRD-001 registry enumeration only,
PRD-002 one binding, PRD-003 public/internal identity separation, PRD-004/005
capability gate and unknown rejection, PRD-006 Google separation, PRD-007 no
fallback, PRD-008 availability outside registration, PRD-009 Codex exclusion, and
PRD-010 authoritative unavailable versus unevaluated dispatch. Count: 10/10.

## 35. Validation Inventory and Review State

`MCV-001` through `MCV-047` define 23 catalog scenarios and 24 negative cases.
They are future assertions, not executed tests. Optional Implementation & Testing
review should confirm testability. Required future coverage includes effective
capability precedence, state transitions, public projection, transport isolation,
manual inference without listing, invalidation, and no-fallback spies.

**Review State:** primary design received a required fresh read-only Google /
Antigravity `APPROVE` with no blocking or important findings. The reviewer found
headless `agy` image input officially unverified (`unknown`), confirmed this
document keeps CLI `images=unknown`, found no SDK-to-CLI or CLI-to-SDK evidence
leakage, and found no Plus/Pro/Ultra availability assumption. The optional
implementation handoff is to test image rejection before any CLI subprocess and
isolate SDK/CLI evidence stores when MCD-D02 is implemented. No live SDK/CLI call,
credential/keyring access, or network probe was performed; upstream versions
remain unpinned.

A read-only Implementation & Testing review first returned `APPROVE WITH REQUIRED
CHANGES` for deterministic capability/availability ties, discovery outcomes,
listing evidence, auth readiness, and request-time transitions. Sections 13, 18,
20, 23-25, and MCV-036 through MCV-046 resolved those findings. A fresh final
verification returned `APPROVE` with no blocking, important, or optional findings;
it confirmed the closed comparator, complete transition matrix, separate SDK/CLI
evidence stores, and contiguous validation inventory. No executable test was run
or claimed.

A fresh independent Architecture & Security review initially returned `REJECT`
for one merge-determinism blocker: authority/directness lacked a closed rank.
Sections 12, 13, and 18 plus MCV-047, MCI-028, and MCD-017 added admissibility
filtering and the total five-rank authority order. A new independent verification
returned `APPROVE` with no blocking, important, or optional findings and confirmed
Task 02/04 compatibility, security, Google isolation, Task 06 independence, and
registry continuity.

Because that correction formalized Google evidence admissibility, a fresh focused
Google / Antigravity verification also returned `APPROVE` with no blocking or
important findings. It confirmed exact transport/auth/model/version scope,
separate SDK/CLI evidence, CLI images remaining `unknown`, no subscription-tier
truth, and no credential/private-API access. Its optional implementation assertion
is that successful CLI text inference must never promote unexercised image support.

Coordinator QA passed, and the user explicitly approved Task 05 on 2026-10-13.
Task 05 is completed and this artifact is ready for the approved commit.

## 36. Freeze and Handoff

**Frozen conceptual scope:** catalog identity/metadata boundary, evidence and
precedence, tri-state capabilities, availability/discovery states, configured versus
discovered policy, conservative listing, selected-transport isolation, and
request-time revalidation semantics. **Not frozen:** runtime implementation,
numeric freshness, provider calls, errors/lifecycle (Task 06), and auth lifecycle
(Task 07).

Handoff to Task 07: consume `auth_blocked` only as sanitized catalog state; do not
alter auth ownership. Handoff to Task 06: map catalog/provider outcomes without
changing state semantics or allowing retries/fallback. No Task 06 dependency is
introduced. This document changes no prior artifact and is accepted for Wave
integration.