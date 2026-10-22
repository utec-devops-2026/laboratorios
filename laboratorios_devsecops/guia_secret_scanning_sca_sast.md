# Laboratorio: Análisis de Dependencias y Secret Scanning con Trivy, Semgrep, Gitleaks y TruffleHog

**Duración estimada:** 150–180 min  
**Nivel:** Intermedio / Avanzado  
**Contexto:** Este laboratorio amplía el flujo DevSecOps aplicado al proyecto `juice-shop-devsecops`. Aquí aprenderás a ejecutar análisis manuales de vulnerabilidades y exposición de secretos utilizando contenedores Docker con herramientas SCA (Software Composition Analysis) y secret scanning, para luego automatizar todo el proceso en GitLab CI/CD.

---

## 🎯 Objetivos de aprendizaje

- Detectar vulnerabilidades en dependencias con **Trivy** (SCA)
- Identificar vulnerabilidades en código fuente con **Semgrep** (SAST)
- Validar SAST en el IDE con **SonarQube for IDE** (shift-left)
- Detectar secretos y credenciales expuestas con **Gitleaks** y **TruffleHog**
- Prevenir secretos en commits futuros con **pre-commit hooks**
- Automatizar todos los escaneos en **GitLab CI/CD**

---

## 🧩 Requisitos previos

✅ Docker Desktop instalado  
✅ VS Code y Node.js ≥ 20 instalados (para SonarQube for IDE)  
✅ GitLab y acceso al proyecto `juice-shop-devsecops`  
✅ Conocimientos básicos de Node.js y dependencias  
✅ Conectividad a Internet para descargar imágenes Docker  

---

## 📁 Estructura base del proyecto

```
juice-shop-devsecops/
├── .gitlab-ci.yml            # pipeline (Parte 4)
├── .pre-commit-config.yaml   # hooks locales (Parte 5)
├── Dockerfile
├── package.json
├── package-lock.json         # generado en 2.1, necesario para Trivy
├── routes/  lib/  models/    # backend Express/TypeScript (código que analiza Semgrep y SonarQube)
├── frontend/                 # Angular
├── kubernetes/  iac/         # manifiestos y Terraform (no se usan en este lab)
├── reports/                  # salidas de los escaneos (local, no se commitea)
└── sbom/                     # SBOM CycloneDX (local, no se commitea)
```

> `reports/` y `sbom/` contienen los secretos que las herramientas detectan: los hooks de la Parte 5 bloquearán cualquier intento de commitearlos. Van en `.gitignore`; en GitLab se obtienen como **artefactos del pipeline** (Build → Artifacts).

---

## Parte 1: Configuración del entorno

### 1.1 Clonar el proyecto

```bash
git clone https://github.com/wils0n/juice-shop-devsecops
cd juice-shop-devsecops
mkdir -p reports
```

### 1.2 Apuntar remote a GitLab

El repo se clona desde GitHub, pero el pipeline corre en GitLab. Redirige el origin:

```bash
git remote set-url origin git@gitlab.com:<tu-usuario>/juice-shop-devsecops-20262.git

# Verificar
git remote -v
```

> Reemplaza `<tu-usuario>` por tu nombre de usuario en GitLab (ej. `pytuxi`).

### 1.3 Validar entorno Docker

```bash
docker version
docker images
```

### 1.4 Levantar app

```bash
docker-compose up -d
```

---

## Parte 2: Análisis con Trivy (SCA)

### 2.1 Ejecutar Trivy desde Docker

```bash
docker run --rm -v $(pwd):/src aquasec/trivy:latest fs /src   --format json --output /src/reports/trivy-report.json   --severity HIGH,CRITICAL
```

> **⚠️ Sin lockfile no hay SCA.** Trivy resuelve dependencias npm a partir de `package-lock.json`. El repo no lo incluye (`Number of language-specific files num=0` → 0 vulnerabilidades). Genéralo sin instalar `node_modules`:
>
> ```bash
> npm install --package-lock-only --ignore-scripts --no-audit --no-fund
> ```
>
> Vuelve a ejecutar Trivy: deberías ver ~100+ vulnerabilidades HIGH/CRITICAL en `package-lock.json`.

### 2.2 Reporte en formato tabla

```bash
docker run --rm -v $(pwd):/src aquasec/trivy:latest fs /src \
  --format table --severity HIGH,CRITICAL
```

### 2.3 Generar SBOM (CycloneDX)

```bash
docker run --rm -v $(pwd):/src aquasec/trivy:latest fs /src \
  --format cyclonedx --output /src/sbom/sbom.json
```

> **¿Qué es un SBOM?** Un *Software Bill of Materials* es un inventario exhaustivo de todos los componentes de software incluidos en un proyecto: librerías, versiones, licencias y dependencias transitivas.

> **¿Cuándo se usa en DevSecOps?**
>
> | Momento | Uso |
> |---|---|
> | **CI/CD — stage SCA** | El pipeline genera el SBOM como artefacto del build. Si hay un CVE crítico en alguna dependencia, el pipeline falla antes de llegar a producción. |

---

## Parte 3: Detección de secretos

### 3.1 Escaneo con Gitleaks

```bash
docker run --rm -it -v $(pwd):/src zricethezav/gitleaks detect \
  --source /src \
  --report-path /src/reports/gitleaks-report.json
```

### 3.2 Escaneo con TruffleHog

```bash
docker run --rm -it -v $(pwd):/src trufflesecurity/trufflehog:latest filesystem /src \
  --results=verified,unknown \
  --json > reports/trufflehog-report.json
```

> Ambas herramientas buscarán API keys, tokens, passwords o certificados mal protegidos dentro del código o el historial Git.

---

## Parte 3.5: Análisis SAST con Semgrep

### 3.5.1 Ejecutar Semgrep desde Docker

```bash
docker run --rm -v $(pwd):/src semgrep/semgrep semgrep \
  --config "p/ci" \
  --config "p/security-audit" \
  --config "p/javascript" \
  --json --output /src/reports/semgrep-report.json \
  /src
```

### 3.5.2 Reporte en formato texto (consola)

```bash
docker run --rm -v $(pwd):/src semgrep/semgrep semgrep \
  --config "p/ci" \
  --config "p/security-audit" \
  /src
```

### 3.5.3 Escaneo con regla específica

```bash
docker run --rm -v $(pwd):/src semgrep/semgrep semgrep \
  --config "p/javascript" \
  --severity ERROR \
  --json --output /src/reports/semgrep-js-report.json \
  /src
```

> **¿Qué analiza Semgrep?** Busca patrones inseguros en código fuente: inyecciones SQL, XSS, uso de funciones peligrosas, credenciales hardcodeadas y malas prácticas específicas por lenguaje. A diferencia de las herramientas de dependencias (SCA), Semgrep analiza el **código que tú escribes** (SAST).

---

## Parte 3.6: SAST en el IDE con SonarQube for IDE

Semgrep y el pipeline detectan problemas **después** de escribir el código. SonarQube for IDE (antes SonarLint) los muestra **mientras escribes**, directamente en VS Code: es la forma más temprana de "shift-left" para SAST.

> **Requisitos:** VS Code y Node.js ≥ 20.12 (o ≥ 22.11) instalados en tu máquina. La extensión descarga su propio JRE automáticamente.

### 3.6.1 Instalar la extensión

1. Abre VS Code → **Extensions** (`Ctrl+Shift+X` / `Cmd+Shift+X`)
2. Busca **SonarQube for IDE** (ID: `SonarSource.sonarlint-vscode`) e instálala
3. Abre la carpeta `juice-shop-devsecops` en VS Code

Si VS Code no encuentra Node.js, indícalo en `settings.json`:

```json
{
  "sonarlint.pathToNodeExecutable": "/ruta/a/node"
}
```

> Obtén la ruta con `which node` (macOS/Linux) o `where node` (Windows).
>
> **Si usas nvm o fnm (muy común):** VS Code abierto desde el Dock o el menú no hereda el PATH de tu terminal, así que la extensión no encuentra Node y **no analiza nada** (solo verás errores `ts(...)` de TypeScript, ninguno `sonarqube(...)`). Usa la ruta real de la versión, no la del shell:
>
> ```bash
> # nvm
> ls ~/.nvm/versions/node/           # ej. v22.11.0 → ~/.nvm/versions/node/v22.11.0/bin/node
> # fnm
> ls ~/.local/share/fnm/node-versions/   # ej. v22.22.2 → ~/.local/share/fnm/node-versions/v22.22.2/installation/bin/node
> ```
>
> Pon esa ruta en `sonarlint.pathToNodeExecutable` y recarga VS Code (**Developer: Reload Window**).

### 3.6.2 Analizar archivos vulnerables

SonarQube for IDE analiza **solo los archivos abiertos**, al abrirlos o guardarlos. No necesita token ni cuenta: el modo standalone funciona sin configuración. Abre estos archivos y observa los subrayados:

```
lib/insecurity.ts      # clave privada RSA (línea 23), MD5 (43), password fijo (44), Math.random (55), http:// (129-132)
routes/captcha.ts      # Math.random() para generar captchas
routes/b2bOrder.ts     # ejecución dinámica de código (vm)
server.ts              # directory listing en rutas estáticas
```

> **`routes/login.ts` no muestra hallazgos de SonarQube** aunque tiene la inyección SQL: la regla `S2077` no está activa en el perfil por defecto de TypeScript. Esa vulnerabilidad la detecta Semgrep (`express-sequelize-injection`, Parte 3.5). Buen ejemplo de por qué se combinan herramientas.

Los hallazgos aparecen en:

- Editor: subrayado ondulado sobre la línea afectada
- Panel **Problems** (`Ctrl+Shift+M` / `Cmd+Shift+M`), con prefijo `sonarqube(<regla>)`. Filtra escribiendo `sonarqube` en la caja del panel
- Vista **SONARQUBE** en la Activity Bar (icono de SonarQube)

> **Si no ves nada en `lib/insecurity.ts`:** los errores rojos de `import ... from 'express'` son de TypeScript por no tener `node_modules`, no de SonarQube. Revisa **Output** (`Ctrl+Shift+U` / `Cmd+Shift+U`) → selector **SonarQube for IDE**: si dice que no encuentra Node.js, configura `sonarlint.pathToNodeExecutable` (3.6.1) y recarga la ventana (**Developer: Reload Window**).

### 3.6.3 Entender y corregir un hallazgo

1. Pasa el cursor sobre una línea subrayada → **Quick Fix** (`Ctrl+.` / `Cmd+.`) → **Show rule description**
2. Lee la descripción: **por qué** es un riesgo, ejemplo *noncompliant* y *compliant*
3. Corrige el código y guarda: el hallazgo desaparece del panel **Problems**

Ejemplo para `lib/insecurity.ts` línea 43 (regla `S4790`, MD5 es un hash débil):

```ts
// antes
export const hash = (data: string) => crypto.createHash('md5').update(data).digest('hex')
// después
export const hash = (data: string) => crypto.createHash('sha256').update(data).digest('hex')
```

Al guardar, el subrayado y la entrada en **Problems** desaparecen. Revierte el cambio después de la captura (`git checkout HEAD -- lib/insecurity.ts`): Juice Shop usa MD5 a propósito para sus retos.

### 3.6.4 Personalizar reglas

En la Activity Bar → **SONARQUBE SETUP** → **RULES** puedes activar o desactivar reglas por lenguaje. También desde `settings.json`:

```json
{
  "sonarlint.rules": {
    "javascript:S1481": { "level": "off" },
    "typescript:S2077": { "level": "on" }
  }
}
```

> **Nota:** los **Security Hotspots** se muestran solo en **Connected Mode**. En SonarQube Community Build 26.x varias reglas antes clasificadas como hotspot (MD5 `S4790`, `Math.random()` `S2245`, HTTP en claro `S5332`, clave privada `S6706`) ya se reportan como **vulnerabilidades** y sí aparecen en modo standalone.

### 3.6.5 (Opcional) Connected Mode con SonarQube Community Build

Para aplicar el perfil de calidad del servidor, sincronizar issues y escanear **toda** la carpeta, conecta el IDE a un SonarQube local en Docker.

**a) Levantar SonarQube**

```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:community
```

Espera ~1–2 min hasta que `curl -s http://localhost:9000/api/system/status` devuelva `"status":"UP"`. Abre http://localhost:9000, entra con `admin` / `admin` y define la nueva contraseña cuando lo pida.

**b) Crear el proyecto**

1. **Projects → Create Project → Create a local project**
2. Project display name y Project key: `juice-shop-devsecops` → **Next**
3. Deja **Use the global setting** → **Create project**

**c) Generar el token (User Token)**

1. Clic en tu avatar (arriba a la derecha) → **My Account**
2. Pestaña **Security** → sección **Generate Tokens**
3. Name: `vscode-lab` · Type: **User Token** · Expires in: `30 days` → **Generate**
4. Copia el token **ahora**: SonarQube lo muestra una sola vez. Guárdalo fuera del repo (por ejemplo en `~/.sonar-token`), nunca en un archivo commiteado.

> **¿Por qué User Token?** Es el único tipo que acepta Connected Mode. Los *Project Analysis Token* y *Global Analysis Token* sirven para `sonar-scanner`, pero VS Code los rechaza.

Alternativa por terminal (misma cuenta `admin` y la contraseña que definiste):

```bash
curl -s -u admin:<tu-password> -X POST \
  "http://localhost:9000/api/user_tokens/generate?name=vscode-lab&type=USER_TOKEN" \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['token'])"
```

**d) Primer análisis con sonar-scanner (Docker)**

```bash
docker run --rm \
  -e SONAR_HOST_URL="http://host.docker.internal:9000" \
  -e SONAR_TOKEN="<tu-token>" \
  -v "$(pwd):/usr/src" \
  sonarsource/sonar-scanner-cli \
  -Dsonar.projectKey=juice-shop-devsecops \
  -Dsonar.sources=routes,lib,models,server.ts,app.ts \
  -Dsonar.exclusions="**/node_modules/**,frontend/**,build/**"
```

> En Linux reemplaza `host.docker.internal` por `172.17.0.1` o usa `--network host`.

**e) Vincular VS Code**

1. Activity Bar → **SONARQUBE SETUP** → **CONNECTED MODE** → **Add SonarQube Server Connection**
2. Server URL `http://localhost:9000`, pega el token, nombre `local` → **Save Connection**
3. **Add Project Binding** (icono +) → selecciona la carpeta y el proyecto `juice-shop-devsecops`

**f) Escanear toda la carpeta**

Command Palette (`Ctrl+Shift+P` / `Cmd+Shift+P`) → **SonarQube: Scan for Hotspots in Folder**. Los hotspots aparecen agrupados por archivo en la vista **SONARQUBE**; clic derecho → **Show Rule Description** o **Review on Server**.

> En SonarQube Community Build 26.x este escaneo puede devolver **0 hotspots**: las reglas MD5, `Math.random()`, HTTP en claro o clave privada ya se clasifican como vulnerabilidades. Para ver el análisis completo del proyecto usa la web: http://localhost:9000/project/issues?id=juice-shop-devsecops. Al abrir `lib/insecurity.ts` en VS Code, Connected Mode sincroniza esas mismas issues con el servidor.

**g) Limpiar**

```bash
docker rm -f sonarqube
```

> **Semgrep vs SonarQube for IDE:** Semgrep corre en CLI/CI con reglas de comunidad (`p/security-audit`) y es ideal para automatizar. SonarQube for IDE da feedback inmediato al desarrollador con explicación de cada regla. Ambos son SAST; se complementan, no se reemplazan.

---

## Parte 4: Automatización en GitLab CI/CD

Una vez validados los análisis manuales, automatiza todo el proceso creando el siguiente pipeline en la raíz del proyecto:

**Archivo:** `.gitlab-ci.yml`

```yaml
stages:
  - prepare
  - secret_scanning
  - sca
  - sast

# Preparación
prepare:
  image: node:20
  stage: prepare
  script:
    - node --version
    - npm ci --ignore-scripts || true
  artifacts:
    paths:
      - node_modules/
    expire_in: 1h
  rules:
    - when: always

# Secret Scanning con Gitleaks
gitleaks-scan:
  image: alpine:3.18
  stage: secret_scanning
  before_script:
    - apk add --no-cache git curl tar gzip
  script:
    - GITLEAKS_VERSION=8.27.2
    - curl -sSL "https://github.com/gitleaks/gitleaks/releases/download/v${GITLEAKS_VERSION}/gitleaks_${GITLEAKS_VERSION}_linux_x64.tar.gz" -o gitleaks.tar.gz
    - tar -xzf gitleaks.tar.gz
    - chmod +x gitleaks
    - ./gitleaks detect --source . --report-path gitleaks-report.json || true
  artifacts:
    paths:
      - gitleaks-report.json
    expire_in: 1 week
  allow_failure: true
  rules:
    - when: always

# Secret Scanning con TruffleHog
trufflehog-scan:
  image: ubuntu:22.04
  stage: secret_scanning
  before_script:
    - apt-get update -y && apt-get install -y git curl jq python3 python3-pip --no-install-recommends
  script:
    - curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh | sh -s -- -b .
    - ./trufflehog filesystem . --results=verified,unknown --json > trufflehog-report.json || true
  artifacts:
    paths:
      - trufflehog-report.json
    expire_in: 1 week
  allow_failure: true
  rules:
    - when: always

# SCA con Trivy
trivy-scan:
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  stage: sca
  variables:
    REPORTS_DIR: security-reports
  before_script:
    - mkdir -p $REPORTS_DIR
  script:
    - echo "🔍 Ejecutando Trivy (SCA)..."
    - trivy filesystem . --format json --severity HIGH,CRITICAL --output $REPORTS_DIR/trivy-report.json || true
    - trivy filesystem . --format template --template '@contrib/gitlab.tpl' --output $REPORTS_DIR/gl-dependency-scanning-report.json --severity HIGH,CRITICAL || true
    - trivy filesystem . --format table --output $REPORTS_DIR/trivy-table.txt --severity HIGH,CRITICAL || true
    - trivy filesystem . --format cyclonedx --output $REPORTS_DIR/sbom.json || true
  artifacts:
    paths:
      - $REPORTS_DIR/
    reports:
      dependency_scanning: $REPORTS_DIR/gl-dependency-scanning-report.json
    expire_in: 1 week
  allow_failure: true
  rules:
    - when: always

# SAST con Semgrep
semgrep-scan:
  image: semgrep/semgrep:latest
  stage: sast
  needs:
    - gitleaks-scan
    - trufflehog-scan
  variables:
    SEMGREP_SRC: "$CI_PROJECT_DIR"
  script:
    - echo "🔍 Ejecutando Semgrep (SAST)..."
    - semgrep --version
    - semgrep --config "p/ci" --config "p/security-audit" --config "p/javascript" --json --output semgrep-report.json "$SEMGREP_SRC" || true
    - semgrep --config "p/ci" --config "p/security-audit" --config "p/javascript" "$SEMGREP_SRC" > semgrep-report.txt || true
  artifacts:
    paths:
      - semgrep-report.json
      - semgrep-report.txt
    expire_in: 1 week
  allow_failure: true
  rules:
    - when: always
```

---

## Parte 5: Secret Scanning con Pre-commit Hooks

Bloquea secretos **antes** de que lleguen al repositorio usando `pre-commit` como capa de defensa local.

### 5.1 Instalar pre-commit

```bash
# Con pip
pip install pre-commit

# Verificar
pre-commit --version
```

### 5.2 Crear `.pre-commit-config.yaml`

En la raíz del proyecto `juice-shop-devsecops`:

```yaml
repos:
  # Gitleaks: detecta secretos y credenciales
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.27.2
    hooks:
      - id: gitleaks

  # TruffleHog: verifica secretos con entropía y patrones
  - repo: https://github.com/trufflesecurity/trufflehog
    rev: v3.88.28
    hooks:
      - id: trufflehog
        name: TruffleHog Secret Scanner
        entry: trufflehog filesystem --results=verified,unknown
        language: golang
        pass_filenames: false

  # Detecta archivos con posibles credenciales por nombre
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: detect-private-key
      - id: check-added-large-files
        args: ['--maxkb=500']
        exclude: ^package-lock\.json$   # el lockfile pesa ~1 MB
```

### 5.3 Instalar hooks en el repositorio

```bash
# Instala los hooks en .git/hooks/pre-commit
pre-commit install

# Verificar instalación
cat .git/hooks/pre-commit
```

### 5.4 Probar manualmente

```bash
# Ejecutar sobre todos los archivos
pre-commit run --all-files

# Ejecutar solo gitleaks
pre-commit run gitleaks --all-files
```

> **Resultado esperado:** `detect-private-key` **falla** en 4 archivos (`lib/insecurity.ts`, `build/lib/insecurity.js`, `iac/gcp/credentials.json`, `docs/guia-secrets.md`). Es correcto: el repo trae esas claves a propósito. El hook de TruffleHog compila Go la primera vez (~5 min); puedes saltarlo con `SKIP=trufflehog pre-commit run --all-files`.

### 5.5 Simular detección de secreto

Agrega un secreto falso para validar que el hook bloquea el commit:

```bash
echo 'AWS_ACCESS_KEY_ID=AKIAQ3ZX7YT2LMNOPQ4R' >> config/default.yml
git add config/default.yml
git commit -m "test: secret scan hook"
# ❌ El commit debe ser BLOQUEADO por gitleaks (regla aws-access-token)
```

> **Nota:** los valores de documentación de AWS (`AKIAIOSFODNN7EXAMPLE`, `wJalrXUtnFEMI/...EXAMPLEKEY`) **no** disparan gitleaks porque contienen la palabra `EXAMPLE`, incluida en su allowlist. Usa un valor con formato real pero inventado, como el de arriba, o un token `ghp_` de 36 caracteres.

Elimina el secreto y verifica que el commit pasa:

```bash
# Revertir el cambio (HEAD -- porque el archivo ya está en stage)
git checkout HEAD -- config/default.yml
git commit --allow-empty -m "chore: verify pre-commit hooks work"
# ✅ Commit debe pasar
```

### 5.6 Actualizar hooks

```bash
pre-commit autoupdate
```

### 5.7 Configuración para CI (opcional)

Para que GitLab CI también valide los hooks en pipeline:

```yaml
# Agregar al .gitlab-ci.yml existente
pre-commit-check:
  image: python:3.12-slim
  stage: secret_scanning
  before_script:
    - pip install pre-commit
    - apt-get update -y && apt-get install -y git --no-install-recommends
  script:
    - SKIP=trufflehog pre-commit run --all-files
  allow_failure: true   # el repo contiene claves privadas a propósito; detect-private-key siempre falla
  rules:
    - when: always
```

> **Flujo recomendado:** pre-commit bloquea en local → GitLab CI valida en pipeline → doble barrera contra secretos expuestos.

---

## ✅ Checklist de Éxito

- [ ] Trivy, Semgrep, Gitleaks y TruffleHog ejecutan correctamente desde Docker  
- [ ] Reportes generados en `/reports` y `/sbom`  
- [ ] SonarQube for IDE instalado y hallazgos visibles en `routes/login.ts` y `lib/insecurity.ts`  
- [ ] Al menos un hallazgo de SonarQube corregido y verificado en el panel Problems  
- [ ] Se documentó el plan de corrección de dependencias (paquete, versión instalada, versión con fix). Juice Shop es vulnerable a propósito: aplicar los `npm install` en una rama aparte, no en `master`  
- [ ] Pipeline de GitLab CI ejecuta los escaneos automáticamente  
- [ ] Artefactos JSON/TXT visibles en GitLab  
- [ ] `.pre-commit-config.yaml` creado con hooks de Gitleaks y TruffleHog  
- [ ] `pre-commit install` ejecutado y hooks activos en `.git/hooks/`  
- [ ] Commit con secreto falso bloqueado correctamente por pre-commit  
- [ ] Comprensión de diferencias entre análisis manual y automatizado  

---

## 📘 Entregables

1. Captura de la carpeta `reports/` local con los JSON generados (no se commitea: los hooks lo bloquean; en GitLab están como artefactos)  
2. Capturas de pantalla de:
   - Ejecuciones Docker (Trivy, Semgrep, Gitleaks, TruffleHog)
   - Panel Problems de VS Code con hallazgos de SonarQube for IDE (antes y después de corregir)
   - Commit con secreto de prueba bloqueado por pre-commit (Parte 5.5)
   - Reportes generados en GitLab CI  
3. Documento de reflexión:  
   - ¿Qué diferencias observas entre los análisis manuales y automatizados?  
   - ¿Qué ventajas aporta integrar SCA y Secret Scanning al pipeline DevSecOps?  
   - ¿Qué hallazgos coinciden entre Semgrep y SonarQube for IDE y cuáles no? ¿Por qué?  
   - ¿Qué otras herramientas agregarías al pipeline de este laboratorio?  

---

**Autor:** Wilson Julca Mejía  
Curso: *DevSecOps y Seguridad en CI/CD – UTEC*  
Universidad de Ingeniería y Tecnología (UTEC)  
