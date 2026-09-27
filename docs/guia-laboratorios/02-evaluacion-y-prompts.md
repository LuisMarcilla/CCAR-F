# 02 — Evaluación de prompts y prompt engineering

**Cubre:** notebooks `04_test_evaluation`, `05_prompt_eng` y los laboratorios
`lab_evaluation`, `lab_grading`.

**Objetivo del bloque:** dejar de juzgar un prompt "a ojo" y convertir su calidad en un
número comparable entre versiones; y estructurar prompts para que sean claros y
controlables.

---

## Paso 4 — Evaluación (`04_test_evaluation.ipynb`)

### Qué contiene

1. **`classify_intent(customer_message)`**: pide a Claude clasificar el mensaje en una de
   cinco intenciones y devolver **sólo** JSON (sin markdown, sin explicación, sin bloque de
   código), con un ejemplo `{"intent": "refund_request"}`. Luego hace `json.loads(...)`.

   Intenciones permitidas: `refund_request`, `order_status`, `billing_issue`,
   `product_question`, `other`.

2. **Dataset de prueba** (`test_cases`) con `input` y `expected_intent`:

   | Mensaje | Intención esperada |
   |---|---|
   | "I want to return my shoes. They arrived damaged." | `refund_request` |
   | "Where is my order? It was supposed to arrive yesterday." | `order_status` |
   | "I was charged twice for the same order." | `billing_issue` |

   Un bucle compara `actual == expected`, imprime `True/False` y acumula cada caso en
   `results` con un `score` de 10 (acierto) o 0 (fallo).

3. **Técnicas de grading:**
   - Promedio de puntajes de `results` con `statistics.mean`.
   - `grade_response(customer_message, assistant_response)`: una segunda llamada a Claude
     con `system=grading_prompt`. El prompt tiene 4 reglas (cortesía, no prometer reembolso
     sin revisar el pedido, pedir número de pedido, escalar si hay fraude o acción legal) y
     la función devuelve `score` (1-10), `passed` y `reason`.
   - Graders en código: `has_valid_intent`, `has_required_fields`, `is_valid_json`.

### Laboratorio `lab_evaluation` (3 tareas)

| Tarea | Qué se hace | Lección |
|---|---|---|
| 1 | Escribir la etiqueta esperada de cada caso **antes** de ejecutar nada | Lo que separa un eval de una demo: la respuesta correcta se decide de antemano |
| 2 | Recorrer los casos con `classify_intent(test_case["input"])` y guardar `response["intent"]` | Un solo mensaje no prueba nada; hay que correr todo el set cada vez |
| 3 | Marcar `passed = actual == expected` y calcular el puntaje | Cambiar el prompt → repetir los mismos casos → comparar puntajes |

**El caso "trampa":** *"I was charged twice and I want my money back."* está etiquetado como
`refund_request`, pero Claude puede responder `billing_issue`. **No es un bug**: ambas
etiquetas son defendibles. La decisión la toma el equipo y se documenta; el modelo no puede
adivinarla. Encontrar estos desacuerdos es precisamente para lo que sirven los evals.

### Laboratorio `lab_grading` (3 tareas)

Se evalúan dos respuestas de ShopAssist al mismo cliente. Ambas son JSON válido y tienen
todos los campos, pero la "version 1" **aprueba un reembolso sin revisar el pedido**.

| Tarea | Qué se hace | Lección |
|---|---|---|
| 1 | Completar `has_required_fields` (con `REQUIRED_FIELDS`) y `has_valid_intent` (con `ALLOWED_INTENTS`) | Grading por código: rápido, gratis, determinista. Se ejecuta **primero** |
| 2 | Enviar cada respuesta a una segunda llamada con `system=grading_prompt` | Grading por modelo: juzga lo que un `if` no puede (tono, promesas indebidas). El grader también puede equivocarse; por eso las reglas son específicas |
| 3 | `final_score = (code_score + model_score) / 2` y promedio del dataset | Un número por salida permite comparar versiones; el valor absoluto no significa nada |

**Conclusión del lab:** ambas salidas obtienen 10 en el grading por código. El grading por
código dice "parseable, completo y enrutable"; si la frase debía enviarse es otra pregunta y
necesita otro tipo de grader. En producción se usan ambos, el barato primero, y siempre se
**leen los fallos**, no sólo el promedio (un promedio mejor puede esconder una regresión).

Los tres tipos de grading del curso:

| Tipo | Qué evalúa | Costo | Determinista |
|---|---|---|---|
| Código | Formato, campos requeridos, valores permitidos | ~0 | Sí |
| Modelo (LLM-as-judge) | Cumplimiento de reglas, tono, promesas | Una llamada extra | No |
| Humano | Casos ambiguos, calibración del grader | Alto | No |

---

## Paso 5 — Prompt engineering (`05_prompt_eng.ipynb`)

### Qué contiene

Un único prompt estructurado con **etiquetas XML** para separar responsabilidades:

```xml
<role>You are a customer support assistant for ShopAssist AI.</role>
<task>Write a short response to the customer.</task>
<rules>
- Be polite and calm.
- Do not approve a refund.
- Ask for the order number.
- Keep the response under 80 words.
</rules>
<customer_message>{customer_message}</customer_message>
<output_format>Return only the message that should be sent to the customer.</output_format>
```

### Qué observar

- Las etiquetas separan claramente **instrucciones** de **datos del cliente** (el mensaje
  del cliente queda aislado en `<customer_message>`), lo que reduce ambigüedad.
- Las reglas son concretas y verificables ("menos de 80 palabras", "pedir número de pedido"),
  lo que permite evaluarlas después con los graders del paso 4.
- `<output_format>` indica exactamente qué debe devolverse, sin preámbulos.

No tiene laboratorio asociado.

---

## Resumen del bloque

| Concepto | Regla práctica |
|---|---|
| Eval | Dataset con respuestas esperadas escritas **antes** de ejecutar |
| Comparación | Mismo dataset + prompt nuevo → comparar puntajes |
| Grading por código | Primero; formato y valores permitidos |
| Grading por modelo | Reglas específicas, no "¿es buena?" |
| Fallos | Leerlos siempre; el promedio resume y esconde |
| Prompt estructurado | XML para rol, tarea, reglas, datos y formato de salida |

## Preguntas de repaso

1. ¿Por qué el valor esperado debe fijarse antes de ejecutar el prompt?
2. Una salida pasa todos los checks de código. ¿Qué riesgo sigue existiendo?
3. ¿Por qué el `grading_prompt` lista cuatro reglas concretas en vez de preguntar si la
   respuesta es buena?
4. ¿Qué ventaja aporta envolver el mensaje del cliente en `<customer_message>`?
