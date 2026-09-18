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

- Kubernetes local funcionando: **Docker Desktop** (Kubernetes habilitado en Settings) o **Minikube**.
- `kubectl` instalado y conectado al clúster.
- Docker para construir la imagen.

Verifica antes de empezar:

```bash
kubectl get nodes
# NAME             STATUS   ROLES           AGE   VERSION
# docker-desktop   Ready    control-plane   ...   v1.3x
```

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
├── capturas/
└── README.md
```

---

## Parte 1: Aplicación Flask e imagen (8 min)

### 1.1 Crear la aplicación

```bash
mkdir -p flask-k8s-app/app flask-k8s-app/k8s flask-k8s-app/capturas
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
> **Docker Desktop:** la imagen local ya es visible para el clúster; no hay que hacer nada más.

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
kubectl get endpoints flask-service   # Debe listar las IPs de los 2 Pods
```

### 2.3 Probar el balanceo

```bash
# Docker Desktop: el NodePort responde en localhost
curl http://localhost:30080/

# Minikube: obtén la URL con
# minikube service flask-service --url
```

Lanza varias peticiones seguidas y fíjate en el campo `pod`:

```bash
for i in $(seq 1 6); do curl -s http://localhost:30080/; echo; done
```

Deberías ver que las respuestas alternan entre los 2 nombres de Pod. Ese es el `Service` balanceando el tráfico.

> Si `curl` da `Connection refused`, revisa que los Pods estén `Running` y `READY 1/1`. Si el puerto 30080 está ocupado, cambia `nodePort` por otro dentro del rango 30000-32767.

**Capturas requeridas:**
- `kubectl get pods -o wide` con 2 Pods `Running`.
- Salida del bucle de `curl` mostrando 2 nombres de Pod distintos.

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
kubectl get endpoints flask-service     # Ahora 5 IPs
for i in $(seq 1 10); do curl -s http://localhost:30080/; echo; done
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
kubectl get endpoints flask-service    # Sin endpoints
curl http://localhost:30080/           # Falla: no hay Pods que atiendan

# Restaurar
kubectl scale deployment flask-app --replicas=2
kubectl get pods
```

El `Deployment` y el `Service` siguen existiendo aunque haya 0 réplicas. Esto es útil para "apagar" una aplicación sin borrar su configuración.

**Capturas requeridas:**
- `kubectl get deployment flask-app` con `READY 5/5` tras el escalado.
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
- [ ] `curl` repetido muestra nombres de Pod distintos (balanceo).
- [ ] Escalado a 5 réplicas con `kubectl scale`.
- [ ] Escalado a 3 réplicas editando el YAML y aplicando.
- [ ] Pod eliminado y recreado automáticamente.
- [ ] Recursos eliminados con `kubectl delete -f k8s/`.

---

## Troubleshooting

### Pod en `ImagePullBackOff` o `ErrImagePull`

```bash
kubectl describe pod <pod-name> | grep -A 3 "Events"
```

- **Minikube:** carga la imagen con `minikube image load flask-k8s-app:1.0`.
- Verifica que el nombre y tag en el YAML coinciden con `docker images`.
- Como alternativa cambia `imagePullPolicy: IfNotPresent` por `Never`.

### `curl` a `localhost:30080` no responde

```bash
kubectl get pods                      # ¿Running y READY 1/1?
kubectl get endpoints flask-service   # ¿Hay IPs?
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
