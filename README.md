# DevSecOps Security Pipeline

![Security Pipeline](https://github.com/Iriome-Santana/devsecops-expense-tracker/actions/workflows/security-pipeline.yml/badge.svg)
![Schedule](https://img.shields.io/badge/schedule-nightly%202AM%20UTC-blue)
![Tools](https://img.shields.io/badge/tools-Gitleaks%20%7C%20Bandit%20%7C%20Trivy%20%7C%20pip--audit-green)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## What is this?

Este repo no es una aplicación. Es una **capa de seguridad independiente** aplicada sobre infraestructura real ya desplegada: el [SRE Expense Tracker](https://github.com/Iriome-Santana/expense-tracker-sre), una API REST corriendo en producción en AWS EC2.

La decisión de mantener la seguridad en un repo separado es deliberada y está documentada. En entornos profesionales, el equipo de seguridad opera sobre los sistemas existentes desde fuera, no vive dentro del equipo de producto. Este repo replica ese modelo: un pipeline de seguridad autónomo que escanea una aplicación externa, bloquea si encuentra problemas reales, y se ejecuta automáticamente cada noche aunque no haya habido cambios en el código.

La seguridad no depende de que nadie se acuerde de ejecutarla. Está automatizada.

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

El expense tracker tiene credenciales reales en producción, una imagen Docker desplegada en AWS, dependencias Python con versiones concretas, y código que podría contener patrones inseguros. Sin un proceso sistemático, cualquier problema de seguridad en cualquiera de esas capas es invisible hasta que es explotado.

Este proyecto automatiza la detección en cuatro capas independientes:

- **Secrets** en el código y en el historial de Git
- **Patrones inseguros** en el código Python
- **CVEs** en la imagen Docker desplegada en producción
- **CVEs** en las dependencias Python de la aplicación

Cada capa usa una herramienta diferente con una base de datos diferente. Si una no lo detecta, otra sí. Eso es defense in depth.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              devsecops-expense-tracker (este repo)              │
│                                                                 │
│  .github/workflows/security-pipeline.yml                        │
│                                                                 │
│  Triggers:                                                      │
│  ├── push to main                                               │
│  ├── pull request to main                                       │
│  └── schedule: every day at 2:00 AM UTC                        │
└────────────────────────────┬────────────────────────────────────┘
                             │ apunta a
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│           expense-tracker-sre (repo objetivo)                   │
│                                                                 │
│  ├── src/              ← Bandit escanea aquí                    │
│  ├── pyproject.toml    ← pip-audit escanea las dependencias     │
│  └── historial Git     ← Gitleaks escanea todo el historial     │
└─────────────────────────────────────────────────────────────────┘
                             │
                             │ imagen publicada en
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│           Docker Hub: iriome2512/expense-tracker:latest         │
│                                                                 │
│           Trivy escanea esta imagen directamente                │
└─────────────────────────────────────────────────────────────────┘
```

El pipeline corre en runners efímeros de GitHub Actions. No requiere infraestructura propia ni coste adicional. Todo el trabajo ocurre en la nube, no en la máquina del desarrollador.

---

## Pipeline

```
Push / PR / Schedule (2AM UTC)
           │
           ▼
┌──────────────────────────────┐
│  Job 1: secret-scan          │
│  Gitleaks                    │
│  Escanea historial completo  │
│  del repo del expense tracker│
│  BLOQUEA si encuentra secrets│
└──────────────┬───────────────┘
               │ needs: secret-scan
               ▼
┌──────────────────────────────┐
│  Job 2: sast                 │
│  Bandit                      │
│  Analiza src/ buscando       │
│  patrones Python inseguros   │
│  BLOQUEA en severidad HIGH   │
│  Reporte JSON como artefacto │
└──────────────┬───────────────┘
               │ needs: sast
               ▼
┌──────────────────────────────┐
│  Job 3: container-scan       │
│  Trivy                       │
│  Escanea imagen en Docker Hub│
│  BLOQUEA en CRITICAL/HIGH    │
│  con fix disponible          │
│  ignore-unfixed: true        │
└──────────────┬───────────────┘
               │ needs: container-scan
               ▼
┌──────────────────────────────┐
│  Job 4: dependency-scan      │
│  pip-audit                   │
│  Escanea dependencias Python │
│  declaradas en pyproject.toml│
│  BLOQUEA si encuentra CVEs   │
└──────────────────────────────┘
```

Cada job depende del anterior. Si Gitleaks encuentra un secret, los tres jobs siguientes no se ejecutan. El deploy del expense tracker solo ocurre si su propio CI/CD pasa — este pipeline es una capa adicional de visibilidad y alerta, no el gate de deploy.

El schedule diario tiene un beneficio que un trigger por push no tiene: detecta CVEs nuevos publicados en las bases de datos aunque no haya habido cambios en el código. Una vulnerabilidad publicada hoy aparecerá en el scan de mañana.

---

## What was found and fixed

Estos son los problemas reales que el pipeline encontró durante su implementación y que se corrigieron en el expense tracker como resultado directo.

### pip CVE-2025-8869, CVE-2026-6357, CVE-2026-1703
**Herramienta:** Trivy  
**Severidad:** MEDIUM  
**Causa:** pip 25.0.1 instalado en la imagen Docker tenía tres CVEs con fix disponible  
**Fix:** pip actualizado a 26.1 en el Dockerfile  
**Commit:** `fix(security): upgrade pip to 26.1 to address CVE-2025-8869, CVE-2026-6357, CVE-2026-1703`

### setuptools CVE-2024-6345, PYSEC-2025-49
**Herramienta:** pip-audit  
**Severidad:** HIGH / MEDIUM  
**Causa:** setuptools 68.1.2 declarado en pyproject.toml tenía CVEs con fix disponible  
**Fix:** setuptools actualizado a 78.1.1  
**Commit:** `fix(security): upgrade setuptools to 78.1.1 and pytest to 9.0.3`

### pytest CVE-2025-71176
**Herramienta:** pip-audit  
**Severidad:** MEDIUM  
**Causa:** pytest 7.4.3 (dependencia de dev) tenía CVE con fix disponible  
**Fix:** pytest actualizado a 9.0.3  
**Commit:** `fix(security): upgrade setuptools to 78.1.1 and pytest to 9.0.3`

### 7 CVEs HIGH en imagen base Debian — aceptados
**Herramienta:** Trivy  
**Severidad:** HIGH  
**Estado:** Sin fix disponible en Debian a fecha del análisis  
**Decisión:** Aceptados con criterio documentado en `docs/security-findings/trivy-risk-acceptance.md`  
Las vulnerabilidades afectan a ncurses, systemd y libcap2. Ninguna es accesible desde la superficie de ataque de la API. El pipeline las ignora con `ignore-unfixed: true` y las revisa automáticamente en cada ejecución nocturna.

---

## Tools

| Herramienta | Versión | Qué escanea | Bloquea en |
|---|---|---|---|
| Gitleaks | v8.18.4 | Secrets en código e historial Git | Cualquier secret detectado |
| Bandit | latest | Patrones inseguros en código Python | Severidad HIGH |
| Trivy | latest | CVEs en imagen Docker | CRITICAL/HIGH con fix disponible |
| pip-audit | latest | CVEs en dependencias Python | Cualquier CVE encontrado |

Todas las herramientas son open source y gratuitas. El pipeline corre en GitHub Actions gratis para repos públicos. Coste total de infraestructura: 0€.

---

## Repository structure

```
devsecops-expense-tracker/
├── .github/
│   └── workflows/
│       └── security-pipeline.yml   ← pipeline completo
├── docs/
│   ├── adr/
│   │   ├── 001-gitleaks-secret-detection.md
│   │   ├── 002-bandit-sast.md
│   │   ├── 003-trivy-container-security.md
│   │   └── 004-pip-audit-dependency-scan.md
│   └── security-findings/
│       └── trivy-risk-acceptance.md
├── .gitleaks.toml                   ← allowlist de falsos positivos
└── .pre-commit-config.yaml          ← hooks locales pre-commit
```

---

## Architecture Decision Records

Las decisiones técnicas de este proyecto están documentadas en `docs/adr/`. Cada ADR explica el contexto, las alternativas consideradas, la decisión tomada y sus consecuencias.

### Por qué un repo separado y no dentro del expense tracker

Meter el pipeline de seguridad dentro del repo del expense tracker sería más simple pero narrativamente incorrecto. La seguridad es una disciplina que opera sobre sistemas desde fuera, no una feature que vive dentro de la aplicación. Un repo separado refleja ese modelo mental y demuestra que se entiende la diferencia.

### Por qué schedule diario y no repository dispatch

Repository dispatch requiere configurar tokens entre repos y añadir jobs al pipeline del expense tracker. El schedule es más simple y tiene una ventaja que repository dispatch no tiene: detecta CVEs nuevos publicados en las bases de datos aunque no haya habido cambios en el código. La seguridad no es solo reactiva a cambios, también es proactiva ante nuevas vulnerabilidades.

### Por qué ignore-unfixed en Trivy

Bloquear el pipeline por vulnerabilidades sin fix disponible genera ruido sin valor: no hay acción posible. La política correcta es bloquear solo cuando existe un fix que no se ha aplicado, y documentar con criterio los findings sin fix en el risk acceptance. Eso es lo que hace un equipo de seguridad real.

### Por qué estas cuatro herramientas en este orden

El orden refleja el coste de detección tardía. Los secrets son lo más crítico porque son irreversibles una vez expuestos: primero Gitleaks. El código inseguro es más fácil de corregir antes de buildear que después: segundo Bandit. La imagen ya buildeada se escanea después: tercero Trivy. Las dependencias se verifican al final porque dependen de que la imagen exista: cuarto pip-audit.

---

## Further documentation

| Documento | Contenido |
|---|---|
| [docs/adr/001-gitleaks-secret-detection.md](docs/adr/001-gitleaks-secret-detection.md) | Por qué Gitleaks, configuración de allowlist, política de bloqueo |
| [docs/adr/002-bandit-sast.md](docs/adr/002-bandit-sast.md) | Por qué Bandit vs SonarQube vs Semgrep, resultado del análisis inicial |
| [docs/adr/003-trivy-container-security.md](docs/adr/003-trivy-container-security.md) | Por qué Trivy vs Snyk, política ignore-unfixed, schedule vs repository dispatch |
| [docs/adr/004-pip-audit-dependency-scan.md](docs/adr/004-pip-audit-dependency-scan.md) | Por qué pip-audit vs OWASP Dependency-Check, versiones explícitas |
| [docs/security-findings/trivy-risk-acceptance.md](docs/security-findings/trivy-risk-acceptance.md) | Findings HIGH sin fix disponible, análisis de riesgo, metodología de decisión |

---

## Author

Built by **Iriome Santana** as part of a self-taught journey into Site Reliability Engineering and DevSecOps.

This project demonstrates that security is not a checklist added at the end. It is an automated layer that runs independently, finds real problems, and documents every decision with engineering criteria.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Iriome%20Santana-0077B5?logo=linkedin)](https://www.linkedin.com/in/iriome-santana-socorro)

> 💬 **Feedback welcome.** If you're learning DevSecOps and want to discuss the architecture decisions, open an issue or reach out on LinkedIn.