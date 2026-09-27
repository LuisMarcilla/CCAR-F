# 04 — El bucle agéntico y los errores de herramientas

**Cubre:** notebooks `09_agentic_loop` y `10_tools_errors`. No tienen laboratorio asociado.

**Objetivo del bloque:** que Claude deje de sólo *pedir* herramientas y la aplicación las
**ejecute en un bucle** hasta resolver el caso; y que las herramientas comuniquen los fallos
de forma estructurada para que el modelo reaccione correctamente.

---

## Paso 9 — El bucle agéntico (`09_agentic_loop.ipynb`)

Es el notebook central del curso: **todos los agentes posteriores son variaciones de este
bucle**.

### Qué contiene

1. **Cuatro herramientas** (definición para Claude): `get_customer(email)`,
   `lookup_order(order_id)`, `process_refund(order_id, reason)`, `escalate_to_human(reason)`.
2. **Implementaciones simuladas** en Python que devuelven dicts fijos, y un diccionario de
   despacho:

```python
tool_functions = {
    "get_customer": get_customer,
    "lookup_order": lookup_order,
    "process_refund": process_refund,
    "escalate_to_human": escalate_to_human,
}
```

3. **El bucle:**

```python
MAX_STEPS = 10
steps = 0

while True:
    # 0. Tope de seguridad: un agente que nunca llega a end_turn no debe girar para siempre
    steps += 1
    if steps > MAX_STEPS:
        raise RuntimeError(f"Agent did not finish within {MAX_STEPS} steps")

    response = client.messages.create(model=model, max_tokens=1000,
                                      tools=tools, messages=messages)

    # 1. El turno del asistente se guarda COMPLETO (texto + bloques tool_use)
    messages.append({"role": "assistant", "content": response.content})

    # 2. Terminó: mostrar la respuesta final (el primer bloque de texto)
    if response.stop_reason == "end_turn":
        print(next(block.text for block in response.content if block.type == "text"))
        break

    # 3. Pidió herramientas: ejecutar cada una y devolver resultados
    if response.stop_reason == "tool_use":
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = tool_functions[block.name](**block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,     # enlaza resultado con la petición
                    "content": str(result),
                })
        # 4. Los resultados vuelven como un turno "user"
        messages.append({"role": "user", "content": tool_results})
        continue

    # 5. Cualquier otro stop_reason es inesperado
    raise RuntimeError(f"Unexpected stop_reason: {response.stop_reason}")
```

### Qué observar

La salida muestra la secuencia que Claude decidió por sí mismo:

```
Claude requested tool: get_customer      {'email': 'alex@example.com'}
Claude requested tool: lookup_order      {'order_id': 'ORD-1001'}
Claude requested tool: process_refund    {'order_id': 'ORD-1001', 'reason': 'Keyboard arrived damaged'}
Your refund has been **approved**! ...
```

Puntos clave del patrón:

| Elemento | Por qué importa |
|---|---|
| `stop_reason` gobierna el bucle | `tool_use` → ejecutar y continuar; `end_turn` → terminar |
| Se guarda `response.content` entero | La siguiente llamada necesita ver los `tool_use` que originaron los resultados |
| `tool_use_id` | Cada `tool_result` debe referenciar el `id` exacto de su `tool_use` |
| Resultados en rol `user` | Desde la perspectiva de la API, los resultados los "aporta" el lado de la aplicación |
| Varias herramientas por turno | Se recorren todos los bloques; se devuelven todos los resultados juntos |

### Debilidades intencionales de esta versión (que los pasos siguientes corrigen)

- `process_refund` aprueba **cualquier** reembolso: no hay verificación de elegibilidad, ni
  de propiedad del pedido, ni de montos. El modelo decidió reembolsar y nadie se lo impidió.
- `lookup_order` recibe sólo `order_id`: no comprueba que el pedido sea del cliente.
- Las herramientas no pueden fallar, así que el modelo nunca aprende a manejar errores.
- El resultado se envía con `str(result)` (repr de Python) en vez de JSON.

`MAX_STEPS` no aparece en el video: se agregó después como tope de seguridad, porque un
`while True` sin límite no termina si el modelo nunca devuelve `end_turn`.

---

## Paso 10 — Errores estructurados (`10_tools_errors.ipynb`)

### Qué contiene

Dos funciones que **nunca lanzan excepciones**: devuelven un dict tanto en el éxito como en
el fallo.

`lookup_order_by_id(order_id)` cubre cinco escenarios:

| Entrada | Resultado | `errorCategory` | `isRetryable` |
|---|---|---|---|
| No empieza con `ORD-` | Error | `validation` | `False` |
| `ORD-00000000` | **No es error**: `isError: False`, `orders: []` | — | — |
| `ORD-99999999` | Error: el cliente autenticado no es dueño | `permission` | `False` |
| `ORD-TIMEOUT` | Error: el servicio no respondió | `transient` | `True` |
| Cualquier otro | Éxito con el pedido (entregado hace 42 días, 89.99) | — | — |

`check_refund_eligibility(order)`: si `delivered_days_ago > 30` devuelve error de categoría
`business`; si no, `eligible: True`.

### La convención de error

```python
{
    "isError": True,
    "errorCategory": "validation" | "permission" | "business" | "transient",
    "isRetryable": False,
    "customerMessage": "...",   # seguro para mostrar al cliente final
    "developerMessage": "...",  # detalle interno / para logs
}
```

### Qué observar

- **"No encontrado" no es un error.** Una búsqueda vacía es un resultado válido
  (`isError: False` con payload vacío o `null`). Mezclarlo con un error hace que el agente
  reintente o escale sin motivo.
- **La categoría le dice al modelo qué hacer:**
  - `validation` → pedir al cliente que corrija el dato.
  - `permission` → no insistir; no revelar datos; posiblemente escalar.
  - `business` → la política lo impide; explicar y ofrecer escalamiento.
  - `transient` → es el **único** caso donde reintentar tiene sentido.
- **Dos mensajes separados:** `customerMessage` puede mostrarse tal cual; `developerMessage`
  es para diagnóstico y no debería llegar al cliente.
- Devolver dicts en vez de lanzar excepciones evita que el bucle agéntico se rompa y le da
  al modelo información para decidir el siguiente paso.

---

## Resumen del bloque

| Concepto | Regla práctica |
|---|---|
| Bucle agéntico | `while True` + `stop_reason` (`tool_use` / `end_turn` / otro) |
| Historia | Guardar el `content` completo del asistente y responder con `tool_result` |
| Enlace | `tool_use_id` del resultado = `id` del bloque `tool_use` |
| Despacho | Diccionario `nombre → función`; la app ejecuta, el modelo sólo pide |
| Herramientas | No lanzan excepciones; devuelven éxito o error estructurado |
| No encontrado | Resultado vacío, no error |
| Reintento | Sólo si `isRetryable` (fallos transitorios) |

## Preguntas de repaso

1. ¿Qué ocurre si se agrega a `messages` sólo el texto del asistente y no sus bloques
   `tool_use`?
2. ¿Por qué "pedido no encontrado" no debería devolver `isError: True`?
3. ¿Qué categoría de error justifica un reintento automático?
4. En el notebook 09, ¿qué impidió que el modelo reembolsara un pedido no elegible? (Nada:
   ese es el problema que resuelven los notebooks 10-13.)
