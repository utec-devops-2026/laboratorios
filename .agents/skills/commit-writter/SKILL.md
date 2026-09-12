---
name: staged-commit-message-writer
description: >
  Genera mensajes de commit usando Conventional Commits a partir de cambios en stage.
  Usame cuando el usuario pida "escribe un mensaje de commit", "commitea mis cambios",
  "resume mi diff staged", "genera commit message" o "crear mensaje para git commit".
  Analiza primero los archivos staged y luego produce asunto, cuerpo opcional y footer.
---

**NUNCA incluyas `Co-Authored-By`, `Generated-By`, `AI-assisted` ni ninguna línea de atribución de IA en ningún mensaje de commit.**
El output debe ser exclusivamente el mensaje de commit, sin metadatos de herramientas.


## Objetivo

Generar un mensaje de commit claro, consistente y accionable usando solamente los cambios en staging.

## Cuándo usar este skill

Activa este skill cuando el usuario pida cualquiera de estas intenciones:

- Crear mensaje de commit
- Resumir cambios staged
- Preparar texto para `git commit -m`
- Proponer Conventional Commit para cambios listos

## Flujo de trabajo

1. Verifica que existan cambios en stage.
2. Si no hay cambios staged, responde con una salida breve: `No hay cambios en staging para commitear.`
3. Si hay cambios, inspecciona:
- Lista de archivos staged
- Tipo de cambio por archivo (A, M, D, R)
- Resumen del diff staged
4. Clasifica el cambio principal y selecciona el tipo de Conventional Commit.
5. Propone un scope corto cuando sea evidente (por ejemplo: `tests`, `api`, `docs`, `ci`).
6. Construye el mensaje final en formato estándar.

## Comandos de referencia

Usa comandos equivalentes a estos para analizar stage:

```bash
git diff --cached --name-status
git diff --cached --stat
git diff --cached
```

Si necesitas verificar ausencia de cambios staged:

```bash
git diff --cached --quiet
```

## Formato de salida

Entrega siempre este formato:

```text
type(scope): short imperative summary

optional body with key changes

optional footer
```

Si no hay scope claro, usa:

```text
type: short imperative summary
```

## Tipos permitidos

- `feat`: nueva funcionalidad
- `fix`: corrección de bug
- `docs`: documentación
- `refactor`: cambios internos sin modificar comportamiento esperado
- `test`: creación o ajuste de pruebas
- `chore`: mantenimiento, tooling, configuración
- `ci`: cambios de integración/automatización

## Reglas estrictas

1. Usa voz imperativa en la primera línea (ejemplo: `add`, `fix`, `update`).
2. Máximo 72 caracteres en el asunto.
3. No uses frases vagas como `update stuff` o `fix things`.
4. No inventes cambios que no estén en stage.
5. Prioriza el cambio dominante; evita mezclar múltiples temas sin relación.
6. Si hay cambios heterogéneos, sugiere separar commits y proporciona una opción por grupo.
7. Devuelve directamente el resultado sin preguntas innecesarias.
8. **Nunca incluyas `Co-Authored-By` ni ninguna línea de atribución de IA en el mensaje de commit.**

## Criterios para elegir `type`

- Si cambia comportamiento visible para usuario: `feat` o `fix`
- Si solo cambia pruebas: `test`
- Si solo cambia documentación: `docs`
- Si cambia estructura interna sin alterar comportamiento: `refactor`
- Si es configuración, dependencias o tareas operativas: `chore` o `ci`

## Manejo de cambios mixtos

Cuando haya grupos no relacionados, usa este formato:

```text
Se detectaron cambios staged en grupos distintos. Recomendación: separar commits.

Opción 1:
<commit message 1>

Opción 2:
<commit message 2>
```

## Ejemplos

Correcto:

```text
test(calculadora): add multiply unit tests for edge cases
```

```text
fix(calculadora): handle divide by zero with ValueError
```

Incorrecto:

- `updated calculator`
- `fix: fixed stuff`
- `changes in files`

## Plantilla de respuesta final

Usa una de estas dos salidas:

1. Caso estándar:

```text
type(scope): summary

- bullet 1
- bullet 2
```

2. Sin staged:

```text
No hay cambios en staging para commitear.
```
