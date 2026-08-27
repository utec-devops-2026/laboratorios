# Laboratorio: Creando tus propios Skills en Claude Code

**Duración estimada:** 30–45 min  
**Nivel:** Intermedio  
**Objetivo:** Aprender a crear, instalar y mejorar skills personalizados para Claude Code

---

## Objetivos de aprendizaje

Al finalizar, podrás:

- Explicar qué es un **skill** y cuándo conviene crearlo.
- Crear la estructura de un skill en `~/.claude/skills/`.
- Escribir un **frontmatter** con una descripción efectiva que active el skill correctamente.
- Redactar instrucciones con formato de output, reglas y ejemplos.
- Probar un skill por **invocación directa** y **lenguaje natural**.
- Diagnosticar y corregir problemas comunes: *undertriggering*, *format drift* y *scope creep*.
- Portar el skill a **GitHub Copilot** como custom agent en la carpeta `agents`.

---

## ¿Qué es un Skill?

Un skill es un archivo markdown (`SKILL.md`) que se carga automáticamente en el contexto de Claude Code cuando lo necesitas. Contiene frontmatter YAML con metadatos y las instrucciones que el agente debe seguir.

**Estructura básica de un skill:**

```
~/.claude/skills/
└── nombre-del-skill/
    └── SKILL.md
```

**Anatomía del archivo SKILL.md:**

```markdown
---
name: nombre-del-skill
description: Descripción clara que define cuándo se activa este skill
---

## Instrucciones

Aquí van las instrucciones para el agente...
```

---

## ¿Cuándo usar un Skill?

**Buenos candidatos para un skill:**
- Mensajes de commit con formato consistente
- Descripciones de Pull Requests
- Code reviews estandarizados
- Entradas de changelog

**Malos candidatos para un skill:**
- Solicitudes abiertas como "ayúdame a pensar esto"
- Tareas que requieren mucho contexto variable

---

## Ejercicio 1: Crear la estructura del skill

Crea el directorio e instala tu primer skill:

```bash
mkdir -p ~/.claude/skills/commit-message-writer
touch ~/.claude/skills/commit-message-writer/SKILL.md
```

---

## Ejercicio 2: Escribir el frontmatter y la descripción

El campo `description` es **crítico**: el agente decide si cargar o no el skill basándose únicamente en este campo.

**Descripción débil (evitar):**
```yaml
---
name: commit-message-writer
description: Genera mensajes de commit
---
```

**Descripción efectiva (usar):**
```yaml
---
name: commit-message-writer
description: >
  Genera mensajes de commit con formato Conventional Commits.
  Úsame cuando quieras escribir un commit, hacer commit de tus cambios,
  o resumir tu diff staged. Produce una línea de asunto, cuerpo opcional
  y footer. Se activa con frases como "escribe un mensaje de commit",
  "commitea mis cambios" o "resume mi diff staged".
---
```

**Reglas para una buena descripción:**
1. Especifica el tipo de output (ej: "una línea de asunto + cuerpo")
2. Lista frases de activación explícitas
3. Sé ligeramente imperativo — no esperes que el usuario adivine cómo invocarte

---

## Ejercicio 3: Escribir las instrucciones del skill

Crea el archivo `~/.claude/skills/commit-message-writer/SKILL.md` con el siguiente contenido:

```markdown
---
name: commit-message-writer
description: >
  Genera mensajes de commit con formato Conventional Commits.
  Úsame cuando quieras escribir un commit, hacer commit de tus cambios,
  o resumir tu diff staged. Se activa con frases como "escribe un mensaje
  de commit", "commitea mis cambios" o "resume mi diff staged".
---

## Formato de output

Usa la especificación Conventional Commits:


type(scope): descripción corta

[cuerpo opcional]

[footer opcional]
```

## Tipos permitidos
```
- `feat` — nueva funcionalidad
- `fix` — corrección de bug
- `docs` — cambios en documentación
- `refactor` — refactorización sin cambio de comportamiento
- `test` — agregar o corregir tests
- `chore` — tareas de mantenimiento

## Reglas

1. La descripción corta debe estar en modo imperativo (ej: "add", no "added")
2. Máximo 72 caracteres en la primera línea
3. Genera el output directamente, sin hacer preguntas
4. Nunca uses lenguaje vago como "update stuff" o "fix things"
5. Si hay cambios en archivos no relacionados, agrupa por tipo de cambio
```

---

## Ejercicio 4: Probar el skill

Una vez instalado, puedes invocar el skill de dos formas:

**Invocación directa:**
```
/commit-message-writer
```

**Lenguaje natural:**
```
escribe un mensaje de commit para mis cambios staged
commitea mis cambios
resume mi diff staged
```

**Casos de prueba para validar:**

| Escenario | Resultado esperado |
|---|---|
| Sin cambios staged | El skill indica que no hay nada staged |
| Cambios en múltiples archivos no relacionados | Agrupa o separa por tipo |
| Distintas formas de pedir el commit | El skill se activa correctamente |

---

## Ejercicio 5: Mejorar el skill con el tiempo

Los skills se refinan iterativamente. Estos son los problemas más comunes y cómo resolverlos:

### Problema: Undertriggering (el skill no se activa)

El agente no reconoce que debe usar el skill.

**Solución:** Agrega más frases de activación al campo `description`:

```yaml
description: >
  ... Se activa también con "haz commit", "crea un commit message",
  "genera el mensaje para git commit"...
```

### Problema: Format drift (el output no respeta el formato)

El agente produce output con estructura inconsistente.

**Solución:** Agrega contraejemplos explícitos en las instrucciones:

```markdown
## Ejemplos

Correcto:
feat(auth): add JWT token refresh endpoint

Incorrecto:
- "Updated the auth stuff" (vago)
- "feat: added new feature for authentication" (tiempo pasado, sin scope)
```

### Problema: Scope creep (el skill hace demasiado)

El skill intenta resolver múltiples problemas y se vuelve confuso.

**Solución:** Divide en skills separados. Un skill = una responsabilidad.

---

## Ejercicio 6: Versión para GitHub Copilot (carpeta `agents`)

GitHub Copilot CLI soporta dos mecanismos de extensión:

1. **Skills** (`SKILL.md`) — el mismo formato de este laboratorio. Copilot los descubre desde varias rutas:

| Alcance | Rutas |
|---|---|
| Personal (usuario) | `~/.copilot/skills/` o `~/.agents/skills/` |
| Proyecto | `.github/skills/`, `.agents/skills/` o `.claude/skills/` |

   > Nota: `~/.agents/skills/` es la carpeta del estándar abierto **Agent Skills**, compartida entre herramientas. Y como Copilot también lee `.claude/skills/` dentro del proyecto, un skill de Claude Code funciona en Copilot **sin ningún cambio**. Verifica con:
   > ```bash
   > copilot skill list
   > ```

2. **Custom agents** — un archivo markdown por agente dentro de la carpeta `agents`:

| Alcance | Ruta |
|---|---|
| Global (usuario) | `~/.copilot/agents/` |
| Por repositorio | `.github/agents/` |

Los siguientes pasos crean la versión **custom agent**.

**Diferencias clave con Claude Code:**

- No hay subcarpeta por skill: el archivo se llama directamente `nombre-del-agente.md`
- El frontmatter usa `name`, `description` y opcionalmente `tools`
- El cuerpo del markdown actúa como prompt del agente

### Paso 1: Crear el archivo

```bash
mkdir -p ~/.copilot/agents
touch ~/.copilot/agents/commit-message-writer.md
```

### Paso 2: Escribir el agente

Crea `~/.copilot/agents/commit-message-writer.md` con el siguiente contenido:

```markdown
---
name: commit-message-writer
description: >
  Genera mensajes de commit con formato Conventional Commits.
  Úsame cuando quieras escribir un commit, hacer commit de tus cambios,
  o resumir tu diff staged. Se activa con frases como "escribe un mensaje
  de commit", "commitea mis cambios" o "resume mi diff staged".
---

## Formato de output

Usa la especificación Conventional Commits:

type(scope): descripción corta

[cuerpo opcional]

[footer opcional]

## Tipos permitidos

- `feat` — nueva funcionalidad
- `fix` — corrección de bug
- `docs` — cambios en documentación
- `refactor` — refactorización sin cambio de comportamiento
- `test` — agregar o corregir tests
- `chore` — tareas de mantenimiento

## Reglas

1. La descripción corta debe estar en modo imperativo (ej: "add", no "added")
2. Máximo 72 caracteres en la primera línea
3. Genera el output directamente, sin hacer preguntas
4. Nunca uses lenguaje vago como "update stuff" o "fix things"
5. Si hay cambios en archivos no relacionados, agrupa por tipo de cambio
```

### Paso 3: Probar el agente en Copilot CLI

**Listar agentes disponibles (modo interactivo):**
```
/agents
```

**Invocación directa:**
```bash
copilot --agent commit-message-writer -p "genera el mensaje de commit para mis cambios staged"
```

**Casos de prueba:** los mismos del Ejercicio 4 aplican sin cambios.

---

## Compatibilidad entre plataformas

Las instrucciones del skill se reutilizan entre agentes de IA; en la mayoría de casos solo cambia la ruta:

| Herramienta | Directorio | Formato |
|---|---|---|
| Claude Code | `~/.claude/skills/nombre-del-skill/` | `SKILL.md` |
| GitHub Copilot (skills) | `~/.copilot/skills/` o `~/.agents/skills/` | `SKILL.md` |
| GitHub Copilot (custom agents) | `~/.copilot/agents/` o `.github/agents/` | `nombre-del-agente.md` |
| Cursor | `~/.cursor/skills/` o `~/.agents/skills/` | `SKILL.md` |
| Gemini CLI | `~/.gemini/skills/` o `~/.agents/skills/` | `SKILL.md` |
| Antigravity (workspace) | `.agents/skills/` o `.agent/skills/` (en la raíz del proyecto) | `SKILL.md` |
| Antigravity (global) | `~/.gemini/config/skills/` | `SKILL.md` |

La carpeta `~/.agents/skills/` (estándar abierto **Agent Skills**) permite compartir un mismo skill **sin duplicarlo**. En la práctica, **solo funciona de forma compartida en Cursor y Gemini CLI**: si dejas el skill ahí, esas dos herramientas lo descubren. **Antigravity no lee `~/.agents/skills/`**; usa su propia estructura (`~/.gemini/config/skills/` a nivel global, y `.agents/skills/` o `.agent/skills/` en el proyecto). Si quieres el mismo skill en Antigravity, cópialo o enlázalo a esas rutas.

---

## Resumen: Propiedades de un skill efectivo

1. **Encapsula un workflow repetible** — no tareas únicas o muy variables
2. **Tiene un trigger claro y específico** — la descripción es la clave
3. **Produce un output de formato consistente** — define estructura, campos y límites
4. **Genera output directamente** — sin hacer preguntas innecesarias al usuario

¡Buen trabajo!