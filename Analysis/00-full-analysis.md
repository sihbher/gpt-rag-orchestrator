# GPT-RAG Orchestrator: Análisis Completo

## Tabla de contenidos
- [1. ¿Qué es GPT-RAG Orchestrator?](#1-qué-es-gpt-rag-orchestrator)
- [2. Conceptos Clave](#2-conceptos-clave)
- [3. Capacidades Completas](#3-capacidades-completas)
- [4. Prerrequisitos](#4-prerrequisitos)
- [5. Arquitectura Final](#5-arquitectura-final)
- [6. Fuera de Alcance](#6-fuera-de-alcance)
- [7. Escenarios de Ejemplo](#7-escenarios-de-ejemplo)
- [8. Índice de Samples Incluidos](#8-índice-de-samples-incluidos)

## 1. ¿Qué es GPT-RAG Orchestrator?

**Descripción en una frase**

GPT-RAG Orchestrator es un servicio FastAPI que expone una API de preguntas y respuestas en tiempo real y coordina estrategias de RAG, MCP y NL2SQL sobre Azure AI Foundry, Azure AI Search y Cosmos DB.

**Propuesta de valor principal**

Su valor está en desacoplar la experiencia conversacional de la lógica de recuperación y razonamiento: la API mantiene contexto conversacional, selecciona una estrategia de ejecución, aplica seguridad documental y transmite respuestas por SSE con latencia optimizada.

**Repositorio, licencia y runtime**

| Campo | Valor |
| --- | --- |
| Nombre | `GPT-RAG Orchestrator` |
| Repositorio | `https://github.com/Azure/gpt-rag-orchestrator` |
| Solución mayor | Parte de `https://github.com/Azure/gpt-rag` |
| Licencia | MIT |
| Runtime principal | Python 3.12 |
| Framework API | FastAPI + Uvicorn |
| Modelo de ejecución | Servicio HTTP asincrónico con streaming SSE |
| Contenedor | Docker sobre `python:3.12-slim` |
| Puerto local por defecto | `9000` |
| Puerto en contenedor | `8080` |
| Nube objetivo | Azure Container Apps dentro de la solución GPT-RAG |

**Lecturas de referencia usadas en este análisis**

- README principal del repositorio.
- CHANGELOG del servicio.
- `requirements.txt`, `Dockerfile`, `azure.yaml`, scripts de despliegue y pipelines CI.
- Código fuente completo de `src/`.
- Assets de `samples/`, `dataset/` y `evaluations/`.
- Infraestructura local (`infra/main.bicep`) y la referencia explícita al repositorio padre `Azure/GPT-RAG` donde vive la infraestructura completa.

**Observación importante sobre el alcance del análisis**

Este repositorio no contiene un `AGENTS.md` local ni un catálogo tipo `executor_catalog.yaml`. El “catálogo” real del proyecto está distribuido entre:

- Estrategias registradas en `src/strategies/agent_strategies.py`.
- Plugins y funciones kernel en `src/plugins/`.
- Conectores Azure en `src/connectors/`.
- Prompts versionados en `src/prompts/`.
- Samples de esquemas y consultas en `samples/`.

## 2. Conceptos Clave

### 2.1 Orchestrator

El `Orchestrator` es la clase que representa la sesión de una conversación. Su responsabilidad es:

- Resolver o crear `conversation_id`.
- Cargar la conversación desde Cosmos DB.
- Instanciar la estrategia configurada por `AGENT_STRATEGY`.
- Inyectar `user_context` y, cuando aplica, el token del usuario.
- Emitir la respuesta en streaming.
- Persistir el estado actualizado de la conversación al final.

Ejemplo Python real del patrón:

```python
orchestrator = await Orchestrator.create(
    conversation_id=body.conversation_id,
    user_context=user_context,
    request_access_token=access_token if authorization else None,
)

async for chunk in orchestrator.stream_response(ask, question_id):
    yield chunk
```

### 2.2 Estrategia de agente

El proyecto usa Strategy Pattern para desacoplar la API del modo de razonamiento. Las estrategias disponibles son:

- `single_agent_rag`
- `single_agent_rag_v1`
- `mcp`
- `nl2sql`

La selección se hace por App Configuration:

```yaml
app_config:
  AGENT_STRATEGY: single_agent_rag
```

Relación entre conceptos:

- La API llama al `Orchestrator`.
- El `Orchestrator` resuelve una estrategia.
- La estrategia usa conectores y prompts.
- Los conectores resuelven acceso a Azure.
- Cosmos DB conserva el contexto conversacional.

### 2.3 Single Agent RAG v1 vs v2

Ambas implementaciones responden preguntas apoyándose en Azure AI Foundry y, opcionalmente, Azure AI Search y Bing Grounding, pero difieren en la arquitectura operativa.

| Estrategia | Enfoque | Rasgos distintivos |
| --- | --- | --- |
| `single_agent_rag_v1` | Azure AI Foundry Agents clásico | Crea o reutiliza agente, usa auto function calls, consolida historial al final |
| `single_agent_rag` | Alias a `SingleAgentRAGStrategyV2` | Optimizada para latencia, usa `AgentsClient`, prewarming, cache de agente, bypass local cuando el índice está vacío |

Ejemplo del bypass en V2:

```python
if is_empty and not self.cfg.get("BING_RETRIEVAL_ENABLED", False, type=bool):
    logging.info("[Agent Flow V2] ⚡ Routing locally directly to LLM")
```

### 2.4 Recuperación segura

La recuperación documental no es solo “buscar”. El proyecto implementa dos capas:

1. Autenticación del servicio hacia Azure AI Search con Managed Identity.
2. Trimming de resultados por identidad del usuario mediante `x-ms-query-source-authorization` cuando hay token del usuario y el índice soporta ACL/RBAC a nivel documento.

Según documentación oficial de Microsoft, Azure AI Search puede aplicar control de acceso a nivel documento usando ACLs o RBAC scopes y evaluar el token enviado en `x-ms-query-source-authorization` en tiempo de consulta.

Ejemplo conceptual de request saliente:

```yaml
search_request:
  endpoint: https://<search-service>.search.windows.net
  headers:
    Authorization: Bearer <managed-identity-token>
    x-ms-query-source-authorization: Bearer <user-token>
```

### 2.5 Contexto conversacional

La conversación se persiste en Cosmos DB. No se guarda el token del usuario, pero sí:

- `id`
- `thread_id`
- `agent_id`
- `messages`
- `questions`
- `feedback`
- `user_context`
- `prompt_tokens`
- `completion_tokens`

Ejemplo de documento esperado:

```json
{
  "id": "0e0ec765-8db2-4a4d-b0f5-88f63df4a111",
  "thread_id": "thread_123",
  "agent_id": "agent_456",
  "questions": [
    {
      "question_id": "q-1",
      "text": "¿Cuál fue el total de ventas del último trimestre?"
    }
  ],
  "messages": [
    {
      "role": "assistant",
      "text": "El total fue 1.2M USD."
    }
  ],
  "feedback": [],
  "user_context": {
    "principal_name": "user@contoso.com"
  }
}
```

### 2.6 MCP como puente de herramientas externas

La estrategia `mcp` conecta el orquestador con un servidor MCP por SSE (`/sse`). Esto convierte al servicio en un front-end conversacional para herramientas externas publicadas por otro proceso.

Ejemplo de configuración:

```yaml
app_config:
  AGENT_STRATEGY: mcp
  MCP_APP_ENDPOINT: https://my-mcp-service
  MCP_SERVER_TRANSPORT: sse
  MCP_CLIENT_TIMEOUT: 600
```

### 2.7 NL2SQL como patrón multi-agente

La estrategia `nl2sql` crea tres agentes:

- `TriageAgent`
- `SQLQueryAgent`
- `SyntetizerAgent`

El flujo:

1. Determina la fuente de datos relevante.
2. Busca tablas y queries similares.
3. Valida la consulta.
4. Ejecuta SQL o DAX.
5. Sintetiza una respuesta natural.

Blueprint YAML documental del flujo:

```yaml
strategy: nl2sql
agents:
  - TriageAgent
  - SQLQueryAgent
  - SyntetizerAgent
tools:
  - GetAllDatasourcesInfo
  - TablesRetrieval
  - QueriesRetrieval
  - ValidateSQLQuery
  - ExecuteSQLQuery
```

### 2.8 App Configuration como plano de control

Prácticamente toda la operación está gobernada por Azure App Configuration. El cliente carga etiquetas con prioridad:

1. `orchestrator`
2. `gpt-rag-orchestrator`
3. `gpt-rag`
4. `sin etiqueta`

Esto permite reutilizar infraestructura compartida y sobrescribir variables específicas del servicio.

### 2.9 Prewarming y singletons

El servicio optimiza cold start con:

- `IdentityManager` singleton.
- `SearchClient` singleton.
- `CosmosDBClient` singleton.
- `GenAIModelClient` singleton.
- `AgentsClient` singleton en V2.
- Precreación opcional de agente reutilizable.

Esto reduce latencia de autenticación, apertura de conexiones y creación de agentes.

## 3. Capacidades Completas

### 3.1 Capacidades de la API

| ID | Ruta / interfaz | Descripción | Inputs | Outputs | Configuración relevante |
| --- | --- | --- | --- | --- | --- |
| `orchestrator_endpoint` | `POST /orchestrator` | Punto de entrada principal para preguntas y feedback | `ask`, `question`, `conversation_id`, `type`, `user_context`, headers de auth | SSE para respuestas o JSON de feedback | `AGENT_STRATEGY`, auth headers, App Configuration |
| `feedback_mode` | `POST /orchestrator` con `type=feedback` | Guarda feedback asociado a conversación y pregunta | `conversation_id`, `question_id`, `is_positive`, `stars_rating`, `feedback_text` | JSON `{status,message}` | Cosmos DB container de conversaciones |

**Características API**

- Streaming en `text/event-stream`.
- Autenticación por `dapr-api-token` o `X-API-KEY`.
- Autenticación de usuario final opcional por Bearer token Entra ID.
- Soporte de acceso anónimo configurable con `ALLOW_ANONYMOUS`.
- Compatibilidad con correlación de conversación y `question_id`.
- Persistencia automática del contexto.

### 3.2 Estrategias soportadas

| ID | Clase | Descripción detallada | Inputs | Outputs | Configuración clave |
| --- | --- | --- | --- | --- | --- |
| `single_agent_rag` | `SingleAgentRAGStrategyV2` | Estrategia principal optimizada para latencia. Puede ir por bypass directo a LLM si el índice está vacío o usar AgentsClient con herramientas | pregunta del usuario, contexto conversacional, token opcional | stream de texto | `SEARCH_RETRIEVAL_ENABLED`, `BING_RETRIEVAL_ENABLED`, `AGENT_ID`, `CHAT_DEPLOYMENT_NAME` |
| `single_agent_rag_v1` | `SingleAgentRAGStrategyV1` | Implementación RAG previa basada en `project_client.agents` y auto function calls | igual que V2 | stream de texto | mismas variables base de Foundry/Search |
| `mcp` | `McpStrategy` | Conecta el orquestador a un servidor MCP remoto por SSE y expone sus herramientas dentro de un `ChatCompletionAgent` | pregunta, `user_context` | respuesta del agente | `MCP_APP_ENDPOINT`, `MCP_APP_APIKEY`, `MCP_CLIENT_TIMEOUT` |
| `nl2sql` | `NL2SQLStrategy` | Multi-agente para traducir lenguaje natural a SQL o DAX sobre catálogos preindexados | pregunta, metadata de datasources | stream final sintetizado | `DATASOURCES_CONTAINER`, índices de tablas, queries y medidas |

### 3.3 Conectores y clientes de infraestructura

| ID | Componente | Descripción | Inputs | Outputs | Configuración |
| --- | --- | --- | --- | --- | --- |
| `AppConfigClient` | `src/connectors/appconfig.py` | Carga configuración remota y Key Vault references desde Azure App Configuration | endpoint + credenciales | diccionario de configuración en memoria | `APP_CONFIG_ENDPOINT`, etiquetas |
| `IdentityManager` | `src/connectors/identity_manager.py` | Singleton de credenciales sync/async con Managed Identity y Azure CLI fallback | `AZURE_CLIENT_ID` opcional | credenciales reutilizables | `AZURE_CLIENT_ID` |
| `SearchClient` | `src/connectors/search.py` | Cliente REST para Azure AI Search con búsqueda híbrida, token OBO y cache de índice vacío | query, body, user token opcional | resultados JSON o string RAG | `SEARCH_SERVICE_QUERY_ENDPOINT`, `SEARCH_RAG_INDEX_NAME`, `SEARCH_APPROACH` |
| `GenAIModelClient` | `src/connectors/aifoundry.py` | Cliente unificado para chat/completions y embeddings sobre Foundry/Azure OpenAI | prompt o texto | completion o embeddings | endpoints y nombres de deployment |
| `CosmosDBClient` | `src/connectors/cosmosdb.py` | Persistencia de conversaciones y documentos auxiliares | container, key, body | documentos | `DATABASE_ACCOUNT_NAME`, `DATABASE_NAME` |
| `KeyVault` helper | `src/connectors/keyvault.py` | Recupera secretos y sanitiza nombres de secreto | secret name | secret value | `KEY_VAULT_URI` |
| `SQLDBClient` | `src/connectors/sqldbs.py` | Ejecuta queries sobre Azure SQL Database usando SQL auth o token AAD | datasource config | conexión pyodbc | server, database, uid opcional |
| `SQLEndpointClient` | `src/connectors/fabric.py` | Conecta a SQL Endpoint de Microsoft Fabric con service principal | datasource config | conexión pyodbc | tenant, client_id, server, database |
| `SemanticModelClient` | `src/connectors/fabric.py` | Ejecuta consultas DAX vía REST API de Power BI / Fabric semantic model | DAX, token de usuario | filas JSON | dataset, workspace, tenant, client_id |

### 3.4 Plugins y funciones kernel disponibles

#### Plugins comunes

| ID exacto | Plugin | Descripción | Inputs | Outputs |
| --- | --- | --- | --- | --- |
| `GetTodayDate` | `CommonPlugin` | Devuelve la fecha actual | ninguno | string |
| `GetTime` | `CommonPlugin` | Devuelve la hora actual | ninguno | string |

#### Plugin de retrieval legado

| ID exacto | Descripción | Inputs | Outputs | Notas |
| --- | --- | --- | --- | --- |
| `VectorIndexRetrieve` | Búsqueda vectorial/híbrida contra Azure AI Search | `input`, `security_ids` | `VectorIndexRetrievalResult` | Usa campo `contentVector` y filtro `metadata_security_id` |
| `MultimodalVectorIndexRetrieve` | Recupera texto, imágenes y captions relacionados | `input`, `security_ids` | `MultimodalVectorIndexRetrievalResult` | Apunta a `contentVector`, `captionVector`, `relatedImages` |
| `GetDataPointsFromChatLog` | Extrae datapoints desde un chat log | `chat_log` | `DataPointsResult` | Útil para auditoría / postprocesado |

#### Plugin NL2SQL

| ID exacto | Descripción | Inputs | Outputs |
| --- | --- | --- | --- |
| `GetAllDatasourcesInfo` | Lista datasources disponibles | ninguno | `DataSourcesList` |
| `GetAllTablesInfo` | Lista tablas por datasource | `datasource` | `TablesList` |
| `GetSchemaInfo` | Trae descripción y columnas de una tabla | `datasource`, `table_name` | `SchemaInfo` |
| `TablesRetrieval` | Recupera tablas relevantes vía Search híbrido | `input`, `datasource` opcional | `TablesRetrievalResult` |
| `MeasuresRetrieval` | Lista medidas de semantic models | `datasource` | `MeasuresList` |
| `QueriesRetrieval` | Recupera consultas similares guardadas | `input`, `datasource` opcional | `QueriesRetrievalResult` |
| `ValidateSQLQuery` | Valida sintaxis SQL con `sqlparse` | `query` | `ValidateSQLQueryResult` |
| `ExecuteDAXQuery` | Ejecuta DAX sobre semantic model | `datasource`, `query`, `access_token` | `ExecuteQueryResult` |
| `ExecuteSQLQuery` | Ejecuta SQL de solo lectura | `datasource`, `query` | `ExecuteQueryResult` |

### 3.5 Prompts incluidos

| Ruta lógica | Propósito | Observaciones |
| --- | --- | --- |
| `prompts/single_agent_rag/main.jinja2` | Instrucciones del asistente RAG | Fuerza idioma del usuario, búsqueda obligatoria y citas |
| `prompts/mcp/main.txt` | Instrucciones mínimas para agente con herramientas MCP | Ordena usar herramientas antes que conocimiento interno |
| `prompts/nl2sql/triage_agent.txt` | Selección de datasource | Decide si responde directo o deriva a SQL |
| `prompts/nl2sql/sqlquery_agent.txt` | Generación y ejecución de SQL | Obliga a usar `QueriesRetrieval` |
| `prompts/nl2sql/syntetizer_agent.txt` | Síntesis final | Reenvía la respuesta final y agrega `TERMINATE` |

### 3.6 Persistencia y estructuras auxiliares

| Artefacto | Descripción | Ubicación física |
| --- | --- | --- |
| Conversaciones | Historial conversacional, preguntas y feedback | Cosmos DB container `CONVERSATIONS_DATABASE_CONTAINER` |
| Datasources | Definiciones de SQL DB, Fabric SQL endpoint y semantic models | Cosmos DB container `DATASOURCES_CONTAINER` |
| Prompts | Contenido de prompts opcionalmente subido | Cosmos DB container `prompts` mediante `upload_prompts.py` |
| Índice RAG | Documentos y chunks de grounding | Azure AI Search, definido por App Config |
| Índices NL2SQL | tablas, medidas y queries similares | Azure AI Search |

### 3.7 Motor, pipeline o framework interno

Aunque el repositorio no implementa un DSL declarativo de pipelines, sí tiene un motor composicional claro:

1. **FastAPI** recibe el request.
2. **Dependencias** resuelven autenticación.
3. **Orchestrator** resuelve la estrategia.
4. **Estrategia** orquesta agentes, herramientas y contexto.
5. **Conectores** hablan con Search, Foundry, Cosmos, Key Vault o Fabric.
6. **StreamingResponse** emite el resultado.
7. **Persistencia asíncrona** guarda la conversación.

### 3.8 Características de UI

Este repositorio **no incluye UI**. El consumidor típico es:

- GPT-RAG UI del repositorio hermano.
- Un cliente HTTP propio.
- `TestClient` para evaluaciones.
- Scripts de prueba o herramientas internas.

### 3.9 Características de workers o procesos secundarios

| Proceso | Descripción |
| --- | --- |
| FastAPI runtime | Servicio principal que atiende `/orchestrator` |
| Warmup de `lifespan` | Precalienta credenciales, Search, Cosmos y AgentsClient |
| Persistencia en background | `asyncio.create_task` para guardar conversación sin bloquear la respuesta |
| Evaluación offline | `evaluations/generate_eval_input.py` + `evaluations/evaluate.py` |
| Despliegue | scripts `deploy.sh` y `deploy.ps1` |

### 3.10 Extensibilidad

La extensibilidad real del proyecto se apoya en estos puntos:

- Añadir una nueva estrategia e incorporarla en `AgentStrategyFactory`.
- Añadir nuevas funciones kernel a un plugin o exponer nuevas herramientas MCP.
- Agregar nuevos prompts en `src/prompts/<namespace>/`.
- Sembrar nuevos datasources y catálogos de tablas/queries en Cosmos/Search.
- Personalizar seguridad enviando `user_context` o Bearer tokens.

Ejemplo de registro de una estrategia nueva:

```python
class MyStrategy(BaseAgentStrategy):
    async def initiate_agent_flow(self, user_message: str):
        yield "respuesta"

if key == "my_strategy":
    return MyStrategy()
```

## 4. Prerrequisitos

### 4.1 Servicios cloud requeridos

La infraestructura completa se define en el repositorio padre `Azure/GPT-RAG`, pero este servicio asume al menos los siguientes recursos.

| Servicio Azure | Obligatorio | Propósito en este repo | SKU / nivel recomendado |
| --- | --- | --- | --- |
| Azure Container Apps | Sí para despliegue en Azure | Hospedaje del contenedor FastAPI | Consumption para dev; entorno dedicado/workload profiles para productivo |
| Azure AI Foundry / Azure AI Services | Sí | Agentes, chat completions, embeddings | Modelo chat y embeddings desplegados; capacidad según uso |
| Azure AI Search | Sí en RAG/NL2SQL | Índice RAG, tablas, medidas, queries similares, ACL trimming | Standard S1 como base; S2 o superior para alta concurrencia |
| Azure Cosmos DB for NoSQL | Sí | Persistencia de conversaciones y catálogos | Serverless o autoscale para dev; autoscale/provisioned para productivo |
| Azure App Configuration | Sí | Plano de control de variables y referencias a Key Vault | Standard |
| Azure Key Vault | Recomendado y casi implícito | Secretos de datasources y evaluaciones | Standard |
| Azure Container Registry | Sí para `deploy.sh` / `deploy.ps1` | Build y push de la imagen | Basic para dev; Standard para pipelines estables |
| Application Insights / Azure Monitor | Recomendado | Telemetría y trazas OpenTelemetry | Workspace-based Application Insights |
| Azure Storage / fuentes documentales | Dependencia de la solución mayor | Origen documental para el pipeline de ingestión del repo padre | Standard general purpose v2 |
| Microsoft Fabric / Power BI | Opcional | Casos NL2SQL sobre SQL Endpoint o semantic model | Según tenant/licenciamiento existente |
| Bing Grounding connection | Opcional | Fallback web en estrategias RAG | Requiere proveedor `Microsoft.Bing` |

### 4.2 Toolchain de desarrollo

| Herramienta | Versión mínima / usada | Evidencia en repo |
| --- | --- | --- |
| Python | 3.12 | `Dockerfile`, GitHub Actions |
| FastAPI | `0.115.12` | `requirements.txt` |
| Uvicorn | `0.34.1` | `requirements.txt` |
| Azure CLI | 2.64 o superior recomendado por Microsoft para Foundry | README y documentación oficial |
| Azure Developer CLI | Opcional, recomendado para deploy | README, `azure.yaml` |
| Docker | Requerido para build local | scripts de deploy |
| PowerShell | 5.1+ o Core para script Windows | comentario en `deploy.ps1` |
| Bash | requerido para scripts POSIX | `deploy.sh`, `evaluate.sh` |
| ODBC Driver 18 for SQL Server | requerido para NL2SQL SQL/Fabric | `Dockerfile` |

### 4.3 Dependencias Python destacadas

| Paquete | Versión |
| --- | --- |
| `azure-ai-projects` | `2.0.0b3` |
| `azure-ai-agents` | `>=1.2.0b4` |
| `azure-search-documents` | `11.7.0b1` |
| `semantic-kernel[mcp]` | `1.34.0` |
| `openai` | `2.9.0` |
| `azure-identity` | `1.21.0` |
| `azure-cosmos` | `4.5.1` |
| `fastapi` | `0.115.12` |
| `pydantic` | `2.11.4` |
| `aiohttp` | `3.13.3` |

### 4.4 Permisos y roles necesarios

| Identidad | Rol / permiso | Scope típico | Motivo |
| --- | --- | --- | --- |
| Usuario que provisiona la solución | `Azure AI Account Owner` | suscripción | Crear/configurar recursos Foundry |
| Usuario que provisiona la solución | `Role Based Access Administrator` o `Owner` | suscripción o RG | Asignar RBAC a Search, Cosmos, Storage |
| Managed Identity del orquestador | `App Configuration Data Reader` | App Configuration | Leer claves y referencias |
| Managed Identity del orquestador | `Search Index Data Reader` | Search service / índices | Leer índices RAG y NL2SQL |
| Managed Identity del orquestador | `Search Index Data Contributor` | solo si escribe índices | No requerido por este repo de runtime puro |
| Managed Identity del orquestador | permisos de lectura de secretos | Key Vault | Resolver secretos y credenciales |
| Managed Identity del Container App | `AcrPull` | ACR | Ejecutar imagen en ACA |
| Identidad de despliegue | `AcrPush` / login ACR | ACR | `docker push` o `az acr build` |
| Cliente final de búsqueda segura | `Search Index Data Reader` + token Entra | Search | Query-time ACL/RBAC trimming |

### 4.5 Proveedores Azure a registrar

Microsoft documenta como prerequisito de escenarios Foundry y BYO resources registrar al menos:

- `Microsoft.KeyVault`
- `Microsoft.CognitiveServices`
- `Microsoft.Storage`
- `Microsoft.MachineLearningServices`
- `Microsoft.Search`
- `Microsoft.App`
- `Microsoft.ContainerService`
- `Microsoft.Bing` si se usa Grounding with Bing Search

### 4.6 Cuotas y límites relevantes

| Área | Límite / consideración |
| --- | --- |
| Modelos chat | La capacidad depende de la cuota del deployment en Azure AI Foundry / Azure AI Services |
| Embeddings | El cliente local trunca o reescribe si el input excede `EMBEDDING_MODEL_INPUT_TOKENS` (default `8192`) |
| Chat input | `CHAT_INPUT_TOKENS` default `128000` en el cliente unificado |
| Search Top K RAG | default `3` en varias rutas |
| Search ACL/RBAC document-level | Requiere preview APIs/SDK y envío de `x-ms-query-source-authorization` |
| Vector dimensions | El script `tests/rbac.ps1` asume `3072` dimensiones en campos vectoriales |
| NL2SQL | Solo permite `SELECT` en `ExecuteSQLQuery` |
| Evaluación | Requiere dataset `golden-dataset.jsonl` y acceso a Azure AI Projects |

## 5. Arquitectura Final

### 5.1 Diagrama de componentes en ASCII art

```text
            +----------------------+
            |  Cliente / UI / SDK  |
            +----------+-----------+
                       |
                 HTTPS POST /orchestrator
                       |
                       v
          +------------+-------------+
          |  FastAPI GPT-RAG         |
          |  Orchestrator Service    |
          +------------+-------------+
                       |
          +------------+------------+-------------------+
          |                         |                   |
          v                         v                   v
 +--------+--------+     +---------+---------+ +-------+--------+
 | App Configuration|     | Cosmos DB         | | Application    |
 | + Key Vault refs |     | conversations      | | Insights / OTEL|
 +--------+--------+     +--------------------+ +----------------+
          |
          |
          v
 +--------+------------------------------------------------------+
 | Strategy Layer                                                 |
 | - single_agent_rag (V2)                                        |
 | - single_agent_rag_v1                                          |
 | - mcp                                                          |
 | - nl2sql                                                       |
 +--------+-----------------------+-------------------------------+
          |                       |
          |                       |
          v                       v
 +--------+--------+     +--------+------------------------------+
 | Azure AI Search |     | Azure AI Foundry / Azure OpenAI       |
 | RAG + ACL/RBAC  |     | Agents, chat completions, embeddings  |
 +--------+--------+     +--------+------------------------------+
          |                       |
          |                       +------------------------------+
          |                                                      |
          v                                                      v
 +--------+---------+                                  +---------+--------+
 | MCP Server / SSE |                                  | Fabric / SQL DB  |
 | external tools   |                                  | via NL2SQL       |
 +------------------+                                  +------------------+
```

### 5.2 Mapa de comunicación entre servicios

| Origen | Destino | Puerto / protocolo | Cuándo ocurre | Qué transporta |
| --- | --- | --- | --- | --- |
| Cliente | FastAPI `/orchestrator` | HTTPS / HTTP local | siempre | pregunta, feedback, headers de auth |
| FastAPI | Azure App Configuration | HTTPS 443 | startup y lecturas de config | claves de configuración |
| FastAPI | Key Vault | HTTPS 443 | resolución de secretos | secretos de datasources o evaluaciones |
| FastAPI | Azure AI Search | HTTPS 443 REST | RAG, NL2SQL, ACL check, is_index_empty | queries y resultados |
| FastAPI | Azure AI Foundry Agent Service | HTTPS 443 | estrategias RAG y NL2SQL | threads, agents, runs, completions |
| FastAPI | Azure OpenAI-compatible endpoint | HTTPS 443 | embeddings | vectores |
| FastAPI | Cosmos DB | HTTPS 443 | carga y persistencia de conversación | documentos JSON |
| FastAPI | MCP server | SSE / HTTP(S) | estrategia `mcp` | tool calls y respuestas |
| FastAPI | Fabric / Power BI REST | HTTPS 443 | DAX en semantic model | query results |
| FastAPI | SQL Database / Fabric SQL Endpoint | ODBC / SQL | `ExecuteSQLQuery` | filas SQL |

### 5.3 Flujo de datos extremo a extremo

1. El cliente llama `POST /orchestrator` con una pregunta.
2. `validate_auth` valida `dapr-api-token` o `X-API-KEY`.
3. Si viene `Authorization`, se valida el token Entra ID y se resuelve el usuario.
4. El `Orchestrator` crea o carga la conversación desde Cosmos DB.
5. Se selecciona la estrategia configurada en `AGENT_STRATEGY`.
6. La estrategia usa prompts, herramientas y conectores para responder.
7. Si hay búsqueda segura, `SearchClient` intenta OBO para obtener un token delegado de Search.
8. Azure AI Search devuelve chunks o resultados NL2SQL según el caso.
9. Azure AI Foundry produce la respuesta final o coordina herramientas adicionales.
10. La API transmite chunks al cliente por SSE.
11. La conversación actualizada se persiste en background en Cosmos DB.

### 5.4 Ciclo de vida de una tarea

```text
Trigger HTTP
  -> Auth técnica
  -> Auth de usuario opcional
  -> Load/Create conversation
  -> Resolve strategy
  -> Prepare request context
  -> Retrieval / tool calls / model inference
  -> Stream response chunks
  -> Persist conversation asynchronously
  -> End request
```

### 5.5 Esquema de base de datos y documentos

#### Container de conversaciones

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `id` | string | `conversation_id`, también partition key lógica en las operaciones del repo |
| `thread_id` | string | Thread del servicio de agentes |
| `agent_id` | string | Agente reutilizado o temporal |
| `messages` | array | Historial consolidado |
| `questions` | array | Preguntas con `question_id` |
| `feedback` | array | Feedback del usuario |
| `user_context` | object | contexto de identidad o negocio |
| `prompt_tokens` | int | uso de tokens prompt |
| `completion_tokens` | int | uso de tokens completion |

#### Container de datasources

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `id` | string | nombre lógico del datasource |
| `description` | string | descripción operativa |
| `type` | string | `sql_database`, `sql_endpoint`, `semantic_model` |
| `server` | string | servidor SQL / endpoint |
| `database` | string | base o lakehouse |
| `dataset` | string | semantic model dataset |
| `tenant_id` | string | tenant del SP o entorno |
| `client_id` | string | app registration usada para Fabric |

#### Índice RAG esperado

El código muestra que los documentos de Search suelen exponer estos campos:

- `title`
- `content`
- `url`
- `filepath`
- `chunk_id`
- `contentVector`
- opcionalmente metadatos de permisos como `userIds`, `groupIds`, `rbacScope` o campos equivalentes preservados por el pipeline de ingestión

### 5.6 Modos de despliegue comparados

| Característica | Local | Básico Azure | Enterprise recomendado |
| --- | --- | --- | --- |
| Hosting | `uvicorn` local | Azure Container Apps público | Azure Container Apps con hardening y red privada |
| Exposición | localhost | endpoint público HTTPS | ingress restringido, WAF o private ingress según arquitectura mayor |
| Auth usuario | opcional | opcional | Entra ID obligatorio, `ALLOW_ANONYMOUS=false` |
| Search security | sin ACL o entorno dev | security filters o ACL preview | ACL/RBAC trimming con `x-ms-query-source-authorization` |
| Configuración | `.env` o App Config | App Config | App Config + Key Vault + RBAC estricta |
| Observabilidad | logs locales | App Insights básico | App Insights + trazas + alertas + auditoría |
| Contención de secretos | variables locales | Key Vault references | Key Vault + MI + rotación y revisiones |
| Escalado | manual | autoscale básico | perfiles dedicados, capacidad ajustada y pruebas de carga |

### 5.7 Modelo de seguridad

**Autenticación técnica de la API**

- `dapr-api-token` si se usa Dapr u otro caller interno.
- `X-API-KEY` como alternativa.

**Autenticación de usuario final**

- Bearer token Entra ID.
- Validación de `tenant_id`, `audience`, issuer y claves JWKS.
- Permite listas de usuarios/IDs autorizados mediante App Config.

**Autorización documental**

- Si el usuario envía token y el entorno está configurado, `SearchClient` intenta OBO.
- El token delegado de Search se pasa como `x-ms-query-source-authorization`.
- Azure AI Search puede aplicar trimming por `userIds`, `groupIds` y `rbacScope` cuando el índice fue configurado para ello.

**Secretos**

- No se hardcodean secretos en código.
- Los scripts y conectores esperan secretos en Key Vault o App Config.

**Persistencia segura**

- El token de usuario no se persiste en Cosmos.
- Solo se conserva `user_context` no sensible.

## 6. Fuera de Alcance

| Gap | Descripción | Esfuerzo estimado | Librerías / servicios sugeridos |
| --- | --- | --- | --- |
| UI nativa | El repo no incluye frontend | Bajo si se reutiliza `gpt-rag-ui`; medio si se crea uno nuevo | React, Next.js, `fetch-event-source` |
| Ingesta documental | No indexa documentos por sí mismo | Medio/alto | repo padre `gpt-rag-ingestion`, Azure Functions, Data Factory |
| Provisionamiento completo | `azd provision` está explícitamente bloqueado aquí | Bajo si se usa repo padre; alto si se rehace | Bicep, azd, Terraform |
| DSL YAML nativo | No existe motor declarativo de pipelines YAML en este repo | Medio | Pydantic config loaders, DAG engine, Temporal, Prefect |
| Jobs batch nativos | No hay cola, cron ni worker dedicado para procesamiento masivo | Medio | Container Apps Jobs, Azure Functions, Durable Functions |
| Escritura en sistemas transaccionales | NL2SQL restringe a `SELECT` | Bajo/medio | capa de comandos separada, approval workflow, SQLAlchemy |
| Memoria distribuida / cache cross-instance | Los singletons son intra-proceso | Medio | Azure Cache for Redis |
| Gobierno multi-tenant fuerte | El aislamiento es por configuración, no por tenancy hard multi-tenant | Alto | tenancy-aware routing, claims-based partitioning |
| Human-in-the-loop | No hay aprobaciones humanas ni colas de revisión | Medio | Logic Apps, Durable Functions, Power Automate |
| Knowledge graph nativo | No genera grafos de conocimiento de forma interna | Medio/alto | Neo4j, Cosmos DB Gremlin, GraphRAG pipelines |
| Web scraping nativo | El repo solo contempla Bing Grounding o MCP; no scraper propio | Medio | Playwright, Scrapy, MCP tools |

## 7. Escenarios de Ejemplo

**Nota importante**

El proyecto no expone un motor declarativo YAML. Para mantener fidelidad técnica, los siguientes ejemplos usan YAML como **plantilla operacional/documental** que mapea exactamente a capacidades reales del repo: `AGENT_STRATEGY`, plugins, datasources, prompts y payloads HTTP.

### 7.1 Finanzas: asistente de políticas y reportes internos

```yaml
name: finanzas-rag-interno
strategy: single_agent_rag
app_config:
  AGENT_STRATEGY: single_agent_rag
  SEARCH_RETRIEVAL_ENABLED: true
  BING_RETRIEVAL_ENABLED: false
request:
  ask: "Resume la política de cierre financiero mensual"
  conversation_id: null
components:
  - SearchClient.search_knowledge_base
  - SingleAgentRAGStrategyV2
result:
  mode: sse
  citations: required
```

### 7.2 Salud: consulta segura de documentación clínica interna

```yaml
name: salud-rag-con-acl
strategy: single_agent_rag
security:
  allow_anonymous: false
  user_token: required
search:
  permission_filter: enabled
  query_header: x-ms-query-source-authorization
request:
  ask: "¿Cuál es el protocolo de triage para dolor torácico?"
components:
  - validate_access_token
  - SearchClient._acquire_search_user_token_via_obo
  - SearchClient.search_knowledge_base
```

### 7.3 Legal: búsqueda de cláusulas y fallback web controlado

```yaml
name: legal-rag-plus-bing
strategy: single_agent_rag
app_config:
  SEARCH_RETRIEVAL_ENABLED: true
  BING_RETRIEVAL_ENABLED: true
request:
  ask: "Compara la cláusula de terminación con la regulación vigente"
components:
  - search_knowledge_base
  - BingGroundingTool
  - citation_postprocessing
```

### 7.4 Manufactura: FAQs técnicas con bypass de índice vacío

```yaml
name: manufactura-empty-index-bypass
strategy: single_agent_rag
precondition:
  search_index_state: empty
request:
  ask: "¿Qué es mantenimiento preventivo?"
components:
  - SearchClient.is_index_empty
  - GenAIModelClient.get_completion
expected_behavior:
  bypass_agents_backend: true
```

### 7.5 Retail: ventas sobre Azure SQL Database

```yaml
name: retail-nl2sql-sqldb
strategy: nl2sql
datasource: adventureworks
seed_files:
  - samples/sql_database/datasources.json
  - samples/sql_database/tables/*.json
  - samples/sql_database/queries/*.json
request:
  ask: "¿Cuáles son los 5 productos más caros disponibles?"
components:
  - GetAllDatasourcesInfo
  - TablesRetrieval
  - QueriesRetrieval
  - ValidateSQLQuery
  - ExecuteSQLQuery
```

### 7.6 Retail/Fabric: KPIs ejecutivos sobre semantic model

```yaml
name: retail-fabric-dax
strategy: nl2sql
datasource: wwi-sales-aggregated-data
request:
  ask: "¿Cuáles son las 5 ciudades con mayor utilidad?"
components:
  - GetAllDatasourcesInfo
  - MeasuresRetrieval
  - QueriesRetrieval
  - ExecuteDAXQuery
```

### 7.7 Sector público: asistente con herramientas especializadas vía MCP

```yaml
name: sector-publico-mcp
strategy: mcp
app_config:
  AGENT_STRATEGY: mcp
  MCP_APP_ENDPOINT: https://my-mcp-tools
request:
  ask: "Busca el expediente 2024-018 y resume sus hallazgos"
components:
  - McpStrategy
  - MCPSsePlugin
  - external_tools
```

### 7.8 Energía: procesamiento condicional con rutas diferentes

```yaml
name: energia-routing-condicional
strategy_router:
  if: pregunta estructurada sobre datos
  then: nl2sql
  else: single_agent_rag
examples:
  - ask: "¿Cuánto generó la planta A en marzo?"
  - ask: "Explica el procedimiento de mantenimiento en turbinas"
```

### 7.9 Seguros: comparación multi-documento con MCP

```yaml
name: seguros-comparacion-multidocumento
strategy: mcp
request:
  ask: "Compara la póliza A con la póliza B y lista diferencias"
components:
  - external_compare_documents_tool
  - ChatCompletionAgent
notes:
  requires_custom_mcp_server: true
```

### 7.10 Operaciones: evaluación masiva del asistente

```yaml
name: evaluacion-batch
mode: offline
dataset:
  input: dataset/golden-dataset.jsonl
steps:
  - python evaluations/generate_eval_input.py
  - python evaluations/evaluate.py
metrics:
  - completeness
  - relevance
  - retrieval
  - safety
```

## 8. Índice de Samples Incluidos

### 8.1 Samples de Fabric

| Sample | Descripción | Ruta |
| --- | --- | --- |
| `fabric/datasources.json` | Define dos datasources de ejemplo: SQL endpoint y semantic model | `samples/fabric/datasources.json` |
| `wwi_sales_aggregate_data_query_1.json` | Top 5 empleados por ventas con impuesto | `samples/fabric/queries/wwi_sales_aggregate_data_query_1.json` |
| `wwi_sales_aggregate_data_query_2.json` | Top 5 ciudades por utilidad | `samples/fabric/queries/wwi_sales_aggregate_data_query_2.json` |
| `wwi_sales_aggregate_data_query_3.json` | Ventas totales en New York durante 2025 | `samples/fabric/queries/wwi_sales_aggregate_data_query_3.json` |
| `aggregate_sale_by_date_city` | Tabla agregada por ciudad | `samples/fabric/tables/wwi_sales_aggregated_data-aggregate_sale_by_date_city.json` |
| `aggregate_sale_by_date_employee_` | Tabla agregada por empleado | `samples/fabric/tables/wwi_sales_aggregated_data-aggregate_sale_by_date_employee_.json` |
| `dimension_city` | Dimensión de ciudades | `samples/fabric/tables/wwi_sales_star_schema-dimension_city.json` |
| `dimension_customer` | Dimensión de clientes | `samples/fabric/tables/wwi_sales_star_schema-dimension_customer.json` |
| `dimension_date` | Dimensión de fechas | `samples/fabric/tables/wwi_sales_star_schema-dimension_date.json` |
| `dimension_employee` | Dimensión de empleados | `samples/fabric/tables/wwi_sales_star_schema-dimension_employee.json` |
| `dimension_stock_item` | Dimensión de productos | `samples/fabric/tables/wwi_sales_star_schema-dimension_stock_item.json` |
| `fact_sale` | Hecho de ventas detalladas | `samples/fabric/tables/wwi_sales_star_schema-fact_sale.json` |

### 8.2 Samples de SQL Database

| Sample | Descripción | Ruta |
| --- | --- | --- |
| `sql_database/datasources.json` | Define el datasource `adventureworks` | `samples/sql_database/datasources.json` |
| `adventureworks_query_1.json` | Top 5 productos más caros | `samples/sql_database/queries/adventureworks_query_1.json` |
| `adventureworks_query_2.json` | Clientes con órdenes el último mes | `samples/sql_database/queries/adventureworks_query_2.json` |
| `adventureworks_query_3.json` | Clientes que compraron en categoría Bikes | `samples/sql_database/queries/adventureworks_query_3.json` |
| `adventureworks_query_4.json` | Productos discontinuados | `samples/sql_database/queries/adventureworks_query_4.json` |
| `adventureworks_query_5.json` | Ventas y taxes del último año por orden | `samples/sql_database/queries/adventureworks_query_5.json` |
| `saleslt_address` | Diccionario de la tabla de direcciones | `samples/sql_database/tables/adventureworks-saleslt_address.json` |
| `saleslt_customer` | Diccionario de clientes | `samples/sql_database/tables/adventureworks-saleslt_customer.json` |
| `saleslt_product` | Diccionario de productos | `samples/sql_database/tables/adventureworks-saleslt_product.json` |
| `saleslt_productcategory` | Diccionario de categorías | `samples/sql_database/tables/adventureworks-saleslt_productcategory.json` |
| `saleslt_productdescription` | Diccionario de descripciones | `samples/sql_database/tables/adventureworks-saleslt_productdescription.json` |
| `saleslt_productmodel` | Diccionario de modelos | `samples/sql_database/tables/adventureworks-saleslt_productmodel.json` |
| `saleslt_salesorderdetail` | Diccionario de detalles de orden | `samples/sql_database/tables/adventureworks-saleslt_salesorderdetail.json` |
| `saleslt_salesorderheader` | Diccionario de encabezados de orden | `samples/sql_database/tables/adventureworks-saleslt_salesorderheader.json` |

### 8.3 Dataset y evaluación

| Asset | Descripción | Ruta |
| --- | --- | --- |
| `golden-dataset.jsonl` | Dataset base de ground truth para evaluación | `dataset/golden-dataset.jsonl` |
| `generate_eval_input.py` | Genera `eval-input.jsonl` llamando al orquestador y a Search | `evaluations/generate_eval_input.py` |
| `evaluate.py` | Envía evaluación a Azure AI Projects | `evaluations/evaluate.py` |
| `evaluate.sh` | Orquesta venv, generación de input y evaluación en POSIX | `evaluations/evaluate.sh` |
| `evaluate.ps1` | Equivalente PowerShell | `evaluations/evaluate.ps1` |

---

Este documento puede usarse como contexto único de un LLM para responder preguntas técnicas sobre el servicio, con una salvedad explícita: la infraestructura completa de ingestión, red y despliegue global de GPT-RAG vive en el repositorio padre `Azure/GPT-RAG`, no en este repositorio específico del orquestador.