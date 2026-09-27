# Guía de estudio — Laboratorios ShopAssist AI (CCAR-F)

Esta guía explica, en orden secuencial, qué contiene cada notebook y cada laboratorio del
repositorio, qué concepto enseña y qué conviene retener para la certificación
**Claude Certified Architect — Foundations**.

Todo el material construye un mismo caso de negocio: **ShopAssist AI**, un agente ficticio de
soporte al cliente para una tienda online (devoluciones, reembolsos, cobros duplicados,
escalamiento a humanos). Cada lección agrega una capacidad sobre el mismo dominio, así que
el proyecto "crece" a medida que se avanza.

## Dos tipos de material

| Tipo | Dónde | Necesita API key | Qué es |
|---|---|---|---|
| **Notebooks de lección** | raíz: `01_*.ipynb` … `13_*.ipynb` | Sí (`.env` con `ANTHROPIC_API_KEY`) | El código que se escribe en los videos. Llamadas reales a Claude. |
| **Laboratorios** | `labs/lab_*/` | No, con el kernel de `.venv-labs` | Ejercicios con blancos `...` que se validan con `check(...)`. Corren contra un simulador offline (`shopassist_lab.py`) **sólo** si el kernel no tiene `anthropic` instalado; ver [09](09-preparacion-entorno.md#laboratorios-entorno-venv-labs). |
| **Servidor MCP** | `shopassist_mcp_server.py` | No | Versión "desplegable" de las 5 herramientas de ShopAssist como servidor FastMCP. |

## Orden de estudio recomendado

Los laboratorios acompañan a las lecciones. El orden correcto es: ver/ejecutar el notebook
de la lección y, a continuación, resolver el laboratorio que le corresponde.

| Paso | Notebook de lección | Laboratorio asociado | Documento de esta guía |
|---|---|---|---|
| 1 | `01_first_request` | `lab_first_request` | [01 — Fundamentos de la API](01-fundamentos-api.md) |
| 2 | `02_multi_turn_chat` | `lab_multi_turn`, `lab_helpers`, `lab_system_prompt` | [01 — Fundamentos de la API](01-fundamentos-api.md) |
| 3 | `03_temperature` | `lab_temperature` | [01 — Fundamentos de la API](01-fundamentos-api.md) |
| 4 | `04_test_evaluation` | `lab_evaluation`, `lab_grading` | [02 — Evaluación y prompt engineering](02-evaluacion-y-prompts.md) |
| 5 | `05_prompt_eng` | — | [02 — Evaluación y prompt engineering](02-evaluacion-y-prompts.md) |
| 6 | `06_tool_use` | — | [03 — Tool use y salida estructurada](03-salida-estructurada.md) |
| 7 | `07_json_output` | `lab_structured_output` | [03 — Tool use y salida estructurada](03-salida-estructurada.md) |
| 8 | `08_tools_extract` | `lab_validation`, `lab_extract_returns` (BUILD) | [03 — Tool use y salida estructurada](03-salida-estructurada.md) |
| 9 | `09_agentic_loop` | — | [04 — Bucle agéntico y errores de herramientas](04-bucle-agentico-y-errores.md) |
| 10 | `10_tools_errors` | — | [04 — Bucle agéntico y errores de herramientas](04-bucle-agentico-y-errores.md) |
| 11 | `11_shopassist_tools` + `shopassist_mcp_server.py` | — | [05 — Herramientas ShopAssist y MCP](05-herramientas-mcp.md) |
| 12 | `12_agents_hub` | — | [06 — Multi-agente y `case_facts`](06-multiagente-y-case-facts.md) |
| 13 | `13_shopassist_agents` | — | [06 — Multi-agente y `case_facts`](06-multiagente-y-case-facts.md) |

Material de apoyo:

- [09 — Preparación del entorno (Windows, macOS/Linux, kernel, verificación, errores típicos)](09-preparacion-entorno.md) — **empieza por aquí**
- [07 — Cómo funcionan los laboratorios (simulador `shopassist_lab`)](07-simulador-labs.md)
- [08 — Observaciones del código, conceptos clave y glosario](08-observaciones-y-glosario.md)

## La idea que atraviesa todo el curso

> **El prompt guía; el código de la aplicación hace cumplir.**

La progresión completa se puede leer como un traslado gradual de responsabilidad desde el
modelo hacia la aplicación:

```
01-03  El modelo responde texto libre. La app sólo envía y lee.
02     La app es dueña de la memoria (lista messages) y de las reglas (system).
04     La app mide la calidad con evals y graders, no "a ojo".
06-08  La app exige estructura (tool + schema + tool_choice) y valida en código.
09     La app ejecuta las herramientas que el modelo pide, en un bucle.
10-11  Las herramientas devuelven errores estructurados y aplican la política.
12-13  La app orquesta subagentes, acumula hechos del caso y escala a humanos
       cuando un límite de negocio lo exige.
```

## Datos de referencia (fixtures) que se repiten

| Dato | Valor | Dónde aparece |
|---|---|---|
| Cliente canónico | `alex@example.com` → `CUS-1001` (Alex Morgan) | 11, servidor MCP |
| Pedido dentro de ventana | `ORD-12345678`, entregado hace 12 días, 89.99 USD | 11, servidor MCP, labs |
| Pedido fuera de ventana | `ORD-87654321`, entregado hace 45 días, 149.99 USD | 11, servidor MCP, labs |
| Caso completo del notebook 13 | `CUS-1842`, `ORD-77819`, chaqueta azul 149.00, límite automático 75.00 | 13 |
| Modelo | `model = "claude-sonnet-4-6"` en una variable al inicio de cada notebook | todos |

## Requisitos para ejecutar

- Python 3.12 con entorno virtual en `.venv/`.
- `python -m pip install -r requirements.txt` (fija `anthropic==0.111.0`; mantenerse en 0.x es
  deliberado, ver [01 — Fundamentos](01-fundamentos-api.md#nota-temperature-está-siendo-retirado)).
- `.env` con `ANTHROPIC_API_KEY` sólo para los notebooks de lección. Los labs no lo necesitan.

Paso a paso completo en [09 — Preparación del entorno](09-preparacion-entorno.md).
