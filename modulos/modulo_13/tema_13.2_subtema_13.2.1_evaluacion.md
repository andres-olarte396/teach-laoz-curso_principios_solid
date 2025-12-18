# Evaluación: Kubernetes Orchestration

**Duración**: 3 horas  
**Puntuación total**: 100 puntos

---

## Parte 1: Preguntas Teóricas (20 puntos)

### Pregunta 1 (5 puntos)

Explica la diferencia entre un **Pod** y un **Deployment** en Kubernetes. ¿Por qué generalmente se utilizan Deployments en lugar de Pods directos en producción?

**Respuesta esperada**:

- **Pod**: Unidad básica de ejecución, contiene 1+ containers, efímero (IP cambia al reiniciar)
- **Deployment**: Gestiona réplicas de pods, proporciona rolling updates, rollback, self-healing
- **Razón**: Deployments garantizan disponibilidad, permiten actualizaciones sin downtime, y se recuperan automáticamente de fallos

---

### Pregunta 2 (5 puntos)

¿Cuál es la diferencia entre **liveness probe** y **readiness probe**? Proporciona un ejemplo de cuándo cada uno fallaría.

**Respuesta esperada**:

| Probe      | Propósito                      | Acción al fallar                      | Ejemplo de fallo                       |
| ---------- | ------------------------------ | ------------------------------------- | -------------------------------------- |
| Liveness   | ¿El container está vivo?       | Reinicia el container                 | Deadlock, crash, app colgada           |
| Readiness  | ¿Listo para recibir tráfico?   | Remueve del Service temporalmente     | DB no accesible, dependencia caída     |

---

### Pregunta 3 (5 puntos)

Describe el proceso de un **Rolling Update** con `maxUnavailable: 1` y `maxSurge: 1` en un deployment con 3 réplicas.

**Respuesta esperada**:

```
Estado inicial: 3 pods (v1)
Paso 1: Crear 1 pod nuevo (v2) - total 4 pods (maxSurge=1)
Paso 2: Terminar 1 pod viejo (v1) - total 3 pods
Paso 3: Crear otro pod nuevo (v2) - total 4 pods
Paso 4: Terminar otro pod viejo (v1) - total 3 pods
... repite hasta que todos sean v2
Estado final: 3 pods (v2)
```

**Zero-downtime**: Siempre hay al menos 2 pods running.

---

### Pregunta 4 (5 punos)

¿Cuál es la diferencia entre **ConfigMap** y **Secret** en Kubernetes? ¿Cómo se almacenan los datos en cada uno?

**Respuesta esperada**:

| Recurso    | Uso                          | Almacenamiento                         | Ejemplo                        |
| ---------- | ---------------------------- | -------------------------------------- | ------------------------------ |
| ConfigMap  | Configuración no sensible    | Texto plano                            | DATABASE_HOST, LOG_LEVEL       |
| Secret     | Credenciales                 | Base64 (no es cifrado real)            | PASSWORD, API_KEY              |

**Importante**: Secrets en base64 no son seguros por sí solos. En producción usar **Sealed Secrets** o **External Secrets Operator**.

---

## Parte 2: Análisis de Código (25 puntos)

### Escenario

Tienes el siguiente Deployment que presenta problemas en producción:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 5
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: webapp:latest # ← Problema 1
          ports:
            - containerPort: 3000
          env:
            - name: DATABASE_PASSWORD
              value: "password123" # ← Problema 2
            - name: API_KEY
              value: "sk-abc123xyz" # ← Problema 2
          # No health checks ← Problema 3
          # No resource limits ← Problema 4
```

### Pregunta 2.1 (10 puntos)

Identifica **4 problemas** en el YAML anterior y explica por qué cada uno es problemático en producción.

**Respuesta esperada**:

1. **`image: webapp:latest`**: Tag `latest` es mutable, puede causar inconsistencias. Usar tags específicos (`webapp:1.2.3`)
2. **Secrets en plaintext**: `DATABASE_PASSWORD` y `API_KEY` expuestos. Usar Secret resource
3. **Sin health checks**: No hay liveness/readiness probes. Kubernetes no puede detectar fallos ni evitar enviar tráfico a pods no listos
4. **Sin resource limits**: Sin `resources.requests` y `resources.limits`, un pod puede consumir toda la CPU/memoria del node

---

### Pregunta 2.2 (15 puntos)

Refactoriza el Deployment para solucionar **todos los problemas identificados**. Incluye:

- Secret para credenciales
- Health checks apropiados
- Resource limits
- Tag de imagen específico

**Solución de referencia**:

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: webapp-secrets
type: Opaque
data:
  DATABASE_PASSWORD: cGFzc3dvcmQxMjM= # base64: "password123"
  API_KEY: c2stYWJjMTIzeHl6 # base64: "sk-abc123xyz"

---
# deployment.yaml (corregido)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: webapp:1.2.3 # ✅ Tag específico
          ports:
            - containerPort: 3000
          envFrom:
            - secretRef:
                name: webapp-secrets # ✅ Secret resource
          livenessProbe: # ✅ Health check
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe: # ✅ Readiness check
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2
          resources: # ✅ Resource limits
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

---

## Parte 3: Implementación Práctica (55 puntos)

### Proyecto: Sistema de Mensajería en Tiempo Real

Implementa un sistema de chat en tiempo real con la siguiente arquitectura:

```
Internet
    ↓
LoadBalancer
    ↓
┌────────────────────────────────┐
│  Frontend (React + WebSockets) │
└────────────────────────────────┘
    ↓
┌────────────────────────────────┐
│  Backend API (Node.js + WS)    │
└────────────────────────────────┘
    ↓
┌─────────────┬─────────────┐
│             │             │
PostgreSQL   Redis
(messages)   (sessions)
```

### Requisitos Funcionales

1. **Backend API** debe:

   - Almacenar mensajes en PostgreSQL
   - Cachear sesiones activas en Redis
   - Proporcionar endpoints:
     - `GET /health` - Liveness check
     - `GET /ready` - Readiness check (verifica DB y Redis)
     - `POST /api/messages` - Crear mensaje
     - `GET /api/messages` - Obtener mensajes (cache en Redis)

2. **Configuración**:

   - Usar ConfigMap para configuración no sensible
   - Usar Secret para credenciales de DB

3. **Alta Disponibilidad**:
   - Backend: 3 réplicas
   - PostgreSQL: 1 réplica (en producción real sería StatefulSet)
   - Redis: 1 réplica

---

### Tarea 3.1: Backend API (15 puntos)

Implementa el backend en TypeScript con Express:

```typescript
// src/index.ts
import express from "express";
import { Pool } from "pg";
import { createClient } from "redis";

const app = express();
app.use(express.json());

// PostgreSQL
const pool = new Pool({
  host: process.env.DATABASE_HOST,
  port: parseInt(process.env.DATABASE_PORT || "5432"),
  database: process.env.DATABASE_NAME,
  user: process.env.DATABASE_USER,
  password: process.env.DATABASE_PASSWORD,
});

// Redis
const redisClient = createClient({
  url: `redis://${process.env.REDIS_HOST}:${process.env.REDIS_PORT}`,
});

// Health checks
app.get("/health", (req, res) => {
  res.json({ status: "healthy" });
});

app.get("/ready", async (req, res) => {
  try {
    await pool.query("SELECT 1");
    await redisClient.ping();
    res.json({ status: "ready", database: "ok", redis: "ok" });
  } catch (error) {
    res.status(503).json({ status: "not ready", error: error.message });
  }
});

// Get messages (con cache)
app.get("/api/messages", async (req, res) => {
  try {
    // Check cache
    const cached = await redisClient.get("messages");
    if (cached) {
      return res.json(JSON.parse(cached));
    }

    // Query DB
    const result = await pool.query(
      "SELECT * FROM messages ORDER BY created_at DESC LIMIT 100"
    );

    // Cache for 1 minute
    await redisClient.setEx("messages", 60, JSON.stringify(result.rows));

    res.json(result.rows);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create message
app.post("/api/messages", async (req, res) => {
  const { username, content } = req.body;

  try {
    const result = await pool.query(
      "INSERT INTO messages (username, content, created_at) VALUES ($1, $2, NOW()) RETURNING *",
      [username, content]
    );

    // Invalidate cache
    await redisClient.del("messages");

    res.status(201).json(result.rows[0]);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

const PORT = parseInt(process.env.PORT || "8080");

async function start() {
  await redisClient.connect();
  app.listen(PORT, () => {
    console.log(`Chat API running on port ${PORT}`);
  });
}

start();
```

**Dockerfile**:

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY tsconfig.json ./
COPY src ./src
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist
EXPOSE 8080
CMD ["node", "dist/index.js"]
```

---

### Tarea 3.2: Kubernetes Manifests (25 puntos)

Crea los manifests completos de Kubernetes:

**Solución de referencia**:

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: chat-app

---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: chat-config
  namespace: chat-app
data:
  DATABASE_HOST: "postgres-service"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "chat_db"
  DATABASE_USER: "postgres"
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6379"

---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: chat-secrets
  namespace: chat-app
type: Opaque
data:
  DATABASE_PASSWORD: Y2hhdHBhc3N3b3Jk # base64: "chatpassword"

---
# postgres-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: chat-app
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
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: chat-config
                  key: DATABASE_NAME
            - name: POSTGRES_USER
              valueFrom:
                configMapKeyRef:
                  name: chat-config
                  key: DATABASE_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: chat-secrets
                  key: DATABASE_PASSWORD
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"

---
# postgres-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: chat-app
spec:
  selector:
    app: postgres
  ports:
    - port: 5432

---
# redis-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: chat-app
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
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"

---
# redis-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: chat-app
spec:
  selector:
    app: redis
  ports:
    - port: 6379

---
# backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: chat-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
    spec:
      containers:
        - name: api
          image: chat-backend:1.0.0
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: chat-config
            - secretRef:
                name: chat-secrets
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"

---
# backend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-api-service
  namespace: chat-app
spec:
  type: LoadBalancer
  selector:
    app: backend-api
  ports:
    - port: 80
      targetPort: 8080
```

---

### Tarea 3.3: Deployment y Testing (15 puntos)

**Comandos de deployment**:

```bash
# 1. Crear namespace
kubectl apply -f namespace.yaml

# 2. Aplicar ConfigMap y Secret
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml

# 3. Desplegar PostgreSQL y Redis
kubectl apply -f postgres-deployment.yaml
kubectl apply -f postgres-service.yaml
kubectl apply -f redis-deployment.yaml
kubectl apply -f redis-service.yaml

# Esperar a que estén ready
kubectl wait --for=condition=ready pod -l app=postgres -n chat-app --timeout=60s
kubectl wait --for=condition=ready pod -l app=redis -n chat-app --timeout=60s

# 4. Inicializar base de datos
POSTGRES_POD=$(kubectl get pod -n chat-app -l app=postgres -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n chat-app -it $POSTGRES_POD -- psql -U postgres -d chat_db <<EOF
CREATE TABLE messages (
  id SERIAL PRIMARY KEY,
  username VARCHAR(255) NOT NULL,
  content TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL
);

INSERT INTO messages (username, content, created_at) VALUES
  ('Alice', 'Hello, world!', NOW()),
  ('Bob', 'Hi Alice!', NOW());
EOF

# 5. Desplegar backend
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml

# Esperar a que estén ready (con readiness probe)
kubectl wait --for=condition=ready pod -l app=backend-api -n chat-app --timeout=120s

# 6. Verificar todo
kubectl get all -n chat-app

# 7. Test API
EXTERNAL_IP=$(kubectl get service backend-api-service -n chat-app -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Health check
curl http://$EXTERNAL_IP/health
# {"status":"healthy"}

# Readiness check
curl http://$EXTERNAL_IP/ready
# {"status":"ready","database":"ok","redis":"ok"}

# Get messages (primera vez: desde DB, cachea)
curl http://$EXTERNAL_IP/api/messages
# [{"id":2,"username":"Bob","content":"Hi Alice!","created_at":"2024-01-15T10:30:00Z"},
#  {"id":1,"username":"Alice","content":"Hello, world!","created_at":"2024-01-15T10:29:00Z"}]

# Create message
curl -X POST http://$EXTERNAL_IP/api/messages \
  -H "Content-Type: application/json" \
  -d '{"username":"Charlie","content":"Hey everyone!"}'
# {"id":3,"username":"Charlie","content":"Hey everyone!","created_at":"2024-01-15T10:31:00Z"}

# 8. Verificar cache en Redis
REDIS_POD=$(kubectl get pod -n chat-app -l app=redis -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n chat-app -it $REDIS_POD -- redis-cli KEYS '*'
# 1) "messages"

# 9. Test self-healing: eliminar un pod backend
BACKEND_POD=$(kubectl get pod -n chat-app -l app=backend-api -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod -n chat-app $BACKEND_POD

# Verificar que se crea uno nuevo
kubectl get pods -n chat-app -l app=backend-api --watch

# API sigue funcionando (load balancer usa los otros 2 pods)
curl http://$EXTERNAL_IP/health
# {"status":"healthy"}

# 10. Test rolling update
kubectl set image deployment/backend-api -n chat-app api=chat-backend:2.0.0

# Ver progreso sin downtime
kubectl rollout status deployment/backend-api -n chat-app

# Durante el update, hacer requests continuos
while true; do
  curl -s http://$EXTERNAL_IP/health | jq -r '.status'
  sleep 1
done
# Verás "healthy" continuamente (zero downtime)
```

---

## Parte 4: Bonus (10 puntos)

### Pregunta Bonus 1 (5 puntos)

Implementa **Horizontal Pod Autoscaler (HPA)** para el backend API que escale entre 3 y 10 réplicas basándose en:

- CPU > 70%
- Memory > 80%

**Solución**:

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-api-hpa
  namespace: chat-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

```bash
# Aplicar HPA
kubectl apply -f hpa.yaml

# Ver estado
kubectl get hpa -n chat-app
# NAME              REFERENCE              TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
# backend-api-hpa   Deployment/backend-api 15%/70%, 20%/80%   3         10        3          30s

# Generar carga para test
kubectl run load-generator --image=busybox -n chat-app -it --rm -- \
  sh -c "while true; do wget -q -O- http://backend-api-service/api/messages; done"

# Ver escalado automático
kubectl get hpa -n chat-app --watch
# Verás REPLICAS aumentar cuando CPU/memory suban
```

---

### Pregunta Bonus 2 (5 puntos)

Implementa **Network Policies** para asegurar que:

- Backend API solo puede acceder a PostgreSQL y Redis
- PostgreSQL y Redis no son accesibles desde fuera del namespace

**Solución**:

```yaml
# network-policy.yaml
# Denegar todo el tráfico por defecto
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: chat-app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# Permitir backend -> PostgreSQL
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-postgres
  namespace: chat-app
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend-api
      ports:
        - protocol: TCP
          port: 5432

---
# Permitir backend -> Redis
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-redis
  namespace: chat-app
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend-api
      ports:
        - protocol: TCP
          port: 6379

---
# Permitir tráfico externo -> Backend API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-to-backend
  namespace: chat-app
spec:
  podSelector:
    matchLabels:
      app: backend-api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector: {} # Desde cualquier namespace
      ports:
        - protocol: TCP
          port: 8080
```

---

## Criterios de Evaluación

| Sección                  | Puntos | Criterios                                                        |
| ------------------------ | ------ | ---------------------------------------------------------------- |
| Parte 1: Teoría          | 20     | Comprensión de conceptos de Kubernetes                           |
| Parte 2: Análisis        | 25     | Identificación y solución de problemas en manifests              |
| Parte 3.1: Backend       | 15     | Implementación correcta con health checks                        |
| Parte 3.2: Manifests     | 25     | ConfigMap, Secret, Deployments, Services correctos               |
| Parte 3.3: Testing       | 15     | Deployment exitoso, API funcional, self-healing verificado       |
| Bonus 1: HPA             | 5      | Autoscaling funcional                                            |
| Bonus 2: Network Policy  | 5      | Seguridad de red implementada                                    |
| **Total**                | **110** | **(100 base + 10 bonus)**                                       |

---

## Entrega

1. **Código fuente**: Backend API completo (TypeScript)
2. **Manifests**: Todos los archivos YAML de Kubernetes
3. **Screenshots**: Evidencia de:
   - `kubectl get all -n chat-app`
   - Requests exitosos a la API
   - Self-healing en acción (pod eliminado y recreado)
   - Rolling update sin downtime
4. **Opcional**: Video de 2-3 minutos demostrando el sistema funcionando
