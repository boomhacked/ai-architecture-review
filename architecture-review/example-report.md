# Example Output: AI Architecture Review Report

Illustrative only. The application, findings, file paths, and evidence below are fictional. This exists to fix the output format produced by `ai-architecture-review.md`, and can be handed to the reviewing agent as a formatting example.

---

# AI Architecture Review: Atlas Assistant

**Commit:** `a3f91c2` · **Reviewed:** 2026-09-27 · **Agent run:** 2026-09-26 · **Validated by:** M. Okonkwo

## 1. Containment posture: UNBOUNDED

An attacker who achieves prompt injection in this system can read documents belonging to other tenants, and can cause outbound email to be sent under a shared service identity with no human approval step.

**Demonstrated chain:** attacker uploads a document to a shared Confluence space → connector sync ingests it with no authorization check (R3) → vectors land in the shared index → a different tenant's query retrieves the chunk because tenant scoping is applied after retrieval (R1) → injected instruction enters that user's context → model invokes `send_email` → message delivered externally under the platform service account (T1, T4).

| Red line | Result |
|---|---|
| Cross-tenant reach | **Yes** (R1, R8) |
| Irreversible action without server-side gate | **Yes** (T4) |
| Credential reach beyond invoking user | **Yes** (T1) |

## 2. Architecture summary

Retrieval-augmented assistant over a pooled multi-tenant index, calling a third-party model API, with six registered tools including two write actions. Branches selected: **R**, **T**, **E**, **M**. Branches N/A: **H**, **F** (no self-hosted inference or tuning code present, D2).

Trust boundaries identified: client→application, application→model, model→tools, external content→prompt (two ingestion paths), application→user.

## 3. Findings

### F-01 · Critical · R1 · Tenant scoping applied after retrieval rather than in the query

**Evidence:** `services/retrieval/search.py:88-104`
```python
results = index.query(vector=embedding, top_k=25)
return [r for r in results if r.metadata["tenant_id"] == ctx.tenant_id]
```
**Mechanism:** the vector query is unscoped. All 25 nearest neighbours are read from the shared index regardless of tenant, and filtering happens in application memory afterwards.
**Impact:** any code path that consumes `index.query` without replicating the filter returns cross-tenant content. One such path exists at `services/summarize/batch.py:41`, which does not filter.
**Remediation:** tenant scope must be bound into the query as a filter parameter derived from the verified session, not applied to results. The retrieval client should not expose an unscoped query method at all.
**Boundary:** application→retrieval · **OWASP 2026:** LLM09, LLM02

### F-02 · Critical · T1 · Tool handlers execute under a shared service identity

**Evidence:** `tools/email.py:12-19`, `tools/calendar.py:15-22`
**Mechanism:** both write tools authenticate with `ATLAS_SERVICE_TOKEN`, which holds send-on-behalf-of rights across the whole workspace. The invoking user's identity is available in context but unused.
**Impact:** an injection reaching either tool acts with authority no individual user holds. Combined with F-01, content from one tenant can be emailed out through another tenant's session.
**Remediation:** tool execution must carry delegated user authority. Where the downstream API supports it, exchange the session for a scoped token per invocation; where it does not, the tool should be gated behind F-03's approval step and its scope narrowed to the invoking user's resources.
**Boundary:** model→tools · **OWASP 2026:** LLM06

### F-03 · High · T4 · No server-side approval gate on write actions

**Evidence:** `api/routes/chat.py:210-226`; UI confirmation at `web/components/ToolConfirm.tsx:33`
**Mechanism:** the confirmation dialog is client-side only. The execute endpoint accepts the tool call without an approval token and performs no state check.
**Impact:** the gate is bypassable by calling the API directly, and is absent entirely for agent-initiated calls that do not surface in the UI.
**Remediation:** approval must be a server-side state transition, with the agent unable to satisfy it on its own behalf.
**Boundary:** model→tools · **OWASP 2026:** LLM06

### F-04 · High · U20 · Full prompts and retrieved content written to shared observability platform

**Evidence:** `api/middleware/logging.py:64`
```python
logger.info("inference", extra={"prompt": full_prompt, "response": completion})
```
**Mechanism:** the complete assembled prompt, including retrieved document text, is emitted at INFO to the org-wide logging platform. That platform is readable by all of engineering and support, roughly 140 accounts; the source documents are restricted per tenant.
**Impact:** logs are a second, less-restricted copy of every document the assistant has ever retrieved. No attacker is required.
**Remediation:** exclude retrieved content and user input from log payloads, or route AI-path logs to a sink whose access control matches the source data classification. Retention should match the source system's, not the observability platform's default.
**Boundary:** cross-cutting · **OWASP 2026:** LLM02

### F-05 · High · U17 · Output guardrail fails open

**Evidence:** `services/guardrails/client.py:47-53`
**Mechanism:** the moderation call is wrapped in a bare `except Exception` that logs and returns `allowed=True`.
**Impact:** any outage, timeout, or quota exhaustion at the guardrail provider silently disables output filtering system-wide, with no alert.
**Remediation:** fail closed. A guardrail dependency failure should degrade the feature, not the enforcement.
**Boundary:** model→user · **OWASP 2026:** LLM01, LLM10

### F-06 · Medium · U7 · Client-supplied conversation history accepted

**Evidence:** `api/schemas/chat.py:18`, `api/routes/chat.py:96`
**Mechanism:** the request schema accepts a `messages` array, which is passed to the provider unmodified after the system prompt is prepended.
**Impact:** a client can forge prior assistant turns and fabricated tool results, defeating behavioural instructions in the system prompt.
**Remediation:** reconstruct history server-side from the session store; accept only the new user turn from the client.
**Boundary:** client→application · **OWASP 2026:** LLM01

## 4. Undetermined items

| ID | Question | Searched | What would resolve it |
|---|---|---|---|
| R4 | Ingestion-time content sanitization | `services/ingest/**`, `parsers/**`; found parsing and chunking, no sanitization stage | Confirmation from the ingestion owner whether sanitization happens in the connector service, which is a separate repository not in scope |
| E1 | Provider retention configured | `clients/llm.py`, `infra/`, env templates; no retention or zero-retention parameters present | Provider console configuration, or the account-level agreement |
| M3 | Credential scope passed to MCP servers | `mcp/config.json` references `${MCP_TOKEN}`; scope not determinable from repo | The token's actual grant in the identity provider |
| U3 | Per-user cost caps | Rate limiting found at `api/middleware/ratelimit.py:20` (requests only); no token or spend accounting located | Confirmation whether spend control exists at the gateway tier |

## 5. Runtime verification list

| ID | Repo default | Must verify in deployment |
|---|---|---|
| U15 | `SANITIZE_MARKDOWN=true` in `web/.env.example` | The value actually set in production. If false, F-05 compounds into an exfiltration path. |
| U24 | Feature flag `atlas.enabled` present, default on | Whether per-tenant disablement is wired to the same flag |
| U12 | Egress unrestricted in `infra/network.tf` | Whether a perimeter policy restricts outbound from the inference tier |

## 6. Coverage statement

Commit `a3f91c2`, branch `main` only. In scope: the application repository and its `infra/` directory. Out of scope: the connector service repository (separate, referenced by R4), provider console configuration, and the identity provider. Documentation consulted: `docs/architecture.md`, `docs/data-flow.md` (last updated four months prior to this commit; two components in it no longer exist in code, noted as a documentation finding).

## 7. Recommendation

**No-go in current state.** F-01 and F-02 together produce an unbounded containment posture with a demonstrated cross-tenant path. Both must close before release.

F-03 and F-05 are acceptable as fast-follows only if F-01 and F-02 close first, since their severity derives largely from what the other two make reachable. F-04 requires a decision from the data owner rather than an engineering fix, and should not block release if logging is disabled for the AI path in the interim.

## 8. Appendix A: Answer register

```
Applicable: 49 of 65   Pass: 31   Fail: 6   Partial: 4   Undetermined: 4   Runtime-dependent: 3   N/A: 16
```

| ID | Question | Answer | Status | Severity | Evidence |
|---|---|---|---|---|---|
| U1 | Is every entry point into the AI pipeline authenticated, including internal, debug, evaluation, and admin routes? | All routes that reach the model sit behind the same authentication layer, applied by default rather than added per route. The evaluation harness is not exposed outside the build environment. | PASS | | `api/routes/__init__.py:14-28` |
| U2 | Is the authenticated user's identity carried through the pipeline as a first-class value, or dropped after the entry point? | The signed-in user is carried through every stage as a context object, so later components can make decisions based on who is asking. | PASS | | `api/context.py:31` |
| U3 | Are per-user request rate, token, and cost limits enforced server-side? | There is a server-side cap on how many requests a user can make, but nothing limits how much the model may be asked to process or how much a single user can spend. | PARTIAL | Medium | `api/middleware/ratelimit.py:20` |
| U5 | Are system instructions, user input, retrieved content, and tool results distinguishable in the assembled prompt? | Each kind of content is delivered in its own labelled section rather than pasted into one block, so the model can tell instructions apart from documents. | PASS | | `prompts/assemble.py:40-58` |
| U7 | Is conversation history reconstructed server-side, or accepted from the client? | The application trusts whatever conversation history the client sends it, which means a caller can invent earlier replies that the assistant never actually gave. | FAIL | Medium | `api/schemas/chat.py:18` → F-06 |
| U13 | Is model output treated as untrusted input by every consumer? | Most places that use the model's answer check it first, but the web interface and the tool dispatcher both accept it as-is. | PARTIAL | High | `web/render.tsx:88`, `tools/dispatch.py:24` |
| U14 | Does model output reach any execution sink: shell, `eval`, SQL, file path, deserialization, or dynamic import? | Nothing the model says is ever run as a command, used to build a database query, or turned into a file path. | PASS | | no sinks located across `tools/**`, `services/**` |
| U15 | Is model output rendered as markdown or HTML, and if so, is it sanitized? | Answers are shown as formatted text, and whether that formatting is cleaned first depends on a setting that is switched on in the sample configuration but must be confirmed in production. | RUNTIME_DEPENDENT | | `web/.env.example:9` |
| U17 | Are guardrails positioned where they can actually intercept, and do they fail closed? | The content filter runs in the right place, but if the filtering service is unavailable the application lets the answer through instead of blocking it. | FAIL | High | `services/guardrails/client.py:47` → F-05 |
| U20 | What portion of prompts, responses, and retrieved content is written to logs, and who can read those logs? | The full question, the full answer, and the text of every document the assistant read are written into the company-wide logging system, which roughly 140 staff can search. The documents themselves are restricted per customer. | FAIL | High | `api/middleware/logging.py:64` → F-04 |
| R1 | Is authorization applied to the retrieval query itself, or to the results after retrieval? | The search runs across all customers' content first, and the results are narrowed to the right customer afterwards in application code. | FAIL | Critical | `services/retrieval/search.py:88` → F-01 |
| R3 | Is every ingestion path into the index authenticated and authorized? | Documents synced in by the connector are indexed without any check on who added them or whether they should be searchable. | FAIL | Critical | `services/ingest/connector.py:55` |
| R8 | How is tenant isolation enforced in the vector store specifically? | All customers share a single vector database. Separation depends on a customer label attached to each stored item, and that label is checked only after a search returns results. | FAIL | Critical | `services/retrieval/search.py:88` → F-01 |
| T1 | For each tool, what credential does it use, and is that credential scoped to the invoking user's authority? | The email and calendar tools act using one shared workspace account rather than on behalf of the person using the assistant, so they can do more than any individual user could. | FAIL | Critical | `tools/email.py:12` → F-02 |
| T4 | Do consequential or irreversible actions require human approval, and is the gate enforced server-side? | The interface asks the user to confirm before sending, but the server performs the action whenever it is asked, with no record that anyone approved it. | FAIL | High | `api/routes/chat.py:210` → F-03 |
| E4 | Do provider credentials have blast radius beyond this application? | The model provider key belongs to this application alone, carries a spending limit, and is rotated on a schedule. | PASS | | `infra/secrets.tf:22-31` |
| | *remaining rows omitted for brevity* | | | | |

**N/A, grouped:** H1 through H6 and F1 through F5 ruled out by D2, which found no self-hosted inference, no managed-platform model deployment, and no training or fine-tuning code anywhere in the repository.

## 9. Appendix B: Verification queue

| S-ID | Linked | Status | Evidence | Human verified |
|---|---|---|---|---|
| S1 | U14 | PASS | `tools/*.py` reviewed, no execution sinks located | Yes, confirmed |
| S2 | R1 | FAIL | `services/retrieval/search.py:88` | Yes |
| S3 | T1 | FAIL | `tools/email.py:12` | Yes |
| S4 | U17 | FAIL | `services/guardrails/client.py:47` | Yes |
| S6 | U7 | FAIL | `api/schemas/chat.py:18` | Yes |
| S7 | U15 | RUNTIME_DEPENDENT | `web/.env.example:9` | Pending |
| S9 | R3 | FAIL | `services/ingest/connector.py:55` | Yes |
