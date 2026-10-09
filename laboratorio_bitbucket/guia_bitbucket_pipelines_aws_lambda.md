# Guía: Bitbucket Pipelines con Python — despliegue de FastAPI a AWS Lambda con Serverless Framework v4

**Duración estimada:** 2.5–3 h  
**Nivel:** Intermedio  
**Contexto:** Esta guía continúa la de GitLab (`guia_gitlab_ci_runner.md`). Allí dejaste una API FastAPI con handler Lambda (Mangum) que corre **offline** con Serverless Framework v3, sin credenciales ni cuenta de AWS. Aquí tomas **el mismo repositorio**, creas una rama nueva, lo conectas a **Bitbucket** como segundo remoto y construyes un `bitbucket-pipelines.yml` que prueba la API, la levanta offline y, con aprobación manual, la **despliega de verdad a AWS Lambda + API Gateway** con Serverless Framework v4. Al final ves lo que se creó en AWS y lo eliminas desde el propio pipeline.

```
GitLab (guía anterior):   curl → serverless-offline (laptop / runner) → Mangum → FastAPI
Bitbucket (esta guía):    curl → API Gateway (HTTP API) → AWS Lambda → Mangum → FastAPI
```

---

## Objetivos de aprendizaje

- Entender la arquitectura de Bitbucket Pipelines (pipelines, steps, stages, parallel, artifacts, caches, deployments) y mapearla contra GitLab CI/CD
- Trabajar un mismo repositorio con dos remotos (`origin` → Bitbucket, `gitlab` → GitLab)
- Migrar un proyecto de Serverless Framework v3 a v4 y empaquetar `requirements.txt` con el soporte nativo de Python de v4
- Crear credenciales mínimas para CI/CD: Access Key de Serverless y usuario IAM dedicado en AWS
- Diferenciar *Repository variables* de *Deployment variables* y limitar las credenciales al step que despliega
- Reutilizar steps con anclas YAML (`&step`, `*step`, `<<:`)
- Controlar cuándo corre cada pipeline: `default`, `branches`, `pull-requests` y `custom`
- Desplegar a AWS con un `stage` manual por entorno (`test`, `production`) y verificar el endpoint real desde el pipeline
- Leer lo que Serverless Framework crea en AWS (CloudFormation, Lambda, API Gateway, IAM, CloudWatch) y eliminarlo con `serverless remove`

---

## Requisitos previos

✅ Haber completado la guía de GitLab hasta la **Parte 14** (repositorio con `hello.py`, `handler.py`, `serverless.yml`, `package.json`, `tests/`)  
✅ Cuenta en Bitbucket ([https://bitbucket.org/](https://bitbucket.org/)) con un *workspace* donde puedas crear repositorios  
✅ Cuenta en AWS con permisos para crear usuarios IAM (Lambda, API Gateway, CloudFormation, S3, CloudWatch Logs)  
✅ Cuenta en Serverless Framework ([https://app.serverless.com/](https://app.serverless.com/)) — obligatoria para v4, plan gratuito  
✅ Node.js 20+ y npm; Python 3.11 (conda o `venv`) con `requirements-dev.txt` instalado  
✅ Git y `curl` en la terminal

> **Minutos de build:** el plan Free de Bitbucket incluye **50 minutos/mes** de Pipelines. Un pipeline completo de esta guía (test + offline + deploy) consume ~5–7 min. Usa el cache y no relances pipelines sin motivo.

---

## Estructura del proyecto

```
gitlab_lambda_demo/                 # mismo repo de la guía de GitLab
├── .gitlab-ci.yml                  # Pipeline de GitLab (se mantiene; ver nota en 9.2)
├── bitbucket-pipelines.yml         # NUEVO: pipeline de Bitbucket
├── hello.py                        # API FastAPI (greet, add, /hello, /add)
├── handler.py                      # Handler Lambda con Mangum
├── serverless.yml                  # MODIFICADO: Serverless v4 + pythonRequirements + package.patterns
├── package.json                    # MODIFICADO: serverless@4, serverless-offline@14
├── package-lock.json               # regenerado con npm install (el pipeline usa npm ci)
├── requirements.txt                # fastapi, mangum → van dentro del ZIP de Lambda
├── requirements-dev.txt            # pytest, pytest-cov, httpx, uvicorn
├── tests/
│   ├── test_hello.py
│   └── test_handler.py
└── .gitignore                      # + test-reports/, .requirements/, requirements.zip
```

---

## Parte 1: Bitbucket Pipelines frente a GitLab CI/CD

### 1.1 Equivalencias

| Concepto | GitLab CI/CD (`.gitlab-ci.yml`) | Bitbucket Pipelines (`bitbucket-pipelines.yml`) |
|---|---|---|
| Unidad de ejecución | `job` | `step` (un contenedor Docker por step) |
| Agrupación | `stages:` + `stage:` en cada job | `stage:` (lista de steps secuenciales) |
| Orden | por `stages` o DAG con `needs:` | **el orden de la lista**: cada elemento espera al anterior |
| Paralelismo | jobs del mismo stage | bloque `parallel:` |
| Imagen Docker | `image:` global o por job | `image:` global o por step |
| Cuándo corre | `rules:` / `only` / `except` | secciones `default`, `branches`, `pull-requests`, `tags`, `custom` |
| Job manual | `when: manual` | `trigger: manual` (en un `stage` o en un step suelto) |
| Entornos | `environment:` | `deployment:` → entornos Test / Staging / Production |
| Variables | Settings → CI/CD → Variables (masked/protected) | Repository variables y Deployment variables (*Secured*) |
| Variables predefinidas | `CI_COMMIT_SHA`, `CI_PROJECT_DIR`, … | `BITBUCKET_COMMIT`, `BITBUCKET_CLONE_DIR`, … |
| Artefactos | `artifacts:` (expiran según `expire_in`) | `artifacts:` (14 días; pasan automáticamente a los steps siguientes) |
| Cache | `cache:` con `key` y `paths` | `caches:` (predefinidos como `pip`, `node`, o propios en `definitions.caches`) |
| Reutilizar YAML | `extends:`, `!reference`, `include:` | anclas YAML (`&`, `*`, `<<:`) en `definitions.steps` |
| Runner | shared runners o `gitlab-runner` con `tags:` | runners de Atlassian (Linux x86_64) o *self-hosted runners* con `runs-on:` |
| Reportes de tests | `artifacts.reports.junit` | automático: XML JUnit en `test-reports/`, `test-results/`, … |

### 1.2 Reglas que conviene saber antes de escribir el YAML

- Un `stage` agrupa steps **secuenciales**; no admite `parallel:` dentro, ni steps con `trigger: manual` propio. El `trigger` y el `deployment` van en el `stage`.
- Un step dentro de un `stage` **no puede** sobrescribir una propiedad del stage (p. ej. `deployment`).
- `default` corre en cualquier push a una rama **que no tenga** regla en `branches`. Los pipelines de `pull-requests` corren **además** del de la rama.
- `custom` solo corre cuando alguien lo lanza desde la UI (**Pipelines → Run pipeline**).
- Los artifacts se guardan 14 días y se descargan automáticamente en los steps posteriores del mismo pipeline. Los caches pesan máximo 1 GB y expiran a los 7 días sin uso.
- Las variables se resuelven en este orden de prioridad: **Pipeline > Deployment > Repository > Workspace**.

---

## Parte 2: Preparar el repositorio y el remoto de Bitbucket

### 2.1 Crear la rama de trabajo

Parte de la rama donde quedó la API offline en la guía de GitLab:

```bash
cd gitlab_lambda_demo
git checkout feature/utec-3-serverless-offline
git checkout -b feature/utec-4-bitbucket-pipelines
```

### 2.2 Crear el repositorio en Bitbucket

1. [bitbucket.org](https://bitbucket.org/) → **Create → Repository**
2. *Workspace*: el tuyo · *Project*: crea uno (p. ej. `UTEC`) · *Repository name*: `gitlab_lambda_demo` (mismo nombre que en GitLab)
3. *Include a README?* → **No** · *Default branch*: `main`
4. **Create repository**

Bitbucket te muestra la URL del remoto. Guárdala.

### 2.3 Agregar Bitbucket como remoto

El repo ya tiene `origin` apuntando a GitLab. Para que Bitbucket sea el remoto principal de esta guía, renombra el de GitLab a `gitlab` y agrega Bitbucket como `origin`:

```bash
git remote rename origin gitlab
git remote add origin git@bitbucket.org:<workspace>/gitlab_lambda_demo.git
# o por HTTPS:  https://<usuario>@bitbucket.org/<workspace>/gitlab_lambda_demo.git

git remote -v
# gitlab   git@gitlab.com:<usuario>/demo-gitlab.git (fetch)
# gitlab   git@gitlab.com:<usuario>/demo-gitlab.git (push)
# origin   git@bitbucket.org:<workspace>/gitlab_lambda_demo.git (fetch)
# origin   git@bitbucket.org:<workspace>/gitlab_lambda_demo.git (push)
```

Verifica el acceso SSH y sube primero `main` (así queda como rama por defecto) y luego la rama de trabajo:

```bash
ssh -T git@bitbucket.org
# authenticated via ssh key.

git push origin main
git push -u origin feature/utec-4-bitbucket-pipelines
```

> Si usas SSH y Bitbucket rechaza la conexión, agrega tu llave pública en **Personal settings → SSH keys**. Si tienes varias cuentas, fija la llave en `~/.ssh/config`:
>
> ```
> Host bitbucket.org
>   HostName bitbucket.org
>   User git
>   IdentityFile ~/.ssh/id_ed25519_bitbucket
>   IdentitiesOnly yes
> ```
>
> Con HTTPS necesitas un *App password* (**Personal settings → App passwords**) con permiso *Repositories: Write*; se usa como contraseña.

### 2.4 Habilitar Pipelines

**Repository settings → Pipelines → Settings → Enable Pipelines**. Hasta que no exista un `bitbucket-pipelines.yml` en la rama, no se ejecuta nada.

---

## Parte 3: Migrar Serverless Framework v3 → v4

### 3.1 Por qué v4

| | v3 (guía GitLab) | v4 (esta guía) |
|---|---|---|
| Estado | mantenimiento; sin nuevas funciones | versión activa |
| Cuenta Serverless | no requerida | **obligatoria** para cualquier comando (login o `SERVERLESS_ACCESS_KEY`) |
| Empaquetar `requirements.txt` | plugin `serverless-python-requirements` | **nativo** desde v4.22 (`custom.pythonRequirements`), el plugin quedó deprecado |
| `serverless-offline` | v13 | v14 |
| Bucket de despliegue | uno por stack | uno compartido por cuenta/región (`serverless-framework-deployments-<region>-<id>`) |
| Licencia | gratis | gratis para organizaciones con menos de USD 2M de ingresos anuales |

En GitLab evitamos v4 porque el runner no tenía credenciales. En Bitbucket sí las tendrá, así que es el momento de migrar.

### 3.2 `package.json`

```json
{
  "name": "gitlab-lambda-demo",
  "version": "1.0.0",
  "private": true,
  "scripts": { "offline": "serverless offline" },
  "devDependencies": {
    "serverless": "^4.43.0",
    "serverless-offline": "^14.8.3"
  }
}
```

```bash
npm install serverless@^4.43.0 serverless-offline@^14.8.3 --save-dev
# regenera package-lock.json → commitéalo: el pipeline usa `npm ci`
```

> No instales `serverless-python-requirements`. En v4 el bloque `custom.pythonRequirements` activa el empaquetado nativo.

### 3.3 `serverless.yml`

```yaml
service: utec-lambda-demo
frameworkVersion: '4'

provider:
  name: aws
  runtime: python3.11
  architecture: x86_64              # el pipeline de Bitbucket corre en x86_64: los wheels que descarga pip sirven en Lambda
  region: ${env:AWS_DEFAULT_REGION, "us-east-1"}
  stage: ${opt:stage, "dev"}
  memorySize: 256
  timeout: 10
  logRetentionInDays: 7             # CloudWatch Logs: no acumular logs del laboratorio
  environment:
    STAGE: ${sls:stage}

plugins:
  - serverless-offline              # simula API Gateway + Lambda en local; no afecta el deploy

package:
  patterns:                         # el ZIP solo lleva el código; las dependencias las inyecta pythonRequirements
    - '!**'
    - 'hello.py'
    - 'handler.py'

custom:
  # Serverless v4 (>= 4.22) empaqueta requirements.txt de forma nativa al ver este bloque.
  pythonRequirements:
    dockerizePip: false             # el runner de Bitbucket ya es Linux x86_64 con Python 3.11
    slim: false                     # slim borra *.dist-info y pydantic lee la versión de email-validator
                                    # con importlib.metadata al importar FastAPI → Runtime.ImportModuleError
  serverless-offline:
    httpPort: 3000
    lambdaPort: 3002

functions:
  api:
    handler: handler.handler        # módulo handler.py → objeto Mangum llamado handler
    description: "API FastAPI (UTEC - Bitbucket Pipelines)"
    events:
      - httpApi:                    # API Gateway v2 (HTTP API)
          path: /hello
          method: GET
      - httpApi:
          path: /add
          method: GET
```

Qué cambió respecto a la guía de GitLab:

| Campo | Por qué |
|---|---|
| `frameworkVersion: '4'` | Fija la versión mayor. Con v3 instalado, Serverless se niega a arrancar |
| `service: utec-lambda-demo` | Nombre del stack de CloudFormation: `<service>-<stage>`. Cambia el nombre si otra persona del curso despliega en la misma cuenta AWS |
| `architecture: x86_64` | Debe coincidir con la máquina que ejecuta `pip` al empaquetar. Los runners de Bitbucket son x86_64 |
| `memorySize`, `timeout` | Lambda: 256 MB bastan para FastAPI; 10 s de tope por invocación |
| `logRetentionInDays: 7` | Crea el *log group* `/aws/lambda/<función>` con retención, en vez de dejarlo infinito |
| `package.patterns` | `'!**'` excluye todo y luego incluye solo `hello.py` y `handler.py`. Sin esto el ZIP llevaría `node_modules/`, `tests/`, `htmlcov/`… |
| `custom.pythonRequirements` | Instala `requirements.txt` con el `python3` del `PATH` y lo inyecta en el ZIP. `slim: false` a propósito: `slim` borra los `*.dist-info` y FastAPI/pydantic los necesitan al importar (ver 9.2 Error 8) |

> **`'!**'` y las dependencias:** los patrones solo filtran *tu* código. `pythonRequirements` agrega las librerías al ZIP después, así que no hace falta incluirlas en `patterns`.

### 3.4 `.gitignore`

Agrega lo que generan Bitbucket y Serverless v4:

```
test-reports/        # JUnit XML del step test
.requirements/       # cache local de pythonRequirements
requirements.zip
```

### 3.5 Verificar en local

v4 exige sesión. En tu laptop basta con iniciarla una vez (queda en `~/.serverlessrc`):

```bash
npx serverless login          # abre el navegador → app.serverless.com
npx serverless print          # resuelve y muestra el serverless.yml; si hay un error de sintaxis, aquí sale
```

`offline` sigue funcionando igual que en la guía de GitLab (activa el entorno con Python 3.11 antes):

```bash
conda activate gitlab-lambda-demo   # o source .venv/bin/activate
npx serverless offline
```

```
Starting Offline at stage dev (us-east-1)

Offline [http for lambda] listening on http://localhost:3002
Function names exposed for local invocation by aws-sdk:
   * api: utec-lambda-demo-dev-api

   ┌───────────────────────────────────────────────────────────────────────┐
   │   GET  | http://localhost:3000/hello                                  │
   │   POST | http://localhost:3000/2015-03-31/functions/api/invocations   │
   │   GET  | http://localhost:3000/add                                    │
   │   POST | http://localhost:3000/2015-03-31/functions/api/invocations   │
   └───────────────────────────────────────────────────────────────────────┘

Server ready: http://localhost:3000 🚀
```

```bash
curl "http://localhost:3000/hello?name=UTEC"   # {"message":"Hello, UTEC!","source":"FastAPI"}
curl "http://localhost:3000/add?a=2&b=3"       # {"result":5}
```

> **`serverless package` / `serverless deploy` desde la laptop:** v4 necesita credenciales AWS incluso para `package` (resuelve el ID de cuenta). Además, en una Mac con Apple Silicon `pip` descarga wheels **arm64**, que no sirven en una Lambda x86_64 (`pydantic-core` es código compilado). Por eso en esta guía **el empaquetado y el deploy ocurren solo en el pipeline**. Si quieres desplegar desde tu máquina, usa `dockerizePip: non-linux` (requiere Docker) o `pipCmdExtraArgs: ["--platform", "manylinux2014_x86_64", "--only-binary=:all:"]`.

```bash
git add package.json package-lock.json serverless.yml .gitignore
git commit -m "build(serverless): migrate to Serverless Framework v4 with native python requirements"
```

---

## Parte 4: Credenciales

Necesitas tres secretos: uno para Serverless Framework y dos para AWS.

### 4.1 `SERVERLESS_ACCESS_KEY`

En CI no hay navegador para `serverless login`; se usa una Access Key.

1. [app.serverless.com](https://app.serverless.com/) → tu perfil (arriba a la derecha) → **Settings**
2. **Access Keys → Create** · nombre: `bitbucket-ci` → **Create**
3. Copia el token: **solo se muestra una vez**

Sin esta variable, **cualquier** comando de v4 (incluido `offline`) falla en el pipeline con:

```
✖ You must sign in or use a License Key with Serverless Framework V.4 and later versions ...
```

### 4.2 Usuario IAM `bitbucket-ci-deployer`

Nunca uses las credenciales root ni tus credenciales personales en CI. Crea un usuario dedicado:

1. AWS Console → **IAM → Users → Create user** · nombre: `bitbucket-ci-deployer` → **Next**
2. **Attach policies directly** y agrega:
   - `AWSCloudFormationFullAccess` — Serverless despliega como stack de CloudFormation
   - `AWSLambda_FullAccess` — crea/actualiza la función
   - `AmazonAPIGatewayAdministrator` — crea el HTTP API y sus rutas
   - `IAMFullAccess` — crea el *execution role* de la Lambda
   - `AmazonS3FullAccess` — bucket de despliegue donde sube el ZIP
   - `CloudWatchLogsFullAccess` — *log group* con retención
3. **Create user**
4. Abre el usuario → **Security credentials → Access keys → Create access key** → caso de uso **Command Line Interface (CLI)** → copia `Access key ID` y `Secret access key` (solo se muestran una vez)

> Estas políticas son amplias para que el laboratorio no se trabe en permisos. En un proyecto real se acotan a los recursos del stack (`utec-lambda-demo-*`) o se reemplaza el usuario por **OIDC** (Bitbucket → rol de AWS sin llaves de larga duración).

### 4.3 Variables en Bitbucket

Dos niveles:

- **Repository variables**: llegan a **todos** los steps.
- **Deployment variables**: llegan **solo** a los steps/stages con `deployment: <entorno>`.

**Repository settings → Pipelines → Repository variables**

| Variable | Valor | Secured |
|---|---|---|
| `SERVERLESS_ACCESS_KEY` | token de app.serverless.com | ✅ |
| `AWS_DEFAULT_REGION` | `us-east-1` | — |

**Repository settings → Pipelines → Deployments → Test → Add variable** (y repite en **Production**)

| Variable | Valor | Secured |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | Access key del usuario IAM | ✅ |
| `AWS_SECRET_ACCESS_KEY` | Secret access key del usuario IAM | ✅ |

> Para el laboratorio puedes usar el mismo usuario IAM en `test` y `production`. En un proyecto real cada entorno tendría credenciales (o cuentas AWS) distintas: eso es exactamente lo que las Deployment variables permiten.

¿Por qué `SERVERLESS_ACCESS_KEY` es de repositorio y no de deployment? Porque el step `offline` también ejecuta Serverless v4 y no despliega a ningún entorno. Las llaves de AWS, en cambio, solo las necesita el step que despliega.

```
Repository variables ───────────────► todos los steps (test, offline, deploy, remove)
  SERVERLESS_ACCESS_KEY → Serverless Framework
  AWS_DEFAULT_REGION    → serverless.yml (${env:AWS_DEFAULT_REGION})

Deployment variables (test / production) ──► solo steps con deployment: <entorno>
  AWS_ACCESS_KEY_ID      → AWS SDK credential chain
  AWS_SECRET_ACCESS_KEY  → AWS SDK credential chain
  BITBUCKET_DEPLOYMENT_ENVIRONMENT (automática) → nombre del entorno = stage de Serverless
```

*Secured* oculta el valor en la UI y lo enmascara en los logs (`$AWS_SECRET_ACCESS_KEY` aparece como `****`).

---

## Parte 5: `bitbucket-pipelines.yml`

### 5.1 Diseño

```
push a feature/*  ──► [test ‖ offline] ──► stage deploy-test (manual, deployment: test)
push a main       ──► [test ‖ offline] ──► stage deploy-production (manual, deployment: production)
pull request      ──► [test ‖ offline]
Run pipeline      ──► custom: remove-test | remove-production   (serverless remove)
```

### 5.2 Archivo completo

```yaml
# Bitbucket Pipelines: API FastAPI + Mangum → AWS Lambda con Serverless Framework v4
#
# Variables necesarias (Repository settings → Pipelines):
#   Repository variables : SERVERLESS_ACCESS_KEY (secured), AWS_DEFAULT_REGION
#   Deployment variables : AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY (secured) en los entornos test y production

image: nikolaik/python-nodejs:python3.11-nodejs20   # Python 3.11 (runtime de Lambda) + Node 20 (Serverless Framework)

definitions:
  caches:
    npm: ~/.npm                      # tarballs de npm; "pip" es un cache predefinido (~/.cache/pip)

  steps:
    # ── Tests unitarios (FastAPI + handler Lambda) ──────────────────────────
    - step: &test
        name: test
        caches:
          - pip
        script:
          - python --version
          - pip install -r requirements-dev.txt
          - export PYTHONPATH="$BITBUCKET_CLONE_DIR"
          # Bitbucket lee automáticamente los JUnit XML que estén en test-reports/ (pestaña Tests)
          - pytest tests/ -v --junitxml=test-reports/junit.xml --cov=. --cov-report=xml --cov-report=html
        artifacts:
          - htmlcov/**
          - coverage.xml

    # ── Smoke test con serverless-offline (API Gateway + Lambda simulados) ───
    - step: &offline
        name: offline
        caches:
          - pip
          - npm
        script:
          - python --version && node --version
          - pip install -r requirements.txt
          - npm ci --cache ~/.npm --prefer-offline
          - npx serverless offline > offline.log 2>&1 &
          - for i in $(seq 1 60); do curl -sf "http://localhost:3000/hello" > /dev/null && break; sleep 1; done
          - curl -sf "http://localhost:3000/hello?name=UTEC" -o hello.json && cat hello.json
          - grep -q "Hello, UTEC!" hello.json
          - curl -sf "http://localhost:3000/add?a=2&b=3" -o add.json && cat add.json
          - grep -q '"result":5' add.json
        after-script:
          - echo "===== offline.log (servidor serverless-offline) ====="
          - cat offline.log
        artifacts:
          - offline.log
          - hello.json
          - add.json

    # ── Deploy a AWS + smoke test contra el endpoint real ───────────────────
    # El stage de Serverless = nombre del entorno de Bitbucket (test | production).
    # Las credenciales AWS llegan como Deployment variables del entorno declarado en el stage.
    - step: &deploy
        name: deploy
        caches:
          - pip
          - npm
        script:
          - export STAGE="$BITBUCKET_DEPLOYMENT_ENVIRONMENT"
          # Entre comillas simples: " #" (espacio + #) iniciaría un comentario YAML y truncaría la línea
          - 'echo "Deploy de $BITBUCKET_REPO_FULL_NAME ($BITBUCKET_BRANCH @ $BITBUCKET_COMMIT) -> stage $STAGE, build #$BITBUCKET_BUILD_NUMBER"'
          - npm ci --cache ~/.npm --prefer-offline
          - npx serverless deploy --stage "$STAGE" --verbose
          # v4 escribe la salida de "info" por stderr: 2>&1 para capturarla en el artifact
          - npx serverless info --stage "$STAGE" 2>&1 | tee deploy-info.txt
          - export API_URL=$(grep -oE 'https://[a-z0-9]+\.execute-api\.[a-z0-9-]+\.amazonaws\.com' deploy-info.txt | head -1)
          - test -n "$API_URL" || { echo "ERROR - no se encontró la URL del HTTP API en deploy-info.txt"; exit 1; }
          - echo "API_URL=$API_URL"
          - curl -sf --retry 5 --retry-delay 3 --retry-all-errors "$API_URL/hello?name=UTEC" -o hello-aws.json && cat hello-aws.json
          - grep -q "Hello, UTEC!" hello-aws.json
          - curl -sf "$API_URL/add?a=2&b=3" -o add-aws.json && cat add-aws.json
          - grep -q '"result":5' add-aws.json
        # Diagnóstico: corre siempre (también si falla). Invoca la Lambda directamente y trae los logs de CloudWatch
        after-script:
          - echo "===== serverless invoke (respuesta cruda de la Lambda) ====="
          - npx serverless invoke -f api --stage "$BITBUCKET_DEPLOYMENT_ENVIRONMENT" -p events/hello.json 2>&1 || true
          - echo "===== serverless logs (CloudWatch, últimos 5 min) ====="
          - npx serverless logs -f api --stage "$BITBUCKET_DEPLOYMENT_ENVIRONMENT" --startTime 5m 2>&1 || true
        artifacts:
          - deploy-info.txt
          - hello-aws.json
          - add-aws.json

    # ── Eliminar el stack (Lambda, API Gateway, IAM role, logs) ─────────────
    - step: &remove
        name: remove
        caches:
          - npm
        script:
          - npm ci --cache ~/.npm --prefer-offline
          - npx serverless remove --stage "$BITBUCKET_DEPLOYMENT_ENVIRONMENT" --verbose

pipelines:
  # Cualquier rama sin regla propia (p. ej. feature/*): valida y permite desplegar al entorno "test"
  default:
    - parallel:
        - step: *test
        - step: *offline
    - stage:
        name: deploy-test
        deployment: test
        trigger: manual
        steps:
          - step: *deploy

  branches:
    main:
      - parallel:
          - step: *test
          - step: *offline
      - stage:
          name: deploy-production
          deployment: production
          trigger: manual
          steps:
            - step: *deploy

  # Los PR corren además del pipeline de la rama; aquí solo validan, nunca despliegan
  pull-requests:
    '**':
      - parallel:
          - step: *test
          - step: *offline

  # Pipelines manuales (Pipelines → Run pipeline → Custom) para limpiar AWS al terminar el laboratorio
  custom:
    remove-test:
      - step:
          <<: *remove
          name: remove-test
          deployment: test
    remove-production:
      - step:
          <<: *remove
          name: remove-production
          deployment: production
```

### 5.3 Explicación bloque por bloque

#### `image`

Una sola imagen para todos los steps: `nikolaik/python-nodejs:python3.11-nodejs20`, la misma del job `offline` de GitLab. Trae **Python 3.11** (el runtime de la Lambda, así `pip` descarga wheels compatibles) y **Node 20** (Serverless Framework es una herramienta Node).

#### `definitions.caches` y `caches:`

| Cache | Ruta | Qué ahorra |
|---|---|---|
| `pip` (predefinido) | `~/.cache/pip` | descargas de `fastapi`, `pytest`, … |
| `npm` (propio) | `~/.npm` | tarballs de `serverless` y `serverless-offline`; `npm ci --cache ~/.npm --prefer-offline` los reutiliza |

Un cache se guarda al terminar el primer step exitoso que lo declara y se reutiliza hasta 7 días. Los steps en `parallel` pueden **leer** caches pero no escribirlos a la vez.

#### `definitions.steps` y las anclas YAML

`&test`, `&offline`, `&deploy`, `&remove` definen cada step **una sola vez**. Luego `- step: *test` lo reutiliza tal cual en `default`, `branches` y `pull-requests`. Con `<<: *remove` se copia el step y se **sobrescriben** campos (`name`, `deployment`). Es el equivalente de `extends:` en GitLab, pero es YAML puro.

#### Step `test`

Igual que el job `test` de GitLab, con dos diferencias:

- `PYTHONPATH="$BITBUCKET_CLONE_DIR"` (en GitLab era `$CI_PROJECT_DIR`).
- El JUnit XML se escribe en `test-reports/junit.xml`. Bitbucket **escanea automáticamente** `test-reports/`, `test-results/`, `surefire-reports/`, … y muestra la pestaña **Tests** del step. No hace falta declararlo como `artifacts.reports`.

#### Step `offline`

Mismo smoke test que en GitLab: arranca `serverless offline` en segundo plano, espera hasta 60 s a que `/hello` responda y valida las dos rutas con `curl -sf` + `grep -q`. Diferencias:

- `after-script` corre **siempre**, pase o falle el `script`, y muestra `offline.log` (el log del servidor) en el propio log del step.
- `artifacts` sin `when: always`: en Bitbucket los artifacts se recogen aunque el step falle.
- Como ahora es Serverless **v4**, este step necesita `SERVERLESS_ACCESS_KEY` (Repository variable). Sin ella falla con el mensaje de la sección 4.1.

#### Step `deploy`

| Línea | Qué hace |
|---|---|
| `STAGE="$BITBUCKET_DEPLOYMENT_ENVIRONMENT"` | Bitbucket inyecta el nombre del entorno del `deployment:` (`test` o `production`). Se usa como `--stage`, así el stack se llama `utec-lambda-demo-test` o `utec-lambda-demo-production` y el mismo step sirve para ambos entornos |
| `echo "Deploy de $BITBUCKET_REPO_FULL_NAME …"` | Trazabilidad: qué commit, rama y build se desplegó (`BITBUCKET_COMMIT`, `BITBUCKET_BRANCH`, `BITBUCKET_BUILD_NUMBER`) |
| `serverless deploy --stage "$STAGE" --verbose` | Instala `requirements.txt` con pip, arma el ZIP, lo sube a S3 y crea/actualiza el stack de CloudFormation. `--verbose` imprime los eventos de CloudFormation |
| `serverless info … \| tee deploy-info.txt` | Muestra los endpoints y los guarda como artifact |
| `API_URL=$(grep -oE 'https://…execute-api…amazonaws\.com' …)` | Extrae la URL base del HTTP API de la salida de `info` |
| `curl -sf --retry 5 --retry-all-errors …` | Primera invocación = *cold start* + propagación de API Gateway; reintenta hasta 5 veces |
| `grep -q "Hello, UTEC!"` | El deploy solo se marca verde si la API **real** responde lo esperado |

Las credenciales de AWS no se mencionan en ningún lado: el AWS SDK las toma de `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`, que existen en este step únicamente porque el `stage` declara `deployment:`.

#### Step `remove`

`serverless remove` borra el stack completo (Lambda, API Gateway, rol IAM, log group). Necesita `npm ci` porque `serverless.yml` declara el plugin `serverless-offline` y v4 lo carga aunque no se use. Se expone solo como pipeline `custom`.

#### `pipelines`

| Sección | Cuándo corre | Qué hace |
|---|---|---|
| `default` | push a cualquier rama **sin** regla propia (tu `feature/utec-4-…`) | valida y deja un stage manual `deploy-test` |
| `branches.main` | push a `main` | valida y deja un stage manual `deploy-production` |
| `pull-requests.'**'` | al crear/actualizar un PR hacia cualquier rama | solo valida; corre **además** del pipeline de la rama |
| `custom.remove-*` | solo desde **Pipelines → Run pipeline** | `serverless remove` del entorno indicado |

El `stage` de deploy cumple las tres reglas de Bitbucket: `trigger: manual` y `deployment:` están en el **stage** (no en el step), y dentro del stage no hay `parallel:`.

### 5.4 Validar el YAML antes del push

```bash
python3 -c "import yaml; yaml.safe_load(open('bitbucket-pipelines.yml')); print('OK')"
```

Esto detecta errores de sintaxis y de anclas. Las reglas propias de Bitbucket (steps dentro de stages, triggers, etc.) las valida el **validador oficial**: [https://bitbucket-pipelines.atlassian.io/validator](https://bitbucket-pipelines.atlassian.io/validator). Pega el archivo y corrige antes de gastar minutos.

```bash
git add bitbucket-pipelines.yml
git commit -m "ci(pipeline): add Bitbucket Pipelines with manual deploy to AWS"
git push origin feature/utec-4-bitbucket-pipelines
```

---

## Parte 6: Ejecutar el pipeline

### 6.1 Observar la validación

**Repositorio → Pipelines**. Verás un pipeline para la rama con:

```
┌─────────┐
│  test   │ ─┐
└─────────┘  ├──► ⏸ deploy-test (manual)
┌─────────┐  │
│ offline │ ─┘
└─────────┘
```

- `test` y `offline` corren **a la vez** (dos contenedores). Abre `test` → pestaña **Tests**: Bitbucket muestra los 11 tests leídos de `test-reports/junit.xml`. Pestaña **Artifacts**: `htmlcov/` y `coverage.xml` descargables.
- Abre `offline` → en el log verás el JSON de `hello.json` y `add.json`, y al final el bloque de `after-script` con `offline.log` (`GET /hello (λ: api)`, `Duration: …`).
- La primera ejecución descarga la imagen y las dependencias; en la segunda, con cache, `pip install` y `npm ci` tardan segundos. Compara los tiempos.

### 6.2 Aprobar el deploy

El stage `deploy-test` aparece en **pausa** con un botón **Run** (o **Deploy**). Solo quien tenga permiso de escritura puede lanzarlo. Púlsalo.

### 6.3 Leer el log del deploy

Salida (resumida) del step `deploy`:

```
Deploy de <workspace>/utec-lambda-demo (feature/utec-4-bitbucket-pipelines @ 3f2a9c1…) → stage test, build #4

Deploying "utec-lambda-demo" to stage "test" (us-east-1)

Packaging Python dependencies for "api" …          ← pip install -r requirements.txt (x86_64, 3.11)
Uploading (19.02 MB)                               ← ZIP con fastapi, pydantic-core, starlette, mangum, uvicorn…
Creating CloudFormation stack …                    ← --verbose: CREATE_IN_PROGRESS / CREATE_COMPLETE por recurso

✔ Service deployed to stack utec-lambda-demo-test (57s)

endpoints:
  GET - https://ab12cd34ef.execute-api.us-east-1.amazonaws.com/hello
  GET - https://ab12cd34ef.execute-api.us-east-1.amazonaws.com/add
functions:
  api: utec-lambda-demo-test-api (19 MB)

API_URL=https://ab12cd34ef.execute-api.us-east-1.amazonaws.com
{"message":"Hello, UTEC!","source":"FastAPI"}
{"result":5}
```

El primer deploy tarda 1.5–3 min (crea todos los recursos). Los siguientes, menos de un minuto: CloudFormation solo actualiza lo que cambió.

### 6.4 Probar desde tu laptop

```bash
API_URL=https://ab12cd34ef.execute-api.us-east-1.amazonaws.com   # cópiala del log o del artifact deploy-info.txt
curl "$API_URL/hello?name=$USER"
curl "$API_URL/add?a=40&b=2"
curl -i "$API_URL/add?a=x&b=2"      # 422: FastAPI valida tipos, igual que en local
```

Y en el navegador: `https://ab12cd34ef.execute-api.us-east-1.amazonaws.com/hello?name=UTEC`.

> ¿Y `/docs` (Swagger)? Responde 404 porque el HTTP API solo tiene rutas para `/hello` y `/add`. **Reto:** reemplaza los dos `events` por `- httpApi: '*'` (ruta *catch-all*), vuelve a desplegar y abre `/docs`. FastAPI enruta por dentro; API Gateway solo necesita dejar pasar todo.

### 6.5 Panel de Deployments

**Repositorio → Deployments** muestra los entornos Test / Staging / Production con el último deploy de cada uno: commit, rama, quién lo aprobó y el enlace al pipeline. Es el historial de despliegues sin tener que buscar en los pipelines.

### 6.6 Pipeline de Pull Request

Crea un PR en Bitbucket de `feature/utec-4-bitbucket-pipelines` hacia `main`. Verás **dos** pipelines para el mismo commit: el de la rama (con su stage manual) y el de `pull-requests` (solo `test` + `offline`). Los PR nunca despliegan.

### 6.7 Producción

Al hacer *merge* a `main`, corre `branches.main` y deja el stage `deploy-production` en pausa. Al aprobarlo, se crea **otro** stack: `utec-lambda-demo-production`, con su propia URL. Es el patrón habitual: una rama por entorno, un stack por entorno, aprobación manual en producción.

---

## Parte 7: Qué creó Serverless Framework en AWS

Abre **AWS Console → CloudFormation → Stacks → `utec-lambda-demo-test` → Resources**:

| Recurso (CloudFormation) | Servicio | Para qué |
|---|---|---|
| `ApiLambdaFunction` | Lambda `utec-lambda-demo-test-api` | tu código + dependencias (runtime python3.11, 256 MB, 10 s) |
| `ApiLambdaVersion…` | Lambda | versión inmutable del código de este deploy |
| `IamRoleLambdaExecution` | IAM | *execution role*: permite a la Lambda escribir en CloudWatch Logs |
| `ApiLogGroup` | CloudWatch Logs `/aws/lambda/utec-lambda-demo-test-api` | logs de cada invocación, retención 7 días |
| `HttpApi` | API Gateway (HTTP API) | la URL pública `https://….execute-api….amazonaws.com` |
| `HttpApiIntegrationApi` | API Gateway | conecta las rutas con la Lambda |
| `HttpApiRouteGetHello`, `HttpApiRouteGetAdd` | API Gateway | rutas `GET /hello`, `GET /add` |
| `HttpApiStage` | API Gateway | stage `$default` (por eso la URL no lleva `/dev` ni `/test`) |
| `ApiLambdaPermissionHttpApi` | Lambda | autoriza a API Gateway a invocar la función |

Además, fuera del stack, v4 usa un **bucket S3 compartido** por cuenta y región (`serverless-framework-deployments-us-east-1-<id>`) donde sube los ZIP de todos tus servicios. `serverless remove` **no** borra ese bucket.

**Ver los logs de una invocación:** CloudWatch → Log groups → `/aws/lambda/utec-lambda-demo-test-api`. Cada request deja `START`, `END` y `REPORT Duration: … Billed Duration: … Memory Size: 256 MB Max Memory Used: …`: las mismas líneas que `serverless-offline` imitaba en local. Si tienes credenciales en tu laptop también puedes usar `npx serverless logs -f api --stage test --tail`.

**Costo:** la capa gratuita de AWS cubre 1 M de invocaciones Lambda al mes de forma permanente y 1 M de requests de HTTP API durante 12 meses. Este laboratorio cuesta USD 0, pero **borra los recursos al terminar** (Parte 8) por higiene y porque las llaves IAM siguen activas.

---

## Parte 8: Limpieza

### 8.1 Eliminar el stack desde el pipeline

**Pipelines → Run pipeline** → *Branch*: `feature/utec-4-bitbucket-pipelines` → *Pipeline*: `custom: remove-test` → **Run**.

El step `remove-test` declara `deployment: test`, así recibe las mismas credenciales que usó el deploy, y `serverless remove --stage test` elimina el stack completo. Verifica en CloudFormation que `utec-lambda-demo-test` desapareció (estado `DELETE_COMPLETE` y luego ya no se lista). Repite con `remove-production` si desplegaste a producción.

### 8.2 Lo que queda y conviene revisar

- Bucket `serverless-framework-deployments-…`: vacíalo o déjalo; cuesta centavos solo si acumula ZIPs.
- Usuario IAM `bitbucket-ci-deployer`: **desactiva o elimina la access key** (IAM → Users → Security credentials) cuando termines el curso.
- Variables en Bitbucket: puedes dejarlas; son *Secured* y solo se usan si corre un pipeline.

---

## Parte 9: Troubleshooting

### 9.1 Tabla rápida

| Problema | Causa probable | Solución |
|---|---|---|
| El push no dispara nada | Pipelines no habilitado o el YAML no está en esa rama | Repository settings → Pipelines → Settings → Enable; `git ls-files bitbucket-pipelines.yml` |
| `offline` o `deploy` fallan con `You must sign in or use a License Key…` | falta `SERVERLESS_ACCESS_KEY` | agrégala como Repository variable **Secured** (todos los steps la necesitan con v4) |
| `deploy` falla con `The security token included in the request is invalid` / `AWS credentials missing` | llaves AWS no están en el entorno del `deployment:` | Deployments → **test** (o production) → variables `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` |
| `AccessDenied` / `is not authorized to perform: iam:CreateRole` | políticas IAM insuficientes | revisa las 6 políticas de 4.2 |
| `A step within stage can't contain a manual trigger` | `trigger: manual` en el step | súbelo al `stage` |
| `The step section is empty or null` | `parallel:` dentro de `stage > steps` | saca `parallel:` al nivel de la lista del pipeline |
| Step dentro de stage con `deployment:` rechazado | un step no puede sobrescribir propiedades del stage | declara `deployment:` solo en el stage |
| Un `echo` falla en `<1s` y en el log la línea aparece cortada | ` #` (espacio + `#`) dentro de la línea: YAML lo toma como comentario | envuelve la línea en comillas simples (ver 9.2 Error 6) |
| `grep -q "Hello, UTEC!" hello-aws.json` falla y `API_URL=` sale vacío | `serverless info` no listó endpoints | abre el artifact `deploy-info.txt`; revisa que el stack esté `CREATE_COMPLETE` |
| `deploy-info.txt` pesa 1 B y `API_URL=` vacío | v4 imprime `serverless info` por stderr | `serverless info … 2>&1 \| tee deploy-info.txt` (ver 9.2 Error 7) |
| Lambda responde `{"message":"Internal Server Error"}` | excepción al importar o ejecutar el handler | bloque `serverless invoke` / `serverless logs` al final del log de `deploy`; ver 9.2 Errores 2 y 8 |
| Stack en `ROLLBACK_COMPLETE` | el primer `deploy` falló a medias | ejecuta `custom: remove-test` (o borra el stack en la consola) y vuelve a desplegar |
| La pestaña **Tests** no aparece | el XML no está en una carpeta reconocida | `--junitxml=test-reports/junit.xml` (o `test-results/`) |
| `minutes quota exceeded` | se agotaron los 50 min del plan Free | revisa **Workspace settings → Plan details**; aprovecha caches, no relances pipelines |
| Cache no se aplica | primer run o cache expirado (7 días) | normal; verifica en el log `Cache "pip": Downloading` |

### 9.2 Errores concretos y su causa raíz

#### Error 1: `You must sign in or use a License Key with Serverless Framework V.4`

Aparece en **offline** y **deploy**. Serverless v4 valida la licencia antes de ejecutar cualquier comando. El pipeline no tiene `~/.serverlessrc`; necesita la variable de entorno `SERVERLESS_ACCESS_KEY`. Como `offline` no tiene `deployment:`, la variable debe ser de **repositorio**, no de deployment.

**En GitLab, esta rama rompe el job `offline`** por la misma razón: si también haces push a `gitlab`, agrega `SERVERLESS_ACCESS_KEY` en **Settings → CI/CD → Variables** (*masked*). Está anotado como comentario en `.gitlab-ci.yml`.

#### Error 2: `Unable to import module 'handler': No module named 'pydantic_core._pydantic_core'` (CloudWatch)

**Causa:** el ZIP lleva una versión de `pydantic-core` compilada para otra plataforma o versión de Python. Ocurre al desplegar desde una Mac con Apple Silicon (wheels `arm64`) o con un `python3` que no es 3.11.

**Solución:** despliega desde el pipeline (Linux x86_64 + Python 3.11, por eso la imagen `python3.11-nodejs20` y `architecture: x86_64`). Para desplegar desde la laptop: `dockerizePip: non-linux` o `pipCmdExtraArgs: ["--platform", "manylinux2014_x86_64", "--only-binary=:all:"]`.

Variante: `ELF load command address/offset not properly aligned` → `slim: true` con `strip: true` dañó un `.so`. Por eso `serverless.yml` fija `strip: false`.

#### Error 3: `Unzipped size must be smaller than 262144000 bytes`

**Causa:** falta `package.patterns`: el ZIP incluyó `node_modules/`, `htmlcov/`, `.venv/`… Lambda limita a **250 MB** descomprimidos.

**Solución:** `patterns: ['!**', 'hello.py', 'handler.py']`. Antes de desplegar en un proyecto real, revisa el tamaño en el log (`api: … (19 MB)`).

#### Error 4: el stage `deploy-test` no aparece en el pipeline del PR

No es un error: `pull-requests` tiene su propia lista sin stage de deploy. Es deliberado: un PR valida, no despliega. Busca el stage en el pipeline de la **rama** (mismo commit, pipeline distinto).

#### Error 5: `deploy` funciona en `test` pero falla en `production` con credenciales inválidas

Las Deployment variables son **por entorno**. Haber configurado `AWS_ACCESS_KEY_ID` en *Test* no las crea en *Production*. Agrégalas también allí.

#### Error 6: el step `deploy` falla en el primer `echo` y la línea sale truncada

**Log del step:**
```
+ echo "Deploy de $BITBUCKET_REPO_FULL_NAME ($BITBUCKET_BRANCH @ $BITBUCKET_COMMIT) → stage $STAGE, build
```

**Causa:** la línea original terminaba en `build #$BITBUCKET_BUILD_NUMBER"`. En YAML, un `#` precedido de espacio **inicia un comentario**, así que el parser cortó la línea justo ahí: quedó una comilla doble sin cerrar y bash falló. `yaml.safe_load` no lo detecta porque el YAML es válido; solo está recortado. Es el mismo problema que el `echo` del job `whoami` en la guía de GitLab.

**Solución:** envolver la línea completa en comillas simples (YAML) y dejar las dobles para bash:

```yaml
- 'echo "Deploy de $BITBUCKET_REPO_FULL_NAME (...) -> stage $STAGE, build #$BITBUCKET_BUILD_NUMBER"'
```

> **Regla:** cualquier línea de `script` con `#`, `:` seguido de espacio, `{`, `[` o `*` al inicio conviene escribirla entre comillas simples.

#### Error 7: `grep: hello-aws.json: No such file or directory` y `deploy-info.txt` pesa 1 B

**Log del step:**
```
+ grep -q "Hello, UTEC!" hello-aws.json
grep: hello-aws.json: No such file or directory
…
Artifact patterns matched 1 files with a total size of 1 B     ← deploy-info.txt vacío
```

**Causa:** el deploy **sí** terminó bien (el stack existe en CloudFormation). Pero Serverless v4 escribe la salida de `serverless info` por **stderr**, y `| tee deploy-info.txt` solo captura stdout. El archivo quedó vacío, `API_URL` también, y `curl` llamó a `/hello?name=UTEC` sin host.

**Solución:** redirigir stderr a stdout antes del `tee`, y cortar con un mensaje claro si no se encontró la URL:

```yaml
- npx serverless info --stage "$STAGE" 2>&1 | tee deploy-info.txt
- export API_URL=$(grep -oE 'https://[a-z0-9]+\.execute-api\.[a-z0-9-]+\.amazonaws\.com' deploy-info.txt | head -1)
- test -n "$API_URL" || { echo "ERROR - no se encontró la URL del HTTP API en deploy-info.txt"; exit 1; }
```

> Lección: un step rojo no siempre significa que el deploy falló. Lee **qué línea** falló: aquí fue el smoke test posterior, no `serverless deploy`.

#### Error 8: `Runtime.ImportModuleError: No package metadata was found for email-validator` (Lambda 500)

**Log (CloudWatch o `serverless invoke` en el `after-script`):**
```
[ERROR] Runtime.ImportModuleError: Unable to import module 'handler': No package metadata was found for email-validator
INIT_REPORT Init Duration: 724.82 ms  Phase: init  Status: error  Error Type: Runtime.ImportModuleError
```

**Causa:** `slim: true` en `pythonRequirements` borra los directorios `*.dist-info` para achicar el ZIP. Pero pydantic, al importar FastAPI, comprueba la versión de `email-validator` con `importlib.metadata`, que lee precisamente esos `dist-info`. Sin ellos la importación falla **antes** de ejecutar tu handler: todas las rutas devuelven `{"message":"Internal Server Error"}` y API Gateway responde 500.

El deploy en sí fue exitoso (`Service deployed to stack …`, ZIP de 11.75 MB con las dependencias inyectadas). Por eso el error solo aparece en el smoke test, no en `serverless deploy`.

**Solución:** `slim: false`. El ZIP pasa de 11.75 MB a 19 MB; irrelevante frente al límite de 250 MB.

```yaml
custom:
  pythonRequirements:
    dockerizePip: false
    slim: false
```

> Si quisieras mantener `slim`, tendrías que excluir los `dist-info` del borrado con `slimPatternsAppendDefaults: false` y tus propios `slimPatterns` (solo `__pycache__`, `*.pyc`). Para el laboratorio no vale la pena.

> `serverless logs --startTime 5m` muestra todo lo ocurrido en esa ventana, incluidos errores de un deploy anterior. Mira la hora de cada `REPORT` y compárala con la del step.

**Cómo lo encontramos:** el `after-script` del step `deploy` ejecuta `serverless invoke` y `serverless logs`, así el traceback de la Lambda aparece en el log de Bitbucket sin abrir la consola de AWS. Deja ese bloque: es tu primera herramienta de diagnóstico.

### 9.3 Reglas de seguridad

- `SERVERLESS_ACCESS_KEY`, `AWS_ACCESS_KEY_ID` y `AWS_SECRET_ACCESS_KEY` siempre **Secured**; nunca en el YAML ni en el código.
- Usuario IAM dedicado para CI; nunca root ni credenciales personales. Rota o elimina la access key al terminar.
- Credenciales de AWS como **Deployment variables**, no de repositorio: solo las ve el step que despliega.
- `trigger: manual` para cualquier deploy; los PR no despliegan.
- Mira el log del deploy una vez: comprueba que los secretos aparecen como `****`.

---

## Parte 10: GitLab vs Bitbucket, lo que viviste

| Tema | GitLab CI/CD | Bitbucket Pipelines |
|---|---|---|
| Orden de ejecución | `stages` + `needs:` (DAG explícito) | orden de la lista; `parallel:` para concurrencia |
| Reutilizar YAML | `extends:` / `!reference` | anclas YAML en `definitions.steps` |
| Job manual con entorno | `when: manual` + `environment:` | `stage` con `trigger: manual` + `deployment:` |
| Secretos por entorno | variables con *environment scope* | Deployment variables |
| Reporte de tests | `artifacts.reports.junit` | detección automática por carpeta |
| Log del servidor cuando falla | `after_script` + `artifacts.when: always` | `after-script` (siempre corre) + artifacts (siempre se recogen) |
| Runner propio | `gitlab-runner` registrado con `tags:` | *self-hosted runner* con `runs-on:` |
| Minutos gratis | 400/mes (gitlab.com Free) | 50/mes (Bitbucket Free) |
| Dónde desplegaste | en ningún lado (offline) | AWS Lambda + API Gateway con Serverless v4 |

---

## Checklist de Éxito

- [ ] Rama `feature/utec-4-bitbucket-pipelines` creada y subida a Bitbucket (`origin`)
- [ ] `serverless.yml` con `frameworkVersion: '4'`, `custom.pythonRequirements` y `package.patterns`
- [ ] `npx serverless offline` funciona en local con v4 (sesión iniciada)
- [ ] `SERVERLESS_ACCESS_KEY` y `AWS_DEFAULT_REGION` como Repository variables; llaves AWS como Deployment variables en `test`
- [ ] `test` y `offline` corren en paralelo y en verde; pestaña **Tests** muestra 11 tests
- [ ] Segunda ejecución más rápida gracias a los caches `pip` y `npm`
- [ ] Stage `deploy-test` en pausa hasta aprobarlo manualmente
- [ ] Log de `deploy` muestra `Service deployed to stack utec-lambda-demo-test`, los endpoints y el JSON de `/hello` y `/add`
- [ ] El endpoint responde desde tu laptop/navegador: `{"message":"Hello, <tu-nombre>!","source":"FastAPI"}`
- [ ] Stack y recursos visibles en CloudFormation, Lambda, API Gateway y CloudWatch Logs
- [ ] Pipeline de PR ejecuta solo `test` + `offline`
- [ ] Secretos aparecen como `****` en los logs
- [ ] `custom: remove-test` elimina el stack

---

## Entregables

1. **Repositorio en Bitbucket** con la rama `feature/utec-4-bitbucket-pipelines`:
   - `bitbucket-pipelines.yml` con los steps `test`, `offline`, `deploy`, `remove` y las secciones `default`, `branches`, `pull-requests`, `custom`
   - `serverless.yml` migrado a v4 con `pythonRequirements` y `package.patterns`
   - `package.json` / `package-lock.json` con `serverless@4` y `serverless-offline@14`
2. **Capturas de pantalla:**
   - Pipeline con `test` y `offline` en paralelo y el stage `deploy-test` en pausa
   - Pestaña **Tests** del step `test` con los 11 tests
   - Log del step `deploy` con los endpoints y las líneas `API_URL=…` y el JSON de `/hello`
   - Respuesta del endpoint desde `curl` o el navegador con tu nombre
   - Stack `utec-lambda-demo-test` en CloudFormation con sus recursos
   - Log group en CloudWatch con una línea `REPORT … Duration`
   - Repository variables y Deployment variables configuradas (sin mostrar valores)
   - Pipeline `custom: remove-test` en verde y CloudFormation sin el stack
3. **Reflexión:**
   - ¿Por qué `SERVERLESS_ACCESS_KEY` es Repository variable y las llaves de AWS son Deployment variables?
   - ¿Qué garantiza que el ZIP que llega a Lambda tenga binarios compatibles (`pydantic-core`)? Nombra los tres ajustes que lo aseguran.
   - ¿Qué diferencias encontraste entre `needs:` de GitLab y el orden de lista + `parallel:` de Bitbucket?
   - ¿Qué recursos crea Serverless Framework por ti y cuáles tendrías que crear a mano sin él?

---

## Resumen de comandos

| Paso | Comando / Acción |
|---|---|
| Rama | `git checkout -b feature/utec-4-bitbucket-pipelines` |
| Remoto | `git remote rename origin gitlab` · `git remote add origin git@bitbucket.org:<workspace>/gitlab_lambda_demo.git` · `git push -u origin <rama>` |
| Migrar a v4 | `npm install serverless@^4.43.0 serverless-offline@^14.8.3 --save-dev` |
| Sesión local | `npx serverless login` · `npx serverless print` |
| Offline | `npx serverless offline` → `curl "http://localhost:3000/hello?name=UTEC"` |
| Validar YAML | `python3 -c "import yaml; yaml.safe_load(open('bitbucket-pipelines.yml'))"` · validador en bitbucket-pipelines.atlassian.io/validator |
| Variables | Repository settings → Pipelines → Repository variables / Deployments → Test, Production |
| Deploy | Pipelines → stage `deploy-test` → **Run** |
| Ver endpoints | artifact `deploy-info.txt` o `npx serverless info --stage test` |
| Logs | CloudWatch → `/aws/lambda/utec-lambda-demo-test-api` · `npx serverless logs -f api --stage test --tail` |
| Limpiar | Pipelines → Run pipeline → `custom: remove-test` |

---

## Referencias

- [Bitbucket Pipelines — Configuration reference](https://support.atlassian.com/bitbucket-cloud/docs/bitbucket-pipelines-configuration-reference/)
- [Pipeline start conditions (`default`, `branches`, `pull-requests`, `custom`)](https://support.atlassian.com/bitbucket-cloud/docs/pipeline-start-conditions/)
- [Stage options](https://support.atlassian.com/bitbucket-cloud/docs/stage-options/)
- [Step options](https://support.atlassian.com/bitbucket-cloud/docs/step-options/)
- [YAML anchors](https://support.atlassian.com/bitbucket-cloud/docs/yaml-anchors/)
- [Variables and secrets](https://support.atlassian.com/bitbucket-cloud/docs/variables-and-secrets/)
- [Set up and monitor deployments](https://support.atlassian.com/bitbucket-cloud/docs/set-up-and-monitor-deployments/)
- [Caches](https://support.atlassian.com/bitbucket-cloud/docs/cache-dependencies/)
- [Test reporting in Pipelines](https://support.atlassian.com/bitbucket-cloud/docs/test-reporting-in-pipelines/)
- [Serverless Framework v4 — `serverless.yml` (AWS)](https://www.serverless.com/framework/docs/providers/aws/guide/serverless.yml)
- [Serverless Framework v4 — Python packaging (`pythonRequirements`)](https://www.serverless.com/framework/docs/providers/aws/guide/python)
- [Serverless Framework v4 — Getting started (login, Access Keys y License Keys para CI/CD)](https://www.serverless.com/framework/docs/getting-started)
- [`serverless info`](https://www.serverless.com/framework/docs/providers/aws/cli-reference/info)
- [serverless-offline](https://github.com/dherault/serverless-offline)
- [Mangum](https://mangum.fastapiexpert.com/)
- [AWS Lambda — Python runtime](https://docs.aws.amazon.com/lambda/latest/dg/lambda-python.html)
- [AWS Lambda — límites de despliegue (250 MB)](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)

---

📘 **Autor:**  
Wilson Julca Mejía  
Curso: *DevOps y Bitbucket Pipelines – FastAPI y AWS Lambda*  
Continuación de: `laboratorio_gitlab/guia_gitlab_ci_runner.md`  
Universidad de Ingeniería y Tecnología (UTEC)
