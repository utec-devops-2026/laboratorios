# Laboratorio: Generación de mensajes de commit desde cambios staged

**Duración estimada:** 15-20 min
**Nivel:** Básico
**Objetivo:** Generar mensajes de commit claros con Conventional Commits a partir de los cambios que estén en staging.

---

## Prerrequisitos

- Tener Git instalado y configurado.
- Trabajar en un repositorio Git con al menos un archivo modificado.
- Tener disponible la skill `staged-commit-message-writer`.

---

## Ejercicio 1: Preparar cambios en staging

1. Revisa el estado del repositorio:

   ```bash
   git status
   ```

2. Agrega al área de staging únicamente los cambios que deseas incluir en el commit:

   ```bash
   git add <archivo>
   ```

3. Confirma qué cambios están preparados:

   ```bash
   git diff --cached --name-status
   git diff --cached --stat
   ```

La skill analiza exclusivamente los cambios staged. Las modificaciones que no se hayan agregado con `git add` no forman parte de la propuesta.

---

## Ejercicio 2: Generar el mensaje de commit

Invoca la skill desde Copilot CLI:

```text
/staged-commit-message-writer
```

La skill realiza este flujo:

1. Verifica si existen cambios en staging.
2. Inspecciona los archivos, el tipo de cambio (`A`, `M`, `D` o `R`) y el diff.
3. Identifica el cambio dominante.
4. Selecciona el tipo de Conventional Commit y un scope breve si es evidente.
5. Devuelve un mensaje listo para usar, sin metadatos ni atribuciones de IA.

Si no hay cambios staged, la salida será:

```text
No hay cambios en staging para commitear.
```

---

## Ejercicio 3: Interpretar el formato generado

El mensaje sigue esta estructura:

```text
type(scope): short imperative summary

optional body with key changes

optional footer
```

Cuando no sea claro un scope, se omite:

```text
type: short imperative summary
```

El asunto debe usar voz imperativa, describir un cambio concreto y tener como máximo 72 caracteres.

### Tipos permitidos

- `feat`: nueva funcionalidad.
- `fix`: corrección de un error.
- `docs`: cambios de documentación.
- `refactor`: cambios internos sin modificar el comportamiento esperado.
- `test`: creación o ajuste de pruebas.
- `chore`: mantenimiento, tooling o configuración.
- `ci`: integración continua o automatización.

---

## Ejercicio 4: Aplicar el mensaje propuesto

Copia el mensaje generado y crea el commit:

```bash
git commit -m "type(scope): short imperative summary"
```

Para un mensaje con cuerpo, usa un editor o varias opciones `-m`:

```bash
git commit -m "type(scope): short imperative summary" \
  -m "Describe the key changes."
```

Ejemplos de resultados válidos:

```text
test(calculadora): add multiply unit tests for edge cases
```

```text
fix(calculadora): handle divide by zero with ValueError
```

Evita mensajes ambiguos como:

```text
updated calculator
fix: fixed stuff
changes in files
```

---

## Ejercicio 5: Manejar cambios no relacionados

Si los cambios staged pertenecen a temas distintos, la skill recomienda separarlos en commits y propone una opción para cada grupo:

```text
Se detectaron cambios staged en grupos distintos. Recomendación: separar commits.

Opción 1:
<commit message 1>

Opción 2:
<commit message 2>
```

En ese caso, restaura el staging de forma selectiva y crea un commit por grupo:

```bash
git restore --staged <archivo>
git add <archivos-del-primer-grupo>
```

---

## Conclusión

Has aprendido a:

- Preparar cambios concretos en el área de staging.
- Generar mensajes de commit basados solo en esos cambios.
- Elegir tipos y scopes de Conventional Commits.
- Separar commits cuando los cambios no están relacionados.
