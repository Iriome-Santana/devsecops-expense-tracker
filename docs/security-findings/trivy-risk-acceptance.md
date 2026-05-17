# Trivy Risk Acceptance — expense-tracker image

Fecha del análisis: mayo 2026
Imagen analizada: iriome2512/expense-tracker:latest
Herramienta: Trivy v0.70.0

---

## Findings resueltos

### pip CVE-2025-8869, CVE-2026-6357, CVE-2026-1703
**Severidad:** MEDIUM / LOW  
**Fix disponible:** Sí  
**Acción:** pip actualizado a 26.1 en el Dockerfile del expense tracker.  
**Commit:** fix(security): upgrade pip to 26.1  
**Estado:** Resuelto ✅

### CVE-2026-4878 — libcap2
**Severidad:** HIGH
**Fix disponible:** Sí — parcheado el 2026-05-17
**Acción:** apt-get upgrade añadido al Dockerfile del expense tracker
**Commit:** fix(security): add apt-get upgrade to patch libcap2 CVE-2026-4878 and systemd CVE-2026-29111
**Estado:** Resuelto ✅

### CVE-2026-29111 — systemd (libsystemd0, libudev1)
**Severidad:** HIGH
**Fix disponible:** Sí — parcheado el 2026-05-17
**Acción:** apt-get upgrade añadido al Dockerfile del expense tracker
**Commit:** fix(security): add apt-get upgrade to patch libcap2 CVE-2026-4878 and systemd CVE-2026-29111
**Estado:** Resuelto ✅

---

## Findings aceptados como riesgo

### CVE-2025-69720 — ncurses (libncursesw6, libtinfo6, ncurses-base, ncurses-bin)
**Severidad:** HIGH  
**Fix disponible:** No — Debian no ha publicado parche  
**Descripción:** Buffer overflow en ncurses que puede llevar a ejecución
de código arbitrario.  
**Análisis de riesgo:** ncurses es una librería de interfaz de terminal.
El expense tracker es una API REST sin interfaz de terminal interactiva
en producción. La superficie de ataque de esta vulnerabilidad no es
accesible desde el exterior de la aplicación.  
**Decisión:** Aceptado hasta que Debian publique fix. Revisar en cada
ejecución del pipeline.  
**Estado:** Aceptado con criterio ⚠️


---

## Metodología de decisión

Para cada finding HIGH o CRITICAL sin fix disponible se evalúa:

1. **¿Es la superficie de ataque accesible desde el exterior?**
   Si la vulnerabilidad requiere acceso local o afecta a componentes
   que no están activos en el contenedor, el riesgo real es menor
   que la severidad nominal indica.

2. **¿Existe fix disponible?**
   Si no existe fix, la única alternativa sería cambiar la imagen base,
   lo cual introduce otros riesgos. Se acepta y se revisa periódicamente.

3. **¿Afecta a las dependencias Python de la aplicación?**
   Las dependencias Python del expense tracker tienen 0 vulnerabilidades.
   El riesgo está concentrado en la imagen base de Debian, no en el
   código de la aplicación.

---

## Próxima revisión

Ejecutar Trivy de nuevo cuando Debian publique actualizaciones de
seguridad para ncurses, systemd y libcap2. El pipeline de GitHub
Actions ejecuta
