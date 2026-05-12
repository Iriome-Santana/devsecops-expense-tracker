# ADR 003 — Trivy para seguridad de contenedores

## Contexto

La imagen Docker del expense tracker hereda el sistema operativo
base de python:3.12-slim (Debian). Cualquier vulnerabilidad conocida
en los paquetes del sistema operativo o en las dependencias Python
está presente en la imagen desplegada en producción en AWS EC2.

Sin escaneo de imagen, estas vulnerabilidades son invisibles hasta
que son explotadas.

## Decisión

Implementar Trivy en el pipeline de GitHub Actions para escanear
la imagen publicada en Docker Hub en cada push. El job depende de
que sast pase primero.

## Resultado del análisis inicial

El análisis inicial encontró:
- 112 vulnerabilidades en Debian (0 CRITICAL, 7 HIGH, 42 MEDIUM, 63 LOW)
- 4 vulnerabilidades en pip (0 HIGH, 3 MEDIUM, 1 LOW)
- 0 vulnerabilidades en dependencias Python de la aplicación

Acciones tomadas:
- pip actualizado a 26.1 resolviendo CVE-2025-8869, CVE-2026-6357
  y CVE-2026-1703
- 7 findings HIGH sin fix disponible documentados en
  docs/security-findings/trivy-risk-acceptance.md y aceptados
  con criterio de riesgo

## Política de bloqueo

El pipeline bloquea si encuentra vulnerabilidades CRITICAL o HIGH
con fix disponible. Los findings sin fix disponible están excluidos
con ignore-unfixed: true. Esta política equilibra seguridad real
con operabilidad: no bloquea el pipeline por vulnerabilidades que
no se pueden resolver, pero sí bloquea si hay un fix disponible
que no se ha aplicado.

## Por qué Trivy y no Snyk

Snyk tiene un tier gratuito limitado y requiere cuenta externa.
Trivy es open source, sin límites, con acción oficial de GitHub,
y sin dependencias externas más allá de su base de datos pública
de CVEs. Para un proyecto de una persona con presupuesto cero,
Trivy es la elección correcta.

## Consecuencias

Cualquier actualización de la imagen base que introduzca un nuevo
CVE con fix disponible bloqueará el pipeline automáticamente.
El developer tendrá que actualizar la dependencia afectada antes
de que el deploy llegue a producción.

## Por qué schedule y no repository dispatch

Repository dispatch requiere configurar un Personal Access Token
con permisos entre repos y añadir un job extra al pipeline del
expense tracker. La complejidad añadida no justifica el beneficio
para un proyecto de una persona.

El schedule tiene además una ventaja que repository dispatch no
tiene: detecta CVEs nuevos publicados en la base de datos de Trivy
aunque no haya habido cambios en el código. Una vulnerabilidad
publicada hoy aparecerá en el scan de mañana sin necesidad de
hacer ningún commit. La seguridad no es solo reactiva a cambios
de código sino también proactiva ante nuevas vulnerabilidades.
