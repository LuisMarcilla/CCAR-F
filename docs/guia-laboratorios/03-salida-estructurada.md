# 03 — Tool use y salida estructurada

**Cubre:** notebooks `06_tool_use`, `07_json_output`, `08_tools_extract` y los laboratorios
`lab_structured_output`, `lab_validation`, `lab_extract_returns` (BUILD).

**Objetivo del bloque:** pasar de pedirle prosa a Claude a obtener **datos** en los que el
código puede confiar. La fórmula del bloque es:

> **Claude extrae. La aplicación valida.**

---

## Paso 6 — Definir herramientas (`06_tool_use.ipynb`)

### Qué contiene

1. **Un ejemplo malo** de definición de herramienta: `get_info` con descripción
   *"Gets information."* y un parámetro `query`. Es deliberadamente vago: el modelo no sabe
   cuándo usarla ni qué devuelve.
2. **Un ejemplo correcto**: `lookup_order` con descripción clara, `input_schema` JSON Schema
   con `order_id` descrito y marcado `required`.
3. Una llamada con `tools=tools` y el mensaje *"Can I return order 12345?"*.

### Qué observar

La salida es:

```
tool_use
[TextBlock(text='Let me look up that order for you right away!'),
 ToolUseBlock(id='toolu_...', name='lookup_order', input={'order_id': '12345'})]
```

- `stop_reason` pasa a ser `"tool_use"`.
- `content` es una **lista de bloques**, no sólo texto: puede traer un `TextBlock` y uno o
  más `ToolUseBlock`.
- **Claude no ejecuta la herramienta.** Sólo pide que se ejecute, con argumentos que
  cumplen el schema. Ejecutarla es trabajo de la aplicación (paso 9).

Anatomía de una herramienta:

| Campo | Para qué |
|---|---|
| `name` | Identificador que Claude usará en el `tool_use` |
| `description` | Superficie de prompt: cuándo usarla y cuándo **no** |
| `input_schema` | JSON Schema de los argumentos (`type`, `properties`, `required`, `enum`…) |

---

## Paso 7 — JSON confiable (`07_json_output.ipynb`)

### Qué contiene

Una herramienta `extract_return_request` usada **sólo para extraer datos** (nunca se
ejecuta nada), con este schema:

| Campo | Tipo | Decisión de diseño |
|---|---|---|
| `order_id`, `item` | `["string", "null"]` | Requerido pero **nullable**: puede decir "no hay" sin inventar |
| `reason` | enum: `normal_return`, `damaged_item`, `billing_dispute`, `policy_exception`, `unclear`, `other` | Enum en vez de texto libre |
| `reason_detail` | nullable | Explica cuando `reason` es `other` o `unclear` |
| `desired_action` | enum: `refund`, `replacement`, `exchange`, `store_credit`, `unclear`, `other` | Idem |
| `desired_action_detail` | nullable | Idem |
| `evidence_provided` | boolean | |
| `urgency` | enum: `low`, `normal`, `high`, `unclear` | |
| `missing_information` | array de strings | Qué pedirle al cliente después |

Todos los campos están en `required`.

La llamada fuerza la herramienta con:

```python
tool_choice={"type": "tool", "name": "extract_return_request"}
```

y agrega **reglas de normalización** en el prompt (p. ej. "si el cliente dice que *puede*
mandar evidencia después, `evidence_provided` es `false`"; "usa `damaged_item` sólo si llegó
roto, rayado, defectuoso o inutilizable").

Se lee el resultado así:

```python
tool_use = next(block for block in message.content if block.type == "tool_use")
structured_data = tool_use.input   # ya es un dict de Python
```

### Qué observar — comparación directa

| Pedir "Return only valid JSON" en prosa | Tool + schema + `tool_choice` forzado |
|---|---|
| Viene envuelto en ```` ```json ```` → `json.loads` falla | Viene como dict, no hay que parsear |
| Campos inventados: `resolution`, `evidence_type`, `product` | Exactamente los campos definidos |
| `"reason": "scratched"` (no existe en el router) | `"reason": "damaged_item"` (valor del enum) |
| `evidence_available: true` (el cliente sólo *puede* mandar foto) | `evidence_provided: false` (aplica la regla) |
| No indica qué falta | `missing_information: ["order_id"]` |

### Laboratorio `lab_structured_output` (3 tareas)

| Tarea | Qué se hace | Lección |
|---|---|---|
| 1 | Pedir JSON en prosa, sin herramientas | Casi siempre "funciona", y esa es la trampa: la estructura es la que el modelo decidió |
| 2 | Misma extracción con `tools=tools` y `tool_choice` forzado a `extract_return_request` | Claude llena una estructura que tú definiste. Cuatro decisiones de diseño: requerido-pero-nullable, enums, opción `unclear`, opción `other` + detalle |
| 3 | Escribir `route(data)`: `order_id is None` → `ask_for_order_id`; `billing_dispute` → `billing_team`; `policy_exception` → `human_escalation`; resto → `returns_workflow` | El schema vale la pena porque el código puede **enrutar** por campos con nombre. El orden de los checks importa: sin `order_id` no hay nada útil que hacer |

**Límite honesto:** un schema controla la **forma** de la respuesta, no su **significado**.
Claude puede devolver un `normal_return` válido para un artículo claramente dañado. Eso lo
resuelve el paso siguiente.

---

## Paso 8 — Extracción con validación y revisión humana (`08_tools_extract.ipynb`)

### Qué contiene

1. El schema se amplía con `confidence` (0 a 1), `human_review_required` y
   `human_review_reason`, y el enum de `reason` incluye `wrong_item` y `changed_mind`.
2. Se envía un mensaje contradictorio: *"The item works perfectly, but it arrived broken, and
   I need a replacement today"*. Claude marca `confidence: 0.75`,
   `human_review_required: true` y explica la contradicción.
3. Dos funciones de validación en código:

```python
def validate_return_request(data):
    errors = []
    if data["order_id"] is None:          errors.append("order_id is missing")
    if data["item"] is None:              errors.append("item is missing")
    if data["reason"] == "other" and not data["reason_details"]:
        errors.append("reason_details is required when reason is other")
    if data["confidence"] < 0.7:          errors.append("confidence is below review threshold")
    return errors

def needs_human_review(data):
    return (data["confidence"] < 0.7
            or data["reason"] in ["unclear", "policy_exception"]
            or data["human_review_required"])
```

### Qué observar

En la salida grabada, Claude devolvió `"order_id": "<UNKNOWN>"` en lugar de `null` porque
este notebook no incluye las reglas de normalización del notebook 07. Por eso
`validate_return_request` (que busca `None`) **no** lo detectaría como faltante. Es un buen
ejemplo de por qué hay que validar significado, no sólo forma, y de por qué conviene decir
explícitamente en el prompt "usa null si no se proporcionó".

### Laboratorio `lab_validation` (3 tareas, sin llamadas a la API)

Trabaja sobre cuatro extracciones ya obtenidas. **Todas** son JSON válido con todos los
campos. La arquitectura es la lección: la mitad que valida nunca habla con un modelo.

| Caso | Mensaje | Problema |
|---|---|---|
| `clean` | Zapatos rayados, pide reemplazo, adjunta foto | Ninguno |
| `wrong label` | Chaqueta talla incorrecta, quiere cambiarla | `desired_action = "exchange"`, que la política no soporta |
| `no order number` | Audífonos fallan, pide reembolso | `order_id = None` |
| `contradiction` | Pide reembolso "pero si se puede, envíenme otro" | `conflict_detected = True`, `confidence = 0.45` |

| Tarea | Función | Lección |
|---|---|---|
| 1 | `schema_errors(data)`: `desired_action` debe estar en `ALLOWED_ACTIONS` (`refund`, `replacement`, `store_credit`, `unclear`) | Error **corregible**: la información estaba en el mensaje, el modelo la etiquetó mal → **reintentar con feedback específico** |
| 2 | `missing_fields(data)`: campos de `MUST_BE_PRESENT` (`order_id`, `item`) que vienen `None` | Error **no corregible**: la información nunca estuvo. **No reintentar**: un reintento sólo puede producir un número de pedido inventado. Se pregunta al cliente |
| 3 | `needs_human_review(data)`: `conflict_detected` o `confidence < CONFIDENCE_THRESHOLD (0.7)` | Contradicción o baja confianza → persona. Lo demás pasa: una cola de revisión que atrapa todo es una cola que nadie lee |

El lab muestra cómo se ve un **reintento útil**. Tiene cuatro ingredientes: el mensaje
original, la extracción fallida, el error exacto y la instrucción de no inventar nada.
"Please try again" no tiene ninguno.

Tabla de decisión final (el **orden** de los checks es la decisión):

```
clean              automate
wrong label        retry with feedback
no order number    ask the customer
contradiction      human review
```

### Laboratorio `lab_extract_returns` — BUILD (3 tareas)

Integra todo lo anterior en el módulo que ShopAssist realmente usaría: entra una frase,
sale una **decisión**. El schema agrega `confidence`, `conflict_detected`, `conflict_reason` y
`human_review_required`.

| Tarea | Qué se hace | Lección |
|---|---|---|
| 1 | `process(customer_message)`: extraer → `schema_errors` → `missing_fields` → `needs_human_review` → `automate` | El orden es el diseño. Si se reintenta por un `order_id` faltante, el sistema "aprende" a inventarlos |
| 2 | `review_required(data)` agrega `data["human_review_required"]` a los checks propios | El modelo puede **levantar** una alerta que tus reglas no previeron (compra de hace 6 meses = cuestión de política), pero nunca **bajarla**: "confiar en la bandera hacia arriba, nunca hacia abajo" |
| 3 | Pasar 5 mensajes reales por `process` y calcular la **tasa de automatización** | Es el número que mira un equipo de soporte. Se sube mejorando prompt y schema, **nunca** relajando los checks |

---

## Resumen del bloque

| Concepto | Regla práctica |
|---|---|
| Herramienta como extractor | `tools` + `tool_choice={"type":"tool","name":...}` fuerza estructura |
| Nullable + required | El campo debe existir; el valor puede ser `null` → nada se inventa |
| Enums | Limitan las categorías a las que el router sabe manejar |
| `unclear` / `other` + detalle | Válvulas honestas para ambigüedad |
| Schema ≠ validación | La forma la controla el schema; el significado, el código |
| Reintentar | Sólo cuando el problema es corregible y con feedback específico |
| Dato ausente | `null` + preguntar; nunca reintentar para "sacarlo" |
| Revisión humana | Conflicto, baja confianza o bandera del modelo (sólo hacia arriba) |

## Preguntas de repaso

1. ¿Por qué `"type": ["string", "null"]` y `required` a la vez no es contradictorio?
2. Un `desired_action` fuera de política y un `order_id` nulo: ¿cuál se reintenta y por qué?
3. ¿Por qué el modelo puede activar `human_review_required` pero no desactivar un check de
   la aplicación?
4. ¿Qué tres diferencias concretas hay entre "Return only valid JSON" y una herramienta
   forzada con schema?
