# ADR 001 — Gitleaks para detección de secrets

## Contexto

El expense tracker maneja credenciales reales en producción: contraseña
de base de datos, ADMIN_SECRET para gestión de API keys, y credenciales
de AWS para acceso a S3. Si cualquiera de estos valores acabara en el
repositorio de Git, estaría expuesto públicamente de forma permanente
porque Git tiene memoria: borrar un fichero no elimina su contenido
del historial.

GitHub es escaneado activamente por bots automatizados que buscan
patrones de credenciales AWS. El tiempo entre un push accidental y
el primer uso malicioso se mide en minutos.

## Decisión

Implementar Gitleaks en dos capas:

1. Pre-commit hook local — bloquea el commit antes de que el secret
   salga de la máquina del desarrollador
2. GitHub Actions — escanea el historial completo del repo en cada
   push como segunda línea de defensa

## Por qué Gitleaks y no alternativas

**TruffleHog** fue considerado. Tiene más reglas por defecto pero
genera más falsos positivos y su integración con pre-commit es más
compleja. Para un proyecto de una persona, Gitleaks es más simple
de configurar y mantener.

**detect-secrets de Yelp** fue considerado. Requiere mantener un
fichero de baseline actualizado manualmente. Gitleaks es más
ergonómico para este caso de uso.

## Configuración de allowlist

El fichero `.gitleaks.toml` ignora el `.env.example` completo y
patrones de placeholder explícitos. Esta decisión reduce el ruido
de falsos positivos sin reducir la cobertura sobre código real.

## Consecuencias

Todo commit que contenga un patrón reconocido como secret es
bloqueado automáticamente. El desarrollador debe o corregir
el secret o añadir una excepción documentada al allowlist
con justificación explícita.
