---
name: web-app-penetration-testing
description: 对web应用程序或网站进行端到端的Pentest——对实时URL、临时环境或本地开发服务器进行黑盒测试，发现并利用真正的漏洞（身份验证绕过、访问控制中断、IDOR、注入、XSS、SSRF、业务逻辑），并用可工作的概念证明而不是签名匹配来证明每个漏洞。使用Strix运行，可以是自托管的开源CLI或托管的app.Strix.ai云。当用户要求对其web应用程序、网站、web应用程序或临时站点进行渗透测试、黑客攻击、安全测试或审计时使用。
license: Apache-2.0
metadata:
  author: usestrix
  homepage: https://docs.strix.ai
---
# Pentest a web application

Black-box (and optionally source-assisted) penetration testing of a running web app with Strix's autonomous agents. Every reported finding is validated with a working exploit, so there are no signature-based false positives to triage.

Install, LLM setup, all CLI flags, and the managed-cloud alternative are covered in the **penetration-testing-with-strix** skill — read it if the target is not a running web app, or if `strix --version` fails. For a run with no Docker and no LLM key, the same binary drives the managed platform: `strix cloud login`, then `strix cloud scans start ...` (details in **managed-pentesting-with-strix**). This skill is the web-app-specific workflow.

## 1. Confirm authorization and scope

Before running anything, establish:

- **The target is the user's** (or they are explicitly authorized to test it). Never pentest a third-party site on a hunch.
- **Which environment.** Prefer staging over production; agents send real exploit payloads and will create/modify data.
- **Out-of-scope paths** — payment flows, mass-email endpoints, admin destructive actions, third-party SSO providers.
- **Credentials.** Most real vulnerabilities live behind login. Without a test account, the agents only ever see the marketing surface.

Ask for anything missing rather than guessing.

## 2. Run the scan

```bash
strix -n -t https://staging.example.com --max-budget 20 \
  --instruction "Test account: qa@example.com / <password>. In scope: /app/*, /api/*. Do not touch /billing or send email. Focus on access control between the two seeded orgs."
```

Notes that matter for web apps specifically:

- **Give it credentials via `--instruction`** (or `--instruction-file` for anything long), including how to log in if the flow is unusual (magic link, SSO, MFA-exempt test user).
- **Two accounts beat one.** Multi-tenant IDOR and broken-access-control bugs — consistently the highest-impact class in web apps — can only be proven when the agent can attempt cross-account access.
- **Add the repo for white-box depth** when you have the source: `-t https://github.com/org/app -t https://staging.example.com` (or a local path). Source access materially improves coverage of business-logic and authorization flaws.
- **Localhost works.** Point at `http://host.docker.internal:3000` (Docker Desktop) so the sandbox can reach a dev server on the host.
- `--scan-mode quick` for a fast dev-loop pass, `standard` (~30 min) for a normal review, `deep` for pre-release assurance. Always set `--max-budget`.

For a hosted run with no Docker/LLM key, or when the user wants a shareable dashboard and an auditor-ready PDF, use the cloud path in **managed-pentesting-with-strix** instead — same engine, same findings.

## 3. Review results

Read `strix_runs/<run>/penetration_test_report.md` first, then per-finding files in `vulnerabilities/`. Each contains the PoC — re-run it yourself to confirm before reporting to the user.

Exit codes: `0` no validated vulns in what was analyzed, `2` vulnerabilities found, `1` fatal error. A `0` is not proof of full coverage — if the budget or turn cap was hit the scan wraps up early, so check `run.json` status and cost against `--max-budget` before calling the app clean.

## 4. Fix and verify

Hand findings to the **fix-security-vulnerabilities-with-strix** skill: patch the root cause, then re-run Strix against the same target to prove the exploit no longer works. Re-testing is the only reliable confirmation a fix landed.

To keep the app tested on every change rather than once, wire Strix into CI with **ci-security-scanning-with-strix**.
