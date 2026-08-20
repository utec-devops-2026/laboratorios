# Lab: Control de versiones con Git en **GitHub**

**Duración estimada:** 45--60 min\
**Nivel:** Principiante--Intermedio\
**Contexto:** Realizarás clonación, commits, exploración de
historial, trabajo con ramas, **pull requests**, **branch protection
rules** y **tags**. Usaremos comandos de **Git** en terminal.

---

## Objetivos de aprendizaje

Al finalizar, podrás:

- Clonar un repositorio existente desde GitHub.
- Guardar trabajo local con **commits** y **staging**.
- Revisar historial y comparar cambios.
- Trabajar con ramas desde terminal.
- Crear **Pull Requests (PRs)** con políticas de protección (revisiones requeridas).
- Aplicar **tags** y crear un **release**.

---

## Requisitos del laboratorio

- Navegador compatible (Chrome/Firefox/Safari/Microsoft Edge).
- Cuenta de **GitHub** (gratuita u organización).
- **Git** instalado.
- Terminal (zsh, bash, PowerShell, etc.).

---

## Repositorio de práctica

Usaremos **eShopOnWeb**

- URL de origen: `https://github.com/MicrosoftLearning/eShopOnWeb.git`

> Puedes **importar** el repo a tu usuario/organización en GitHub (crea
> tu propia copia administrable) o **forkear**.

---

## Ejercicio 0: Configurar prerrequisitos

### Tarea 1: Configurar Git

1.  Abre una **terminal**.
2.  Configura tu helper de credenciales y tu identidad:

    - **Windows** (Git Credential Manager):

      ```bash
      git config --global credential.helper manager-core
      git config --global user.name "Tu Nombre"
      git config --global user.email tu-email@example.com
      ```

    - **macOS** (Keychain) o Linux (GCM si instalado):

      ```bash
      git config --global credential.helper osxkeychain   # macOS
      git config --global user.name "Tu Nombre"
      git config --global user.email tu-email@example.com
      # Linux con GCM (si lo usas):
      # git config --global credential.helper manager-core
      ```

3.  Verifica la configuración:

    ```bash
    git config --list --show-origin
    ```

### Tarea 2: Crear repositorio en **GitHub** por importación

1.  Entra a **github.com** con tu cuenta.
2.  En la esquina superior derecha, haz clic en **+** → **Import
    repository**.
3.  En **Old repository's clone URL**, pega:
    `https://github.com/MicrosoftLearning/eShopOnWeb.git`.
4.  En **Owner**, elige tu usuario u organización.\
5.  En **Repository Name**, escribe: `eshoponweb` (o similar).\
6.  Mantén **Public/Private** según prefieras y pulsa **Begin import**.
7.  Al terminar, verás tu **nuevo repo** `owner/eshoponweb`.

> Estructura relevante del repo: `.github/` (workflows),
> `.devcontainer/`, `infra/` (Bicep/ARM), `src/` (.NET 8 Web).

![Import](images/import.png)

### Tarea 3: Establecer `main` como rama por defecto (si aplica)

- GitHub crea `main` por defecto; si no, ve a **Settings** →
  **Branches** → **Default branch** → **main**.

![branch_main](images/branch_main.png)

## Ejercicio 1: Clonar un repositorio existente

### Tarea 1: Clonar con Git

1.  En GitHub, abre tu repo `owner/eshoponweb` y haz clic en **Code** →
    pestaña **HTTPS** → **Copy** (URL de clonación).

2.  En la terminal, navega a la carpeta donde quieres clonar el repositorio:

    ```bash
    # Crear directorio Git si no existe
    mkdir -p ~/Git
    cd ~/Git
    ```

3.  Clona el repositorio:

    ```bash
    git clone https://github.com/TU_USUARIO/eshoponweb.git
    ```

4.  Navega al directorio del repositorio:

    ```bash
    cd eshoponweb
    ```

5.  Verifica el estado del repositorio:

    ```bash
    git status
    git branch -a
    ```

---

## Ejercicio 2: Guardar trabajo con commits

### Conceptos

- **Working Directory** → **Staging Area (Index)** → **Commit local**.
- Un commit tiene: cambios, referencia(s) padre y un **mensaje
  claro**.

### Tarea 1: Primer commit

1.  Abre el archivo `src/Web/Program.cs` con tu editor favorito:

    ```bash
    # Puedes usar nano, vim, code, etc.
    nano src/Web/Program.cs
    # o si tienes VS Code instalado:
    code src/Web/Program.cs
    ```

2.  En la **primera línea**, agrega un comentario:

    ```csharp
    // My first change
    ```

3.  Guarda el archivo y cierra el editor.

4.  Verifica los cambios:

    ```bash
    git status
    git diff src/Web/Program.cs
    ```

5.  Añade el archivo al staging area y realiza el commit:

    ```bash
    git add src/Web/Program.cs
    git commit -m "feat(web): My commit"
    ```

6.  Sube los cambios al repositorio remoto:

    ```bash
    git push origin main
    ```

### Tarea 2: Revisar commits en GitHub

1.  Ve al repo en GitHub → pestaña **Commits** (o **History** en la
    vista de archivos).
2.  Verifica que tu commit esté arriba.

### Tarea 3: Staging selectivo

1.  Edita `src/Web/Program.cs` y cambia el comentario a:

    ```csharp
    // My second change
    ```

2.  Edita `src/Web/Constants.cs` y agrega:

    ```csharp
    // My third change
    ```

3.  Verifica los cambios en ambos archivos:

    ```bash
    git status
    git diff src/Web/Program.cs
    git diff src/Web/Constants.cs
    ```

4.  Añade solo `Program.cs` al staging area:

    ```bash
    git add src/Web/Program.cs
    ```

5.  Verifica que solo `Program.cs` está en staging:

    ```bash
    git status
    ```

6.  Realiza el commit de los cambios staged:

    ```bash
    git commit -m "chore(web): Added comments"
    ```

7.  Sube los cambios:

    ```bash
    git push origin main
    ```

    Observa que `Constants.cs` sigue sin commitear (cambio pendiente).

8.  Puedes ver las diferencias usando:

    ```bash
    git diff src/Web/Constants.cs
    ```

---

## Ejercicio 3: Revisar historial

### Tarea 1: Comparar archivos y explorar historial

1.  Verifica los cambios pendientes en `Constants.cs`:

    ```bash
    git diff src/Web/Constants.cs
    ```

2.  Explora el historial de commits:

    ```bash
    # Ver log con formato compacto
    git log --oneline

    # Ver log con más detalles
    git log --graph --pretty=format:'%h -%d %s (%cr) <%an>' --abbrev-commit

    # Ver cambios en un commit específico
    git show <commit-hash>
    ```

3.  Para ver el estado del repositorio en un commit específico:

    ```bash
    # Ver archivos en un commit específico
    git ls-tree -r <commit-hash>

    # Ver el contenido de un archivo en un commit específico
    git show <commit-hash>:src/Web/Program.cs
    ```

4.  Compara diferencias entre commits:

    ```bash
    # Diferencias entre los últimos 2 commits
    git diff HEAD~1 HEAD

    # Diferencias entre commits específicos
    git diff <commit-hash1> <commit-hash2>
    ```

5.  En GitHub, abre **Commits** y entra a un commit para ver **Browse
    files** (estado del repo en ese punto) o **Changes** para ver el
    patch.

> Recuerda que la línea temporal no siempre es estrictamente cronológica
> por **merges**; piensa en **deltas** entre commits.

---

## Ejercicio 4: Trabajar con ramas y Pull Requests

### Tarea 1: Crear una nueva rama local

1.  Verifica en qué rama estás actualmente:

    ```bash
    git branch
    git status
    ```

2.  Crea y cambia a una nueva rama llamada `develop`:

    ```bash
    git checkout -b develop
    ```

    O alternativamente (Git 2.23+):

    ```bash
    git switch -c develop
    ```

3.  Verifica que estás en la rama `develop`:

    ```bash
    git branch
    ```

### Tarea 2: Publicar y gestionar ramas

1.  Publica la rama `develop` en GitHub:

    ```bash
    git push -u origin develop
    ```

    El flag `-u` establece `origin/develop` como rama upstream para futuros push/pull.

2.  Verifica las ramas locales y remotas:

    ```bash
    # Ramas locales
    git branch

    # Ramas remotas
    git branch -r

    # Todas las ramas
    git branch -a
    ```

3.  En GitHub, ve a **Branches** (pestaña del repo) y verifica que
    exista `develop`.

### Tarea 3: Definir un **Ruleset** (políticas sobre `main`)

> GitHub reemplazó las clásicas **Branch protection rules** por
> **Rulesets**. Al entrar a **Settings** → **Branches** verás un banner
> indicando que **Rulesets** es la forma recomendada actual; úsalo en
> lugar de "Add rule".

1.  En GitHub, entra a **Settings** → **Rules** → **Rulesets** → **New
    ruleset** → **New branch ruleset**.
2.  En **Ruleset Name**, escribe un nombre descriptivo, p. ej. `Protect
    main`.
3.  En **Enforcement status**, cambia de **Disabled** (valor por
    defecto) a **Active**.

    > ⚠️ Si dejas el ruleset en **Disabled**, las reglas no se aplican y
    > `main` queda sin protección aunque las hayas configurado.

4.  En **Bypass list**, déjala **vacía**. Una bypass list vacía
    significa que nadie —ni siquiera los administradores— puede
    saltarse las reglas.
5.  En **Target branches**, clic en **Add target** → **Include default
    branch** (o **Include by pattern** y escribe `main`).

<img src="images/ruleset_target_branches.png" width="700"/>

6.  En **Rules**, activa:
    - ✅ **Require a pull request before merging** (mismas sub-opciones
      que antes: **Require approvals**, **Dismiss stale approvals**,
      **Require review from Code Owners**, etc.)
    - (Opcional) **Require approvals** → **1**
    - (Opcional) **Require status checks to pass**
    - (Opcional) **Require signed commits**
    - (Opcional) **Require linear history**
    - **Block force pushes** (viene marcado por defecto; bloquea los
      `push --force` sobre `main`)
    - **Restrict deletions** (viene marcado por defecto; impide borrar
      la rama `main`)
    - (Opcional) **Restrict updates** (antes "Lock branch")

<img src="images/ruleset_require_pr.png" width="700"/>

7.  Guarda con **Create**.

> La configuración de **Default branch** no forma parte del ruleset:
> sigue estando en **Settings** → **General** → sección **Default
> branch**.

> GitHub no exige "issue vinculado" de forma nativa. Simularemos el
> control con una **Issue** y el texto **`Closes #<número>`** en el PR
> para enlazar trazabilidad.

### Tarea 4: Probar la política de `main` (bloqueo de commits directos)

1.  Cambia a la rama main:

    ```bash
    git checkout main
    ```

2.  Intenta editar un archivo directamente en `main`:

    ```bash
    echo "// Testing main branch policy" >> src/Web/Program.cs
    git add src/Web/Program.cs
    git commit -m "feat(web): Testing direct commit to main"
    git push origin main
    ```

3.  Si la política está correctamente configurada, el push será **rechazado**.

![reject_main](images/reject_main.png)

4.  Deshaz los cambios locales:

    ```bash
    git reset --hard HEAD~1
    ```

### Tarea 5: Crear una plantilla de Pull Request

Con `main` ya protegido por el **Ruleset**, cualquier cambio —incluida
la plantilla de PR— debe entrar por una rama y un PR.

1.  Actualiza `main` y crea una rama para la plantilla:

    ```bash
    git checkout main
    git pull origin main
    git checkout -b chore/pr-template
    ```

2.  Crea la carpeta `.github` y el archivo `pull_request_template.md`:

    ```bash
    mkdir -p .github
    nano .github/pull_request_template.md
    # o si tienes VS Code instalado:
    # code .github/pull_request_template.md
    ```

3.  Pega el siguiente contenido:

    ````markdown
    ## Descripción

    <!-- Explica qué cambia este PR, por qué y qué riesgo toca -->

    ## Tipo de cambio

    - [ ] 🐛 Bug fix
    - [ ] ✨ Nueva funcionalidad
    - [ ] ♻️ Refactor
    - [ ] 📝 Documentación
    - [ ] 🔐 Seguridad
    - [ ] 🧪 Tests / automatización

    ## Checklist DoD

    ### Alcance funcional
    - [ ] Los criterios de aceptación / comportamiento esperado están cubiertos
    - [ ] Validé el flujo afectado localmente
    - [ ] Si hubo cambio visible para usuario, adjunté evidencia (captura, video o notas)

    ### Código
    - [ ] El código sigue las convenciones del proyecto
    - [ ] No dejé código comentado, `console.log` de debug ni artefactos temporales
    - [ ] Los nombres y mensajes son claros para negocio y equipo técnico

    ### Calidad y tests
    - [ ] Agregué o actualicé tests para los cambios introducidos
    - [ ] `npm run test:coverage` pasa con cobertura mínima esperada (≥ 80%) si aplica
    - [ ] `npm run test:bdd` pasa si el cambio afecta comportamiento E2E / Gherkin
    - [ ] Cubrí happy path y al menos un caso de error o borde relevante

    ### Seguridad
    - [ ] No expongo secretos, tokens, contraseñas, PII ni datos sensibles
    - [ ] Validé inputs, manejo de errores y permisos según corresponda
    - [ ] Revisé dependencias o configuración nueva por riesgos de seguridad si aplica

    ### Documentación y mantenimiento
    - [ ] Actualicé documentación, feature files o steps cuando el comportamiento cambió
    - [ ] El PR está acotado, con contexto suficiente para revisión

    ## Notas para la persona revisora

    <!-- Riesgos, decisiones, cosas a mirar con más atención -->
    ````

4.  Guarda, añade y commitea:

    ```bash
    git add .github/pull_request_template.md
    git commit -m "chore: add PR template"
    git push -u origin chore/pr-template
    ```

5.  En GitHub, crea el PR contra `main` (banner "Compare & pull
    request"), **Approve** y **Merge**.

6.  Verifica que funcionó: al abrir un **nuevo PR** contra `main` (lo
    harás en la siguiente tarea), la descripción vendrá **precargada**
    con esta plantilla.

### Tarea 6: Crear una **Issue**, hacer cambio en y abrir PR siguiendo GitFlow

1.  En GitHub, pestaña **Issues** → **New issue**.
    - **Title:** `Testing my first PR`\
    - **Submit new issue** (anota el número, p. ej. `#1`).
2.  Cambia a rama **feature/new-feature-1** que nace de la rama develop:

    ```bash
    git checkout feature/new-feature-1 
    ```

    Edita `src/Web/Program.cs` y agrega:

    ```csharp
    // Testing my first PR
    ```

    Realiza **Commit** en la rama `feature/new-feature-1`:

    ```bash
    git add src/Web/Program.cs
    git commit -m "feat(web): test my first PR - Closes #1"
    git push origin feature/new-feature-1
    ```

3.  GitHub mostrará un banner "Compare & pull request". Clic en **Create
    pull request**.

![PR create](images/PR-create.png)

4.  En la **descripción del PR**, escribe `Closes #1` para **vincular**
    la Issue.
5.  Crea el PR contra `develop` y luego crear PR contra `main`.
6.  En la pestaña **Files changed**, revisa el diff. Añade comentarios
    si deseas.
7.  En la pestaña **Conversation**, **Approve** (si eres revisor) y
    luego **Merge**.

    - **Merge method:** _Create a merge commit_ (equivalente a "Merge
      no fast-forward").
    - Activa "**Delete branch**" tras el merge.

8.  Actualiza tu repositorio local tras el merge:

    ```bash
    git checkout main
    git pull origin main
    ```

    También puedes eliminar la rama local `feature/new-feature-1`:

    ```bash
    git branch -d feature/new-feature-1
    ```

> Al hacer merge a `main`, la Issue `#1` se cierra automáticamente por el
> `Closes #1`.

---

## Ejercicio 5: Aplicar **tags** y crear un **release**

### Tarea 1: Crear tag `v1.1.0-beta`

1.  Asegúrate de estar en la rama `main` y con los últimos cambios:

    ```bash
    git checkout main
    git pull origin main
    ```

2.  Crea el tag localmente:

    ```bash
    git tag -a v1.1.0-beta -m "Beta release v1.1.0"
    ```

3.  Sube el tag al repositorio remoto:

    ```bash
    git push origin v1.1.0-beta
    ```

4.  En GitHub, ve a **Releases** → **Create a new release**.

![release create](images/release-create.png)

5.  En **Choose a tag**, selecciona `v1.1.0-beta` (que ya existe).

6.  **Release title:** `Beta release v1.1.0`.

7.  (Opcional) Marca como **pre-release**.

8.  Clic en **Publish release**.

![release-created.png](images/release-created.png)

> Alternativamente, puedes crear el tag directamente desde GitHub:

    ```bash
    # Ver todos los tags
    git tag

    # Ver información del tag
    git show v1.1.0-beta
    ```

> Esto crea un **tag** y un **release** asociado al último commit de
> `main`.

---

## Comandos útiles de Git (referencia rápida)

```bash
# Ver configuración
git config --list --show-origin
git config --global user.name "Tu Nombre"
git config --global user.email "tu-email@example.com"

# Clonar repositorio
git clone https://github.com/usuario/repo.git
cd repo

# Verificar estado
git status
git branch
git branch -a                # ver todas las ramas (locales y remotas)

# Cambiar/crear rama
git checkout -b feature/new-feature-1          # crear y moverse
git checkout main            # volver a main
git switch -c feature/new-feature-1            # alternativa moderna para crear rama

# Staging/commit
git add archivo.txt          # agregar archivo específico
git add .                    # agregar todos los cambios
# Commits con Conventional Commits
git commit -m "feat(web): descripción breve del cambio"
git commit -m "fix(api): corrige error en endpoint"
git commit -m "chore: actualiza dependencias"
git commit -am "refactor(core): mejora estructura de datos"

# Sincronizar
git pull                     # traer cambios del remoto
git push                     # subir cambios
git push -u origin feature/new-feature-1        # publicar rama por 1ª vez

# Historial y diferencias
git log --oneline            # historial compacto
git log --graph --pretty=format:'%h -%d %s (%cr) <%an>' --abbrev-commit
git diff                     # diferencias no staged
git diff --staged            # diferencias staged
git show <commit-hash>       # ver commit específico

# Tags
git tag                      # listar tags
git tag -a v1.1.0-beta -m "mensaje"  # crear tag anotado
git push origin v1.1.0-beta # subir tag
git show v1.1.0-beta        # ver información del tag

# Gestión de ramas
git branch -d feature/new-feature-1            # eliminar rama local
git push origin --delete feature/new-feature-1 # eliminar rama remota
```

---

## Criterios de éxito (Checklist)

- [ ] Repo importado a **GitHub**.
- [ ] Clonación local exitosa usando **Git**.
- [ ] Al menos **2 commits** en `main` (uno con **staging
      selectivo**).
- [ ] Historial revisado con comandos Git y en GitHub.
- [ ] Rama `feature/new-feature-1` creada, publicada y PR creado.
- [ ] **Ruleset** en `main` (Active, bypass list vacía) con **1 aprobación** requerida.
- [ ] Plantilla de PR (`.github/pull_request_template.md`) creada, mergeada a `main` y visible al abrir un PR.
- [ ] PR aprobado, merge realizado y rama `feature/new-feature-1` eliminada.
- [ ] **Tag** `v1.1.0-beta` y **release** publicado.

---

## Extensiones y extras (opcional)

- **GitHub CLI** (`gh`): Herramienta de línea de comandos para gestionar PRs/Issues:

  ```bash
  # Instalar GitHub CLI (macOS)
  brew install gh

  # Autenticarse
  gh auth login

  # Crear PR desde línea de comandos
  gh pr create --title "Mi PR" --body "Closes #1"

  # Ver PRs
  gh pr list
  ```

- **Codeowners**: añade `CODEOWNERS` para requerir aprobaciones de
  equipos/usuarios específicos.
- **Checks**: integra **GitHub Actions** (linting/tests) y marca
  **Required status checks** en la regla de branch.

---

## Solución de problemas frecuentes

- **403 al hacer push**: revisa que el remoto sea
  `https://github.com/<owner>/<repo>.git` y que el helper de
  credenciales esté guardando el token correcto. Puedes verificar con:
  ```bash
  git remote -v
  git config --list | grep credential
  ```
- **No aparece "Compare & pull request"**: crea el PR manualmente:
  **Pull requests** → **New pull request** (base: `main`, compare:
  `develop`) o usa GitHub CLI:
  ```bash
  gh pr create --base main --head develop
  ```
- **No puedo commitear en `main`**: es lo esperado si la protección
  está activa; usa una rama y PR.
- **El tag no aparece**: si lo creaste localmente, recuerda
  `git push origin <tag>` o usa la UI de **Releases**.
- **Error "divergent branches"**: sincroniza antes de hacer push:
  ```bash
  git pull origin main
  git push origin main
  ```
- **Conflictos de merge**: resuelve manualmente y luego:
  ```bash
  git add archivo-resuelto.txt
  git commit -m "fix: Resolve merge conflict"
  ```
