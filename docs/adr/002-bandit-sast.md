# ADR 002 — Bandit para análisis estático de seguridad (SAST)

## Contexto

El expense tracker tiene 407 líneas de código Python en producción.
El análisis estático de seguridad (SAST) busca patrones de programación
inseguros en el código fuente antes de que lleguen a producción:
uso inseguro de subprocess, algoritmos de hash débiles, binding
a 0.0.0.0 sin justificación, uso de assert para validaciones
de seguridad, entre otros.

## Decisión

Implementar Bandit como herramienta de SAST sobre el directorio
src/ del expense tracker en el pipeline de GitHub Actions.
El job depende de que secret-scan pase primero.

## Resultado del análisis inicial

Bandit escaneó 407 líneas de código y no encontró ningún issue
de ninguna severidad. El código del expense tracker no contiene
patrones de programación inseguros conocidos por Bandit.

## Por qué Bandit y no alternativas

**SonarQube** fue considerado. Es más completo pero requiere
infraestructura propia o cuenta en SonarCloud. Para un proyecto
de una persona con presupuesto cero, Bandit es la herramienta
correcta: sin dependencias externas, instalable con pip,
resultados claros.

**Semgrep** fue considerado. Más potente y con reglas personalizables,
pero más complejo de configurar para empezar. Bandit cubre los
patrones más comunes en Python con configuración mínima.

## Política de bloqueo

Actualmente el job no bloquea el pipeline aunque encuentre issues
(flag || true). Cuando se identifiquen y revisen los findings
reales, la política cambiará a bloquear en severidad HIGH o CRITICAL
sin fix disponible.

## Consecuencias

Cada push genera un reporte JSON descargable como artefacto
en GitHub Actions con el resultado completo del análisis.
Cualquier patrón inseguro introducido en el código futuro
será detectado en el siguiente push.
