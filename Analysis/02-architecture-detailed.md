# GPT-RAG Orchestrator: Arquitectura Detallada

## Tabla de contenidos
- [1. Stack Tecnológico Completo](#1-stack-tecnológico-completo)
- [2. Estructura de Directorios](#2-estructura-de-directorios)
- [3. Arquitectura de Componentes](#3-arquitectura-de-componentes)
- [4. Modelo de Datos](#4-modelo-de-datos)
- [5. Flujo de Ejecución de Tareas](#5-flujo-de-ejecución-de-tareas)
- [6. Motor de Procesamiento / Orquestación](#6-motor-de-procesamiento--orquestación)
- [7. Modelo de Seguridad](#7-modelo-de-seguridad)
- [8. Comparación de Modos de Despliegue](#8-comparación-de-modos-de-despliegue)
- [9. Patrones de Extensibilidad](#9-patrones-de-extensibilidad)

## 1. Stack Tecnológico Completo

### Backend

| Categoría | Tecnología | Versión / detalle |
| --- | --- | --- |
| Lenguaje | Python | 3.12 |
| API | FastAPI | `0.115.12` |
| ASGI server | Uvicorn | `0.34.1` |
| Modelado | Pydantic | `2.11.4` |
| HTTP async | aiohttp | `3.13.3` |
| Retries | tenacity | sin pin explícito |
| Templates | Jinja2 | `3.1.6` |
| JWT | PyJWT | `2.8.0` |

### Azure / IA

| Área | Librería | Versión / detalle |
| --- | --- | --- |
| Foundry Projects | `azure-ai-projects` | `2.0.0b3` |
| Agents | `azure-ai-agents` | `>=1.2.0b4` |
| Search | `azure-search-documents` | `11.7.0b1` |
| Identity | `azure-identity` | `1.21.0` |
| Cosmos | `azure-cosmos` | `4.5.1` |
| Key Vault | `azure-keyvault-secrets` | `4.8.0` |
| App Config | `azure-appconfiguration` | `1.7.1` |
| OpenAI client | `openai` | `2.9.0` |
| Tokenization | `tiktoken` | `0.8.0` |

### Agentes y orquestación

| Área | Tecnología | Uso |
| --- | --- | --- |
| Agentes Foundry | `AIProjectClient`, `AgentsClient` | RAG v1 y v2 |
| Semantic Kernel | `semantic-kernel[mcp]` `1.34.0` | MCP y NL2SQL multi-agente |
| MCP | `MCPSsePlugin` | Conexión a herramientas externas |

### NL2SQL y datos tabulares

| Área | Tecnología | Uso |
| --- | --- | --- |
| SQL parsing | `sqlparse` | validación sintáctica |
| Conectividad SQL | `pyodbc` | Azure SQL / Fabric SQL endpoint |
| Power BI REST | `aiohttp` + token Entra | consultas DAX sobre semantic models |

### Observabilidad

| Tecnología | Uso |
| --- | --- |
| `azure-monitor-opentelemetry` | integración con Azure Monitor |
| `opentelemetry-instrumentation-fastapi` | trazas de FastAPI |
| `opentelemetry-instrumentation-httpx` | trazas HTTP salientes |

### Infraestructura, CI/CD y contenedores

| Área | Tecnología |
| --- | --- |
| Contenedor | Docker + `python:3.12-slim` |
| Empaquetado | `requirements.txt` |
| IaC local | Bicep (`infra/main.bicep` mínimo) |
| Orquestación de despliegue | `azure.yaml`, `azd deploy`, scripts bash/PowerShell |
| CI/CD | GitHub Actions |

### Versiones mínimas de runtime y sistema

| Componente | Requisito |
| --- | --- |
| Python | 3.12 |
| Docker | requerido para build local |
| PowerShell | 5.1+ o Core |
| Bash | shell POSIX |
| ODBC Driver 18 | requerido para SQL / Fabric |

## 2. Estructura de Directorios

```text
.
├── Analysis/                    # Documentación generada en este trabajo
├── dataset/                     # Golden dataset para evaluación
├── evaluations/                 # Scripts y docs de evaluación offline
├── infra/                       # Infra mínima local; la infraestructura completa vive en el repo padre
├── notebooks/                   # Notebook de pruebas
├── samples/                     # Catalogo de datasources, tablas y queries de ejemplo
├── scripts/                     # Deploy y prevalidaciones
├── src/                         # Código fuente del orquestador
│   ├── connectors/              # Clientes de Azure y backends externos
│   ├── orchestration/           # Clase Orchestrator
│   ├── plugins/                 # Plugins retrieval, NL2SQL y common
│   ├── prompts/                 # Prompts versionados
│   ├── strategies/              # Implementaciones de estrategias
│   ├── telemetry/               # Logging y OpenTelemetry
│   └── util/                    # utilidades de entorno y JWT
└── tests/                       # Script utilitario RBAC / ACL para Search
```

### Archivos y carpetas clave

| Ruta | Propósito |
| --- | --- |
| `src/main.py` | Bootstrap de FastAPI, lifecycle, endpoint y auth principal |
| `src/dependencies.py` | Auth técnica, validación de tokens Entra, caché de JWKS |
| `src/orchestration/orchestrator.py` | Clase central de sesión y persistencia |
| `src/strategies/` | Implementaciones de RAG, MCP y NL2SQL |
| `src/connectors/` | Clientes de App Config, Search, Foundry, Cosmos, Key Vault y SQL/Fabric |
| `src/plugins/` | Funciones kernel y modelos de retorno |
| `src/prompts/` | Prompts textuales y Jinja2 por estrategia |
| `evaluations/` | Generación de input y ejecución de evaluación en Azure AI Projects |
| `samples/` | Corpus de ejemplos para NL2SQL |
| `scripts/deploy.*` | Build, push y actualización de Container App |

## 3. Arquitectura de Componentes

### 3.1 API

**Responsabilidad principal**

- Exponer la interfaz HTTP del orquestador.
- Resolver autenticación técnica y de usuario.
- Lanzar la estrategia apropiada.
- Stream de salida por SSE.

**Endpoints / interfaces expuestas**

| Método | Ruta | Descripción |
| --- | --- | --- |
| `POST` | `/orchestrator` | pregunta al orquestador o envía feedback |

**Payload principal**

```python
class OrchestratorRequest(BaseModel):
    ask: Optional[str]
    question: Optional[str]
    type: Optional[str]
    question_id: Optional[str]
    is_positive: Optional[bool]
    stars_rating: Optional[int]
    feedback_text: Optional[str]
    conversation_id: Optional[str]
    user_context: Optional[Dict[str, Any]]
```

**Dependencias externas**

- Azure App Configuration
- Cosmos DB
- Azure AI Foundry
- Azure AI Search
- Key Vault

**Diagrama interno**

```text
Request
  -> validate_auth
  -> validate_access_token (si Authorization)
  -> Orchestrator.create
  -> strategy.initiate_agent_flow
  -> StreamingResponse
```

### 3.2 Orchestrator

**Responsabilidad principal**

- Gestionar el ciclo de vida de la conversación.
- Resolver `AGENT_STRATEGY`.
- Unir estrategia y persistencia.

**Métodos relevantes**

| Método | Firma |
| --- | --- |
| `create` | `async def create(conversation_id=None, user_context={}, request_access_token=None)` |
| `stream_response` | `async def stream_response(self, ask: str, question_id: Optional[str] = None)` |
| `save_feedback` | `async def save_feedback(self, feedback: Dict)` |

**Dependencias externas**

- Cosmos DB para conversación.
- StrategyFactory para resolución de estrategia.

### 3.3 Strategy Layer

#### BaseAgentStrategy

Responsabilidad:

- Cargar configuración base.
- Inicializar credenciales y `AIProjectClient`.
- Resolver directorios de prompts.
- Exponer contrato abstracto `initiate_agent_flow`.

Firma principal:

```python
@abstractmethod
async def initiate_agent_flow(self, user_message: str):
    ...
```

#### SingleAgentRAGStrategyV2

Responsabilidad:

- RAG principal optimizado para latencia.
- Reutilizar `AgentsClient` y un agente cacheado.
- Bypass local a LLM cuando el índice está vacío.

Dependencias externas:

- Azure AI Search
- Azure AI Foundry Agents
- Azure OpenAI embeddings

Flujo interno:

```text
user_message
  -> is_index_empty()
     -> if empty: direct LLM streaming
     -> else: AgentsClient + thread + agent + tool calls
```

#### SingleAgentRAGStrategyV1

Responsabilidad:

- Implementación anterior basada en `project_client.agents`.
- Crea o reutiliza thread y agent.
- Activa auto function calls sobre `search_knowledge_base`.

#### McpStrategy

Responsabilidad:

- Conectar herramientas externas vía MCP SSE.
- Inyectar `user-context` como header.
- Usar `ChatCompletionAgent` de Semantic Kernel con un plugin MCP.

Dependencias externas:

- servidor MCP
- Azure AI model endpoint

#### NL2SQLStrategy

Responsabilidad:

- Crear tres agentes especializados.
- Orquestar selección de datasource, generación de SQL/DAX y síntesis.

Agentes:

- `TriageAgent`
- `SQLQueryAgent`
- `SyntetizerAgent`

Flujo interno:

```text
question
  -> TriageAgent
  -> SQLQueryAgent
  -> SyntetizerAgent
  -> stream final
```

### 3.4 Connectors

| Componente | Responsabilidad principal | Servicios externos |
| --- | --- | --- |
| `AppConfigClient` | bulk load de configuración y Key Vault refs | App Configuration, Key Vault |
| `IdentityManager` | credenciales singleton sync/async | Managed Identity, Azure CLI |
| `SearchClient` | búsqueda híbrida, OBO, trimming | Azure AI Search |
| `GenAIModelClient` | chat y embeddings | Foundry / Azure OpenAI |
| `CosmosDBClient` | CRUD de documentos | Cosmos DB |
| `SQLDBClient` | consultas Azure SQL | Azure SQL Database |
| `SQLEndpointClient` | consultas Fabric SQL Endpoint | Microsoft Fabric |
| `SemanticModelClient` | consultas DAX REST | Power BI / Fabric |

### 3.5 Plugins

| Plugin | Responsabilidad |
| --- | --- |
| `CommonPlugin` | utilidades de fecha y hora |
| `RetrievalPlugin` | retrieval vectorial/multimodal legado |
| `NL2SQLPlugin` | catálogo de datasources, tablas, queries, validación y ejecución |

### 3.6 Infra y despliegue

| Artefacto | Rol |
| --- | --- |
| `azure.yaml` | hooks de `predeploy` y `preprovision` |
| `scripts/deploy.sh` / `.ps1` | build, push y update de Container App |
| `scripts/preProvision.sh` / `.ps1` | bloquean `azd provision` y remiten al repo padre |
| `Dockerfile` | imagen runtime con ODBC Driver 18 |

## 4. Modelo de Datos

### 4.1 Entidad: conversación

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `id` | string | ID único de conversación |
| `thread_id` | string | thread remoto de agentes |
| `agent_id` | string | agent ID remoto |
| `questions` | array | pares `question_id` + texto |
| `messages` | array | historial consolidado |
| `feedback` | array | evaluación del usuario |
| `user_context` | object | principal y contexto adicional |
| `prompt_tokens` | int | consumo de entrada |
| `completion_tokens` | int | consumo de salida |

Ejemplo realista:

```json
{
  "id": "3eae8a0d-c0b0-41a6-835e-fdd2a4cbfa54",
  "thread_id": "thread_abc",
  "agent_id": "agent_xyz",
  "questions": [
    {
      "question_id": "q-42",
      "text": "¿Qué ciudades tuvieron mayor utilidad?"
    }
  ],
  "messages": [
    {
      "role": "assistant",
      "text": "Las cinco ciudades con mayor utilidad fueron..."
    }
  ],
  "feedback": [
    {
      "question_id": "q-42",
      "is_positive": true,
      "stars_rating": 5,
      "feedback_text": "Respuesta correcta"
    }
  ]
}
```

### 4.2 Entidad: datasource

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `id` | string | identificador lógico |
| `description` | string | descripción funcional |
| `type` | string | `sql_database`, `sql_endpoint`, `semantic_model` |
| `server` | string | servidor SQL / endpoint |
| `database` | string | base de datos |
| `dataset` | string | semantic model dataset |
| `workspace` | string | workspace Fabric / Power BI |
| `tenant_id` | string | tenant ID |
| `client_id` | string | app registration / SP |
| `uid` | string | usuario SQL opcional |

Ejemplo desde samples:

```json
{
  "id": "adventureworks",
  "description": "AdventureWorks LT is a database...",
  "type": "sql_database",
  "database": "adventureworkslt",
  "server": "sqlservername.database.windows.net"
}
```

### 4.3 Entidad: índices de Search

#### Índice RAG

Campos consumidos por el runtime:

| Campo | Tipo | Uso |
| --- | --- | --- |
| `title` | string | citación |
| `content` | string | grounding textual |
| `url` | string | enlace fuente |
| `filepath` | string | fuente alternativa de cita |
| `chunk_id` | string | correlación de chunk |
| `contentVector` | vector | búsqueda híbrida/vectorial |

#### Índices NL2SQL

| Índice | Campos consumidos |
| --- | --- |
| `SEARCH_TABLES_INDEX_NAME` | `table`, `description`, `datasource`, `columns`, `contentVector` |
| `SEARCH_MEASURES_INDEX_NAME` | `name`, `description`, `datasource`, `source_table`, `data_type`, `source_model` |
| `SEARCH_QUERIES_INDEX_NAME` | `question`, `query`, `reasoning`, `datasource`, `contentVector` |

### 4.4 Metadata de seguridad documental

El código y la documentación oficial apuntan a un modelo de permisos donde Azure AI Search puede usar:

- `userIds`
- `groupIds`
- `rbacScope`
- o filtros equivalentes si el pipeline de ingestión preserva esos metadatos

## 5. Flujo de Ejecución de Tareas

### 5.1 Flujo técnico paso a paso

1. `main.py::orchestrator_endpoint` recibe el request.
2. `dependencies.py::validate_auth` valida `dapr-api-token` o `X-API-KEY`.
3. Si existe `Authorization`, `validate_access_token` valida el JWT.
4. `Orchestrator.create` resuelve la estrategia mediante `AgentStrategyFactory.get_strategy`.
5. `Orchestrator.stream_response` crea o carga la conversación.
6. La estrategia asignada ejecuta `initiate_agent_flow`.
7. Dependiendo de la estrategia:
   - RAG V2 consulta `SearchClient.is_index_empty` y puede usar `prewarm_agents_client`.
   - RAG V1 habilita auto function calls y crea agent/thread.
   - MCP crea `MCPSsePlugin` y llama a un agente SK.
   - NL2SQL crea tres agentes y usa `NL2SQLPlugin`.
8. Se emiten chunks por `StreamingResponse`.
9. `Orchestrator.stream_response` agenda `persist_conversation` con `asyncio.create_task`.

### 5.2 Manejo de errores, reintentos y timeouts

| Área | Mecanismo |
| --- | --- |
| App Configuration | `tenacity` con backoff exponencial aleatorio |
| Search OBO | errores explícitos y degradación controlada si `ALLOW_ANONYMOUS=true` |
| Embeddings | retry único si OpenAI devuelve `retry-after-ms` |
| Cosmos update | warnings si el documento no existe o falla el replace |
| MCP | timeout configurable con `MCP_CLIENT_TIMEOUT` |
| FastAPI | captura de errores en el generador SSE y emite evento `error` |

### 5.3 Timeouts y limpieza

- `uvicorn` local usa `timeout_keep_alive=60`.
- MCP usa timeout desde App Config.
- NL2SQL borra agentes en background para evitar fugas.
- `SearchClient` y `CosmosDBClient` reutilizan conexiones para evitar overhead.

## 6. Motor de Procesamiento / Orquestación

### 6.1 Cómo funciona el engine interno

No existe un “workflow engine” externo. El engine es el conjunto de estas capas:

- FastAPI como dispatcher.
- `Orchestrator` como coordinador de sesión.
- `BaseAgentStrategy` como contrato.
- strategies concretas como implementación de ejecución.
- plugins y conectores como capacidades operativas.

### 6.2 Clases base disponibles para extender

| Clase base | Firma o contrato |
| --- | --- |
| `BaseAgentStrategy` | `async def initiate_agent_flow(self, user_message: str)` |
| `AppConfigClient` | `get`, `get_value`, `get_config_with_retry` |
| `IdentityManager` | `get_credential`, `get_aio_credential` |

### 6.3 Ciclo de vida de un componente

```text
Inicialización
  -> carga de config
  -> resolución de credenciales
  -> preparación de clientes singleton
Ejecución
  -> tool calls / retrieval / inference / streaming
Persistencia
  -> update de conversación
Cleanup
  -> delete de agentes temporales / cierre implícito de requests
```

### 6.4 Sistema de condiciones y routing

| Condición | Routing actual |
| --- | --- |
| `AGENT_STRATEGY=single_agent_rag` | V2 |
| `AGENT_STRATEGY=single_agent_rag_v1` | V1 |
| `AGENT_STRATEGY=mcp` | MCP |
| `AGENT_STRATEGY=nl2sql` | NL2SQL |
| Índice vacío en V2 y Bing deshabilitado | bypass a LLM directo |
| `type=feedback` | flujo de feedback en vez de Q&A |

### 6.5 Sistema de eventos / pub-sub

No hay bus pub/sub interno. Los eventos principales son:

- eventos de streaming de AgentsClient / AI Projects
- SSE hacia el cliente
- tareas asíncronas internas con `asyncio.create_task`

## 7. Modelo de Seguridad

### 7.1 Identidades utilizadas

| Identidad | Uso |
| --- | --- |
| Managed Identity del runtime | acceso a App Config, Search, Cosmos, Key Vault, Foundry |
| Azure CLI credential | fallback en desarrollo local |
| Bearer token del usuario final | autorización de usuario y trimming documental |
| Service principal de Fabric | acceso a SQL endpoint o semantic model, con secreto en Key Vault |

### 7.2 Asignaciones RBAC por servicio

| Identidad | Rol | Scope |
| --- | --- | --- |
| Container App MI | `App Configuration Data Reader` | App Configuration |
| Container App MI | `Search Index Data Reader` | Search service / index |
| Container App MI | acceso a secretos | Key Vault |
| Container App MI | `AcrPull` | Azure Container Registry |
| Usuario/provisionador | `Azure AI Account Owner` | suscripción |
| Usuario/provisionador | `Role Based Access Administrator` | suscripción o RG |

### 7.3 Flujo de autenticación entre servicios

```text
Cliente -> Orchestrator API
  -> auth técnica (API key o Dapr token)
  -> auth de usuario opcional (Bearer Entra ID)
  -> Orchestrator usa Managed Identity para Search/AppConfig/Cosmos/Foundry
  -> SearchClient puede intercambiar el token del usuario vía OBO
  -> Azure AI Search aplica trimming por ACL/RBAC
```

### 7.4 Secretos y configuración sensible

| Tipo de secreto | Gestión |
| --- | --- |
| claves de evaluación | Key Vault |
| secretos de datasources SQL/Fabric | Key Vault |
| variables operativas | App Configuration |
| credenciales locales | `.env` o variables de entorno en dev |

## 8. Comparación de Modos de Despliegue

### 8.1 Tabla comparativa

| Característica | Modo Básico | Modo Enterprise |
| --- | --- | --- |
| Hosting | Container Apps público | Container Apps endurecido en red corporativa |
| Identity | Managed Identity + acceso anónimo opcional | Managed Identity + Entra obligatorio |
| Search | sin ACL avanzada o filtros simples | ACL/RBAC trimming por token del usuario |
| Secrets | App Config + KV | KV + RBAC estricto + referencias centralizadas |
| Networking | público | Private Endpoints / segmentación según la solución mayor |
| Observabilidad | App Insights básico | alertas, dashboards, auditoría y trazas centralizadas |

### 8.2 Cambios por recurso Azure para enterprise

| Recurso | Cambio | Justificación |
| --- | --- | --- |
| Container Apps | restringir ingress y endurecer escalado | minimizar superficie expuesta |
| Azure AI Search | habilitar permission filter option y metadatos ACL/RBAC | seguridad documental |
| App Configuration | solo acceso por MI / RBAC | evitar secretos en texto plano |
| Key Vault | acceso mínimo por identidad | principio de mínimo privilegio |
| Cosmos DB | controles RBAC y monitoreo | protección del contexto conversacional |
| AI Foundry | despliegues controlados por entorno | separación dev/test/prod |

### 8.3 Topología ASCII enterprise

```text
Corp Users
   |
   v
WAF / Front Door
   |
   v
Container Apps Environment
   |
   +--> App Configuration
   +--> Key Vault
   +--> Azure AI Search
   +--> Cosmos DB
   +--> Azure AI Foundry
   +--> ACR
```

**Nota**

La topología enterprise completa no está materializada en Bicep dentro de este repo; se infiere del diseño de la solución GPT-RAG y de las necesidades del runtime local.

## 9. Patrones de Extensibilidad

### 9.1 Cómo añadir nuevos componentes / estrategias / plugins

1. Crear una clase nueva que herede de `BaseAgentStrategy`.
2. Implementar `initiate_agent_flow`.
3. Registrar el valor en `AgentStrategies`.
4. Conectar la nueva clase en `AgentStrategyFactory`.
5. Añadir prompts en `src/prompts/<namespace>/`.
6. Si se requieren datos o herramientas, crear un plugin o un conector nuevo.

### 9.2 Cómo registrar nuevos componentes en el catálogo real del repo

El repo no tiene un catálogo YAML central. El “registro” efectivo ocurre en tres puntos:

- `AgentStrategies` para estrategias.
- `AgentStrategyFactory` para resolución.
- `plugins/*` para herramientas expuestas a agentes.

### 9.3 Firmas exactas que deben implementarse

```python
class BaseAgentStrategy(ABC):
    @abstractmethod
    async def initiate_agent_flow(self, user_message: str):
        pass
```

```python
class AgentStrategyFactory:
    @staticmethod
    async def get_strategy(key: str):
        ...
```

### 9.4 Paso a paso para contribuir

1. Agregar o modificar código en `src/` respetando el patrón singleton y la configuración en App Config.
2. Si la capacidad necesita datos estructurados, agregar samples y catálogos en `samples/`.
3. Si requiere prompts, versionarlos en `src/prompts/`.
4. Actualizar `CHANGELOG.md`.
5. Probar localmente con FastAPI y, si aplica, con `evaluations/`.
6. Desplegar con `azd deploy` o scripts de deploy, nunca con `azd provision` desde este repo.

### 9.5 Anti-patrones a evitar

- Crear credenciales nuevas por request en lugar de usar `IdentityManager`.
- Persistir tokens del usuario en Cosmos DB.
- Saltarse Search antes de responder en prompts RAG.
- Permitir consultas que no sean `SELECT` en NL2SQL.
- Duplicar configuración en código cuando ya vive en App Configuration.