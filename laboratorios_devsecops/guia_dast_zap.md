# Laboratorio: DAST con OWASP ZAP sobre Juice Shop e integración al pipeline DevSecOps

**Duración estimada:** 120–150 min (el escaneo activo completo es opcional y puede superar los 30 min)  
**Nivel:** Intermedio / Avanzado  
**Contexto:** El laboratorio `guia_secret_scanning_sca_sast.md` dejó un pipeline con secret scanning, SCA y SAST sobre `juice-shop-devsecops`. Todas esas herramientas analizan el **código sin ejecutarlo**. Aquí agregas la capa que falta: **DAST** (Dynamic Application Security Testing) con OWASP ZAP contra la aplicación **corriendo**, primero a mano y luego como stage `dynamic_scanning` del mismo `.gitlab-ci.yml`, con gate por reglas, escaneo de API por OpenAPI y carga de hallazgos a DefectDojo.

> Si nunca usaste ZAP, haz primero `ejemplo_intro_dast.md` (20 min): explica el baseline, el exit code y `rules.conf` sobre una app de 20 líneas.

---

## 🎯 Objetivos de aprendizaje

- Ejecutar los tres escaneos empaquetados de ZAP (**baseline**, **API**, **full**) desde Docker contra Juice Shop
- Interpretar alertas pasivas (cabeceras, CORS, cookies) y activas (inyección, errores 5xx)
- Definir un **gate** con `rules.conf`: qué bloquea el pipeline, qué solo avisa y qué se ignora
- Integrar DAST como stage de **GitLab CI** usando `services` para levantar la app efímera
- Enviar el reporte ZAP a **DefectDojo** junto con los de Gitleaks, TruffleHog, Trivy y Semgrep

---

## 🧩 Requisitos previos

✅ Laboratorio `guia_secret_scanning_sca_sast.md` completado (repo `juice-shop-devsecops` en GitLab con `.gitlab-ci.yml` de 4 stages)  
✅ Docker Desktop u OrbStack corriendo  
✅ `curl` y `python3`  
✅ Para la Parte 7: DefectDojo levantado según `guia_defect_dojo.md` (variables `DEFECTDOJO_*` en GitLab y `scripts/upload_to_defectdojo.sh` en el repo)  
✅ Conectividad para descargar `zaproxy/zap-stable` (~2 GB) y `bkimminich/juice-shop` (~530 MB)

---

## 📁 Qué se agrega al proyecto

```
juice-shop-devsecops/
├── .gitlab-ci.yml            # + stage dynamic_scanning (Parte 6) y upload de ZAP (Parte 7)
├── zap/
│   └── rules.conf            # gate: FAIL / WARN / IGNORE por regla (Parte 3)
├── swagger.yml               # ya existe: contrato OpenAPI de la API B2B (Parte 4)
└── reports/                  # salidas locales de ZAP (no se commitea)
    ├── baseline.{html,json,xml}
    └── api.{html,json}
```

---

## Parte 1: Levantar Juice Shop en una red Docker

ZAP corre en su propio contenedor. Para que alcance la app, ambos deben compartir una red; `localhost` dentro del contenedor de ZAP no es tu máquina.

```bash
cd juice-shop-devsecops
mkdir -p reports zap

docker network create dast-net
docker run -d --name juice-shop --network dast-net -p 3000:3000 bkimminich/juice-shop:latest
```

Juice Shop tarda 20–40 s en arrancar. Espera a que responda:

```bash
until curl -sf -o /dev/null http://localhost:3000/; do sleep 2; done; echo "Juice Shop lista"
```

### 1.1 Observar las cabeceras antes de escanear

```bash
curl -s -I http://localhost:3000/
```

Salida observada:

```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Feature-Policy: payment 'self'
X-Recruiting: /#/jobs
Cache-Control: public, max-age=0
Content-Type: text/html; charset=UTF-8
```

Juice Shop usa `helmet` solo para `noSniff()` y `frameguard()` (míralo en `server.ts` líneas 184–185). No hay CSP, `Feature-Policy` es la versión deprecada de `Permissions-Policy`, y `Access-Control-Allow-Origin: *` abre CORS a cualquier origen. Anota estas tres cosas: ZAP debe encontrarlas.

> **¿Por qué la imagen pública y no el `Dockerfile` del repo?** Para el laboratorio basta la app tal como la publica el proyecto. En la Parte 6 verás cómo escanear la imagen construida desde tu rama.

---

## Parte 2: Baseline scan (pasivo)

### 2.1 Ejecutar

```bash
docker run --rm --network dast-net \
  -v "$(pwd)/reports:/zap/wrk/:rw" \
  -t zaproxy/zap-stable zap-baseline.py \
  -t http://juice-shop:3000 \
  -r baseline.html -J baseline.json -x baseline.xml
echo "exit=$?"
```

| Flag | Propósito |
|---|---|
| `-t http://juice-shop:3000` | target con protocolo, por nombre de contenedor |
| `-r` / `-J` / `-x` | HTML para leer, JSON para procesar, **XML para DefectDojo** (su parser `ZAP Scan` solo acepta XML) |
| `-v …:/zap/wrk/:rw` | ZAP lee/escribe archivos solo en `/zap/wrk` |

> `zaproxy/zap-stable` (Docker Hub) y `ghcr.io/zaproxy/zaproxy:stable` son la misma imagen; si `ghcr.io` devuelve `denied`, usa Docker Hub.

Tiempo observado: **~6,5 min** (ZAP 2.17.0). La mayor parte es escaneo pasivo del bundle Angular: la SPA carga varios `chunk-*.js` grandes y ZAP los analiza todos.

### 2.2 Leer la salida

Salida observada:

```
WARN-NEW: Content Security Policy (CSP) Header Not Set [10038] x 5
WARN-NEW: Storable and Cacheable Content [10049] x 7
WARN-NEW: Deprecated Feature Policy Header Set [10063] x 5
WARN-NEW: Cross-Domain Misconfiguration [10098] x 5
WARN-NEW: Modern Web Application [10109] x 5
WARN-NEW: Dangerous JS Functions [10110] x 2
WARN-NEW: Cross-Origin-Embedder-Policy Header Missing or Invalid [90004] x 10
FAIL-NEW: 0  FAIL-INPROG: 0  WARN-NEW: 7  WARN-INPROG: 0  INFO: 0  IGNORE: 0  PASS: 61
exit=2
```

| ID | Alerta | Riesgo ZAP | Evidencia | Qué significa |
|---|---|---|---|---|
| 10038 | CSP Header Not Set | Medium | — | sin `Content-Security-Policy`: cualquier XSS ejecuta scripts sin restricción |
| 10098 | Cross-Domain Misconfiguration | Medium | `Access-Control-Allow-Origin: *` | cualquier sitio puede leer respuestas de la API desde el navegador |
| 10063 | Deprecated Feature Policy Header Set | Low | `Feature-Policy` | cabecera obsoleta; debe ser `Permissions-Policy` |
| 90004 | COEP / COOP Header Missing | Low | — | sin aislamiento de origen (`Cross-Origin-Embedder-Policy`, `Cross-Origin-Opener-Policy`) |
| 10110 | Dangerous JS Functions | Low | `bypassSecurityTrustHtml(` en `chunk-*.js` | Angular desactiva la sanitización en ese punto: candidato a XSS |
| 10109 | Modern Web Application | Info | `<script>` | ZAP avisa que es una SPA: el spider tradicional ve poco; usar `-j` (AJAX spider) |
| 10049 | Storable and Cacheable Content | Info | `max-age=0` | respuestas cacheables; revisar en endpoints con datos sensibles |

Compara con la Parte 1.1: las tres cosas que anotaste (sin CSP, `Feature-Policy`, CORS `*`) están en la lista. DAST pasivo no inventa nada: formaliza lo que `curl -I` ya mostraba, pero sobre **todas** las URLs descubiertas y con CWE, riesgo y solución.

Lo que **no** aparece: el SQL injection de `routes/login.ts` que Semgrep detectó en el laboratorio anterior, ni ningún XSS. El baseline no envía payloads; eso es territorio del escaneo activo (Partes 4 y 5).

### 2.3 Inspeccionar el JSON

```bash
python3 -c "
import json
d = json.load(open('reports/baseline.json'))
for site in d['site']:
    print('site', site['@name'], '| alertas', len(site['alerts']))
    for a in site['alerts']:
        i = a['instances'][0]
        print(' ', a['pluginid'], a['riskdesc'], '|', a['alert'], '| CWE', a['cweid'], '|', i['uri'][:45], '| ev:', (i.get('evidence') or '')[:40])
"
```

Abre `reports/baseline.html` para ver la descripción y la **solución** de cada alerta.

> **Entregable parcial:** captura de la terminal con el resumen `WARN-NEW … PASS: 61` y una fila de la tabla anterior explicada con tus palabras.

---

## Parte 3: Definir el gate con `rules.conf`

Por defecto toda alerta es WARN → exit 2. Un pipeline necesita decidir: ¿qué bloquea un merge?

### 3.1 Generar la lista completa de reglas

```bash
docker run --rm --network dast-net -v "$(pwd)/reports:/zap/wrk/:rw" \
  -t zaproxy/zap-stable zap-baseline.py -t http://juice-shop:3000 -g gen.conf -s
grep -c . reports/gen.conf      # ~70 reglas, todas WARN
```

> Es otro baseline completo (~6 min): `-g` solo agrega la escritura del archivo. Si ya tienes el `gen.conf` del ejemplo introductorio, reutilízalo: la lista de reglas no depende del target.

### 3.2 Escribir `zap/rules.conf`

Criterio sugerido para este proyecto: **FAIL** en las cabeceras obligatorias y en CORS abierto (las que un equipo de Ethical Hacking observa), **WARN** en buenas prácticas, **IGNORE** en informativas.

```bash
printf '%s\t%s\t%s\n' \
  10038 FAIL "(Content Security Policy (CSP) Header Not Set)" \
  10020 FAIL "(Anti-clickjacking Header)" \
  10021 FAIL "(X-Content-Type-Options Header Missing)" \
  10035 FAIL "(Strict-Transport-Security Header)" \
  10098 FAIL "(Cross-Domain Misconfiguration)" \
  10063 WARN "(Permissions Policy Header Not Set)" \
  10054 WARN "(Cookie without SameSite Attribute)" \
  10110 WARN "(Dangerous JS Functions)" \
  90004 WARN "(Cross-Origin-Embedder-Policy Header Missing or Invalid)" \
  10049 IGNORE "(Storable and Cacheable Content)" \
  10109 IGNORE "(Modern Web Application)" \
  > zap/rules.conf
cat zap/rules.conf
```

> Las columnas van separadas por **tabulador**; con espacios ZAP ignora la línea. `10035` (HSTS) solo aplica a HTTPS: en local no dispara, en producción sí.

### 3.3 Aplicar el gate

El archivo debe estar dentro de `/zap/wrk`, así que cópialo a `reports/`:

```bash
cp zap/rules.conf reports/
docker run --rm --network dast-net -v "$(pwd)/reports:/zap/wrk/:rw" \
  -t zaproxy/zap-stable zap-baseline.py -t http://juice-shop:3000 -c rules.conf -s
echo "exit=$?"
```

Resultado esperado: `FAIL-NEW: Content Security Policy (CSP) Header Not Set [10038]` y `FAIL-NEW: Cross-Domain Misconfiguration [10098]` → **exit 1**. Con este archivo, el pipeline de la Parte 6 bloquea mientras Juice Shop no tenga CSP ni CORS restringido.

> **Apps legacy:** si el primer escaneo trae 20 FAIL, el gate se apaga el primer día. Arranca con 2–3 reglas en FAIL (las que ya cumples o puedes cumplir en un sprint) y sube el umbral cada iteración.

---

## Parte 4: API scan con OpenAPI (activo, acotado)

Juice Shop publica el contrato de su API B2B en `swagger.yml` (lo sirve en http://localhost:3000/api-docs). `zap-api-scan.py` importa el contrato y **ataca cada endpoint** definido: inyecciones, errores 5xx, content-types inesperados. Es escaneo activo, pero acotado a la API, por eso tarda segundos.

### 4.1 Ejecutar

```bash
cp swagger.yml reports/
docker run --rm --network dast-net -v "$(pwd)/reports:/zap/wrk/:rw" \
  -t zaproxy/zap-stable zap-api-scan.py \
  -t swagger.yml -f openapi -O http://juice-shop:3000 \
  -r api.html -J api.json
echo "exit=$?"
```

| Flag | Propósito |
|---|---|
| `-t swagger.yml -f openapi` | contrato local (debe estar en `/zap/wrk`); también acepta URL, `soap` o `graphql` |
| `-O http://juice-shop:3000` | el contrato solo declara `servers: url: /b2b/v2`; `-O` fija el host real |
| `-S` (no usado) | *safe mode*: salta el activo y hace baseline sobre los endpoints |

### 4.2 Leer la salida

Salida observada (**26 s**):

```
Using override, new target: http://juice-shop:3000
Number of Imported URLs: 4
WARN-NEW: A Server Error response code was returned by the server [100000] x 9
WARN-NEW: Unexpected Content-Type was returned [100001] x 8
WARN-NEW: Cross-Domain Misconfiguration [10098] x 1
WARN-NEW: Application Error Disclosure [90022] x 1
FAIL-NEW: 0  FAIL-INPROG: 0  WARN-NEW: 4  WARN-INPROG: 0  INFO: 0  IGNORE: 0  PASS: 118
exit=2
```

| ID | Alerta | Qué pasó |
|---|---|---|
| 100000 | Server Error response code | `POST /b2b/v2/orders` devolvió 5xx ante payloads inesperados: la API no valida la entrada |
| 100001 | Unexpected Content-Type | respuestas HTML donde se esperaba JSON (páginas de error) |
| 90022 | Application Error Disclosure | un stack trace o mensaje interno llegó al cliente |
| 10098 | Cross-Domain Misconfiguration | el mismo CORS `*` de la Parte 2, ahora en la API |

> **Lo que este escaneo no cubrió:** `/b2b/v2/orders` exige un JWT (`bearerAuth` en el contrato) y un body válido. ZAP lo atacó sin token y con datos generados, así que la mayoría de respuestas fueron errores, no lógica real. Es exactamente la **capa activa** del pipeline corporativo vista en clase: para que aporte valor, el equipo debe entregar credenciales de prueba y un body correcto por cada POST/PUT/PATCH. Sin eso, el activo produce ruido (5xx) en lugar de hallazgos.

### 4.3 (Opcional) Autenticar el API scan

Obtén un JWT del usuario de prueba y pásalo como cabecera global con el *replacer* de ZAP:

```bash
TOKEN=$(curl -s -X POST http://localhost:3000/rest/user/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"admin@juice-sh.op","password":"admin123"}' | python3 -c "import sys,json;print(json.load(sys.stdin)['authentication']['token'])")

docker run --rm --network dast-net -v "$(pwd)/reports:/zap/wrk/:rw" \
  -t zaproxy/zap-stable zap-api-scan.py \
  -t swagger.yml -f openapi -O http://juice-shop:3000 -r api-auth.html \
  -z "-config replacer.full_list(0).description=auth -config replacer.full_list(0).enabled=true -config replacer.full_list(0).matchtype=REQ_HEADER -config replacer.full_list(0).matchstr=Authorization -config replacer.full_list(0).regex=false -config replacer.full_list(0).replacement=Bearer\ $TOKEN"
```

Resultado observado: ZAP envía `Authorization: Bearer <jwt>` en **todas** las peticiones (verificado con un servidor eco), pero el resumen sigue igual: `100000 x 9`, `100001 x 13`. Es decir, el token resolvió la autenticación y los 5xx **no** desaparecieron: el problema es el body generado por ZAP para `POST /orders`, no el acceso. Conclusión para la capa activa: credenciales **y** datos válidos, las dos cosas. (La contraseña `admin123` es la del usuario administrador de Juice Shop; es uno de sus retos.)

---

## Parte 5 (opcional, fuera de clase): Full scan

`zap-full-scan.py` hace spider sin límite, AJAX spider con `-j`, y escaneo activo sobre **todo** lo descubierto. Contra Juice Shop puede superar los 30 min y genera cientos de peticiones de ataque. Córrelo solo en tu máquina y nunca contra una app ajena.

```bash
mkdir -p reports-full
docker run --rm --network dast-net -v "$(pwd)/reports-full:/zap/wrk/:rw" \
  -t zaproxy/zap-stable zap-full-scan.py -t http://juice-shop:3000 -j -m 3 -T 60 \
  -r full.html -J full.json -s
```

| Flag | Propósito |
|---|---|
| `-j` | agrega el AJAX spider: imprescindible en una SPA Angular |
| `-m 3` | limita el spider a 3 min (por defecto sin límite) |
| `-T 60` | tiempo máximo total en minutos |

Busca en el reporte `Cross Site Scripting`, `SQL Injection` y `Path Traversal`. Como referencia, en `ejemplo_intro_dast.md` el full scan sobre una app de 3 rutas tardó 9 min y encontró el XSS reflejado con el payload exacto.

---

## Parte 6: Integrar DAST al pipeline de GitLab

Hasta aquí todo fue manual. Ahora el baseline corre en cada push, con gate, como un stage más del `.gitlab-ci.yml` del laboratorio anterior.

### 6.1 Cómo levantar la app dentro del job

GitLab CI permite declarar **`services`**: contenedores auxiliares que arrancan antes del job y se alcanzan por un alias de red. El runner espera a que el puerto expuesto (3000) acepte conexiones y luego ejecuta tu `script`. Así Juice Shop vive solo mientras dura el job: un ambiente efímero por pipeline.

### 6.2 Agregar el stage

Edita `.gitlab-ci.yml`:

```yaml
stages:
  - prepare
  - secret_scanning
  - sca
  - sast
  - dynamic_scanning   # nuevo (no lo llames "dast": ver nota abajo)
  - defectdojo    # si ya hiciste guia_defect_dojo.md
```

> **No llames al stage `dast`.** GitLab reserva ese nombre para su DAST nativo (licencia Ultimate): con `stage: dast` el pipeline queda en `yaml invalid` con `Insufficient permissions for dast_configuration keyword` y **no corre ningún job**. Cualquier otro nombre funciona.

Y agrega el job:

```yaml
# DAST pasivo con OWASP ZAP contra Juice Shop efímero
zap-baseline:
  image: zaproxy/zap-stable
  stage: dynamic_scanning
  needs: []                       # no depende de SAST/SCA: corre en paralelo
  services:
    - name: bkimminich/juice-shop:latest
      alias: juice-shop
  variables:
    FF_NETWORK_PER_BUILD: "true"  # red por pipeline: job y servicios se resuelven por alias
    TARGET: "http://juice-shop:3000"
  before_script:
    # ZAP corre como uid 1000 (zap) y solo escribe en /zap/wrk, que NO existe en la imagen
    - mkdir -p /zap/wrk
    - cp zap/rules.conf /zap/wrk/
    # el runner espera el puerto; Juice Shop todavía puede tardar unos segundos en responder HTTP
    - for i in $(seq 1 30); do curl -sf -o /dev/null "$TARGET" && break; sleep 2; done
    - curl -s -I "$TARGET" | head -5
  script:
    - zap-baseline.py -t "$TARGET" -c rules.conf -r zap-report.html -J zap-report.json -x zap-report.xml -s
  after_script:
    - cp /zap/wrk/zap-report.* "$CI_PROJECT_DIR/" || true
  artifacts:
    when: always                  # el reporte se guarda aunque el gate falle
    paths:
      - zap-report.html
      - zap-report.json
      - zap-report.xml
    expire_in: 1 week
  allow_failure: true             # primera iteración: ver resultados sin bloquear; quítalo en 6.4
  rules:
    - when: always
```

| Decisión | Por qué |
|---|---|
| `image: zaproxy/zap-stable` | trae `zap-baseline.py`, `curl` y `python3`; `ENTRYPOINT` vacío y `CMD bash`, así que GitLab ejecuta el `script` sin ajustes |
| `mkdir -p /zap/wrk` + `cp rules.conf` | los scripts leen `-c` y escriben `-r/-J/-x` **relativos a `/zap/wrk`**; el proyecto está en `/builds/...` |
| `after_script` copia a `$CI_PROJECT_DIR` | los artifacts solo se recogen desde el directorio del proyecto; `after_script` corre aunque `script` falle |
| `needs: []` | DAST no necesita los reportes de SAST; arranca con el pipeline y ahorra minutos |
| `-s` | salida corta en el log del job (sin PASS ni URLs de ejemplo) |

### 6.3 Commit y pipeline

```bash
git add .gitlab-ci.yml zap/rules.conf
git commit -m "ci: add DAST stage with OWASP ZAP baseline"
git push
```

En GitLab: **Build → Pipelines** → job `zap-baseline`. Verifica en el log:

1. `curl -s -I` muestra las cabeceras de Juice Shop (el servicio respondió)
2. El resumen `FAIL-NEW: 2 … WARN-NEW: …` igual al de la Parte 3.3
3. Estado del job: **warning** (naranja) por `allow_failure: true`
4. **Artifacts → Browse**: `zap-report.html`, `.json`, `.xml`

Tiempo esperado en runners SaaS: 7–10 min.

### 6.4 Activar el gate de verdad

Igual que en la Parte 4.4 de `guia_defect_dojo.md`: quita `allow_failure: true`. Con `rules.conf` en FAIL para CSP y CORS, el job queda **rojo** y el pipeline falla mientras esas dos alertas existan. El reporte se sigue subiendo gracias a `artifacts.when: always`.

> Para que el pipeline vuelva a verde sin arreglar Juice Shop (que es vulnerable a propósito), baja 10038 y 10098 a WARN en `zap/rules.conf`. Documenta la decisión en el commit: un gate que se afloja sin explicación es deuda técnica invisible.

### 6.5 (Opcional) Escanear la imagen construida desde tu rama

El job anterior escanea la imagen pública, no tu código. Para escanear lo que hay en la rama: agrega un stage `build` que construya el `Dockerfile` del repo con `docker:dind` y lo publique en el registry de GitLab, y usa esa imagen como servicio.

Con la imagen pública daba igual que `zap-baseline` corriera en paralelo (`needs: []`): no ejecutaba tu código. Con tu imagen **sí** lo ejecuta, incluidas sus dependencias. Si una librería trae código malicioso (supply chain), corre dentro del servicio durante el escaneo, y GitLab pasa al servicio **las mismas variables CI/CD que al job**. Por eso el orden cambia: primero los gates de SCA/SAST, después el build, al final DAST.

```yaml
stages:
  - prepare
  - secret_scanning
  - sca
  - sast
  - build               # nuevo: después de los análisis estáticos
  - dynamic_scanning
  - defectdojo

build-image:
  image: docker:27
  stage: build
  services: [docker:27-dind]
  needs: [trivy-scan, semgrep-scan]   # no se construye si SCA/SAST fallan
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"

zap-baseline:
  services:
    - name: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
      alias: juice-shop
  needs: [build-image]
  # ...resto igual
```

> `needs` solo puede apuntar a jobs de stages anteriores o del mismo stage; por eso `build-image` sale de `prepare` y pasa a un stage `build` después de `sast`. Con Juice Shop, `trivy-scan` y `semgrep-scan` fallan siempre (la app es vulnerable a propósito), así que este build nunca correría: en un proyecto real es justo lo que quieres.

El orden no basta: Trivy detecta CVEs **conocidos**, no un paquete malicioso publicado ayer. Lo que protege de verdad es aislar el job DAST:

| Medida | Cómo |
|---|---|
| Sin secretos en el job DAST | en *Settings → CI/CD → Variables*, pon el *Environment scope* de `DEFECTDOJO_*` en `defectdojo` y agrega `environment: defectdojo` solo a `upload-defectdojo`: el resto de jobs (y sus servicios) no reciben esas variables |
| Ambiente descartable | el servicio vive lo que dura el job, sin acceso a producción ni datos reales |
| Runner sin privilegios | no ejecutar la app en el mismo job que usa `docker:dind` privilegiado |

El `Dockerfile` de Juice Shop compila el frontend Angular: el build tarda 10–15 min en runners SaaS. Por eso el laboratorio usa la imagen pública.

---

## Parte 7: Enviar el reporte a DefectDojo

Requiere `guia_defect_dojo.md` (Partes 1–4): DefectDojo accesible por ngrok, variables `DEFECTDOJO_URL`, `DEFECTDOJO_TOKEN`, `DEFECTDOJO_PRODUCT_ID` y el script `scripts/upload_to_defectdojo.sh` en el repo.

### 7.1 Agregar ZAP al job `upload-defectdojo`

```yaml
upload-defectdojo:
  # ...
  needs:
    - gitleaks-scan
    - trufflehog-scan
    - trivy-scan
    - semgrep-scan
    - zap-baseline          # nuevo
  dependencies:
    - gitleaks-scan
    - trufflehog-scan
    - trivy-scan
    - semgrep-scan
    - zap-baseline          # nuevo
  script:
    - bash scripts/upload_to_defectdojo.sh "Gitleaks Scan"       "gitleaks-report.json"
    - bash scripts/upload_to_defectdojo.sh "Trufflehog Scan"     "trufflehog-report.json"
    - bash scripts/upload_to_defectdojo.sh "Trivy Scan"          "$REPORTS_DIR/trivy-report.json"
    - bash scripts/upload_to_defectdojo.sh "Semgrep JSON Report" "semgrep-report.json"
    - bash scripts/upload_to_defectdojo.sh "ZAP Scan"            "zap-report.xml"     # nuevo: el parser solo acepta XML
```

> El parser `ZAP Scan` de DefectDojo lee el **XML** (`<OWASPZAPReport>` → `site` → `alerts/alertitem`); rechaza el JSON. Por eso el job genera `-x zap-report.xml`.

### 7.2 Carga manual (sin pipeline)

```bash
curl -s -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
  -H "Authorization: Token $DEFECTDOJO_TOKEN" \
  -F "scan_type=ZAP Scan" \
  -F "file=@reports/baseline.xml" \
  -F "product_name=Juice Shop" \
  -F "engagement_name=DAST manual" \
  -F "auto_create_context=true" \
  -F "active=true" -F "verified=false"
```

### 7.3 Verificar en el dashboard

**Assets → Juice Shop → Findings**: filtra por **Test type = ZAP Scan**. Observa que:

- DefectDojo mapea `riskcode` de ZAP a su severidad (High / Medium / Low / Info)
- Cada finding trae el CWE y las URLs afectadas como *endpoints*
- La alerta 10098 (CORS) aparece **dos veces** si también subiste el API scan: una por el baseline y otra por la API; la deduplicación del producto las agrupa por `hash_code`

> **Reflexión:** SAST (Semgrep) reportó el SQLi en `routes/login.ts` por archivo y línea; DAST reporta por URL y parámetro. DefectDojo es el único lugar donde ambos conviven con la misma severidad y el mismo SLA.

---

## Parte 8: Limpiar

```bash
docker rm -f juice-shop
docker network rm dast-net
```

---

## ✅ Checklist de éxito

- [ ] Juice Shop responde en `http://localhost:3000` dentro de la red `dast-net`
- [ ] Baseline manual: 7 WARN, `PASS: 61`, exit 2; reportes `.html`, `.json`, `.xml` en `reports/`
- [ ] Puedes relacionar 10038, 10098 y 10063 con las cabeceras vistas en `curl -I`
- [ ] `zap/rules.conf` con al menos 2 FAIL, 2 WARN y 1 IGNORE; baseline con `-c` devuelve exit 1
- [ ] API scan sobre `swagger.yml` con `-O`: 4 WARN, incluidos 5xx (100000) y Application Error Disclosure (90022)
- [ ] Job `zap-baseline` en GitLab con servicio `juice-shop`, artifacts `zap-report.*` visibles
- [ ] Gate activo (sin `allow_failure`): job rojo por CSP + CORS, reporte igual subido
- [ ] (Parte 7) findings `ZAP Scan` visibles en DefectDojo junto a Semgrep/Trivy/Gitleaks
- [ ] Puedes explicar por qué el SQLi de `routes/login.ts` no aparece en el baseline

---

## 📘 Entregables

1. Capturas de:
   - Resumen del baseline manual (Parte 2.2) y del baseline con `rules.conf` mostrando `FAIL-NEW` y `exit=1`
   - Resumen del API scan (Parte 4.2)
   - Job `zap-baseline` en GitLab: log con el resumen y pestaña Artifacts
   - (Si hiciste la Parte 7) findings `ZAP Scan` en DefectDojo
2. Tu `zap/rules.conf` con un comentario por regla FAIL justificando por qué bloquea
3. Documento de reflexión:
   - ¿Qué hallazgos del baseline ya eran visibles con `curl -I` y cuáles no? ¿Qué aporta entonces el escáner?
   - ¿Por qué el API scan devolvió sobre todo errores 5xx? ¿Qué necesitaría el equipo para que la capa activa sea útil?
   - Semgrep encontró SQLi en `routes/login.ts` y ZAP no. ¿En qué condiciones DAST sí lo encontraría? ¿Qué escaneo y qué flags usarías?
   - Si fueras responsable del gate en un equipo con 800 repos legacy, ¿qué reglas pondrías en FAIL el primer mes?

---

## 🔧 Troubleshooting

| Error | Causa | Solución |
|---|---|---|
| `error from registry: denied` al descargar `ghcr.io/zaproxy/zaproxy` | red o proxy bloquea GHCR | `zaproxy/zap-stable` desde Docker Hub (misma imagen) |
| `Target Unreachable` / exit 3 | ZAP fuera de la red, o target `localhost` | `--network dast-net` y `-t http://juice-shop:3000` (con protocolo) |
| Reporte no aparece en `reports/` | sin volumen en `/zap/wrk` | `-v "$(pwd)/reports:/zap/wrk/:rw"` |
| `Permission denied` al escribir | ZAP corre como uid 1000 | `chmod 777 reports` o `-u $(id -u)` |
| `rules.conf` ignorado / `No such file` | está fuera de `/zap/wrk` o usa espacios en vez de tabuladores | `cp zap/rules.conf reports/`; `printf '%s\t%s\t%s\n'` |
| API scan: `Failed to import` | `swagger.yml` fuera de `/zap/wrk`, o falta `-O` | copiar a `reports/` y pasar `-O http://juice-shop:3000` |
| En GitLab: pipeline `yaml invalid`, ningún job corre, `Insufficient permissions for dast_configuration keyword` | el stage se llama `dast`: GitLab lo reserva para su DAST nativo (Ultimate) | renombrar el stage, p. ej. `dynamic_scanning` |
| En GitLab: `/zap/wrk: No such file or directory` | la imagen no trae el directorio | `mkdir -p /zap/wrk` en `before_script` |
| En GitLab: artifacts vacíos | el reporte quedó en `/zap/wrk` | `cp /zap/wrk/zap-report.* "$CI_PROJECT_DIR/"` en `after_script` |
| En GitLab: `curl` nunca responde | el servicio no levantó (`WARNING: Service … probably didn't start properly`) | revisar el log del servicio en el job; `FF_NETWORK_PER_BUILD: "true"` |
| Baseline tarda >10 min | escaneo pasivo de los chunks JS de la SPA | `-T 10` para acotar; en CI acepta 7–10 min |
| DefectDojo: `Unable to parse` en ZAP Scan | subiste el JSON | subir el `.xml` (`-x`) |

---

## ➡️ Siguiente paso

| Capa | Qué resuelve | Estado en tu pipeline |
|---|---|---|
| Secret scanning (Gitleaks, TruffleHog, pre-commit) | credenciales en código e historial | ✅ lab anterior |
| SCA (Trivy, SBOM) | CVEs en dependencias | ✅ lab anterior |
| SAST (Semgrep, SonarQube for IDE) | patrones inseguros en tu código | ✅ lab anterior |
| **DAST pasivo (ZAP baseline)** | configuración HTTP de la app desplegada, gate de PR | ✅ este lab |
| **DAST activo (ZAP API / full)** | vulnerabilidades explotables; requiere datos y credenciales de prueba | ⚠️ manual / nightly |
| **DAST por plantillas (Nuclei)** | controles conocidos, CVEs, fuzzing dirigido y plantillas propias en minutos; complementa a ZAP | `guia_nuclei_dast.md` |
| IaC scanning (Checkov, Trivy config, Terrascan) | Terraform y Kubernetes mal configurados | `laboratorio_IaC/` |
| Gestión centralizada (DefectDojo) | un backlog, dedup, SLA por severidad | `guia_defect_dojo.md` |

---

**Autor:** Wilson Julca Mejía  
Curso: *DevSecOps y Seguridad en CI/CD – UTEC*  
Universidad de Ingeniería y Tecnología (UTEC)
