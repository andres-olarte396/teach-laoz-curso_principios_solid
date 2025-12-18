# Ejercicios: Kubernetes Orchestration

## ⭐ Ejercicio 1: Deployment Básico y Escalado

**Objetivo**: Crear un deployment, escalar manualmente y verificar self-healing.

### Contexto

Tienes una aplicación Node.js simple que necesitas desplegar en Kubernetes.

```javascript
// app.js
const express = require("express");
const os = require("os");
const app = express();

app.get("/", (req, res) => {
  res.json({
    message: "Hello from Kubernetes!",
    hostname: os.hostname(),
    version: "1.0.0",
  });
});

app.get("/health", (req, res) => {
  res.json({ status: "healthy" });
});

const PORT = 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

```dockerfile
# Dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY app.js ./
EXPOSE 3000
CMD ["node", "app.js"]
```

### Tareas

**1.1** Crea un Deployment con 2 réplicas.

**1.2** Crea un Service tipo ClusterIP.

**1.3** Escala a 5 réplicas manualmente.

**1.4** Simula un pod fallido eliminándolo manualmente y verifica que Kubernetes lo recrea.

**1.5** Verifica load balancing haciendo múltiples requests al Service.

### Solución

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodeapp-deployment
  labels:
    app: nodeapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodeapp
  template:
    metadata:
      labels:
        app: nodeapp
    spec:
      containers:
        - name: nodeapp
          image: nodeapp:1.0.0
          ports:
            - containerPort: 3000
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"

---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeapp-service
spec:
  type: ClusterIP
  selector:
    app: nodeapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
```

**Comandos**:

```bash
# 1. Aplicar deployment
kubectl apply -f deployment.yaml

# 2. Ver pods creados
kubectl get pods -l app=nodeapp
# NAME                                  READY   STATUS    RESTARTS   AGE
# nodeapp-deployment-7d8f5c9b7f-2kx9p   1/1     Running   0          10s
# nodeapp-deployment-7d8f5c9b7f-7h5nq   1/1     Running   0          10s

# 3. Escalar a 5 réplicas
kubectl scale deployment nodeapp-deployment --replicas=5

# Verificar escalado
kubectl get pods -l app=nodeapp --watch
# Verás 3 pods nuevos en estado ContainerCreating -> Running

# 4. Simular fallo: eliminar un pod
kubectl delete pod nodeapp-deployment-7d8f5c9b7f-2kx9p

# Inmediatamente ver que Kubernetes crea uno nuevo
kubectl get pods -l app=nodeapp
# Verás un pod nuevo con RESTARTS: 0 y AGE: 5s

# 5. Test load balancing desde otro pod
kubectl run test-pod --image=busybox -it --rm -- sh

# Dentro del pod
for i in $(seq 1 10); do
  wget -qO- http://nodeapp-service;
done

# Verás diferentes hostnames en las respuestas
# {"message":"Hello from Kubernetes!","hostname":"nodeapp-deployment-7d8f5c9b7f-2kx9p","version":"1.0.0"}
# {"message":"Hello from Kubernetes!","hostname":"nodeapp-deployment-7d8f5c9b7f-7h5nq","version":"1.0.0"}
# {"message":"Hello from Kubernetes!","hostname":"nodeapp-deployment-7d8f5c9b7f-xm8wz","version":"1.0.0"}
```

**Resultado esperado**:

✅ Deployment con 5 réplicas running  
✅ Self-healing: pod eliminado es reemplazado automáticamente  
✅ Load balancing: requests distribuidas entre todos los pods

---

## ⭐⭐ Ejercicio 2: Rolling Update y Rollback

**Objetivo**: Actualizar una aplicación sin downtime y hacer rollback si algo falla.

### Contexto

Tienes una app en v1.0.0 y quieres actualizar a v2.0.0 con rolling update.

```javascript
// app.js v2.0.0 - nueva feature
const express = require("express");
const os = require("os");
const app = express();

app.get("/", (req, res) => {
  res.json({
    message: "Hello from Kubernetes!",
    hostname: os.hostname(),
    version: "2.0.0", // ← Cambio principal
    newFeature: "Advanced metrics!", // ← Nueva feature
  });
});

app.get("/health", (req, res) => {
  res.json({ status: "healthy" });
});

// Simular bug introducido en v2.0.0
app.get("/api/data", (req, res) => {
  // BUG: crash intencional
  throw new Error("Unhandled exception in v2.0.0!");
});

const PORT = 3000;
app.listen(PORT, () => {
  console.log(`Server v2.0.0 running on port ${PORT}`);
});
```

### Tareas

**2.1** Crea un Deployment con v1.0.0 (3 réplicas) con estrategia RollingUpdate.

**2.2** Configura health checks (liveness y readiness).

**2.3** Actualiza a v2.0.0 y observa el rolling update.

**2.4** Detecta el bug (endpoint `/api/data` crashea) y haz rollback.

**2.5** Verifica que la app volvió a v1.0.0.

### Solución

```yaml
# deployment-v1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1 # Máximo 1 pod down durante update
      maxSurge: 1 # Máximo 1 pod extra temporalmente
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0.0
          ports:
            - containerPort: 3000
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 2
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"

---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 3000
```

**Comandos**:

```bash
# 1. Desplegar v1.0.0
kubectl apply -f deployment-v1.yaml

# Verificar que todo está running
kubectl get pods -l app=myapp
# NAME                    READY   STATUS    RESTARTS   AGE
# myapp-7d8f5c9b7f-2kx9p  1/1     Running   0          30s
# myapp-7d8f5c9b7f-7h5nq  1/1     Running   0          30s
# myapp-7d8f5c9b7f-xm8wz  1/1     Running   0          30s

# 2. Actualizar a v2.0.0
kubectl set image deployment/myapp myapp=myapp:2.0.0

# O editar el YAML y cambiar image:
# image: myapp:2.0.0
# kubectl apply -f deployment-v1.yaml

# 3. Ver rolling update en tiempo real
kubectl rollout status deployment/myapp
# Waiting for deployment "myapp" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "myapp" rollout to finish: 2 out of 3 new replicas have been updated...
# Waiting for deployment "myapp" rollout to finish: 3 new replicas have been updated...
# deployment "myapp" successfully rolled out

# Ver pods durante el update
kubectl get pods -l app=myapp --watch
# Verás pods viejos terminándose y nuevos iniciándose gradualmente

# 4. Verificar versión actualizada
kubectl run test-pod --image=busybox -it --rm -- wget -qO- http://myapp-service
# {"message":"Hello from Kubernetes!","hostname":"myapp-6f9d8c7b8f-abc12","version":"2.0.0","newFeature":"Advanced metrics!"}

# 5. Detectar bug: test del endpoint problemático
kubectl run test-pod --image=busybox -it --rm -- wget -qO- http://myapp-service/api/data
# Error: 500 Internal Server Error

# Ver logs del pod con error
kubectl logs <pod-name>
# Error: Unhandled exception in v2.0.0!

# 6. Rollback inmediato
kubectl rollout undo deployment/myapp

# Verificar rollback en progreso
kubectl rollout status deployment/myapp
# deployment "myapp" successfully rolled out

# 7. Verificar que volvió a v1.0.0
kubectl run test-pod --image=busybox -it --rm -- wget -qO- http://myapp-service
# {"message":"Hello from Kubernetes!","hostname":"myapp-7d8f5c9b7f-xyz89","version":"1.0.0"}

# Ver historial de rollouts
kubectl rollout history deployment/myapp
# REVISION  CHANGE-CAUSE
# 2         kubectl set image deployment/myapp myapp=myapp:2.0.0
# 3         kubectl rollout undo deployment/myapp
```

**Resultado esperado**:

✅ Rolling update ejecutado sin downtime  
✅ Bug detectado en v2.0.0  
✅ Rollback exitoso a v1.0.0  
✅ Zero-downtime durante todo el proceso

---

## ⭐⭐⭐ Ejercicio 3: ConfigMap, Secrets y Multi-Environment

**Objetivo**: Gestionar configuración para múltiples ambientes (dev, staging, prod) con ConfigMaps y Secrets.

### Contexto

Tienes una app que necesita configuración diferente por ambiente:

```typescript
// src/index.ts
import express from "express";
import { createClient } from "redis";

const app = express();

// Configuración desde environment
const config = {
  environment: process.env.APP_ENV || "development",
  port: parseInt(process.env.PORT || "3000"),
  database: {
    host: process.env.DATABASE_HOST || "localhost",
    port: parseInt(process.env.DATABASE_PORT || "5432"),
    name: process.env.DATABASE_NAME || "mydb",
    password: process.env.DATABASE_PASSWORD || "", // ← Secret
  },
  redis: {
    host: process.env.REDIS_HOST || "localhost",
    port: parseInt(process.env.REDIS_PORT || "6379"),
  },
  features: {
    enableCache: process.env.ENABLE_CACHE === "true",
    enableMetrics: process.env.ENABLE_METRICS === "true",
  },
  apiKey: process.env.API_KEY || "", // ← Secret
};

// Redis client
const redisClient = createClient({
  url: `redis://${config.redis.host}:${config.redis.port}`,
});

app.get("/config", (req, res) => {
  res.json({
    environment: config.environment,
    database: {
      host: config.database.host,
      name: config.database.name,
      // No exponer password
    },
    features: config.features,
  });
});

app.get("/health", async (req, res) => {
  try {
    await redisClient.ping();
    res.json({ status: "healthy", redis: "connected" });
  } catch (error) {
    res.status(503).json({ status: "unhealthy", redis: "disconnected" });
  }
});

const PORT = config.port;
app.listen(PORT, async () => {
  await redisClient.connect();
  console.log(`Server running in ${config.environment} mode on port ${PORT}`);
});
```

### Tareas

**3.1** Crea 3 namespaces: `development`, `staging`, `production`.

**3.2** Crea ConfigMaps específicos por ambiente.

**3.3** Crea Secrets para credenciales.

**3.4** Despliega la app en cada ambiente con configuración apropiada.

**3.5** Verifica que cada ambiente tiene configuración diferente.

### Solución

```yaml
# namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: production

---
# configmap-dev.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: development
data:
  APP_ENV: "development"
  DATABASE_HOST: "postgres-dev-service"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "mydb_dev"
  REDIS_HOST: "redis-dev-service"
  REDIS_PORT: "6379"
  ENABLE_CACHE: "false" # Sin cache en dev
  ENABLE_METRICS: "false"

---
# configmap-staging.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: staging
data:
  APP_ENV: "staging"
  DATABASE_HOST: "postgres-staging-service"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "mydb_staging"
  REDIS_HOST: "redis-staging-service"
  REDIS_PORT: "6379"
  ENABLE_CACHE: "true" # Con cache en staging
  ENABLE_METRICS: "true"

---
# configmap-prod.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  APP_ENV: "production"
  DATABASE_HOST: "postgres-prod-service"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "mydb_prod"
  REDIS_HOST: "redis-prod-service"
  REDIS_PORT: "6379"
  ENABLE_CACHE: "true"
  ENABLE_METRICS: "true"

---
# secrets-dev.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: development
type: Opaque
data:
  DATABASE_PASSWORD: ZGV2cGFzc3dvcmQ= # base64: "devpassword"
  API_KEY: ZGV2LWtleS0xMjM= # base64: "dev-key-123"

---
# secrets-staging.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: staging
type: Opaque
data:
  DATABASE_PASSWORD: c3RhZ2luZ3Bhc3N3b3Jk # base64: "stagingpassword"
  API_KEY: c3RhZ2luZy1rZXktNDU2 # base64: "staging-key-456"

---
# secrets-prod.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
data:
  DATABASE_PASSWORD: cHJvZHVjdGlvbnBhc3N3b3Jk # base64: "productionpassword"
  API_KEY: cHJvZC1rZXktNzg5 # base64: "prod-key-789"

---
# deployment-template.yaml (aplicar a cada namespace)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  # namespace se especifica al aplicar: kubectl apply -f ... -n <namespace>
spec:
  replicas: 1 # 1 en dev, 2 en staging, 3 en prod
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0.0
          ports:
            - containerPort: 3000
          # Inyectar ConfigMap y Secret
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 3000
```

**Comandos**:

```bash
# 1. Crear namespaces
kubectl apply -f namespaces.yaml

# 2. Crear ConfigMaps y Secrets por ambiente
kubectl apply -f configmap-dev.yaml
kubectl apply -f configmap-staging.yaml
kubectl apply -f configmap-prod.yaml

kubectl apply -f secrets-dev.yaml
kubectl apply -f secrets-staging.yaml
kubectl apply -f secrets-prod.yaml

# 3. Desplegar app en cada ambiente

# Development: 1 réplica
kubectl apply -f deployment-template.yaml -n development
kubectl scale deployment myapp --replicas=1 -n development

# Staging: 2 réplicas
kubectl apply -f deployment-template.yaml -n staging
kubectl scale deployment myapp --replicas=2 -n staging

# Production: 3 réplicas
kubectl apply -f deployment-template.yaml -n production
kubectl scale deployment myapp --replicas=3 -n production

# 4. Verificar configuración en cada ambiente

# Development
kubectl run test-pod --image=busybox -n development -it --rm -- \
  wget -qO- http://myapp-service/config
# {
#   "environment": "development",
#   "database": {"host": "postgres-dev-service", "name": "mydb_dev"},
#   "features": {"enableCache": false, "enableMetrics": false}
# }

# Staging
kubectl run test-pod --image=busybox -n staging -it --rm -- \
  wget -qO- http://myapp-service/config
# {
#   "environment": "staging",
#   "database": {"host": "postgres-staging-service", "name": "mydb_staging"},
#   "features": {"enableCache": true, "enableMetrics": true}
# }

# Production
kubectl run test-pod --image=busybox -n production -it --rm -- \
  wget -qO- http://myapp-service/config
# {
#   "environment": "production",
#   "database": {"host": "postgres-prod-service", "name": "mydb_prod"},
#   "features": {"enableCache": true, "enableMetrics": true}
# }

# 5. Verificar secrets inyectados
kubectl exec -n development <pod-name> -- env | grep -E "DATABASE_PASSWORD|API_KEY"
# DATABASE_PASSWORD=devpassword
# API_KEY=dev-key-123

kubectl exec -n production <pod-name> -- env | grep -E "DATABASE_PASSWORD|API_KEY"
# DATABASE_PASSWORD=productionpassword
# API_KEY=prod-key-789
```

**Resultado esperado**:

✅ 3 ambientes separados con configuración diferente  
✅ ConfigMaps con settings por ambiente  
✅ Secrets con credenciales por ambiente  
✅ Apps desplegadas con réplicas apropiadas (1 dev, 2 staging, 3 prod)  
✅ Verificación de configuración correcta en cada ambiente

---

## ⭐⭐⭐⭐ Ejercicio 4: Sistema E-Commerce Completo en Kubernetes

**Objetivo**: Desplegar un sistema multi-tier completo con PostgreSQL, Redis, backend API y frontend.

### Arquitectura

```
Internet
    ↓
LoadBalancer
    ↓
Frontend Service (React SPA)
    ↓
Backend Service (Node.js API)
    ↓
┌─────────────┬─────────────┐
│             │             │
PostgreSQL   Redis       RabbitMQ
(orders)     (cache)     (events)
```

### Componentes

**Backend API** (TypeScript):

```typescript
// src/index.ts
import express from "express";
import { Pool } from "pg";
import { createClient } from "redis";

const app = express();
app.use(express.json());

// PostgreSQL connection
const pool = new Pool({
  host: process.env.DATABASE_HOST,
  port: parseInt(process.env.DATABASE_PORT || "5432"),
  database: process.env.DATABASE_NAME,
  user: process.env.DATABASE_USER,
  password: process.env.DATABASE_PASSWORD,
});

// Redis connection
const redisClient = createClient({
  url: `redis://${process.env.REDIS_HOST}:${process.env.REDIS_PORT}`,
});

// Health checks
app.get("/health", async (req, res) => {
  try {
    await pool.query("SELECT 1");
    await redisClient.ping();
    res.json({ status: "healthy", database: "ok", redis: "ok" });
  } catch (error) {
    res.status(503).json({ status: "unhealthy", error: error.message });
  }
});

app.get("/ready", async (req, res) => {
  try {
    await pool.query("SELECT 1");
    await redisClient.ping();
    res.json({ status: "ready" });
  } catch (error) {
    res.status(503).json({ status: "not ready", error: error.message });
  }
});

// Products endpoint
app.get("/api/products", async (req, res) => {
  try {
    // Check cache first
    const cached = await redisClient.get("products");
    if (cached) {
      return res.json(JSON.parse(cached));
    }

    // Query database
    const result = await pool.query("SELECT * FROM products ORDER BY id");
    const products = result.rows;

    // Cache for 5 minutes
    await redisClient.setEx("products", 300, JSON.stringify(products));

    res.json(products);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create order
app.post("/api/orders", async (req, res) => {
  const { productId, quantity } = req.body;

  try {
    const result = await pool.query(
      "INSERT INTO orders (product_id, quantity, created_at) VALUES ($1, $2, NOW()) RETURNING *",
      [productId, quantity]
    );

    // Invalidate cache
    await redisClient.del("products");

    res.status(201).json(result.rows[0]);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

const PORT = parseInt(process.env.PORT || "8080");
app.listen(PORT, async () => {
  await redisClient.connect();
  console.log(`API running on port ${PORT}`);
});
```

### Tareas

**4.1** Crea un namespace `ecommerce`.

**4.2** Despliega PostgreSQL con StatefulSet (tema siguiente, por ahora usa Deployment simple).

**4.3** Despliega Redis.

**4.4** Despliega backend API con ConfigMap y Secrets.

**4.5** Configura health checks (liveness y readiness).

**4.6** Expone todo via LoadBalancer.

**4.7** Verifica funcionamiento end-to-end.

### Solución

```yaml
# ecommerce.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce

---
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: ecommerce-config
  namespace: ecommerce
data:
  DATABASE_HOST: "postgres-service"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "ecommerce_db"
  DATABASE_USER: "postgres"
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6379"

---
# Secrets
apiVersion: v1
kind: Secret
metadata:
  name: ecommerce-secrets
  namespace: ecommerce
type: Opaque
data:
  DATABASE_PASSWORD: cG9zdGdyZXNwYXNzd29yZA== # base64: "postgrespassword"

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
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: ecommerce-config
                  key: DATABASE_NAME
            - name: POSTGRES_USER
              valueFrom:
                configMapKeyRef:
                  name: ecommerce-config
                  key: DATABASE_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ecommerce-secrets
                  key: DATABASE_PASSWORD
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"

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
      targetPort: 5432

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
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"

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
      targetPort: 6379

---
# Backend API Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: ecommerce
spec:
  replicas: 3
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
          image: ecommerce-backend:1.0.0
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: ecommerce-config
            - secretRef:
                name: ecommerce-secrets
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
            initialDelaySeconds: 10
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
# Backend API Service
apiVersion: v1
kind: Service
metadata:
  name: backend-api-service
  namespace: ecommerce
spec:
  type: LoadBalancer
  selector:
    app: backend-api
  ports:
    - port: 80
      targetPort: 8080
```

**Comandos**:

```bash
# 1. Aplicar todo
kubectl apply -f ecommerce.yaml

# 2. Verificar que todos los pods están running
kubectl get pods -n ecommerce
# NAME                          READY   STATUS    RESTARTS   AGE
# postgres-7d8f5c9b7f-abc12     1/1     Running   0          2m
# redis-6f9d8c7b8f-def34        1/1     Running   0          2m
# backend-api-5c8d9f7b6f-ghi56  1/1     Running   0          1m
# backend-api-5c8d9f7b6f-jkl78  1/1     Running   0          1m
# backend-api-5c8d9f7b6f-mno90  1/1     Running   0          1m

# 3. Ver services
kubectl get services -n ecommerce
# NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
# postgres-service      ClusterIP      10.96.123.45    <none>          5432/TCP       2m
# redis-service         ClusterIP      10.96.234.56    <none>          6379/TCP       2m
# backend-api-service   LoadBalancer   10.96.345.67    35.123.45.67    80:31234/TCP   1m

# 4. Inicializar base de datos
kubectl exec -n ecommerce -it <postgres-pod-name> -- psql -U postgres -d ecommerce_db

# Dentro de psql
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  price DECIMAL(10, 2) NOT NULL
);

CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  product_id INTEGER REFERENCES products(id),
  quantity INTEGER NOT NULL,
  created_at TIMESTAMP NOT NULL
);

INSERT INTO products (name, price) VALUES
  ('Laptop', 999.99),
  ('Mouse', 29.99),
  ('Keyboard', 79.99);

\q

# 5. Test API
EXTERNAL_IP=$(kubectl get service backend-api-service -n ecommerce -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Health check
curl http://$EXTERNAL_IP/health
# {"status":"healthy","database":"ok","redis":"ok"}

# Get products (primera vez: desde DB, cachea)
curl http://$EXTERNAL_IP/api/products
# [
#   {"id":1,"name":"Laptop","price":"999.99"},
#   {"id":2,"name":"Mouse","price":"29.99"},
#   {"id":3,"name":"Keyboard","price":"79.99"}
# ]

# Create order
curl -X POST http://$EXTERNAL_IP/api/orders \
  -H "Content-Type: application/json" \
  -d '{"productId": 1, "quantity": 2}'
# {"id":1,"product_id":1,"quantity":2,"created_at":"2024-01-15T10:30:00Z"}

# 6. Verificar cache en Redis
kubectl exec -n ecommerce -it <redis-pod-name> -- redis-cli

# Dentro de redis-cli
KEYS *
# 1) "products"

GET products
# "[{\"id\":1,\"name\":\"Laptop\",\"price\":\"999.99\"}...]"

# 7. Simular fallo de pod y verificar self-healing
kubectl delete pod -n ecommerce <backend-api-pod-name>

# Inmediatamente ver que Kubernetes crea uno nuevo
kubectl get pods -n ecommerce --watch

# API sigue funcionando (load balancer redirige a pods healthy)
curl http://$EXTERNAL_IP/health
# {"status":"healthy","database":"ok","redis":"ok"}
```

**Resultado esperado**:

✅ Sistema e-commerce completo running en Kubernetes  
✅ PostgreSQL con base de datos inicializada  
✅ Redis cacheando products  
✅ Backend API con 3 réplicas  
✅ Health checks funcionales (liveness y readiness)  
✅ Load balancer distribuyendo tráfico  
✅ Self-healing: pods fallidos se recrean automáticamente  
✅ Zero-downtime: API siempre disponible

---

## Resumen de Ejercicios

| Ejercicio | Dificultad | Conceptos                                       |
| --------- | ---------- | ----------------------------------------------- |
| 1         | ⭐         | Deployment, Service, escalado, self-healing     |
| 2         | ⭐⭐       | Rolling update, rollback, health checks         |
| 3         | ⭐⭐⭐     | ConfigMap, Secrets, multi-environment           |
| 4         | ⭐⭐⭐⭐   | Sistema completo, PostgreSQL, Redis, API        |
