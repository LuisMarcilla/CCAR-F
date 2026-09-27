# 06 — Diseño multi-agente y el caso completo con `case_facts`

**Cubre:** notebooks `12_agents_hub` y `13_shopassist_agents`. No tienen laboratorio
asociado.

**Objetivo del bloque:** repartir un caso complejo entre agentes especializados con
herramientas restringidas y, en el ejemplo final, resolver un caso real de principio a fin
con un bucle agéntico que acumula hechos estructurados y escala a un humano cuando un límite
de negocio lo exige.

---

## Paso 12 — Hub de agentes (`12_agents_hub.ipynb`)

### Qué contiene

Sólo **configuración** (no ejecuta llamadas): cuatro agentes definidos como dicts con
`name`, `description`, `allowedTools` y `system`.

| Agente | Herramientas permitidas | Instrucciones clave |
|---|---|---|
| `shopassist_coordinator` | `Task`, `get_customer`, `escalate_to_human` | Descomponer el pedido en tareas, delegar la investigación a subagentes, **no responder al cliente hasta tener los hallazgos**, toda comunicación con el cliente pasa por él |
| `billing_analysis_agent` | `get_payment_events`, `get_refund_status` | Sólo hechos de facturación; devolver hallazgos estructurados; no contactar al cliente; no hacer excepciones de política |
| `order_investigation_agent` | `lookup_order`, `get_shipment_events` | Sólo hechos de pedido y envío; qué pasó, qué evidencia hay, qué falta |
| `policy_review_agent` | `search_policy` | Revisar reglas; no procesar reembolsos; no prometer resultados; devolver regla aplicable, excepciones y confianza |

### Qué observar

- **Patrón coordinador + subagentes (hub-and-spoke).** El coordinador tiene la herramienta
  `Task`, que es la que lanza subagentes (convención del Claude Agent SDK / Claude Code). Los
  subagentes no hablan entre sí ni con el cliente; devuelven hallazgos al coordinador.
- **Mínimo privilegio vía `allowedTools`.** Cada subagente sólo ve las herramientas de su
  especialidad. El agente de políticas no puede reembolsar; el de facturación no puede
  escalar.
- **Un solo punto de contacto con el cliente.** Evita respuestas contradictorias desde
  varios agentes.
- **El `description` de cada agente** es lo que el coordinador usa para decidir a quién
  delegar, igual que la descripción de una herramienta.
- **Los subagentes devuelven hechos, no decisiones.** La síntesis y la respuesta final son
  del coordinador.

---

## Paso 13 — El caso completo (`13_shopassist_agents.ipynb`)

Es el ejemplo más completo del curso: un único bucle agéntico que resuelve un caso con dos
problemas simultáneos.

### El caso

> "Hi, I received my blue jacket yesterday and it arrived damaged. I also think I was
> charged twice. I want a full refund, not a replacement. My order number is ORD-77819."

### Las siete herramientas

| Herramienta | Qué devuelve (simulado) |
|---|---|
| `extract_case_facts(customer_message)` | `order_id`, `issues: ["damaged_item", "possible_duplicate_charge"]`, expectativa "full refund, not replacement" |
| `verify_customer()` | **Sin parámetros**: toma la identidad de la sesión autenticada (`SESSION`), no de lo que diga el cliente → `CUS-1842` |
| `lookup_order(customer_id, order_id)` | Chaqueta azul, 149.00, entregada 2026-06-24, `flags: ["damage_claim"]`; error `ORDER_NOT_FOUND` si cliente/pedido no coinciden |
| `check_return_policy(...)` | Política `returns-v7`, `eligible: True`, `automatic_refund_limit: 75.00` |
| `investigate_duplicate_charge(...)` | `duplicate_charge_found: False`; "second authorization pending, not captured" |
| `process_refund(..., refund_amount, ...)` | Si `refund_amount > 75.00` → `status: "blocked"`, `REFUND_LIMIT_EXCEEDED`, `next_step: "escalate_to_human"` |
| `escalate_to_human(...)` | Handoff estructurado: `root_cause`, `refund_amount`, `recommended_action`, `evidence`, `missing_information`, `escalation_reason` → `HANDOFF-3007` |

El despacho se hace con `run_tool(name, tool_input)` (una cadena de `if`), que devuelve
`UNKNOWN_TOOL` si el nombre no existe.

### El registro `case_facts`

Un dict que la aplicación va completando después de **cada** resultado de herramienta con
`update_case_facts(case_facts, tool_name, result)`:

```python
case_facts = {
    "customer_id": None, "order_id": None, "item_id": None,
    "issues": [], "customer_expectation": None,
    "refund_amount_requested": None, "evidence": [],
    "missing_information": [], "policy": None,
    "payment": None, "escalation_reason": None,
}
```

| Herramienta | Qué escribe en `case_facts` |
|---|---|
| cualquiera con `status: "error"` | agrega `{tool, error_code}` a `missing_information` |
| `extract_case_facts` | `order_id`, `issues`, `customer_expectation` |
| `verify_customer` | `customer_id` |
| `lookup_order` | `order_id`, `item_id`, `refund_amount_requested` (= precio), `evidence` (+flags) |
| `check_return_policy` | `policy` (`policy_id`, `eligible`, `automatic_refund_limit`) |
| `investigate_duplicate_charge` | `payment` |
| `process_refund` bloqueado | `escalation_reason` |
| `escalate_to_human` | `handoff_id` |

### El prompt de orquestación

El mensaje inicial le da a Claude un procedimiento explícito:

1. Extraer los hechos del caso con `extract_case_facts`.
2. Verificar la sesión del cliente autenticado.
3. Revisar el pedido, la política de devoluciones y el posible cobro duplicado.
4. Si el cliente pide reembolso y el artículo es elegible, solicitar `process_refund`.
5. **"Do not decide by yourself whether the refund is allowed."** El backend aplica los
   límites.
6. Si `process_refund` devuelve `blocked`, escalar con un handoff estructurado.
7. No dar respuesta final hasta que el reembolso se haya procesado **o** el caso se haya
   escalado.

### El bucle

Es el mismo patrón del paso 9, con tres mejoras:

- Resultados serializados con `json.dumps(result)` (no `str`).
- `update_case_facts` tras cada herramienta.
- **Salida de seguridad**: si `stop_reason` no es `tool_use` ni `end_turn` (por ejemplo
  `max_tokens`), en vez de lanzar una excepción se escala automáticamente a un humano con
  `case_facts` y el motivo.
- **Tope de pasos** (`MAX_STEPS = 15`): si el agente no termina en ese número de
  iteraciones, también se escala con `case_facts`. Se agregó después de la grabación.

### Resultado grabado

```
extract_case_facts → verify_customer → lookup_order → check_return_policy
→ investigate_duplicate_charge → process_refund (BLOCKED: 149.00 > 75.00)
→ escalate_to_human (HANDOFF-3007)
```

La respuesta final al cliente:

- Confirma el daño y la elegibilidad.
- Aclara que **no** hubo doble cobro, sólo una autorización pendiente no capturada.
- Explica que el reembolso de 149.00 supera el límite automático y requiere aprobación
  humana, y que eso **no** significa un rechazo.
- Entrega el identificador del handoff.

### Qué observar — por qué este es el cierre del curso

| Principio | Cómo se ve aquí |
|---|---|
| La política la aplican las herramientas | El modelo pide reembolsar 149.00; `process_refund` lo bloquea por el límite de 75.00 |
| Identidad desde la sesión, no desde el chat | `verify_customer()` no recibe parámetros |
| Investigar antes de afirmar | El "cobro doble" que decía el cliente resultó ser una autorización pendiente |
| Escalamiento estructurado | El humano recibe causa raíz, evidencia, monto, acción recomendada y lo que falta, no un texto libre |
| Estado del caso en la aplicación | `case_facts` es la fuente de verdad del lado del backend, independiente de la conversación |
| Degradación segura | Un `stop_reason` inesperado termina en escalamiento, no en silencio |

---

## Preguntas de repaso

1. ¿Por qué `verify_customer` no tiene parámetros?
2. El modelo decidió pedir un reembolso de 149.00. ¿Qué componente impidió que se ejecutara?
3. ¿Qué gana el humano que recibe el handoff frente a un simple "el cliente quiere un
   reembolso"?
4. En el diseño del paso 12, ¿por qué los subagentes no pueden hablar con el cliente?
5. ¿Qué hace el bucle del notebook 13 si Claude se queda sin tokens a mitad del caso? ¿Y si
   supera `MAX_STEPS`?
