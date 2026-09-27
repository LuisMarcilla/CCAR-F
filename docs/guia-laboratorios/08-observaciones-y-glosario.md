# 08 — Observaciones del código, conceptos clave y glosario

## A. Observaciones encontradas al revisar el repositorio

Se detectaron al revisar el material. Las marcadas como **Corregido** ya están resueltas en
el código; las otras dos se dejaron como están a propósito (se explica por qué).

| # | Archivo | Observación | Estado |
|---|---|---|---|
| 1 | `04_test_evaluation.ipynb`, celda 3 | Usaba `results` para `mean(...)` sin que ninguna celda lo definiera → `NameError` | **Corregido**: la celda 2 ahora acumula `results` (con `score` 10/0 según acierte la etiqueta) |
| 2 | `04_test_evaluation.ipynb`, celda 4 | `grade_response` era pseudo-código: un comentario y un dict literal con `false` → `NameError: name 'false'` | **Corregido**: hace la segunda llamada con `system=grading_prompt` y devuelve `json.loads(...)`, igual que `lab_grading` |
| 3 | `11_shopassist_tools.ipynb`, celdas 9 y 10 | Llamaban a `check_refund_eligibility(order)` con un solo dict, pero la función requiere `order_id, delivered_days_ago, status` → `TypeError` | **Corregido**: pasan los tres argumentos por nombre |
| 4 | `11_shopassist_tools.ipynb` vs `shopassist_mcp_server.py` | `process_refund` y `create_human_escalation` no tenían `@mcp.tool()` ni docstring, y la firma de `create_human_escalation` difería | **Corregido**: las 5 herramientas coinciden en firma, valores por defecto, docstring y comportamiento |
| 5 | `CLAUDE.md` | La convención de errores no incluía `transient`, que usa `10_tools_errors.ipynb` | **Corregido**: se agregó `transient` y la regla "`isRetryable` es `True` sólo para fallos transitorios" |
| 6 | `08_tools_extract.ipynb` | La salida grabada tiene `"order_id": "<UNKNOWN>"` en vez de `null`, y `validate_return_request` (que compara con `None`) no lo detectaría | **Sin cambios**: es una salida real del modelo que ilustra por qué hay que validar significado y pedir `null` en el prompt (como hace el 07). Cambiar el prompt dejaría la salida grabada desalineada del código |
| 7 | `13_shopassist_agents.ipynb` | `case_facts` se envía una vez (vacío) en el prompt inicial y luego sólo se actualiza del lado de la app | **Sin cambios**: es una decisión de diseño, no un error. El modelo razona con los `tool_result`, y `case_facts` es el estado del backend que se usa en el handoff de seguridad |
| 8 | `13_shopassist_agents.ipynb` | `update_case_facts` leía `recommended_action` de `check_return_policy`, que nunca lo devuelve | **Corregido**: se eliminó esa lectura muerta |
| 9 | `09` y `13` | Al terminar se leía `content[0].text`, suponiendo que el primer bloque era texto | **Corregido**: se toma el primer bloque con `type == "text"` |
| 10 | `CLAUDE.md` | Mencionaba `claude-temperature-slide.html`, que no existe | **Corregido**: se quitó la referencia |
| 11 | `09` y `13` | El `while True` no tenía límite de iteraciones | **Corregido**: `MAX_STEPS` (10 en el 09, 15 en el 13). El 09 lanza `RuntimeError`; el 13 escala a un humano con `case_facts`, igual que ante un `stop_reason` inesperado |

Validación: las celdas de código de los notebooks 04, 09, 11 y 13 se ejecutaron de punta a
punta contra el simulador de los labs (sin llamadas reales). También se comprobó que el
notebook 11 y el servidor MCP devuelvan lo mismo para las mismas entradas y que el límite
de pasos se active cuando corresponde. Las salidas grabadas en los notebooks **no** se
regeneraron: siguen siendo las de la grabación original.

## B. Conceptos clave que el curso repite

1. **La API no guarda estado.** Historia (`messages`) y reglas (`system`) se reenvían en
   cada llamada.
2. **`stop_reason` gobierna el control de flujo**: `end_turn`, `max_tokens`,
   `stop_sequence`, `tool_use`.
3. **El prompt guía; el código hace cumplir.** Las reglas sobre dinero, identidad y
   permisos viven en herramientas deterministas.
4. **Evals antes que opiniones.** Dataset con respuestas esperadas escritas de antemano,
   grading por código primero y por modelo después; leer los fallos.
5. **Estructura por schema, no por pedido en prosa.** `tools` + `tool_choice` forzado +
   campos nullable + enums con `unclear`/`other`.
6. **El schema controla la forma; la validación controla el significado.**
7. **Reintentar sólo lo corregible**, con feedback específico. Lo ausente se marca `null` y
   se pregunta al cliente.
8. **Revisión humana por señales concretas**: conflicto, baja confianza, bandera del modelo
   (que sólo puede subir la alerta, nunca bajarla).
9. **Bucle agéntico**: guardar el `content` completo del asistente, ejecutar cada
   `tool_use`, devolver `tool_result` con su `tool_use_id`, repetir.
10. **Errores estructurados**: nunca lanzar excepciones; `isError`, `errorCategory`,
    `isRetryable`, `customerMessage`, `developerMessage`; "no encontrado" no es error.
11. **Descripciones de herramientas = superficie de prompt**: cuándo usarla y cuándo no.
12. **Verificación antes de actuar**: email → `customer_id` → pedido de ese cliente.
13. **Separar decidir de actuar**: `check_refund_eligibility` ≠ `process_refund`.
14. **Mínimo privilegio**: `allowedTools` por agente; un solo agente habla con el cliente.
15. **Escalamiento estructurado** con causa raíz, evidencia, monto, acción recomendada e
    información faltante.
16. **Degradación segura**: ante un estado inesperado, escalar en vez de fallar en silencio.

## C. Glosario

| Término | Significado en este curso |
|---|---|
| `messages` | Lista de turnos `{"role": "user"/"assistant", "content": ...}` que forma la conversación |
| `system` | Argumento separado con rol, tono y reglas del asistente |
| `max_tokens` | Presupuesto máximo de tokens de salida |
| `stop_reason` | Por qué terminó la generación (`end_turn`, `max_tokens`, `stop_sequence`, `tool_use`) |
| `stop_sequences` | Lista de marcadores que cortan la generación; el marcador no aparece en el texto |
| `temperature` | Aleatoriedad del muestreo (0 = repetible); en retiro en SDK 1.x y modelos nuevos |
| `usage` | `input_tokens` / `output_tokens` de la llamada, base para costos |
| Eval | Conjunto de casos con respuesta esperada para medir un prompt |
| Grader por código | Verificación determinista (JSON válido, campos, valores permitidos) |
| Grader por modelo | Segunda llamada a Claude que juzga contra reglas explícitas |
| Tool / herramienta | `name` + `description` + `input_schema` que Claude puede pedir ejecutar |
| `tool_choice` | `auto`, `any`, `none` o `{"type": "tool", "name": ...}` para forzar una herramienta |
| `tool_use` (bloque) | Petición de Claude de ejecutar una herramienta, con `id`, `name` e `input` |
| `tool_result` (bloque) | Resultado que la app devuelve, enlazado por `tool_use_id` |
| Bucle agéntico | Ciclo llamar → ejecutar herramientas → devolver resultados → repetir hasta `end_turn` |
| Nullable | Campo `["string", "null"]`: debe existir, pero puede no tener valor |
| Enum | Lista cerrada de valores permitidos para un campo |
| MCP | Model Context Protocol: estándar para exponer herramientas a clientes/agentes |
| FastMCP | SDK de Python para crear servidores MCP con decoradores `@mcp.tool()` |
| stdio | Transporte de MCP por stdin/stdout usado por `shopassist_mcp_server.py` |
| Coordinador | Agente que descompone el caso, delega y es el único que habla con el cliente |
| Subagente | Agente especializado con herramientas restringidas que devuelve hallazgos |
| `allowedTools` | Lista de herramientas que un agente puede usar |
| `Task` | Herramienta con la que el coordinador lanza subagentes |
| `case_facts` | Registro estructurado del caso que la app acumula tras cada herramienta |
| Handoff | Escalamiento estructurado a un humano con todo el contexto del caso |
| Tasa de automatización | Porcentaje de casos resueltos sin intervención humana |
