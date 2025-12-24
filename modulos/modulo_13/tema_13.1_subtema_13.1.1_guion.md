# Guion de Video: 12-Factor App y Containerización

**Duración total**: 35 minutos  
**Objetivo**: Comprender los 12 principios de aplicaciones cloud-native y cómo implementarlos con Docker

---

## [00:00 - 01:30] Introducción

**[Pantalla: Título "12-Factor App y Containerización"]**

Hola, bienvenidos. Hoy vamos a explorar los fundamentos de las aplicaciones cloud-native: los **12-Factor App principles** y la **containerización con Docker**.

**[Visual: Transición de monolito a cloud-native]**

Cuando hablamos de "cloud-native", no nos referimos simplemente a "ejecutar en la nube". Se trata de un conjunto de prácticas arquitecturales que garantizan:

- ✅ Escalabilidad elástica
- ✅ Resiliencia ante fallos
- ✅ Despliegue continuo
- ✅ Portabilidad

**[Visual: Logo 12-Factor + Docker]**

Los **12-Factor App principles**, definidos por Heroku en 2011, son la base fundamental. Docker y Kubernetes son las tecnologías que materializan estos principios.

Vamos a ver cada factor en detalle y cómo aplicarlo con ejemplos prácticos.

---

## [01:30 - 04:00] Factor 1-2: Codebase y Dependencies

**[Pantalla: Factor 1 - Codebase]**

### Factor 1. Un repositorio, múltiples deploys

**[Visual: Diagrama de un repo con múltiples branches]**

```
✅ BIEN: Un repo, diferentes environments
Repo: myapp
├── main → production
├── staging → staging
└── develop → development
```

El mismo código en diferentes ambientes, configurado de forma distinta. **NO** repositorios separados para cada environment.

**[Pantalla: Factor 2 - Dependencies]**

### Factor 2: Declarar y aislar dependencias

**[Mostrar código: package.json]**

```json
{
  "name": "my-app",
  "engines": {
    "node": ">=18.0.0"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.0"
  }
}
```

**[Resaltar]** Todo debe estar declarado explícitamente. Nada de "requiere Python 3.9 instalado en tu máquina" sin especificarlo en código.

**[Transición a Docker]**

Y aquí es donde Docker brilla:

```dockerfile
FROM python:3.11-slim
COPY requirements.txt .
RUN pip install -r requirements.txt
```

**Aislamiento completo**: las dependencias viven dentro del container, no en tu máquina.

---

## [04:00 - 08:00] Factor 3: Config (El más importante)

**[Pantalla: Factor 3 - Config]**

Este es probablemente el factor **más violado** en aplicaciones legacy.

**[Mostrar código: ❌ Anti-pattern]**

```javascript
// ❌ MAL: Config hardcoded
const config = {
    database: 'prod-db.example.com',
    password: 'super-secret-123'  // 🚨 PELIGRO
};
```

**[Resaltar en rojo]** Dos problemas graves:

1. **Password en código** → comprometido si alguien ve el repo
2. **Misma config para todos** → no puedes usar esto en desarrollo

**[Transición: ✅ Solución]**

**[Mostrar código: Correcto]**

```typescript
function loadConfig(): AppConfig {
    return {
        database: {
            host: requireEnv('DATABASE_HOST'),
            password: requireEnv('DATABASE_PASSWORD')
        }
    };
}
```

**[Visual: Environment variables fluyendo al container]**

```bash
# .env.production
DATABASE_HOST=prod-db.example.com
DATABASE_PASSWORD=<from-secrets-manager>

# .env.development  
DATABASE_HOST=localhost
DATABASE_PASSWORD=dev-password
```

**Mismo código, diferente configuración** según el environment.

**[Demo: Kubernetes ConfigMap + Secret]**

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
data:
  DATABASE_HOST: "postgres-service"

# secret.yaml
apiVersion: v1
kind: Secret
data:
  DATABASE_PASSWORD: cGFzc3dvcmQxMjM=  # base64
```

**[Resaltar]** Kubernetes inyecta estas variables al container automáticamente.

---

## [08:00 - 12:00] Factor 6: Processes (Stateless)

**[Pantalla: Factor 6 - Processes]**

**Principio crítico**: Tu aplicación debe ser **stateless**. Todo estado va en backing services.

**[Diagrama: Problema con estado en memoria]**

Imagina este código:

```javascript
let sessionData = {};  // ❌ En memoria

app.post('/login', (req, res) => {
    sessionData[sessionId] = { userId: '123' };
});
```

**[Animación: 3 pods en Kubernetes]**

```
Usuario login → Pod 1 (crea sesión en memoria)
Usuario request → Pod 2 (no tiene la sesión) ❌
```

**[Resaltar en rojo]** Con 3 réplicas en Kubernetes, cada pod tiene su propia memoria. El load balancer envía requests a diferentes pods → **sesión se pierde**.

**[Transición: ✅ Solución con Redis]**

```typescript
// ✅ Session en Redis (compartido)
await redis.setex(`session:${sessionId}`, 3600, JSON.stringify(data));
```

**[Diagrama: Redis como estado compartido]**

```
Usuario login → Pod 1 → Redis (guarda sesión)
Usuario request → Pod 2 → Redis (lee sesión) ✅
```

**[Resaltar en verde]** Todos los pods acceden al mismo Redis → estado consistente.

**[Mostrar segundo ejemplo: File uploads]**

```python
# ❌ MAL: Local filesystem
file.save(f'/tmp/uploads/{filename}')

# ✅ BIEN: S3 storage
s3_client.upload_file(bucket='my-bucket', key=filename)
```

**[Visual: Container filesystem es efímero]**

Cuando un pod se reinicia, el filesystem local **desaparece**. S3 persiste.

---

## [12:00 - 15:00] Factor 9: Disposability (Graceful Shutdown)

**[Pantalla: Factor 9 - Disposability]**

Tu app debe:

1. **Iniciar rápido** (segundos, no minutos)
2. **Terminar gracefully** (sin cortar requests)

**[Diagrama: Kubernetes terminando un pod]**

```
1. Kubernetes envía SIGTERM al pod
2. App tiene 30s para terminar
3. Si no termina, SIGKILL (forzado)
```

**[Mostrar código: Sin graceful shutdown]**

```javascript
// ❌ MAL: No maneja SIGTERM
app.listen(3000);
// Kubernetes envía SIGTERM → app ignora → SIGKILL → requests cortados
```

**[Transición: ✅ Correcto]**

```typescript
const server = app.listen(3000);

process.on('SIGTERM', async () => {
    console.log('SIGTERM received, shutting down...');
    
    // 1. Stop accepting new requests
    server.close();
    
    // 2. Finish ongoing requests (with timeout)
    await Promise.race([
        waitForOngoingRequests(),
        timeout(30000)
    ]);
    
    // 3. Close DB connections
    await database.close();
    
    process.exit(0);
});
```

**[Animación: Timeline de shutdown]**

```
t=0s:  SIGTERM received
t=1s:  Server stops accepting new requests
t=5s:  Ongoing requests complete
t=6s:  Database connection closed
t=6s:  Process exits cleanly ✅
```

**[Resaltar]** Sin pérdida de requests, sin errores para usuarios.

---

## [15:00 - 18:00] Factor 11. Logs (Structured Logging)

**[Pantalla: Factor 11 - Logs]**

**Principio**: Logs son **event streams**, escribir a **stdout/stderr**.

**[Mostrar código: ❌ Anti-pattern]**

```typescript
// ❌ MAL: Log a archivo
fs.appendFileSync('/var/log/app.log', message);
```

**[Diagrama: Problemas en Kubernetes]**

```
Pod 1 → /var/log/app.log
Pod 2 → /var/log/app.log
Pod 3 → /var/log/app.log

¿Cómo ves todos los logs? ❌
Pod se reinicia → logs desaparecen ❌
```

**[Transición: ✅ Solución]**

```typescript
// ✅ BIEN: Structured logging a stdout
console.log(JSON.stringify({
    timestamp: new Date().toISOString(),
    level: 'info',
    message: 'Order created',
    orderId: '123',
    userId: 'u456'
}));
```

**[Visual: Flujo de logs en Kubernetes]**

```
Pods → stdout → Kubernetes → Log Aggregator (ELK/Loki)
                                         ↓
                               Dashboard centralizado ✅
```

**[Demo: Query logs]**

```bash
# Kubernetes logs (todos los pods)
kubectl logs deployment/myapp --tail=100

# Loki query
{app="myapp"} |= "error" | json | level="error"
```

**[Resaltar]** Logs centralizados, buscables, persistentes.

---

## [18:00 - 22:00] Factor 10: Dev/Prod Parity

**[Pantalla: Factor 10 - Dev/Prod Parity]**

**Principio**: Desarrollo y producción deben ser **lo más similares posible**.

**[Diagrama: ❌ Anti-pattern]**

```
Development:               Production:
- SQLite                   - PostgreSQL
- In-memory cache          - Redis
- Mock email               - SendGrid
- Windows                  - Linux
```

**[Resaltar en rojo]** Diferentes bases de datos → comportamiento diferente → bugs en producción.

**[Transición: ✅ Docker Compose]**

**[Mostrar código: docker-compose.yml]**

```yaml
services:
  app:
    build: .
    environment:
      DATABASE_URL: postgres://postgres:5432/myapp
      REDIS_URL: redis://redis:6379
  
  postgres:
    image: postgres:15  # Misma versión que prod
  
  redis:
    image: redis:7      # Misma versión que prod
```

**[Terminal: Ejecutar]**

```bash
docker-compose up
```

**[Animación: Levantando stack completo]**

```
Starting postgres... done
Starting redis... done
Starting app... done

✅ Developer tiene exactamente lo mismo que producción
```

**[Resaltar]** Un comando, stack completo, dev/prod identical.

---

## [22:00 - 27:00] Docker: Multi-Stage Build

**[Pantalla: Dockerfile Optimizado]**

Ahora veamos cómo Docker implementa varios de estos factores.

**[Mostrar código: Dockerfile multi-stage]**

```dockerfile
# STAGE 1. Build
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# STAGE 2: Runtime
FROM node:18-alpine
RUN adduser -S nodejs -u 1001
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER nodejs
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

**[Diagrama: Comparación de tamaños]**

```
Builder stage:  850 MB (Node + build tools)
Runtime stage:  180 MB (Node + app) ✅

Reducción: 79%
```

**[Resaltar 3 optimizaciones clave]**

1. **Layer caching**:

```dockerfile
# ✅ Dependencies primero
COPY package*.json ./
RUN npm ci
# Cache se reutiliza si package.json no cambia

# Código después
COPY . .
```

1. **Non-root user** (security):

```dockerfile
RUN adduser -S nodejs -u 1001
USER nodejs  # No corre como root
```

1. **Alpine base** (imagen pequeña):

```dockerfile
FROM node:18-alpine  # 40MB vs node:18 (900MB)
```

**[Demo: Build]**

```bash
docker build -t myapp:1.0.0 .

# Ver tamaño
docker images
# myapp  1.0.0  180MB ✅
```

---

## [27:00 - 31:00] Demo: Sistema Completo

**[Pantalla: Arquitectura de ejemplo]**

Veamos cómo todo esto se une en un sistema real.

**[Diagrama: E-commerce microservices]**

```
API Gateway
    ├── Product Service
    ├── Order Service
    └── Payment Service

Backing Services:
- PostgreSQL (data)
- Redis (cache/sessions)
- RabbitMQ (events)
```

**[Terminal: Levantar con Docker Compose]**

```bash
docker-compose up -d

# Ver servicios
docker-compose ps
```

**[Mostrar output]**

```
NAME                STATUS         PORTS
product-service     Up             0.0.0.0:3001->3001
order-service       Up             0.0.0.0:3002->3002
payment-service     Up             0.0.0.0:3003->3003
postgres            Up (healthy)   5432
redis               Up             6379
rabbitmq            Up (healthy)   5672, 15672
```

**[Resaltar]** Todo el stack corriendo localmente, idéntico a producción.

**[Terminal: Test]**

```bash
# Crear producto
curl -X POST http://localhost:3001/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Laptop","price":999.99}'

# Crear orden
curl -X POST http://localhost:3002/orders \
  -d '{"userId":"u123","items":[{"productId":"p1","qty":1}]}'
```

**[Mostrar logs estructurados]**

```bash
docker-compose logs -f order-service
```

**[Output en JSON]**

```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "level": "info",
  "message": "Order created",
  "orderId": "o123",
  "userId": "u123"
}
```

**[Resaltar]** Logs estructurados (Factor 11) ✅

**[Terminal: Graceful shutdown test]**

```bash
# Enviar SIGTERM
docker-compose stop order-service
```

**[Mostrar logs de shutdown]**

```
SIGTERM received, shutting down gracefully...
Closing database connections...
Closing RabbitMQ connection...
Shutdown complete
```

**[Resaltar]** Graceful shutdown (Factor 9) ✅

---

## [31:00 - 34:00] Kubernetes: Deploy a Producción

**[Pantalla: De Docker a Kubernetes]**

Ahora desplegamos a Kubernetes en producción.

**[Mostrar código: deployment.yaml]**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3  # Factor 8: Concurrency
  template:
    spec:
      containers:
      - name: app
        image: order-service:1.0.0
        envFrom:
        - configMapRef:
            name: app-config     # Factor 3: Config
        - secretRef:
            name: app-secrets    # Factor 3: Secrets
        livenessProbe:
          httpGet:
            path: /health
            port: 3002
        readinessProbe:
          httpGet:
            path: /health
            port: 3002
      terminationGracePeriodSeconds: 30  # Factor 9
```

**[Terminal: Deploy]**

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml
```

**[Mostrar output]**

```
deployment.apps/order-service created
service/order-service created
horizontalpodautoscaler.autoscaling/order-service-hpa created
```

**[Terminal: Ver pods]**

```bash
kubectl get pods
```

```
NAME                             READY   STATUS
order-service-6d4f8c9b7f-2kx9p   1/1     Running
order-service-6d4f8c9b7f-7h5nq   1/1     Running
order-service-6d4f8c9b7f-xm8wz   1/1     Running
```

**[Resaltar]** 3 réplicas (Factor 8: Concurrency) ✅

**[Diagrama: HPA en acción]**

```
CPU > 70% → Kubernetes añade pods automáticamente
3 pods → 5 pods → 8 pods

CPU < 70% → Reduce pods
8 pods → 5 pods → 3 pods
```

**[Resaltar]** Escalado automático basado en demanda.

---

## [34:00 - 35:00] Resumen y Próximos Pasos

**[Pantalla: Checklist de 12-Factor]**

Hemos cubierto:

✅ **Factor 1-2**: Codebase único, dependencias declaradas  
✅ **Factor 3**: Config en environment variables  
✅ **Factor 6**: Stateless con Redis/S3  
✅ **Factor 9**: Graceful shutdown con SIGTERM  
✅ **Factor 10**: Dev/Prod parity con Docker Compose  
✅ **Factor 11**: Structured logging a stdout  

**[Visual: Docker → Kubernetes pipeline]**

```
Local Development (Docker Compose)
         ↓
Build (Multi-stage Dockerfile)
         ↓
CI/CD (GitHub Actions)
         ↓
Production (Kubernetes)
```

**[Pantalla: Próximo tema]**

En el próximo video exploraremos **Kubernetes en profundidad**:

- Deployments y StatefulSets
- Services y Ingress
- ConfigMaps y Secrets avanzados
- Observabilidad con Prometheus

**[Cierre]**

¡Gracias por ver! Nos vemos en el próximo video.

---

## Recursos Visuales Necesarios

1. **Diagramas**:
   - Flujo de environment variables
   - Problema de estado en memoria (3 pods)
   - Redis como estado compartido
   - Timeline de graceful shutdown
   - Multi-stage build comparison
   - Arquitectura de microservices
   - HPA scaling animation

2. **Code Highlights**:
   - ❌ Anti-patterns en rojo
   - ✅ Soluciones en verde
   - Números de línea importantes resaltados

3. **Terminal Demos**:
   - docker-compose up
   - kubectl apply
   - kubectl get pods
   - Log streaming

4. **Comparaciones Lado a Lado**:
   - Legacy vs Cloud-Native
   - Development vs Production
   - Builder vs Runtime images

**Total**: 35 minutos de contenido denso pero práctico, con demos en vivo y ejemplos concretos.
