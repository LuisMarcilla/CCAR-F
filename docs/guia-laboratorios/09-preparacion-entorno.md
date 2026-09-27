# 09 — Preparación del entorno

Pasos para dejar listo el entorno y ejecutar **todos** los notebooks (`01`–`13`), el
servidor MCP y los laboratorios sin errores de librerías.

## Resumen

| Qué | Valor |
|---|---|
| Python | **3.12** (la grabación usó 3.12.7) |
| Entorno virtual | `.venv/` en la raíz del repositorio |
| Dependencias | `requirements.txt`: `anthropic==0.111.0`, `python-dotenv==1.2.2`, `mcp==1.28.1`, `jupyter` |
| API key | `.env` en la raíz con `ANTHROPIC_API_KEY=...` (sólo para los notebooks de lección) |
| Modelo | `claude-sonnet-4-6`, ya fijado en cada notebook |

> **Por qué `anthropic==0.111.0` y no la última versión:** el SDK 1.0 (agosto 2026) eliminó
> `temperature` de `messages.create()`, y `03_temperature.ipynb` lo usa. Con la versión 0.x
> todos los notebooks corren como en la grabación. Ver
> [Known issue: `temperature`](../../README.md#known-issue-temperature).

Todos los comandos se ejecutan **desde la raíz del repositorio** (la carpeta que contiene
`README.md` y `requirements.txt`).

---

## 1. Instalar Python 3.12

Comprueba primero si ya lo tienes:

| Sistema | Comando | Resultado esperado |
|---|---|---|
| Windows | `py -0` | Una línea con `3.12` |
| macOS / Linux | `python3.12 --version` | `Python 3.12.x` |

Si no está instalado:

| Sistema | Instalación |
|---|---|
| Windows | `winget install Python.Python.3.12` o el instalador de [python.org](https://www.python.org/downloads/) (marca *Add python.exe to PATH*) |
| macOS | `brew install python@3.12` |
| Ubuntu / Debian | `sudo apt install python3.12 python3.12-venv` |

> Con otras versiones (por ejemplo 3.14), pip resuelve el `requirements.txt` sin conflictos,
> pero el curso sólo se grabó y probó con 3.12. Usa 3.12 para evitar sorpresas.

---

## 2. Crear el entorno e instalar dependencias

### Windows (PowerShell)

```powershell
py -3.12 -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Si `Activate.ps1` da *"running scripts is disabled on this system"*, ver
[errores típicos](#6-errores-típicos-y-solución). También se puede trabajar **sin activar**
el entorno, llamando siempre al Python del venv:

```powershell
.venv\Scripts\python.exe -m pip install -r requirements.txt
.venv\Scripts\python.exe -m jupyter lab
```

### Windows (cmd)

```bat
py -3.12 -m venv .venv
.venv\Scripts\activate.bat
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

### macOS / Linux (bash / zsh)

```bash
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Con el entorno activo, el prompt muestra `(.venv)` al inicio. Usar `python -m pip` en lugar
de `pip` garantiza que se instala en el mismo Python que se está ejecutando.

---

## 3. Configurar la API key

Sólo la necesitan los notebooks de lección. Los laboratorios de `labs/` funcionan sin ella.

Crea un archivo llamado `.env` **en la raíz del repositorio**:

```
ANTHROPIC_API_KEY=sk-ant-...
```

| Sistema | Comando para crearlo |
|---|---|
| Windows (PowerShell) | `Set-Content -Path .env -Value "ANTHROPIC_API_KEY=sk-ant-..." -Encoding ascii` |
| macOS / Linux | `echo "ANTHROPIC_API_KEY=sk-ant-..." > .env` |

- La key se obtiene en <https://console.anthropic.com>.
- `.env` ya está en `.gitignore`: **nunca** lo subas al repositorio ni pegues la key en un
  notebook.
- Las llamadas consumen créditos de tu cuenta.

---

## 4. Verificar el entorno

Con el entorno activo, ejecuta este comando. Funciona igual en PowerShell, cmd, bash y zsh:

```
python -c "import sys, os, importlib.metadata as m; print('Python', sys.version.split()[0], '|', sys.prefix); [print(p, m.version(p)) for p in ('anthropic', 'python-dotenv', 'mcp', 'jupyter', 'ipykernel')]; from anthropic import Anthropic; from dotenv import load_dotenv; from mcp.server.fastmcp import FastMCP; load_dotenv(); print('ANTHROPIC_API_KEY:', 'OK' if os.getenv('ANTHROPIC_API_KEY') else 'FALTA')"
```

Salida esperada:

```
Python 3.12.x | <ruta-del-repo>/.venv
anthropic 0.111.0
python-dotenv 1.2.2
mcp 1.28.1
jupyter 1.1.1
ipykernel 7.x.x
ANTHROPIC_API_KEY: OK
```

Qué revisar:

| Si ves… | Significa |
|---|---|
| `sys.prefix` no termina en `.venv` | Estás usando otro Python; activa el entorno (paso 2) |
| `PackageNotFoundError: ... anthropic` | Las dependencias no están instaladas en este Python |
| `anthropic 1.x` | Se instaló una versión incompatible; reinstala con `requirements.txt` |
| `ANTHROPIC_API_KEY: FALTA` | Falta `.env` o no está en la raíz (los labs funcionan igual) |

El comando no imprime la key, sólo si existe.

---

## 5. Abrir los notebooks y seleccionar el kernel

El **kernel** es el Python que ejecuta las celdas. Si no es el del `.venv`, aparecen errores
de módulos aunque la instalación esté bien. Es la causa más común de problemas.

### Opción A — Jupyter Lab

```bash
python -m jupyter lab
```

Lanzado con el entorno activo, Jupyter usa por defecto el kernel **Python 3 (ipykernel)** del
`.venv`. Para que aparezca con un nombre inconfundible (útil si tienes varios entornos),
registra el kernel una sola vez:

```bash
python -m ipykernel install --user --name ccar-f --display-name "Python (CCAR-F)"
```

Luego, en cada notebook: menú **Kernel → Change Kernel… → Python (CCAR-F)**.

### Opción B — VS Code

1. Instala las extensiones **Python** y **Jupyter** de Microsoft.
2. Abre la carpeta del repositorio (**File → Open Folder**), no un notebook suelto.
3. Abre un notebook y pulsa **Select Kernel** (arriba a la derecha).
4. Elige **Python Environments… → `.venv` (Python 3.12.x)**.
   Si no aparece: **Ctrl/Cmd+Shift+P → Python: Select Interpreter → Enter interpreter path**
   y apunta a `.venv\Scripts\python.exe` (Windows) o `.venv/bin/python` (macOS/Linux).

### Comprobar el kernel desde un notebook

Ejecuta en una celda:

```python
import sys; print(sys.executable)
```

La ruta debe contener `.venv`.

### Laboratorios

Cada lab se abre **desde su propia carpeta**, porque importa `shopassist_lab.py`, que está
junto al notebook:

```bash
cd labs/lab_first_request
python -m jupyter lab
```

En VS Code basta con abrir `labs/lab_*/notebook.ipynb`: el kernel arranca en la carpeta del
notebook. Los labs sólo necesitan Jupyter; `anthropic` y `python-dotenv` son opcionales
(el simulador los reemplaza si faltan).

### Servidor MCP

```bash
python shopassist_mcp_server.py
```

No imprime nada y queda esperando: habla MCP por stdin/stdout y lo debe lanzar un cliente
MCP. Se detiene con **Ctrl+C**.

---

## 6. Errores típicos y solución

| Error | Causa | Solución |
|---|---|---|
| `ModuleNotFoundError: No module named 'anthropic'` (o `dotenv`, `mcp`) | El kernel no es el `.venv`, o no se instalaron las dependencias | Revisa el kernel (paso 5) con `sys.executable`; si es el correcto, ejecuta `python -m pip install -r requirements.txt` y reinicia el kernel |
| `TypeError: Messages.create() got an unexpected keyword argument 'temperature'` | Tienes `anthropic` 1.x | `python -m pip install -r requirements.txt` (vuelve a 0.111.0) y **reinicia el kernel** (Kernel → Restart) |
| HTTP 400 mencionando `temperature` | Cambiaste el modelo a Sonnet 5 / Opus 4.7 o posterior, que rechazan el parámetro | Deja `model = "claude-sonnet-4-6"` en `03_temperature.ipynb` |
| `Could not resolve authentication method…` o `AuthenticationError` / HTTP 401 | No se cargó la key, o es inválida | Crea `.env` en la raíz (paso 3), abre Jupyter desde la raíz, reinicia el kernel; revisa la key en la consola |
| `running scripts is disabled on this system` al activar en PowerShell | Política de ejecución de Windows | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`. Si tu equipo no lo permite, usa `activate.bat` desde cmd o llama a `.venv\Scripts\python.exe` directamente |
| `'python3.12' is not recognized` en Windows | En Windows el lanzador es `py` | `py -3.12 -m venv .venv` |
| `No Python at '...'` o `Unable to create process` | El `.venv` se creó con un Python que se desinstaló o movió | Borra `.venv` y vuelve a crearlo (paso 2) |
| `SSL: CERTIFICATE_VERIFY_FAILED` o timeouts en `pip install` | Proxy o inspección TLS de la red corporativa | Configura el proxy (`pip install --proxy http://usuario:clave@proxy:puerto ...`) o el certificado corporativo (`pip config set global.cert <ruta-al-certificado.pem>`); consulta a Mesa de Ayuda / Seguridad TI |
| Mismo error SSL al **llamar** a la API desde el notebook | El proxy corporativo también intercepta `api.anthropic.com` | Define `SSL_CERT_FILE` / `REQUESTS_CA_BUNDLE` apuntando al certificado corporativo, o pide que se habilite el dominio |
| `ModuleNotFoundError: No module named 'shopassist_lab'` | El lab se abrió desde otra carpeta | Abre Jupyter desde `labs/lab_<tema>/` (paso 5, *Laboratorios*) |
| El lab llama a Claude de verdad (sin banner `SIMULATED MODE`) | `ANTHROPIC_API_KEY` está definida en el entorno o hay un `.env` en la carpeta del lab | Es el comportamiento esperado; si quieres modo offline, abre el lab en una terminal sin esa variable |
| `NameError: name '...' is not defined` | Se ejecutó una celda antes que las anteriores | Kernel → **Restart & Run All**, o ejecuta las celdas en orden desde la primera |
| El notebook 01 reinstala paquetes con `%pip install` | Esa celda existe para quien empieza sin entorno | Con el `.venv` ya preparado se puede saltar; si la ejecutas, reinicia el kernel después |

---

## 7. Checklist rápido

- [ ] `py -0` / `python3.12 --version` muestra 3.12
- [ ] `.venv` creado y activo (el prompt muestra `(.venv)`)
- [ ] `python -m pip install -r requirements.txt` sin errores
- [ ] El comando de verificación muestra `anthropic 0.111.0` y `sys.prefix` en `.venv`
- [ ] `.env` en la raíz con `ANTHROPIC_API_KEY` (sólo notebooks de lección)
- [ ] Kernel del notebook apuntando a `.venv` (`sys.executable`)
