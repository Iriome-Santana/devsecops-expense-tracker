# Threat Model — SRE Expense Tracker

**Sistema analizado:** SRE Expense Tracker  
**Fecha:** Junio 2026  
**Autor:** Iriome Santana  
**Metodología:** STRIDE  
**Repositorios relacionados:**
- [expense-tracker-sre](https://github.com/Iriome-Santana/expense-tracker-sre) — aplicación
- [expense-tracker-terraform](https://github.com/Iriome-Santana/expense-tracker-terraform) — infraestructura
- [devsecops-expense-tracker](https://github.com/Iriome-Santana/devsecops-expense-tracker) — pipeline de seguridad

---

## 1. Descripción del sistema

El SRE Expense Tracker es una API REST desplegada en producción en AWS EC2 que permite a usuarios autenticados gestionar gastos personales. La aplicación corre en un contenedor Docker junto a una base de datos PostgreSQL, expone una API pública en el puerto 8000, y realiza backups automáticos a S3 en cada operación de escritura.

### Componentes principales

```
Internet
    │
    ▼
Elastic IP (52.31.3.15)
    │
    ▼
EC2 t3.micro (eu-west-1a)
    ├── Docker: FastAPI (puerto 8000)
    │       ├── API key authentication
    │       ├── Rate limiting (SlowAPI)
    │       └── Per-owner data isolation
    └── Docker: PostgreSQL 15 (puerto 5432)
            └── Solo accesible desde el contenedor API
    │
    ├── IAM Instance Profile → S3 backups
    ├── SSM Session Manager → acceso administrativo
    └── CloudWatch Agent → logs centralizados
```

### Flujo de datos

```
Cliente HTTP
    │ HTTPS/HTTP :8000
    ▼
FastAPI (autenticación API key)
    │ SQLAlchemy ORM
    ▼
PostgreSQL 15
    │ boto3 s3.put_object()
    ▼
S3 (expense-tracker-backups-iriome-2026)
```

---

## 2. Activos a proteger

| Activo | Clasificación | Descripción |
|---|---|---|
| Datos de gastos de usuarios | Confidencial | Información financiera personal de los usuarios registrados |
| API keys de usuarios | Secreto | Credenciales de acceso a la API |
| Credenciales de base de datos | Secreto | Usuario y contraseña de PostgreSQL |
| Backups en S3 | Confidencial | Exportaciones CSV de todos los datos de gastos |
| Instancia EC2 | Crítico | Plataforma de ejecución de toda la aplicación |
| Rol IAM de la instancia | Crítico | Permite acceso a S3 y SSM desde la EC2 |
| Pipeline CI/CD | Crítico | Acceso de escritura al código y al proceso de despliegue |

---

## 3. Actores

| Actor | Tipo | Nivel de confianza |
|---|---|---|
| Usuario autenticado | Externo | Bajo — accede solo a sus propios datos |
| Administrador (operador) | Interno | Alto — accede vía SSM y admin API |
| Atacante externo | Externo | Cero |
| Pipeline CI/CD (GitHub Actions) | Externo | Medio — acceso de despliegue vía SSM |
| AWS (proveedor cloud) | Externo | Alto — gestiona la infraestructura subyacente |

---

## 4. Análisis de amenazas STRIDE

### 4.1 Capa de red y acceso

---

**THREAT-001: Acceso no autorizado a la API**
- **Categoría STRIDE:** Spoofing
- **Componente:** FastAPI, API key authentication
- **Descripción:** Un atacante intenta acceder a la API sin API key válida o con una key de otro usuario.
- **Impacto:** Acceso a datos financieros de un usuario.
- **Probabilidad:** Alta — la API es pública en internet.
- **Control implementado:** API key authentication en todos los endpoints `/expenses/*`. Cada key está vinculada a un owner. Las queries filtran por owner, haciendo imposible acceder a datos de otro usuario aunque se tenga una key válida.
- **Riesgo residual:** Bajo. Si una key se compromete, el atacante solo puede ver los datos del owner de esa key.
- **Estado:** ✅ Mitigado

---

**THREAT-002: Fuerza bruta sobre API keys**
- **Categoría STRIDE:** Spoofing
- **Componente:** FastAPI, SlowAPI rate limiting
- **Descripción:** Un atacante intenta probar API keys por fuerza bruta haciendo miles de peticiones.
- **Impacto:** Compromiso de una API key válida.
- **Probabilidad:** Media.
- **Control implementado:** Rate limiting con SlowAPI — 60 peticiones/minuto en reads, 20 en writes. Las peticiones que superan el límite reciben 429 Too Many Requests.
- **Riesgo residual:** Bajo. El espacio de API keys es de 64 caracteres hexadecimales (2^256 combinaciones posibles). El rate limiting hace inviable la fuerza bruta.
- **Estado:** ✅ Mitigado

---

**THREAT-003: Acceso SSH no autorizado a la EC2**
- **Categoría STRIDE:** Spoofing, Elevation of Privilege
- **Componente:** EC2, security groups
- **Descripción:** Un atacante intenta conectarse vía SSH a la instancia EC2.
- **Impacto:** Acceso completo al sistema operativo, contenedores, y datos.
- **Probabilidad:** Alta — la instancia tiene IP pública.
- **Control implementado:** El puerto 22 solo está abierto para la IP del administrador (`85.55.121.247/32`) en `sg.public-web`. El acceso administrativo real usa SSM Session Manager, que no requiere puerto 22 abierto. El puerto 22 en el SG es un remanente de configuración inicial — candidato a eliminación.
- **Riesgo residual:** Medio. La restricción por IP protege contra atacantes externos, pero si la IP del administrador cambia o se compromete, el acceso SSH queda expuesto.
- **Mejora pendiente:** Eliminar la regla de puerto 22 del SG y usar exclusivamente SSM.
- **Estado:** ⚠️ Parcialmente mitigado

---

**THREAT-004: Ataque SSRF para robar credenciales del Instance Profile**
- **Categoría STRIDE:** Information Disclosure, Elevation of Privilege
- **Componente:** EC2, IMDSv2, IAM Instance Profile
- **Descripción:** Un atacante explota una vulnerabilidad SSRF en la aplicación para hacer una petición al endpoint de metadata de EC2 (`169.254.254.169`) y robar las credenciales temporales del Instance Profile, obteniendo acceso a S3.
- **Impacto:** Acceso de lectura/escritura al bucket S3 de backups.
- **Probabilidad:** Baja — requiere vulnerabilidad SSRF en la aplicación.
- **Control implementado:** La política IAM del Instance Profile sigue least-privilege: solo `s3:PutObject`, `s3:GetObject`, `s3:ListBucket` en el bucket específico. El daño está acotado.
- **Riesgo residual:** Medio. IMDSv2 no está forzado (deuda técnica documentada). Con IMDSv2 forzado, el SSRF no podría acceder al endpoint de metadata sin el token de sesión.
- **Mejora pendiente:** Forzar IMDSv2 (`http_tokens = "required"`) en la siguiente ventana de mantenimiento.
- **Estado:** ⚠️ Parcialmente mitigado

---

### 4.2 Capa de aplicación

---

**THREAT-005: Inyección SQL**
- **Categoría STRIDE:** Tampering, Information Disclosure
- **Componente:** FastAPI, SQLAlchemy ORM
- **Descripción:** Un atacante intenta inyectar SQL a través de los parámetros de la API para acceder a datos de otros usuarios o modificar la base de datos.
- **Impacto:** Acceso a datos de todos los usuarios, modificación o borrado de datos.
- **Probabilidad:** Baja — el ORM parametriza las queries automáticamente.
- **Control implementado:** SQLAlchemy ORM usa queries parametrizadas en todas las operaciones. No hay queries SQL construidas por concatenación de strings.
- **Riesgo residual:** Muy bajo.
- **Estado:** ✅ Mitigado

---

**THREAT-006: Exposición de stack traces en producción**
- **Categoría STRIDE:** Information Disclosure
- **Componente:** FastAPI, error handlers
- **Descripción:** Un error no manejado expone stack traces en la respuesta HTTP, revelando estructura interna del código, rutas de ficheros, o información de la base de datos.
- **Impacto:** Información útil para un atacante para planificar ataques más sofisticados.
- **Probabilidad:** Media sin mitigación.
- **Control implementado:** Handler genérico de excepciones en `main.py` captura todas las excepciones no manejadas y devuelve un error genérico sin stack trace. Los stack traces solo aparecen en los logs internos.
- **Riesgo residual:** Muy bajo.
- **Estado:** ✅ Mitigado

---

**THREAT-007: Abuso de la API admin para crear API keys maliciosas**
- **Categoría STRIDE:** Elevation of Privilege
- **Componente:** Admin endpoint `/admin/api-keys`
- **Descripción:** Un atacante descubre o compromete el `ADMIN_SECRET` y crea API keys para acceder a datos de cualquier usuario.
- **Impacto:** Acceso completo a todos los datos de la aplicación.
- **Probabilidad:** Baja si el secret es robusto.
- **Control implementado:** El endpoint admin requiere el header `X-Admin-Secret`. El secret se configura vía variable de entorno, no está en el código.
- **Riesgo residual:** Medio. El secret está en un fichero `.env` en la EC2, no en AWS Secrets Manager. Si la instancia se compromete, el secret es accesible.
- **Mejora pendiente:** Migrar a AWS Secrets Manager.
- **Estado:** ⚠️ Parcialmente mitigado

---

### 4.3 Capa de datos

---

**THREAT-008: Acceso no autorizado a backups en S3**
- **Categoría STRIDE:** Information Disclosure
- **Componente:** S3 bucket, IAM policy
- **Descripción:** Un atacante accede al bucket S3 con los backups CSV de todos los gastos de los usuarios.
- **Impacto:** Exposición de datos financieros de todos los usuarios.
- **Probabilidad:** Baja.
- **Control implementado:** Public Access Block activado en el bucket (las cuatro opciones). IAM policy con least privilege — solo la EC2 puede acceder vía Instance Profile. No hay credenciales de AWS en el código ni en el repositorio.
- **Riesgo residual:** Bajo.
- **Estado:** ✅ Mitigado

---

**THREAT-009: Pérdida de datos por borrado accidental o ataque**
- **Categoría STRIDE:** Tampering
- **Componente:** S3 bucket, versioning
- **Descripción:** Un administrador borra datos accidentalmente, o un atacante con acceso a la instancia borra los backups.
- **Impacto:** Pérdida de datos históricos de gastos.
- **Probabilidad:** Baja.
- **Control implementado:** S3 Versioning activado en el bucket de backups. Los objetos borrados se convierten en delete markers y las versiones anteriores son recuperables. Lifecycle policy retiene datos hasta 365 días.
- **Riesgo residual:** Bajo.
- **Estado:** ✅ Mitigado

---

**THREAT-010: Exposición de credenciales en el repositorio**
- **Categoría STRIDE:** Information Disclosure
- **Componente:** GitHub, CI/CD pipeline, Terraform
- **Descripción:** Credenciales reales (contraseñas de BD, API keys, tokens) se commitean accidentalmente al repositorio público.
- **Impacto:** Acceso inmediato a la infraestructura por cualquier actor malicioso.
- **Probabilidad:** Media sin mitigación — es el error más común en proyectos con credenciales.
- **Control implementado:** Gitleaks en pre-commit hook (bloquea el commit localmente) y en GitHub Actions (escanea el historial en cada push). `.env` en `.gitignore`. `terraform.tfvars` en `.gitignore`. Variables sensibles marcadas como `sensitive = true` en Terraform.
- **Riesgo residual:** Bajo.
- **Estado:** ✅ Mitigado

---

### 4.4 Capa de infraestructura y CI/CD

---

**THREAT-011: Compromiso del pipeline CI/CD**
- **Categoría STRIDE:** Tampering, Elevation of Privilege
- **Componente:** GitHub Actions, SSM
- **Descripción:** Un atacante compromete el repositorio o los secrets de GitHub Actions e inyecta código malicioso en el pipeline, que se despliega automáticamente en producción.
- **Impacto:** Código malicioso ejecutándose en la EC2 de producción.
- **Probabilidad:** Baja.
- **Control implementado:** El despliegue usa SSM Session Manager — no hay puerto 22 abierto al pipeline. Las credenciales AWS del pipeline tienen permisos mínimos para el comando SSM. Los cambios al código pasan por el pipeline de seguridad (Bandit, Gitleaks, Trivy) antes del despliegue.
- **Riesgo residual:** Medio. Si los secrets de GitHub Actions se comprometen, un atacante podría ejecutar comandos arbitrarios en la EC2 vía SSM.
- **Mejora pendiente:** Activar branch protection con reviews requeridas en main.
- **Estado:** ⚠️ Parcialmente mitigado

---

**THREAT-012: Vulnerabilidades en dependencias y en la imagen Docker**
- **Categoría STRIDE:** Tampering, Elevation of Privilege
- **Componente:** Docker image, Python dependencies
- **Descripción:** Una dependencia Python o un paquete del sistema operativo en la imagen Docker tiene una vulnerabilidad conocida que permite ejecución de código o escalada de privilegios.
- **Impacto:** Variable según la CVE — desde denegación de servicio hasta ejecución remota de código.
- **Probabilidad:** Alta sin mitigación — las CVEs se publican continuamente.
- **Control implementado:** Trivy escanea la imagen en Docker Hub en cada push y en el schedule nocturno. pip-audit escanea las dependencias Python. El pipeline bloquea si hay CVEs HIGH o CRITICAL con fix disponible. apt-get upgrade en el Dockerfile asegura que los paquetes del sistema estén actualizados en cada build.
- **Riesgo residual:** Bajo. El sistema demostró detectar y corregir CVEs reales en producción de forma autónoma.
- **Estado:** ✅ Mitigado

---

## 5. Resumen de riesgos

| ID | Amenaza | Severidad | Estado |
|---|---|---|---|
| THREAT-001 | Acceso no autorizado a la API | Alta | ✅ Mitigado |
| THREAT-002 | Fuerza bruta sobre API keys | Media | ✅ Mitigado |
| THREAT-003 | Acceso SSH no autorizado | Alta | ⚠️ Parcial |
| THREAT-004 | SSRF para robar credenciales IMDSv1 | Media | ⚠️ Parcial |
| THREAT-005 | Inyección SQL | Alta | ✅ Mitigado |
| THREAT-006 | Exposición de stack traces | Baja | ✅ Mitigado |
| THREAT-007 | Compromiso del ADMIN_SECRET | Media | ⚠️ Parcial |
| THREAT-008 | Acceso no autorizado a S3 | Alta | ✅ Mitigado |
| THREAT-009 | Pérdida de datos | Media | ✅ Mitigado |
| THREAT-010 | Credenciales en repositorio | Alta | ✅ Mitigado |
| THREAT-011 | Compromiso del pipeline CI/CD | Media | ⚠️ Parcial |
| THREAT-012 | Vulnerabilidades en dependencias | Alta | ✅ Mitigado |

**Mitigados:** 8 de 12  
**Parcialmente mitigados:** 4 de 12  
**Sin mitigar:** 0 de 12

---

## 6. Mejoras pendientes por prioridad

| Prioridad | Mejora | Amenaza relacionada | Esfuerzo |
|---|---|---|---|
| Alta | Forzar IMDSv2 (`http_tokens = "required"`) | THREAT-004 | Requiere recreación de instancia |
| Alta | Eliminar regla SSH del SG y usar solo SSM | THREAT-003 | Bajo — cambio en Terraform |
| Media | Migrar ADMIN_SECRET a AWS Secrets Manager | THREAT-007 | Medio |
| Media | Branch protection con reviews en main | THREAT-011 | Bajo — configuración GitHub |
| Baja | Cifrado KMS en EBS | THREAT-004 | Requiere recreación de instancia |

---

## 7. Controles de seguridad implementados

### Pipeline de seguridad automático
- **Gitleaks:** detecta secrets en código e historial Git antes de que lleguen al repositorio
- **Bandit:** análisis estático de código Python buscando patrones inseguros
- **Trivy:** escanea la imagen Docker buscando CVEs en paquetes del SO y dependencias
- **pip-audit:** escanea dependencias Python contra la NVD
- **Checkov:** analiza la infraestructura Terraform buscando misconfiguraciones

### Controles en la aplicación
- Autenticación con API keys por owner con aislamiento de datos
- Rate limiting con SlowAPI (60/min reads, 20/min writes)
- Validación de inputs con Pydantic v2
- Error handlers genéricos sin exposición de stack traces
- Logging estructurado con JSON y run_id único por sesión

### Controles en la infraestructura
- IAM least privilege con política custom acotada al bucket específico
- SSM Session Manager para acceso administrativo sin puerto 22
- S3 Public Access Block en las cuatro dimensiones
- S3 Versioning para recuperación ante borrado accidental
- VPC Flow Logs para auditoría de tráfico de red
- Default security group de la VPC sin reglas de tráfico
- CloudWatch Agent para logs centralizados en producción

### Controles en el proceso de desarrollo
- Pre-commit hooks con Gitleaks
- `.env` y `terraform.tfvars` en `.gitignore`
- Variables sensibles con `sensitive = true` en Terraform
- ADRs documentando cada decisión técnica y de seguridad
