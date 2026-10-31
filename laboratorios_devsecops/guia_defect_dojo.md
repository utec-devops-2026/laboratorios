# Guía de Laboratorio: Gestión Centralizada de Hallazgos con DefectDojo

**Duración estimada:** 90–120 min

**Nivel:** Intermedio

**Contexto:** Este laboratorio cubre la instalación local de **DefectDojo** mediante Docker Compose, su configuración inicial, la carga manual de hallazgos generados por herramientas SAST/SCA/Secrets y la automatización de esa carga desde un pipeline de **GitLab CI/CD**. DefectDojo actúa como plataforma centralizada de gestión de vulnerabilidades, normalizando resultados de distintas herramientas en un único panel. Se usa el proyecto `juice-shop-devsecops` con reportes de Semgrep, Trivy, Gitleaks y TruffleHog.

---

## 🎯 Objetivos de aprendizaje

- Desplegar DefectDojo localmente con Docker Compose.
- Configurar productos, engagements y pruebas dentro de la plataforma.
- Cargar hallazgos manualmente desde reportes JSON nativos de Semgrep, Trivy, Gitleaks y TruffleHog.
- Automatizar la ingesta de hallazgos desde un job de GitLab CI/CD usando la API REST.
- Interpretar el dashboard de deuda técnica de seguridad.

---

## 🧩 Requisitos previos

✅ Docker Desktop instalado y corriendo.

✅ Docker Compose v2 disponible (`docker compose version`).

✅ Al menos **8 GB RAM** asignados a Docker Desktop.

✅ Proyecto `juice-shop-devsecops` con reportes generados de fases anteriores.

✅ Acceso a GitLab con permisos para editar variables CI/CD.

> **Nota:** DefectDojo requiere varios contenedores corriendo simultáneamente. En máquinas con menos de 8 GB puede ser lento en el arranque inicial (2–5 minutos).

---

## Parte 1: Instalación local con Docker Compose

### 1.1 Clonar el repositorio oficial

```bash
git clone https://github.com/DefectDojo/django-DefectDojo.git
cd django-DefectDojo
```

### 1.2 Variables de entorno

No se requiere copiar ningún archivo `.env`. El `docker-compose.yml` tiene todos los valores por defecto hardcodeados con la sintaxis `${VAR:-default}`, lo que es suficiente para uso local en laboratorio.

> **Para producción:** define un `.env` en la raíz del repo con las variables que quieras sobreescribir, por ejemplo `DD_SECRET_KEY=tu-clave-segura`.

### 1.3 Iniciar DefectDojo

```bash
docker compose -f docker-compose.yml -f docker-compose.override.dev.yml up -d --build
```

> No uses `--profile postgres-redis` — la versión actual no tiene perfiles definidos. Todos los servicios arrancan con el comando base.

> **Importante:** usa siempre `docker-compose.override.dev.yml` y `--build`. Las imágenes `defectdojo/defectdojo-django:latest` publicadas en Docker Hub pueden estar **desactualizadas respecto al código local** (este repo). Si solo corres `docker compose up -d`, se descarga la imagen pública vieja y puede fallar el `initializer` por migraciones que no coinciden con el código (ver troubleshooting).


Este comando levanta los siguientes servicios:

| Servicio | Rol |
|----------|-----|
| `nginx` | Proxy reverso, expone puerto 8080 |
| `uwsgi` | Aplicación Django |
| `celerybeat` + `celeryworker` | Tareas asíncronas (notificaciones, deduplicación) |
| `postgres` | Base de datos |
| `valkey` | Cola de mensajes (reemplazó a Redis desde 2024) |
| `initializer` | Crea la BD y usuario admin en primer arranque |

El `initializer` puede tardar **2–5 minutos** en el primer arranque mientras crea la base de datos y las migraciones.

### 1.4 Verificar que los contenedores estén corriendo

```bash
docker compose ps
```

Salida esperada (todos en estado `Up`, `initializer` puede mostrar `Exited (0)` — es normal):

```
NAME                           STATUS
django-defectdojo-nginx-1      Up
django-defectdojo-uwsgi-1      Up
django-defectdojo-celerybeat-1 Up
django-defectdojo-celeryworker-1 Up
django-defectdojo-postgres-1   Up
django-defectdojo-valkey-1     Up
django-defectdojo-initializer-1 Exited (0)
```

### 1.5 Credenciales del admin

Con `docker-compose.override.dev.yml`, el usuario y contraseña del admin son **fijos** (no se generan al azar):

- **Usuario:** `admin`
- **Contraseña:** `admin`

(definidos en `DD_ADMIN_USER` / `DD_ADMIN_PASSWORD`, por defecto `admin`/`admin`).

> Si usas el `docker-compose.yml` base sin override, sí se genera una contraseña aleatoria. En ese caso obténla con:
> ```bash
> docker compose logs initializer | grep -i "admin user\|password"
> ```

### 1.6 Acceder a la interfaz web

Abre el navegador en: `http://localhost:8080`

- **Usuario:** `admin`
- **Contraseña:** `admin` (o la obtenida en el paso anterior)

---

## Parte 2: Configuración inicial de la plataforma

DefectDojo organiza los hallazgos en una jerarquía: **Product Type → Product → Engagement → Test → Findings**.

> **Nombres en la UI (DefectDojo 3.x):** desde la versión 3.0 la interfaz renombró **Product Type → Organization** y **Product → Asset**. La API REST conserva los nombres antiguos (`/api/v2/product_types/`, `/api/v2/products/`). Ambos conceptos aparecen bajo el menú lateral **Assets**. Si tu instancia muestra `Products` / `Product Types`, tiene desactivada la variable `DD_ENABLE_V3_ORGANIZATION_ASSET_RELABEL`; los pasos son los mismos.

### 2.1 Crear un Product Type (Organization)

1. Ve a **Assets → Add Organization** en el menú lateral (en versiones 2.x: **Product Types → + Add Product Type**).
2. Ingresa:
   - **Name:** `Aplicaciones Web`
   - **Description:** `Portafolio de aplicaciones web internas`
3. Clic en **Submit**.

### 2.2 Crear un Product (Asset)

1. Ve a **Assets → Add Asset** (en versiones 2.x: **Products → + Add Product**).
2. Ingresa:
   - **Name:** `Juice Shop`
   - **Description:** `Aplicación de e-commerce vulnerable para laboratorio`
   - **Organization / Product Type:** `Aplicaciones Web`
   - **Business Criticality:** `High`
   - **Platform:** `Web`
   - **Lifecycle:** `Production`
3. Clic en **Submit**.

### 2.3 Crear un Engagement

Un Engagement agrupa todas las pruebas de seguridad de un sprint o release.

> El menú lateral **Engagements** solo lista; no tiene botón de creación. El engagement se crea desde el Asset.

1. Ve a **Assets → All Assets → Juice Shop**.
2. En la barra de pestañas del asset, abre **Engagements → Add New CI/CD Engagement** (en versiones 2.x: **+ Add Engagement**).
3. Ingresa:
   - **Name:** `Sprint 1 - Análisis de seguridad`
   - **Status:** `In Progress`
   - **Target Start / Target End:** fechas del sprint actual
4. Clic en **Submit**.

---

## Parte 3: Obtener el API Key

Para automatizar la ingesta necesitas un token de autenticación.

1. Clic en el usuario `admin` (esquina superior derecha).
2. Ve a **API v2 Key**.
3. Copia el token — lo usarás como variable en GitLab CI/CD.

También puedes obtenerlo vía CLI:

```bash
curl -X POST http://localhost:8080/api/v2/api-token-auth/ \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "TU_PASSWORD"}' | jq '.token'
```

### 3.1 Obtener IDs necesarios para la API

Necesitas el ID del **Product** (el engagement se crea automáticamente en el pipeline):

```bash
export DD_TOKEN="TU_API_TOKEN"
export DD_URL="http://localhost:8080"

# Listar productos — anota el ID de "Juice Shop"
curl -s -H "Authorization: Token $DD_TOKEN" \
  "$DD_URL/api/v2/products/" | jq '.results[] | {id, name}'

# Verificar engagements creados (referencia)
curl -s -H "Authorization: Token $DD_TOKEN" \
  "$DD_URL/api/v2/engagements/?product=1" | jq '.results[] | {id, name, status}'
```

Anota el **product ID** — lo usarás como variable `DEFECTDOJO_PRODUCT_ID` en GitLab.

---

## Parte 4: Automatización desde GitLab CI/CD

### 4.1 Configurar variables en GitLab

En tu proyecto GitLab, ve a **Settings → CI/CD → Variables** y agrega:

| Variable | Valor | Protegida | Enmascarada |
|----------|-------|-----------|-------------|
| `DEFECTDOJO_URL` | URL pública de ngrok (ver abajo) | No | No |
| `DEFECTDOJO_TOKEN` | `tu-api-token` | Sí | Sí |
| `DEFECTDOJO_PRODUCT_ID` | `1` (ID del producto Juice Shop) | No | No |

> **Los runners SaaS de GitLab.com corren en la nube** — no pueden alcanzar `localhost` ni `host.docker.internal`. Necesitas exponer DefectDojo con un túnel público.

**Exponer DefectDojo local con ngrok:**

```bash
brew install ngrok
ngrok config add-authtoken TU_TOKEN_DE_NGROK   # cuenta gratis en ngrok.com
ngrok http 8080
# Copia la URL: https://abc123.ngrok-free.app
```

Usa esa URL como valor de `DEFECTDOJO_URL` en GitLab. La URL cambia cada vez que reinicias ngrok — actualiza la variable en cada sesión.

> ⚠️ **Protegida = solo ramas protegidas.** Una variable marcada *Protected* no se entrega a pipelines de ramas que no estén en Settings → Repository → Protected branches (por defecto solo `master`/`main`). Si trabajas en ramas `feature/…`, el job `upload-defectdojo` fallará con `Variable requerida no definida: DEFECTDOJO_TOKEN`. Protege tus ramas de trabajo con el wildcard `feature/*` (Allowed to push and merge: Developers + Maintainers) o desmarca *Protected* en la variable.

> El engagement se crea **automáticamente por pipeline** usando `CI_PIPELINE_ID` y `CI_COMMIT_REF_NAME`, lo que permite trazabilidad por build.

### 4.2 Script de importación reutilizable

El archivo `scripts/upload_to_defectdojo.sh`, maneja:
- Validación de archivo existente y no vacío
- Limpieza automática del NDJSON de TruffleHog (escape sequences)
- Respuesta HTTP con mensaje de error detallado

```bash
#!/bin/bash
# Sube un reporte a DefectDojo via API REST usando reimport-scan.
# Uso: ./upload_to_defectdojo.sh <scan_type> <report_file>
#
# reimport-scan + engagement/test fijos por rama permite que DefectDojo
# deduplique hallazgos entre ejecuciones (mismo hash_code -> mismo finding,
# no se duplica en cada pipeline).
#
# Variables de entorno requeridas:
#   DEFECTDOJO_URL            URL base (ej: http://localhost:8080)
#   DEFECTDOJO_TOKEN          API token
#   DEFECTDOJO_PRODUCT_ID     ID del producto destino

set -e

SCAN_TYPE="$1"
REPORT_FILE="$2"

if [[ -z "$SCAN_TYPE" || -z "$REPORT_FILE" ]]; then
  echo "Uso: $0 <scan_type> <report_file>"
  exit 1
fi

if [[ ! -f "$REPORT_FILE" ]]; then
  echo "Archivo no encontrado: $REPORT_FILE — omitiendo"
  exit 0
fi

if [[ ! -s "$REPORT_FILE" ]]; then
  echo "Archivo vacío: $REPORT_FILE — omitiendo"
  exit 0
fi

for var in DEFECTDOJO_URL DEFECTDOJO_TOKEN DEFECTDOJO_PRODUCT_ID; do
  if [[ -z "${!var}" ]]; then
    echo "Variable requerida no definida: $var"
    exit 1
  fi
done

UPLOAD_FILE="$REPORT_FILE"
CLEAN_FILE=""

# TruffleHog genera NDJSON con secuencias de escape al inicio.
# DefectDojo espera líneas JSON limpias; se normaliza antes de subir.
if [[ "$SCAN_TYPE" == "Trufflehog Scan" ]]; then
  CLEAN_FILE="/tmp/trufflehog-clean-$$.json"
  # Filtrar solo líneas con hallazgos reales (tienen SourceMetadata)
  grep '"SourceMetadata"' "$REPORT_FILE" | \
    sed 's/\r//' | \
    grep -v '^[^{]' > "$CLEAN_FILE" || true

  if [[ ! -s "$CLEAN_FILE" ]]; then
    echo "TruffleHog: sin hallazgos válidos tras limpieza — omitiendo"
    rm -f "$CLEAN_FILE"
    exit 0
  fi
  UPLOAD_FILE="$CLEAN_FILE"
fi

ENGAGEMENT_NAME="CI - ${CI_COMMIT_REF_NAME}"
TEST_TITLE="$SCAN_TYPE"

# reimport-scan con auto_create_context requiere product_name (no basta product_id)
# para resolver/crear el engagement.
PRODUCT_NAME=$(curl -sf \
  -H "Authorization: Token $DEFECTDOJO_TOKEN" \
  "$DEFECTDOJO_URL/api/v2/products/$DEFECTDOJO_PRODUCT_ID/" \
  | grep -o '"name":"[^"]*"' | head -1 | cut -d'"' -f4)

if [[ -z "$PRODUCT_NAME" ]]; then
  echo "No se pudo resolver el nombre del producto (DEFECTDOJO_PRODUCT_ID=$DEFECTDOJO_PRODUCT_ID)"
  exit 1
fi

echo "Subiendo '$REPORT_FILE' como '$SCAN_TYPE' -> producto '$PRODUCT_NAME' / engagement '$ENGAGEMENT_NAME' / test '$TEST_TITLE'..."

RESPONSE=$(curl -s -w "\n%{http_code}" \
  -X POST "$DEFECTDOJO_URL/api/v2/reimport-scan/" \
  -H "Authorization: Token $DEFECTDOJO_TOKEN" \
  -F "scan_type=$SCAN_TYPE" \
  -F "file=@$UPLOAD_FILE" \
  -F "product_id=$DEFECTDOJO_PRODUCT_ID" \
  -F "product_name=$PRODUCT_NAME" \
  -F "engagement_name=$ENGAGEMENT_NAME" \
  -F "test_title=$TEST_TITLE" \
  -F "auto_create_context=true" \
  -F "active=true" \
  -F "verified=false" \
  -F "close_old_findings=true" \
  -F "close_old_findings_product_scope=false" \
  -F "version=$CI_PIPELINE_ID" \
  -F "build_id=$CI_PIPELINE_ID" \
  -F "commit_hash=$CI_COMMIT_SHORT_SHA" \
  -F "branch_tag=$CI_COMMIT_REF_NAME")

HTTP_STATUS=$(echo "$RESPONSE" | tail -1)
BODY=$(echo "$RESPONSE" | head -n -1)

if [[ -n "$CLEAN_FILE" ]]; then
  rm -f "$CLEAN_FILE"
fi

if [[ "$HTTP_STATUS" == "201" ]]; then
  echo "OK importado (HTTP 201) — $SCAN_TYPE"
else
  echo "Error al importar $SCAN_TYPE (HTTP $HTTP_STATUS)"
  echo "Respuesta: $BODY"
  exit 1
fi

```



### 4.3 Stage defectdojo en el pipeline existente

El `.gitlab-ci.yml` agregar el stage `defectdojo`.

```yaml
stages:
  - prepare
  - secret_scanning
  - sca
  - sast
  - defectdojo
```

Tambien agregar el job `upload-defectdojo`:

1. Espera que terminen `gitleaks-scan`, `trufflehog-scan`, `trivy-scan` y `semgrep-scan`.
2. Crea un engagement nuevo por pipeline usando `CI_PIPELINE_ID` y `CI_COMMIT_SHORT_SHA`.
3. Sube los 4 reportes con los scan types correctos.

```yaml
upload-defectdojo:
  image: alpine/curl:latest
  stage: defectdojo
  needs:
    - gitleaks-scan
    - trufflehog-scan
    - trivy-scan
    - semgrep-scan
  dependencies:
    - gitleaks-scan
    - trufflehog-scan
    - trivy-scan
    - semgrep-scan
  variables:
    REPORTS_DIR: security-reports
  before_script:
    - apk add --no-cache bash
    - chmod +x scripts/upload_to_defectdojo.sh
  script:
    - bash scripts/upload_to_defectdojo.sh "Gitleaks Scan"       "gitleaks-report.json"
    - bash scripts/upload_to_defectdojo.sh "Trufflehog Scan"     "trufflehog-report.json"
    - bash scripts/upload_to_defectdojo.sh "Trivy Scan"          "$REPORTS_DIR/trivy-report.json"
    - bash scripts/upload_to_defectdojo.sh "Semgrep JSON Report" "semgrep-report.json"
  rules:
    - if: $DEFECTDOJO_URL == null
      when: never
    - when: always
  allow_failure: true
```

> **Notas de implementación:**
> - La creación del engagement va en `script`, no en `before_script` — si falla la conexión, el error es visible y el job termina limpiamente con `exit 1`.
> - `curl -sf --max-time 15` — falla rápido (15 s timeout) si DefectDojo no responde.
> - Las reglas usan `when: always` en lugar de filtrar por branch — el job corre en cualquier rama mientras `DEFECTDOJO_URL` esté definida.
> - Los paths de artifacts: Gitleaks y TruffleHog en raíz, Trivy en `security-reports/`, Semgrep en raíz.

---
### 4.4 Hacer que cada job se marque en rojo (error) si hay hallazgos, sin frenar el pipeline

Objetivo: si un scanner encuentra hallazgos, su job debe verse **rojo (failed)**, no naranja (warning) — pero el reporte debe generarse igual y `upload-defectdojo` debe seguir corriendo.

Para lograrlo en cada job de escaneo:
1. **Quitar `|| true`** del comando que determina el resultado del scan, para que el exit code real (1 = hallazgos) se propague al job.
2. **Quitar `allow_failure: true`** — así el job falla "de verdad" (rojo) en vez de quedar como warning.
3. **Agregar `artifacts.when: always`** — el reporte se sube como artifact aunque el job falle.
4. El job ya tiene `rules: - when: always`, así que se ejecuta igual aunque la rama/pipeline tenga jobs en rojo, y `upload-defectdojo` (también con `rules: - when: always`) corre y sube el reporte a DefectDojo sin problema.

**Gitleaks:**
Cambiar
```
    - ./gitleaks detect --source . --report-path gitleaks-report.json || true
```
por
```
    - ./gitleaks detect --source . --report-path gitleaks-report.json
```
y en `artifacts`, agregar `when: always`; quitar `allow_failure: true`.

**TruffleHog:**
Cambiar
```
    - ./trufflehog filesystem . --results=verified,unknown --json > trufflehog-report.json || true
```
por
```
    - ./trufflehog filesystem . --results=verified,unknown --json --fail > trufflehog-report.json
```
(`--fail` hace que TruffleHog devuelva exit code distinto de 0 si encontró secretos). En `artifacts`, agregar `when: always`; quitar `allow_failure: true`.

**Trivy:**
La última línea ya usa `--exit-code 1` sin `|| true`:
```
    - trivy filesystem . --scanners vuln --severity HIGH,CRITICAL --exit-code 1 --format table
```
Dejarla así (no agregar `|| true`). En `artifacts`, agregar `when: always`; quitar `allow_failure: true`.

**Semgrep:**
Las dos primeras líneas (que generan los reportes) se dejan con `|| true` para que el reporte siempre se genere. Se agrega una tercera línea solo para fallar el job si hay hallazgos:
```
    - semgrep --config "p/ci" --config "p/security-audit" --config "p/javascript" --json --output semgrep-report.json "$SEMGREP_SRC" || true
    - semgrep --config "p/ci" --config "p/security-audit" --config "p/javascript" "$SEMGREP_SRC" > semgrep-report.txt || true
    - semgrep --config "p/ci" --config "p/security-audit" --config "p/javascript" --error "$SEMGREP_SRC" > /dev/null
```
(`--error` hace que Semgrep devuelva exit code 1 si hay findings). En `artifacts`, agregar `when: always`; quitar `allow_failure: true`.

---

## Parte 5: Verificar resultados en el dashboard

### 5.1 Ver resumen del producto

1. Ve a **Assets → All Assets → Juice Shop**.
2. Observa el panel con:
   - Total de findings por severidad
   - Tendencia en el tiempo
   - Findings por herramienta de origen

### 5.2 Analizar deuda técnica

1. Ve a **Assets → All Assets → Juice Shop** y abre la pestaña **Metrics**.
3. Revisa:
   - **Open findings** por severidad acumulados
   - **Closed/Accepted findings** por sprint
   - **Age of findings** — hallazgos antiguos sin remediar

### 5.3 Gestionar un hallazgo individual

1. Abre cualquier finding de severidad `Critical` o `High`.
2. Acciones disponibles:
   - **Risk Acceptance** — aceptar el riesgo con justificación y fecha de expiración
   - **False Positive** — marcar como falso positivo
   - **Assign** — asignar a un miembro del equipo
   - **Add Note** — agregar contexto o plan de remediación
   - **Link to JIRA** — si tienes la integración configurada

---

## Parte 6: Detener el entorno

Cuando termines el laboratorio:

```bash
# Detener sin borrar datos
docker compose down

# Detener Y borrar volúmenes (reset completo)
docker compose down -v
```

### 6.1 Reiniciar el entorno desde cero

Si necesitas un reset completo (ej. tras cambios en migraciones o errores de inicialización), borra volúmenes y reconstruye:

```bash
docker compose -f docker-compose.yml -f docker-compose.override.dev.yml down -v
docker compose -f docker-compose.yml -f docker-compose.override.dev.yml up -d --build
```

---

## 📋 Resumen de tipos de escaneo soportados

| Herramienta | Scan Type en DefectDojo |
|-------------|------------------------|
| Semgrep (JSON nativo `--json`) | `Semgrep JSON Report` |
| Semgrep (SARIF `--sarif`) | `SARIF` |
| Trivy (filesystem) | `Trivy Scan` |
| Trivy (container) | `Trivy Scan` |
| Gitleaks | `Gitleaks Scan` |
| OWASP ZAP | `ZAP Scan` |
| Snyk | `Snyk Scan` |
| Bandit | `Bandit Scan` |
| Checkov (IaC) | `Checkov Scan` |
| TruffleHog | `Trufflehog Scan` |
| Grype | `Anchore Grype` |

> Lista completa en: `http://localhost:8080/api/v2/#/import-scan/import_scan_create` → campo `scan_type`.

---

## 🔧 Troubleshooting común

**DefectDojo no arranca / contenedores en `Restarting`:**
```bash
# Ver logs del inicializador
docker compose logs initializer

# Ver logs de la app
docker compose logs uwsgi | tail -50

# El servicio de mensajería ahora se llama valkey (no redis)
docker compose logs valkey | tail -20
```

**Error 400 al importar via API:**
- Verifica que el `scan_type` coincida exactamente con el nombre esperado (sensible a mayúsculas).
- Para Semgrep: usar `Semgrep JSON Report` con archivos `--json`, no `SARIF`.
- Para TruffleHog: el archivo debe contener solo líneas NDJSON con `SourceMetadata` — el script las filtra automáticamente.
- Verifica que el formato del archivo sea válido para ese tipo de escaneo.

**Error 401 Unauthorized:**
- Verifica que el token sea correcto y no tenga espacios extra.
- Asegúrate de usar `Token <valor>` y no `Bearer <valor>`.

**Stage `defectdojo` no aparece en el pipeline:**
- Las `rules` del job lo están omitiendo. Si usas ramas distintas a `main`/`develop`, cambia a `when: always` con guardia `$DEFECTDOJO_URL == null`.
- Si `DEFECTDOJO_URL` no está definida como variable GitLab, el job se omite silenciosamente.

**`ERROR: No se pudo conectar a DefectDojo`:**
- Los runners SaaS de GitLab.com no alcanzan `localhost`. Usa ngrok u otro túnel público.
- Verifica que ngrok siga corriendo y que la URL en la variable de GitLab sea la actual.

**`mktemp: : Invalid argument` en Alpine:**
- Alpine usa BusyBox `mktemp` que no acepta sufijos. El script usa `$$` (PID del proceso) para generar nombres únicos: `/tmp/trufflehog-clean-$$.json`.

**`File extension "" is not allowed`:**
- DefectDojo rechaza archivos sin extensión. Causa: archivo temporal sin `.json`. Resuelto con la corrección de `mktemp` anterior.

**`initializer` crashea en loop con `relation "dojo_cred_user" does not exist` / `Failed to enable pghistory triggers`:**
- Causa: la imagen `defectdojo/defectdojo-django:latest` de Docker Hub está desactualizada respecto al código local (este repo ya eliminó el modelo `Cred_User` en la migración `0266_remove_credential_manager`, pero la imagen pública aún lo registra para `pgtrigger`).
- Solución: reconstruir la imagen desde el código local y resetear datos:
  ```bash
  docker compose -f docker-compose.yml -f docker-compose.override.dev.yml down -v
  docker compose -f docker-compose.yml -f docker-compose.override.dev.yml up -d --build
  ```
- Verifica que terminó bien: `docker compose ps -a initializer` debe mostrar `Exited (0)`.

**`Bind for 0.0.0.0:5432 failed: port is already allocated`:**
- Causa: `docker-compose.override.dev.yml` publica el puerto 5432 de Postgres, y otro contenedor (de otro proyecto) ya lo usa.
- Solución A: detener el otro contenedor que ocupa 5432 (`docker stop <nombre>`).
- Solución B: usar otro puerto para DefectDojo:
  ```bash
  DD_DATABASE_PORT=5433 docker compose -f docker-compose.yml -f docker-compose.override.dev.yml up -d
  ```

**Hallazgos duplicados:**
- Usa `close_old_findings=true` en la llamada API para que DefectDojo cierre automáticamente hallazgos de importaciones anteriores que ya no aparezcan.
- DefectDojo usa un hash de deduplicación por `title + severity + file_path + line`.

---

## 📎 Anexo: Carga manual de hallazgos

Esta sección es **opcional** — útil para explorar la UI de DefectDojo antes de automatizar con CI/CD (Parte 4). El pipeline ya sube los reportes automáticamente, así que no es necesario repetir estos pasos si solo te interesa la automatización.

### A.1 Reportes del proyecto

El proyecto `juice-shop-devsecops` ya incluye reportes generados en fases anteriores:

```
juice-shop-devsecops/reports/
├── semgrep-report.json      # SAST — formato JSON nativo de Semgrep (26 hallazgos)
├── trivy-report.json        # Secrets en código e IaC detectados por Trivy
├── gitleaks-report.json     # Secret scanning — 93 hallazgos
└── trufflehog-report.json   # Secret scanning — NDJSON con escape sequences
```

> **Nota sobre formatos:** Semgrep genera JSON nativo (no SARIF) con la bandera `--json`. TruffleHog genera NDJSON con secuencias de escape al inicio del archivo — DefectDojo requiere que estas líneas sean filtradas antes de importar.

### A.2 Importar reporte Semgrep (SAST)

1. Dentro del Engagement creado, clic en **+ Add Tests**.
2. Configura la prueba:
   - **Test Type:** `Semgrep JSON Report`
   - **Environment:** `Development`
   - **Version:** `1.0`
3. Clic en **Submit**.
4. Dentro del test creado, clic en **Import Scan Results**.
5. Selecciona:
   - **Scan Type:** `Semgrep JSON Report`
   - **File:** selecciona `reports/semgrep-report.json`
   - **Active:** habilitado
6. Clic en **Import**.

> **Importante:** El scan type es `Semgrep JSON Report`, no `SARIF`. Usar SARIF con un archivo JSON nativo de Semgrep producirá error 400.

### A.3 Importar reporte Trivy

1. Crea un nuevo test dentro del mismo Engagement.
   - **Test Type:** `Trivy Scan`
   - **Environment:** `Development`
2. En **Import Scan Results**:
   - **Scan Type:** `Trivy Scan`
   - **File:** selecciona `reports/trivy-report.json`
3. Clic en **Import**.

> El reporte de Trivy en este proyecto contiene hallazgos de clase `secret` (no vulnerabilidades de dependencias), ya que se ejecutó sobre el filesystem sin imagen de contenedor.

### A.4 Importar reporte Gitleaks (Secrets)

1. Crea un nuevo test:
   - **Test Type:** `Gitleaks Scan`
   - **Environment:** `Development`
2. En **Import Scan Results**:
   - **Scan Type:** `Gitleaks Scan`
   - **File:** selecciona `reports/gitleaks-report.json`
3. Clic en **Import**.

### A.5 Importar reporte TruffleHog (Secrets)

TruffleHog genera NDJSON con secuencias de escape terminales al inicio. Antes de importar, limpia el archivo:

```bash
# Filtrar solo líneas con hallazgos reales (tienen "SourceMetadata")
grep '"SourceMetadata"' reports/trufflehog-report.json | \
  sed 's/\r//' > /tmp/trufflehog-clean.json

# Verificar que el archivo no esté vacío
wc -l /tmp/trufflehog-clean.json
```

Luego importa:
1. Crea un nuevo test:
   - **Test Type:** `Trufflehog Scan`
   - **Environment:** `Development`
2. En **Import Scan Results**:
   - **Scan Type:** `Trufflehog Scan`
   - **File:** selecciona `/tmp/trufflehog-clean.json`
3. Clic en **Import**.

### A.6 Verificar hallazgos importados

1. Ve a **Findings** en el menú lateral.
2. Aplica filtros por:
   - **Severity:** `Critical`, `High`
   - **Asset / Product:** `Juice Shop`
3. Explora un hallazgo para ver:
   - Descripción del vulnerabilidad
   - Archivo y línea afectados
   - Herramienta de origen
   - CVSS Score (cuando aplica)
   - Referencias CWE / CVE
