# 07 — Cómo funcionan los laboratorios (simulador `shopassist_lab`)

Los 10 laboratorios de `labs/` son los mismos que corren dentro de Udemy, publicados en el
repositorio para practicarlos sin plan de laboratorio y **sin API key**.

## Estructura de cada laboratorio

```
labs/lab_<tema>/
├── START_HERE.md       instrucciones y solución de problemas
├── notebook.ipynb      el ejercicio, con blancos "..."
├── solution.ipynb      el notebook resuelto
└── shopassist_lab.py   simulador offline del SDK de Anthropic (idéntico en los 10 labs)
```

## Formato de cada tarea

Cada celda de tarea tiene el mismo bloque de comentarios:

| Sección | Contenido |
|---|---|
| `WHAT TO DO` | Qué hay que completar |
| `WHY IT MATTERS` | El concepto que enseña (es la parte que más sirve para el examen) |
| `WHERE TO SEE IT IN THE LECTURE` | Minuto exacto del video donde se escribe el mismo código |
| `HOW TO DO IT` | Qué va en cada `...` |
| `WHAT YOU SHOULD SEE` | Resultado esperado (en algunas tareas) |

Al final de la celda, `check("<nombre>", ...)` imprime `[PASS]` con una explicación o
`[FAIL]` con una pista (`Hint:`) concreta.

## Cómo ejecutarlos

```bash
cd labs/lab_first_request
jupyter lab            # o abrir notebook.ipynb en VS Code
```

Requisitos: Python 3.10+ y Jupyter. `anthropic` y `python-dotenv` son **opcionales**: si no
están instalados, el simulador los reemplaza; si están, se usan tal cual.

La primera celda es la única distinta a un proyecto real:

```python
import shopassist_lab
from shopassist_lab import check
# desde aquí, código normal de la API de Claude
```

## Qué es y qué no es el simulador

`shopassist_lab.py` (~135 KB, sólo librería estándar) instala *shims* de `anthropic` y
`dotenv`. **No es Claude**: las respuestas son fixtures predefinidas que se eligen con una
tabla de reglas. No usa red ni un modelo de lenguaje.

Se identifica claramente:

- imprime un banner al crear el cliente;
- los `id` simulados empiezan con `msg_sim_`;
- los mensajes simulados tienen `.simulated == True`;
- `explain(message)` indica qué fixture eligió y por qué.

Lo que sí reproduce fielmente (los parámetros importan):

| Parámetro | Comportamiento simulado |
|---|---|
| `max_tokens` | Trunca y pone `stop_reason="max_tokens"` |
| `stop_sequences` | Corta en el marcador y pone `stop_reason="stop_sequence"` |
| `temperature` | `0` repite byte a byte; `>0` varía de forma reproducible |
| `system` | Detecta reglas del system prompt y cambia la respuesta |
| `tools` | Produce bloques `tool_use` reales con `input` válido según el schema |
| `tool_choice` | Respeta `auto`, `any`, `none` y `{"type": "tool", ...}` forzado |

Con `ANTHROPIC_API_KEY` definida (entorno o `.env`), el mismo notebook llama a Claude de
verdad y el simulador se aparta. Para `temperature`, reenvía el valor al SDK real y muestra
una nota, por el retiro del parámetro en SDK 1.x.

## Comandos útiles

| Llamada | Qué hace |
|---|---|
| `lab_info()` | Qué se simula, qué no y las limitaciones conocidas |
| `explain(message)` | Por qué el simulador devolvió esa respuesta |
| `check_all(...)` | Ejecuta todos los checks de los que haya artefactos |

## Solución de problemas (de `START_HERE.md`)

| Síntoma | Causa |
|---|---|
| "You still have ... in this request" | Queda un blanco sin completar |
| `NameError` | Se saltó una celda; ejecutar en orden desde arriba |
| Un check falla | Leer la línea `Hint:` |
| No imprime nada | Ejecutar primero la celda de setup |

## Mapa de laboratorios

| # | Laboratorio | Tareas | Llama a la API | Guía |
|---|---|---|---|---|
| 1 | `lab_first_request` | request, `stop_reason`, objeto respuesta | Sí | [01](01-fundamentos-api.md) |
| 2 | `lab_multi_turn` | apertura, sin historia, con historia | Sí | [01](01-fundamentos-api.md) |
| 3 | `lab_helpers` | helpers, 1.er intercambio, 2.º intercambio | Sí | [01](01-fundamentos-api.md) |
| 4 | `lab_system_prompt` | sin system, con system, system en cada turno | Sí | [01](01-fundamentos-api.md) |
| 5 | `lab_temperature` | temp 0, temp 1.0, stop sequence | Sí | [01](01-fundamentos-api.md) |
| 6 | `lab_evaluation` | dataset, ejecución, comparación | Sí | [02](02-evaluacion-y-prompts.md) |
| 7 | `lab_grading` | grading por código, por modelo, combinado | Sí | [02](02-evaluacion-y-prompts.md) |
| 8 | `lab_structured_output` | JSON libre, tool + schema, enrutamiento | Sí | [03](03-salida-estructurada.md) |
| 9 | `lab_validation` | valores fuera de política, faltantes, revisión humana | **No** | [03](03-salida-estructurada.md) |
| 10 | `lab_extract_returns` (BUILD) | pipeline, bandera del modelo, bandeja de 5 mensajes | Sí | [03](03-salida-estructurada.md) |

Los notebooks 05, 06 y 09-13 (prompt engineering, tool use básico, bucle agéntico,
errores, MCP y multi-agente) **no tienen laboratorio** en el repositorio.
