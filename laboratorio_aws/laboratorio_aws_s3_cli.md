# Lab: Ciclo de vida completo de un bucket S3 con AWS CLI

**Duración estimada:** 60–75 min  
**Nivel:** Principiante–Intermedio  
**Contexto:** Crearás un bucket S3 desde cero, subirás un sitio estático, lo compartirás de forma temporal (URL prefirmada) y luego pública (política de bucket + website hosting), y finalmente eliminarás todo. Todo desde la terminal. Al final automatizarás el flujo en dos scripts: `deploy_site.sh` y `cleanup.sh`.

> Este lab es el "paso manual" de lo que luego harás con Terraform en `laboratorio_IaC/terraform-s3.md`. Cada comando `aws s3api ...` de aquí corresponde a un `resource "aws_s3_..."` allá.

---

## Objetivos de aprendizaje

- Crear y eliminar buckets S3 (`mb`, `rb`)
- Subir archivos individuales y sincronizar carpetas (`cp`, `sync`)
- Inspeccionar objetos con `ls`, `s3api list-objects-v2` y `--query`
- Compartir un objeto privado con URL prefirmada (`presign`)
- Exponer un bucket como sitio web público (`put-public-access-block`, `put-bucket-policy`, `website`)
- Limpiar recursos para no generar costos ni basura en la cuenta

---

## Requisitos

- AWS CLI v2 instalado y configurado (ver `guia_configuracion_aws_cli.md`)
- Usuario IAM con la política `AmazonS3FullAccess` (o equivalente restringida a tus buckets)
- `curl` y `bash`
- `aws sts get-caller-identity` responde sin error

---

## Costos

S3 tiene capa gratuita (5 GB, 20 000 GET, 2 000 PUT al mes durante 12 meses). Este lab usa unos pocos KB. **El costo real es olvidar el bucket creado**: ejecuta siempre el Ejercicio 6 (cleanup) al terminar.

---

## Paso 0 (obligatorio): definir las variables `$BUCKET` y `$REGION`

**Todos los comandos de este lab usan `$BUCKET` y `$REGION`.** No son valores mágicos: son variables de tu shell que defines aquí, una sola vez, y que el resto de comandos reutiliza. Si saltas este paso, verás errores como `Parameter validation failed` o `Invalid bucket name ""`.

Los nombres de bucket son **globales y únicos** en todo AWS, solo minúsculas, números y guiones, 3–63 caracteres. Por eso no puedes usar `mi-bucket`: alguien ya lo tiene. Generamos uno con tu usuario y un timestamp:

```bash
# REGION: la lee de tu ~/.aws/config (la que pusiste en `aws configure`)
export REGION=$(aws configure get region)

# BUCKET: prefijo del lab + tu usuario en minúsculas + segundos desde 1970 (garantiza unicidad)
export BUCKET="utec-s3-lab-$(whoami | tr '[:upper:]' '[:lower:]' | tr -cd 'a-z0-9')-$(date +%s)"

# Comprueba que quedaron definidas ANTES de seguir
echo "Región: $REGION"      # ejemplo: us-east-1
echo "Bucket: $BUCKET"      # ejemplo: utec-s3-lab-wilson-1757000000
```

Si `echo "$REGION"` sale vacío, tu CLI no tiene región configurada: ejecuta `aws configure` o define a mano `export REGION=us-east-1`.

Si prefieres elegir el nombre tú mismo, también vale, siempre que respete las reglas:

```bash
export BUCKET="utec-s3-lab-tunombre-2026"
```

> **Las variables viven solo en esta terminal.** Si abres otra pestaña, cierras la sesión o reinicias, `$BUCKET` desaparece y los comandos fallan. Para no perder el nombre:
>
> ```bash
> echo "$BUCKET" > .bucket_name          # guardar
> export BUCKET=$(cat .bucket_name)      # recuperar en una terminal nueva
> export REGION=$(aws configure get region)
> ```

---

## Ejercicio 1: Crear el bucket

### Tarea 1: Crear y verificar

`$BUCKET` y `$REGION` vienen del Paso 0. Verifica con `echo "$BUCKET"` si tienes dudas.

```bash
aws s3 mb "s3://$BUCKET" --region "$REGION"
# Salida esperada: make_bucket: utec-s3-lab-<tu-usuario>-<timestamp>
```

Verifica de tres formas distintas:

```bash
# 1. Listado simple
aws s3 ls | grep "$BUCKET"

# 2. Verificación "silenciosa" (útil en scripts): exit code 0 si existe
aws s3api head-bucket --bucket "$BUCKET" && echo "Bucket existe"

# 3. Ubicación real del bucket
aws s3api get-bucket-location --bucket "$BUCKET"
```

> `get-bucket-location` devuelve `null` para `us-east-1`. Es comportamiento histórico de S3, no un error.

### Tarea 2: Explorar con `--query` y `--output`

`--query` usa JMESPath para filtrar la salida JSON. Es la herramienta más útil de AWS CLI en scripts.

```bash
# Solo nombres de bucket, uno por línea
aws s3api list-buckets --query 'Buckets[].Name' --output text

# Nombre y fecha en tabla
aws s3api list-buckets --query 'Buckets[].{Nombre:Name,Creado:CreationDate}' --output table

# Solo los buckets del lab
aws s3api list-buckets --query "Buckets[?starts_with(Name, 'utec-s3-lab')].Name" --output text
```

---

## Ejercicio 2: Subir contenido

### Tarea 1: Crear el sitio local

```bash
mkdir -p site/assets
cat > site/index.html <<'HTML'
<!DOCTYPE html>
<html lang="es">
<head><meta charset="utf-8"><title>Lab S3</title><link rel="stylesheet" href="assets/style.css"></head>
<body>
  <h1>Sitio estático en S3</h1>
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
```

### Tarea 2: Subir un archivo individual

```bash
aws s3 cp site/index.html "s3://$BUCKET/index.html"
```

### Tarea 3: Sincronizar la carpeta completa

`sync` sube solo lo que cambió. Ejecútalo dos veces y observa la diferencia:

```bash
aws s3 sync site/ "s3://$BUCKET/"
aws s3 sync site/ "s3://$BUCKET/"     # segunda vez: no sube nada
```

Modifica un archivo y vuelve a sincronizar:

```bash
echo "/* v2 */" >> site/assets/style.css
aws s3 sync site/ "s3://$BUCKET/"     # solo sube style.css
```

> `--delete` borra en S3 lo que ya no existe localmente. `--dryrun` muestra qué haría sin ejecutar. Prueba: `aws s3 sync site/ "s3://$BUCKET/" --delete --dryrun`

### Tarea 4: Inspeccionar objetos

```bash
# Vista "humana"
aws s3 ls "s3://$BUCKET" --recursive --human-readable --summarize

# Vista programable: clave, tamaño y content-type
aws s3api list-objects-v2 --bucket "$BUCKET" \
  --query 'Contents[].{Key:Key,Bytes:Size,Modificado:LastModified}' --output table

# Metadatos de un objeto (Content-Type detectado automáticamente por la CLI)
aws s3api head-object --bucket "$BUCKET" --key index.html --query ContentType
```

---

## Ejercicio 3: Compartir de forma temporal (URL prefirmada)

Por defecto todo objeto es **privado**. Compruébalo:

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" \
  "https://$BUCKET.s3.$REGION.amazonaws.com/index.html"
# Esperado: HTTP 403
```

Una URL prefirmada lleva tu firma embebida y expira. Es la forma correcta de compartir un objeto privado sin tocar permisos:

```bash
URL=$(aws s3 presign "s3://$BUCKET/index.html" --expires-in 120)
echo "$URL"
curl -s "$URL" | head -5
```

Espera 2 minutos y repite el `curl`: recibirás `AccessDenied` con `Request has expired`.

> Pregunta para discutir: ¿por qué la URL funciona sin que el receptor tenga credenciales de AWS? ¿Qué parte de la URL contiene la firma?

---

## Ejercicio 4: Exponer como sitio web público

Necesitas tres cosas, en este orden. Las tres existen como recursos separados en Terraform.

### Tarea 1: Desactivar Block Public Access

Todo bucket nuevo bloquea políticas públicas. Hay que desactivarlo explícitamente:

```bash
aws s3api put-public-access-block --bucket "$BUCKET" \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

aws s3api get-public-access-block --bucket "$BUCKET"
```

### Tarea 2: Aplicar la política de bucket

```bash
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
```

Repite el `curl` del Ejercicio 3 sobre la URL del objeto. Ahora responde `HTTP 200`.

### Tarea 3: Habilitar website hosting

```bash
aws s3 website "s3://$BUCKET/" --index-document index.html --error-document error.html
aws s3api get-bucket-website --bucket "$BUCKET"
```

### Tarea 4: Probar el sitio

```bash
export SITE_URL="http://$BUCKET.s3-website-$REGION.amazonaws.com"
echo "$SITE_URL"
curl -s "$SITE_URL" | head -5
curl -s -o /dev/null -w "HTTP %{http_code}\n" "$SITE_URL/no-existe.html"   # 404 con error.html
```

Abre `$SITE_URL` en el navegador. El endpoint website es solo HTTP; HTTPS requiere CloudFront (fuera de alcance).

> Nota: algunas regiones usan `s3-website.<region>` (punto) en vez de `s3-website-<region>` (guion). Si el guion falla, prueba el punto. `get-bucket-location` te dice la región real.

---

## Ejercicio 5: Automatizar el despliegue

Escribe `deploy_site.sh`. Debe ser **idempotente**: ejecutarlo dos veces no falla ni duplica nada.

```bash
#!/usr/bin/env bash
# Despliega una carpeta como sitio estático en S3.
# Uso: ./deploy_site.sh <bucket> <carpeta> [region]
set -euo pipefail

BUCKET="${1:?Uso: $0 <bucket> <carpeta> [region]}"
SRC="${2:?Uso: $0 <bucket> <carpeta> [region]}"
REGION="${3:-$(aws configure get region)}"

[[ -d "$SRC" ]] || { echo "La carpeta '$SRC' no existe"; exit 3; }
aws sts get-caller-identity >/dev/null || { echo "No autenticado. Ejecuta aws configure"; exit 2; }

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

# SITE_URL_OVERRIDE permite apuntar a otro endpoint (p. ej. LocalStack) sin tocar el script
SITE_URL="${SITE_URL_OVERRIDE:-http://$BUCKET.s3-website-$REGION.amazonaws.com}"
printf '\n✅ Sitio desplegado: %s\n' "$SITE_URL"
curl -s -o /dev/null -w "   HTTP %{http_code}\n" "$SITE_URL"
```

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

```bash
chmod +x deploy_site.sh
./deploy_site.sh "$BUCKET" site
./deploy_site.sh "$BUCKET" site      # segunda ejecución: reutiliza el bucket, sync no sube nada
```

> Observa `set -euo pipefail`: el script aborta ante cualquier comando fallido, variable no definida o error en un pipe. Es el estándar mínimo para scripts de infraestructura.

---

## Ejercicio 6: Cleanup

Un bucket **no se puede borrar si tiene objetos**. El orden importa.

### Tarea 1: Paso a paso

```bash
# 1. Quitar la política pública (buena práctica antes de borrar)
aws s3api delete-bucket-policy --bucket "$BUCKET"

# 2. Vaciar el bucket
aws s3 rm "s3://$BUCKET" --recursive

# 3. Borrar el bucket
aws s3 rb "s3://$BUCKET"

# 4. Verificar que ya no existe (head-bucket falla con exit code != 0)
aws s3api head-bucket --bucket "$BUCKET" >/dev/null 2>&1 || echo "Bucket eliminado"
```

Atajo equivalente a pasos 2 y 3: `aws s3 rb "s3://$BUCKET" --force`.

### Tarea 2: Script `cleanup.sh`

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
./cleanup.sh "$BUCKET"
./cleanup.sh "$BUCKET"     # segunda vez: "no existe", exit 0
rm -f .bucket_name policy.json
```

### Tarea 3: Auditoría final

Confirma que no dejaste buckets del lab en la cuenta:

```bash
aws s3api list-buckets --query "Buckets[?starts_with(Name, 'utec-s3-lab')].Name" --output text
# Esperado: salida vacía
```

---

## Reto (sin solución): mínimo privilegio

Hasta ahora usaste `AmazonS3FullAccess`. Escribe la **política IAM mínima** que permite ejecutar `deploy_site.sh` y `cleanup.sh` sobre buckets con prefijo `utec-s3-lab-*` y **nada más**.

Pistas:
- Inventaría cada comando `aws` de los dos scripts y busca en la documentación qué acción IAM requiere (`aws s3 sync` son varias).
- El ARN del bucket (`arn:aws:s3:::nombre`) y el de sus objetos (`arn:aws:s3:::nombre/*`) son recursos distintos. Algunas acciones aplican a uno, otras al otro.
- Método iterativo: usuario con política vacía → correr el script → leer `AccessDenied ... not authorized to perform: s3:XXX on resource: arn:...` → agregar exactamente eso → repetir.

Entrega: el JSON de la política, y evidencia de que con ella (a) ambos scripts funcionan y (b) `aws s3 mb s3://otro-nombre`, `aws s3 ls` y `aws iam list-users` devuelven `AccessDenied`.

> Solo se puede validar en AWS real. LocalStack gratuito no evalúa IAM.

---

## Criterios de éxito (Checklist)

- [ ] Bucket creado y verificado con `head-bucket`
- [ ] Carpeta sincronizada; segunda ejecución de `sync` no subió nada
- [ ] `curl` al objeto devolvió 403 antes de la política y 200 después
- [ ] URL prefirmada funcionó y expiró
- [ ] Sitio accesible por el endpoint website, con `error.html` en 404
- [ ] `deploy_site.sh` idempotente (dos ejecuciones sin error)
- [ ] `cleanup.sh` ejecutado; auditoría final vacía

---

## Mapa CLI → Terraform

| Comando de este lab | Recurso en `terraform-s3.md` |
|---|---|
| `aws s3 mb` | `aws_s3_bucket` |
| `aws s3 cp` / `sync` | `aws_s3_object` |
| `put-public-access-block` | `aws_s3_bucket_public_access_block` |
| `put-bucket-policy` | `aws_s3_bucket_policy` |
| `aws s3 website` | `aws_s3_bucket_website_configuration` |
| `cleanup.sh` | `terraform destroy` |

---

## Comandos útiles

```bash
aws s3 mb s3://bucket                          # Crear bucket
aws s3 rb s3://bucket --force                  # Vaciar y borrar bucket
aws s3 cp archivo s3://bucket/                 # Subir archivo
aws s3 sync carpeta/ s3://bucket/ --delete     # Sincronizar carpeta
aws s3 ls s3://bucket --recursive              # Listar objetos
aws s3 presign s3://bucket/key --expires-in N  # URL temporal
aws s3api head-bucket --bucket bucket          # ¿Existe? (exit code)
aws s3api put-bucket-policy --policy file://p.json
aws s3 website s3://bucket/ --index-document index.html
aws <servicio> <comando> help                  # Documentación local de cualquier comando
```

---

¡Listo! Ya dominas el ciclo completo de un bucket: crear, poblar, compartir, publicar y destruir. Cuando llegues a Terraform, reconocerás cada paso.
