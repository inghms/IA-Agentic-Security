# AWS Security Hub Agent — Bedrock AgentCore Runtime

Agente de análisis de seguridad que consulta AWS Security Hub y genera resúmenes ejecutivos con recomendaciones de remediación, utilizando AWS Bedrock AgentCore Runtime con Strands Agents.

## Arquitectura

```
Usuario (lenguaje natural)
       ↓
AgentCore Runtime (HTTP POST /invocations)
       ↓
BedrockAgentCoreApp + @app.entrypoint
       ↓
Strands Agent (Claude Sonnet + Tools)
       ↓                    ↓
  Tool Calls            Respuesta Final
       ↓                    ↓
  securityhub_tools     Resumen ejecutivo
  (boto3 directo)       + recomendaciones
       ↓
  AWS Security Hub API
       ↓
  Hallazgos / Insights / Standards
```

## Estructura del Proyecto

```
handy_security_agent/
├── agent/
│   ├── __init__.py
│   ├── main.py                    # Entry point (BedrockAgentCoreApp + Strands Agent)
│   ├── agent_handler.py           # Handler alternativo (Converse API directa)
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── securityhub_tools.py   # 7 herramientas de Security Hub (boto3)
│   │   └── tool_registry.py       # Registro de tools (para handler alternativo)
│   └── utils/
│       ├── __init__.py
│       ├── pagination.py          # Paginación de APIs de Security Hub
│       ├── error_handler.py       # Manejo centralizado de errores AWS
│       └── logger.py              # Logging estructurado (JSON)
├── agentcore/
│   ├── agentcore.json             # Configuración del proyecto AgentCore
│   ├── aws-targets.json           # Cuenta y región de despliegue
│   └── .env.local                 # Variables locales (gitignored)
├── config/
│   ├── agent_config.yaml          # Configuración adicional del agente
│   ├── runtime_config.json        # Configuración detallada del runtime
│   └── trust-policy.json          # Trust policy para IAM Role
├── examples/
│   ├── test_agent.py              # Script de pruebas
│   └── example_responses.md       # Ejemplos de respuestas del agente
├── .gitignore
├── Dockerfile
├── pyproject.toml
├── requirements.txt
├── deploy.sh
└── README.md
```

## Requisitos Previos

- Python 3.11+
- Node.js 20+ (para AgentCore CLI)
- Cuenta AWS con Security Hub habilitado
- Acceso a Amazon Bedrock (Claude Sonnet habilitado)
- AWS CLI v2 configurado
- AWS CDK bootstrapped (`cdk bootstrap`)
- [uv](https://docs.astral.sh/uv/) instalado (recomendado para gestión de dependencias)

## Instalación

### 1. Instalar AgentCore CLI

```bash
npm install -g @aws/agentcore
agentcore --help
```

### 2. Instalar dependencias Python

```bash
# Con uv (recomendado)
uv venv
uv pip install -r requirements.txt

# O con pip
pip install -r requirements.txt
```

### 3. Configurar cuenta AWS

Edita `agentcore/aws-targets.json` con tu Account ID y región:

```json
{
  "targets": [
    {
      "accountId": "123456789012",
      "region": "us-east-1"
    }
  ]
}
```

## Desarrollo Local

```bash
# Iniciar servidor de desarrollo con hot-reload
agentcore dev

# O con logs visibles en la terminal
agentcore dev --logs

# Invocar el agente localmente (en otra terminal)
agentcore dev "¿Qué hallazgos críticos existen?"
```

El servidor local arranca en `http://localhost:8080` y expone:
- `POST /invocations` — Invocar el agente
- `GET /ping` — Health check

## Despliegue a AgentCore Runtime

### Opción 1: AgentCore CLI (Recomendada)

```bash
# Vista previa del despliegue
agentcore deploy --plan

# Desplegar
agentcore deploy

# Verificar estado
agentcore status
```

### Opción 2: Container (Docker)

```bash
# Build para ARM64 (requerido por AgentCore Runtime / Graviton)
docker buildx build --platform linux/arm64 -t security-hub-agent .

# Deploy con build tipo Container
# Cambiar "build": "Container" en agentcore/agentcore.json
agentcore deploy
```

## Invocar el Agente Desplegado

```bash
# Invocación simple
agentcore invoke "¿Qué hallazgos críticos existen?"

# Con streaming
agentcore invoke --stream "Resume los findings de esta semana"

# Mantener sesión conversacional
agentcore invoke --session-id sesion-1 "¿Cuáles son los riesgos más importantes?"
agentcore invoke --session-id sesion-1 "¿Y qué recomendaciones tienes?"

# Chat interactivo (TUI)
agentcore invoke
```

## Herramientas Disponibles

| Herramienta | Descripción | Parámetros clave |
|---|---|---|
| `tool_get_securityhub_findings` | Hallazgos con filtros generales | severity, status, days_back |
| `tool_get_high_severity_findings` | Solo CRITICAL y HIGH activos | max_results, days_back |
| `tool_get_findings_by_resource` | Por ARN de recurso | resource_arn |
| `tool_get_findings_by_account` | Por cuenta AWS | account_id, severity |
| `tool_get_findings_by_region` | Por región | target_region, severity |
| `tool_get_securityhub_insights` | Insights configurados | max_results |
| `tool_list_enabled_standards` | Estándares y controles | region |

## Permisos IAM Requeridos

El agente necesita un IAM Role con las siguientes políticas:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "securityhub:GetFindings",
        "securityhub:GetInsights",
        "securityhub:GetInsightResults",
        "securityhub:GetEnabledStandards",
        "securityhub:DescribeStandardsControls",
        "securityhub:ListEnabledProductsForImport"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": "arn:aws:bedrock:*::foundation-model/anthropic.*"
    }
  ]
}
```

O usar la política administrada `AWSSecurityHubReadOnlyAccess`.

## Ejemplos de Consultas

```
"¿Qué hallazgos críticos existen?"
"¿Cuáles son los riesgos más importantes?"
"Resume los findings de esta semana."
"¿Qué recursos presentan mayor riesgo?"
"¿Hay controles que estén fallando?"
"¿Qué recomendaciones tienes para reducir el riesgo?"
"Muestra hallazgos de la cuenta 123456789012"
"¿Qué problemas hay en us-east-1?"
"¿Qué estándares de seguridad tenemos habilitados?"
```

Ver `examples/example_responses.md` para respuestas de ejemplo completas.

## Observabilidad

### Logs

```bash
# Streaming de logs del agente desplegado
agentcore logs

# Ver trazas recientes
agentcore traces list
```

### Métricas por herramienta

Cada invocación registra en logs estructurados (JSON → CloudWatch):
- `tool_name`, duración, éxito/error
- `session_id` para correlación
- Severity counts por consulta

### Alertas recomendadas (CloudWatch Alarms)

- Errores `ACCESS_DENIED` → Revisar IAM
- Timeouts frecuentes → Revisar red/VPC
- Alto volumen de findings CRITICAL → Notificar equipo de seguridad

## Buenas Prácticas de Seguridad

1. **IAM Roles** — Nunca hardcodear credenciales; AgentCore usa roles automáticamente
2. **Mínimo privilegio** — Solo permisos de lectura sobre Security Hub
3. **Sin secretos en código** — Variables sensibles en `.env.local` (gitignored)
4. **Logs sin datos sensibles** — No loguear ARNs completos ni contenido de findings
5. **Cifrado en tránsito** — TLS automático en AgentCore Runtime
6. **Validación de inputs** — Sanitización antes de construir filtros de Security Hub
7. **Imagen base mínima** — `python:3.11-slim` sin paquetes innecesarios
8. **Escaneo de vulnerabilidades** — ECR con `scanOnPush=true`
9. **Timeout de sesiones** — Configurar `IdleRuntimeSessionTimeout` para control de costos

## Limpieza

```bash
# Remover todos los recursos y destruir infraestructura
agentcore remove all
agentcore deploy
```

## Licencia

Uso interno — Adaptar según necesidad.
