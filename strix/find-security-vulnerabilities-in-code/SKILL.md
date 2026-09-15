---
name: find-security-vulnerabilities-in-code
description: 使用Strix在代码库或存储库中查找安全漏洞——这是一种白盒AI安全审查，可以读取您的源代码、实际数据流和授权模型的原因，然后利用它在实时沙箱中发现的内容，这样每个报告的问题都有一个可工作的概念证明，而不是嘈杂的静态分析警报。涵盖注入、XSS、SSRF、损坏的访问控制和IDOR、不安全的反序列化、代码中的秘密、不安全依赖关系和业务逻辑缺陷。当用户要求安全扫描、安全审查或审计其代码、仓库或漏洞拉取请求时使用。
license: Apache-2.0
metadata:
  author: usestrix
  homepage: https://docs.strix.ai
---
# Find security vulnerabilities in code

White-box security review with Strix: the agents read the source to build a model of routes, sinks, and authorization checks, then attempt real exploitation. Findings come with a proof-of-concept, so the output is a short list of proven issues rather than the hundreds of "potential" hits a pattern-matching scanner produces.

Install, LLM setup, all flags, and the managed-cloud path are in the **penetration-testing-with-strix** skill. For a run with no Docker and no LLM key, the same binary drives the managed platform: `strix cloud login`, then `strix cloud scans start ...` (details in **managed-pentesting-with-strix**).

## Run it

```bash
# Local working tree
strix -n -t ./ --scan-mode standard --max-budget 15

# A GitHub repo directly
strix -n -t https://github.com/org/app --max-budget 15

# Monorepo: point at the service that matters, not the whole tree
strix -n -t ./services/checkout --max-budget 20

# Only what a branch changed (whole-repo review is wasteful on a large repo)
strix -n -t ./ --scope-mode diff --diff-base origin/main --max-budget 10
```

A local path is mounted into the sandbox **writable**, so the agents can modify it. Run against a clean checkout.

Two things sharply improve results:

1. **Add a running instance of the app.** `-t ./ -t http://host.docker.internal:3000` lets the agents confirm exploitability against live behavior instead of reasoning about it statically — this is the difference between "this looks unsafe" and a validated finding. If nothing is running, static-only findings should be described as unconfirmed.
2. **Scope the review.** Point at the risky subtree and say what matters:
   ```bash
   strix -n -t ./services/api --max-budget 15 \
     --instruction "Focus on the authorization layer in src/auth and every route under src/routes/admin. Multi-tenant app: tenant id comes from the JWT. Flag any query that filters by object id without also filtering by tenant."
   ```

   Tenancy model, trust boundaries, and which inputs are attacker-controlled are things the agents cannot infer reliably — tell them.

## Reviewing a pull request instead of the whole repo

For diff-scoped review of a branch or PR (and blocking merges on findings), use **ci-security-scanning-with-strix** — it covers diff scoping, PR comments, and SARIF upload to GitHub code scanning. The managed platform can also review PRs directly via API (**managed-pentesting-with-strix**).

## Read the results

In `strix_runs/<run>/`: `penetration_test_report.md` (start here), `vulnerabilities/*.md` (one per finding, with PoC and remediation), `vulnerabilities.json` / `.csv`, `findings.sarif` (upload to code scanning), `run.json`.

Before reporting to the user, open each finding and check the PoC actually demonstrates impact. Report file and line alongside the exploit so the fix is obvious.

Exit `0` means nothing exploitable was proven in what was analyzed — not that the codebase is clean. Check `run.json` status and cost against `--max-budget`, and note which paths went unreviewed if the run was capped.

## Complementary tooling

This is exploit-validated review, not an exhaustive inventory. Keep a dependency scanner (SCA) and secret scanning in place for complete coverage of known-CVE dependencies and committed credentials; use this for the logic, authorization, and injection bugs those tools structurally cannot find.

## Fix and verify

Hand results to **fix-security-vulnerabilities-with-strix**: patch the root cause (the shared authorization helper, not the one route), then re-run Strix to prove the exploit no longer works.
