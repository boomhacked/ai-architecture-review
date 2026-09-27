# AI Security Review Questionnaire

A branching set of security questions for reviewing AI vendors during onboarding or renewal, or for threat-modeling and reviewing an AI system being built or deployed within your own organization.

## What's here

- [`questionnaire.md`](./questionnaire.md): the full questionnaire. Universal questions apply regardless of architecture, whether you're evaluating a vendor or reviewing an internal system; Section 9 is a classification checklist that routes you to the relevant architecture-specific branches (RAG, API-based, self-hosted/custom-trained, agentic, fine-tuned-on-your-data, open-weight). Every section is collapsible.
- Questions are tagged with their OWASP GenAI/LLM Top 10 (2026) code where applicable, e.g. `(LLM01)`, with a reference legend near the top of the file.
- 92 questions total, broken down as:
    - 44 universal questions (Sections 1-8, apply to every vendor or system)
    - 6 classification questions (Section 9, the branch-point checklist)
    - 42 architecture-specific questions across the six branches: A-RAG (8), B-API-based (7), C-self-hosted/custom-trained (9), D-agentic (8), E-fine-tuned-on-your-data (5), F-open-weight/marketplace (5)

Since most vendors or systems match more than one branch, a typical review ends up asking somewhere between 55 and 75 questions in practice, not all 92.

## How to use it

1. Work through the Universal Questions, with the vendor if this is a vendor review, or with your own engineering/platform team if you're reviewing a system built in-house.
2. Use Section 9 to identify which architecture branches apply (most match more than one), then open those sections too.
3. Score each answer as Satisfactory, Partial, or Unsatisfactory. See the Appendix at the end of `questionnaire.md` for scoring guidance.

## License

See [LICENSE](./LICENSE).
