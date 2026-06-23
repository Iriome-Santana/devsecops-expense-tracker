# ADR 005 — Checkov para seguridad de infraestructura como código

## Contexto

El repo expense-tracker-terraform define 18 recursos AWS. Sin análisis
automático de seguridad, misconfiguraciones como políticas IAM demasiado
permisivas, S3 buckets públicos, o instancias sin monitoring pueden
introducirse sin detección.

## Decisión

Implementar Checkov en el pipeline de GitHub Actions escaneando el repo
de Terraform en cada push. El job depende de que dependency-scan pase
primero. Se ejecuta con soft_fail: false — cualquier finding no
documentado bloquea el pipeline.

## Resultado del análisis inicial

49 checks passed, 20 failed en la primera ejecución. Tras correcciones:

Corregidos en el código:
- VPC Flow Logs activados con retención de 365 días
- Política IAM de flow logs acotada al log group específico
- Detailed monitoring activado en EC2
- Abort multipart uploads añadido al lifecycle de S3
- Default security group de la VPC restringido

Aceptados con excepción documentada en .checkov.yaml:
- EBS sin cifrar — requiere recreación de instancia, deuda técnica documentada
- IMDSv2 no forzado — mismo motivo
- S3 con AES256 en lugar de KMS — suficiente para este caso de uso
- Subnet pública con IP automática — arquitectura intencional
- SGs con egress abierto — necesario para acceso a Docker Hub y AWS APIs

## Por qué Checkov y no tfsec o Terrascan

Checkov tiene la mayor cobertura de checks para AWS, acción oficial de
GitHub Actions, y fichero de configuración YAML para excepciones
documentadas. tfsec y Terrascan son alternativas válidas pero con menor
ecosistema de integraciones.

## Consecuencias

Cualquier nueva misconfiguration introducida en el Terraform que no esté
en el allowlist bloqueará el pipeline automáticamente. Las excepciones
requieren justificación explícita en .checkov.yaml — no se pueden ignorar
findings silenciosamente.
