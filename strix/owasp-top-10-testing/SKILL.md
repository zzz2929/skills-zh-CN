---
name: owasp-top-10-testing
description: 使用Strix对应用程序进行OWASP Top 10测试，Strix是一种自主的人工智能代理，尝试对当前OWASP Top 10:2025的每一类进行真正的攻击（访问控制中断，包括SSRF、安全配置错误、软件供应链故障、加密故障、注入、不安全设计、身份验证故障、完整性故障、日志记录和警报故障、异常情况处理不当），并仅报告他们能够实际证明的内容，并通过概念证明映射回该类别。还包括OWASP API安全性前十（2023）。当用户要求进行OWASP Top 10评估、OWASP合规性测试或映射到OWASP类别的安全审查时使用。
license: Apache-2.0
metadata:
  author: usestrix
  homepage: https://docs.strix.ai
---
# Test against the OWASP Top 10

The OWASP Top 10 is a taxonomy of risk categories, not a test suite — "OWASP Top 10 testing" means exercising each category against the real application and reporting what's actually exploitable. Strix's agents do the exploitation; this skill covers running it category-by-category and reporting coverage honestly.

**Use the current edition: [OWASP Top 10:2025](https://owasp.org/Top10/)** (8th installment, superseding 2021). Ask the user before targeting an older edition — some compliance checklists still reference 2021, and a report labelled with the wrong edition is misleading. Key differences from 2021: **SSRF is folded into A01**, **A03 Software Supply Chain Failures** expands the old "Vulnerable and Outdated Components", and **A10 Mishandling of Exceptional Conditions** is new; A02 Security Misconfiguration moved 5→2.

Install, LLM setup, and the managed-cloud alternative: **penetration-testing-with-strix**. For a run with no Docker and no LLM key, the same binary drives the managed platform: `strix cloud login`, then `strix cloud scans start ...` (details in **managed-pentesting-with-strix**).

## What is and is not testable by an agent

Be straight with the user about this — claiming a clean sweep of all ten is misleading.

| Category (2025)                           | Coverage                                                                                                                                                                                                                                                                                        |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A01 Broken Access Control (incl. SSRF)    | **Strong** — cross-user/tenant access, privilege escalation, IDOR, and SSRF (including blind, via out-of-band callbacks) are all exploit-validated. Needs two accounts plus a privileged one to prove the authorization half.                                                            |
| A02 Security Misconfiguration             | **Strong** — debug endpoints, verbose errors, permissive CORS, missing hardening, default credentials, exposed admin surfaces.                                                                                                                                                           |
| A03 Software Supply Chain Failures        | **Partial** — version fingerprinting, and vulnerable/outdated dependency review when source is supplied. Build-system and distribution-infrastructure compromise (the broader half of this category) is out of scope for a runtime scan — pair with SCA plus build-provenance controls. |
| A04 Cryptographic Failures                | **Partial** — transport config, unencrypted data in transit, secrets and tokens leaked in responses. At-rest crypto and key management need source or infra review.                                                                                                                      |
| A05 Injection                             | **Strong** — SQL/NoSQL/command/template injection and XSS, exploit-validated.                                                                                                                                                                                                            |
| A06 Insecure Design                       | **Partial** — business-logic abuse (price/quantity tampering, workflow skipping, race conditions) is found where reachable; design intent still needs human review and threat modelling.                                                                                                 |
| A07 Authentication Failures               | **Strong** — auth bypass, weak session/token handling, password-reset and MFA flaws.                                                                                                                                                                                                     |
| A08 Software or Data Integrity Failures   | **Partial** — insecure deserialization and unsigned-update paths where reachable; CI/CD trust boundaries are not runtime-testable.                                                                                                                                                       |
| A09 Security Logging & Alerting Failures  | **Not testable from outside** — requires reviewing the logging and alerting pipeline. State this rather than reporting it as passed.                                                                                                                                                     |
| A10 Mishandling of Exceptional Conditions | **Partial** — agents actively probe error handling and fail-open behavior (malformed input, forced errors, race and timeout conditions) and report what leaks or bypasses a control; exhaustive coverage of internal error paths needs source review.                                    |

For APIs, run the same exercise against the **OWASP API Security Top 10 (2023)** — API1 BOLA, API3 Broken Object Property Level Authorization (2019's excessive data exposure + mass assignment merged), API5 broken function-level authorization — using the **api-security-testing** skill.

## Run it

Maximum category coverage comes from giving the agents both the source and a running instance, plus credentials at two privilege levels:

```bash
strix -n \
  -t https://github.com/org/app \
  -t https://staging.example.com \
  --scan-mode deep --max-budget 30 \
  --instruction "OWASP Top 10:2025 assessment. Cover every category systematically and map each finding to its 2025 category id.
Accounts: userA@example.com/<pw> (org 1), userB@example.com/<pw> (org 2), admin@example.com/<pw>.
Prioritise A01 (cross-org access, privilege escalation, SSRF), A02, A05, A07, A10.
Out of scope: /billing/*, outbound email."
```

- `--scan-mode deep` matters here: systematically walking ten categories is not a quick scan.
- Without a second account, A01 results are structurally incomplete — say so in the report rather than leaving it implied.
- Need an auditor-facing PDF? Run it through the managed platform and pull the technical report (**managed-pentesting-with-strix**).

## Report honestly

From `strix_runs/<run>/`, group `vulnerabilities/*.md` by category and state, per category: what was attempted, what was proven, and what could not be assessed (A09 always; A03/A04/A06/A08/A10 partially). Label the report with the edition used. Verify each PoC yourself before it goes in front of the user.

A `0` exit code means nothing exploitable was proven **in what was analyzed** — check `run.json` status and cost against `--max-budget`; a budget-capped run is not a completed assessment.

## Then fix and re-test

Remediate with **fix-security-vulnerabilities-with-strix** and re-run to prove each exploit is closed. For ongoing coverage as the app changes, gate pull requests using **ci-security-scanning-with-strix**.
