# ADR 004 — pip-audit para escaneo de dependencias Python

## Contexto

El expense tracker tiene dependencias Python declaradas en pyproject.toml.
Cualquiera de estas puede tener CVEs conocidos en versiones desactualizadas.
Trivy ya escanea las dependencias Python dentro de la imagen Docker, pero
usa una base de datos diferente a la NVD. Tener dos herramientas con bases
de datos distintas escaneando lo mismo es defense in depth: capas de defensa
independientes que se complementan.

## Decisión

Implementar pip-audit en el pipeline escaneando las dependencias directas
del expense tracker con versiones explícitas. El job depende de que
container-scan pase primero.

## Resultado del análisis inicial

El análisis inicial sobre el entorno del sistema encontró 49 findings
en 20 paquetes, pero la mayoría correspondían a dependencias del sistema
Ubuntu, no del expense tracker.

Aislando únicamente las dependencias reales del proyecto se encontraron
4 findings en 2 paquetes, todos con fix disponible:

- pytest 7.4.3 → CVE-2025-71176 → actualizado a 9.0.3
- setuptools 68.1.2 → CVE-2024-6345, PYSEC-2025-49 → actualizado a 78.1.1

Ambas dependencias actualizadas en pyproject.toml del expense tracker.

## Por qué pip-audit y no OWASP Dependency-Check

OWASP Dependency-Check es una herramienta Java diseñada para múltiples
ecosistemas. Para Python específicamente, pip-audit es más preciso,
más ligero, y está mantenido por el equipo de seguridad de pip y Google.
Usar la herramienta específica del ecosistema da mejores resultados que
una herramienta genérica.

## Por qué versiones explícitas en el pipeline

pip-audit necesita versiones exactas para escanear con precisión.
Las versiones con >= generan ambigüedad sobre qué versión está
realmente instalada. Esta forma es más explícita y más fácil
de mantener. Cuando se actualiza una dependencia en el expense
tracker, se actualiza también esta lista.

## Consecuencias

El pipeline bloquea si cualquier dependencia directa del expense
tracker tiene un CVE conocido. El developer debe actualizar la
dependencia afectada antes de que el deploy llegue a producción.
