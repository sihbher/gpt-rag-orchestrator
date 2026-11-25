# GPT-RAG Orchestrator: Casos de Uso y Ejemplos

## Tabla de contenidos
- [1. Criterios de Selección de Casos de Uso](#1-criterios-de-selección-de-casos-de-uso)
- [2. Catálogo de Casos de Uso](#2-catálogo-de-casos-de-uso)
- [3. Tabla Resumen de Componentes por Caso de Uso](#3-tabla-resumen-de-componentes-por-caso-de-uso)
- [4. Índice de Samples Oficiales Incluidos en el Repositorio](#4-índice-de-samples-oficiales-incluidos-en-el-repositorio)
- [5. Combinaciones de Componentes Recomendadas](#5-combinaciones-de-componentes-recomendadas)

## 1. Criterios de Selección de Casos de Uso

### Qué tipos de problemas puede resolver este proyecto

GPT-RAG Orchestrator resuelve mejor problemas que encajan en uno de estos patrones:

- Preguntas sobre corpus documental indexado.
- Preguntas seguras donde la identidad del usuario condiciona qué documentos puede ver.
- Preguntas de negocio respondibles con SQL o DAX.
- Flujos que necesitan herramientas externas conectadas por MCP.
- Validación offline de calidad y seguridad de un asistente.

### Cómo mapear un caso de negocio a componentes del proyecto

| Si el problema es... | Estrategia recomendada | Componentes clave |
| --- | --- | --- |
| FAQ documental o knowledge base | `single_agent_rag` | `SearchClient.search_knowledge_base`, prompts RAG |
| RAG con seguridad documental | `single_agent_rag` | auth Entra + OBO + `x-ms-query-source-authorization` |
| Preguntas sobre tablas o KPIs | `nl2sql` | `GetAllDatasourcesInfo`, `TablesRetrieval`, `QueriesRetrieval`, `ExecuteSQLQuery` / `ExecuteDAXQuery` |
| Integración con herramientas o sistemas no modelados en el repo | `mcp` | `MCPSsePlugin`, servidor MCP externo |
| Evaluación masiva de respuestas | modo offline | `generate_eval_input.py`, `evaluate.py` |

### Nota sobre los YAML de este documento

Este repositorio no tiene un motor declarativo YAML nativo. Los YAML mostrados aquí son **blueprints operativos fieles al runtime**: usan IDs reales de estrategias, funciones y assets del repo, y describen cómo activar el comportamiento mediante App Configuration, seeds de datos y requests HTTP.

## 2. Catálogo de Casos de Uso

### Caso 1. FAQ corporativo sobre documentos internos

**Sector/Industria:** corporativo general

**Descripción del problema:** los empleados necesitan respuestas rápidas sobre políticas, manuales y documentos ya indexados.

**Solución con GPT-RAG Orchestrator:**

1. El cliente envía una pregunta al endpoint.
2. La estrategia `single_agent_rag` consulta primero el índice documental.
3. El agente sintetiza una respuesta con citas.
4. La conversación se guarda para continuidad futura.

**Componentes utilizados:**

- `single_agent_rag`
- `SearchClient.search_knowledge_base`
- `prompts/single_agent_rag/main.jinja2`

**Pipeline YAML completo:**

```yaml
name: faq-corporativo-rag
app_config:
  AGENT_STRATEGY: single_agent_rag
  SEARCH_RETRIEVAL_ENABLED: true
  BING_RETRIEVAL_ENABLED: false
request:
  ask: "Resume la política de trabajo remoto"
  conversation_id: null
runtime_steps:
  - search_knowledge_base
  - azure_ai_foundry_agent_response
  - persist_conversation
```

**Notas de implementación:**

- Es el caso más natural del repositorio.
- Requiere que el pipeline de ingestión del repo mayor ya haya poblado Azure AI Search.

---

### Caso 2. Salud: consulta segura de SOPs clínicos por identidad del usuario

**Sector/Industria:** salud

**Descripción del problema:** distintos perfiles clínicos no deben ver la misma documentación.

**Solución con GPT-RAG Orchestrator:**

1. El usuario llama al API con Bearer token Entra ID.
2. El orquestador valida identidad y puede obtener un token delegado para Azure AI Search.
3. Search trimmea resultados según ACL/RBAC del documento.
4. El agente responde solo con contenido autorizado.

**Componentes utilizados:**

- `single_agent_rag`
- `validate_access_token`
- `SearchClient._acquire_search_user_token_via_obo`
- `SearchClient.search_knowledge_base`

**Pipeline YAML completo:**

```yaml
name: salud-rag-con-acl
app_config:
  AGENT_STRATEGY: single_agent_rag
  ALLOW_ANONYMOUS: false
  SEARCH_RETRIEVAL_ENABLED: true
security:
  authorization_header: required
  query_time_header: x-ms-query-source-authorization
request:
  ask: "¿Cuál es el protocolo para dolor torácico?"
runtime_steps:
  - validate_access_token
  - acquire_search_user_token_via_obo
  - search_knowledge_base
  - azure_ai_foundry_agent_response
```

**Notas de implementación:**

- Este caso depende de que el índice haya sido preparado con metadatos de permisos.
- El repo soporta el runtime; la configuración del índice y la ingesta viven fuera de este repo.

---

### Caso 3. Legal: asistente de contratos con fallback a Bing

**Sector/Industria:** legal

**Descripción del problema:** el usuario necesita combinar cláusulas internas con contexto externo actualizado.

**Solución con GPT-RAG Orchestrator:**

1. Busca primero en la base documental.
2. Si no hay grounding suficiente, usa Bing Grounding.
3. Devuelve la respuesta con citas.

**Componentes utilizados:**

- `single_agent_rag`
- `search_knowledge_base`
- `BingGroundingTool`

**Pipeline YAML completo:**

```yaml
name: legal-rag-bing
app_config:
  AGENT_STRATEGY: single_agent_rag
  SEARCH_RETRIEVAL_ENABLED: true
  BING_RETRIEVAL_ENABLED: true
request:
  ask: "Compara nuestra cláusula de terminación con la regulación vigente"
runtime_steps:
  - search_knowledge_base
  - if_no_relevant_results: bing_grounding
  - respond_with_citations
```

**Notas de implementación:**

- El prompt del repo ya ordena esta secuencia.
- Requiere `BING_CONNECTION_ID` y proveedor `Microsoft.Bing` registrado.

---

### Caso 4. Retail: extracción de datos numéricos desde Azure SQL Database

**Sector/Industria:** retail

**Descripción del problema:** negocio necesita responder preguntas numéricas sobre productos, clientes y órdenes.

**Solución con GPT-RAG Orchestrator:**

1. `TriageAgent` identifica `adventureworks` como datasource.
2. Recupera tablas y queries similares desde Search.
3. Genera SQL de solo lectura.
4. Ejecuta SQL y sintetiza la respuesta.

**Componentes utilizados:**

- `nl2sql`
- `GetAllDatasourcesInfo`
- `TablesRetrieval`
- `QueriesRetrieval`
- `ValidateSQLQuery`
- `ExecuteSQLQuery`

**Pipeline YAML completo:**

```yaml
name: retail-nl2sql-azure-sql
app_config:
  AGENT_STRATEGY: nl2sql
seed_files:
  - samples/sql_database/datasources.json
  - samples/sql_database/tables/*.json
  - samples/sql_database/queries/*.json
request:
  ask: "¿Cuántos clientes compraron en el último mes?"
runtime_steps:
  - GetAllDatasourcesInfo
  - TablesRetrieval
  - QueriesRetrieval
  - ValidateSQLQuery
  - ExecuteSQLQuery
  - SyntetizerAgent
```

**Notas de implementación:**

- `ExecuteSQLQuery` bloquea sentencias que no empiecen con `SELECT`.
- Los samples `adventureworks_query_2` a `adventureworks_query_5` sirven como corpus de few-shot retrieval.

---

### Caso 5. Finanzas: KPIs ejecutivos sobre semantic model de Fabric

**Sector/Industria:** finanzas

**Descripción del problema:** se necesitan respuestas ejecutivas sobre métricas agregadas sin exponer SQL complejo al usuario.

**Solución con GPT-RAG Orchestrator:**

1. El flujo NL2SQL identifica un datasource de tipo `semantic_model`.
2. Recupera medidas y queries DAX similares.
3. Ejecuta DAX sobre Power BI/Fabric REST.
4. Devuelve un resumen en lenguaje natural.

**Componentes utilizados:**

- `nl2sql`
- `MeasuresRetrieval`
- `QueriesRetrieval`
- `ExecuteDAXQuery`

**Pipeline YAML completo:**

```yaml
name: finanzas-fabric-kpis
app_config:
  AGENT_STRATEGY: nl2sql
seed_files:
  - samples/fabric/datasources.json
  - samples/fabric/queries/*.json
request:
  ask: "¿Cuáles son las 5 ciudades con mayor utilidad?"
runtime_steps:
  - GetAllDatasourcesInfo
  - MeasuresRetrieval
  - QueriesRetrieval
  - ExecuteDAXQuery
  - SyntetizerAgent
```

**Notas de implementación:**

- Requiere token de usuario para DAX y un semantic model accesible.
- Los samples de Fabric ya muestran preguntas y DAX representativos.

---

### Caso 6. Manufactura: routing condicional entre conocimiento y datos estructurados

**Sector/Industria:** manufactura

**Descripción del problema:** algunas preguntas son de manuales y otras son de KPIs operacionales.

**Solución con GPT-RAG Orchestrator:**

1. Un caller externo decide la estrategia según el tipo de pregunta.
2. Si la pregunta es documental, usa `single_agent_rag`.
3. Si es tabular, usa `nl2sql`.

**Componentes utilizados:**

- `single_agent_rag`
- `nl2sql`
- endpoint único `/orchestrator`

**Pipeline YAML completo:**

```yaml
name: manufactura-routing-condicional
router:
  - when: ask_contains_metric_or_count
    use_strategy: nl2sql
  - when: default
    use_strategy: single_agent_rag
examples:
  - ask: "¿Cuál fue el OEE de la línea 2?"
    strategy: nl2sql
  - ask: "Explica el procedimiento de mantenimiento preventivo"
    strategy: single_agent_rag
```

**Notas de implementación:**

- El routing no está dentro del repo; lo hace el caller o la App Config por entorno.
- El repositorio sí soporta ambas estrategias con el mismo contrato HTTP.

---

### Caso 7. Seguros: procesamiento masivo de calidad y regresión

**Sector/Industria:** seguros

**Descripción del problema:** antes de poner el asistente en producción se necesita evaluar cientos de preguntas de regresión.

**Solución con GPT-RAG Orchestrator:**

1. Se carga un `golden-dataset.jsonl` con preguntas y ground truth.
2. `generate_eval_input.py` llama al orquestador y recoge contexto top-3 de Search.
3. `evaluate.py` sube el dataset a Azure AI Projects y ejecuta métricas.

**Componentes utilizados:**

- `generate_eval_input.py`
- `evaluate.py`
- `TestClient(app)`
- Azure AI Projects evaluators

**Pipeline YAML completo:**

```yaml
name: batch-evaluation
dataset:
  source: dataset/golden-dataset.jsonl
steps:
  - python evaluations/generate_eval_input.py
  - python evaluations/evaluate.py
metrics:
  - completeness
  - relevance
  - retrieval
  - safety
```

**Notas de implementación:**

- Es el único caso batch nativo del repo.
- El script `evaluate.sh --skip-eval` permite generar inputs sin lanzar la evaluación cloud.

---

### Caso 8. E-commerce: contenido online y enriquecimiento en tiempo real

**Sector/Industria:** retail / e-commerce

**Descripción del problema:** se necesita responder usando catálogo interno y señales web recientes.

**Solución con GPT-RAG Orchestrator:**

1. Busca primero en el knowledge base interno.
2. Usa Bing si no encuentra suficiente grounding.
3. Devuelve una respuesta enriquecida y citada.

**Componentes utilizados:**

- `single_agent_rag`
- `search_knowledge_base`
- `BingGroundingTool`

**Pipeline YAML completo:**

```yaml
name: ecommerce-rag-web-enrichment
app_config:
  AGENT_STRATEGY: single_agent_rag
  SEARCH_RETRIEVAL_ENABLED: true
  BING_RETRIEVAL_ENABLED: true
request:
  ask: "¿Qué diferencias hay entre nuestro producto y la tendencia actual del mercado?"
runtime_steps:
  - search_knowledge_base
  - if_no_relevant_results: bing_grounding
  - respond_with_citations
```

**Notas de implementación:**

- El repo no hace scraping nativo; el componente web nativo es Bing Grounding.
- Para scraping real, conviene combinar `mcp` con un servidor MCP especializado.

---

### Caso 9. Telco: fan-out/fan-in y paralelismo con herramientas MCP

**Sector/Industria:** telecomunicaciones

**Descripción del problema:** una pregunta necesita consultar múltiples backends al mismo tiempo y unir resultados.

**Solución con GPT-RAG Orchestrator:**

1. Se usa la estrategia `mcp`.
2. El servidor MCP publica varias herramientas externas.
3. El agente puede pedir varias consultas y luego sintetizarlas.

**Componentes utilizados:**

- `mcp`
- `MCPSsePlugin`
- `ChatCompletionAgent`

**Pipeline YAML completo:**

```yaml
name: telco-fanout-fanin-mcp
app_config:
  AGENT_STRATEGY: mcp
  MCP_APP_ENDPOINT: https://my-mcp-tools
request:
  ask: "Resume el estado de red, tickets abiertos y SLA del cliente Contoso"
runtime_steps:
  - connect_mcp_sse_plugin
  - invoke_parallel_external_tools
  - aggregate_results
  - respond
```

**Notas de implementación:**

- El paralelismo concreto vive en el servidor MCP, no en este repo.
- El orquestador sí ofrece el canal conversacional y la identidad contextual.

---

### Caso 10. Gobierno / investigación: comparación multi-documento y grafo de conocimiento

**Sector/Industria:** sector público / investigación

**Descripción del problema:** se necesitan comparar múltiples documentos y construir relaciones entre entidades.

**Solución con GPT-RAG Orchestrator:**

1. El caller usa `mcp`.
2. El servidor MCP expone herramientas como `compare_documents` o `build_knowledge_graph`.
3. El orquestador centraliza la conversación y el control de acceso.

**Componentes utilizados:**

- `mcp`
- `MCPSsePlugin`
- herramientas externas publicadas por el servidor MCP

**Pipeline YAML completo:**

```yaml
name: gobierno-comparacion-y-grafo
app_config:
  AGENT_STRATEGY: mcp
  MCP_APP_ENDPOINT: https://my-mcp-tools
request:
  ask: "Compara los expedientes A y B y genera un grafo de actores y relaciones"
runtime_steps:
  - connect_mcp_sse_plugin
  - compare_documents_tool
  - build_knowledge_graph_tool
  - synthesize_result
```

**Notas de implementación:**

- La capacidad es implementable con el repo, pero no es nativa; requiere un MCP server que exponga esas herramientas.
- Es el patrón correcto cuando el negocio pide capacidades fuera del núcleo RAG/NL2SQL.

## 3. Tabla Resumen de Componentes por Caso de Uso

| Caso de Uso | Componentes Requeridos | Servicios Azure | Complejidad |
| --- | --- | --- | --- |
| FAQ corporativo | `single_agent_rag`, `search_knowledge_base` | AI Search, AI Foundry, Cosmos | Baja |
| SOPs clínicos seguros | `single_agent_rag`, auth Entra, OBO Search | AI Search, Entra ID, AI Foundry, Cosmos | Media |
| Contratos + Bing | `single_agent_rag`, `BingGroundingTool` | AI Search, Bing, AI Foundry | Media |
| Azure SQL numérico | `nl2sql`, `ExecuteSQLQuery` | AI Search, SQL Database, AI Foundry, Cosmos | Media |
| KPIs Fabric | `nl2sql`, `ExecuteDAXQuery` | AI Search, Fabric / Power BI, AI Foundry | Media |
| Routing documental/tabular | caller externo + `single_agent_rag`/`nl2sql` | Search, SQL/Fabric, Foundry | Media |
| Evaluación batch | `generate_eval_input.py`, `evaluate.py` | Search, AI Projects, App Config, Key Vault | Media |
| Contenido online | `single_agent_rag`, Bing | Search, Bing, Foundry | Media |
| Fan-out/fan-in | `mcp`, MCP server externo | Container Apps, Foundry, sistemas externos | Alta |
| Comparación y grafo | `mcp`, tools externos | Container Apps, Foundry, backend gráfico externo | Alta |

## 4. Índice de Samples Oficiales Incluidos en el Repositorio

### Directorio `samples/fabric/`

| Nombre | Descripción en una oración | Componentes que demuestra | Ruta relativa |
| --- | --- | --- | --- |
| `datasources.json` | Define un SQL endpoint y un semantic model para Fabric | `nl2sql`, `ExecuteSQLQuery`, `ExecuteDAXQuery` | `samples/fabric/datasources.json` |
| `wwi_sales_aggregate_data_query_1.json` | Query DAX para top 5 empleados por ventas | `QueriesRetrieval`, `ExecuteDAXQuery` | `samples/fabric/queries/wwi_sales_aggregate_data_query_1.json` |
| `wwi_sales_aggregate_data_query_2.json` | Query DAX para top 5 ciudades por utilidad | `QueriesRetrieval`, `ExecuteDAXQuery` | `samples/fabric/queries/wwi_sales_aggregate_data_query_2.json` |
| `wwi_sales_aggregate_data_query_3.json` | Query DAX para ventas en New York durante 2025 | `QueriesRetrieval`, `ExecuteDAXQuery` | `samples/fabric/queries/wwi_sales_aggregate_data_query_3.json` |
| `tables/*.json` | Diccionario de tablas de star schema y agregados | `GetAllTablesInfo`, `GetSchemaInfo`, `TablesRetrieval` | `samples/fabric/tables/` |

### Directorio `samples/sql_database/`

| Nombre | Descripción en una oración | Componentes que demuestra | Ruta relativa |
| --- | --- | --- | --- |
| `datasources.json` | Define el datasource `adventureworks` | `GetAllDatasourcesInfo`, `ExecuteSQLQuery` | `samples/sql_database/datasources.json` |
| `adventureworks_query_1.json` | Ejemplo de top 5 productos más caros | `QueriesRetrieval`, `ExecuteSQLQuery` | `samples/sql_database/queries/adventureworks_query_1.json` |
| `adventureworks_query_2.json` | Cuenta clientes con órdenes recientes | `QueriesRetrieval`, `ExecuteSQLQuery` | `samples/sql_database/queries/adventureworks_query_2.json` |
| `adventureworks_query_3.json` | Une customer, orders y categories para una pregunta compleja | `QueriesRetrieval`, `ExecuteSQLQuery` | `samples/sql_database/queries/adventureworks_query_3.json` |
| `adventureworks_query_4.json` | Lista productos discontinuados | `QueriesRetrieval`, `ExecuteSQLQuery` | `samples/sql_database/queries/adventureworks_query_4.json` |
| `adventureworks_query_5.json` | Devuelve subtotal e impuestos por orden | `QueriesRetrieval`, `ExecuteSQLQuery` | `samples/sql_database/queries/adventureworks_query_5.json` |
| `tables/*.json` | Diccionario de Address, Customer, Product, ProductCategory, ProductDescription, ProductModel, SalesOrderDetail y SalesOrderHeader | `GetAllTablesInfo`, `GetSchemaInfo`, `TablesRetrieval` | `samples/sql_database/tables/` |

## 5. Combinaciones de Componentes Recomendadas

### Patrones frecuentes

| Patrón | Combinación recomendada | Cuándo usarlo |
| --- | --- | --- |
| RAG clásico | `single_agent_rag` + `search_knowledge_base` | knowledge bases documentales |
| RAG seguro | `single_agent_rag` + Entra + OBO Search | información con permisos por usuario |
| RAG con actualidad | `single_agent_rag` + Search + Bing | cuando el corpus interno puede no estar completo |
| NL2SQL robusto | `nl2sql` + `TablesRetrieval` + `QueriesRetrieval` + `ValidateSQLQuery` | preguntas tabulares de negocio |
| Integración avanzada | `mcp` + servidor MCP propio | sistemas externos, procesos complejos, tools custom |
| Evaluación continua | orquestador + `generate_eval_input.py` + `evaluate.py` | QA, regresión y benchmark |

### Anti-patrones a evitar

| Anti-patrón | Riesgo |
| --- | --- |
| Usar `nl2sql` sin poblar catálogos de tablas y queries | respuestas pobres o consultas inventadas |
| Habilitar acceso anónimo en escenarios sensibles | exposición de contenido no autorizado |
| Intentar batch productivo con el endpoint síncrono sin capa adicional | mala experiencia y falta de control operacional |
| Esperar scraping o grafo nativo del repo sin MCP | funcionalidad inexistente |
| Usar MCP para preguntas que ya resuelve Search o NL2SQL | complejidad innecesaria |
