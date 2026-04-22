# AI Governance sin drama: Logic Apps como orquestador multi-modelo con Content Safety, routing inteligente y control de gasto

> **Global Azure Lima 2026** · Speaker: Luis Franco · Solutions Architect @ Inetum / Pacífico Seguros  
> **Charla:** Documentos Complejos, Agentes Simples: De Logic Apps a MCP en Azure AI Foundry

---

## 📋 Descripción

Este repositorio contiene los recursos, flujos y documentación de la charla presentada en **Global Azure Lima 2026**, donde exploramos cómo evolucionar desde integraciones tradicionales con Logic Apps hacia arquitecturas modernas basadas en agentes con MCP (Model Context Protocol), aplicando AI Governance real en cada paso.

La demo utiliza un caso de uso real de la industria aseguradora peruana: **comparación automatizada de Formato de Colocación vs Documento emitido**, con detección de discrepancias usando IA generativa y una capa de governance que protege el pipeline completo.

---

## 🎯 Conceptos principales

### Azure AI Content Safety

Servicio de Azure que analiza texto en tiempo real y devuelve un **severity score de 0 a 6** por cada una de las 4 categorías de contenido dañino, antes de que el prompt llegue a cualquier LLM.

| Categoría | Qué detecta |
|---|---|
| **Hate** | Discurso de odio y discriminación |
| **Violence** | Contenido violento o amenazas |
| **Sexual** | Contenido sexual explícito |
| **SelfHarm** | Autolesión o suicidio |

**Severity scale:** `0` = Limpio · `2` = Leve · `4` = Moderado · `6` = Severo  
**Regla en los flujos:** severity `>= 4` → bloquea. Configurable según industria.

**Planes:**
- `F0` — Free · 5,000 text records/mes · $0.00 · Sin overage (ideal para PoC)
- `S0` — Standard · $0.38 / 1,000 records · Pay-as-you-go (producción)

---

### Prompt Shields

Capa especializada de Content Safety que detecta **prompt injection** — instrucciones maliciosas disfrazadas de texto legítimo, embebidas en documentos PDF, emails o cualquier contenido que procese el LLM.

**¿Por qué es diferente a Content Safety?**  
Content Safety detecta contenido dañino explícito (hate, violence, etc.). Prompt Shields detecta la **intención de manipular al modelo** aunque el texto parezca inocente.

**Endpoint:**
```
POST {endpoint}/contentsafety/text:shieldPrompt?api-version=2024-09-01
```

**Body:**
```json
{
  "userPrompt": "texto a analizar",
  "documents": []
}
```

**Response:**
```json
{
  "userPromptAnalysis": {
    "attackDetected": true
  }
}
```

**Caso real demostrado:** Un PDF de Documento de seguros con instrucciones maliciosas en texto gris (invisible visualmente, pero legible por Document Intelligence OCR) que intentan manipular al LLM para aprobar siniestros sin revisión.

---

### Azure Document Intelligence

Servicio de OCR avanzado que extrae texto estructurado de PDFs, Word y otros formatos. En los flujos de esta demo usa el modelo `prebuilt-read` para extraer el contenido de Formato y Documento antes de enviarlo al LLM.

**Flujo de uso:**
1. `POST` al endpoint de análisis → recibe `operation-location` en headers
2. `Wait` 5 segundos (procesamiento async)
3. `GET` al `operation-location` → obtiene `analyzeResult.content`

---

### Logic Apps como AI Orchestrator

**Logic Apps Consumption** actúa como orquestador agnóstico al modelo, sin escribir una línea de código de aplicación. Conecta SharePoint, Document Intelligence, Content Safety, Prompt Shields y GPT-4o en un flujo visual con governance integrada.

**Patrón de governance en el flujo:**
```
SharePoint trigger
  └─ Get PDF Documento + Formato
      └─ Document Intelligence OCR x2
          └─ Prompt Shields (detecta injection)
              └─ Content Safety (detecta hate/violence/etc)
                  └─ Safety Gate (OR: attackDetected OR severity >= 4)
                      ├─ BLOCK → Log + Email alerta + Update SharePoint "Bloqueado"
                      └─ PASS  → Get Prompt → GPT-4o → Email resultado al analista
```

**Cost tracking por llamada:**  
Cada ejecución loguea `prompt_tokens`, `completion_tokens`, `total_tokens` y `estimated_cost_usd`. El email al analista incluye el costo estimado del procesamiento — base para governance financiero.

---

### Multi-model Router (Demo 2)

Logic Apps enruta inteligentemente entre múltiples modelos según el tipo de tarea:

| `task_type` | Modelo | Razón |
|---|---|---|
| `analysis` | GPT-4o (Azure OpenAI) | Razonamiento complejo |
| `code` | Claude Sonnet (Anthropic) | Contexto largo, precisión técnica |
| `document` | Claude Sonnet (Anthropic) | Extracción de documentos complejos |
| `simple` | Gemini Flash (Google) | Bajo costo — ~30x más barato |
| `preferred_model` override | Cualquiera | A/B testing en runtime |

**Cost governance:** Si el costo por request supera el umbral configurado, dispara alerta. El campo `governance` en cada response documenta modelo usado, severity, tokens y routing reason — auditable desde el primer request.

---

### MCP — Model Context Protocol

Protocolo abierto creado por Anthropic y adoptado por Microsoft, Google y el ecosistema, que **estandariza cómo los agentes de IA se conectan a herramientas externas**. Es el "USB-C de la IA".

**Antes de MCP:** cada modelo tenía su propia forma de llamar a SharePoint, una base de datos o una API.  
**Con MCP:** todos usan el mismo contrato. El agente no sabe si la herramienta está en Azure, en tu laptop o en un servidor — solo sabe que puede llamarla.

**Logic Apps Standard como MCP Server:**  
Cada workflow con trigger `McpTool` se convierte en una herramienta invocable por el agente de AI Foundry.

```
get_Documento(Documento_id)       → SharePoint → PDF
extract_text(pdf_base64)    → Document Intelligence → texto
safety_check(text)          → Content Safety + Prompt Shields → { passed, severity }
compare_docs(Formato, Documento)  → GPT-4o → discrepancias en HTML
```

**El agente decide el orden en runtime** — no hay `if` hardcodeado en el designer. Si `safety_check` retorna `attackDetected: true`, el agente no llama a `compare_docs`. Esa decisión la toma el agente, no el código.

---

### Azure AI Foundry Agent

Orquestador inteligente que razona sobre qué tools MCP invocar. Recibe lenguaje natural del usuario, planifica los pasos y retorna el resultado.

**AS-IS vs TO-BE:**

| | AS-IS (Logic Apps Consumption) | TO-BE (Agente MCP) |
|---|---|---|
| Trigger | SharePoint cada 3 min | Lenguaje natural del analista |
| Orden de pasos | Hardcodeado en el designer | El agente decide en runtime |
| Input | Item en lista SharePoint | "Compara la Documento 2024-001" |
| Memoria | Ninguna | Contexto entre sesiones |
| Governance | Content Safety en el flujo | Content Safety como tool del agente |

**Frase clave:** El Logic App que funciona hoy no muere — se convierte en herramienta del agente. Eso es evolución, no revolución.

---

## 🏗️ Arquitectura — Los 3 Actos

```
ACTO 1 — El flujo real
SharePoint → Document Intelligence OCR → GPT-4o → Email al analista

         ↓ upgrade

ACTO 2 — AI Governance
SharePoint → Doc Intelligence → Prompt Shields → Content Safety
                                    ↓
                              Safety Gate (OR)
                             /              \
                        BLOCK              PASS
                    Email alerta      GPT-4o → Email
                    SP "Bloqueado"    SP "Procesado"

         ↓ TO-BE

ACTO 3 — MCP + AI Foundry Agent
Analista (lenguaje natural) → Foundry Agent
                                    ↓
                              MCP Server (Logic Apps Standard)
                         ┌────┬──────────┬────────────┐
                    get_Documento  extract_text  safety_check  compare_docs
                         └────┴──────────┴────────────┘
                              Agente razona y responde
```

---

## 🔑 Conceptos clave resumidos

| Concepto | Definición en 2 líneas |
|---|---|
| **Content Safety** | Analiza texto y retorna severity 0-6 por 4 categorías. Bloquea antes de que el prompt llegue al LLM. |
| **Prompt Shields** | Detecta instrucciones maliciosas disfrazadas de texto legal en documentos. `attackDetected: true` aunque severity sea 0. |
| **Prompt Injection via PDF** | Ataque donde instrucciones maliciosas se embeben en un PDF. OCR las extrae y llegan al modelo como texto válido. |
| **Logic Apps Orchestrator** | Flujo visual que conecta servicios de Azure sin código de app. Governance integrada en cada step. |
| **Multi-model Router** | Switch por `task_type` que elige el modelo más adecuado y económico para cada tarea. |
| **Cost Governance** | Token tracking por llamada con costo estimado en USD. Alerta si supera umbral. Auditable en cada response. |
| **MCP** | Protocolo open source que estandariza cómo agentes se conectan a herramientas. El "USB-C de la IA". |
| **MCP Server** | Logic Apps Standard con trigger `McpTool`. Cada workflow es una tool que el agente invoca. |
| **AI Foundry Agent** | Orquestador que razona sobre qué tools MCP usar. No hay orden hardcodeado — decide en runtime. |

---

## 🚀 ¿Escala a microservicios?

Sí. El patrón MCP es agnóstico al runtime:

- Cada tool puede ser un **Azure Function**, un **Container App** o un **pod en AKS**
- **Dapr** (proyecto CNCF) puede actuar como sidecar de governance en lugar de Logic Apps
- **Semantic Kernel** (C# / Python) conecta MCP Servers, Azure AI Foundry y memoria en una sola capa de orquestación enterprise

```
AKS Cluster
├── Pod: mcp-server (Logic Apps Standard o Azure Functions)
│   ├── tool: safety_check  → Content Safety
│   ├── tool: extract_text  → Document Intelligence
│   └── tool: compare_docs  → Azure OpenAI
└── Pod: ai-foundry-agent
    └── Llama tools del MCP Server via protocolo estándar
```

---

## 🛠️ SDKs disponibles

| SDK | Lenguaje | Caso de uso |
|---|---|---|
| `@modelcontextprotocol/sdk` | TypeScript / Node | Construir MCP Servers custom |
| `mcp` | Python | Servers y clients en Python |
| **Semantic Kernel** | C# / Python | Integración enterprise con Azure AI Foundry |

---

## 📁 Estructura del repositorio

```
/
├── README.md                          ← este archivo
├── flows/
│   ├── demo1-single-model-safety/    ← Logic App: Content Safety + GPT-4o
│   ├── demo2-multimodel-router/      ← Logic App: routing GPT + Claude + Gemini
│   ├── Documentos-con-content-safety/   ← Logic App: flujo real de Documentos con governance
│   └── mcp-server/                   ← Logic Apps Standard: 4 tools MCP
├── docs/
│   ├── architecture/                 ← diagramas C4 de la arquitectura
│   └── openapi/                      ← OpenAPI spec del MCP Server
└── samples/
    └── documents/                    ← documentos de prueba (sin datos reales)
```

---

## 🔗 Recursos

- [Azure AI Content Safety — Docs](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/)
- [Prompt Shields — Quickstart](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/quickstart-jailbreak)
- [Model Context Protocol — Spec](https://modelcontextprotocol.io)
- [Azure AI Foundry — Agents](https://ai.azure.com)
- [Logic Apps Standard — MCP](https://learn.microsoft.com/en-us/azure/logic-apps/)
- [Semantic Kernel](https://learn.microsoft.com/en-us/semantic-kernel/)

---

## 👤 Speaker

**Luis Franco** — Solutions Architect con 10+ años de experiencia en Azure, AI Integration y DevSecOps.  
Certificaciones activas: MCT · AZ-400 · Azure Administrator · DevOps Expert · SCRUM Master · AEA Member.

🔗 [linkedin.com/in/luisfrancor](https://linkedin.com/in/luisfrancor)

---

> *"El código que construimos hoy no es un prototipo que vamos a tirar. Es el foundation del agente de mañana."*  
> — Global Azure Lima 2026
