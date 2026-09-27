# AI Architecture Review

This repository contains two complementary AI security review assets for evaluating AI systems from different angles:

- the [security-review-questionnaire](./security-review-questionnaire) package for vendor and architecture reviews based on a structured questionnaire
- the [architecture-review-agent](./architecture-review-agent) package for a repository-driven architecture review performed against code, infrastructure as code, and design documentation

Together they cover both a broad review workflow and a deeper, evidence-based architecture assessment aligned to modern AI security concerns.

## Repository contents

| Path | Purpose | Best for |
| --- | --- | --- |
| [`security-review-questionnaire/`](./security-review-questionnaire) | A branching security questionnaire covering universal and architecture-specific questions for AI vendors and internal systems | Vendor onboarding, procurement review, architecture assessments, and security triage |
| [`architecture-review-agent/`](./architecture-review-agent) | A repo-driven review process with an agent workflow, discovery questions, blast-radius analysis, and sign-off guidance | Reviewing a live or planned AI application from code and infrastructure evidence |

## Security questionnaire

The questionnaire is designed to help teams review AI vendors or internal AI systems using a structured set of questions.

Highlights from the questionnaire package:

- Universal questions apply across all architectures
- Section 9 identifies relevant architecture branches
- Questions are tagged to OWASP GenAI/LLM Top 10 (2026) where applicable
- Typical reviews usually cover 55–75 of the 92 questions, depending on the applicable architecture branches
- Results are scored as Satisfactory, Partial, or Unsatisfactory

For details, see [`security-review-questionnaire/README.md`](./security-review-questionnaire/README.md) and [`security-review-questionnaire/questionnaire.md`](./security-review-questionnaire/questionnaire.md).

## Architecture review agent

The architecture-review package is designed for repo-driven assessments of AI-powered applications. It focuses on whether trust boundaries and security controls are correctly placed in the system design, rather than simply validating implementation details.

Highlights from the architecture-review package:

- Reviews architecture from repository evidence, IaC, and product/design documentation
- Uses a structured set of questions and a blast-radius exercise
- Produces a draft report for human validation before sign-off
- Emphasizes verification of critical findings and runtime-dependent controls
- Complements SAST, DAST, dependency scanning, and penetration testing rather than replacing them

For details, see [`architecture-review-agent/README.md`](./architecture-review-agent/README.md) and [`architecture-review-agent/AGENTS.md`](./architecture-review-agent/AGENTS.md).

## How to use this repo

1. Start with the questionnaire if you need a broad risk review or an onboarding/renewal checklist.
2. Use the architecture review if you need to evaluate an actual application design from repository evidence.
3. Combine them when you want both a broad controls review and a deep design assessment.

## License

See [LICENSE](./LICENSE).
