# AI Security Review

This questionnaire is built for a security analyst or engineer evaluating a third-party AI tool vendor during onboarding or renewal, or reviewing and threat-modeling an AI system being built or deployed within your own organization. It has two layers.

Throughout, "vendor" refers to whoever is answering these questions, a third-party AI vendor, or your own engineering or platform team if you're reviewing a system built in-house. The questions and structure work the same either way.

**Universal Questions** apply to every AI vendor or system regardless of how it's built. Ask these every time.

**Architecture-Specific Questions** branch based on how the system is actually designed. Section 9 is a classification checklist. Most vendors or systems match more than one pattern (a product might combine RAG with a third-party API model, for example), so answer Section 9 first and then open every branch it points to.

Each question can be asked directly in a written questionnaire, an RFP, a live interview, or worked through internally as a self-assessment. Where useful, a "Why this matters" note explains the risk being probed. Treat an inability to answer, a vague or evasive answer, or an answer that reveals the question has never been considered, as a risk signal in itself.

Click a section heading below to expand it.

### OWASP GenAI/LLM Top 10 (2026) reference

Where a question maps clearly to a specific 2026 OWASP LLM risk category, the code appears in parentheses at the end of the question. Not every question maps to one; governance, legal, and process questions generally don't correspond to a single named technical risk.

| Code | Category |
|---|---|
| LLM01 | Prompt Injection |
| LLM02 | Sensitive Information Disclosure |
| LLM03 | Excessive Agency |
| LLM04 | Supply Chain |
| LLM05 | Data and Model Poisoning |
| LLM06 | Unbounded Consumption |
| LLM07 | Misinformation |
| LLM08 | Hidden Context Exposure |
| LLM09 | Vector and Embedding Weaknesses |
| LLM10 | Improper Output Handling |

---

## Universal Questions

<details>
<summary><strong>1. Company, Governance, and AI Program Maturity</strong></summary>

**1.1** Describe your organization's AI governance structure. Is there a named individual or team accountable for AI security and risk, and do they have authority independent of the product/engineering roadmap?

> **Why this matters:** a vendor with no accountable owner for AI risk typically has no process for catching AI-specific issues before they ship, regardless of how good their general security program is.

**1.2** Do you maintain a written AI risk management program aligned to a recognized framework (for example, the NIST AI Risk Management Framework, or an equivalent internal framework)?

**1.3** Have you conducted, or had a third party conduct, a threat model of your AI system's architecture? Can you describe the categories of threats it identified?

> **Why this matters:** a vendor who has never threat-modeled their own system is unlikely to have deliberately addressed AI-specific attack classes like prompt injection or model extraction. They may only have inherited generic web-application security.

**1.4** Have you had a penetration test or red-team exercise performed specifically against your AI/LLM components (as opposed to only your general web application or API layer) within the last 12 months? Can you share a summary of findings and remediation status?

**1.5** Do you maintain a model card, system card, or equivalent documentation describing your AI system's intended use, known limitations, and out-of-scope use cases? Can you provide it?

**1.6** Who are your key AI-related subprocessors and subcontractors (foundation model providers, hosting/cloud providers, vector database providers, embedding providers, evaluation/monitoring tool vendors)?

> **Why this matters:** nearly every AI vendor is itself a downstream consumer of one or more other AI vendors. Your actual data exposure often runs three or four hops deep. You need the full chain, not just the immediate vendor.

**1.7** What is your process for notifying customers of a material change to your AI architecture, model provider, or subprocessor list (for example, switching foundation model providers, or beginning to use customer data for a new purpose)?

**1.8** Do you carry cyber insurance, and does the policy explicitly cover AI/ML-specific incidents (model theft, training data breach, AI-caused erroneous output/decision liability), or does it exclude them?

</details>

<details>
<summary><strong>2. Data Handling, Flow, and Privacy</strong></summary>

**2.1** Provide a data flow diagram or written description showing exactly where our data goes from the moment it is submitted to your product until final output is returned: every system, subprocessor, storage location, and third-party API call in the path.

> **Why this matters:** this is the single most important artifact in an AI vendor assessment. If a vendor cannot produce this, nothing else in the questionnaire can be evaluated with confidence.

**2.2** Is our data used, in any form, to train, fine-tune, or improve your models, including your own models and any third-party foundation models you call? If so, specify exactly which data, is there an opt-out, and is the opt-out the default or something we must actively request? (LLM02)

**2.3** Is our data used to train or fine-tune models that are shared with, or whose outputs are served to, other customers (i.e., could learning from our data influence what another customer's instance produces)? (LLM02)

> **Why this matters:** this is the core multi-tenancy question for AI specifically. A "no cross-customer data sharing" answer for storage does not automatically mean "no cross-customer influence" for a shared or fine-tuned model, so ask this explicitly and separately.

**2.4** What is your data retention policy for: (a) submitted prompts/inputs, (b) generated outputs, (c) any intermediate representations such as embeddings, (d) logs and telemetry? Provide specific retention periods for each, not a single blanket answer.

**2.5** Upon contract termination, what is your data deletion process and timeline? Does deletion include data embedded in trained/fine-tuned model weights, cached embeddings, and backups, or only primary data stores? Will you provide written certification of deletion?

> **Why this matters:** data absorbed into fine-tuned model weights or embeddings is often technically infeasible to cleanly delete without retraining. A vendor should be honest about this limitation rather than promising blanket deletion they cannot deliver.

**2.6** What data classification levels (e.g., public, internal, confidential, regulated/PII) is your AI system approved to process, and do you have technical controls (not just policy) that prevent regulated data from being submitted to components not approved for it?

**2.7** Where geographically is our data processed and stored at each stage of the data flow, including any third-party AI API calls? Does any hop in that flow cross a border relevant to our regulatory obligations (for example, GDPR data residency requirements)?

**2.8** If your product ever produces an output that includes or resembles content from another customer's data, or from a third party's copyrighted or proprietary material, what is your incident process, and do you offer contractual indemnification for that scenario? (LLM02)

**2.9** Do you conduct or commission independent privacy impact assessments or data protection impact assessments (DPIAs) for your AI processing activities?

</details>

<details>
<summary><strong>3. AI-Specific Application Security</strong></summary>

**3.1** What defenses do you have against prompt injection, covering both direct injection (an end user instructing the model to ignore its instructions) and indirect injection (malicious instructions hidden inside a document, webpage, email, or other content the model processes on our behalf)? (LLM01)

> **Why this matters:** indirect injection is the higher-risk variant for most enterprise deployments, since it does not require the attacker to have any access to your product at all, only the ability to place content somewhere your AI system will later read it.

**3.2** Describe your input validation and output filtering/guardrail approach. Do you use a layered approach (e.g., pattern-based filtering first, escalating to more expensive semantic or model-based classification), and what is your measured false-positive and false-negative rate on adversarial test sets? (LLM01, LLM10)

**3.3** Has your system been tested against known jailbreak technique categories (roleplay/persona framing, multi-turn conditioning, obfuscation/encoding, and similar)? What was the outcome, and how frequently is this testing repeated as new techniques emerge? (LLM01)

**3.4** How does your system handle a case where the model attempts to take an action, call a tool, or generate output that falls outside its intended scope? Is there a hard technical boundary, or does the system rely solely on the model's own judgment/training to stay in scope? (LLM03)

**3.5** What rate limiting, cost controls, and resource caps are in place to prevent a single user or a malicious actor from driving excessive model calls, oversized context windows, or repeated expensive operations against your system (denial-of-wallet / resource exhaustion protection)? (LLM06)

**3.6** If your product retains conversational memory or context across sessions, what prevents an attacker from poisoning that persistent memory in one session to influence behavior in a later, unrelated session? (LLM01)

**3.7** Do you log and monitor for anomalous prompts or usage patterns that might indicate an extraction, enumeration, or abuse attempt against the model itself, distinct from general application security monitoring?

</details>

<details>
<summary><strong>4. Access Control, Authentication, and Tenant Isolation</strong></summary>

**4.1** Describe your multi-tenant isolation model for AI-specific components specifically: model inference, any fine-tuned model instances, vector stores/embeddings, and conversational memory, not just your general application database. Is isolation logical (shared infrastructure with access controls) or physical (dedicated infrastructure per tenant), and can we choose?

**4.2** What authentication and authorization controls exist around administrative access to model configuration, prompts/system instructions, training or fine-tuning data, and model weights? Is multi-factor authentication enforced for these specifically, not just for general account login?

**4.3** Do you follow least-privilege principles for the AI system's own permissions: for example, if the AI can query a database, read files, or call internal APIs on our behalf, is its access scoped narrowly to only what a given task requires, or does it operate with broad standing access? (LLM03)

**4.4** How are API keys, service credentials, and secrets used by your AI pipeline (including keys to any third-party model providers) stored, rotated, and access-controlled? Are these ever exposed in logs, prompts, or error messages?

**4.5** If our organization integrates your AI product with our own internal systems (via plugins, connectors, or APIs), what mechanism prevents your AI system from taking actions in our environment beyond what we explicitly authorized? (LLM03)

</details>

<details>
<summary><strong>5. Monitoring, Logging, and Incident Response</strong></summary>

**5.1** What AI-specific events do you log and monitor (e.g., anomalous prompt patterns, unusual output volume, repeated refusal-triggering attempts, drift in output characteristics), in addition to standard infrastructure and application logs?

**5.2** What is your incident response process specifically for an AI-related incident (a successful jailbreak, a data leak through model output, a poisoned data source, or a compromised model file) versus your general security incident process? Are AI incidents triaged and escalated differently?

**5.3** What is your customer notification SLA for a confirmed AI-related security incident affecting our data or our users, and does it meet or exceed applicable regulatory notification deadlines (for example, GDPR's 72-hour requirement)?

**5.4** Do you maintain a baseline of expected model behavior/output characteristics so that a silent, unannounced change (a model update by an upstream provider, a configuration drift, or a compromise) can be detected? How often is that baseline checked?

**5.5** Have you experienced a security incident involving your AI system (model theft, data poisoning, prompt injection leading to data exposure, or similar) in the past 24 months? If so, can you describe it and the remediation taken?

</details>

<details>
<summary><strong>6. Model and Training Data Provenance (general)</strong></summary>

**6.1** Regardless of whether the model is yours or a third party's, can you provide documentation of what data the model in production was trained or fine-tuned on, including sourcing, licensing, and any known provenance gaps? (LLM04)

**6.2** Has the training/fine-tuning data undergone any process to identify and remove personal data, credentials, or other sensitive content that could later be surfaced through the model's output? (LLM02)

**6.3** What integrity or verification checks are applied to any model artifact (weights, configuration, or adapter) before it is deployed to production, to confirm it has not been tampered with or substituted? (LLM04)

</details>

<details>
<summary><strong>7. Compliance, Legal, and Contractual</strong></summary>

**7.1** What certifications or attestations do you hold that cover your AI systems specifically (SOC 2 Type II with AI/ML in scope, ISO/IEC 27001, ISO/IEC 42001 for AI management systems, or similar)? Are AI components explicitly in scope of the audit, or excluded?

**7.2** If your product or its outputs fall within scope of the EU AI Act, what risk category do you assess your system as, and what obligations have you implemented accordingly (for example, transparency obligations, human oversight, technical documentation)?

**7.3** Does your contract include specific representations and warranties about AI behavior, such as accuracy, non-infringement of third-party rights in generated output, and non-discrimination, and does it include indemnification for AI-caused harms, or are these excluded/disclaimed?

**7.4** If your AI system makes or materially influences a decision that affects a person (eligibility, pricing, employment, access, or similar), what human oversight or appeal mechanism exists, and how do you support our compliance obligations around automated decision-making (for example, GDPR Article 22)?

**7.5** Have you evaluated your system for bias or disparate impact across protected classes? Can you share the methodology and results, and how frequently this evaluation is repeated?

</details>

<details>
<summary><strong>8. Business Continuity and Vendor Viability</strong></summary>

**8.1** If your primary foundation model provider became unavailable (outage, contract termination, deprecation of the model version you rely on), what is your continuity plan, and how would that affect the behavior, quality, or availability of our service?

**8.2** What is your model versioning and rollback process? If an update to the underlying model (yours or a third party's) causes a regression in behavior or a new security issue, how quickly can you roll back, and will we be notified before or after a model version change goes live?

</details>

---

## Architecture Classification (branch point)

<details>
<summary><strong>9. Which patterns describe the vendor's system? (select every one that applies)</strong></summary>

**9.1** Does your product retrieve and inject external documents or data (yours, ours, or a third party's) into the model's context at query time before generating a response, i.e., is it retrieval-augmented (RAG)? → **Branch A**

**9.2** Does your product send our data to a third-party foundation model provider's API (e.g., a hosted LLM API you call over the network but do not host or control) as part of generating output? → **Branch B**

**9.3** Do you train, fine-tune, or host your own model, including a model built on a managed platform such as Amazon Bedrock, SageMaker, Google Vertex AI, or Azure AI Foundry, where you control the weights or a customized version of them? → **Branch C**

**9.4** Does your AI system take autonomous or semi-autonomous actions on our behalf, such as calling tools, functions, or APIs; reading or writing files; or executing multi-step tasks without a human approving each individual step? → **Branch D**

**9.5** Is any part of your model specifically fine-tuned, adapted, or personalized using our organization's data? → **Branch E**

**9.6** Do you use openly published/open-weight models (downloaded from a public model repository or marketplace) anywhere in your pipeline, either as-is or as a base for further fine-tuning? → **Branch F**

Most vendors match more than one: a product might combine RAG with a third-party API model, or an agentic layer on top of a self-hosted model. Open every branch section a "yes" points to.

</details>

---

## Architecture-Specific Branches

<details>
<summary><strong>Branch A: Retrieval-Augmented Generation (RAG) Systems</strong></summary>

*A RAG system's attack surface extends beyond the model itself into everything that feeds its retrieval pipeline: the vector database, the embedding process, and the document ingestion path. A vendor with a well-secured model but a poorly secured retrieval pipeline can still be made to leak or manipulate data purely through the documents it retrieves.*

**A.1** What vector database do you use, and is it dedicated to us or shared/multi-tenant with logical isolation (namespaces, metadata filtering) between customers? What specifically prevents a query in our tenant from retrieving another tenant's vectors? (LLM09)

**A.2** Who or what can add, modify, or delete documents in the knowledge base/vector store that your system retrieves from? Is there authentication and authorization on ingestion, or can any upstream data source write into the index unchecked? (LLM05, LLM09)

> **Why this matters:** if ingestion is unauthenticated or unvalidated, an attacker who can place a single document anywhere the system indexes from (a shared drive, a public wiki, an inbound email) can poison the knowledge base and influence every subsequent query that happens to retrieve it, without ever touching your product directly.

**A.3** What content validation or sanitization is applied to documents before they are indexed: do you scan for hidden/invisible instructions embedded in retrieved content, unusual formatting designed to manipulate the model, or other injection patterns before that content ever reaches the model's context? (LLM01, LLM09)

**A.4** If our data is used as source material in the knowledge base, is it embedded and indexed separately from other customers' data, and is the embedding model itself shared across tenants? Could our data's embeddings be retrievable, even indirectly, by another tenant's queries? (LLM09)

**A.5** What monitoring exists to detect an unusual shift in what gets retrieved for a given query pattern, a sign that the knowledge base may have been poisoned or that a malicious document is being over-retrieved across unrelated queries? (LLM05, LLM09)

**A.6** Does your ingestion pipeline automatically detect and remove content from the index when the source document is deleted, moved, or has its access permissions revoked? (LLM09)

**A.7** If we need to force immediate removal of a specific document from the index (for example, during an incident), what is that process, how quickly does it take effect, and does it also purge cached embeddings and any downstream caches? (LLM09)

**A.8** Does the retrieval pipeline enforce document-level or field-level access control at query time (so a user only retrieves content they are authorized to see), or is access control applied only at the collection/index level? (LLM02, LLM09)

</details>

<details>
<summary><strong>Branch B: API-Based Consumption of a Third-Party Foundation Model</strong></summary>

*When a vendor calls out to an external LLM API rather than hosting their own model, your data's actual exposure depends heavily on that upstream provider's own policies and controls, which the vendor may or may not have negotiated favorably on your behalf.*

**B.1** Which foundation model provider(s) and specific model version(s) do you call, and do you have a documented data processing agreement with that provider that flows down our data protection requirements? (LLM04)

**B.2** Does the foundation model provider retain, log, or use the data you send them (our prompts and any injected context) for any purpose, including abuse monitoring, model improvement, or training? Have you enabled any available zero-retention, opt-out, or enterprise-tier data protections offered by that provider? (LLM02)

**B.3** If the foundation model provider changes their model version, deprecates the version you rely on, or changes their data handling policy, how are you notified, and how do you flow that change through to us? (LLM04)

**B.4** Where is the foundation model provider's infrastructure located, and does routing our data through it cross any data residency boundary relevant to our compliance obligations?

**B.5** How are your API credentials to the foundation model provider secured, scoped, and rotated? If those credentials were compromised, what is the blast radius: could an attacker use them to access other customers' data or run up costs/abuse the model under your account?

**B.6** Do you apply any filtering, redaction, or transformation to our data before it is sent to the third-party API (for example, stripping personal data or secrets before the prompt leaves your environment)? If so, describe the mechanism and its known failure modes. (LLM02)

**B.7** If the third-party model provider suffers a security incident or data breach, what is your process for determining whether our data was affected, and what is your obligation to notify us? (LLM04)

</details>

<details>
<summary><strong>Branch C: Self-Hosted or Custom-Trained Models (including Bedrock/SageMaker/Vertex AI/Azure AI Foundry)</strong></summary>

*Hosting or fine-tuning your own model shifts substantially more of the AI supply chain and infrastructure security burden onto the vendor: model weights become a sensitive asset in their own right, and the vendor's MLOps pipeline becomes part of your attack surface.*

**C.1** What was the base/foundation model this system is built on, what is its provenance (original publisher, license, and how you obtained it), and was its integrity (checksum/signature) verified before use? (LLM04)

**C.2** Describe your model training and fine-tuning pipeline's security: who has access to modify training code or data, what change control and code review applies to it, and is the pipeline itself isolated from your general production environment? (LLM04, LLM05)

**C.3** Where and how are model weights stored, and what access controls, encryption at rest, and audit logging apply to that storage? Who inside your organization can read, download, or export the raw weights? (LLM04)

**C.4** What defenses do you have against model extraction (an attacker systematically querying your model's API to reconstruct a functionally equivalent copy), and against model inversion or membership-inference attacks aimed at recovering details of your training data?

**C.5** If hosted on a managed cloud AI platform (Bedrock, SageMaker, Vertex AI, Azure AI Foundry, or similar), what specific configuration have you applied for tenant isolation, encryption, network access controls (private endpoints/VPC configuration), and logging? Have you had this configuration independently reviewed?

**C.6** What serialization/file format are your model artifacts stored in, and does that format carry code-execution risk at load time (as legacy pickle-based formats do), or have you standardized on a safe-by-design format? What scanning or verification runs on any model artifact before it is loaded into production? (LLM04)

**C.7** What is your process for detecting a backdoored or poisoned model (one that behaves normally on typical inputs but misbehaves on an attacker-chosen trigger), given that this class of compromise will not show up in a standard file-integrity or malware scan? (LLM05)

**C.8** Do you maintain a software bill of materials or equivalent inventory covering your ML pipeline's dependencies (training frameworks, libraries, base images), and do you monitor it for known vulnerabilities the way you would any other production software? (LLM04)

**C.9** What is your model versioning discipline: can you roll back to a known-good prior model version quickly if a new version introduces a regression or vulnerability, and do you retain the artifacts needed to do so?

</details>

<details>
<summary><strong>Branch D: Agentic / Tool-Using Systems</strong></summary>

*Once a system can take actions (calling APIs, executing code, reading or writing files, invoking other tools), a successful prompt injection or jailbreak stops being purely an output-quality problem and becomes a mechanism for real-world unauthorized action. This branch is about containing that blast radius.*

**D.1** Provide a complete list of every tool, function, or external system your AI agent can invoke. For each, describe the scope of access granted (read-only vs. write, which data/systems, under what credentials). (LLM03)

**D.2** Is each tool call scoped to the minimum permission needed for that specific task, or does the agent operate with broad, standing credentials that happen to be capable of more than any single task requires? (LLM03)

**D.3** For actions with meaningful consequence (financial transactions, data deletion, sending communications on our behalf, modifying configurations), is there a mandatory human-in-the-loop approval step, or can the agent execute these autonomously end-to-end? (LLM03)

**D.4** If the agent is manipulated via prompt injection into attempting an unauthorized or malicious tool call, what technical control, independent of the model's own judgment, stops that call from actually executing? (LLM01, LLM03)

**D.5** How are the outputs of one tool call validated before being used as input to a subsequent step in a multi-step agentic task? Is there any sanitization applied to prevent a compromised or malicious tool response from redirecting the agent's subsequent behavior? (LLM10)

**D.6** Is the agent's execution environment sandboxed or isolated from your broader production infrastructure, such that a fully compromised agent session cannot pivot beyond the specific task and data it was scoped to? (LLM03)

**D.7** What logging exists at the level of individual tool invocations (not just final output), so that a security review can reconstruct exactly what actions an agent took and why, after the fact?

**D.8** Is there a cap on the number of sequential steps, tool calls, or total resource consumption an agent can take on a single task, to bound the damage of a runaway or manipulated agentic loop? (LLM06)

</details>

<details>
<summary><strong>Branch E: Models Fine-Tuned or Personalized on Our Organization's Data</strong></summary>

*Fine-tuning on customer-specific data creates a distinct risk that generic multi-tenancy controls don't fully address: the model's learned parameters themselves can absorb and later leak fragments of the data they were tuned on, across a boundary that access controls alone cannot enforce after the fact.*

**E.1** Is the model fine-tuned specifically and exclusively on our data, or is our data combined with other customers' data in a shared fine-tuning process? If shared, what prevents our data's influence from surfacing in another customer's outputs? (LLM02)

**E.2** What technical isolation exists between fine-tuned model instances: is each customer's fine-tuned model a fully separate artifact/deployment, or a shared base model with customer-specific adapters (e.g., LoRA) applied at inference time? What prevents cross-tenant adapter mixing? (LLM02)

**E.3** If we request deletion of our data, can the fine-tuned model derived from it actually be "un-learned," or does deletion require retraining/re-tuning from a clean base? What is your actual technical process and timeline for this? (LLM02)

**E.4** Have you tested the fine-tuned model for training-data memorization: the ability to extract verbatim or near-verbatim fragments of our data back out of the model through crafted queries? (LLM02)

**E.5** Who has access to the fine-tuning dataset itself (as distinct from the resulting model), and how long is that raw dataset retained after the fine-tuning process completes? (LLM02)

</details>

<details>
<summary><strong>Branch F: Open-Weight or Model-Marketplace-Sourced Models</strong></summary>

*A model downloaded from a public repository carries its own supply chain: the same category of risk as any third-party software dependency, plus AI-specific risks like backdoored weights that a conventional software composition scan will not catch.*

**F.1** Which public model repository or marketplace did you source this model from, and what verification did you perform on the specific file downloaded, such as checksum comparison against a value published by the original author, and confirmation of the correct namespace/publisher to avoid a similarly-named impersonating upload? (LLM04)

**F.2** What file format was the downloaded model in, and if it was in a format capable of executing code at load time, what process (automated scanning, manual review, or conversion to a safe format) did you apply before ever loading it? (LLM04)

**F.3** Do you pin to a specific, verified model version/commit, or do you pull the latest version from the repository automatically? Automatic pulls introduce risk that an upstream compromise or malicious update reaches your production system without review. (LLM04)

**F.4** If the model includes any bundled code (custom layers, loading scripts, or similar), was that code reviewed for the kind of hidden logic that can execute silently at load or inference time, distinct from reviewing the model weights themselves? (LLM04)

**F.5** Do you have a documented process for periodically re-verifying sourced models against newly published vulnerability or compromise disclosures for that specific model or repository? (LLM04)

</details>

---

<details>
<summary><strong>Appendix: Suggested Risk Scoring Approach</strong></summary>

For each section, score vendor responses on a simple scale: **Satisfactory** (clear, specific, verifiable answer with evidence offered), **Partial** (answered but vague, unverified, or missing a sub-part), or **Unsatisfactory** (no answer, evasive, or answer reveals the control does not exist).

Weight Section 2 (Data Handling), Section 3 (AI-Specific Application Security), and whichever branch sections apply most heavily, since these map most directly to the risk categories that are unique to AI systems rather than general vendor security. A vendor that scores well on general sections (governance, compliance paperwork) but poorly on the architecture-specific branch questions should be treated as higher risk than the paperwork alone would suggest: the branch questions are where AI-specific technical risk actually surfaces.

</details>
