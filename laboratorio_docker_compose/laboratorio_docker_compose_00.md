# Guía: Docker Compose + PostgreSQL con Flask

**Duración estimada:** 30–45 min  
**Nivel:** Básico–Intermedio  
**Prerrequisito:** Haber completado `laboratorio_docker_01.md` (imagen `flask-docker-app:1.0` construida)

---

## Objetivos de aprendizaje

- Migrar de `docker run` a Docker Compose
- Agregar un servicio PostgreSQL al stack
- Conectar Flask a PostgreSQL con `psycopg`
- Implementar endpoints básicos contra la base de datos
- Entender qué pasa con los datos cuando **no** hay volumen
- Identificar los riesgos de contraseñas en `environment` y de una red sin aislamiento (se corrigen en otro laboratorio)

---

## Requisitos

- Docker y Docker Compose instalados
- Directorio `flask-docker-app/` del lab anterior

---

## Estructura final

```
flask-docker-app/
├── app.py              ← modificado
├── requirements.txt    ← modificado
├── Dockerfile
├── .dockerignore
├── compose.yaml        ← nuevo
└── init.sql            ← nuevo (opcional)
```

Sin `.env.dev`, sin `pg_password.txt`. `init.sql` es opcional (ver paso 6).

---

## 1. Preparar el proyecto base

```bash
cd flask-docker-app
```

---

## 2. Actualizar dependencias

`requirements.txt`:

```
Flask==2.3.3
gunicorn==21.2.0
psycopg[binary,pool]==3.2.1
```

---

## 3. Crear compose.yaml

```bash
vim compose.yaml
```

```yaml
services:
  flask:
    build: .
    ports:
      - "8080:5000"
    environment:
      - APP_VERSION=2.0.0
      - DB_HOST=postgres
      - DB_DATABASE=mydb
      - DB_USER=myuser
      - DB_PASSWORD=devops123
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16.3
    environment:
      - POSTGRES_USER=myuser
      - POSTGRES_DB=mydb
      - POSTGRES_PASSWORD=devops123
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 5s
      timeout: 5s
      retries: 5
```

**Puntos clave:**

| Configuración | Propósito |
|---|---|
| `build: .` | Compose construye la imagen desde el `Dockerfile` del directorio actual |
| `DB_HOST=postgres` | El nombre del servicio es el hostname dentro de la red de Compose |
| `depends_on` + `condition: service_healthy` | Flask arranca solo cuando Postgres responde a `pg_isready` |
| `POSTGRES_PASSWORD` | Postgres crea el usuario `myuser` con esa contraseña al iniciar |

> **Nota sobre la red:** no definimos `networks`. Compose crea una red por defecto (`flask-docker-app_default`) y conecta ambos servicios. Por eso Flask resuelve `postgres` por nombre.

---

## 3.1 Riesgos de esta configuración

Este `compose.yaml` funciona, pero tiene dos debilidades deliberadas. Conviene entenderlas antes de seguir. **Ambas se corrigen en otro laboratorio** (`laboratorio_docker_compose_01.md`); aquí solo las identificamos.

### Riesgo 1: contraseña en texto plano (`environment`)

`POSTGRES_PASSWORD=devops123` y `DB_PASSWORD=devops123` viven directamente en `compose.yaml`.

| Consecuencia | Cómo comprobarlo |
|---|---|
| La contraseña queda en git si commiteas `compose.yaml` | `git log -p compose.yaml` la mostrará para siempre, aunque la cambies después |
| Cualquier usuario con acceso al daemon la lee | `docker inspect flask-docker-app-postgres-1 --format '{{.Config.Env}}'` |
| Aparece en el entorno del proceso | `docker compose exec postgres env \| grep PASSWORD` |
| Procesos hijos y librerías la heredan | Un log de error que vuelque `os.environ` la filtra |

Compruébalo ahora:

```bash
docker inspect flask-docker-app-postgres-1 --format '{{.Config.Env}}'
```

**Forma correcta (otro laboratorio):** `secrets` montados como archivo en `/run/secrets/` + `POSTGRES_PASSWORD_FILE`, y `env_file` fuera de git. La contraseña no aparece en `docker inspect` ni en `env`.

### Riesgo 2: una sola red sin aislamiento

Sin `networks`, Compose pone ambos servicios en la misma red bridge por defecto. Esa red **sí tiene salida** al host e internet.

| Consecuencia | Por qué importa |
|---|---|
| Postgres es alcanzable desde cualquier contenedor que se una a la red | Un servicio comprometido (o uno que agregues después) llega directo a la DB |
| Si alguien agrega `ports: - "5432:5432"` a `postgres`, la DB queda expuesta en el host | Con red interna, ese mapeo ni siquiera funcionaría |
| Postgres tiene salida a internet | Superficie innecesaria: una base de datos no debería iniciar conexiones externas |
| No hay separación entre "lo que atiende al público" y "lo que es interno" | Todo está a un salto de todo |

Compruébalo ahora:

```bash
docker network inspect flask-docker-app_default --format 'internal={{.Internal}}'
docker compose exec postgres sh -c 'getent hosts google.com && echo "postgres tiene salida a internet"'
```

**Forma correcta (otro laboratorio):** dos redes. Una `public` solo para Flask (recibe el port mapping) y una `private` con `internal: true` para Flask + Postgres. Postgres queda sin ruta al host ni a internet.

### Resumen

| Aspecto | Este laboratorio | Laboratorio siguiente |
|---|---|---|
| Contraseña | `environment` en `compose.yaml` | `secrets` + `env_file` (ignorados por git) |
| Visible en `docker inspect` | Sí | No |
| Red | `default`, con salida externa | `public` + `private` (`internal: true`) |
| Postgres alcanzable desde host | Posible con un `ports` | Imposible |
| Datos tras `down` | Se pierden | Persisten (`volumes`) |

> Para un lab local esto es aceptable. En cualquier entorno compartido o real, no.

---

## 4. Actualizar app.py

```bash
vim app.py
```

```python
from flask import Flask, jsonify, request
import os
from psycopg_pool import ConnectionPool


def db_connect():
    url = (
        f"host={os.environ.get('DB_HOST')} "
        f"dbname={os.environ.get('DB_DATABASE')} "
        f"user={os.environ.get('DB_USER')} "
        f"password={os.environ.get('DB_PASSWORD')}"
    )
    pool = ConnectionPool(url)
    pool.wait()
    return pool


pool = db_connect()

app = Flask(__name__)


@app.route('/')
def home():
    return '<h1>Flask + PostgreSQL</h1><a href="/items">/items</a>'


@app.route('/api/health')
def health():
    return jsonify({"status": "healthy", "version": os.environ.get("APP_VERSION")})


@app.route('/items', methods=['GET', 'POST'])
def items():
    with pool.connection() as conn:
        with conn.cursor() as cur:
            if request.method == 'POST':
                body = request.get_json()
                cur.execute(
                    'INSERT INTO item (priority, task) VALUES (%s, %s)',
                    (body['priority'], body['task'])
                )
                conn.commit()
                return {'message': 'item saved!'}, 201
            cur.execute('SELECT item_id, priority, task FROM item')
            return [{'id': r[0], 'priority': r[1], 'task': r[2]} for r in cur], 200


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

---

## 5. Levantar el stack

```bash
docker compose up --build -d
docker compose ps
```

Ambos servicios deben estar `running`; `postgres` además `healthy`.

```bash
docker compose logs -f
```

---

## 6. Crear la tabla

```bash
docker compose exec postgres psql -U myuser -d mydb
```

```sql
CREATE TABLE item (
  item_id serial PRIMARY KEY,
  priority varchar(256),
  task varchar(256)
);
\q
```

### Alternativa: SQL en archivo (`init.sql`)

Como sin volumen la tabla se pierde tras cada `docker compose down` (ver paso 8), conviene tener el SQL en un archivo para recrearla con un solo comando en vez de abrir `psql` cada vez.

```bash
vim init.sql
```

```sql
CREATE TABLE item (
  item_id serial PRIMARY KEY,
  priority varchar(256),
  task varchar(256)
);
```

Ejecutarlo contra el contenedor:

```bash
docker compose exec -T postgres psql -U myuser -d mydb < init.sql
```

| Flag | Por qué |
|---|---|
| `-T` | Desactiva la pseudo-TTY; necesario para pasar un archivo por `stdin` |
| `< init.sql` | `psql` lee el SQL desde el archivo en lugar de la sesión interactiva |

Resultado esperado: `CREATE TABLE`. Verifica la tabla:

```bash
docker compose exec postgres psql -U myuser -d mydb -c '\dt'
```

---

## 7. Probar los endpoints

```bash
curl localhost:8080/api/health

curl -H "Content-Type: application/json" \
     -d '{"priority":"high","task":"doctor appointment"}' \
     localhost:8080/items

curl -H "Content-Type: application/json" \
     -d '{"priority":"low","task":"buy dinner"}' \
     localhost:8080/items

curl localhost:8080/items
```

Respuesta esperada:

```json
[
  {"id": 1, "priority": "high", "task": "doctor appointment"},
  {"id": 2, "priority": "low",  "task": "buy dinner"}
]
```

> Si `curl` devuelve error justo después de `up -d`, espera 2–3 segundos: gunicorn termina de abrir el pool de conexiones antes de aceptar requests.

---

## 8. ¿Qué pasa con los datos sin volumen?

Baja el stack y levántalo de nuevo:

```bash
docker compose down
docker compose up -d
curl localhost:8080/items
```

Resultado: **error 500**. La tabla `item` ya no existe. Verifícalo:

```bash
docker compose exec postgres psql -U myuser -d mydb -c '\dt'
```

**Causa:** `docker compose down` elimina los contenedores. Sin volumen, el directorio de datos de Postgres (`/var/lib/postgresql/data`) vivía dentro del contenedor y desapareció con él.

Esto es **intencional** en este lab: muestra por qué `laboratorio_docker_compose_01.md` agrega un volumen nombrado. Para seguir trabajando, repite el paso 6 (o el atajo `psql < init.sql`).

> `docker compose stop` / `docker compose start` **sí** conservan los datos, porque no destruyen el contenedor. Solo `down` los pierde.

---

## 9. Comandos útiles

```bash
docker compose up --build -d      # construir y levantar
docker compose ps                 # estado
docker compose logs -f flask      # logs en vivo de un servicio
docker compose exec postgres psql -U myuser -d mydb   # shell SQL
docker compose stop               # pausar (conserva datos)
docker compose start              # reanudar
docker compose down               # destruir contenedores (pierde datos)
```

---

## Checklist de éxito

- [ ] `compose.yaml` con servicios `flask` y `postgres`
- [ ] Stack levantado con `docker compose up --build -d`
- [ ] Ambos contenedores `running`, postgres `healthy`
- [ ] Tabla `item` creada
- [ ] `POST /items` devuelve `201`
- [ ] `GET /items` lista los items
- [ ] Comprobado que los datos se pierden tras `down` + `up`
- [ ] Verificado con `docker inspect` que la contraseña es visible y con `docker network inspect` que la red no es `internal`

---

## Troubleshooting

### `port is already allocated (8080)`

```bash
lsof -i :8080
docker ps -a        # buscar contenedor que usa el puerto
docker rm -f <nombre>
```

O cambia el puerto en `compose.yaml` a `"8081:5000"`.

### `psycopg.OperationalError: connection refused`

Flask arrancó antes de que Postgres estuviera listo. Verifica el `healthcheck` en `compose.yaml` y reinicia:

```bash
docker compose restart flask
```

### `relation "item" does not exist`

La tabla no existe (stack nuevo o tras `down`). Repite el paso 6:

```bash
docker compose exec -T postgres psql -U myuser -d mydb < init.sql
```

---

## Siguiente paso

Continúa con `laboratorio_docker_compose_01.md` para agregar:

| Capa | Qué resuelve |
|---|---|
| `volumes` | Persistencia de datos tras `down` |
| `secrets` + `env_file` | Contraseña fuera de `compose.yaml` y fuera de git |
| `networks` con `internal: true` | Postgres inaccesible desde el host |
