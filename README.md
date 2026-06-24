# DevSecOps Security Pipeline

![Security Pipeline](https://github.com/Iriome-Santana/devsecops-expense-tracker/actions/workflows/security-pipeline.yml/badge.svg)
![Schedule](https://img.shields.io/badge/schedule-nightly%202AM%20UTC-blue)
![Tools](https://img.shields.io/badge/tools-Gitleaks%20%7C%20Bandit%20%7C%20Trivy%20%7C%20pip--audit%20%7C%20Checkov-green)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## What is this?

This repository implements a continuous security pipeline for a production-like AWS-hosted application.

Instead of embedding security checks inside the application repository, this project models security as an independent operational layer that continuously scans, validates and reports risks against a live system.

This separation is intentional and models security as an independent operational concern rather than an application feature.

The target system is the [SRE Expense Tracker](https://github.com/Iriome-Santana/expense-tracker-sre): a REST API running in production on AWS EC2 with automated CI/CD, PostgreSQL, and S3 backups. The infrastructure is defined as code in [expense-tracker-terraform](https://github.com/Iriome-Santana/expense-tracker-terraform).

---

## Key Outcomes

- Automated nightly security scanning against a live AWS-hosted application
- Detected and fixed real CVEs in pip, setuptools, pytest, starlette and openssl
- Defense-in-depth scanning across five independent layers: secrets, SAST, container vulnerabilities, dependency vulnerabilities, and IaC misconfigurations
- Implemented documented risk acceptance for unfixed Debian CVEs with attack surface analysis
- Zero additional infrastructure cost using GitHub Actions ephemeral runners
- Nightly schedule detected newly published fixes for libcap2 and systemd CVEs that were previously unfixable — patched within hours of detection
- Checkov IaC scan reduced Terraform misconfigurations from 20 findings to 0 actionable findings

---

## Table of Contents

- [Why this exists](#why-this-exists)
- [Architecture](#architecture)
- [Pipeline](#pipeline)
- [What was found and fixed](#what-was-found-and-fixed)
- [Tools](#tools)
- [Repository structure](#repository-structure)
- [Architecture Decision Records](#architecture-decision-records)
- [Further documentation](#further-documentation)
- [Author](#author)

---

## Why this exists

The expense tracker has real credentials in production, a Docker image deployed on AWS, Python dependencies with specific versions, Terraform infrastructure code, and application code that could contain insecure patterns. Without a systematic process, any security issue across those layers is invisible until exploited.

This project automates detection across five independent layers:

- **Secrets** in code and Git history
- **Insecure patterns** in Python code
- **CVEs** in the Docker image deployed in production
- **CVEs** in the application's Python dependencies
- **Misconfigurations** in the Terraform infrastructure code

Each layer uses a different tool with a different database. If one misses something, another catches it. That is defense in depth.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              devsecops-expense-tracker (this repo)              │
│                                                                 │
│  .github/workflows/security-pipeline.yml                        │
│                                                                 │
│  Triggers:                                                      │
│  ├── push to main                                               │
│  ├── pull request to main                                       │
│  └── schedule: every day at 2:00 AM UTC                        │
└────────────────────────────┬────────────────────────────────────┘
                             │ targets
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│           expense-tracker-sre (target repo)                     │
│                                                                 │
│  ├── src/              ← Bandit scans here                      │
│  ├── pyproject.toml    ← pip-audit scans dependencies           │
│  └── Git history       ← Gitleaks scans full history            │
└─────────────────────────────────────────────────────────────────┘
                             │
                             │ image published at
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│           Docker Hub: iriome2512/expense-tracker:latest         │
│                                                                 │
│           Trivy scans this image directly                       │
└─────────────────────────────────────────────────────────────────┘
                             │
                             │ infrastructure defined at
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│           expense-tracker-terraform (target repo)               │
│                                                                 │
│           Checkov scans .tf files for misconfigurations         │
└─────────────────────────────────────────────────────────────────┘
```

The pipeline runs on ephemeral GitHub Actions runners. No additional infrastructure required. No additional cost. Everything runs in the cloud, not on the developer's machine.

---

## Pipeline

```
Push / PR / Schedule (2AM UTC)
           │
           ▼
┌──────────────────────────────┐
│  Job 1: secret-scan          │
│  Gitleaks                    │
│  Scans full Git history      │
│  of the expense tracker repo │
│  BLOCKS if secrets found     │
└──────────────┬───────────────┘
               │ needs: secret-scan
               ▼
┌──────────────────────────────┐
│  Job 2: sast                 │
│  Bandit                      │
│  Scans src/ for insecure     │
│  Python patterns             │
│  BLOCKS on HIGH severity     │
│  JSON report as artifact     │
└──────────────┬───────────────┘
               │ needs: sast
               ▼
┌──────────────────────────────┐
│  Job 3: container-scan       │
│  Trivy                       │
│  Scans image on Docker Hub   │
│  BLOCKS on CRITICAL/HIGH     │
│  with fix available          │
│  ignore-unfixed: true        │
└──────────────┬───────────────┘
               │ needs: container-scan
               ▼
┌──────────────────────────────┐
│  Job 4: dependency-scan      │
│  pip-audit                   │
│  Scans declared Python deps  │
│  from pyproject.toml         │
│  BLOCKS if CVEs found        │
└──────────────┬───────────────┘
               │ needs: dependency-scan
               ▼
┌──────────────────────────────┐
│  Job 5: iac-scan             │
│  Checkov                     │
│  Scans Terraform repo for    │
│  IaC misconfigurations       │
│  BLOCKS on any finding not   │
│  in .checkov.yaml allowlist  │
└──────────────────────────────┘
```

Each job depends on the previous one. If Gitleaks finds a secret, the four following jobs do not run. The expense tracker deploy only happens if its own CI/CD passes — this pipeline is an additional visibility and alerting layer, not the deploy gate.

The nightly schedule has a benefit that a push trigger does not: it detects new CVEs published in vulnerability databases even when no code has changed. A vulnerability published today will appear in tomorrow's scan.

---

## What was found and fixed

These are the real issues the pipeline found during implementation and operation, corrected as a direct result.

### pip CVE-2025-8869, CVE-2026-6357, CVE-2026-1703
**Tool:** Trivy · **Severity:** MEDIUM
**Fix:** pip upgraded to 26.1 in the Dockerfile

### setuptools CVE-2024-6345, PYSEC-2025-49
**Tool:** pip-audit · **Severity:** HIGH / MEDIUM
**Fix:** setuptools upgraded to 78.1.1 in pyproject.toml

### pytest CVE-2025-71176
**Tool:** pip-audit · **Severity:** MEDIUM
**Fix:** pytest upgraded to 9.0.3 in pyproject.toml

### starlette PYSEC-2026-161, CVE-2026-54283
**Tool:** pip-audit · **Severity:** MEDIUM / HIGH
**Fix:** FastAPI upgraded to 0.136.3, starlette pinned to >=1.3.1

### openssl CVE-2026-45447
**Tool:** Trivy · **Severity:** HIGH
**Fix:** apt-get upgrade in Dockerfile pulled the patched Debian package

### libcap2 CVE-2026-4878, systemd CVE-2026-29111
**Tool:** Trivy · **Severity:** HIGH
**Fix:** Detected by nightly schedule when Debian published the patches. apt-get upgrade in Dockerfile resolved both. Patched within hours of detection.

### 20 Terraform misconfigurations
**Tool:** Checkov · **Severity:** Various
**Fix:** VPC Flow Logs added, EC2 detailed monitoring enabled, IAM policy scoped to specific resources, default security group restricted, S3 lifecycle abort rule added. Remaining findings documented with justified exceptions in `.checkov.yaml`.

### 7 HIGH CVEs in Debian base image — accepted
**Tool:** Trivy · **Severity:** HIGH
**Status:** No fix available in Debian at time of analysis. Accepted with documented criteria in `docs/security-findings/trivy-risk-acceptance.md`.

---

## Tools

| Tool | Version | What it scans | Blocks on |
|---|---|---|---|
| Gitleaks | v8.18.4 | Secrets in code and Git history | Any detected secret |
| Bandit | latest | Insecure patterns in Python code | HIGH severity |
| Trivy | latest | CVEs in Docker image | CRITICAL/HIGH with available fix |
| pip-audit | latest | CVEs in Python dependencies | Any CVE found |
| Checkov | latest | Terraform IaC misconfigurations | Any finding not in allowlist |

All tools are open source and free. The pipeline runs on GitHub Actions for free on public repos. Total infrastructure cost: €0.

---

## Repository structure

```
devsecops-expense-tracker/
├── .github/
│   └── workflows/
│       └── security-pipeline.yml   ← full pipeline (5 jobs)
├── docs/
│   ├── adr/
│   │   ├── 001-gitleaks-secret-detection.md
│   │   ├── 002-bandit-sast.md
│   │   ├── 003-trivy-container-security.md
│   │   ├── 004-pip-audit-dependency-scan.md
│   │   └── 005-checkov-iac-security.md
│   ├── security-findings/
│   │   └── trivy-risk-acceptance.md
│   └── threat-model.md             ← STRIDE threat model
├── .gitleaks.toml                   ← false positive allowlist
└── .pre-commit-config.yaml          ← local pre-commit hooks
```

---

## Architecture Decision Records

Technical decisions for this project are documented in `docs/adr/`. Each ADR explains the context, alternatives considered, the decision taken and its consequences.

### Why a separate repo instead of inside the expense tracker

Putting the security pipeline inside the expense tracker repository would be simpler but architecturally incorrect. This separation models security as an independent operational concern rather than an application feature — and makes that design decision explicit and reviewable.

### Why nightly schedule instead of repository dispatch

Repository dispatch requires configuring cross-repo tokens and adding jobs to the expense tracker pipeline. The schedule is simpler and has one advantage repository dispatch does not: it detects new CVEs published in vulnerability databases even when no code has changed. Security is not only reactive to code changes, it is also proactive against new vulnerabilities.

### Why ignore-unfixed in Trivy

Blocking the pipeline for vulnerabilities with no available fix generates noise without value: there is no possible action. The correct policy is to block only when a fix exists and has not been applied, and to document findings without a fix using documented risk acceptance criteria. That is what a real security team does.

### Why these five tools in this order

The order reflects the cost of late detection. Secrets are the most critical because they are irreversible once exposed: Gitleaks first. Insecure code is easier to fix before building than after: Bandit second. The already-built image is scanned next: Trivy third. Dependencies are verified after the image exists: pip-audit fourth. Infrastructure code is scanned last as it targets a separate repository: Checkov fifth.

### Why Checkov with a documented allowlist instead of soft_fail

Running Checkov with soft_fail permanently means findings are informational and never block anything. The correct approach is to triage findings once, fix what can be fixed, document exceptions with explicit justifications, and then enforce hard fail. An undocumented exception is a silent ignored finding. A documented exception is a conscious risk decision.

---

## Further documentation

| Document | Content |
|---|---|
| [docs/adr/001-gitleaks-secret-detection.md](docs/adr/001-gitleaks-secret-detection.md) | Why Gitleaks, allowlist configuration, blocking policy |
| [docs/adr/002-bandit-sast.md](docs/adr/002-bandit-sast.md) | Why Bandit vs SonarQube vs Semgrep, initial analysis result |
| [docs/adr/003-trivy-container-security.md](docs/adr/003-trivy-container-security.md) | Why Trivy vs Snyk, ignore-unfixed policy, schedule vs repository dispatch |
| [docs/adr/004-pip-audit-dependency-scan.md](docs/adr/004-pip-audit-dependency-scan.md) | Why pip-audit vs OWASP Dependency-Check, explicit versions |
| [docs/adr/005-checkov-iac-security.md](docs/adr/005-checkov-iac-security.md) | Why Checkov, findings resolved, exception policy |
| [docs/security-findings/trivy-risk-acceptance.md](docs/security-findings/trivy-risk-acceptance.md) | HIGH findings without available fix, risk analysis, decision methodology |
| [docs/threat-model.md](docs/threat-model.md) | STRIDE threat model — assets, threats, controls, residual risks |

---

## Author

Built by **Iriome Santana** as part of a self-taught journey into Site Reliability Engineering and DevSecOps.

This project demonstrates that security is not a checklist added at the end. It is an automated layer that runs independently, finds real problems, and documents every decision with engineering criteria.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Iriome%20Santana-0077B5?logo=linkedin)](https://www.linkedin.com/in/iriome-santana-socorro)

> 💬 **Feedback welcome.** If you're learning DevSecOps and want to discuss the architecture decisions, open an issue or reach out on LinkedIn.