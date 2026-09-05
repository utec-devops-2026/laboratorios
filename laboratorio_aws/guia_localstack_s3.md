# Guía: El mismo lab de S3, sin cuenta AWS, con LocalStack

**Duración estimada:** 45–60 min  
**Nivel:** Principiante–Intermedio  
**Contexto:** LocalStack emula la API de AWS en un contenedor Docker. Los comandos `aws s3` y `aws s3api` funcionan igual, pero contra `http://localhost:4566`: sin cuenta AWS, sin credenciales reales y sin costo. En esta guía harás el ciclo completo de un bucket S3 (crear → poblar → compartir → publicar como sitio web → destruir), lo automatizarás en dos scripts y verás en qué se diferencia de AWS real.

**Esta guía es autocontenida.** No necesitas haber hecho ningún otro laboratorio. Existe una versión del mismo ejercicio contra AWS real (`laboratorio_aws_s3_cli.md`); los scripts que escribas aquí sirven allá sin cambios.

**Cuándo usar esta guía:**
- No tienes cuenta AWS o no quieres arriesgar costos
- Quieres probar scripts antes de correrlos contra una cuenta real
- Estás sin acceso a internet estable (solo necesitas descargar la imagen una vez)

**Cuándo NO sirve:** para aprender permisos, seguridad o facturación. La edición gratuita de LocalStack **no aplica IAM ni políticas**: todo es accesible siempre. Ver sección "Diferencias con AWS real".

---

## Requisitos

- **Docker** corriendo. Verifica con `docker info` (debe responder sin error) y `docker compose version` (Compose v2). Instalación: https://docs.docker.com/get-docker/
- **AWS CLI v2 ≥ 2.13**. Verifica con `aws --version`. Versiones anteriores no soportan `endpoint_url` en el perfil. Instalación: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
  - macOS: `brew install awscli`
  - Linux: `curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip && unzip awscliv2.zip && sudo ./aws/install`
- `curl`, `bash` (macOS/Linux; en Windows usa WSL2 o Git Bash)
- **No necesitas cuenta AWS ni `aws configure` previo.** Las credenciales las inventamos en el Paso 2.

---

## Paso 1: Levantar LocalStack

Crea una carpeta de trabajo y un `docker-compose.yml`:

```bash
mkdir -p localstack-lab && cd localstack-lab
cat > docker-compose.yml <<'YAML'
services:
  localstack:
    image: localstack/localstack:4.9   # NO uses latest: ver nota abajo
    container_name: localstack
    ports:
      - "4566:4566"          # puerto único para todos los servicios
    environment:
      - SERVICES=s3,sts      # solo S3 y STS: más rápido y menos memoria. sts es necesario para get-caller-identity
      - DEBUG=0
    volumes:
      - "./volume:/var/lib/localstack"
YAML

docker compose up -d
```

> ⚠️ **No uses `latest`.** Desde 2026 la imagen `latest` (versionado `2026.x`) exige `LOCALSTACK_AUTH_TOKEN` y se apaga con `License activation failed` si no lo tiene. La serie **4.x** (`4.9` es la última) sigue siendo edición *community*, sin token. Verificado el 2026-09-04 con `4.9.2`.

Verifica que está sano:

```bash
curl -s http://localhost:4566/_localstack/health | python3 -m json.tool
# Debe mostrar "s3": "available", "sts": "available" y "edition": "community"
docker compose logs --tail 20 localstack
```

---

## Paso 2: Perfil de AWS CLI apuntando a LocalStack

LocalStack acepta cualquier credencial. Crea un perfil dedicado con `endpoint_url` para no tener que escribir `--endpoint-url` en cada comando:

```bash
aws configure set aws_access_key_id     test      --profile localstack
aws configure set aws_secret_access_key test      --profile localstack
aws configure set region                us-east-1 --profile localstack
aws configure set output                json      --profile localstack
aws configure set endpoint_url http://localhost:4566 --profile localstack
```

Revisa lo que quedó en `~/.aws/config`:

```ini
[profile localstack]
region = us-east-1
output = json
endpoint_url = http://localhost:4566
```

Activa el perfil para toda la sesión y comprueba:

```bash
export AWS_PROFILE=localstack
aws sts get-caller-identity
# Devuelve una cuenta ficticia: "Account": "000000000000"
aws s3 ls
# Vacío, sin error
```

> **Importante:** mientras `AWS_PROFILE=localstack` esté exportado, **ningún comando toca AWS real**. Para volver: `unset AWS_PROFILE`. Verifica siempre con `aws sts get-caller-identity`: si el Account es `000000000000`, estás en LocalStack.

**Alternativas** (no necesarias si usaste el perfil):
- Flag por comando: `aws --endpoint-url http://localhost:4566 s3 ls`
- Variable de entorno: `export AWS_ENDPOINT_URL=http://localhost:4566`
- Wrapper `awslocal` (`pip install awscli-local`): `awslocal s3 ls`

---

## Paso 3: Ciclo completo de S3

### 3.1 Variables `$BUCKET` y `$REGION` (obligatorio)

**Todos los comandos siguientes usan `$BUCKET` y `$REGION`.** Son variables de tu shell: las defines aquí una vez y el resto de comandos las reutiliza. Si las saltas verás errores como `Invalid bucket name ""`.

Defínelas en la misma terminal donde exportaste `AWS_PROFILE=localstack`:

```bash
# REGION: la lee del perfil localstack que creaste en el Paso 2 (us-east-1)
export REGION=$(aws configure get region)

# BUCKET: prefijo del lab + tu usuario en minúsculas + sufijo "local"
export BUCKET="utec-s3-lab-$(whoami | tr '[:upper:]' '[:lower:]' | tr -cd 'a-z0-9')-local"

# Comprueba ANTES de seguir
echo "Región: $REGION"      # esperado: us-east-1
echo "Bucket: $BUCKET"      # ejemplo: utec-s3-lab-wilson-local
```

Reglas de nombre de bucket (S3 las valida, LocalStack también): solo minúsculas, números y guiones, 3–63 caracteres. En AWS real además debe ser único en todo el mundo; en LocalStack solo dentro de tu contenedor. Si prefieres un nombre manual: `export BUCKET="utec-s3-lab-tunombre-local"`.

> **Las variables viven solo en esta terminal.** Si abres otra pestaña, vuelve a ejecutar `export AWS_PROFILE=localstack`, `export REGION=...` y `export BUCKET=...`.

### 3.2 Crear bucket

```bash
aws s3 mb "s3://$BUCKET"
aws s3api head-bucket --bucket "$BUCKET" && echo "Bucket existe"
aws s3api list-buckets --query 'Buckets[].{Nombre:Name,Creado:CreationDate}' --output table
```

### 3.3 Crear el sitio local

Crea una carpeta `site/` con tres archivos: página principal, página de error y una hoja de estilos en subcarpeta (para ver cómo `sync` respeta la estructura).

```bash
mkdir -p site/assets
cat > site/index.html <<'HTML'
<!DOCTYPE html>
<html lang="es">
<head><meta charset="utf-8"><title>Lab S3 LocalStack</title><link rel="stylesheet" href="assets/style.css"></head>
<body>
  <h1>Sitio estático en S3 (LocalStack)</h1>
  <p>Desplegado con AWS CLI.</p>
</body>
</html>
HTML

cat > site/error.html <<'HTML'
<!DOCTYPE html>
<html lang="es"><body><h1>404 - No encontrado</h1></body></html>
HTML

cat > site/assets/style.css <<'CSS'
body { font-family: sans-serif; max-width: 40rem; margin: 3rem auto; }
h1 { color: #232f3e; }
CSS

find site -type f
```

### 3.4 Subir contenido

`cp` sube un archivo; `sync` sube una carpeta completa respetando subcarpetas.

```bash
aws s3 cp site/index.html "s3://$BUCKET/index.html"
aws s3 sync site/ "s3://$BUCKET/"
aws s3 sync site/ "s3://$BUCKET/"                 # ⚠️ en LocalStack vuelve a subir todo (ver nota)
aws s3 ls "s3://$BUCKET" --recursive --human-readable --summarize
aws s3api list-objects-v2 --bucket "$BUCKET" \
  --query 'Contents[].{Key:Key,Bytes:Size}' --output table
```

> ⚠️ **Diferencia:** `sync` compara tamaño y fecha de modificación local contra `LastModified` en S3. LocalStack devuelve fechas que hacen que la CLI considere los archivos "cambiados", así que la segunda ejecución resube todo. En AWS real la segunda ejecución no sube nada. Para ver el comportamiento idempotente de `sync`, hazlo en la cuenta real.

### 3.5 Acceso directo al objeto

LocalStack expone los objetos en dos formatos de URL:

```bash
# Path-style (siempre funciona)
curl -s -o /dev/null -w "HTTP %{http_code}\n" "http://localhost:4566/$BUCKET/index.html"

# Virtual-host style (igual que AWS; el dominio localhost.localstack.cloud resuelve a 127.0.0.1)
curl -s -o /dev/null -w "HTTP %{http_code}\n" "http://$BUCKET.s3.localhost.localstack.cloud:4566/index.html"
```

> ⚠️ **Diferencia clave:** ambos devuelven `HTTP 200` **aunque no hayas aplicado ninguna política**. En AWS real sería `403`. LocalStack gratuito no evalúa permisos.

### 3.6 URL prefirmada

```bash
URL=$(aws s3 presign "s3://$BUCKET/index.html" --expires-in 120)
echo "$URL"          # apunta a http://localhost:4566/... con firma y expiración
curl -s "$URL" | head -5
```

La firma se genera localmente en tu CLI, así que el mecanismo es idéntico al real. LocalStack sí valida expiración: espera 2 minutos y repite el `curl`.

### 3.7 Publicar como sitio web

Los mismos tres pasos. LocalStack acepta y guarda la configuración aunque no la aplique:

```bash
aws s3api put-public-access-block --bucket "$BUCKET" \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

cat > policy.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadGetObject",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/*"
  }]
}
JSON
aws s3api put-bucket-policy --bucket "$BUCKET" --policy file://policy.json
aws s3api get-bucket-policy --bucket "$BUCKET" --query Policy --output text

aws s3 website "s3://$BUCKET/" --index-document index.html --error-document error.html
aws s3api get-bucket-website --bucket "$BUCKET"
```

El endpoint website de LocalStack:

```bash
export SITE_URL="http://$BUCKET.s3-website.localhost.localstack.cloud:4566"
curl -s "$SITE_URL" | head -5
curl -s -o /dev/null -w "HTTP %{http_code}\n" "$SITE_URL/no-existe.html"    # 404 → error.html
```

Ábrelo en el navegador. Funciona porque `*.localhost.localstack.cloud` es un DNS público que apunta a `127.0.0.1`. Si tu red bloquea esa resolución, usa `http://localhost:4566/$BUCKET/index.html` (sin `index-document` automático).

---

## Paso 4: Automatizar con `deploy_site.sh`

Todo lo del Paso 3 en un script **idempotente**: ejecutarlo dos veces no falla ni duplica nada. No lleva nada específico de LocalStack: la CLI hereda `endpoint_url` del perfil activo, así que el mismo script sirve contra AWS real cambiando `AWS_PROFILE`.

Crea `deploy_site.sh`:

```bash
#!/usr/bin/env bash
# Despliega una carpeta como sitio estático en S3.
# Uso: ./deploy_site.sh <bucket> <carpeta> [region]
set -euo pipefail

BUCKET="${1:?Uso: $0 <bucket> <carpeta> [region]}"
SRC="${2:?Uso: $0 <bucket> <carpeta> [region]}"
REGION="${3:-$(aws configure get region)}"

[[ -d "$SRC" ]] || { echo "La carpeta '$SRC' no existe"; exit 3; }
aws sts get-caller-identity >/dev/null || { echo "No autenticado o endpoint caído"; exit 2; }

if aws s3api head-bucket --bucket "$BUCKET" >/dev/null 2>&1; then
  echo "Bucket $BUCKET ya existe, reutilizando"
else
  echo "Creando bucket $BUCKET en $REGION"
  aws s3 mb "s3://$BUCKET" --region "$REGION"
fi

echo "Configurando acceso público"
aws s3api put-public-access-block --bucket "$BUCKET" \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

aws s3api put-bucket-policy --bucket "$BUCKET" --policy "$(cat <<JSON
{"Version":"2012-10-17","Statement":[{"Sid":"PublicReadGetObject","Effect":"Allow",
"Principal":"*","Action":"s3:GetObject","Resource":"arn:aws:s3:::$BUCKET/*"}]}
JSON
)"

echo "Sincronizando $SRC"
aws s3 sync "$SRC/" "s3://$BUCKET/" --delete

aws s3 website "s3://$BUCKET/" --index-document index.html --error-document error.html

# SITE_URL_OVERRIDE permite apuntar a otro endpoint (LocalStack) sin tocar el script
SITE_URL="${SITE_URL_OVERRIDE:-http://$BUCKET.s3-website-$REGION.amazonaws.com}"
printf '\n✅ Sitio desplegado: %s\n' "$SITE_URL"
curl -s -o /dev/null -w "   HTTP %{http_code}\n" "$SITE_URL"
```

Qué hace cada bloque:
- `set -euo pipefail`: aborta ante cualquier comando fallido, variable sin definir o error en un pipe. Estándar mínimo en scripts de infraestructura.
- `${1:?...}`: si falta el argumento, imprime el uso y sale.
- `head-bucket`: exit code 0 si el bucket existe. Así el script crea solo cuando hace falta (idempotencia).
- `sync --delete`: deja el bucket idéntico a la carpeta local, borrando lo que sobra.
- `SITE_URL_OVERRIDE`: la URL del sitio es lo único que cambia entre LocalStack y AWS real.

> **Ruta rápida.** El script no depende de los ejercicios anteriores: crea el bucket, la política y el website por sí mismo. Solo exige dos cosas ya hechas: (1) CLI autenticada (`aws sts get-caller-identity` responde) y (2) la carpeta `site/` con archivos (bloque "Crear el sitio local"). Si falta la carpeta, corta con `La carpeta 'site' no existe`.

Los argumentos son **strings planos** y se leen por posición (`$1`, `$2`, `$3` dentro del script):

| Posición | Valor en el ejemplo | Variable en el script | Qué es |
|---|---|---|---|
| `$1` | `"$BUCKET"` | `BUCKET` | Nombre del bucket, sin `s3://` |
| `$2` | `site` | `SRC` | Carpeta local a subir (la que creaste con `mkdir -p site/assets`) |
| `$3` | `us-east-1` (opcional) | `REGION` | Región; si se omite usa `aws configure get region` |

`"$BUCKET"` solo expande la variable; da lo mismo escribir el nombre a mano. `site` es una ruta relativa: si tu carpeta se llamara `web/`, pasarías `web`.

```bash
./deploy_site.sh "$BUCKET" site                          # usando la variable
./deploy_site.sh utec-s3-lab-wilson-1757000000 site      # mismo resultado con el nombre literal
./deploy_site.sh "$BUCKET" site us-east-1                # tercer argumento opcional: región
```

Ejecuta dos veces y compara la salida:

```bash
chmod +x deploy_site.sh

export SITE_URL_OVERRIDE="http://$BUCKET.s3-website.localhost.localstack.cloud:4566"
./deploy_site.sh "$BUCKET" site      # crea el bucket
./deploy_site.sh "$BUCKET" site      # "ya existe, reutilizando"; termina igual en HTTP 200
```

Salida esperada de la segunda ejecución:

```
Bucket utec-s3-lab-wilson-local ya existe, reutilizando
Configurando acceso público
Sincronizando site
...
✅ Sitio desplegado: http://utec-s3-lab-wilson-local.s3-website.localhost.localstack.cloud:4566
   HTTP 200
```

**Mismo script contra AWS real** (cuando tengas cuenta y `aws configure` hecho en el perfil `default`):

```bash
unset SITE_URL_OVERRIDE
AWS_PROFILE=default ./deploy_site.sh "utec-s3-lab-tunombre-$(date +%s)" site
```

Este es el flujo profesional: **probar el script en LocalStack, ejecutarlo en real solo cuando ya no falla**.

---

## Paso 5: Cleanup

Tres niveles: borrar el bucket (igual que en AWS real), apagar el contenedor y salir del perfil.

### 5.1 Borrar el bucket a mano

Un bucket **no se puede borrar si tiene objetos**. El orden importa:

```bash
aws s3api delete-bucket-policy --bucket "$BUCKET"        # 1. quitar la política
aws s3 rm "s3://$BUCKET" --recursive                      # 2. vaciar
aws s3 rb "s3://$BUCKET"                                  # 3. borrar el bucket
aws s3api head-bucket --bucket "$BUCKET" >/dev/null 2>&1 || echo "Bucket eliminado"
aws s3api list-buckets --query 'Buckets[].Name' --output text     # vacío
```

Atajo de pasos 2 y 3: `aws s3 rb "s3://$BUCKET" --force`.

### 5.2 Script `cleanup.sh`

Vuelve a desplegar (`./deploy_site.sh "$BUCKET" site`) y ahora bórralo con un script. Crea `cleanup.sh`:

```bash
#!/usr/bin/env bash
# Elimina un bucket S3 y todo su contenido.
# Uso: ./cleanup.sh <bucket>
set -euo pipefail

BUCKET="${1:?Uso: $0 <bucket>}"

if ! aws s3api head-bucket --bucket "$BUCKET" >/dev/null 2>&1; then
  echo "El bucket $BUCKET no existe. Nada que limpiar."
  exit 0
fi

read -r -p "Se eliminará s3://$BUCKET y TODO su contenido. ¿Continuar? (y/N): " ok
[[ "$ok" =~ ^[Yy]$ ]] || { echo "Cancelado"; exit 0; }

aws s3api delete-bucket-policy --bucket "$BUCKET" 2>/dev/null || true
aws s3 rm "s3://$BUCKET" --recursive
aws s3 rb "s3://$BUCKET"

echo "✅ Bucket $BUCKET eliminado"
```

```bash
chmod +x cleanup.sh
./cleanup.sh "$BUCKET"      # pide confirmación, borra
./cleanup.sh "$BUCKET"      # "no existe. Nada que limpiar." y exit 0: también idempotente
```

### 5.3 Apagar LocalStack y salir del perfil

```bash
# 2. Apagar LocalStack
docker compose down          # conserva ./volume
docker compose down -v       # además borra el volumen
rm -rf volume policy.json

# 3. Salir del perfil
unset AWS_PROFILE SITE_URL_OVERRIDE
aws sts get-caller-identity  # ahora vuelve a tu cuenta real (o falla si no la configuraste)
```

> En la edición gratuita, **el estado se pierde al reiniciar el contenedor** aunque montes el volumen. La persistencia entre reinicios es función de LocalStack Pro. En clase: si reinicias, vuelve a crear el bucket.

---

## Diferencias con AWS real

| Aspecto | AWS real | LocalStack (gratuito) |
|---|---|---|
| Credenciales | Access key real de IAM | Cualquier valor (`test`/`test`) |
| Account ID | Tu cuenta de 12 dígitos | `000000000000` |
| Nombre de bucket | Único global en todo AWS | Único solo en tu contenedor |
| Objeto sin política | `403 Forbidden` | `200 OK` siempre |
| `put-bucket-policy` | Se aplica | Se guarda, no se evalúa |
| Block Public Access | Bloquea políticas públicas | Sin efecto |
| Endpoint objeto | `https://<b>.s3.<region>.amazonaws.com/<k>` | `http://localhost:4566/<b>/<k>` |
| Endpoint website | `http://<b>.s3-website-<region>.amazonaws.com` | `http://<b>.s3-website.localhost.localstack.cloud:4566` |
| HTTPS | Sí (objeto), no (website) | No por defecto |
| Persistencia | Permanente | Se pierde al reiniciar |
| Costo | Capa gratuita, luego pago | Cero |
| `presign` | Firma + expiración validadas | Firma + expiración validadas |
| `sync` segunda vez | No sube nada | Resube todo (fechas distintas) |
| Imagen Docker | — | `4.x` community; `latest` (2026.x) requiere token |
| Terraform | Provider `aws` normal | Provider `aws` con `endpoints { s3 = "http://localhost:4566" }` |

**Conclusión pedagógica:** LocalStack enseña la *API* (comandos, parámetros, flujo, scripting). AWS real enseña las *consecuencias* (permisos, exposición pública, costo). Haz el lab en LocalStack primero para dominar los comandos; luego en real para entender el 403.

---

## Problemas frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| `Could not connect to the endpoint URL: http://localhost:4566` | Contenedor no está arriba | `docker compose ps`, `docker compose up -d` |
| `aws configure set endpoint_url` no tiene efecto | AWS CLI < 2.13 | Actualiza CLI o usa `--endpoint-url` / `AWS_ENDPOINT_URL` |
| Comandos van a AWS real por accidente | `AWS_PROFILE` no exportado en esa terminal | `aws sts get-caller-identity` → Account debe ser `000000000000` |
| `*.localhost.localstack.cloud` no resuelve | DNS corporativo bloquea | Usa path-style `http://localhost:4566/<bucket>/<key>` |
| El bucket desapareció | Reiniciaste el contenedor | Esperado en edición gratuita; recréalo |
| `s3api put-bucket-policy` responde OK pero `curl` da 200 sin política | Normal | LocalStack no evalúa IAM |

---

## Checklist

- [ ] `curl localhost:4566/_localstack/health` muestra S3 disponible
- [ ] Perfil `localstack` con `endpoint_url`; `get-caller-identity` devuelve `000000000000`
- [ ] Bucket creado, `sync` ejecutado dos veces, objetos listados en tabla
- [ ] URL prefirmada funcionó y expiró
- [ ] Sitio accesible en `*.s3-website.localhost.localstack.cloud:4566`
- [ ] `deploy_site.sh` escrito y ejecutado dos veces sin error (idempotente)
- [ ] `cleanup.sh` escrito y ejecutado dos veces; segunda vez "no existe"
- [ ] `docker compose down` y `unset AWS_PROFILE`; `get-caller-identity` ya no devuelve `000000000000`
- [ ] Puedes explicar por qué el `curl` sin política dio 200 aquí y daría 403 en AWS

---

## Comandos útiles

```bash
docker compose up -d / down -v                     # Arrancar / apagar LocalStack
curl localhost:4566/_localstack/health             # Estado de servicios
aws configure set endpoint_url http://localhost:4566 --profile localstack
export AWS_PROFILE=localstack / unset AWS_PROFILE  # Entrar / salir del entorno local
aws --endpoint-url http://localhost:4566 s3 ls     # Sin perfil
awslocal s3 ls                                     # Wrapper (pip install awscli-local)
```

---

¡Listo! Ahora tienes un entorno donde equivocarte es gratis. Úsalo para probar cada script antes de tocar la cuenta real.
