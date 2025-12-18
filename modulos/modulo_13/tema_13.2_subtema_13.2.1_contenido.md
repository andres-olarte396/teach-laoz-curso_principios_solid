# Kubernetes Orchestration: Deployments, Services y Networking

## Introducción

**Kubernetes** (K8s) es el orquestador de contenedores estándar de facto para aplicaciones cloud-native. Mientras Docker permite empaquetar aplicaciones, Kubernetes gestiona su ejecución a escala:

- ✅ **Auto-scaling**: Ajusta réplicas según demanda
- ✅ **Self-healing**: Reinicia containers fallidos
- ✅ **Load balancing**: Distribuye tráfico
- ✅ **Rolling updates**: Deploys sin downtime
- ✅ **Service discovery**: Comunicación entre servicios

---

## Arquitectura de Kubernetes

### Componentes del Control Plane

```
┌─────────────────────────────────────────────────┐
│              CONTROL PLANE                      │
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │   API    │  │  Sched-  │  │ Control  │     │
│  │  Server  │  │  uler    │  │ Manager  │     │
│  └──────────┘  └──────────┘  └──────────┘     │
│                                                 │
│  ┌──────────────────────────────────────┐      │
│  │           etcd (State)               │      │
│  └──────────────────────────────────────┘      │
└─────────────────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
┌───────▼──────┐ ┌────▼──────┐ ┌───▼───────┐
│   Worker 1   │ │ Worker 2  │ │ Worker 3  │
│              │ │           │ │           │
│  ┌────────┐  │ │ ┌────────┐│ │ ┌────────┐│
│  │ kubelet│  │ │ │kubelet ││ │ │kubelet ││
│  └────────┘  │ │ └────────┘│ │ └────────┘│
│              │ │           │ │           │
│  Pods: 5     │ │ Pods: 7   │ │ Pods: 3   │
└──────────────┘ └───────────┘ └───────────┘
```

**Control Plane**:

- **API Server**: Punto de entrada para todas las operaciones (kubectl, deployments)
- **Scheduler**: Decide en qué worker node ejecutar cada pod
- **Controller Manager**: Mantiene el estado deseado (réplicas, health checks)
- **etcd**: Base de datos distribuida que almacena todo el estado del cluster

**Worker Nodes**:

- **kubelet**: Agente que ejecuta en cada node, gestiona los pods
- **kube-proxy**: Maneja networking y load balancing
- **Container Runtime**: Docker, containerd, o CRI-O

---

## Pods: La Unidad Básica

Un **Pod** es el objeto más pequeño en Kubernetes. Encapsula uno o más contenedores que comparten:

- Red (misma IP, localhost entre containers)
- Almacenamiento (volumes compartidos)
- Ciclo de vida (se crean/destruyen juntos)

### Pod Simple

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
```

**Aplicar**:

```bash
kubectl apply -f nginx-pod.yaml

# Ver pods
kubectl get pods
# NAME        READY   STATUS    RESTARTS   AGE
# nginx-pod   1/1     Running   0          10s

# Ver detalles
kubectl describe pod nginx-pod

# Logs
kubectl logs nginx-pod

# Ejecutar comando en el pod
kubectl exec -it nginx-pod -- bash
```

### Multi-Container Pod (Sidecar Pattern)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-logging
spec:
  containers:
    # Contenedor principal
    - name: app
      image: myapp:1.0.0
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: logs
          mountPath: /var/log/app

    # Sidecar: Log collector
    - name: log-collector
      image: fluentd:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log/app

  volumes:
    - name: logs
      emptyDir: {}
```

**Patrón Sidecar**: El container `log-collector` lee los logs que escribe `app` en el volume compartido y los envía a un sistema centralizado (Elasticsearch, Loki).

---

## Deployments: Gestión Declarativa de Pods

Los **Pods** directos son efímeros. Los **Deployments** proporcionan:

- ✅ Gestión de réplicas
- ✅ Rolling updates
- ✅ Rollbacks
- ✅ Self-healing

### Deployment Básico

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3 # Número de pods
  selector:
    matchLabels:
      app: nginx # Selector para identificar pods
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
```

**Aplicar**:

```bash
kubectl apply -f nginx-deployment.yaml

# Ver deployments
kubectl get deployments
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-deployment   3/3     3            3           30s

# Ver pods creados por el deployment
kubectl get pods -l app=nginx
# NAME                               READY   STATUS    RESTARTS   AGE
# nginx-deployment-7d8f5c9b7f-2kx9p  1/1     Running   0          30s
# nginx-deployment-7d8f5c9b7f-7h5nq  1/1     Running   0          30s
# nginx-deployment-7d8f5c9b7f-xm8wz  1/1     Running   0          30s
```

### Escalado

```bash
# Escalar manualmente
kubectl scale deployment nginx-deployment --replicas=5

# Ver el escalado en tiempo real
kubectl get pods -l app=nginx --watch
```

### Rolling Update

```yaml
# Actualizar imagen
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1 # Máximo 1 pod down durante update
      maxSurge: 1 # Máximo 1 pod extra durante update
  template:
    spec:
      containers:
        - name: nginx
          image: nginx:1.26 # Nueva versión
```

**Proceso de Rolling Update**:

```
Estado inicial: 3 pods con nginx:1.25
├─ Pod 1 (v1.25)
├─ Pod 2 (v1.25)
└─ Pod 3 (v1.25)

Paso 1: Crear 1 nuevo pod con v1.26 (maxSurge=1)
├─ Pod 1 (v1.25)
├─ Pod 2 (v1.25)
├─ Pod 3 (v1.25)
└─ Pod 4 (v1.26) ← nuevo

Paso 2: Terminar 1 pod viejo (maxUnavailable=1)
├─ Pod 2 (v1.25)
├─ Pod 3 (v1.25)
└─ Pod 4 (v1.26)

Paso 3: Crear otro pod nuevo
├─ Pod 2 (v1.25)
├─ Pod 3 (v1.25)
├─ Pod 4 (v1.26)
└─ Pod 5 (v1.26) ← nuevo

Paso 4: Terminar otro pod viejo
├─ Pod 3 (v1.25)
├─ Pod 4 (v1.26)
└─ Pod 5 (v1.26)

... continúa hasta terminar

Estado final: 3 pods con nginx:1.26
├─ Pod 4 (v1.26)
├─ Pod 5 (v1.26)
└─ Pod 6 (v1.26)
```

**Zero-downtime**: Siempre hay al menos 2 pods running durante el update.

```bash
# Aplicar update
kubectl apply -f nginx-deployment.yaml

# Ver progreso
kubectl rollout status deployment nginx-deployment

# Historial de rollouts
kubectl rollout history deployment nginx-deployment
```

### Rollback

```bash
# Si algo sale mal, rollback a versión anterior
kubectl rollout undo deployment nginx-deployment

# Rollback a versión específica
kubectl rollout undo deployment nginx-deployment --to-revision=2
```

---

## Services: Networking y Load Balancing

Los **Pods** son efímeros (IP cambia al reiniciar). Los **Services** proporcionan:

- ✅ IP estable
- ✅ DNS name
- ✅ Load balancing entre pods

### ClusterIP (Interno)

**Uso**: Comunicación entre servicios dentro del cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP # Default
  selector:
    app: nginx # Selecciona pods con label app=nginx
  ports:
    - protocol: TCP
      port: 80 # Puerto del Service
      targetPort: 80 # Puerto del container
```

**Funcionamiento**:

```
Otros pods pueden acceder via:
- http://nginx-service:80 (DNS name)
- http://nginx-service.default.svc.cluster.local:80 (FQDN)

Kubernetes load balancea automáticamente entre los 3 pods:
nginx-service:80 →  ┌─ Pod 1 (10.0.1.5:80)
                    ├─ Pod 2 (10.0.1.6:80)
                    └─ Pod 3 (10.0.1.7:80)
```

```bash
kubectl apply -f nginx-service.yaml

# Ver services
kubectl get services
# NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# nginx-service   ClusterIP   10.96.123.45   <none>        80/TCP    10s

# Test desde otro pod
kubectl run test-pod --image=busybox -it --rm -- wget -O- http://nginx-service:80
```

### NodePort (Externo - Desarrollo)

**Uso**: Acceso desde fuera del cluster en puerto específico.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080 # Puerto en cada node (30000-32767)
```

**Acceso**:

```bash
# Acceder via cualquier node del cluster
curl http://<node-ip>:30080

# En Minikube
minikube service nginx-nodeport --url
# http://192.168.49.2:30080
```

### LoadBalancer (Externo - Producción)

**Uso**: Provisiona un load balancer cloud (AWS ELB, Azure LB, GCP LB).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

```bash
kubectl get services nginx-lb
# NAME       TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
# nginx-lb   LoadBalancer   10.96.45.67   35.123.45.67     80:31234/TCP   2m

# Acceder via external IP
curl http://35.123.45.67
```

**Funcionamiento en Cloud**:

```
Internet
    ↓
AWS Load Balancer (35.123.45.67)
    ↓
Kubernetes Service (nginx-lb)
    ↓
┌───────────────────────────┐
│  Pods (load balanced)     │
├─ nginx-pod-1 (10.0.1.5)   │
├─ nginx-pod-2 (10.0.1.6)   │
└─ nginx-pod-3 (10.0.1.7)   │
└───────────────────────────┘
```

---

## ConfigMaps y Secrets

### ConfigMap: Configuración no sensible

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres-service"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"
  APP_ENV: "production"
```

**Uso en Deployment**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
          envFrom:
            - configMapRef:
                name: app-config
          # O variables individuales:
          env:
            - name: DATABASE_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DATABASE_HOST
```

### Secret: Credenciales

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DATABASE_PASSWORD: cGFzc3dvcmQxMjM= # base64: "password123"
  API_KEY: c2stYWJjMTIzeHl6 # base64: "sk-abc123xyz"
```

**Crear secret desde línea de comandos**:

```bash
# Desde literales
kubectl create secret generic app-secrets \
  --from-literal=DATABASE_PASSWORD=password123 \
  --from-literal=API_KEY=sk-abc123xyz

# Desde archivo
kubectl create secret generic db-credentials \
  --from-file=username.txt \
  --from-file=password.txt
```

**Uso en Deployment**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
          envFrom:
            - secretRef:
                name: app-secrets
          # O como volume (para archivos de config)
          volumeMounts:
            - name: secret-volume
              mountPath: /etc/secrets
              readOnly: true
      volumes:
        - name: secret-volume
          secret:
            secretName: app-secrets
```

**Resultado**: Secrets montados como archivos en `/etc/secrets/`:

```bash
/etc/secrets/
├── DATABASE_PASSWORD
└── API_KEY
```

---

## Health Checks: Liveness y Readiness Probes

### Liveness Probe

**Propósito**: ¿El container está vivo? Si falla, Kubernetes lo reinicia.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30 # Esperar 30s antes del primer check
            periodSeconds: 10 # Check cada 10s
            timeoutSeconds: 5 # Timeout de 5s
            failureThreshold: 3 # Reiniciar después de 3 fallos consecutivos
```

**Ejemplo de endpoint**:

```typescript
// /health endpoint en la app
app.get("/health", (req, res) => {
  // Check básico: ¿la app responde?
  res.status(200).json({ status: "healthy" });
});
```

### Readiness Probe

**Propósito**: ¿El container está listo para recibir tráfico? Si falla, se remueve del Service temporalmente.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2
```

**Ejemplo de endpoint**:

```typescript
// /ready endpoint - checks más profundos
app.get("/ready", async (req, res) => {
  try {
    // Verificar que DB está accesible
    await database.ping();

    // Verificar que Redis está accesible
    await redis.ping();

    res.status(200).json({ status: "ready" });
  } catch (error) {
    // No estoy listo (DB no responde, etc.)
    res.status(503).json({ status: "not ready", error: error.message });
  }
});
```

**Diferencia clave**:

| Probe      | Falla                            | Acción                        |
| ---------- | -------------------------------- | ----------------------------- |
| Liveness   | App crashed, deadlock            | Reinicia el container         |
| Readiness  | DB down, dependencias no listas  | Remueve del Service, no mata  |
| Startup    | App tarda en iniciar (legacy)    | Espera más antes de liveness  |

### Startup Probe (para apps lentas)

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 30 # 30 * 10s = 5 minutos para iniciar
```

**Secuencia**:

```
1. Pod inicia
2. Startup probe checks cada 10s hasta que pase
3. Solo después, liveness y readiness probes comienzan
```

---

## Namespaces: Organización Multi-Tenant

**Namespaces** separan recursos lógicamente en el mismo cluster.

### Crear Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
```

```bash
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production
```

### Deployment en Namespace

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: development # Especificar namespace
spec:
  replicas: 2
  # ...
```

```bash
# Aplicar en namespace
kubectl apply -f deployment.yaml -n development

# Ver recursos en namespace
kubectl get pods -n development
kubectl get services -n development

# Ver todos los namespaces
kubectl get namespaces
```

### DNS entre Namespaces

```
Service en mismo namespace:
http://nginx-service:80

Service en diferente namespace:
http://nginx-service.production.svc.cluster.local:80

Formato completo:
<service>.<namespace>.svc.cluster.local
```

---

## Ejemplo Completo: Aplicación Multi-Tier

```yaml
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce

---
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: ecommerce
data:
  DATABASE_HOST: "postgres-service"
  REDIS_HOST: "redis-service"

---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: ecommerce
type: Opaque
data:
  DATABASE_PASSWORD: cGFzc3dvcmQxMjM=

---
# PostgreSQL Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: ecommerce
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DATABASE_PASSWORD
          ports:
            - containerPort: 5432

---
# PostgreSQL Service
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: ecommerce
spec:
  selector:
    app: postgres
  ports:
    - port: 5432

---
# Redis Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: ecommerce
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379

---
# Redis Service
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: ecommerce
spec:
  selector:
    app: redis
  ports:
    - port: 6379

---
# App Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecommerce-app
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ecommerce
  template:
    metadata:
      labels:
        app: ecommerce
    spec:
      containers:
        - name: app
          image: ecommerce-app:1.0.0
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"

---
# App Service
apiVersion: v1
kind: Service
metadata:
  name: ecommerce-service
  namespace: ecommerce
spec:
  type: LoadBalancer
  selector:
    app: ecommerce
  ports:
    - port: 80
      targetPort: 8080
```

**Deploy completo**:

```bash
kubectl apply -f ecommerce.yaml

# Verificar todo
kubectl get all -n ecommerce
# Verás: deployments, pods, services
```

---

## Resumen

| Recurso        | Propósito                               | Ejemplo                       |
| -------------- | --------------------------------------- | ----------------------------- |
| **Pod**        | Unidad básica (1+ containers)           | nginx-pod                     |
| **Deployment** | Gestión de réplicas y updates           | nginx-deployment (3 réplicas) |
| **Service**    | Networking y load balancing             | ClusterIP, LoadBalancer       |
| **ConfigMap**  | Configuración no sensible               | DATABASE_HOST, LOG_LEVEL      |
| **Secret**     | Credenciales                            | PASSWORD, API_KEY             |
| **Namespace**  | Separación lógica                       | dev, staging, prod            |
| **Probes**     | Health checks (liveness, readiness)     | /health, /ready endpoints     |

**Próximo tema**: Veremos Ingress, StatefulSets, PersistentVolumes y patrones avanzados de Kubernetes.
