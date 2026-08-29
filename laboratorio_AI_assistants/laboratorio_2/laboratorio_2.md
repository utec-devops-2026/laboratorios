# Laboratorio: Configurando MCPs — Pruebas E2E con Playwright MCP

**Duración estimada:** 45–60 min
**Nivel:** Intermedio
**Objetivo:** Configurar un servidor MCP (Model Context Protocol) en tu asistente de IA y usarlo para generar pruebas E2E a partir de criterios de aceptación, validando los flujos contra un sitio web real.

---

## Objetivos de aprendizaje

Al finalizar, podrás:

- Explicar qué es **MCP** y qué problema resuelve.
- Configurar un servidor MCP desde el archivo de configuración (`mcp.json`, `.mcp.json`) en distintos asistentes de IA.
- Verificar que el asistente detecta las herramientas (tools) que expone el servidor.
- Usar **Playwright MCP** para que el agente explore un sitio web real: navegar, llenar formularios, capturar el DOM.
- Generar pruebas E2E a partir de **criterios de aceptación**, con selectores y mensajes reales extraídos del sitio (no inventados).
- Ejecutar las pruebas generadas y cerrar el ciclo: criterio → exploración → test → verde.

---

## ¿Qué es MCP?

**Model Context Protocol (MCP)** es un protocolo abierto que estandariza cómo los asistentes de IA se conectan con herramientas y datos externos. Un **servidor MCP** expone capacidades (tools, resources, prompts) que el agente puede invocar durante la conversación.

```
┌─────────────────┐        ┌──────────────────┐        ┌─────────────┐
│  Asistente IA   │  MCP   │  Servidor MCP    │        │  Navegador  │
│  (Copilot,      │◄──────►│  Playwright MCP  │◄──────►│  Chromium   │
│   Claude Code)  │        │  (npx)           │        │             │
└─────────────────┘        └──────────────────┘        └─────────────┘
```

**Sin MCP:** el agente escribe tests E2E "a ciegas" → inventa selectores y mensajes que no existen → los tests fallan.

**Con Playwright MCP:** el agente navega el sitio REAL antes de escribir código. Ve el DOM verdadero, extrae selectores reales y valida que cada paso del flujo es posible.

---

## El sitio de práctica

Usaremos un sitio público diseñado justamente para practicar automatización:

**https://practice.expandtesting.com/login**

- Página de login con campos **Username** y **Password** y botón **Login**.
- Credenciales válidas: **`practice`** / **`SuperSecretPassword!`** (aparecen en la misma página).
- Con credenciales correctas redirige al **área segura** (`/secure`) con un mensaje de éxito y botón **Logout**.
- Con credenciales incorrectas muestra un mensaje de error y permanece en el login.

No necesitas levantar nada en local: solo Node.js 18+ instalado (`node --version`) y conexión a internet.

> 📂 Crea una carpeta de trabajo para este laboratorio (por ejemplo `lab-mcp-playwright/`) y ábrela como workspace en tu editor: es la "raíz del proyecto" a la que se refieren las rutas de configuración de abajo.

---

## Ejercicio 1: Configurar Playwright MCP desde el archivo de configuración

Playwright MCP es un servidor oficial de Microsoft que le da al agente un navegador controlable: abrir páginas, hacer clic, llenar formularios, tomar snapshots del DOM y screenshots. No requiere API key.

### Opción A — VS Code / GitHub Copilot

Crea (o edita) el archivo `.vscode/mcp.json` en la raíz del proyecto:

```json
{
  "servers": {
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"],
      "type": "stdio"
    }
  }
}
```

> 💡 También puedes configurarlo globalmente (para todos tus proyectos) desde la paleta de comandos: **MCP: Open User Configuration**.

**Verificación:** abre el Chat de Copilot en modo **Agent**, haz clic en el ícono de herramientas 🔧 y confirma que aparecen las tools de `playwright` (por ejemplo `browser_navigate`, `browser_click`, `browser_snapshot`).

### Opción B — Claude Code

Claude Code ofrece **dos formas** de configurar un servidor MCP:

**Forma 1 — Comando `claude mcp add` (recomendada):**

```bash
claude mcp add playwright -- npx -y @playwright/mcp@latest
```

Todo lo que va después de `--` es el comando que levanta el servidor. El flag `-s` controla el **scope** (alcance) de la configuración:

| Scope | Comando | Dónde queda | Cuándo usarlo |
|---|---|---|---|
| `local` (default) | `claude mcp add playwright -- npx ...` | `~/.claude.json` (solo este proyecto, privado) | Probar un server sin afectar al equipo |
| `project` | `claude mcp add playwright -s project -- npx ...` | `.mcp.json` en la raíz (versionado en git) | Compartir la config con todo el equipo |
| `user` | `claude mcp add playwright -s user -- npx ...` | `~/.claude.json` (todos tus proyectos) | Servers de uso general, como Playwright |

**Forma 2 — Editar `.mcp.json` directamente:**

Crea el archivo `.mcp.json` en la raíz del proyecto (equivale al scope `project`: queda versionado y compartido con el equipo):

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"]
    }
  }
}
```

**Comandos útiles de gestión:**

```bash
claude mcp list              # lista servers y su estado de conexión
claude mcp get playwright    # detalle de un server
claude mcp remove playwright # elimina un server
```

**Verificación:** ejecuta `/mcp` dentro de Claude Code. Debe listar `playwright` como conectado y mostrar sus tools.

### Opción C — GitHub Copilot CLI

Copilot CLI (el comando `copilot` en la terminal) también ofrece **dos formas**:

**Forma 1 — Comando interactivo `/mcp add`:**

Dentro de una sesión de `copilot`, escribe `/mcp add` y completa el formulario:

- **Server name:** `playwright`
- **Server type:** `Local`
- **Command:** `npx`
- **Arguments:** `-y, @playwright/mcp@latest`
- **Tools:** `*` (habilita todas las tools del server)

**Forma 2 — Editar `~/.copilot/mcp-config.json` directamente:**

```json
{
  "mcpServers": {
    "playwright": {
      "type": "local",
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"],
      "tools": ["*"]
    }
  }
}
```

> ⚠️ A diferencia de Claude Code, Copilot CLI requiere el campo `"tools"`: define qué tools del server quedan habilitadas. `["*"]` las habilita todas.

**Verificación:** dentro de `copilot`, ejecuta `/mcp show`. Debe listar `playwright` con sus tools disponibles.

### Otros asistentes

| Asistente | Archivo de configuración | Clave raíz |
|---|---|---|
| Cursor | `.cursor/mcp.json` | `mcpServers` |
| Gemini CLI | `~/.gemini/settings.json` | `mcpServers` |
| Codex CLI | `~/.codex/config.toml` | `[mcp_servers.playwright]` |

El bloque del servidor es el mismo en todos: comando `npx`, argumentos `["-y", "@playwright/mcp@latest"]`.

---

## Ejercicio 2: Primer contacto — el agente explora el sitio

Antes de generar tests, comprueba que el agente realmente controla el navegador. Escribe este prompt en modo agente:

> Usa Playwright MCP: navega a https://practice.expandtesting.com/login, toma un snapshot de la página y descríbeme qué elementos interactivos ves (inputs, botones, links).

**Resultado esperado:** se abre una ventana de Chromium, el agente navega al login y responde con los elementos reales de la página: campo **Username**, campo **Password**, botón **Login**, y las credenciales de demo visibles en el texto.

Ahora pídele que ejecute el flujo completo:

> Inicia sesión con practice / SuperSecretPassword! y dime exactamente qué mensaje aparece y a qué URL te redirige.

El agente debe reportar la redirección al **área segura** (`/secure`), el mensaje de éxito y el botón **Logout**.

> 🎯 **Punto clave:** el agente no está adivinando. Cada elemento y cada mensaje que reporta salió de un snapshot real del DOM. Esos son los selectores y textos que usaremos en los tests.

---

## Ejercicio 3: De criterios de aceptación a pruebas E2E

Este es el flujo completo que vas a ejecutar:

```
Criterios de aceptación (HU)
   ↓ 1. El agente EXPLORA el sitio con Playwright MCP
        (ejecuta cada flujo manualmente, anota selectores y mensajes reales)
   ↓ 2. El agente escribe los specs de Playwright con esos selectores
   ↓ 3. El agente ejecuta `npx playwright test` y corrige hasta verde
```

### Historia de usuario

> **Como** usuario registrado,
> **quiero** iniciar y cerrar sesión en el área segura,
> **para** acceder a mi contenido de forma protegida.

### Criterios de aceptación

1. **Login exitoso:** Dado un usuario registrado, cuando ingresa `practice` / `SuperSecretPassword!`, entonces es redirigido al área segura y ve un mensaje de éxito.
2. **Username inválido:** Dado un usuario, cuando ingresa un username incorrecto, entonces permanece en el login y ve un mensaje de error de usuario.
3. **Password inválido:** Dado un usuario, cuando ingresa el username correcto pero un password incorrecto, entonces permanece en el login y ve un mensaje de error de contraseña.
4. **Logout:** Dado un usuario autenticado en el área segura, cuando hace clic en Logout, entonces regresa al login.

> ✍️ Nota que los criterios NO dicen cuál es el texto exacto de los mensajes de error. Ese detalle vive en la aplicación real — y el agente lo va a descubrir explorando, no inventando.

### Prompt para el agente

Copia este prompt (ajusta según tu asistente):

```
Tengo estos criterios de aceptación para https://practice.expandtesting.com/login:

1. Login exitoso: con practice / SuperSecretPassword! soy redirigido al área
   segura y veo un mensaje de éxito.
2. Username inválido: con un username incorrecto permanezco en el login y veo
   un mensaje de error de usuario.
3. Password inválido: con username correcto y password incorrecto permanezco
   en el login y veo un mensaje de error de contraseña.
4. Logout: desde el área segura, al hacer clic en Logout regreso al login.

Paso 1: usa Playwright MCP para ejecutar CADA criterio manualmente en el
navegador. Anota los selectores reales (roles, labels, ids) y el TEXTO EXACTO
de cada mensaje de éxito o error. No inventes selectores ni mensajes.

Paso 2: crea un proyecto de pruebas en esta carpeta:
- npm init -y e instala @playwright/test como devDependency
- genera playwright.config.ts con baseURL https://practice.expandtesting.com
- escribe un spec por criterio en tests/, usando SOLO los selectores y textos
  verificados en el paso 1

Paso 3: ejecuta npx playwright test y corrige los tests hasta que pasen todos.
```

### Qué observar durante la ejecución

- En el **Paso 1**, el navegador se abre y ejecuta cada flujo: verás al agente loguearse, fallar a propósito con credenciales inválidas, y capturar el texto exacto de cada mensaje. Está *verificando* que los criterios son automatizables antes de escribir una sola línea de código.
- En el **Paso 2**, revisa los specs generados: los selectores deben coincidir con lo que el agente reportó (por ejemplo `page.locator('#username')`, `getByRole('button', { name: 'Login' })`) y los `expect` deben usar los mensajes literales capturados en la exploración.
- En el **Paso 3**, si algún test falla, el agente puede volver a explorar el sitio con MCP para diagnosticar — esa es la ventaja frente a un asistente sin acceso al navegador.

**Resultado esperado:**

```
Running 4 tests using 1 worker

  ✓ login exitoso redirige al área segura
  ✓ username inválido muestra error de usuario
  ✓ password inválido muestra error de contraseña
  ✓ logout regresa al login

  4 passed
```

---

## Ejercicio 4: El experimento de control — ¿qué aporta realmente el MCP?

Para dimensionar el valor del MCP, repite la generación **sin** usarlo:

1. Abre una conversación nueva y desactiva las tools de Playwright MCP (o usa un asistente sin el servidor configurado).
2. Pega los mismos criterios de aceptación y pide los specs directamente, sin exploración previa.
3. Ejecuta `npx playwright test` con los tests generados.

**Compara:**

| | Sin MCP | Con MCP |
|---|---|---|
| Selectores | Adivinados a partir del texto del criterio | Extraídos del DOM real |
| Mensajes de error | Inventados o copiados de sitios parecidos | Texto literal capturado del sitio |
| Primer run | Suele fallar (textos/elementos no coinciden) | Suele pasar |
| Iteraciones para llegar a verde | Varias, guiadas por ti | El agente se autocorrige explorando |

---

## Pipeline completo con Skills + MCP

Si completaste el laboratorio anterior de Skills, ya tienes las piezas para el pipeline BDD completo:

```
Historia de Usuario
   ↓  /gherkin-writer        (skill: HU → login.feature)
   ↓  /playwright-bdd        (skill: .feature → page objects + steps)
   ↓  Playwright MCP         (el agente valida selectores y mensajes contra
                              el sitio real y corrige los steps generados)
   ↓  npm run test:bdd       (tests E2E en verde)
```

Prompt sugerido:

> Toma la historia de usuario del Ejercicio 3 y genera tests/features/login.feature con /gherkin-writer. Luego usa /playwright-bdd para generar los step definitions, revisa con Playwright MCP que cada step sea ejecutable en https://practice.expandtesting.com/login, corrige los selectores si no coinciden con el DOM real, y ejecuta los tests hasta que pasen.

Con esto combinas los tres mecanismos de extensión de un asistente de IA: **skills** (conocimiento reutilizable), **MCP** (acceso a herramientas externas) y el **agente** que orquesta ambos.

---

## Troubleshooting

| Problema | Causa probable | Solución |
|---|---|---|
| El asistente no lista las tools de `playwright` | El archivo de configuración tiene otro nombre/clave raíz | Revisa la tabla del Ejercicio 1: VS Code usa `servers`, el resto `mcpServers` |
| `npx` se queda colgado al iniciar el server | Primera descarga del paquete | Espera; o pre-instala con `npx -y @playwright/mcp@latest --version` |
| El navegador no abre | Falta el browser de Playwright | `npx playwright install chromium` |
| Los tests fallan por elementos de publicidad o banners | El sitio de práctica muestra ads | Usa selectores específicos del formulario (ids, roles), nunca posiciones ni `nth()` |
| Tests intermitentes por lentitud del sitio | Sitio público compartido | Aumenta `timeout` en `playwright.config.ts` o agrega `await expect(...).toBeVisible()` en vez de esperas fijas |

---

## Conclusión

Has aprendido a:

- Configurar un servidor MCP desde el archivo de configuración en distintos asistentes de IA.
- Verificar la conexión y descubrir las tools que expone un servidor.
- Usar Playwright MCP para que el agente explore un sitio real y extraiga selectores y mensajes verdaderos.
- Convertir criterios de aceptación en pruebas E2E ejecutables, validadas contra el sitio.
- Medir el valor del MCP comparando la generación con y sin acceso al navegador.

El patrón que practicaste — *el agente verifica contra la realidad antes de generar código* — aplica mucho más allá del testing: bases de datos, APIs, documentación viva. MCP es la puerta estándar hacia ese contexto. ¡Sigue explorando! 🚀
