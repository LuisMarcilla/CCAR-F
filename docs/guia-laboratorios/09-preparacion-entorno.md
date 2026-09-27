# 09 — Preparación del entorno

Pasos para dejar listo el entorno y ejecutar **todos** los notebooks (`01`–`13`), el
servidor MCP y los laboratorios sin errores de librerías.

## Resumen

| Qué | Valor |
|---|---|
| Python | **3.12** (la grabación usó 3.12.7) |
| Entornos virtuales | `.venv/` para los notebooks de lección (API real) y `.venv-labs/` para los laboratorios (simulador, sólo Jupyter) |
| Dependencias | `.venv`: `requirements.txt` (`anthropic==0.111.0`, `python-dotenv==1.2.2`, `mcp==1.28.1`, `jupyter`). `.venv-labs`: sólo `jupyter` |
| API key | `.env` en la raíz con `ANTHROPIC_API_KEY=...` (sólo para los notebooks de lección) |

> **Por qué dos entornos:** el simulador de los labs sólo se activa si `anthropic` **no** está
> instalado. En `.venv` sí lo está, así que un lab abierto con ese kernel llama a la API
> real con tu key, con costo. Ver [Laboratorios](#laboratorios-entorno-venv-labs).
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
| Windows | Instalador de [python.org](https://www.python.org/downloads/) (recomendado, ver la sección siguiente) o `winget install Python.Python.3.12` |
| macOS | `brew install python@3.12` |
| Ubuntu / Debian | `sudo apt install python3.12 python3.12-venv` |

> Con otras versiones (por ejemplo 3.14), pip resuelve el `requirements.txt` sin conflictos,
> pero el curso sólo se grabó y probó con 3.12. Usa 3.12 para evitar sorpresas.

### ¿Importa el número de parche (3.12.0 vs 3.12.7)?

No. Los metadatos de los notebooks de lección dicen `3.12.7`, pero cualquier **3.12.x**
sirve:

- En `3.12.x`, lo que define el lenguaje y la compatibilidad de las librerías es **`3.12`**.
  El último número (`.0`, `.7`, …) sólo trae correcciones de errores y de seguridad, sin
  cambios en el lenguaje ni en la librería estándar.
- Las librerías se publican para "3.12" en general, así que pip instala exactamente las
  versiones de la grabación.
- Los notebooks de los labs se crearon con 3.13 y sólo exigen Python 3.10+: el simulador
  usa únicamente la librería estándar.

Verificado con **3.12.0**: `requirements.txt` instala las versiones de la grabación, una
llamada real a la API funciona desde `.venv`, el kernel de Jupyter arranca en VS Code y los
10 labs pasan sus 30 checks con `.venv-labs`.

**Recomendación opcional:** 3.12.0 fue la primera versión de la serie (octubre 2023) y le
faltan casi dos años de correcciones de seguridad. Para el curso da igual. Si usas esa
instalación en otros proyectos, actualízala a la **última 3.12.x**:

1. Descarga el instalador de la última 3.12 desde [python.org](https://www.python.org/downloads/)
   y ejecútalo. Detecta la 3.12 instalada y la actualiza en la misma carpeta. Mantén
   **desmarcado** *Add python.exe to PATH*.
2. Actualiza los entornos para que usen el Python nuevo. Se conservan las librerías y los
   `sitecustomize.py`:

   ```
   py -3.12 -m venv --upgrade .venv
   py -3.12 -m venv --upgrade .venv-labs
   ```

   En macOS/Linux: `python3.12 -m venv --upgrade .venv` (e igual para `.venv-labs`).
3. Comprueba con el [comando de verificación](#4-verificar-el-entorno) que `.venv` muestra
   la versión nueva.

### Si ya usas otra versión de Python en otros proyectos

Instalar 3.12 **no tiene por qué** cambiar tu Python por defecto. Las versiones conviven, y
cada proyecto usa la versión con la que se creó su `.venv`.

#### Windows

Hay dos mecanismos que deciden qué Python se ejecuta:

| Comando | Qué versión usa | ¿Cambia al instalar 3.12? |
|---|---|---|
| `py` (lanzador) | La **más nueva** instalada (marcada con `*` en `py -0`) | No, si ya tienes una versión más nueva (p. ej. 3.14) |
| `py -3.12` | 3.12, explícitamente | — |
| `python` | El primer `python.exe` que encuentra en el `PATH` | **Puede cambiar** si al instalar marcas *Add python.exe to PATH* |

Para instalar 3.12 sin tocar tu versión principal:

1. Descarga el instalador de 3.12 desde [python.org](https://www.python.org/downloads/).
2. **No marques** *Add python.exe to PATH*.
3. Deja marcada la opción del lanzador **py launcher**.
4. Comprueba que nada cambió:

   ```powershell
   python --version   # debe seguir mostrando tu versión principal (p. ej. 3.14.x)
   py -0              # debe listar 3.12 y tu versión principal, con * en esta última
   ```

5. Crea el entorno de este proyecto pidiendo 3.12 explícitamente:
   `py -3.12 -m venv .venv` (paso 2).

> `winget install Python.Python.3.12` no muestra la casilla de *PATH*. Si lo usas, ejecuta
> después las dos comprobaciones del punto 4.

**Si `python --version` ya muestra 3.12 por error:** abre *Configuración → Aplicaciones →
Python 3.12 → Modificar*, entra en *Advanced Options* y desmarca *Add Python to environment
variables*. También puedes quitar a mano del `PATH` (*Editar las variables de entorno*) las
rutas que terminan en `Python312\` y `Python312\Scripts\`. Abre una terminal nueva para que
el cambio se aplique.

#### macOS / Linux

- `brew install python@3.12` y `apt install python3.12` agregan el comando `python3.12`, pero
  **no** cambian `python3`. Comprueba con `python3 --version`.
- En Linux **no** cambies `python3` con `update-alternatives` ni con enlaces simbólicos: el
  sistema operativo depende de su propia versión y herramientas como `apt` pueden romperse.
- Usa siempre `python3.12 -m venv .venv` para este proyecto.

#### Por qué no se mezclan

El `.venv` guarda qué intérprete lo creó. Con el entorno activo, o llamando a
`.venv\Scripts\python.exe` / `.venv/bin/python`, siempre se ejecuta 3.12. Al salir con
`deactivate`, o en otra terminal, vuelves a tu versión principal.

| Contexto | Versión que se ejecuta |
|---|---|
| Terminal normal, `python` / `python3` | Tu versión principal |
| Este proyecto con `.venv` activo | 3.12 |
| Otros proyectos | La versión con la que se creó su propio `.venv` |

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

Sólo la necesitan los notebooks de lección. Los laboratorios de `labs/`, abiertos con el
kernel de `.venv-labs`, no la usan aunque exista.

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

### Red corporativa con inspección TLS (proxy)

**Cuándo aplica:** en una red de empresa (oficina o VPN), cualquier llamada a Claude
desde Python falla con:

```
APIConnectionError ... [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed:
self-signed certificate in certificate chain
```

**Por qué pasa:** el proxy corporativo intercepta las conexiones HTTPS y presenta un
certificado firmado por una autoridad interna. El sistema operativo (y el navegador)
confían en ella, pero el SDK de Anthropic usa su propia lista de certificados (`certifi`) y
la rechaza. `pip` sí funciona porque, desde la versión 24.2, usa los certificados del
sistema.

**Solución:** [`truststore`](https://pypi.org/project/truststore/) hace que Python use los
certificados del sistema operativo (almacén de Windows, Llavero de macOS o bundle del
sistema en Linux). **La verificación TLS sigue activa**: no uses `verify=False`.

1. Instálalo en el `.venv` (con el entorno activo):

   ```
   python -m pip install truststore
   ```

2. Actívalo para **todo** el entorno con un `sitecustomize.py`, que Python ejecuta al
   arrancar, incluido el kernel de Jupyter. Así no hay que tocar ningún notebook. Este
   comando lo crea en el `site-packages` del `.venv` y no sobrescribe uno existente
   (funciona en PowerShell, cmd, bash y zsh):

   ```
   python -c "import sysconfig, pathlib; p = pathlib.Path(sysconfig.get_paths()['purelib'], 'sitecustomize.py'); print('ya existe, revisalo antes:', p) if p.exists() else (p.write_text('try:\n    import truststore\n    truststore.inject_into_ssl()\nexcept ImportError:\n    pass\n'), print('creado:', p))"
   ```

   El archivo resultante contiene:

   ```python
   try:
       import truststore
       truststore.inject_into_ssl()
   except ImportError:
       pass
   ```

3. Verifica **sin gastar crédito**: la petición va sin key, así que la API responde 401.
   Si llega a responder, la conexión TLS funciona.

   ```
   python -c "import ssl, truststore, httpx; print('truststore activo:', ssl.SSLContext is truststore.SSLContext); print('HTTP', httpx.get('https://api.anthropic.com/v1/models').status_code, '(401 esperado)')"
   ```

   Salida esperada: `truststore activo: True` y `HTTP 401 (401 esperado)`. Si ves
   `True` pero sigue el error SSL, el certificado corporativo no está en el almacén del
   sistema: pide a Mesa de Ayuda / Seguridad TI el certificado raíz (`.pem`) y define
   `SSL_CERT_FILE` apuntando a él.

Notas:

- Fuera de la red corporativa no molesta: el almacén del sistema también incluye las
  autoridades públicas.
- `sitecustomize.py` vive dentro de `.venv/`, que no se versiona. **Si borras o recreas el
  `.venv`, repite los pasos 1 y 2.**
- No hace falta en los labs con simulador: no hacen llamadas de red.
- `truststore` no está en `requirements.txt` porque sólo se necesita en redes con
  inspección TLS.

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

1. Instala las extensiones de Microsoft **Python** (`ms-python.python`) y **Jupyter**
   (`ms-toolsai.jupyter`).
   *Jupyter Renderers* (`ms-toolsai.jupyter-renderers`) **no** la reemplaza: sólo muestra
   salidas. Sin `ms-toolsai.jupyter`, *Select Kernel* no ofrece *Python Environments*.
   Para comprobarlo: `code --list-extensions | findstr jupyter` (Windows) o
   `code --list-extensions | grep jupyter` (macOS/Linux).
2. Abre la carpeta del repositorio (**File → Open Folder**), no un notebook suelto. Si
   aparece **Restricted Mode**, pulsa *Manage → Trust*: en modo restringido las extensiones
   no detectan entornos.
3. Abre un notebook y pulsa **Select Kernel** (arriba a la derecha).
4. Elige **Python Environments… → `.venv` (Python 3.12.x)**. La primera vez VS Code puede
   tardar unos 20-30 s en terminar de descubrir entornos; si no aparece, espera y vuelve a
   abrir el selector.
5. **No elijas "Create Python Environment"**, aunque aparezca como recomendada: crea otro
   entorno (o propone reemplazar el `.venv`) sin las versiones de `requirements.txt` ni el
   ajuste de [red corporativa](#red-corporativa-con-inspección-tls-proxy).

#### Si `.venv` no aparece en la lista

**Causa más común:** tu configuración de usuario de VS Code fija un intérprete global para
todas las carpetas. Revísalo en `settings.json` de usuario (**Ctrl/Cmd+Shift+P → Preferences:
Open User Settings (JSON)**). Si ves algo como:

```json
"python.defaultInterpreterPath": "c:\\Program Files\\Python314\\python.exe"
```

no lo borres (lo usan tus otros proyectos). Sobrescríbelo **sólo para este repositorio**
creando `.vscode/settings.json` en la raíz:

| Sistema | Contenido de `.vscode/settings.json` |
|---|---|
| Windows | `{ "python.defaultInterpreterPath": "${workspaceFolder}\\.venv\\Scripts\\python.exe" }` |
| macOS / Linux | `{ "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python" }` |

Luego **Ctrl/Cmd+Shift+P → Developer: Reload Window** y vuelve a *Select Kernel*.

- `.vscode/` está en `.gitignore`: la ruta depende del sistema operativo, así que cada uno
  crea el suyo.
- Para confirmar que la causa era esa, revisa **View → Output → Python Environments**: si
  aparece `Found venv environment: .venv` pero después `defaultInterpreterPath: ...Python314...`,
  VS Code detectó el entorno pero el ajuste global lo tapaba.

**Alternativa sin archivo:** **Ctrl/Cmd+Shift+P → Python: Select Interpreter → Enter
interpreter path** y apunta a `.venv\Scripts\python.exe` (Windows) o `.venv/bin/python`
(macOS/Linux); luego vuelve a *Select Kernel*.

**Si nada de lo anterior funciona:** actualiza la extensión Jupyter desde la vista de
Extensiones; una versión muy anterior a la de la extensión Python puede no listar los
entornos del proyecto.

### Comprobar el kernel desde un notebook

Ejecuta en una celda:

```python
import sys; print(sys.executable)
```

La ruta debe contener `.venv`.

### Laboratorios (entorno `.venv-labs`)

Los labs están pensados para correr **offline y gratis** con el simulador
`shopassist_lab.py`. Para que eso ocurra de verdad hay que abrirlos con un entorno aparte.

**Por qué no sirve `.venv`:**

- `shopassist_lab` sólo instala el simulador si el paquete `anthropic` **no** está
  instalado. En `.venv` sí está, así que `from anthropic import Anthropic` carga el SDK real y
  `SHOPASSIST_LAB_FORCE_SIM` no tiene efecto.
- Con `python-dotenv` instalado, `load_dotenv()` busca el `.env` **subiendo por las carpetas
  padre** y encuentra el de la raíz del repositorio.
- Resultado: un lab abierto con `.venv` hace **llamadas reales, con costo**. Sin key, falla
  con `Could not resolve authentication method` en vez de usar el simulador. Además, los
  `check(...)` esperan las respuestas del simulador.

**Configuración (una sola vez), desde la raíz del repositorio:**

1. Crea el entorno e instala **sólo** Jupyter (no uses `requirements.txt` aquí):

   | Sistema | Comandos |
   |---|---|
   | Windows | `py -3.12 -m venv .venv-labs`<br>`.venv-labs\Scripts\python.exe -m pip install jupyter` |
   | macOS / Linux | `python3.12 -m venv .venv-labs`<br>`.venv-labs/bin/python -m pip install jupyter` |

2. Fija el simulador para todo el entorno. Este comando crea un `sitecustomize.py` que
   define `SHOPASSIST_LAB_FORCE_SIM=1` cada vez que arranca Python. Así, aunque una
   `ANTHROPIC_API_KEY` llegue al kernel (por ejemplo, si VS Code carga el `.env`), el
   simulador la ignora. Usa el Python de `.venv-labs`, en Windows
   `.venv-labs\Scripts\python.exe` y en macOS/Linux `.venv-labs/bin/python`:

   ```
   .venv-labs\Scripts\python.exe -c "import sysconfig, pathlib; p = pathlib.Path(sysconfig.get_paths()['purelib'], 'sitecustomize.py'); print('ya existe, revisalo antes:', p) if p.exists() else (p.write_text('import os\nos.environ[\'SHOPASSIST_LAB_FORCE_SIM\'] = \'1\'\n'), print('creado:', p))"
   ```

3. Registra el kernel con un nombre inconfundible:

   ```
   .venv-labs\Scripts\python.exe -m ipykernel install --user --name ccar-f-labs --display-name "Python (CCAR-F labs - simulador)"
   ```

4. Comprueba que el entorno **no** tiene el SDK. Debe imprimir `False False 1`:

   ```
   .venv-labs\Scripts\python.exe -c "import importlib.util as u, os; print(bool(u.find_spec('anthropic')), bool(u.find_spec('dotenv')), os.environ.get('SHOPASSIST_LAB_FORCE_SIM'))"
   ```

`.venv-labs/` está en `.gitignore`. Si lo borras o recreas, repite los pasos 1 a 3.

**Qué kernel usar en cada caso:**

| Material | Kernel | Resultado |
|---|---|---|
| `01_*` … `13_*` (lecciones) | `.venv` (Python 3.12) | API real con tu key; consume crédito |
| `labs/lab_*/notebook.ipynb` y `solution.ipynb` | **Python (CCAR-F labs - simulador)** | Simulador; offline y gratis |

**Abrir un lab:**

- **VS Code:** abre `labs/lab_<tema>/notebook.ipynb` → *Select Kernel* → **Jupyter Kernel…**
  → **Python (CCAR-F labs - simulador)** (o *Python Environments* → `.venv-labs`). VS Code
  recuerda la elección por notebook. El kernel arranca en la carpeta del notebook, así que
  encuentra `shopassist_lab.py`. Como `.vscode/settings.json` recomienda `.venv`, **fíjate
  de elegir el de labs**.
- **Jupyter Lab:** ábrelo desde la carpeta del lab, porque importa `shopassist_lab.py`:

  ```bash
  cd labs/lab_first_request
  ../../.venv-labs/Scripts/python.exe -m jupyter lab     # macOS/Linux: ../../.venv-labs/bin/python
  ```

**Cómo saber que estás en el simulador:** la celda de setup debe imprimir
`[shopassist_lab] SIMULATED MODE - replies are canned fixtures, not Claude.` y los `id` de las
respuestas empiezan con `msg_sim_`. Si ves `LIVE MODE`, o ningún banner en un lab que llama
al cliente, el kernel es el equivocado: cámbialo antes de seguir. `lab_validation` no
muestra banner porque no hace llamadas.

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
| Tras instalar 3.12, `python --version` en otros proyectos muestra 3.12 | Se marcó *Add python.exe to PATH* y 3.12 quedó antes en el `PATH` | Desmarca esa opción desde *Modificar* o quita las rutas `Python312` del `PATH` (ver [otra versión de Python](#si-ya-usas-otra-versión-de-python-en-otros-proyectos)) |
| `No Python at '...'` o `Unable to create process` | El `.venv` se creó con un Python que se desinstaló o movió | Borra `.venv` y vuelve a crearlo (paso 2) |
| `SSL: CERTIFICATE_VERIFY_FAILED` o timeouts en `pip install` | Proxy o inspección TLS de la red corporativa (poco común con pip 24.2+, que ya usa los certificados del sistema) | Actualiza pip (`python -m pip install --upgrade pip`); si persiste, configura el proxy (`pip install --proxy http://usuario:clave@proxy:puerto ...`) o el certificado corporativo (`pip config set global.cert <ruta-al-certificado.pem>`); consulta a Mesa de Ayuda / Seguridad TI |
| `APIConnectionError` con `CERTIFICATE_VERIFY_FAILED` / `self-signed certificate in certificate chain` al **llamar** a la API | El proxy corporativo intercepta `api.anthropic.com` y el SDK no confía en su certificado | `truststore` + `sitecustomize.py` (ver [Red corporativa con inspección TLS](#red-corporativa-con-inspección-tls-proxy)); como alternativa, `SSL_CERT_FILE` apuntando al certificado corporativo |
| `ModuleNotFoundError: No module named 'shopassist_lab'` | El lab se abrió desde otra carpeta | Abre Jupyter desde `labs/lab_<tema>/` (paso 5, *Laboratorios*) |
| Un lab muestra `LIVE MODE` o no muestra el banner `SIMULATED MODE` | El lab se abrió con el kernel `.venv`, que tiene `anthropic` instalado y lee el `.env` de la raíz | Cambia el kernel a **Python (CCAR-F labs - simulador)** (ver [Laboratorios](#laboratorios-entorno-venv-labs)). Quitar la key o definir `SHOPASSIST_LAB_FORCE_SIM` **no** basta con `.venv` |
| Un lab falla con `Could not resolve authentication method` | Mismo caso: kernel `.venv` sin key disponible, así que el SDK real no se autentica | Cambia el kernel a **Python (CCAR-F labs - simulador)** |
| `No module named 'ipykernel'` al elegir `.venv-labs` | Se instaló el entorno sin Jupyter | `.venv-labs\Scripts\python.exe -m pip install jupyter` |
| En VS Code, *Select Kernel* no muestra *Python Environments* | Falta la extensión `ms-toolsai.jupyter` (sólo está *Jupyter Renderers*) o la carpeta está en *Restricted Mode* | Instala **Jupyter** de Microsoft, confía en la carpeta y recarga la ventana (paso 5, opción B) |
| En VS Code, *Python Environments* lista sólo los Python globales y no `.venv` | `python.defaultInterpreterPath` en la configuración de usuario fija otro intérprete | Crea `.vscode/settings.json` apuntando al `.venv` (ver [Si `.venv` no aparece en la lista](#si-venv-no-aparece-en-la-lista)) |
| `NameError: name '...' is not defined` | Se ejecutó una celda antes que las anteriores | Kernel → **Restart & Run All**, o ejecuta las celdas en orden desde la primera |
| El notebook 01 reinstala paquetes con `%pip install` | Esa celda existe para quien empieza sin entorno | Con el `.venv` ya preparado se puede saltar; si la ejecutas, reinicia el kernel después |

---

## 7. Checklist rápido

- [ ] `py -0` / `python3.12 --version` muestra 3.12
- [ ] `python --version` fuera del proyecto sigue mostrando tu versión principal
- [ ] `.venv` creado y activo (el prompt muestra `(.venv)`)
- [ ] `python -m pip install -r requirements.txt` sin errores
- [ ] El comando de verificación muestra `anthropic 0.111.0` y `sys.prefix` en `.venv`
- [ ] `.env` en la raíz con `ANTHROPIC_API_KEY` (sólo notebooks de lección)
- [ ] Kernel del notebook apuntando a `.venv` (`sys.executable`)
- [ ] `.venv-labs` creado sólo con Jupyter, `sitecustomize.py` con `SHOPASSIST_LAB_FORCE_SIM`
      y kernel **Python (CCAR-F labs - simulador)** registrado
- [ ] Un lab abierto con ese kernel muestra `SIMULATED MODE`
- [ ] En red corporativa: `truststore` instalado, `sitecustomize.py` creado y la prueba TLS
      devuelve `HTTP 401`
