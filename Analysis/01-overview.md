# GPT-RAG Orchestrator: Guía General de la Solución

## Tabla de contenidos
- [1. ¿Qué es GPT-RAG Orchestrator?](#1-qué-es-gpt-rag-orchestrator)
- [2. ¿Qué problema resuelve?](#2-qué-problema-resuelve)
- [3. Conceptos clave](#3-conceptos-clave)
- [4. Componentes principales](#4-componentes-principales)
- [5. ¿Cómo funciona el flujo completo?](#5-cómo-funciona-el-flujo-completo)
- [6. Servicios de Azure utilizados](#6-servicios-de-azure-utilizados)
- [7. Modos de despliegue](#7-modos-de-despliegue)
- [8. Ejemplo práctico](#8-ejemplo-práctico)
- [9. Resumen ejecutivo](#9-resumen-ejecutivo)

## 1. ¿Qué es GPT-RAG Orchestrator?

Si la solución GPT-RAG fuera una fábrica, este repositorio sería el jefe de planta que recibe la pregunta, decide qué línea de producción usar y coordina a los especialistas correctos para responder.

**Propuesta de valor en una frase**

GPT-RAG Orchestrator convierte una pregunta en una respuesta segura y contextualizada coordinando búsqueda, agentes de IA, fuentes de datos y contexto conversacional en un único endpoint.

## 2. ¿Qué problema resuelve?

| Desafío común | Cómo lo resuelve el proyecto |
| --- | --- |
| La información está dispersa entre documentos, índices y bases de datos | Centraliza el acceso en una API y varias estrategias de ejecución |
| Una sola técnica no sirve para todos los casos | Cambia entre RAG documental, MCP y NL2SQL según la configuración |
| Los usuarios necesitan respuestas con contexto y continuidad | Guarda la conversación en Cosmos DB y reutiliza el historial |
| La seguridad documental no puede quedar solo del lado del frontend | Propaga identidad y puede usar trimming de Azure AI Search a nivel documento |
| Las respuestas deben llegar rápido | Precalienta clientes, reutiliza conexiones y evita llamadas innecesarias |
| Se necesitan integraciones con herramientas externas | Permite conectar un servidor MCP por SSE |
| Las preguntas de negocio requieren datos tabulares | Incluye una estrategia NL2SQL con consultas SQL y DAX |
| Hay que medir calidad antes de producir | Incluye scripts de evaluación con Azure AI Projects |

## 3. Conceptos clave

### Orquestador

Es la capa que recibe la pregunta, carga la conversación y elige cómo responder.

```yaml
request:
  ask: "¿Cuál fue el total de ventas?"
  conversation_id: null
```

### Estrategia

Es el modo de trabajo del sistema. Las principales son:

- `single_agent_rag`: respuesta basada en documentos e IA.
- `mcp`: respuesta apoyada en herramientas externas.
- `nl2sql`: respuesta basada en consultas a datos estructurados.

```yaml
app_config:
  AGENT_STRATEGY: nl2sql
```

### Conversación

Cada interacción puede pertenecer a una sesión persistida para conservar contexto, preguntas anteriores y feedback.

### Recuperación segura

Cuando el usuario se autentica con Microsoft Entra ID, el servicio puede propagar esa identidad a Azure AI Search para que solo devuelva documentos autorizados.

### Diagrama ASCII de conceptos

```text
Pregunta del usuario
        |
        v
  Orchestrator API
        |
        +--> Estrategia RAG ---> Azure AI Search + Azure AI Foundry
        |
        +--> Estrategia MCP ---> MCP Server externo
        |
        +--> Estrategia NL2SQL -> SQL / Fabric
        |
        v
   Cosmos DB (contexto)
```

## 4. Componentes principales

| Componente | Propósito | Tecnologías | Puerto |
| --- | --- | --- | --- |
| API Orchestrator | Punto de entrada único para preguntas y feedback | FastAPI, Uvicorn | `9000` local / `8080` contenedor |
| Capa de estrategias | Decide cómo responder según el caso | Python, Azure AI Foundry Agents, Semantic Kernel | n/a |
| Capa de recuperación | Consulta índices y aplica seguridad documental | Azure AI Search REST, Managed Identity | HTTPS 443 |
| Persistencia conversacional | Guarda contexto, preguntas y feedback | Azure Cosmos DB | HTTPS 443 |
| Configuración y secretos | Centraliza parámetros y secretos | Azure App Configuration, Azure Key Vault | HTTPS 443 |
| Integraciones de datos | Responde sobre SQL Database, Fabric SQL Endpoint y semantic models | pyodbc, Power BI REST, Semantic Kernel | SQL/HTTPS |

### Interconexión de alto nivel

```text
Cliente
  |
  v
FastAPI Orchestrator
  |----> App Configuration / Key Vault
  |----> Cosmos DB
  |----> Azure AI Search
  |----> Azure AI Foundry
  |----> MCP Server opcional
  |----> SQL Database / Fabric opcional
```

## 5. ¿Cómo funciona el flujo completo?

1. El cliente envía una pregunta al endpoint `POST /orchestrator`.
2. El servicio valida credenciales técnicas y, si existe, la identidad del usuario final.
3. Se crea o recupera una conversación desde Cosmos DB.
4. Se consulta App Configuration para conocer la estrategia activa.
5. La estrategia elegida llama a los servicios adecuados.
6. Si es RAG, consulta Azure AI Search y luego Azure AI Foundry.
7. Si es NL2SQL, localiza la fuente de datos, encuentra tablas o queries similares y ejecuta SQL o DAX.
8. Si es MCP, delega el trabajo a herramientas externas vía SSE.
9. La respuesta se transmite en tiempo real al cliente.
10. El contexto actualizado se guarda nuevamente en Cosmos DB.

### Diagrama ASCII del flujo de datos

```text
Usuario -> API -> Auth -> Conversación -> Estrategia
                                  |        |-> Search/Foundry
                                  |        |-> MCP
                                  |        |-> SQL/Fabric
                                  v
                               Cosmos DB
```

## 6. Servicios de Azure utilizados

| Servicio | Propósito | SKU / nivel recomendado |
| --- | --- | --- |
| Azure Container Apps | Ejecutar el contenedor FastAPI | Consumption para dev, perfil dedicado para productivo |
| Azure AI Foundry / Azure AI Services | Chat, embeddings y agentes | Según capacidad del modelo desplegado |
| Azure AI Search | Índice RAG, índices NL2SQL y seguridad documental | Standard S1 mínimo |
| Azure Cosmos DB | Conversaciones y catálogos de datos | Serverless o autoscale |
| Azure App Configuration | Variables de control y feature flags | Standard |
| Azure Key Vault | Secretos y credenciales auxiliares | Standard |
| Azure Container Registry | Publicación de la imagen | Basic o Standard |
| Application Insights | Telemetría y trazas | Workspace-based |

## 7. Modos de despliegue

| Aspecto | Modo básico | Modo enterprise / productivo |
| --- | --- | --- |
| Exposición | Endpoint público HTTPS | Red privada o ingress endurecido |
| Autenticación | Puede permitir anónimo | Entra ID obligatorio |
| Search | Búsqueda estándar | ACL/RBAC trimming con token del usuario |
| Secretos | App Config + Key Vault | Key Vault, RBAC estricto y rotación |
| Observabilidad | Logs y App Insights básicos | Alertas, auditoría y trazas completas |
| Escalado | Autoscale simple | Dimensionamiento dedicado, pruebas de carga y aislamiento |

### Topología enterprise simplificada

```text
Internet / Clientes corporativos
          |
          v
   Front Door / WAF
          |
          v
 Azure Container Apps (Orchestrator)
    |       |        |
    |       |        +--> Application Insights
    |       +----------> App Config / Key Vault
    +------------------> Search / Cosmos / Foundry
```

## 8. Ejemplo práctico

Este repositorio no define un motor declarativo YAML, pero este blueprint refleja una configuración real del servicio para un caso representativo de preguntas sobre ventas.

```yaml
name: ejemplo-retail-nl2sql
app_config:
  AGENT_STRATEGY: nl2sql
datasource:
  id: adventureworks
request:
  ask: "¿Cuáles son los 5 productos más caros disponibles?"
execution:
  - GetAllDatasourcesInfo
  - TablesRetrieval
  - QueriesRetrieval
  - ValidateSQLQuery
  - ExecuteSQLQuery
response:
  transport: sse
```

Qué hace cada paso en lenguaje natural:

1. Selecciona el modo NL2SQL.
2. Busca la fuente de datos correcta.
3. Revisa tablas y consultas similares ya conocidas.
4. Construye una consulta SQL coherente.
5. Valida que la consulta sea sintácticamente correcta.
6. Ejecuta la consulta en modo solo lectura.
7. Devuelve la respuesta ya sintetizada para el usuario.

## 9. Resumen ejecutivo

| Aspecto | Detalle |
| --- | --- |
| Tipo de solución | API de orquestación conversacional para RAG, MCP y NL2SQL |
| Arquitectura | FastAPI + estrategias + conectores Azure |
| Nube | Azure |
| Servicios principales | Container Apps, AI Foundry, AI Search, Cosmos DB, App Config, Key Vault |
| Formatos soportados | HTTP JSON, SSE, prompts de texto/Jinja2, documentos JSON en Cosmos/Search |
| Seguridad | API key o Dapr token, Entra ID opcional/obligatorio, trimming documental en Search |
| Despliegue | Contenedor Docker desplegado con scripts y `azd deploy` |
| Persistencia | Cosmos DB |
| Extensibilidad | nuevas estrategias, plugins, prompts y servidores MCP |
| Dependencias externas | La infraestructura completa vive en el repo padre `Azure/GPT-RAG` |