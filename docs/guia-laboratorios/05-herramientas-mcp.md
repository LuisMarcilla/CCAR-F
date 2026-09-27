# 05 — Las herramientas canónicas de ShopAssist y el servidor MCP

**Cubre:** notebook `11_shopassist_tools` y el script `shopassist_mcp_server.py`. No tienen
laboratorio asociado.

**Objetivo del bloque:** definir el conjunto definitivo de herramientas de ShopAssist como
herramientas **MCP** (Model Context Protocol), con verificación de identidad, errores
estructurados y la política de reembolso aplicada por las herramientas, no por el modelo.

---

## Paso 11 — `11_shopassist_tools.ipynb`

### 1. Herramientas por agente (principio de mínimo privilegio)

El notebook empieza definiendo qué herramientas puede usar cada agente:

```python
main_support_agent_tools = ["get_customer_by_email", "lookup_order_by_id",
                            "check_refund_eligibility", "create_human_escalation"]

refund_agent_tools       = ["lookup_order_by_id", "check_refund_eligibility",
                            "process_refund", "create_human_escalation"]
```

El agente de soporte general **no** tiene `process_refund`: sólo el agente de reembolsos puede
mover dinero. Es la antesala del diseño multi-agente del paso 12.

### 2. Servidor FastMCP y datos de prueba

```python
from mcp.server.fastmcp import FastMCP
mcp = FastMCP("ShopAssistMCP")
```

Datos en memoria: el cliente `alex@example.com` → `CUS-1001`, y dos pedidos suyos,
`ORD-12345678` (12 días, 89.99 USD) y `ORD-87654321` (45 días, 149.99 USD).

### 3. Las cinco herramientas

| Herramienta | Qué hace | Precondición declarada en el docstring | Errores |
|---|---|---|---|
| `get_customer_by_email(email)` | Resuelve email → perfil con `customer_id`. Normaliza con `strip().lower()` | "Úsala cuando el usuario da un email pero no un customer ID verificado. **No** la uses para buscar un pedido" | No encontrado → `isError: False, customer: None` |
| `lookup_order_by_id(order_id, customer_id)` | Devuelve el pedido. Normaliza con `strip().upper()` | "Úsala **sólo** cuando tengas order ID **y** customer ID verificado. No la uses con un email en lugar del customer_id" | `validation` (no empieza con `ORD-`), `permission` (el pedido no es del cliente); no encontrado → `order: None` |
| `check_refund_eligibility(order_id, delivered_days_ago, status)` | Aplica la política: debe estar `delivered` y dentro de 30 días | "Sólo revisa la política. **No** procesa el reembolso. Usa `process_refund` sólo después de que esta confirme elegibilidad" | `business` (no entregado / fuera de ventana) |
| `process_refund(order_id, amount, reason)` | Ejecuta el reembolso (`REF-555000`, estado `submitted`) | "Úsala sólo después de que `check_refund_eligibility` confirme" | `validation` si `amount <= 0` |
| `create_human_escalation(customer_id, reason, summary, order_id=None)` | Crea ticket `TCK-9001` | "Úsala cuando la automatización deba detenerse: permisos, excepciones de política, información contradictoria, casos de alto riesgo" | — |

### 4. Flujos de demostración

- **Camino feliz:** email → `CUS-1001` → `ORD-12345678` → elegible → `process_refund` por el
  total.
- **Camino de escalamiento:** `ORD-87654321` (45 días) → `check_refund_eligibility` devuelve
  error `business` → `create_human_escalation` con `reason="refund_policy_exception"` y el
  `developerMessage` como resumen.

### Qué observar — decisiones de diseño clave

1. **Los docstrings son superficie de prompt.** FastMCP convierte el docstring en la
   descripción de la herramienta que ve el modelo. Frases como "Use this only when…" y
   "Do not use this tool to…" le indican cuándo usarla y cuándo **no**.
2. **Verificación de identidad en cadena.** Un email no sirve para ver pedidos: primero hay
   que resolverlo a un `customer_id`, y `lookup_order_by_id` comprueba que el pedido
   pertenezca a ese cliente (`permission` si no).
3. **Separar "decidir" de "actuar".** `check_refund_eligibility` sólo evalúa la política;
   `process_refund` sólo ejecuta. Así la política vive en código determinista, no en el
   criterio del modelo.
4. **Salida limpia hacia el humano.** Cuando la automatización no puede seguir, se genera un
   ticket con un resumen, no una respuesta improvisada.

---

## El servidor `shopassist_mcp_server.py`

Es la versión desplegable de lo que el notebook 11 prototipa: las mismas cinco
herramientas decoradas con `@mcp.tool()`, con tipos (`dict[str, Any]`) y docstrings, y un
punto de entrada:

```python
if __name__ == "__main__":
    mcp.run()
```

Ejecución:

```bash
.venv/bin/python shopassist_mcp_server.py
```

No imprime nada: habla el protocolo MCP por **stdin/stdout** (transporte stdio) y está
pensado para que lo invoque un **cliente MCP** (por ejemplo Claude Desktop, Claude Code o
un cliente propio), que descubre las herramientas y sus descripciones automáticamente.

### Por qué MCP

| Sin MCP (notebooks 06-10) | Con MCP |
|---|---|
| Las herramientas se definen a mano como dicts JSON Schema en cada app | El servidor publica nombre, schema (derivado de los tipos) y descripción (del docstring) |
| La app mantiene el dict de despacho `tool_functions` | El cliente MCP invoca la herramienta por protocolo |
| Las herramientas viven dentro del notebook | Las herramientas son un servicio reutilizable por cualquier cliente/agente |

### Sincronización entre el notebook y el servidor

El `CLAUDE.md` del proyecto pide mantenerlos sincronizados. Hoy las cinco herramientas
coinciden en firma, valores por defecto, docstring y comportamiento. Antes,
`process_refund` y `create_human_escalation` no tenían `@mcp.tool()` en el notebook y la
firma de la segunda era distinta; ver [08 — Observaciones](08-observaciones-y-glosario.md).

---

## Preguntas de repaso

1. ¿Por qué `lookup_order_by_id` exige `customer_id` y no acepta un email?
2. ¿Qué ventaja tiene separar `check_refund_eligibility` de `process_refund`?
3. ¿De dónde saca FastMCP la descripción de cada herramienta que ve el modelo?
4. ¿Por qué el agente de soporte general no tiene `process_refund` en su lista?
