# Guía introductoria: Nuclei como escáner DAST y su integración al pipeline

**Duración estimada:** 90–120 min  
**Nivel:** Intermedio  
**Contexto:** ZAP (`guia_dast_zap.md`) es un escáner DAST clásico: hace spider, observa tráfico y, en modo activo, ataca todo lo que encuentra. **Nuclei** (ProjectDiscovery) es otra forma de hacer DAST: no explora, **ejecuta plantillas YAML** que describen exactamente qué petición enviar y qué respuesta confirma un hallazgo. Es rápido, casi sin falsos positivos, fácil de extender con tus propias plantillas y encaja en un job de CI de pocos minutos. Aquí lo usarás contra Juice Shop, escribirás dos plantillas propias, y lo integrarás al `.gitlab-ci.yml` de los laboratorios anteriores junto a ZAP y DefectDojo.

> **Idea central:** ZAP responde "¿qué puede estar mal en esta app?"; Nuclei responde "¿esta app tiene exactamente este problema conocido?". El primero descubre, el segundo verifica. Un pipeline maduro usa ambos.

---

## 🎯 Objetivos de aprendizaje

- Entender el modelo de **plantillas** de Nuclei (id, info, http, matchers, extractors) y su repositorio comunitario
- Ejecutar Nuclei desde Docker contra una app en red: por plantilla, por **tags**, por **severidad** y por **perfil**
- Usar el modo **`-dast`** (fuzzing) para encontrar XSS reflejado y SQL injection en parámetros
- Escribir y validar **plantillas propias** para controles de tu organización
- Integrar Nuclei como job de **GitLab CI** con gate por severidad y envío a **DefectDojo**
- Decidir cuándo usar Nuclei, cuándo ZAP y cuándo los dos

---

## 🧩 Requisitos previos

✅ `guia_dast_zap.md` hecha: red `dast-net`, Juice Shop en Docker y, para la Parte 6, el pipeline con stage `dynamic_scanning`  
✅ Docker Desktop u OrbStack corriendo; `curl` y `python3`  
✅ Conectividad para descargar `projectdiscovery/nuclei` y las plantillas (~82 MB, 14 400 archivos)

---

## 📁 Qué se agrega al proyecto

```
juice-shop-devsecops/
├── .gitlab-ci.yml           # + job nuclei-scan en el stage dynamic_scanning (Parte 6)
├── nuclei/
│   ├── juice-ftp-listing.yaml   # plantilla propia (Parte 5)
│   ├── cors-wildcard.yaml       # plantilla propia (Parte 5)
│   └── urls.txt                 # endpoints con parámetros para -dast (Parte 4)
└── reports/                 # salidas locales .jsonl (no se commitea)
```

---

## Parte 1: Cómo funciona Nuclei

### 1.1 Plantillas, no spider

Un escaneo de Nuclei = **lista de targets × lista de plantillas**. Cada plantilla es un YAML con tres bloques:

```yaml
id: owasp-juice-shop-detect          # identificador único, sin espacios

info:                                 # metadatos: nombre, autor, severidad, tags, CWE
  name: OWASP Juice Shop
  author: pikpikcu
  severity: info
  tags: tech,owasp,discovery

http:                                 # protocolo: qué petición enviar
  - method: GET
    path:
      - "{{BaseURL}}"
    matchers-condition: and           # todos los matchers deben cumplirse
    matchers:
      - type: word                    # ¿la respuesta contiene esta cadena?
        part: body
        words:
          - "<title>OWASP Juice Shop</title>"
      - type: status                  # ¿el código HTTP es 200?
        status:
          - 200
```

(Es la plantilla real `http/technologies/owasp-juice-shop-detected.yaml`.) `{{BaseURL}}` se reemplaza por cada target. Los **matchers** deciden si hubo hallazgo; los **extractors** (opcionales) sacan datos de la respuesta para el reporte.

| Tipo de matcher | Qué compara | Ejemplo |
|---|---|---|
| `status` | código HTTP | `status: [200]` |
| `word` | cadenas en `body` / `header` | `words: ["X-Powered-By: Express"]` |
| `regex` | expresión regular | `regex: ['href="[^"]+\.bak"']` |
| `dsl` | expresión con funciones | `dsl: ["!regex('(?i)content-security-policy', header)"]` |
| `size` | longitud del cuerpo | `size: [0]` |

`negative: true` invierte cualquier matcher; `name:` etiqueta cuál disparó (lo verás como `matcher-name` en la salida).

### 1.2 El repositorio `nuclei-templates`

Al primer arranque Nuclei descarga el repo comunitario (v10.5.1 al 2026-10-26: **14 434 plantillas**). Estructura relevante:

| Directorio | Contenido | Plantillas |
|---|---|---|
| `http/cves/` | exploits/detección de CVE concretos | ~9 000 en `http/` |
| `http/misconfiguration/` | cabeceras faltantes, listados de directorio, paneles de debug | |
| `http/exposures/` | archivos sensibles: `.env`, backups, swagger, `/metrics` | |
| `http/technologies/` | fingerprinting (Wappalyzer, Express, Angular…) | |
| `http/default-logins/`, `http/exposed-panels/` | credenciales por defecto, paneles admin | |
| `dast/vulnerabilities/` | **fuzzing** de parámetros: xss, sqli, ssrf, lfi, cmdi, ssti, redirect… | 240 |
| `profiles/` | perfiles listos: `recommended`, `misconfigurations`, `pentest`, `kev`, `cves`… | |
| `network/`, `dns/`, `ssl/`, `javascript/`, `cloud/`, `code/` | otros protocolos y nubes | |

Cada plantilla lleva `tags` (`misconfig`, `exposure`, `tech`, `cve`, `dast`, `xss`…) y `severity` (`info`, `low`, `medium`, `high`, `critical`). Filtrarás por esos dos campos.

### 1.3 Nuclei vs ZAP

| | OWASP ZAP | Nuclei |
|---|---|---|
| Descubrimiento | spider + AJAX spider | **ninguno**: le das las URLs |
| Pasivo | sí, sobre todo el tráfico | plantillas tipo "falta cabecera" (una petición) |
| Activo | fuzzing genérico de todos los parámetros | `-dast` con plantillas de fuzzing + CVEs concretos |
| Falsos positivos | medios (riesgo + confianza) | muy bajos (matcher exacto) |
| Tiempo en Juice Shop | baseline 6,5 min, full > 30 min | tags acotados 5 min, `-dast` sobre 4 URLs **21 s** |
| Extensión | scripts Python/JS | **YAML** de 20 líneas |
| Exit code | 0/1/2/3 según WARN/FAIL | **siempre 0**: el gate lo haces tú |
| Ideal para | auditoría amplia de una app | verificación repetible de controles conocidos en CI |

---

## Parte 2: Preparar Juice Shop y Nuclei

### 2.1 Levantar la app y preparar volúmenes

Reusa la red `dast-net` de `guia_dast_zap.md` (o créala):

```bash
docker network create dast-net 2>/dev/null
docker run -d --name juice-shop --network dast-net -p 3000:3000 bkimminich/juice-shop:latest
until curl -sf -o /dev/null http://localhost:3000/; do sleep 2; done; echo "Juice Shop lista"
curl -s -I http://localhost:3000/ | head -8      # recuerda: CORS *, sin CSP, Feature-Policy deprecada
mkdir -p reports nuclei
```

Nuclei descarga las plantillas en `/root/nuclei-templates` dentro del contenedor. Guárdalas en un volumen para no bajar 82 MB en cada `docker run`:

```bash
docker run --rm \
  -v nuclei-templates:/root/nuclei-templates \
  -v nuclei-config:/root/.config/nuclei \
  projectdiscovery/nuclei:latest -ut
```

Salida esperada: tabla con `14434` plantillas o `[INF] No new updates found for nuclei templates` (primera vez: ~25 s). Nuclei observado: **v3.11.1**, plantillas **v10.5.1**.

Para no repetir los `-v`, define un alias de shell para el resto de la guía:

```bash
nuclei() {
  docker run --rm --network dast-net \
    -v nuclei-templates:/root/nuclei-templates \
    -v nuclei-config:/root/.config/nuclei \
    -v "$(pwd)/reports:/out" \
    -v "$(pwd)/nuclei:/custom" \
    projectdiscovery/nuclei:latest -duc -nc "$@"
}
```

| Flag | Propósito |
|---|---|
| `--network dast-net` | alcanzar `juice-shop` por nombre |
| `-v …/reports:/out` | donde escribir los `.jsonl` |
| `-v …/nuclei:/custom` | tus plantillas propias (Parte 5) |
| `-duc` | no comprobar actualizaciones en cada corrida |
| `-nc` | sin colores ANSI (logs limpios) |

### 2.2 Una plantilla: cabeceras de seguridad

```bash
nuclei -u http://juice-shop:3000 -t http/misconfiguration/http-missing-security-headers.yaml -silent
```

`-t` con ruta relativa busca primero en el directorio actual y luego dentro del directorio de plantillas: esta es una plantilla **oficial** del repo `nuclei-templates` (el log sin `-silent` dice `Executing 1 signed templates from projectdiscovery/nuclei-templates`), no una propia.

Salida observada (**3,6 s** incluido el arranque del contenedor):

```
[http-missing-security-headers:cross-origin-resource-policy] [http] [info] http://juice-shop:3000
[http-missing-security-headers:strict-transport-security] [http] [info] http://juice-shop:3000
[http-missing-security-headers:content-security-policy] [http] [info] http://juice-shop:3000
[http-missing-security-headers:permissions-policy] [http] [info] http://juice-shop:3000
[http-missing-security-headers:x-permitted-cross-domain-policies] [http] [info] http://juice-shop:3000
[http-missing-security-headers:referrer-policy] [http] [info] http://juice-shop:3000
[http-missing-security-headers:cross-origin-embedder-policy] [http] [info] http://juice-shop:3000
[http-missing-security-headers:cross-origin-opener-policy] [http] [info] http://juice-shop:3000
```

Formato: `[template-id:matcher-name] [protocolo] [severidad] URL`. Una sola petición `GET /` y diez matchers `dsl` del tipo `!regex('(?i)content-security-policy', header)`: cada cabecera ausente dispara el suyo. **Disparan 8 de 10**: `x-frame-options` y `x-content-type-options` no aparecen porque Juice Shop sí las envía (`helmet.frameguard()` y `noSniff()` en `server.ts`). Compara con el baseline de ZAP de `guia_dast_zap.md`: las mismas cabeceras (10038 CSP, 90004 COEP…), pero aquí en segundos y sin spider.

**Cómo leer cada línea.** `[http-missing-security-headers:content-security-policy] [http] [info] http://juice-shop:3000` = plantilla `http-missing-security-headers`, matcher `content-security-policy` (la cabecera que faltó), protocolo HTTP, severidad `info`, URL donde se comprobó. La severidad es la de la **plantilla**, no de cada cabecera: Nuclei solo afirma "falta la cabecera", no que sea explotable; por eso todas salen `info` y el gate del pipeline (`medium|high|critical`) las deja pasar. El impacto real lo pones tú:

| Matcher (cabecera ausente) | Qué controla | Riesgo si falta | Prioridad en Juice Shop | ZAP equiv. |
|---|---|---|---|---|
| `content-security-policy` | de qué orígenes puede cargar scripts, estilos, imágenes, frames | cualquier XSS ejecuta código con todo el poder de la página; sin CSP no hay segunda barrera | **Alta**: la app tiene XSS reales (search, feedback) | 10038 (Medium) |
| `cross-origin-opener-policy` | aísla la ventana de quien la abrió (`window.opener`) | ataques entre ventanas: XS-Leaks, tabnabbing | Media | 90004 |
| `cross-origin-resource-policy` | quién puede incrustar tus recursos (`<img>`, `<script>`, fetch no-cors) | otros sitios cargan tus respuestas y las leen por canal lateral (Spectre) | Media | 90004 |
| `cross-origin-embedder-policy` | exige que lo que tú incrustas declare permiso (CORP/CORS) | sin aislamiento completo; habilita `SharedArrayBuffer` solo si está | Baja: suele romper imágenes de terceros | 90004 |
| `referrer-policy` | cuánto de la URL va en `Referer` al navegar a otro sitio | fuga de URLs con ids o tokens en query hacia terceros | Baja: navegadores modernos ya aplican `strict-origin-when-cross-origin` | — |
| `permissions-policy` | qué APIs del navegador (cámara, micrófono, geolocalización) puede usar la página y sus iframes | un iframe o script inyectado accede a esas APIs | Baja | 10063 |
| `strict-transport-security` | obliga HTTPS en visitas futuras | downgrade a HTTP / SSL stripping | **No aplica aquí**: estás en `http://`; HSTS se pone en el terminador TLS (ingress, ALB) | 10035 (Low) |
| `x-permitted-cross-domain-policies` | lectura de datos por clientes Flash/Acrobat | prácticamente nulo hoy | Ruido: poner `none` y olvidarlo | — |

Las dos que **sí** envía Juice Shop y por eso no aparecen: `x-frame-options: SAMEORIGIN` (anti-clickjacking, ZAP 10020) y `x-content-type-options: nosniff` (evita que el navegador adivine el MIME, ZAP 10021). Compruébalo: `curl -sI http://localhost:3000/ | grep -iE 'x-frame|x-content'`.

**Cómo se corrige.** En Express, `app.use(helmet())` con sus valores por defecto agrega CSP, COOP, CORP, Referrer-Policy, HSTS y X-Permitted-Cross-Domain-Policies de una vez; COEP y Permissions-Policy hay que activarlos a mano. Juice Shop usa helmet pieza por pieza (`frameguard()`, `noSniff()`) y omite CSP a propósito porque sus retos necesitan scripts inline. Al corregir, se vuelve a correr la misma plantilla: el número de matchers que disparan es la métrica de avance (8 → 0), y el `curl-command` del JSONL es la prueba para el ticket.

> **Si no sale nada:** `-silent` también oculta los errores de conexión. Un host que no resuelve (por ejemplo un contenedor que no está en `dast-net`) termina con `Scan completed … No results found` y exit 0: **target muerto ≠ app segura**. Depura sin `-silent` y con `-v`, y confirma el target con `docker run --rm --network dast-net alpine wget -qS -O /dev/null http://juice-shop:3000`.

### 2.3 Salida JSONL

La consola sirve para mirar; el pipeline y DefectDojo necesitan **JSONL** (un JSON por línea):

```bash
nuclei -u http://juice-shop:3000 -t http/misconfiguration/http-missing-security-headers.yaml \
  -silent -jsonl -o /out/juice-headers.jsonl
wc -l reports/juice-headers.jsonl        # 8
python3 -c "
import json
for l in open('reports/juice-headers.jsonl'):
    r = json.loads(l)
    print(r['info']['severity'], r['template-id'], r.get('matcher-name',''), r['matched-at'])
"
```

Campos útiles: `template-id`, `info.severity`, `info.classification.cwe-id`, `matcher-name`, `matched-at`, `curl-command` (para reproducir a mano), `request`/`response` completos.

### 2.4 Qué cargaría `-tags` (sin ejecutarlo)

`-tags` carga todas las plantillas con esas etiquetas. Antes de lanzarlas contra Juice Shop mira cuántas son:

```bash
nuclei -tags misconfig,tech,exposure -silent -tl | wc -l                                   # 3240
nuclei -tags misconfig,tech,exposure -severity medium,high,critical -silent -tl | wc -l     # 1152
```

> `-tags`, `-severity` y `-t` se combinan con AND. `-tl` lista las plantillas que entrarían sin ejecutarlas: úsalo siempre antes de un escaneo nuevo. Más de tres mil plantillas a 150 req/s es demasiado para una app Node en un contenedor: la Parte 3 muestra qué pasa si no las frenas y cómo acotarlas con `-rl`/`-c`. El modo `-dast` (fuzzing de parámetros) se ve en la Parte 4.

---

## Parte 3: Juice Shop — escanear sin tumbar la app

### 3.1 Lo que NO hay que hacer

```bash
nuclei -u http://juice-shop:3000 -jsonl -o /out/juice-full.jsonl -stats     # ← todas las plantillas
docker ps -a --filter name=juice-shop                                      # en otra terminal, cada 30 s
```

Resultado observado: **11 212 plantillas, 20 762 peticiones programadas, 150 req/s por defecto**. A 1 min 44 s, con el 46 % hecho y 365 errores, Juice Shop murió (`Exited (133)`, el proceso Node se quedó sin memoria). Nuclei terminó con **un** hallazgo (`swagger-api`) porque la app ya no respondía.

Lecciones: (1) el repositorio completo es para reconocimiento de infraestructura, no para una app en QA; (2) el rate limit por defecto es agresivo; (3) un escáner "activo" puede ser un DoS accidental. Reinicia la app:

```bash
docker rm -f juice-shop
docker run -d --name juice-shop --network dast-net -p 3000:3000 bkimminich/juice-shop:latest
until curl -sf -o /dev/null http://localhost:3000/; do sleep 2; done
```

### 3.2 Escaneo acotado por tags y rate limit

```bash
nuclei -u http://juice-shop:3000 -tags misconfig,tech,exposure -rl 20 -c 10 \
  -silent -jsonl -o /out/juice-tags.jsonl
```

| Flag | Propósito |
|---|---|
| `-rl 20` | máximo 20 peticiones/s (default 150) |
| `-c 10` | máximo 10 plantillas en paralelo (default 25) |

Observado: **5 min**, 13 hallazgos, Juice Shop sigue viva:

| Severidad | Plantilla | Matcher / evidencia |
|---|---|---|
| medium | `prometheus-metrics` | `/metrics` expuesto: métricas internas sin autenticación |
| info | `swagger-api` | `/api-docs/swagger.yaml`: el contrato de la API es público |
| info | `owasp-juice-shop-detect` | fingerprint de la app |
| info | `http-missing-security-headers` ×8 | CSP, HSTS, Permissions-Policy, Referrer-Policy, COEP/COOP/CORP, X-Permitted-Cross-Domain-Policies |
| info | `tech-detect`, `fingerprinthub-web-fingerprints` | Google Fonts, etc. |

Compara con ZAP (`guia_dast_zap.md` Parte 2): ZAP vio las cabeceras y CORS `*`; Nuclei vio las cabeceras y además `/metrics` y `swagger.yaml`, pero **no** CORS (no hay plantilla genérica para eso en `http/misconfiguration/`; la escribirás en la Parte 5).

### 3.3 Perfiles

Los perfiles son conjuntos predefinidos en `profiles/`:

```bash
nuclei -u http://juice-shop:3000 -tp misconfigurations -rl 20 -silent
```

`misconfigurations.yml` equivale a `-t http/misconfiguration/ -t network/misconfig/ -t javascript/misconfiguration/`. Otros: `recommended` (severidad low+ sin plantillas ruidosas), `pentest` (excluye `dos`, `fuzz`, `osint`), `kev` (CVE explotados activamente), `cves`. Míralos con:

```bash
docker run --rm -v nuclei-templates:/root/nuclei-templates alpine cat /root/nuclei-templates/profiles/recommended.yml
```

---

## Parte 4: DAST sobre la API de Juice Shop

El SQL injection de `routes/login.ts` y `rest/products/search` lo detectó Semgrep (SAST) en el laboratorio anterior, y ZAP baseline no lo vio. Con Nuclei basta darle el endpoint con su parámetro.

### 4.1 Lista de URLs

Crea `nuclei/urls.txt` con endpoints que reciben entrada del usuario:

> **Los tres archivos de `nuclei/` y qué hace cada uno.** No se confunden: dos son *plantillas* (qué buscar) y uno es una *lista de objetivos* (dónde buscar).
>
> | Archivo | Tipo | Rol | Se usa con |
> |---|---|---|---|
> | `urls.txt` | lista de objetivos | endpoints GET con parámetros que Nuclei va a fuzzear | `-dast -l urls.txt` |
> | `juice-ftp-listing.yaml` | plantilla propia | detecta el *directory listing* de `/ftp` y **extrae** los `.bak/.md` expuestos (Parte 5.1) | `-t nuclei/` |
> | `cors-wildcard.yaml` | plantilla propia | detecta `Access-Control-Allow-Origin: *` (lo que ZAP marcó como 10098, Parte 5.2) | `-t nuclei/` |
>
> Para **agregar un endpoint GET** a escanear, editas `urls.txt`. Para **agregar un control nuevo** (otra cabecera, otro patrón), escribes un `.yaml` nuevo en `nuclei/`. Para una **API con POST/JWT**, ni uno ni otro: usas el OpenAPI (4.4).

```
http://juice-shop:3000/rest/products/search?q=apple
http://juice-shop:3000/rest/user/whoami
http://juice-shop:3000/api/Products/1
http://juice-shop:3000/rest/track-order/5267-f73dad3f3e8c5ed2
```

> En un proyecto real esta lista sale del contrato OpenAPI (`-l openapi.yaml -im openapi`), de un export de Burp (`-im burp`) o de los logs del proxy de ZAP. Nuclei no la descubre solo.

### 4.2 Ejecutar

```bash
cp nuclei/urls.txt reports/
nuclei -dast -l /out/urls.txt -rl 30 -c 5 -jsonl -o /out/juice-dast.jsonl -stats -si 30
```

Salida observada (**21 s**, 54 plantillas DAST, 194 peticiones):

```
[sqli-error-based] [http] [critical] http://juice-shop:3000/rest/products/search?q=apple'
```

En el JSONL: `"fuzzing_parameter":"q"`, severidad `critical`. La plantilla `dast/vulnerabilities/sqli/sqli-error-based.yaml` agregó una comilla y reconoció el mensaje de error de SQLite en la respuesta. Es la misma vulnerabilidad que Semgrep marcó por archivo y línea; ahora está confirmada **desde afuera**, con la petición exacta para reproducirla (`curl-command`).

### 4.3 Autenticación (opcional)

Para endpoints que exigen JWT, Nuclei acepta un archivo de secretos:

```yaml
# nuclei/secrets.yaml
static:
  - type: bearertoken
    domains:
      - juice-shop
    token: <jwt-de-admin@juice-sh.op>
```

```bash
cp nuclei/secrets.yaml reports/
nuclei -dast -l /out/urls.txt -secret-file /out/secrets.yaml -rl 30 -silent
```

El JWT se obtiene como en `guia_dast_zap.md` Parte 4.3 (`POST /rest/user/login`). Nunca commitees `secrets.yaml`: va en `.gitignore` y, en CI, en una variable enmascarada.

### 4.4 Escanear una API entera desde su OpenAPI/Swagger

`urls.txt` sirve para endpoints **GET** sueltos. Para una API con muchas rutas, o con **POST que exigen JWT y body**, no se listan a mano: se le da a Nuclei el contrato **OpenAPI** y él arma las peticiones. Juice Shop trae `swagger.yml` (la API B2B que ya usaste con ZAP en `guia_dast_zap.md` 4.2): un solo endpoint, `POST /b2b/v2/orders`, protegido con `bearerAuth`.

**Qué archivo se toca:** `swagger.yml`, no `urls.txt` ni las plantillas de `nuclei/`. Es un archivo de **entrada** (describe la API), no una plantilla (lógica de detección).

Dos ajustes antes de correrlo:

1. **Server absoluto.** El spec trae `url: /b2b/v2` (relativo). Con eso Nuclei carga `Targets: 0` y no envía nada. Dale el host:

   ```bash
   sed 's#url: /b2b/v2#url: http://juice-shop:3000/b2b/v2#' swagger.yml > reports/swagger-abs.yml
   ```

2. **Auth declarada.** El spec tiene `security: bearerAuth`. Si no pasas ese parámetro, Nuclei aborta con `[FTL] openapi: missing global auth parameter: Authorization`. Obtén el JWT y pásalo como variable:

   ```bash
   TOKEN=$(curl -s -X POST http://localhost:3000/rest/user/login \
     -H 'Content-Type: application/json' \
     -d '{"email":"admin@juice-sh.op","password":"admin123"}' \
     | python3 -c 'import sys,json;print(json.load(sys.stdin)["authentication"]["token"])')
   ```

Ahora el escaneo, **con rate bajo** (un POST que crea pedidos es caro; ver aviso):

```bash
nuclei -dast -l /out/swagger-abs.yml -im openapi \
  -var "Authorization=Bearer $TOKEN" \
  -rl 10 -c 2 -stats -si 15 -jsonl -o /out/b2b-openapi.jsonl
```

Salida observada (**32 s**, 54 plantillas DAST, 246 peticiones, **0 errores**, app viva todo el tiempo):

```
[INF] Templates loaded for current scan: 54
[INF] Targets loaded for current scan: 1
[INF] Scan completed in 31.5s. 0 matches found.
[INF] HTTP connections: 246 total, 2 new, 244 reused (99.2%)
```

**0 hallazgos aquí no es un error ni app segura:** significa que las plantillas DAST (SQLi error-based, SSTI, command injection, XSS…) fuzzearon el body de `POST /orders` y ninguna disparó. El SQLi de Juice Shop vive en el **GET** `/rest/products/search` (4.2), no en este endpoint. Comparar ambos es el punto: misma herramienta, un endpoint da `critical` y otro limpio.

| Flag | Propósito |
|---|---|
| `-im openapi` | interpreta el archivo como contrato OpenAPI 3.x (`-im swagger` para Swagger 2.0) |
| `-var "Authorization=Bearer …"` | rellena el `securityScheme` del spec; sin esto, `[FTL] missing global auth parameter` |
| `-var server=…` | base URL si el spec no la trae (alternativa al `sed`) |
| `-ro` | *required-only*: genera solo con los campos obligatorios (menos ruido) |
| `-rl 10 -c 2` | rate bajo: este endpoint escribe en BD; ver aviso |

> **Aviso (comprobado).** Fuzzear `POST /b2b/v2/orders` a rate por defecto (`-rl 150`) tumba Juice Shop (`Exited 133`): cada payload intenta crear un pedido y agota memoria. Con `-rl 10 -c 2` la app respondió `200` durante los 32 s. Para cualquier endpoint de escritura: rate bajo y vigila el contenedor en otra terminal (`watch docker ps`).

> **`-im yaml` (petición cruda).** Si no tienes OpenAPI pero sí una petición concreta (copiada de Burp o del navegador), Nuclei acepta un archivo *proxify* y fuzzea esa petición exacta:
> ```yaml
> # nuclei/b2b-request.yaml
> url: http://juice-shop:3000/b2b/v2/orders
> request:
>   raw: |
>     POST /b2b/v2/orders HTTP/1.1
>     Host: juice-shop:3000
>     Content-Type: application/json
>     Authorization: Bearer <JWT>
>
>     {"cid":"JS0815DE","orderLines":[{"productId":1,"quantity":2}],"orderLinesData":""}
> ```
> ```bash
> nuclei -dast -l /out/b2b-request.yaml -im yaml -rl 10 -c 2 -jsonl -o /out/b2b.jsonl
> ```

---

## Parte 5: Plantillas propias

Aquí está la ventaja decisiva de Nuclei para un equipo de plataforma: un control de tu organización ("ninguna API responde CORS `*`", "ningún servicio expone `/ftp`") se convierte en 25 líneas de YAML que corren en todos los repos.

### 5.1 Listado de `/ftp` con extractor

`nuclei/juice-ftp-listing.yaml`:

```yaml
id: juice-ftp-directory-listing

info:
  name: Juice Shop - /ftp directory listing expuesto
  author: utec-devsecops
  severity: medium
  description: El directorio /ftp lista archivos internos (backups, documentos legales, claves). Un atacante puede descargarlos.
  reference:
    - https://owasp.org/www-project-juice-shop/
  classification:
    cwe-id: CWE-548
  tags: juice-shop,exposure,misconfig,utec

http:
  - method: GET
    path:
      - "{{BaseURL}}/ftp"

    matchers-condition: and
    matchers:
      - type: status
        status:
          - 200
      - type: word
        part: body
        words:
          - "listing directory /ftp"
      - type: regex
        part: body
        regex:
          - 'href="[^"]+\.(md|pdf|bak|zip|key)"'

    extractors:
      - type: regex
        part: body
        group: 1
        regex:
          - 'href="([^"]+\.(?:md|pdf|bak|zip|key))"'
```

### 5.2 CORS abierto

`nuclei/cors-wildcard.yaml`:

```yaml
id: cors-allow-origin-wildcard

info:
  name: CORS - Access-Control-Allow-Origin abierto (*)
  author: utec-devsecops
  severity: medium
  description: La API responde Access-Control-Allow-Origin "*", por lo que cualquier origen puede leer sus respuestas desde el navegador.
  classification:
    cwe-id: CWE-942
  tags: cors,misconfig,utec

http:
  - method: GET
    path:
      - "{{BaseURL}}/"
      - "{{BaseURL}}/rest/products/search?q="
    headers:
      Origin: https://evil.example

    matchers-condition: and
    matchers:
      - type: status
        status:
          - 200
      - type: word
        part: header
        case-insensitive: true
        words:
          - "access-control-allow-origin: *"
```

> **Trampa verificada:** la primera versión usaba `type: dsl` con `regex('…\\s*\\*', header)` y fallaba con `error parsing regexp: invalid nested repetition operator: '**'`: el escape de `\*` se pierde al pasar por YAML y luego por el DSL. Para cadenas fijas usa `word` con `case-insensitive`; reserva `dsl`/`regex` para patrones de verdad. Y siempre depura con `-debug-resp`, que vuelca la respuesta real.

### 5.3 Validar y ejecutar

```bash
nuclei -validate -t /custom
#   [INF] All templates validated successfully

nuclei -t /custom -u http://juice-shop:3000 -silent
```

Salida observada (2 s):

```
[cors-allow-origin-wildcard] [http] [medium] http://juice-shop:3000/
[cors-allow-origin-wildcard] [http] [medium] http://juice-shop:3000/rest/products/search?q
[juice-ftp-directory-listing] [http] [medium] http://juice-shop:3000/ftp ["ftp/package.json.bak","ftp/acquisitions.md","ftp/announcement_encrypted.md","ftp/coupons_2013.md.bak","ftp/legal.md","ftp/package-lock.json.bak"]
```

El extractor devolvió la lista de archivos expuestos: `package.json.bak`, `coupons_2013.md.bak`… Ese detalle va directo al ticket.

> **Ejercicio:** escribe `nuclei/x-powered-by.yaml` que dispare (severidad `low`) cuando la respuesta traiga `X-Powered-By`. Contra Juice Shop **no debe disparar**: `server.ts` hace `app.disable('x-powered-by')`. Para verla disparar, levanta cualquier Express sin esa línea en `dast-net` (por ejemplo `demo-dast:1` de `ejemplo_intro_dast.md`, si lo hiciste) y compara. Una plantilla que nunca dispara no está validada.

---

## Parte 6: Integrar al pipeline de GitLab

### 6.1 El problema del exit code

Nuclei **siempre termina con 0**, haya o no hallazgos (verificado: 11 hallazgos → `exit=0`). El gate se construye leyendo el JSONL. La imagen oficial es Alpine con `sh`, `grep` y `wget`, sin `jq` ni `python`, así que el filtro va con `grep`:

```bash
N=$(grep -cE '"severity":"(medium|high|critical)"' nuclei-report.jsonl || true)
echo "Hallazgos medium+: $N"
[ "$N" -eq 0 ] || exit 1
```

(Probado dentro de la imagen sobre `juice-tags.jsonl`: `medium+ findings: 1`, `exit=1`.)

### 6.2 El job

Agrega al `.gitlab-ci.yml` del laboratorio anterior, en el mismo stage `dynamic_scanning` que `zap-baseline`:

```yaml
# DAST con Nuclei: plantillas comunitarias acotadas + plantillas propias + fuzzing de la API
nuclei-scan:
  image:
    name: projectdiscovery/nuclei:latest
    entrypoint: [""]              # la imagen tiene ENTRYPOINT=nuclei; GitLab necesita un shell
  stage: dynamic_scanning
  needs: []
  services:
    - name: bkimminich/juice-shop:latest
      alias: juice-shop
  variables:
    FF_NETWORK_PER_BUILD: "true"
    TARGET: "http://juice-shop:3000"
    NUCLEI_TEMPLATES: "$CI_PROJECT_DIR/.nuclei-templates"
  cache:
    key: nuclei-templates
    paths:
      - .nuclei-templates/       # 82 MB; sin cache se descargan en cada job (~25 s)
  before_script:
    - nuclei -ut -ud "$NUCLEI_TEMPLATES" -silent
    - for i in $(seq 1 30); do wget -q -O /dev/null "$TARGET" && break; sleep 2; done
    - sed "s#http://juice-shop:3000#$TARGET#" nuclei/urls.txt > urls.txt
  script:
    # 1) comunitarias acotadas por tags, sin tumbar la app
    - nuclei -ud "$NUCLEI_TEMPLATES" -u "$TARGET" -tags misconfig,exposure -rl 20 -c 10 -duc -nc -silent -jsonl -o nuclei-report.jsonl
    # 2) plantillas propias del repo (comando aparte: ver tabla)
    - nuclei -ud "$NUCLEI_TEMPLATES" -u "$TARGET" -t nuclei/ -rl 20 -duc -nc -silent -jsonl >> nuclei-report.jsonl
    # 3) fuzzing solo sobre los endpoints con parámetros
    - nuclei -ud "$NUCLEI_TEMPLATES" -dast -l urls.txt -rl 30 -c 5 -duc -nc -silent -jsonl >> nuclei-report.jsonl
    # 4) gate por severidad (nuclei siempre devuelve 0)
    - N=$(grep -cE '"severity":"(medium|high|critical)"' nuclei-report.jsonl || true)
    - echo "Hallazgos medium+: $N"
    - '[ "$N" -eq 0 ] || exit 1'
  artifacts:
    when: always
    paths:
      - nuclei-report.jsonl
    expire_in: 1 week
  # sin allow_failure: el job sale rojo (failed). El artifact se sube igual (artifacts.when: always)
  # y upload-defectdojo corre igual si tiene rules.when: always. Agrega allow_failure: true si prefieres warning.
  rules:
    - when: always
```

| Decisión | Por qué |
|---|---|
| `entrypoint: [""]` | con `ENTRYPOINT=nuclei`, GitLab no podría ejecutar el `script` |
| `-ud $CI_PROJECT_DIR/.nuclei-templates` + `cache` | el cache de GitLab solo guarda rutas dentro del proyecto; `-ud` mueve las plantillas ahí |
| `cache: when: always` | por defecto el cache se guarda solo si el job termina en verde; con el gate en rojo nunca se guardaría |
| `-tags misconfig,exposure` y no el repo completo | 11 212 plantillas a 150 req/s mataron Juice Shop en local (Parte 3.1) |
| `-t nuclei/` en un comando aparte | tus plantillas propias viajan en el repo y se versionan con el código. **No** lo combines con `-tags` en la misma línea: `-t` restringe la carga a esa carpeta y las comunitarias no corren (verificado con `-tl`: 2 plantillas en vez de 2 033) |
| `sed` sobre `urls.txt` | la lista local apunta a `juice-shop:3000`; en CI el alias coincide, pero así puedes cambiar `TARGET` |
| `>>` en el segundo y tercer escaneo | un solo JSONL para el gate y para DefectDojo |
| `-silent -jsonl` sin `-o` | escribe a stdout, que se anexa al archivo |

Tiempo esperado en runners SaaS: 6–8 min (dominado por el escaneo por tags). Commit:

```bash
git add .gitlab-ci.yml nuclei/
git commit -m "ci: add Nuclei DAST job (community templates, custom templates, API fuzzing)"
git push
```

### 6.3 Qué debe salir en el log

1. `[INF] No new updates found` o la tabla de descarga de plantillas
2. Sin salida del escaneo (`-silent` + `-o`), luego `Hallazgos medium+: 4` (prometheus-metrics, cors ×2, ftp-listing) + 1 `critical` (sqli) → **exit 1** → job en rojo (failed); el artifact se sube igual
3. Artifact `nuclei-report.jsonl` con ~20 líneas

### 6.4 Ajustar el gate

El pipeline queda rojo. Si el equipo aún no acepta el gate, agrega `allow_failure: true` (warning). Para que vuelva a verde sin arreglar Juice Shop, sube el umbral del gate a `high|critical` (quedará solo el SQLi) y documenta la decisión en el commit.

---

## Parte 7: DefectDojo

El parser **`Nuclei Scan`** de DefectDojo acepta el JSONL tal cual (una línea por hallazgo) o un array JSON. Lee `template-id`, `info.name`, `info.severity`, `matched-at`, `curl-command`, `extracted-results` y `classification.cwe-id`, y deduplica por plantilla + matcher + host.

> **Variables protegidas.** `DEFECTDOJO_TOKEN` se creó como *Protected* en `guia_defect_dojo.md`; GitLab solo la entrega a ramas **protegidas**. Si trabajas en `feature/…`, el job `upload-defectdojo` falla en la primera subida con `Variable requerida no definida: DEFECTDOJO_TOKEN` y nada llega a DefectDojo. Protege la rama (Settings → Repository → Protected branches, wildcard `feature/*`) o desmarca *Protected* en la variable, y haz *Retry* del job. Para que el job se salte con motivo claro en vez de fallar: `- if: $DEFECTDOJO_URL == null || $DEFECTDOJO_TOKEN == null` → `when: never`.

Agrega al job `upload-defectdojo` (`guia_defect_dojo.md` Parte 4.3):

```yaml
  needs:
    # ...
    - nuclei-scan
  dependencies:
    # ...
    - nuclei-scan
  script:
    # ...
    - bash scripts/upload_to_defectdojo.sh "ZAP Scan"    "zap-report.xml"
    - bash scripts/upload_to_defectdojo.sh "Nuclei Scan" "nuclei-report.jsonl"
```

Carga manual:

```bash
curl -s -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
  -H "Authorization: Token $DEFECTDOJO_TOKEN" \
  -F "scan_type=Nuclei Scan" \
  -F "file=@reports/juice-dast.jsonl" \
  -F "product_name=Juice Shop" \
  -F "engagement_name=DAST nuclei" \
  -F "auto_create_context=true" -F "active=true" -F "verified=false"
```

En **Findings**, filtra por *Test type*: verás el SQLi de Nuclei (critical, con el `curl-command` en *Steps to reproduce*) al lado del SQLi de Semgrep (archivo y línea). Mismo bug, dos evidencias, un solo ticket.

---

## Parte 8: Limpiar

```bash
docker rm -f juice-shop
docker network rm dast-net
# conservar: volúmenes nuclei-templates y nuclei-config (ahorran la descarga)
```

---

## ✅ Checklist de éxito

- [ ] Plantillas descargadas en el volumen `nuclei-templates` (~14 400)
- [ ] `http-missing-security-headers` contra Juice Shop: 8 de 10 matchers en ~4 s, y sabes por qué faltan dos
- [ ] `juice-headers.jsonl` con 8 líneas y puedes leer `template-id`, `matcher-name`, `matched-at`
- [ ] `-tl` muestra 3240 plantillas para `-tags misconfig,tech,exposure` (1152 de medium o más)
- [ ] Puedes explicar por qué el escaneo completo tumbó Juice Shop y qué flags lo evitan
- [ ] Escaneo acotado (`-rl 20`) contra Juice Shop: `prometheus-metrics`, `swagger-api`, cabeceras
- [ ] `-dast -l urls.txt`: `sqli-error-based` critical en `q`
- [ ] `-dast -im openapi` sobre `swagger-abs.yml` con `-var Authorization=Bearer …`: carga 1 target, 0 hallazgos, app viva (sabes por qué 0)
- [ ] Dos plantillas propias validadas y disparando (`cors-allow-origin-wildcard`, `juice-ftp-directory-listing` con extractor)
- [ ] Job `nuclei-scan` en GitLab con gate por `grep`, artifact `nuclei-report.jsonl`
- [ ] (Parte 7) findings `Nuclei Scan` en DefectDojo

---

## 📘 Entregables

1. Capturas: salida de 2.2, JSONL de 2.4, resumen de 3.2, hallazgo de 4.2, ejecución de 5.3, job `nuclei-scan` en GitLab (log + artifact)
2. Tus dos plantillas propias más la del ejercicio (`x-powered-by.yaml`), validadas con `-validate`
3. Reflexión:
   - Nuclei encontró `/metrics` y `swagger.yaml`; ZAP encontró CORS `*`. ¿Por qué cada uno vio lo que el otro no?
   - El SQLi apareció en Semgrep (SAST) y en Nuclei (DAST), pero no en ZAP baseline. ¿Qué dice eso sobre dónde poner cada herramienta en el pipeline?
   - ¿Qué tres controles de tu organización escribirías como plantillas Nuclei y en qué stage los correrías?
   - Nuclei devuelve siempre exit 0. ¿Qué riesgo tiene un gate basado en `grep` del reporte y cómo lo harías más robusto?

---

## 🔧 Troubleshooting

| Error | Causa | Solución |
|---|---|---|
| `nuclei: command not found` en GitLab | la imagen tiene `ENTRYPOINT=nuclei` y GitLab intenta correr `sh` | `image: { name: …, entrypoint: [""] }` |
| Descarga plantillas en cada corrida | sin volumen (local) o sin cache (CI) | `-v nuclei-templates:/root/nuclei-templates` / `cache` + `-ud` |
| `No templates provided for scan` | tag o ruta mal escrita | `-tl` para listar lo que cargaría; rutas relativas a `/root/nuclei-templates` o absolutas |
| La app se cae o `errors` sube en `-stats` | demasiadas peticiones (default 150 req/s, 25 plantillas en paralelo) | `-rl 20 -c 10`; acotar con `-tags`/`-tp`; nunca el repo completo contra una app |
| `-dast` no reporta nada | la URL no tiene parámetros, o falta la flag | targets con `?param=valor` (o `-l urls.txt`), `-dast` explícito |
| `error parsing regexp` en plantilla propia | escapes perdidos entre YAML y DSL | `word` + `case-insensitive` para cadenas fijas; `-debug-resp` para ver la respuesta |
| `The given path (/custom) is outside the default template directory` | aviso, no error | ignorar; solo afecta a workflows con rutas relativas |
| El gate no falla aunque hay hallazgos | `exit 0` de Nuclei | el gate es el `grep` del JSONL (6.1); revisa que el archivo no esté vacío (`-s`) |
| `Variable requerida no definida: DEFECTDOJO_TOKEN` en `upload-defectdojo` | variable *Protected* en rama no protegida | proteger la rama (`feature/*`) o desmarcar *Protected*; *Retry* del job |
| DefectDojo importa 0 findings | archivo con espacios al inicio o formato texto | subir el `.jsonl` de `-jsonl`, no la salida de consola |

---

## ➡️ Siguiente paso

| Capa | Herramienta | Qué aporta | Estado |
|---|---|---|---|
| DAST exploratorio | ZAP baseline / full | spider + reglas pasivas y activas genéricas | ✅ `guia_dast_zap.md` |
| **DAST por plantillas** | **Nuclei** | controles conocidos, CVEs, fuzzing dirigido, plantillas propias, minutos en CI | ✅ este lab |
| Gestión centralizada | DefectDojo | ZAP + Nuclei + SAST + SCA en un backlog | `guia_defect_dojo.md` |
| IaC | Checkov / Trivy config / Terrascan | la infraestructura que despliega la app | `laboratorio_IaC/` |

---

**Autor:** Wilson Julca Mejía  
Curso: *DevSecOps y Seguridad en CI/CD – UTEC*  
Universidad de Ingeniería y Tecnología (UTEC)
