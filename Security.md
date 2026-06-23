# Security Policy

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| 0.9.x   | :white_check_mark: |
| < 0.9   | :x:                |

Only the current beta series (v0.9.x) receives security updates.

## Reporting a Vulnerability

ACODE processes visual media and generates leads that can have significant reputational and legal implications. The integrity of the pipeline, the audit trail, and the human adjudication layer are therefore security-critical.

**Please do not report security vulnerabilities through public GitHub issues.**

Instead, send a detailed report to:

**security@sufficienttoshow.org**

Include the following information when possible:

- Description of the vulnerability and potential impact
- Steps to reproduce (including any affected model versions, thresholds, or data flows)
- Whether the issue affects the inference engine, vector database, audit logging, or human-review workflow
- Any proof-of-concept or example inputs/outputs

You should receive an acknowledgment within 72 hours. We aim to provide a more detailed response within 7 business days, including an assessment of severity and proposed remediation timeline.

## Scope

We consider the following classes of issues to be in scope:

- Bypass or weakening of the mandatory human-in-the-loop adjudication requirement
- Unauthorized access to or exfiltration from the `/flagged_for_review` (append-only audit) partition
- Poisoning or manipulation of the Group_A or Group_B reference embeddings that could systematically degrade matching integrity
- Failures in provenance logging or cryptographic hashing of source imagery
- Remote code execution or container escape paths in the inference or scraper workers
- Model extraction or membership inference attacks that could reveal reference identities or thresholds
- Any issue that could allow automated dissemination of co-occurrence results without human review

Out of scope:

- General complaints about facial recognition technology or its use in public accountability work
- Issues that require physical access to the deployment environment
- Social engineering of maintainers

## Coordinated Disclosure

We follow a coordinated disclosure model. We request that reporters allow a reasonable period for remediation before any public discussion. In return, we commit to:

- Prompt acknowledgment and regular status updates
- Credit to the reporter in release notes (unless anonymity is requested)
- Transparent communication if a vulnerability affects the calibration status or operational readiness of the current beta deployment

Because the current production similarity workload remains gated on pending Chapter 119 production of native official portraits, certain classes of calibration-related issues may have limited immediate exploitability. We will still treat them seriously and document their impact on the provisional embeddings.

## Operational Security Notes

- All candidate matches are written to a restricted, append-only audit partition with source file hashing.
- No automated publication or external notification paths exist.
- The human adjudication workflow is non-bypassable at the infrastructure level.
- Reference image volumes are mounted read-only in the inference container.
- Container images are built from pinned base images where possible and scanned as part of the CI pipeline.

We take the security of the audit trail and the separation between automated detection and human judgment as core design constraints. Contributions or reports that would erode these boundaries will be declined.

## Questions

For non-security questions about responsible use, data handling, or the ethical framework, please open a GitHub Discussion or contact the maintainers through the channels listed in `CONTRIBUTING.md`.
