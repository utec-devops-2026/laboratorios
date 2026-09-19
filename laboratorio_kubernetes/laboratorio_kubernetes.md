# Laboratorio: Escalado en Kubernetes con Flask

**Duración estimada:** 40 min  
**Nivel:** Básico  
**Contexto:** Desplegarás una aplicación Flask en Kubernetes con un `Deployment` y un `Service`, y practicarás lo esencial del escalado: subir y bajar réplicas, observar el balanceo de carga entre Pods y comprobar la autorreparación cuando un Pod muere.

> Este laboratorio es la versión reducida. La versión completa (PostgreSQL, Secrets, ConfigMaps y Nginx) está en [`laboratorio_kubernetes_completo.md`](./laboratorio_kubernetes_completo.md).

---

## Objetivos de aprendizaje

- Desplegar una aplicación Flask en Kubernetes con un `Deployment` y exponerla con un `Service`.
- Escalar réplicas de forma imperativa (`kubectl scale`) y declarativa (YAML + `kubectl apply`).
- Observar cómo el `Service` balancea el tráfico entre las réplicas.
- Comprobar la autorreparación (*self-healing*) del `ReplicaSet` al eliminar un Pod.
- (Opcional) Configurar autoescalado con un `HorizontalPodAutoscaler`.

---

## Requisitos

- Kubernetes local funcionando con una de estas opciones:
  - **Docker Desktop**: Settings → Kubernetes → *Enable Kubernetes* → Apply & restart.
  - **OrbStack** (macOS): `orb start k8s` (o en la app: Kubernetes → *Turn On*).
  - **Minikube**: `minikube start`.
- `kubectl` instalado y apuntando al clúster correcto.
- Docker para construir la imagen.

Verifica antes de empezar:

```bash
kubectl config get-contexts        # el contexto activo (*) debe ser docker-desktop, orbstack o minikube
kubectl get nodes
# NAME             STATUS   ROLES           AGE   VERSION
# docker-desktop   Ready    control-plane   ...   v1.3x
```

Si el contexto activo no es el de tu clúster, cámbialo:

```bash
kubectl config use-context orbstack        # o docker-desktop / minikube
```

| Herramienta | Contexto `kubectl` | Imagen local visible en el clúster | NodePort desde tu máquina |
|---|---|---|---|
| Docker Desktop | `docker-desktop` | Sí, sin pasos extra | `localhost:30080` |
| OrbStack | `orbstack` | Sí, sin pasos extra | `localhost:30080` |
| Minikube | `minikube` | No: `minikube image load <imagen>` | `minikube service <svc> --url` |

---

## Estructura del proyecto

```
flask-k8s-app/
├── app/
│   ├── app.py
│   ├── requirements.txt
│   └── Dockerfile
├── k8s/
│   ├── 01-deployment.yaml
│   └── 02-service.yaml
├── scripts/
│   └── balanceo.sh
└── README.md
```

---

## Parte 1: Aplicación Flask e imagen (8 min)

### 1.1 Crear la aplicación

```bash
mkdir -p flask-k8s-app/app flask-k8s-app/k8s flask-k8s-app/scripts
cd flask-k8s-app
```

**Archivo:** `app/requirements.txt`

```txt
Flask==3.0.3
gunicorn==22.0.0
```

**Archivo:** `app/app.py`

La respuesta incluye el `hostname` del contenedor. En Kubernetes el hostname es el **nombre del Pod**, así que nos servirá para ver qué réplica atendió cada petición.

```python
from flask import Flask, jsonify
import os
import socket

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        "message": "Flask en Kubernetes",
        "version": os.getenv("APP_VERSION", "1.0"),
        "pod": socket.gethostname()
    })

@app.route('/health')
def health():
    return jsonify({"status": "healthy"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### 1.2 Crear el Dockerfile

**Archivo:** `app/Dockerfile`

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN adduser --disabled-password --gecos '' appuser
USER appuser

EXPOSE 5000
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

### 1.3 Construir la imagen

```bash
docker build -t flask-k8s-app:1.0 ./app
docker images flask-k8s-app
```

> **Minikube:** el clúster no ve las imágenes locales de Docker. Cárgala con `minikube image load flask-k8s-app:1.0`.  
> **Docker Desktop y OrbStack:** la imagen local ya es visible para el clúster; no hay que hacer nada más.

---

## Parte 2: Deployment y Service (10 min)

### 2.1 Deployment con 2 réplicas

Un **Deployment** declara *cuántas* copias (réplicas) de un Pod deben existir. Kubernetes crea un **ReplicaSet** que se encarga de que siempre haya exactamente ese número corriendo.

**Archivo:** `k8s/01-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  labels:
    app: flask
spec:
  replicas: 2                # Número deseado de Pods
  selector:
    matchLabels:
      app: flask             # Qué Pods gestiona este Deployment
  template:                  # Plantilla del Pod que se replica
    metadata:
      labels:
        app: flask
    spec:
      containers:
        - name: flask
          image: flask-k8s-app:1.0
          imagePullPolicy: IfNotPresent   # Usa la imagen local si existe
          ports:
            - containerPort: 5000
          env:
            - name: APP_VERSION
              value: "1.0"
          readinessProbe:               # Solo recibe tráfico cuando responde /health
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 3
            periodSeconds: 5
          resources:                    # Necesario para el HPA (parte opcional)
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
```

Aplicar y verificar:

```bash
kubectl apply -f k8s/01-deployment.yaml

kubectl get deployment flask-app     # READY debe llegar a 2/2
kubectl get rs                       # El ReplicaSet creado por el Deployment
kubectl get pods -o wide             # 2 Pods con nombre flask-app-<hash>-<id>
```

### 2.2 Service tipo NodePort

Los Pods tienen IPs que cambian cada vez que se recrean. Un **Service** da una IP y un nombre DNS estables y **reparte el tráfico** entre todos los Pods que coinciden con su `selector`.

**Archivo:** `k8s/02-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  type: NodePort             # Expone el Service en un puerto del nodo (30000-32767)
  selector:
    app: flask               # Envía tráfico a los Pods con esta etiqueta
  ports:
    - port: 5000             # Puerto del Service dentro del clúster
      targetPort: 5000       # Puerto del contenedor
      nodePort: 30080        # Puerto expuesto en el nodo
```

Aplicar y verificar:

```bash
kubectl apply -f k8s/02-service.yaml

kubectl get svc flask-service

# Las IPs de los Pods a los que apunta el Service (una por línea)
kubectl get endpointslices -l kubernetes.io/service-name=flask-service \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{"\n"}{end}'
```

Salida esperada con 2 réplicas:

```
192.168.194.4
192.168.194.6
```

> **Por qué no `kubectl get endpoints`:** en Kubernetes 1.33+ el objeto `Endpoints` está deprecado en favor de `EndpointSlice` y el comando imprime un `Warning`. Además, tanto `get endpoints` como `get endpointslices -o wide` **recortan la lista a 3 direcciones** y añaden `+ N more...`, justo cuando más importa verlas todas (al escalar a 5). Por eso el lab usa `jsonpath`, que las imprime todas.

### 2.3 Probar el balanceo

```bash
# Docker Desktop y OrbStack: el NodePort responde en localhost
curl http://localhost:30080/

# Minikube: obtén la URL con
# minikube service flask-service --url
```

Lanza varias peticiones seguidas y fíjate en el campo `pod`:

```bash
for i in $(seq 1 6); do curl -s http://localhost:30080/; echo; done
```

> Cuidado al escribir el bucle: `/\;` (con barra invertida) convierte la URL en `http://localhost:30080/;` y Flask responde `404 Not Found`. El `;` va sin escapar.

Vas a repetir esta prueba varias veces al escalar, así que guárdala como script. Además de mostrar cada respuesta, cuenta cuántas peticiones atendió cada Pod.

**Archivo:** `scripts/balanceo.sh`

```bash
#!/usr/bin/env bash
# Uso: ./scripts/balanceo.sh [peticiones] [url]
# Lanza N peticiones al Service y cuenta cuántas atendió cada Pod.
set -euo pipefail

N="${1:-10}"
URL="${2:-http://localhost:30080/}"

echo "== $N peticiones a $URL =="
pods=""
for i in $(seq 1 "$N"); do
  body=$(curl -s --max-time 3 "$URL") || { echo "peticion $i: sin respuesta"; continue; }
  pod=$(echo "$body" | sed -n 's/.*"pod": *"\([^"]*\)".*/\1/p')
  echo "peticion $i -> ${pod:-respuesta inesperada: $body}"
  [ -n "$pod" ] && pods="$pods$pod"$'\n'
done

echo
echo "== Peticiones por Pod =="
printf '%s' "$pods" | sort | uniq -c | sort -rn
```

```bash
chmod +x scripts/balanceo.sh
./scripts/balanceo.sh          # 10 peticiones a localhost:30080
./scripts/balanceo.sh 20       # 20 peticiones
```

Salida esperada con 2 réplicas (los nombres cambian en tu clúster):

```
== 10 peticiones a http://localhost:30080/ ==
peticion 1 -> flask-app-8bb8cbd8b-8rd4g
peticion 2 -> flask-app-8bb8cbd8b-q7ksm
...
== Peticiones por Pod ==
   6 flask-app-8bb8cbd8b-8rd4g
   4 flask-app-8bb8cbd8b-q7ksm
```

El reparto no es 50/50 exacto: kube-proxy elige un Pod al azar por conexión. Lo importante es que aparezcan **todos** los Pods. Ese es el `Service` balanceando el tráfico.

> **Minikube:** pasa la URL como segundo argumento: `./scripts/balanceo.sh 10 "$(minikube service flask-service --url)/"`.

> Si `curl` da `Connection refused`, revisa que los Pods estén `Running` y `READY 1/1`. Si el puerto 30080 está ocupado, cambia `nodePort` por otro dentro del rango 30000-32767.

**Capturas requeridas:**
- `kubectl get pods -o wide` con 2 Pods `Running`.
- Salida de `./scripts/balanceo.sh` mostrando 2 nombres de Pod distintos.

---

## Parte 3: Escalado (15 min)

### 3.1 Escalar hacia arriba (imperativo)

```bash
kubectl scale deployment flask-app --replicas=5

# En otra terminal, observa cómo aparecen los Pods nuevos (Ctrl+C para salir)
kubectl get pods -w
```

Verifica el resultado:

```bash
kubectl get deployment flask-app        # READY 5/5
kubectl get endpointslices -l kubernetes.io/service-name=flask-service \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{"\n"}{end}'   # Ahora 5 IPs, una por línea
./scripts/balanceo.sh 20                # Deben aparecer los 5 Pods
```

El `Service` agregó automáticamente los Pods nuevos a sus Endpoints. No tuviste que tocar el `Service`.

### 3.2 Escalar hacia abajo (declarativo)

`kubectl scale` es rápido para probar, pero **no queda registrado en tu YAML**. La forma recomendada es cambiar el manifiesto y aplicarlo.

Edita `k8s/01-deployment.yaml`:

```yaml
spec:
  replicas: 3
```

Aplica y observa:

```bash
kubectl apply -f k8s/01-deployment.yaml
kubectl get pods            # 2 Pods pasan a Terminating
kubectl get deployment flask-app
```

> **Importante:** si escalaste con `kubectl scale` y luego haces `kubectl apply` del YAML, el número de réplicas vuelve al valor del archivo. El YAML es la fuente de verdad.

### 3.3 Autorreparación: eliminar un Pod

Elimina uno de los Pods a mano:

```bash
POD=$(kubectl get pods -l app=flask -o jsonpath='{.items[0].metadata.name}')
echo "Eliminando $POD"
kubectl delete pod $POD

kubectl get pods            # Aparece un Pod nuevo con otro nombre
```

El `ReplicaSet` detectó que había 2 Pods en vez de 3 y creó uno nuevo. Kubernetes siempre reconcilia el **estado actual** con el **estado deseado** declarado en `replicas`.

```bash
kubectl describe deployment flask-app | grep -A 5 "Events"
```

### 3.4 Escalar a cero

```bash
kubectl scale deployment flask-app --replicas=0
kubectl get pods                       # Sin Pods
kubectl get endpointslices -l kubernetes.io/service-name=flask-service \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{"\n"}{end}'   # Sin salida: ninguna IP
./scripts/balanceo.sh 3                # "sin respuesta": no hay Pods que atiendan

# Restaurar
kubectl scale deployment flask-app --replicas=2
kubectl get pods
```

El `Deployment` y el `Service` siguen existiendo aunque haya 0 réplicas. Esto es útil para "apagar" una aplicación sin borrar su configuración.

**Capturas requeridas:**
- `kubectl get deployment flask-app` con `READY 5/5` y `endpointslices` con las 5 IPs tras el escalado.
- `kubectl get pods` justo después de eliminar un Pod, mostrando el Pod nuevo (edad de pocos segundos).

---

## Parte 4: Limpieza (2 min)

```bash
kubectl delete -f k8s/
kubectl get all              # Solo debe quedar el service "kubernetes"
```

---

## (Opcional) Parte 5: Autoescalado con HPA

Un **HorizontalPodAutoscaler (HPA)** ajusta `replicas` automáticamente según el uso de CPU o memoria. Requiere el complemento **metrics-server** y que los contenedores tengan `resources.requests` (ya lo tienen).

Instalar metrics-server:

```bash
# Minikube
minikube addons enable metrics-server

# Docker Desktop
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Esperar ~1 min y comprobar que ya hay métricas
kubectl top pods
```

Crear el HPA y generar carga:

```bash
kubectl apply -f k8s/
kubectl autoscale deployment flask-app --cpu-percent=50 --min=2 --max=6
kubectl get hpa -w

# En otra terminal: generar carga contra el Service
kubectl run load --rm -it --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://flask-service:5000/ > /dev/null; done"
```

En 1-2 minutos la columna `REPLICAS` del HPA debería subir. Detén la carga con Ctrl+C y, tras unos minutos, bajará sola al mínimo.

```bash
kubectl delete hpa flask-app
```

---

## Checklist de éxito

- [ ] Imagen `flask-k8s-app:1.0` construida.
- [ ] `Deployment` con 2 réplicas `READY 2/2`.
- [ ] `Service` NodePort respondiendo en el puerto 30080.
- [ ] `scripts/balanceo.sh` muestra nombres de Pod distintos (balanceo).
- [ ] Escalado a 5 réplicas con `kubectl scale`.
- [ ] Escalado a 3 réplicas editando el YAML y aplicando.
- [ ] Pod eliminado y recreado automáticamente.
- [ ] Recursos eliminados con `kubectl delete -f k8s/`.

---

## Troubleshooting

### `kubectl apply` falla con `connect: connection refused`

```
error validating "k8s/01-deployment.yaml": ... Get "https://127.0.0.1:6443/openapi/v2": dial tcp 127.0.0.1:6443: connect: connection refused
```

El clúster está apagado o `kubectl` apunta a un contexto que ya no existe (por ejemplo, un `docker-desktop` viejo cuando ahora usas OrbStack). No es un error del YAML y `--validate=false` no lo arregla.

```bash
kubectl config get-contexts              # ¿cuál está activo (*)?
orb start k8s                            # OrbStack: enciende el clúster (Docker Desktop: Settings → Kubernetes)
kubectl config use-context orbstack      # o docker-desktop / minikube
kubectl get nodes                        # debe responder Ready
```

Opcional: borrar contextos muertos para no volver a caer.

```bash
kubectl config delete-context docker-desktop
kubectl config delete-cluster docker-desktop
```

### Pod en `ImagePullBackOff` o `ErrImagePull`

```bash
kubectl describe pod <pod-name> | grep -A 3 "Events"
```

- **Minikube:** carga la imagen con `minikube image load flask-k8s-app:1.0`.
- **Docker Desktop / OrbStack:** no hace falta cargar nada; verifica que el nombre y tag en el YAML coinciden con `docker images`.
- Como alternativa cambia `imagePullPolicy: IfNotPresent` por `Never`.

### `curl` a `localhost:30080` no responde

```bash
kubectl get pods                      # ¿Running y READY 1/1?
kubectl get endpointslices -l kubernetes.io/service-name=flask-service \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{"\n"}{end}'   # ¿Hay IPs?
kubectl logs deployment/flask-app     # ¿Errores de la app?
```

- En Minikube usa la URL de `minikube service flask-service --url`.

### El Pod no llega a `READY 1/1`

La `readinessProbe` consulta `/health`. Revisa los logs del Pod:

```bash
kubectl logs <pod-name>
```

---

## Recursos adicionales

- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

---

📘 **Autor:**  
Wilson Julca Mejía  
Curso: *DevOps y Kubernetes – Escalado de aplicaciones*  
Universidad de Ingeniería y Tecnología (UTEC)
