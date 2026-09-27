# 01 — Fundamentos de la API de Claude

**Cubre:** notebooks `01_first_request`, `02_multi_turn_chat`, `03_temperature` y los
laboratorios `lab_first_request`, `lab_multi_turn`, `lab_helpers`, `lab_system_prompt`,
`lab_temperature`.

**Objetivo del bloque:** dominar la única llamada sobre la que se construye todo lo demás,
`client.messages.create(...)`, y entender que la API **no guarda estado**: la memoria, las
reglas y los parámetros de generación los decide la aplicación en cada request.

---

## Paso 1 — Primera solicitud (`01_first_request.ipynb`)

### Qué contiene

1. Celda de instalación: `%pip install "anthropic<1" python-dotenv`.
2. Patrón de arranque que se repite en **todos** los notebooks:

```python
from dotenv import load_dotenv
from anthropic import Anthropic

load_dotenv()          # lee ANTHROPIC_API_KEY desde .env
client = Anthropic()   # el cliente toma la key del entorno; nunca va en el código
model = "claude-sonnet-4-6"
```

3. Una sola llamada con `model`, `max_tokens=300` y una lista `messages` con un mensaje
   `user`: *"A customer wants to return an order. Write a short helpful response."*
4. Se imprime `message.content[0].text`.

### Qué observar

La respuesta es genérica (inventa "30 días", "Returns Portal", "5-7 días hábiles"). El modelo
no conoce la tienda ni sus políticas: todavía no le dimos contexto ni reglas.

### Laboratorio `lab_first_request` (3 tareas)

| Tarea | Qué se completa | Lección |
|---|---|---|
| 1 | La llamada `messages.create` con `model`, `max_tokens=300` y `RETURN_REQUEST` | La base de todo: herramientas y agentes son esta misma llamada repetida |
| 2 | Dos llamadas idénticas con `max_tokens=20` y `max_tokens=300` | `stop_reason = "max_tokens"` significa respuesta **cortada**; `"end_turn"` significa completa y segura de mostrar |
| 3 | Leer `message.usage.input_tokens` y `output_tokens` del objeto | La respuesta trae más que texto: `id` (para logs), `usage` (para costos), `stop_reason` (para ramificar la lógica) |

**Para el examen:** la aplicación siempre debe revisar `stop_reason`. El bucle agéntico
(paso 9) se basa exactamente en esa señal.

---

## Paso 2 — Conversaciones multi-turno (`02_multi_turn_chat.ipynb`)

### Qué contiene

1. **El error intencional:** se envía sólo *"My order number is 12345"*. Claude responde que
   no tiene contexto. La API es *stateless*.
2. **La corrección:** se envía la historia completa en `messages`, alternando roles
   `user` → `assistant` → `user`.
3. **Los tres helpers** que usarán todos los notebooks siguientes:

```python
def add_user_message(messages, text):
    messages.append({"role": "user", "content": text})

def add_assistant_message(messages, text):
    messages.append({"role": "assistant", "content": text})

def chat(messages):
    message = client.messages.create(model=model, max_tokens=300, messages=messages)
    return message.content[0].text
```

4. **El `system_prompt` de ShopAssist**, que se pasa con el argumento `system=`:

```text
You are ShopAssist AI, a helpful customer support assistant for an online store.
Your job is to help customers with order questions, returns, refunds, shipping issues, and product support.
Be concise, polite, and practical.
Do not promise a refund until the order is checked.
If you need more information, ask one clear question at a time.
```

### Qué observar

Sin system prompt, Claude dice "soy una IA y no tengo acceso a pedidos". Con el system
prompt, asume el rol de ShopAssist, pide el número de pedido y **no** promete reembolso.

### Laboratorio `lab_multi_turn` (3 tareas)

| Tarea | Qué se hace | Lección |
|---|---|---|
| 1 | Enviar `FIRST_MESSAGE` ("I want to return my order.") | ShopAssist tiene que preguntar qué pedido |
| 2 | Enviar sólo `ORDER_NUMBER_REPLY`, sin historia | El modelo pierde el hilo: **cada llamada empieza de cero** |
| 3 | Reenviar la frase con los dos turnos previos (elegir el rol `"assistant"` para la respuesta de ShopAssist) | La memoria es responsabilidad de la aplicación (BD o sesión), no del modelo |

### Laboratorio `lab_helpers` (3 tareas)

| Tarea | Qué se hace | Lección |
|---|---|---|
| 1 | Completar los roles en `add_user_message` / `add_assistant_message` y pasar `messages` en `chat()` | `chat()` **no** guarda la respuesta: separarlo permite registrar, revisar o modificar la respuesta antes de que entre a la historia |
| 2 | Primer intercambio: agregar usuario → `chat` → agregar asistente | El paso que se olvida es guardar la respuesta del asistente |
| 3 | Segundo intercambio sobre **la misma lista** (sin `messages = []`) | Claude no recordó nada; la aplicación sí |

### Laboratorio `lab_system_prompt` (3 tareas)

| Tarea | Qué se hace | Lección |
|---|---|---|
| 1 | Enviar la queja sin `system` | Sin reglas, el asistente promete lo que suena amable |
| 2 | Misma queja con `system=system_prompt` | `messages` = lo que se dijo; `system` = cómo debe comportarse |
| 3 | Mover `system=system_prompt` **dentro de `chat()`** | El system prompt tampoco se recuerda entre llamadas: si se olvida en un turno, el asistente vuelve a prometer reembolsos |

**Cierre del lab, clave para el examen:** un system prompt es **guía, no control**. "No
prometas reembolsos" se cumple *casi siempre*, y "casi siempre" no alcanza para una regla
sobre dinero. En la sección de herramientas esa regla se traslada al backend.

---

## Paso 3 — Temperature y stop sequences (`03_temperature.ipynb`)

### Qué contiene

Una llamada con:

- `temperature=0` → respuestas repetibles.
- `stop_sequences=["<END>"]` → el texto se corta en el marcador.
- Se imprimen `stop_reason` (`"stop_sequence"`) y `stop_sequence` (`"<END>"`).

### Laboratorio `lab_temperature` (3 tareas)

| Tarea | Qué se hace | Lección |
|---|---|---|
| 1 | 3 llamadas iguales con `temperature=0` y contar respuestas distintas → **1** | Temperatura baja: clasificación, políticas, respuestas de soporte |
| 2 | Las mismas 3 llamadas con `temperature=1.0` → **más de 1** | Temperatura alta: brainstorming; mala idea para una regla de reembolso |
| 3 | `stop_sequences=["<END>"]` (es una **lista** de strings) | `stop_reason` pasa a `"stop_sequence"` y el marcador **no** aparece en el texto. Una stop sequence controla dónde termina la generación; **no es validación** |

### Nota: `temperature` está siendo retirado

El repositorio documenta un cambio del ecosistema (septiembre 2026):

- El SDK `anthropic` 1.0 (20-ago-2026) eliminó `temperature`, `top_p` y `top_k` de
  `messages.create()` → `TypeError: ... unexpected keyword argument 'temperature'`.
- Claude Sonnet 5, Opus 4.7 y modelos más nuevos lo rechazan a nivel de API (HTTP 400).
- Por eso el curso fija `anthropic<1` y `claude-sonnet-4-6`. En SDK 1.x la única forma de
  enviarlo es `extra_body={"temperature": 0}`.
- En modelos nuevos, la consistencia se controla con instrucciones en el prompt (y con
  `output_config={"effort": ...}`).
- `temperature=0` **nunca** garantizó salida idéntica byte a byte.
- Se quitó `temperature=0` de los notebooks 04, 05 y 07, donde no enseñaba nada.

---

## Resumen del bloque

| Concepto | Regla práctica |
|---|---|
| `messages.create` | Es la única primitiva; todo lo demás se arma encima |
| `max_tokens` | Presupuesto; si se agota, `stop_reason = "max_tokens"` y la respuesta está incompleta |
| `stop_reason` | `end_turn`, `max_tokens`, `stop_sequence`, `tool_use` — la app debe ramificar según su valor |
| Estado | La API no tiene memoria; la app reenvía historia y `system` en **cada** llamada |
| System prompt | Define rol y reglas, pero es guía, no garantía |
| `temperature` | Concepto válido, parámetro en retiro |

## Preguntas de repaso

1. ¿Por qué la Tarea 2 de `lab_multi_turn` produce una respuesta sin sentido si el modelo es
   el mismo que en la Tarea 3?
2. ¿Qué pasa si `system=` se envía sólo en el primer turno de una conversación?
3. ¿Qué significa `stop_reason == "max_tokens"` y qué debería hacer la aplicación?
4. ¿Por qué una stop sequence no reemplaza la validación de un JSON?
