# Guía: Hardening de contenedores y entornos dev/prod con Docker Compose

**Duración estimada:** 60–90 min  
**Nivel:** Avanzado  
**Prerrequisito:** Haber completado `laboratorio_docker_compose_01.md` (stack con `secrets`, redes `public`/`private` y volumen `postgres-data` funcionando)

---

## Objetivos de aprendizaje

- Auditar qué privilegios tiene hoy tu stack antes de tocarlo
- Aplicar el principio de menor privilegio: usuario no-root, filesystem de solo lectura, capabilities mínimas
- Limitar CPU y memoria por servicio con `deploy.resources`
- Rotar logs para que un contenedor no llene el disco
- Separar configuración de desarrollo y producción con **override files** sin duplicar YAML
- **Verificar** cada medida con `docker inspect`, no asumirla

---

## Qué agrega este laboratorio

Los labs 00 y 01 cerraron credenciales, red y persistencia. Pero el contenedor sigue siendo un proceso con más poder del que necesita:

| Lo que quedó abierto en el lab 01 | Consecuencia | Corrección aquí |
|---|---|---|
| Postgres arranca como `root` y Flask depende de que el `Dockerfile` haga bien el `USER` | Un exploit en la app tiene root dentro del contenedor | `user:` explícito + verificación |
| Filesystem del contenedor escribible | Un atacante instala herramientas, modifica el código, deja persistencia | `read_only: true` + `tmpfs` |
| Todas las capabilities por defecto de Docker | `CAP_NET_RAW`, `CAP_MKNOD`, etc. que la app jamás usa | `cap_drop: [ALL]` + `cap_add` mínimo |
| Binarios setuid pueden escalar privilegios | `sudo`, `su`, cualquier setuid mal configurado | `no-new-privileges` |
| Sin límites de CPU/memoria | Un servicio con fuga se come el host y tumba al resto | `deploy.resources.limits` |
| Logs sin rotación | `docker logs` crece hasta llenar el disco | `logging.options` |
| Un solo `compose.yaml` para todo | Dev necesita `ports` y bind mounts que prod no debe tener | `compose.override.yaml` + `compose.prod.yaml` |

---

## Requisitos del laboratorio

- Stack del lab 01 en `flask-docker-app/`
- Docker Compose v2 (`docker compose version` ≥ 2.20)

---

## Estructura del proyecto

```
flask-docker-app/
├── app.py
├── requirements.txt
├── Dockerfile               ← modificado
├── .dockerignore
├── .gitignore
├── .env.dev                 (ignorado por git)
├── pg_password.txt          (ignorado por git)
├── init.sql
├── compose.yaml             ← modificado (base endurecida)
├── compose.override.yaml    ← nuevo (dev, se carga solo)
└── compose.prod.yaml        ← nuevo (prod, se carga con -f)
```

---

## 1. Auditar el punto de partida

Antes de endurecer nada, mide. Levanta el stack del lab 01 y responde con comandos, no con suposiciones:

```bash
cd flask-docker-app
docker compose up -d
```

### ¿Quién corre cada proceso?

```bash
docker compose exec flask id
docker compose exec postgres id
```

`flask` debe dar `uid=1000(appuser)` porque el `Dockerfile` del lab de Docker ya hace `USER appuser`. `postgres` da `uid=0(root)`: el entrypoint de la imagen arranca como root y luego baja a `postgres` con `gosu`, pero el contenedor **sí** tiene root disponible.

### ¿Puede escribir en su propio filesystem?

```bash
docker compose exec flask touch /app/pwned && echo "flask: filesystem ESCRIBIBLE"
docker compose exec flask rm /app/pwned
```

### ¿Qué capabilities tiene?

```bash
docker inspect flask-docker-app-flask-1 --format 'CapAdd={{.HostConfig.CapAdd}} CapDrop={{.HostConfig.CapDrop}}'
```

`[] []` significa: ninguna quitada, ninguna añadida → tiene las **14 por defecto** de Docker (`CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `FSETID`, `KILL`, `SETGID`, `SETUID`, `SETPCAP`, `NET_BIND_SERVICE`, `NET_RAW`, `SYS_CHROOT`, `MKNOD`, `AUDIT_WRITE`, `SETFCAP`). Flask usa cero de ellas.

### ¿Tiene límites?

```bash
docker inspect flask-docker-app-flask-1 --format 'Memory={{.HostConfig.Memory}} NanoCpus={{.HostConfig.NanoCpus}} ReadOnly={{.HostConfig.ReadonlyRootfs}} SecOpt={{.HostConfig.SecurityOpt}}'
```

`Memory=0 NanoCpus=0` = sin límite. `ReadOnly=false`. `SecOpt=[]`.

Anota estos resultados. Al final del lab vas a repetir exactamente estos comandos.

```bash
docker compose down
```

---

## 2. Principio de menor privilegio

Un contenedor **no es** una VM: comparte el kernel del host. Root dentro del contenedor es root frente al kernel, con las capabilities que Docker le dejó. Cada privilegio que no quitas es un privilegio que un atacante hereda gratis si compromete la app.

La regla: el proceso arranca con lo **mínimo** que necesita para funcionar, y nada más. Cuatro palancas en Compose:

| Palanca | Qué quita | Clave YAML |
|---|---|---|
| Usuario no-root | Poder de root dentro del contenedor | `user:` |
| Filesystem read-only | Capacidad de modificar la imagen en caliente | `read_only: true` |
| Capabilities | Privilegios de kernel granulares | `cap_drop` / `cap_add` |
| No new privileges | Escalada vía binarios setuid/setgid | `security_opt` |

---

## 3. Mejorar el Dockerfile

El `Dockerfile` del lab de Docker ya crea `appuser`. Ajustes para que funcione con filesystem de solo lectura:

```bash
vim Dockerfile
```

```dockerfile
FROM python:3.11-slim

# No escribir .pyc (el filesystem será read-only) ni bufferizar stdout (logs en tiempo real)
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Usuario de sistema sin shell de login ni home escribible
RUN adduser --system --group --no-create-home appuser

COPY --chown=appuser:appuser app.py .

USER appuser

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--worker-tmp-dir", "/dev/shm", "app:app"]
```

| Cambio | Por qué |
|---|---|
| `PYTHONDONTWRITEBYTECODE=1` | Con `read_only: true`, Python no podría escribir `__pycache__`. Mejor no intentarlo |
| `adduser --system --no-create-home` | Sin home, sin shell de login: menos superficie que un usuario normal |
| `COPY --chown` | Evita un `RUN chown -R` que duplica la capa |
| `--worker-tmp-dir /dev/shm` | Gunicorn escribe el heartbeat de sus workers en un archivo temporal. `/dev/shm` sigue siendo escribible con `read_only: true` |

---

## 4. Crear el `compose.yaml` endurecido (base)

Este archivo es la **base común** a dev y prod. Regla: solo va aquí lo que es cierto en ambos entornos. Por eso **no tiene `ports`** — dev y prod los publican distinto (paso 9).

```bash
vim compose.yaml
```

```yaml
services:
  flask:
    build: .
    image: flask-docker-app:3.0
    user: "appuser"
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    env_file:
      - .env.dev
    environment:
      - APP_VERSION=3.0.0
      - DB_HOST=postgres
      - DB_DATABASE=mydb
      - DB_USER=myuser
    networks:
      - public
      - private
    depends_on:
      postgres:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 256M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  postgres:
    image: postgres:16.3
    restart: unless-stopped
    read_only: true
    tmpfs:
      - /tmp
      - /var/run/postgresql
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - FOWNER
      - SETGID
      - SETUID
    security_opt:
      - no-new-privileges:true
    environment:
      - POSTGRES_USER=myuser
      - POSTGRES_DB=mydb
      - POSTGRES_PASSWORD_FILE=/run/secrets/pg_password
    secrets:
      - pg_password
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - private
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 5s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

secrets:
  pg_password:
    file: ./pg_password.txt

volumes:
  postgres-data:

networks:
  public:
  private:
    internal: true
```

### Puntos clave

| Configuración | Propósito |
|---|---|
| `user: "appuser"` | Redundante con el `USER` del Dockerfile, y eso es deliberado: si alguien cambia la imagen, Compose sigue forzando no-root |
| `read_only: true` | El rootfs del contenedor se monta de solo lectura. La imagen no puede modificarse en caliente |
| `tmpfs: /tmp` | Lo poco que la app necesita escribir vive en RAM y muere con el contenedor |
| `cap_drop: [ALL]` en flask | Cero capabilities. Gunicorn escucha en 5000 (> 1024), no necesita ni `NET_BIND_SERVICE` |
| `cap_add` en postgres | El entrypoint arranca como root, hace `chown` del data dir y baja a `postgres` con `gosu`. Esas cinco caps son el mínimo para ese baile |
| `no-new-privileges:true` | El kernel ignora bits setuid/setgid de cualquier binario. Aunque haya un `sudo` en la imagen, no escala |
| `deploy.resources.limits` | Compose v2 lo aplica también fuera de Swarm. Si el proceso pasa `memory`, el kernel lo mata (OOM) en vez de tumbar el host |
| `logging.options` | 3 archivos de 10 MB máximo por contenedor = 30 MB de logs, nunca más |

> **¿Por qué postgres no lleva `user: postgres`?** Podrías ponerlo y entonces `cap_drop: [ALL]` sin ningún `cap_add` funcionaría — más estricto. El costo: `pg_password.txt` se monta con los permisos del host, y si tienes el archivo en `600` (lo correcto) el uid `999` del contenedor no puede leerlo. Es una decisión real de trade-off, la exploras en el paso 13.

> **`read_only` y bases de datos.** Postgres escribe en tres sitios: el data dir (ya es un volumen), el socket Unix en `/var/run/postgresql` y temporales en `/tmp`. Los dos últimos van a `tmpfs`. Si el arranque falla con `Read-only file system`, el log te dice exactamente qué ruta falta: agrégala a `tmpfs`, no quites `read_only`.

---

## 5. Conceptos: Linux capabilities

Históricamente Linux tenía dos niveles: root (todo) o usuario (nada privilegiado). Las **capabilities** parten "todo" en ~40 permisos independientes. Docker le da 14 a cada contenedor por defecto.

| Capability | Permite | ¿La necesita Flask? | ¿Postgres (entrypoint)? |
|---|---|---|---|
| `NET_BIND_SERVICE` | Escuchar en puertos < 1024 | No (usa 5000) | No (usa 5432) |
| `NET_RAW` | Sockets raw: `ping`, sniffing, spoofing | **No** | **No** |
| `CHOWN` | Cambiar dueño de archivos | No | Sí (`chown` del data dir) |
| `DAC_OVERRIDE` | Saltarse permisos de archivos | No | Sí (escribe en el volumen antes de bajar a `postgres`) |
| `FOWNER` | Operaciones que requieren ser dueño | No | Sí |
| `SETUID` / `SETGID` | Cambiar de usuario/grupo | No | Sí (`gosu postgres`) |
| `MKNOD` | Crear dispositivos | **No** | **No** |
| `SYS_CHROOT`, `KILL`, `AUDIT_WRITE`, `SETPCAP`, `FSETID`, `SETFCAP` | Varios | **No** | **No** |

**Método:** parte de `cap_drop: [ALL]`, levanta el servicio, mira qué falla, agrega **solo** esa cap. Nunca al revés.

---

## 6. Conceptos: límites de recursos

Sin límites, un contenedor puede usar toda la CPU y RAM del host. En un stack de varios servicios, uno con una fuga de memoria se lleva a los demás por delante — y si el host es una VM pequeña, al propio Docker.

```yaml
deploy:
  resources:
    limits:            # techo: el kernel lo hace cumplir
      cpus: "0.50"     # medio core (50% de un núcleo, o 25% de dos)
      memory: 256M     # si lo supera → OOMKilled
    reservations:      # piso: Compose intenta garantizarlo (informativo fuera de Swarm)
      memory: 128M
```

| Recurso | Qué pasa al superarlo |
|---|---|
| `memory` | El kernel mata el proceso (`OOMKilled`). `restart: unless-stopped` lo levanta de nuevo |
| `cpus` | No se mata: el scheduler simplemente no le da más tiempo de CPU. La app va más lenta |

¿Cómo elegir el número? Mide primero:

```bash
docker stats --no-stream
```

Toma el pico real bajo carga y dale un margen del 50–100%. `256M` para Flask con gunicorn es generoso; `512M` para Postgres en dev, suficiente.

> Estos límites son el mismo concepto que `resources.limits` en Kubernetes. Lo que aprendes aquí lo reusas directo en `laboratorio_kubernetes/`.

---

## 7. Levantar y verificar el hardening

```bash
docker compose up --build -d
docker compose ps
```

Ambos `running`, postgres `healthy`. Si `flask` no arranca, ve a Troubleshooting antes de seguir.

Ahora repite **los mismos comandos del paso 1**:

### 7.1 Usuario

```bash
docker compose exec flask id
```

`uid=100(appuser)` o similar — **no `0`**. (El uid exacto depende de `adduser --system`; lo que importa es que no sea root.)

### 7.2 Filesystem

```bash
docker compose exec flask touch /app/pwned || echo "OK: read-only"
docker compose exec flask touch /tmp/ok && echo "OK: /tmp escribible (tmpfs)"
docker inspect flask-docker-app-flask-1 --format 'ReadOnly={{.HostConfig.ReadonlyRootfs}}'
```

Primer comando: `Read-only file system`. Segundo: funciona. Tercero: `ReadOnly=true`.

### 7.3 Capabilities

```bash
docker inspect flask-docker-app-flask-1 --format 'CapDrop={{.HostConfig.CapDrop}} CapAdd={{.HostConfig.CapAdd}}'
docker inspect flask-docker-app-postgres-1 --format 'CapDrop={{.HostConfig.CapDrop}} CapAdd={{.HostConfig.CapAdd}}'
```

Flask: `CapDrop=[ALL] CapAdd=[]`. Postgres: `CapDrop=[ALL] CapAdd=[CHOWN DAC_OVERRIDE FOWNER SETGID SETUID]`.

Compruébalo desde dentro: `ping` necesita `NET_RAW`.

```bash
docker compose exec flask python3 -c "import socket; socket.socket(socket.AF_INET, socket.SOCK_RAW)" 2>&1 | tail -1
```

Debe dar `PermissionError: [Errno 1] Operation not permitted`. En el lab 01 esto funcionaba.

### 7.4 No new privileges

```bash
docker inspect flask-docker-app-flask-1 --format 'SecOpt={{.HostConfig.SecurityOpt}}'
```

`SecOpt=[no-new-privileges:true]`.

### 7.5 Límites

```bash
docker inspect flask-docker-app-flask-1 --format 'Memory={{.HostConfig.Memory}} NanoCpus={{.HostConfig.NanoCpus}}'
```

`Memory=268435456` (256 MiB en bytes) `NanoCpus=500000000` (0.5 cores).

```bash
docker stats --no-stream
```

La columna `MEM USAGE / LIMIT` ahora muestra `/ 256MiB` en vez de `/ <toda la RAM del host>`.

### 7.6 Logs

```bash
docker inspect flask-docker-app-flask-1 --format '{{.HostConfig.LogConfig}}'
```

`{json-file map[max-file:3 max-size:10m]}`.

### 7.7 La app sigue funcionando

Todo lo anterior no sirve de nada si rompiste la app:

```bash
curl localhost:8080/api/health
```

**Falla.** `Connection refused`. Es correcto: la base **no publica puertos**. Sigue al paso 8 — ahí entra el override de desarrollo.

---

## 8. Conceptos: override files

Un solo `compose.yaml` para dev y prod es una mentira que se paga tarde. Dev quiere `ports` abiertos, bind mount del código para no rebuildear, `debug=True`. Prod no quiere **nada** de eso.

Compose resuelve esto **apilando** archivos:

```
compose.yaml            ← base: lo que es cierto siempre
compose.override.yaml   ← dev: se carga AUTOMÁTICAMENTE si existe
compose.prod.yaml       ← prod: se carga solo con -f
```

| Comando | Archivos que carga |
|---|---|
| `docker compose up` | `compose.yaml` + `compose.override.yaml` |
| `docker compose -f compose.yaml -f compose.prod.yaml up` | `compose.yaml` + `compose.prod.yaml` (**ignora** el override) |

### Reglas de merge

Los archivos se fusionan en orden; el último gana. Pero "gana" depende del tipo de campo:

| Tipo de campo | Ejemplos | Comportamiento |
|---|---|---|
| Escalar | `image`, `restart`, `read_only`, `command` | El último **reemplaza** |
| Mapa | `environment`, `labels`, `deploy.resources` | Se **fusionan** clave a clave |
| Lista | `ports`, `volumes`, `expose`, `dns` | Se **concatenan** |

La consecuencia importante: **no puedes quitar un `ports` en un override, solo agregar**. Si la base tuviera `"8080:5000"`, prod no podría eliminarlo. Por eso la base no publica nada y cada entorno agrega lo suyo.

> Ver el resultado de la fusión antes de levantar: `docker compose config` (o `docker compose -f ... -f ... config`). Es la única forma fiable de saber qué va a correr.

---

## 9. Crear `compose.override.yaml` (desarrollo)

```bash
vim compose.override.yaml
```

```yaml
services:
  flask:
    ports:
      - "8080:5000"
    volumes:
      - ./app.py:/app/app.py:ro
    environment:
      - FLASK_DEBUG=1
    command: ["gunicorn", "--bind", "0.0.0.0:5000", "--worker-tmp-dir", "/dev/shm", "--reload", "app:app"]
```

| Configuración | Propósito |
|---|---|
| `ports` | Solo dev publica al host. Se **concatena** a la lista vacía de la base |
| `./app.py:/app/app.py:ro` | Bind mount del código: editas en el host, el contenedor lo ve. `:ro` porque el contenedor no debe modificar tu archivo |
| `--reload` | Gunicorn reinicia workers al cambiar `app.py`. Sin rebuild |
| `command` | Escalar: **reemplaza** el `CMD` del Dockerfile solo en dev |

Nota lo que **no** está aquí: `read_only`, `cap_drop`, `user`. El hardening viene de la base y dev lo hereda. Desarrollar con los mismos límites que prod es la única forma de descubrir en tu máquina que algo necesita un privilegio, en vez de descubrirlo en el deploy.

Levanta y prueba:

```bash
docker compose up -d
docker compose config | grep -A3 'ports:'
curl localhost:8080/api/health
```

Prueba el hot reload: cambia el texto de `home()` en `app.py`, guarda y vuelve a hacer `curl localhost:8080/`. Sin rebuild.

---

## 10. Crear `compose.prod.yaml` (producción)

```bash
vim compose.prod.yaml
```

```yaml
services:
  flask:
    image: flask-docker-app:3.0
    build: !reset null
    restart: always
    ports:
      - "127.0.0.1:8080:5000"
    env_file: !reset []
    secrets:
      - pg_password
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M

  postgres:
    restart: always
```

| Configuración | Propósito |
|---|---|
| `build: !reset null` | Prod **no construye**: usa la imagen ya publicada. `!reset` es la única forma de borrar un campo de la base (Compose ≥ 2.24) |
| `restart: always` | En prod el servicio vuelve solo, incluso tras reinicio del host |
| `"127.0.0.1:8080:5000"` | Publica solo en loopback. El puerto es alcanzable desde un reverse proxy en el mismo host, **no** desde la red. Sin esto, `"8080:5000"` escucha en `0.0.0.0` y Docker abre el puerto en el firewall del host por ti |
| `env_file: !reset []` + `secrets` | Prod usa la **Forma B** del lab 01: la contraseña llega como archivo en `/run/secrets`, no como variable de entorno |
| `limits` más altos | Mapa: se fusiona con la base y sobreescribe solo `cpus` y `memory` |

Para que la Forma B funcione, `app.py` debe leer el secreto del archivo con fallback a env (ejercicio 12 del lab 01). Si no lo hiciste, hazlo ahora:

```python
def read_secret(path, fallback_env):
    if os.path.exists(path):
        return open(path).read().strip()
    return os.environ.get(fallback_env)


db_password = read_secret('/run/secrets/pg_password', 'DB_PASSWORD')
```

y úsalo en `db_connect()` en lugar de `os.environ.get('DB_PASSWORD')`.

---

## 11. Probar ambos modos

### Dev (override automático)

```bash
docker compose down
docker compose up --build -d
docker compose exec flask env | grep -E 'DB_PASSWORD|FLASK_DEBUG'
```

Ves ambas variables: dev usa `env_file`.

### Prod (override explícito)

```bash
docker compose down
docker compose -f compose.yaml -f compose.prod.yaml config > /dev/null && echo "YAML prod válido"
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

Verifica lo que cambió:

```bash
docker compose -f compose.yaml -f compose.prod.yaml exec flask env | grep DB_PASSWORD || echo "OK: sin password en env"
docker compose -f compose.yaml -f compose.prod.yaml exec flask cat /run/secrets/pg_password
docker inspect flask-docker-app-flask-1 --format '{{.HostConfig.PortBindings}}'
docker inspect flask-docker-app-flask-1 --format 'Memory={{.HostConfig.Memory}} Restart={{.HostConfig.RestartPolicy.Name}}'
curl localhost:8080/api/health
```

| Comprobación | Dev | Prod |
|---|---|---|
| `DB_PASSWORD` en env | Sí | No |
| `/run/secrets/pg_password` | No existe | Existe |
| `PortBindings` | `0.0.0.0:8080` | `127.0.0.1:8080` |
| `Memory` | 268435456 | 536870912 |
| `Restart` | `no` | `always` |
| Bind mount de `app.py` | Sí | No |

> **Tip:** en vez de repetir `-f compose.yaml -f compose.prod.yaml`, exporta `COMPOSE_FILE=compose.yaml:compose.prod.yaml`. Ponlo en un `.env.prod` que cargues con `--env-file`, o en la shell del servidor.

```bash
docker compose -f compose.yaml -f compose.prod.yaml down
```

---

## 12. Comparar con el punto de partida

Repite los comandos del paso 1 sobre el stack en modo dev y contrasta:

| Medida | Paso 1 (lab 01) | Ahora |
|---|---|---|
| `flask` uid | `1000(appuser)` | `appuser` (sistema, sin home) |
| `postgres` con root disponible | Sí, 14 caps | Sí, **5 caps** |
| `touch /app/x` en flask | Funciona | `Read-only file system` |
| `CapDrop` flask | `[]` | `[ALL]` |
| Socket raw desde flask | Funciona | `Operation not permitted` |
| `SecurityOpt` | `[]` | `[no-new-privileges:true]` |
| `Memory` / `NanoCpus` | `0 / 0` | `268435456 / 500000000` |
| Rotación de logs | Ninguna | 3 × 10 MB |
| Puerto 8080 en prod | `0.0.0.0` | `127.0.0.1` |

---

## 13. Ejercicios

### 13.1 Postgres sin root

Sustituye el bloque de caps de postgres por `user: postgres` y `cap_drop: [ALL]` sin `cap_add`. Levanta con volumen nuevo (`down -v`).

- Si `pg_password.txt` tiene permisos `600`, verás `Permission denied` al leer el secreto. Documenta las dos salidas (`chmod 644` en el host vs. gestionar el secreto de otra forma) y elige una con argumento.
- Verifica con `docker compose exec postgres id` que no hay root.

### 13.2 Provocar un OOM

Baja `memory` de flask a `32M`, levanta y haz peticiones. Observa `docker compose ps` (`Exited (137)`) y `docker inspect --format '{{.State.OOMKilled}}'`. Restaura el valor.

### 13.3 `develop.watch`

Reemplaza el bind mount + `--reload` del override por la sintaxis nativa de Compose:

```yaml
    develop:
      watch:
        - action: sync+restart
          path: ./app.py
          target: /app/app.py
```

y levanta con `docker compose watch`. Compara con el bind mount: ¿qué gana, qué pierde?

### 13.4 `profiles`

Agrega un servicio `adminer` (imagen `adminer`, red `private` + `public`, `ports: "8081:8080"`) con `profiles: [debug]`. Verifica que `docker compose up` **no** lo levanta y `docker compose --profile debug up` sí.

---

## Checklist de éxito

- [ ] Auditoría inicial del paso 1 anotada
- [ ] `Dockerfile` con `PYTHONDONTWRITEBYTECODE`, usuario de sistema y `--worker-tmp-dir /dev/shm`
- [ ] `compose.yaml` base **sin `ports`**, con `user`, `read_only`, `tmpfs`, `cap_drop`, `security_opt`, `deploy.resources`, `logging`
- [ ] `touch /app/x` falla con `Read-only file system`; `/tmp` sí escribe
- [ ] `CapDrop=[ALL]` en flask; socket raw da `Operation not permitted`
- [ ] `Memory` y `NanoCpus` distintos de 0 en `docker inspect`
- [ ] `compose.override.yaml` con `ports`, bind mount `:ro` y `--reload`; hot reload comprobado
- [ ] `compose.prod.yaml` con `127.0.0.1:8080`, `restart: always`, Forma B, sin `build`
- [ ] En prod: `DB_PASSWORD` **no** está en env, `/run/secrets/pg_password` sí
- [ ] `docker compose config` usado para ver la fusión de ambos modos
- [ ] Tabla del paso 12 completa con tus valores reales

---

## Consideraciones de seguridad

- Cada medida de este lab **reduce el daño** de un compromiso, no lo evita. Se apilan: no-root sin `read_only` deja instalar herramientas en `/tmp`; `read_only` sin `cap_drop` deja `NET_RAW` para escanear la red interna
- `no-new-privileges` es gratis y no rompe nada razonable. Debería estar en todo servicio, siempre
- `cap_add` es una deuda documentada: cada cap que agregas, escribe en un comentario por qué
- `"127.0.0.1:puerto"` en prod o un reverse proxy delante. `"puerto:puerto"` a secas abre el firewall del host aunque tengas `ufw`/`iptables` configurados — Docker inserta sus propias reglas antes
- Los límites de memoria son también una defensa: un ataque de agotamiento de recursos contra un servicio no derriba el resto
- Nada de esto sustituye escanear la imagen (`docker scout cves flask-docker-app:3.0` o Trivy) ni actualizar la base `python:3.11-slim` con regularidad
- El siguiente nivel es un perfil **seccomp** o **AppArmor** propio. Docker aplica un seccomp por defecto que bloquea ~44 syscalls; para una app Python puedes bloquear muchas más

---

## Troubleshooting

### `flask` sale con código 1 y el log dice `Read-only file system`

```
OSError: [Errno 30] Read-only file system: '/app/__pycache__'
```

**Causa:** falta `PYTHONDONTWRITEBYTECODE=1` en el `Dockerfile`, o gunicorn intenta escribir su heartbeat fuera de `/dev/shm`.

**Solución:** revisa el `Dockerfile` del paso 3 y reconstruye:
```bash
docker compose build --no-cache flask
docker compose up -d
```

Si la ruta que reclama es otra, agrégala a `tmpfs` en `compose.yaml`. **No** quites `read_only`.

---

### `postgres` no arranca: `could not create lock file "/var/run/postgresql/.s.PGSQL.5432.lock"`

**Causa:** `read_only: true` sin `tmpfs: /var/run/postgresql`.

**Solución:** agrega la ruta al bloque `tmpfs` de postgres y levanta de nuevo.

---

### `postgres` no arranca: `chown: changing ownership ... Operation not permitted`

**Causa:** falta `CHOWN`, `FOWNER` o `DAC_OVERRIDE` en `cap_add`. El entrypoint no puede preparar el data dir.

**Solución:** compara tu `cap_add` con el del paso 4. Si estás en el ejercicio 13.1 (`user: postgres`), este error es esperable en volúmenes ya inicializados como root: `docker compose down -v` y vuelve a levantar.

---

### `Connection refused` en `localhost:8080` tras el paso 7

**Causa:** la base no publica puertos. Es intencional.

**Solución:** crea `compose.override.yaml` (paso 9). Verifica con `docker compose config | grep -A2 ports`.

---

### El override no se aplica

```bash
docker compose config | grep -c '8080'   # 0 = no se cargó
```

**Causas:**
- El archivo se llama `docker-compose.override.yaml` o `compose.override.yml`: Compose busca `compose.override.yaml` cuando la base es `compose.yaml` (y `docker-compose.override.yml` cuando la base es `docker-compose.yml`). No mezcles nombres.
- Usaste `-f`: en cuanto pasas un `-f`, la carga automática se apaga. O lo listas explícitamente o no lo usas.
- `COMPOSE_FILE` está exportada en tu shell apuntando a otro archivo: `echo $COMPOSE_FILE`.

---

### `!reset` da error de YAML

```
yaml: unknown tag !reset
```

**Causa:** Compose < 2.24. Verifica con `docker compose version`.

**Solución:** actualiza Docker Desktop / el plugin de Compose. Como paliativo, en lugar de `!reset` mueve el campo conflictivo (`build`, `env_file`) al override de **dev** para que la base no lo tenga.

---

### `OOMKilled` en `docker compose ps` (`Exited (137)`)

```bash
docker inspect flask-docker-app-flask-1 --format '{{.State.OOMKilled}}'
docker compose logs --tail=20 flask
```

**Causa:** el proceso superó `deploy.resources.limits.memory`.

**Solución:** mide con `docker stats` el consumo real y ajusta el límite con margen. Si el consumo crece sin parar con el tiempo, es una fuga en la app — el límite hizo su trabajo al contenerla.

---

### `Operation not permitted` al hacer algo que en el lab 01 funcionaba

**Causa:** `cap_drop: [ALL]`. La operación necesita una capability.

**Solución:** identifica cuál (`man 7 capabilities`), evalúa si la app la necesita de verdad, y solo entonces agrégala a `cap_add` con un comentario explicando por qué. Ejemplo: escuchar en el puerto 80 requiere `NET_BIND_SERVICE` — pero la solución mejor es escuchar en 8080 y no agregar nada.

---

## Siguiente paso

Todo lo de este lab tiene equivalente directo en Kubernetes:

| Compose | Kubernetes |
|---|---|
| `user`, `read_only`, `cap_drop`, `no-new-privileges` | `securityContext` (`runAsNonRoot`, `readOnlyRootFilesystem`, `capabilities.drop`, `allowPrivilegeEscalation: false`) |
| `deploy.resources.limits` | `resources.limits` / `requests` |
| `secrets` | `Secret` montado como volumen |
| `networks` con `internal` | `NetworkPolicy` |
| Override files | Kustomize overlays / Helm values |

Continúa con `laboratorio_kubernetes/`.
