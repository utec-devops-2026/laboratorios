# Ejemplo introductorio: DAST con OWASP ZAP en 20 minutos

**Duración estimada:** 20–30 min (más ~10 min si corres el escaneo activo)  
**Nivel:** Introductorio  
**Contexto:** Antes de integrar DAST a un pipeline real, conviene ver qué hace un escáner dinámico sobre una app diminuta cuyo código entiendes por completo. Levantarás una app Express de 20 líneas con tres fallas típicas (sin cabeceras de seguridad, cookie sin flags, XSS reflejado), la escanearás con OWASP ZAP, corregirás las fallas y volverás a escanear para comprobar que desaparecen.

> **Idea central:** DAST no lee tu código. Habla HTTP con la app en ejecución, como lo haría un atacante, y juzga lo que la app responde. Por eso detecta errores de configuración (cabeceras, cookies, CORS) que SAST/SCA no ven, y por eso necesita la app levantada.

---

## 🎯 Objetivos de aprendizaje

- Distinguir **escaneo pasivo** (observa respuestas) de **escaneo activo** (envía payloads)
- Leer la salida de `zap-baseline.py` y relacionar cada alerta con una cabecera HTTP
- Convertir un escaneo en un **gate** con `rules.conf` y el exit code
- Corregir las fallas con `helmet` y verificar la corrección con un segundo escaneo

---

## 🧩 Requisitos previos

✅ Docker Desktop (u OrbStack) corriendo  
✅ `curl`  
✅ Conectividad a Internet para descargar imágenes (`zaproxy/zap-stable` pesa ~2 GB)

> No necesitas Node.js en tu máquina: la app se construye dentro de Docker.

---

## 📁 Estructura

```
demo-dast/
├── app.js          # app Express vulnerable (v1) → corregida (v2)
├── package.json
├── Dockerfile
└── reports/        # salidas de ZAP (html, json, conf)
```

---

## Parte 1: La app vulnerable

### 1.1 Crear los archivos

```bash
mkdir -p demo-dast/reports && cd demo-dast
```

**`app.js`**

```js
const express = require('express')
const app = express()

app.get('/', (req, res) => {
  res.send('<h1>Demo DAST</h1><a href="/hello?name=UTEC">Saludar</a> · <a href="/login">Login</a>')
})

// Refleja el parámetro sin sanitizar -> XSS reflejado
app.get('/hello', (req, res) => {
  res.send(`<h1>Hola ${req.query.name}</h1>`)
})

app.get('/login', (req, res) => {
  res.cookie('session', 'abc123')          // sin HttpOnly, sin Secure, sin SameSite
  res.send('<form method="post"><input name="user"><input name="pass" type="password"><button>Entrar</button></form>')
})

app.listen(3000, () => console.log('demo-app en http://localhost:3000'))
```

**`package.json`**

```json
{
  "name": "demo-dast",
  "version": "1.0.0",
  "private": true,
  "main": "app.js",
  "scripts": { "start": "node app.js" },
  "dependencies": { "express": "^4.21.2" }
}
```

**`Dockerfile`**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json ./
RUN npm install --omit=dev --no-audit --no-fund
COPY app.js ./
EXPOSE 3000
CMD ["node", "app.js"]
```

### 1.2 Construir y levantar en una red Docker

ZAP correrá en otro contenedor, así que la app y el escáner deben compartir una red. `localhost` dentro del contenedor de ZAP es el propio contenedor, no tu máquina.

```bash
docker network create dast-net
docker build -t demo-dast:1 .
docker run -d --name demo-app --network dast-net -p 3000:3000 demo-dast:1
```

### 1.3 Mirar lo que ve un atacante

```bash
curl -s -I http://localhost:3000/
curl -s -i http://localhost:3000/login | grep -i set-cookie
curl -s 'http://localhost:3000/hello?name=<b>UTEC</b>'
```

Salida esperada:

```
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
...
Set-Cookie: session=abc123; Path=/
<h1>Hola <b>UTEC</b></h1>
```

Tres observaciones, que son exactamente lo que ZAP va a reportar:

| Qué ves | Problema |
|---|---|
| Solo `X-Powered-By` y `Content-Type` | Faltan **todas** las cabeceras de seguridad (CSP, X-Frame-Options, X-Content-Type-Options, HSTS, Referrer-Policy, Permissions-Policy) y además se revela el framework |
| `Set-Cookie: session=abc123; Path=/` | Cookie sin `HttpOnly`, `Secure` ni `SameSite` |
| `<b>UTEC</b>` se renderiza como HTML | El parámetro `name` se refleja sin codificar → XSS reflejado |

---

## Parte 2: Escaneo pasivo (baseline)

### 2.1 Ejecutar `zap-baseline.py`

```bash
docker run --rm --network dast-net \
  -v "$(pwd)/reports:/zap/wrk/:rw" \
  -t zaproxy/zap-stable zap-baseline.py \
  -t http://demo-app:3000 \
  -r baseline.html -J baseline.json
```

| Flag | Propósito |
|---|---|
| `--network dast-net` | ZAP alcanza la app por su nombre de contenedor (`demo-app`) |
| `-v $(pwd)/reports:/zap/wrk/:rw` | Los scripts de ZAP leen y escriben archivos **solo** en `/zap/wrk`; sin el volumen el reporte se pierde |
| `-t http://demo-app:3000` | Target **con protocolo** |
| `-r` / `-J` | Reporte HTML (para leer) y JSON (para procesar o importar) |

> La imagen `zaproxy/zap-stable` (Docker Hub) y `ghcr.io/zaproxy/zaproxy:stable` (GitHub) son la misma. Si `ghcr.io` responde `denied` en tu red, usa la de Docker Hub.

El baseline hace tres cosas: **spider** durante 1 minuto (descubre `/`, `/hello?name=UTEC`, `/login`, y prueba `/robots.txt` y `/sitemap.xml`), **escaneo pasivo** de cada respuesta, y **reporte**. No envía ningún ataque. Tarda ~1 minuto.

### 2.2 Leer la salida

Salida observada (ZAP 2.17.0, resumida):

```
WARN-NEW: Cookie No HttpOnly Flag [10010] x 1
WARN-NEW: Missing Anti-clickjacking Header [10020] x 3
WARN-NEW: X-Content-Type-Options Header Missing [10021] x 3
WARN-NEW: Server Leaks Information via "X-Powered-By" HTTP Response Header Field(s) [10037] x 5
WARN-NEW: Content Security Policy (CSP) Header Not Set [10038] x 3
WARN-NEW: Storable and Cacheable Content [10049] x 5
WARN-NEW: Cookie without SameSite Attribute [10054] x 1
WARN-NEW: CSP: Failure to Define Directive with No Fallback [10055] x 3
WARN-NEW: Permissions Policy Header Not Set [10063] x 5
WARN-NEW: Authentication Request Identified [10111] x 1
WARN-NEW: Cross-Origin-Embedder-Policy Header Missing or Invalid [90004] x 9
FAIL-NEW: 0  FAIL-INPROG: 0  WARN-NEW: 11  WARN-INPROG: 0  INFO: 0  IGNORE: 0  PASS: 57
```

Cada línea es una **regla pasiva** (ID entre corchetes) que falló en N URLs. Las que importan para esta app:

| ID | Alerta | Cabecera / causa | Riesgo |
|---|---|---|---|
| 10020 | Missing Anti-clickjacking Header | falta `X-Frame-Options` (o `frame-ancestors` en CSP) | clickjacking |
| 10021 | X-Content-Type-Options Header Missing | falta `X-Content-Type-Options: nosniff` | MIME sniffing |
| 10037 | Server Leaks Information via X-Powered-By | `X-Powered-By: Express` | revela stack |
| 10038 | CSP Header Not Set | falta `Content-Security-Policy` | XSS sin mitigación en navegador |
| 10054 | Cookie without SameSite | `Set-Cookie` sin `SameSite` | CSRF |
| 10063 | Permissions Policy Header Not Set | falta `Permissions-Policy` | abuso de APIs del navegador |
| 90004 | COEP Header Missing | falta `Cross-Origin-Embedder-Policy` | aislamiento de origen |

Fíjate en lo que **no** aparece: el XSS reflejado en `/hello`. El escaneo pasivo solo mira respuestas normales; para encontrar el XSS hay que enviar un payload (Parte 4).

### 2.3 Entender el exit code

```bash
echo $?    # 2
```

| Exit code | Significado |
|---|---|
| 0 | Sin FAIL ni WARN |
| 1 | Al menos un FAIL |
| 2 | Al menos un WARN, ningún FAIL |
| 3 | Error del escáner (target inalcanzable, timeout) |

Por defecto todas las alertas son WARN → exit 2. Un pipeline que solo mira "distinto de 0" fallaría siempre; el gate real se define en la Parte 3.

### 2.4 Abrir el reporte

```bash
open reports/baseline.html        # macOS; en Linux: xdg-open
```

Cada alerta trae descripción, CWE, URLs afectadas, evidencia y **solución** sugerida. Para procesar el JSON:

```bash
python3 -c "
import json
d = json.load(open('reports/baseline.json'))
for site in d['site']:
    for a in site['alerts']:
        print(a['pluginid'], a['riskdesc'], '|', a['alert'], '| CWE', a['cweid'], '| x', len(a['instances']))
"
```

---

## Parte 3: Convertir el escaneo en un gate

### 3.1 Generar la configuración por defecto

```bash
docker run --rm --network dast-net -v "$(pwd)/reports:/zap/wrk/:rw" \
  -t zaproxy/zap-stable zap-baseline.py -t http://demo-app:3000 -g gen.conf -s
head reports/gen.conf
```

`gen.conf` lista todas las reglas (~70) con la acción `WARN`, separadas por **tabulador**:

```
10020	WARN	(Anti-clickjacking Header)
10021	WARN	(X-Content-Type-Options Header Missing)
10038	WARN	(Content Security Policy (CSP) Header Not Set)
```

### 3.2 Definir qué bloquea

Crea `reports/rules.conf` con las reglas que **sí** deben detener el pipeline (FAIL), las que solo informan (WARN) y las que ignoras (IGNORE):

```bash
printf '10020\tFAIL\t(Anti-clickjacking Header)\n10021\tFAIL\t(X-Content-Type-Options Header Missing)\n10038\tFAIL\t(Content Security Policy (CSP) Header Not Set)\n10037\tFAIL\t(Server Leaks Information via X-Powered-By)\n10054\tWARN\t(Cookie without SameSite Attribute)\n10049\tIGNORE\t(Storable and Cacheable Content)\n' > reports/rules.conf
```

> Usa `printf` con `\t` o un editor que inserte tabuladores reales. Con espacios ZAP no reconoce la línea.

### 3.3 Volver a escanear con el gate

```bash
docker run --rm --network dast-net -v "$(pwd)/reports:/zap/wrk/:rw" \
  -t zaproxy/zap-stable zap-baseline.py -t http://demo-app:3000 -c rules.conf -s
echo "exit=$?"
```

Salida observada:

```
IGNORE: Storable and Cacheable Content [10049] x 5
WARN-NEW: Cookie without SameSite Attribute [10054] x 1
...
FAIL-NEW: Missing Anti-clickjacking Header [10020] x 3
FAIL-NEW: X-Content-Type-Options Header Missing [10021] x 3
FAIL-NEW: Server Leaks Information via "X-Powered-By" HTTP Response Header Field(s) [10037] x 5
FAIL-NEW: Content Security Policy (CSP) Header Not Set [10038] x 3
FAIL-NEW: 4  FAIL-INPROG: 0  WARN-NEW: 6  WARN-INPROG: 0  INFO: 0  IGNORE: 1  PASS: 57
exit=1
```

Ahora el exit code es **1**: en un pipeline, este job falla y no hay merge. Las reglas no listadas en `rules.conf` siguen como WARN (con `-i` pasan a INFO).

---

## Parte 4: Escaneo activo (encontrar el XSS)

El escaneo activo envía payloads reales a cada parámetro descubierto. En esta app son 3 rutas, pero igual tarda **~9 minutos** porque ZAP prueba cientos de variantes por parámetro. Nunca lo corras contra producción ni contra una app que no te pertenece.

```bash
mkdir -p reports-full
docker run --rm --network dast-net -v "$(pwd)/reports-full:/zap/wrk/:rw" \
  -t zaproxy/zap-stable zap-full-scan.py -t http://demo-app:3000 -r full.html -J full.json -s
```

Salida observada (además de las 11 alertas pasivas; el escaneo activo no es determinista, el número exacto de alertas puede variar entre corridas):

```
WARN-NEW: Cross Site Scripting (Reflected) [40012] x 1
WARN-NEW: Cross Site Scripting (DOM Based) [40026] x 1
WARN-NEW: Source Code Disclosure - File Inclusion [43] x 1
WARN-NEW: Cookie Slack Detector [90027] x 2
FAIL-NEW: 0  FAIL-INPROG: 0  WARN-NEW: 13  ...  PASS: 130
```

En el JSON la alerta 40012 trae el ataque exacto y la evidencia:

```
40012 Cross Site Scripting (Reflected) | High (Medium) | CWE 79
  http://demo-app:3000/hello?name=%3C%2Fh1%3E%3CscrIpt%3Ealert%281%29%3B%3C%2FscRipt%3E%3Ch1%3E
  param: name | attack: </h1><scrIpt>alert(1);</scRipt><h1> | evidence: </h1><scrIpt>alert(1);</scRipt><h1>
```

ZAP envió `</h1><scrIpt>alert(1);</scRipt><h1>` en `name`, vio que la respuesta lo devolvía intacto y lo marcó como XSS reflejado con riesgo **High**. Nota `PASS: 130` frente a `57` del baseline: el full scan ejecuta reglas activas (SQLi, path traversal, inyección de comandos…) además de las pasivas.

> **Pasivo vs activo en una línea:** el baseline te dice *cómo está configurada* la app; el full scan te dice *qué se puede explotar*. El primero es seguro y rápido (gate de PR); el segundo altera estado y tarda (nightly en QA).

---

## Parte 5: Corregir y verificar

### 5.1 La app corregida (v2)

Reemplaza `app.js` y agrega `helmet` a `package.json`:

**`package.json`** (solo cambia `dependencies`)

```json
"dependencies": { "express": "^4.21.2", "helmet": "^8.1.0" }
```

**`app.js`**

```js
const express = require('express')
const helmet = require('helmet')
const app = express()

app.disable('x-powered-by')                 // no revelar el framework
app.use(helmet({                            // CSP, X-Frame-Options, nosniff, Referrer-Policy, HSTS, COOP/CORP...
  contentSecurityPolicy: { directives: { defaultSrc: ["'self'"], frameAncestors: ["'none'"], objectSrc: ["'none'"] } }
}))
app.use((req, res, next) => {               // helmet no emite Permissions-Policy ni Cache-Control
  res.set('Permissions-Policy', 'camera=(), geolocation=(), microphone=()')
  res.set('Cache-Control', 'no-store, max-age=0')
  next()
})

const escape = (s = '') => String(s).replace(/[&<>"']/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]))

app.get('/', (req, res) => {
  res.send('<h1>Demo DAST</h1><a href="/hello?name=UTEC">Saludar</a> · <a href="/login">Login</a>')
})

app.get('/hello', (req, res) => {
  res.send(`<h1>Hola ${escape(req.query.name)}</h1>`)   // salida codificada -> sin XSS reflejado
})

app.get('/login', (req, res) => {
  res.cookie('session', 'abc123', { httpOnly: true, secure: true, sameSite: 'strict' })
  res.send('<form method="post"><input name="user"><input name="pass" type="password"><button>Entrar</button></form>')
})

app.listen(3000, () => console.log('demo-app en http://localhost:3000'))
```

| Cambio | Alertas que cierra |
|---|---|
| `app.disable('x-powered-by')` | 10037 |
| `helmet()` con CSP | 10020 (vía `frame-ancestors` y `X-Frame-Options`), 10021, 10038, 90004 (COOP/CORP), HSTS, Referrer-Policy |
| `Permissions-Policy` y `Cache-Control` manuales | 10063, 10049 |
| Cookie con `httpOnly`, `secure`, `sameSite` | 10010, 10011, 10054 |
| `escape()` en la salida | 40012 (XSS reflejado) |

### 5.2 Reconstruir y volver a escanear

```bash
docker build -t demo-dast:2 .
docker rm -f demo-app
docker run -d --name demo-app --network dast-net -p 3000:3000 demo-dast:2

curl -s -I http://localhost:3000/          # ahora ~12 cabeceras de seguridad
curl -s 'http://localhost:3000/hello?name=<b>x</b>'   # <h1>Hola &lt;b&gt;x&lt;/b&gt;</h1>

docker run --rm --network dast-net -v "$(pwd)/reports:/zap/wrk/:rw" \
  -t zaproxy/zap-stable zap-baseline.py -t http://demo-app:3000 -c rules.conf -r baseline-v2.html -s
echo "exit=$?"
```

Salida observada:

```
IGNORE: Non-Storable Content [10049] x 5
WARN-NEW: CSP: Failure to Define Directive with No Fallback [10055] x 7
WARN-NEW: Authentication Request Identified [10111] x 1
WARN-NEW: Cross-Origin-Embedder-Policy Header Missing or Invalid [90004] x 3
FAIL-NEW: 0  FAIL-INPROG: 0  WARN-NEW: 3  WARN-INPROG: 0  INFO: 0  IGNORE: 1  PASS: 64
exit=2
```

De 11 alertas a 4 (3 WARN + 1 IGNORE), y **ninguna FAIL**: el gate pasa. Las 4 restantes son informativas o de ajuste fino:

| ID | Por qué sigue | Qué harías |
|---|---|---|
| 10049 Non-Storable | ZAP avisa que `no-store` impide cachear (es lo que pedimos); ya sale como IGNORE porque `rules.conf` lo marca así | nada |
| 10055 CSP directive without fallback | helmet no define `frame-src`/`worker-src` explícitos | agregar directivas o IGNORE |
| 10111 Authentication Request | ZAP detectó un formulario de login (informativo) | nada |
| 90004 COEP | helmet 8 no emite `Cross-Origin-Embedder-Policy` por defecto | `crossOriginEmbedderPolicy: true` si la app lo tolera |

Si repites el full scan (Parte 4) contra la v2, la alerta 40012 desaparece: el payload vuelve codificado como texto.

---

## Parte 6: Limpiar

```bash
docker rm -f demo-app
docker network rm dast-net
```

---

## ✅ Checklist

- [ ] `curl -I` muestra solo `X-Powered-By` y `Content-Type` en la v1
- [ ] Baseline sobre v1: 11 WARN, exit 2
- [ ] Con `rules.conf`: 4 FAIL, exit 1
- [ ] Full scan sobre v1 reporta `Cross Site Scripting (Reflected) [40012]` con el payload en `name`
- [ ] Baseline sobre v2 con `rules.conf`: 0 FAIL, exit 2
- [ ] Puedes explicar por qué el XSS no aparece en el baseline

---

## 🔧 Troubleshooting

| Error | Causa | Solución |
|---|---|---|
| `error from registry: denied` al descargar `ghcr.io/zaproxy/zaproxy` | tu red o proxy bloquea GitHub Container Registry | usa `zaproxy/zap-stable` (Docker Hub, misma imagen) |
| `Target Unreachable` / exit 3 | ZAP no está en la misma red que la app, o usaste `localhost` | `--network dast-net` y target `http://demo-app:3000` |
| El reporte no aparece en `reports/` | el volumen no está montado en `/zap/wrk` | `-v "$(pwd)/reports:/zap/wrk/:rw"` |
| `Permission denied` al escribir el reporte | ZAP corre como uid 1000 y la carpeta es de otro usuario | `chmod 777 reports` o `-u $(id -u)` en el `docker run` |
| Reglas de `rules.conf` ignoradas | columnas separadas con espacios | separa con tabulador (`\t`) |
| Exit 2 aunque "todo está bien" | WARN sin FAIL devuelve 2 | usa `-I` para no fallar por WARN, o define FAIL en `rules.conf` |

---

## ➡️ Siguiente paso

Este ejemplo usa una app de juguete. En `guia_dast_zap.md` aplicarás lo mismo a **Juice Shop** (SPA Angular + API REST) y lo integrarás al pipeline de GitLab del laboratorio de SCA/SAST, con gate, escaneo de API por OpenAPI y carga de hallazgos a DefectDojo.

| Capa | Qué resuelve | Dónde |
|---|---|---|
| SAST / SCA / secretos | código y dependencias, sin app corriendo | `guia_secret_scanning_sca_sast.md` |
| DAST pasivo | configuración HTTP de la app desplegada, gate de PR | `guia_dast_zap.md` Partes 2–3, 6 |
| DAST activo | vulnerabilidades explotables (XSS, inyección) | `guia_dast_zap.md` Partes 4–5 |
| Gestión centralizada | un backlog para todos los hallazgos | `guia_defect_dojo.md` |

---

**Autor:** Wilson Julca Mejía  
Curso: *DevSecOps y Seguridad en CI/CD – UTEC*  
Universidad de Ingeniería y Tecnología (UTEC)
