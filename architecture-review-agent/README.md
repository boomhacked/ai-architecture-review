# AI Application Security Architecture Review

A repo-driven architecture review for AI-powered applications. Unlike a vendor questionnaire, it is designed to be answered from evidence in the codebase, infrastructure-as-code, and product documentation rather than from a person's recollection. It is executed by a reviewing agent with repository access, with a human architect validating the output.

**What this reviews:** the design. Whether trust boundaries exist, whether controls are enforced at the right layer, and what an attacker reaches when the model misbehaves.

**What this does not replace:** SAST, DAST, dependency scanning, or penetration testing. Those validate implementation. This validates architecture. A system can pass every scan and still be architecturally unsound, and an agent can be flawlessly implemented while holding a credential it should never have been given.

## Files

| File | Audience | Purpose |
|---|---|---|
| `AGENTS.md` | Reviewing agent | The review itself. Operating rules, discovery phase, 65 questions, blast radius exercise, verification queue, report structure. Read and act on this file directly. |
| `README.md` | Human reviewer | This file. Context, rationale, and the sign-off procedure. |
| `example-review-report.md` | Both | A completed report with fictional findings, showing the expected output format. |

## How to run it

1. Give the reviewing agent read access to the application repository, its infrastructure-as-code, and its product and design documentation.
2. Point it at `AGENTS.md`. The file is self-contained, it does not need this README to execute.
3. The agent works in five phases: discovery, universal questions, branch questions, blast radius, report. Discovery determines which question branches apply, so the agent derives the architecture from the repository rather than asking anyone what it is.
4. Validate the output using the sign-off checklist below before the report is issued.

The agent's output is a draft. It is not a review until a human has worked the checklist.

## Coverage

65 questions: 25 universal, organised by trust boundary, and 40 across six architecture branches (retrieval, agentic, external model API, self-hosted, fine-tuned, MCP). Only the branches discovery selects are answered, so a typical application lands around 49.

Questions are mapped to the OWASP GenAI/LLM Top 10 (2026) where a clean mapping exists.

## What this review does not cover

The review is scoped to AI-specific architecture. Four things sit outside it. Each needs an owner somewhere else, or it falls into the gap between this review and whatever comes next.

### Conventional application security

The review assumes a working baseline: authentication and authorization on the non-AI surface, session handling, tenant isolation in the primary datastore, and the standard injection and access-control classes. It verifies none of that. SAST, DAST, dependency scanning, and penetration testing cover it.

### Model behavior

The review can establish that model output is validated before it reaches a downstream consumer. It cannot establish whether that output is any good. Whether the assistant hallucinates, refuses legitimate requests, or states something false with total confidence is measured by running the system, not by reading the repository, and it is a different discipline from security review.

For a product about to ship, the apparatus looks roughly like this:

- **A golden set.** 100 to 300 representative queries from the real domain with known-correct answers. Score accuracy, but track *confident wrongness* separately: answers that are wrong with no hedging. That is the failure that reaches a customer and gets acted on, and an aggregate accuracy number hides it.
- **Faithfulness testing** on retrieval paths. Does the answer actually follow from the documents retrieved? Usually implemented as a second model judging whether each claim in the response is supported by the retrieved context. This catches the case where retrieval worked correctly and the model embellished on top of it, which accuracy scoring alone misses.
- **Refusal behavior in both directions.** Over-refusal is a product failure, under-refusal is a safety failure, and tuning one moves the other. Both need their own test sets, should-refuse and should-comply.
- **An adversarial suite** of injection and jailbreak cases. This is the part that overlaps with security, and it is what question U25 asks about.
- **Release gating.** The operationally critical piece. Every prompt edit, model version change, and retrieval configuration change reruns the suite. Without it, behavior drifts silently with each change and you find out from a customer.

Ownership usually sits with product or ML engineering, with security contributing the adversarial cases. The review's honest role is to ask whether this apparatus exists and gates releases, not to perform the evaluation itself.


### Adversarial validation

The review reads. It does not test. See *Validating the blast radius chain* below.

## Why the verification queue exists

Part 5 of `AGENTS.md` asks the agent to emit a short table of items for human confirmation. That list is not arbitrary. Each entry is a flaw that existing scanning and penetration testing structurally cannot detect, which means the agent's answer is the only signal available for it. Everywhere else in the review, a `PASS` is one signal among several. In this queue, it is the only one, so an unverified `PASS` here is an assumption rather than a result.

| Item | Why existing tooling misses it |
|---|---|
| **S1** Model output reaching an execution sink | Static analysis models request parameters as untrusted. It does not model a string returned from an inference call as attacker-influenceable, so `subprocess.run(llm_response)` frequently does not flag at all: the taint chain never starts. |
| **S2** Authorization sequencing in retrieval | Filtering results after the query is functionally correct code in the wrong position. No rule set flags ordering-of-enforcement, and the application behaves correctly in every tested case. |
| **S3** Credential scope on tool handlers | The handler uses its credential correctly. The finding is that the credential should never have held that authority. This is an IAM design question expressed in application code, invisible to code analysis and to any pentest that did not chain injection into tool invocation. |
| **S4** Guardrails existing only as prompt text | An instruction in a system prompt is a string literal. No tool has an opinion about whether it constitutes a control, and it will appear in design documentation as though it does. |
| **S5** Fail-open guardrails | A filter wrapped in exception handling that proceeds on error works correctly in every test and disappears silently during a dependency outage. Neither DAST nor a pentest exercises the outage path. |
| **S6** Client-supplied conversation history | Accepting a message array is legitimate API design in most frameworks, so nothing flags it. It also lets an attacker forge assistant turns and fabricated tool results. |
| **S7** Rendered-output exfiltration channels | Markdown output containing an image reference to an attacker-controlled URL causes the browser to fetch it, carrying data in the query string. The sink is a renderer rather than an injection point, and the payload originates from the model rather than the request. |
| **S8** Tenant scoping via string interpolation | A tenant identifier concatenated into a filter usually does not flag as injection, because the value is internally sourced. It becomes a cross-tenant issue only when that value proves less internal than assumed. |
| **S9** Indirect injection through an ingestion path | DAST probes the application's HTTP surface. It never sees the shared drive, connector, or inbox that feeds the index, which is where this attack originates. The application under test is not where the payload enters. |
| **S10** Tool-description poisoning from an external server | Definitions fetched from an MCP server land in the model's context as instructions. The attack requires no invocation of the tool and no traffic through the tested application surface. |

## Code truth and runtime truth

The review reads a repository. A meaningful share of real AI incidents are configuration rather than code: an inference endpoint bound to a public interface, a provider retention setting left at its default, a role carrying more grant than anyone intended, a flag flipped in production. Any question whose answer depends on such a value is marked `RUNTIME_DEPENDENT` and collected into its own section of the report.

Putting infrastructure-as-code in scope closes part of this, and the agent is instructed to read it. Declared network exposure, encryption at rest, private endpoints, whether logging is enabled, and IAM policy documents as written are all answerable from Terraform or its equivalent.

However, configurations that are defined outside the IaC files will not be covered. For example:
- Provider console settings, particularly model-provider retention and training-use toggles
- Vector database settings configured through a dashboard
- Guardrail features enabled by clicking
 

## Validating the blast radius chain

Part 4 produces an attack chain assembled from static reading. It is a hypothesis, not a demonstration, and it carries more weight than anything else in the report, since the containment classification is the first thing an executive audience reads.

Before the report is signed, attempt one chain end to end in a staging environment. If the posture came back unbounded, proving the chain converts an argument into a fact and ends any debate about whether the finding is theoretical. If it came back contained, attempting the most plausible chain and failing to complete it is the evidence for that claim. Either way it is probably the highest-value hour in the whole exercise, and it is the step that moves this from a document review to a security review.

## Reviewer validation before sign-off

The agent's output is a draft, not a review. Before the report is issued, a human should:

- **Spot-check every `PASS` on a Critical-severity question** by opening the cited code. A fabricated citation or a misread control is the one failure mode that makes this entire exercise counterproductive.
- **Work the Part 5 verification queue in full.** Those claims have no corroborating signal from any other tool in the pipeline, so an unverified `PASS` there is an assumption, not a result.
- **Confirm the component inventory** against someone who knows the system. Missing components mean missing branches, and a branch never asked is worse than a question answered wrong.
- **Demonstrate one blast radius chain** in staging rather than walking it on paper. See the section above for why this one matters more than the rest of the checklist.
- **Resolve the `RUNTIME_DEPENDENT` list** against actual deployment configuration. The repository shows code truth; these items depend on runtime truth.
- **Read the answer register for plausibility.** Plain-language answers make a wrong determination visible in a way that a status column does not. An answer that describes a mechanism nobody recognises is a signal to reopen the question.
