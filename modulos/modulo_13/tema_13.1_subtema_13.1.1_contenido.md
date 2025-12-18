# 12-Factor App y Containerización

## Introducción

**Cloud-Native** no es solo "ejecutar en la nube". Es un conjunto de prácticas y patrones para construir aplicaciones que aprovechen las características de la nube: escalabilidad elástica, resiliencia, despliegue continuo y automatización.

Los **12-Factor App principles** son la base fundamental para aplicaciones cloud-native, definiendo prácticas que garantizan portabilidad, escalabilidad y mantenibilidad.

**Containerización** con Docker y orquestación con Kubernetes son las tecnologías que materializan estos principios.

---

## Los 12 Factores

### Factor 1: Codebase (Base de Código)

**Principio**: Una codebase rastreada en control de versiones, múltiples deploys.

```
❌ MAL: Diferentes repos para dev/staging/production
✅ BIEN: Un repo, múltiples environments vía configuración

Repo: myapp
├── main branch → production
├── staging branch → staging
└── develop branch → development
```

**Implementación**:

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main, staging, develop]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Determine environment
        id: env
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "environment=production" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == "refs/heads/staging" ]]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
          else
            echo "environment=development" >> $GITHUB_OUTPUT
          fi

      - name: Deploy to ${{ steps.env.outputs.environment }}
        run: |
          ./deploy.sh ${{ steps.env.outputs.environment }}
```

**SOLID Connection (SRP)**:

```typescript
// ✅ Single codebase, different configs
class Application {
  constructor(private config: AppConfig) {}

  start(): void {
    // Same code, different behavior based on config
    const db = this.config.database;
    const port = this.config.port;
    // ...
  }
}

// Different environments use same codebase
const prodApp = new Application(productionConfig);
const devApp = new Application(developmentConfig);
```

---

### Factor 2: Dependencies (Dependencias)

**Principio**: Declarar y aislar dependencias explícitamente.

**❌ Problema Común**:

```bash
# README.md dice:
# "Requiere Python 3.9, PostgreSQL 14, Redis 6"
# Pero no hay forma de verificar o instalar automáticamente
```

**✅ Solución**:

```json
// package.json (Node.js)
{
  "name": "my-app",
  "version": "1.0.0",
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=9.0.0"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.0",
    "redis": "^4.6.5"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "@types/node": "^20.0.0"
  }
}
```

```python
# requirements.txt (Python)
fastapi==0.109.0
uvicorn[standard]==0.27.0
sqlalchemy==2.0.25
redis==5.0.1
pydantic==2.5.3

# requirements-dev.txt
pytest==7.4.4
black==24.1.1
mypy==1.8.0
```

```dockerfile
# Dockerfile - Aislamiento completo
FROM python:3.11-slim

WORKDIR /app

# Copy dependencies first (caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**SOLID Connection (DIP)**:

```java
// ✅ Dependency Injection + Explicit Dependencies
public class OrderService {
    private final OrderRepository repository;
    private final EmailService emailService;

    // Dependencies declared in constructor
    public OrderService(
        OrderRepository repository,
        EmailService emailService
    ) {
        this.repository = repository;
        this.emailService = emailService;
    }
}

// Dependencies declared in pom.xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
        <version>3.2.0</version>
    </dependency>
</dependencies>
```

---

### Factor 3: Config (Configuración)

**Principio**: Almacenar configuración en el entorno, NO en el código.

**❌ Anti-Pattern**:

```javascript
// config.js - ❌ Hardcoded
const config = {
  database: {
    host: "prod-db.example.com", // ❌ En código
    password: "super-secret-123", // ❌ PELIGROSO
  },
  apiKey: "sk-abc123xyz", // ❌ En código
};
```

**✅ Correcto**:

```typescript
// config.ts - ✅ From environment
interface AppConfig {
  database: {
    host: string;
    port: number;
    password: string;
  };
  apiKey: string;
  logLevel: string;
}

function loadConfig(): AppConfig {
  return {
    database: {
      host: process.env.DATABASE_HOST || "localhost",
      port: parseInt(process.env.DATABASE_PORT || "5432"),
      password: requireEnv("DATABASE_PASSWORD"), // Must be set
    },
    apiKey: requireEnv("API_KEY"),
    logLevel: process.env.LOG_LEVEL || "info",
  };
}

function requireEnv(name: string): string {
  const value = process.env[name];
  if (!value) {
    throw new Error(`Environment variable ${name} is required`);
  }
  return value;
}

// Usage
const config = loadConfig();
```

**Diferentes Environments**:

```bash
# .env.development
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_PASSWORD=dev-password
API_KEY=dev-key-123
LOG_LEVEL=debug

# .env.production (en secrets manager, no en repo)
DATABASE_HOST=prod-db.example.com
DATABASE_PORT=5432
DATABASE_PASSWORD=<from-secrets-manager>
API_KEY=<from-secrets-manager>
LOG_LEVEL=warn
```

**Kubernetes ConfigMap + Secrets**:

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres-service"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"

---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DATABASE_PASSWORD: cGFzc3dvcmQxMjM= # base64
  API_KEY: c2stYWJjMTIzeHl6 # base64

---
# deployment.yaml
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
            - secretRef:
                name: app-secrets
```

---

### Factor 4: Backing Services (Servicios de Respaldo)

**Principio**: Tratar backing services (DB, cache, queues) como recursos adjuntos.

**Concepto**: Poder cambiar de un servicio local a uno cloud sin cambiar código.

```typescript
// ✅ Abstracción de backing service
interface CacheService {
  get(key: string): Promise<string | null>;
  set(key: string, value: string, ttl?: number): Promise<void>;
  delete(key: string): Promise<void>;
}

// Implementación local (desarrollo)
class LocalCacheService implements CacheService {
  private cache = new Map<string, string>();

  async get(key: string): Promise<string | null> {
    return this.cache.get(key) || null;
  }

  async set(key: string, value: string): Promise<void> {
    this.cache.set(key, value);
  }

  async delete(key: string): Promise<void> {
    this.cache.delete(key);
  }
}

// Implementación cloud (producción)
class RedisCacheService implements CacheService {
  constructor(private client: Redis) {}

  async get(key: string): Promise<string | null> {
    return await this.client.get(key);
  }

  async set(key: string, value: string, ttl?: number): Promise<void> {
    if (ttl) {
      await this.client.setex(key, ttl, value);
    } else {
      await this.client.set(key, value);
    }
  }

  async delete(key: string): Promise<void> {
    await this.client.del(key);
  }
}

// Factory basado en environment
function createCacheService(): CacheService {
  const redisUrl = process.env.REDIS_URL;

  if (redisUrl) {
    // Production: Use Redis
    const client = new Redis(redisUrl);
    return new RedisCacheService(client);
  } else {
    // Development: Use in-memory
    return new LocalCacheService();
  }
}

// Application code no cambia
class OrderService {
  constructor(private cache: CacheService) {}

  async getOrder(orderId: string): Promise<Order> {
    // Mismo código para local y cloud
    const cached = await this.cache.get(`order:${orderId}`);
    if (cached) {
      return JSON.parse(cached);
    }
    // ...
  }
}
```

**SOLID Connection (DIP)**:

```kotlin
// ✅ Dependency Inversion: Depend on abstraction, not implementation
interface EmailService {
    suspend fun send(to: String, subject: String, body: String)
}

// Development: Log to console
class ConsoleEmailService : EmailService {
    override suspend fun send(to: String, subject: String, body: String) {
        println("Email to $to: $subject")
    }
}

// Production: Use SendGrid
class SendGridEmailService(private val apiKey: String) : EmailService {
    override suspend fun send(to: String, subject: String, body: String) {
        // Call SendGrid API
    }
}

// Factory
fun createEmailService(): EmailService {
    val sendGridKey = System.getenv("SENDGRID_API_KEY")

    return if (sendGridKey != null) {
        SendGridEmailService(sendGridKey)
    } else {
        ConsoleEmailService()
    }
}
```

---

### Factor 5: Build, Release, Run (Construcción, Lanzamiento, Ejecución)

**Principio**: Separar estrictamente las etapas de build, release y run.

```
BUILD → RELEASE → RUN

BUILD:  Source code + dependencies → Executable bundle
        (Dockerfile build, npm build, mvn package)

RELEASE: BUILD + Config → Release ready to run
         (Docker tag, version number, config injection)

RUN:    Execute release in environment
        (docker run, kubectl apply)
```

**Implementación CI/CD**:

```yaml
# GitHub Actions
name: CI/CD Pipeline

on:
  push:
    branches: [main]

jobs:
  # STAGE 1: BUILD
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build Docker image
        run: |
          docker build -t myapp:${{ github.sha }} .

      - name: Run tests
        run: |
          docker run myapp:${{ github.sha }} npm test

      - name: Push to registry
        run: |
          docker tag myapp:${{ github.sha }} registry.example.com/myapp:${{ github.sha }}
          docker push registry.example.com/myapp:${{ github.sha }}

  # STAGE 2: RELEASE
  release:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Create release
        run: |
          # Tag as release
          docker tag registry.example.com/myapp:${{ github.sha }} \
                     registry.example.com/myapp:v1.2.3
          docker push registry.example.com/myapp:v1.2.3

      - name: Generate release manifest
        run: |
          cat > release.yaml <<EOF
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: myapp
          spec:
            template:
              spec:
                containers:
                - name: app
                  image: registry.example.com/myapp:v1.2.3
          EOF

  # STAGE 3: RUN
  deploy:
    needs: release
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: |
          kubectl apply -f release.yaml
          kubectl rollout status deployment/myapp
```

**Beneficios**:

- ✅ Build una vez, deploy muchas veces
- ✅ Rollback fácil (solo cambiar release)
- ✅ Reproducibilidad (mismo build → mismo resultado)

---

### Factor 6: Processes (Procesos)

**Principio**: Ejecutar la app como uno o más procesos stateless.

**❌ Anti-Pattern (Stateful)**:

```javascript
// ❌ Estado en memoria - NO funciona con múltiples instancias
let requestCount = 0; // ❌ Solo en esta instancia

app.get("/stats", (req, res) => {
  requestCount++;
  res.json({ count: requestCount }); // ❌ Diferente en cada pod
});
```

**✅ Correcto (Stateless)**:

```typescript
// ✅ Estado en backing service (Redis)
import { Redis } from "ioredis";

const redis = new Redis(process.env.REDIS_URL);

app.get("/stats", async (req, res) => {
  const count = await redis.incr("request_count");
  res.json({ count }); // ✅ Consistente en todas las instancias
});
```

**Session Management**:

```python
from fastapi import FastAPI, Depends
from fastapi_sessions import SessionMiddleware
import redis

app = FastAPI()

# ❌ In-memory sessions - NO funciona con múltiples pods
# app.add_middleware(SessionMiddleware, secret_key="secret")

# ✅ Redis-backed sessions
redis_client = redis.from_url(os.getenv("REDIS_URL"))

@app.get("/cart")
async def get_cart(session_id: str = Depends(get_session_id)):
    # Session stored in Redis, not in memory
    cart = redis_client.get(f"session:{session_id}:cart")
    return json.loads(cart) if cart else []

@app.post("/cart/add")
async def add_to_cart(item: Item, session_id: str = Depends(get_session_id)):
    cart = await get_cart(session_id)
    cart.append(item)
    redis_client.setex(
        f"session:{session_id}:cart",
        3600,  # 1 hour TTL
        json.dumps(cart)
    )
    return {"status": "added"}
```

**File Uploads**:

```java
// ❌ Local filesystem - NO funciona con múltiples pods
@PostMapping("/upload")
public String uploadFile(MultipartFile file) {
    file.transferTo(new File("/tmp/uploads/" + file.getOriginalFilename()));
    return "uploaded";  // ❌ Solo en este pod
}

// ✅ Object storage (S3)
@PostMapping("/upload")
public String uploadFile(MultipartFile file) {
    String key = "uploads/" + UUID.randomUUID() + "-" + file.getOriginalFilename();

    s3Client.putObject(
        PutObjectRequest.builder()
            .bucket("my-bucket")
            .key(key)
            .build(),
        RequestBody.fromBytes(file.getBytes())
    );

    return "uploaded to " + key;  // ✅ Accesible desde todos los pods
}
```

---

### Factor 7: Port Binding (Vinculación de Puertos)

**Principio**: La app es completamente auto-contenida y expone servicios vía port binding.

**Concepto**: No depender de un servidor externo (Apache, Nginx). La app incluye su propio servidor HTTP.

```typescript
// ✅ App auto-contenida con servidor HTTP embebido
import express from "express";

const app = express();
const port = process.env.PORT || 3000;

app.get("/health", (req, res) => {
  res.json({ status: "healthy" });
});

app.get("/api/users", (req, res) => {
  // ...
});

// La app inicia su propio servidor
app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
```

```python
# ✅ FastAPI con Uvicorn embebido
from fastapi import FastAPI
import uvicorn

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "healthy"}

if __name__ == "__main__":
    # App inicia su propio servidor
    port = int(os.getenv("PORT", "8000"))
    uvicorn.run(app, host="0.0.0.0", port=port)
```

**Dockerfile**:

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

# App expone puerto vía environment variable
ENV PORT=8000
EXPOSE 8000

# App inicia su propio servidor (no Apache/Nginx)
CMD ["python", "main.py"]
```

**Kubernetes Service**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80 # External port
      targetPort: 8000 # App's port (from PORT env var)
```

---

### Factor 8: Concurrency (Concurrencia)

**Principio**: Escalar horizontalmente mediante el modelo de procesos.

```
Scale OUT (add instances), not UP (bigger machine)

❌ Vertical Scaling:
   1 pod with 8 CPU, 16GB RAM

✅ Horizontal Scaling:
   8 pods with 1 CPU, 2GB RAM each
```

**Kubernetes HPA (Horizontal Pod Autoscaler)**:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70 # Scale cuando CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

**Process Types**:

```yaml
# Procfile (diferentes tipos de proceso)
web: uvicorn main:app --host 0.0.0.0 --port $PORT
worker: celery -A tasks worker --loglevel=info
scheduler: celery -A tasks beat --loglevel=info
```

```yaml
# Kubernetes: Diferentes deployments para diferentes procesos
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 5 # Scale web processes
  template:
    spec:
      containers:
        - name: web
          image: myapp:1.0.0
          command: ["uvicorn", "main:app"]

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
spec:
  replicas: 3 # Scale worker processes independently
  template:
    spec:
      containers:
        - name: worker
          image: myapp:1.0.0
          command: ["celery", "-A", "tasks", "worker"]
```

---

### Factor 9: Disposability (Desechabilidad)

**Principio**: Maximizar robustez con fast startup y graceful shutdown.

**Fast Startup**:

```typescript
// ✅ Inicialización rápida
async function startApp() {
  const startTime = Date.now();

  // 1. Load config (fast)
  const config = loadConfig();

  // 2. Create app (fast)
  const app = createApp(config);

  // 3. Connect to backing services (with timeout)
  await Promise.race([
    connectToDatabase(config.database),
    timeout(5000, "Database connection timeout"),
  ]);

  // 4. Start server
  const server = app.listen(config.port);

  console.log(`App started in ${Date.now() - startTime}ms`);

  return server;
}
```

**Graceful Shutdown**:

````typescript
// ✅ Manejo de señales para shutdown graceful
const server = await startApp();

// Handle SIGTERM (Kubernetes envía esto al terminar pod)
process.on('SIGTERM', async () => {
    console.log('SIGTERM received, shutting down gracefully...');

    // 1. Stop accepting new requests
    server.close(() => {
        console.log('HTTP server closed');
    });

    // 2. Finish processing ongoing requests (with timeout)
    await Promise.race([
        waitForOngoingRequests(),
        timeout(30000, 'Shutdown timeout')
    ]);

    // 3. Close database connections
    await database.close();

    // 4. Exit
    console.log('Shutdown complete');
    process.exit(0);
});

// Kubernetes configuration
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30  # Give app 30s to shutdown
      containers:
      - name: app
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]  # Give time for load balancer to update
````

**Worker con Graceful Shutdown**:

```python
import signal
import sys

class Worker:
    def __init__(self):
        self.should_stop = False
        self.current_job = None

    def handle_sigterm(self, signum, frame):
        print("SIGTERM received, finishing current job...")
        self.should_stop = True

    def run(self):
        # Register signal handler
        signal.signal(signal.SIGTERM, self.handle_sigterm)

        while not self.should_stop:
            # Get job from queue
            job = queue.get(timeout=1)

            if job:
                self.current_job = job
                try:
                    self.process_job(job)
                    job.ack()  # Mark as complete
                except Exception as e:
                    job.nack()  # Return to queue for retry
                finally:
                    self.current_job = None

        print("Worker shutdown complete")
        sys.exit(0)
```

---

### Factor 10: Dev/Prod Parity (Paridad Desarrollo/Producción)

**Principio**: Mantener desarrollo, staging y producción lo más similares posible.

**3 Gaps to Minimize**:

1. **Time Gap**: Deploy frecuentemente (horas, no semanas)
2. **Personnel Gap**: Developers que escriben código, lo deploya
3. **Tools Gap**: Mismas herramientas en dev y prod

**❌ Anti-Pattern**:

```
Development:
- SQLite database
- In-memory cache
- Mock email service
- Windows OS

Production:
- PostgreSQL database
- Redis cache
- SendGrid email
- Linux OS
```

**✅ Correcto (Docker Compose para dev)**:

```yaml
# docker-compose.yml - Mismo stack que producción
version: "3.8"

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://user:pass@postgres:5432/myapp
      REDIS_URL: redis://redis:6379
      EMAIL_SERVICE: sendgrid # Mismo que prod
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15 # Misma versión que prod
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7 # Misma versión que prod
    volumes:
      - redis-data:/data

volumes:
  postgres-data:
  redis-data:
```

**Inicio rápido**:

```bash
# Un comando para levantar environment completo
docker-compose up

# ✅ Developer tiene exactamente lo mismo que producción
```

---

### Factor 11: Logs

**Principio**: Tratar logs como event streams, escribir a stdout/stderr.

**❌ Anti-Pattern**:

```typescript
// ❌ Log a archivo local
import fs from "fs";

function log(message: string) {
  fs.appendFileSync("/var/log/app.log", message + "\n");
  // ❌ No funciona en containers (filesystem efímero)
  // ❌ No funciona con múltiples pods
}
```

**✅ Correcto**:

```typescript
// ✅ Log a stdout - Kubernetes lo captura
function log(level: string, message: string, meta?: any) {
  const logEntry = {
    timestamp: new Date().toISOString(),
    level: level,
    message: message,
    ...meta,
  };

  // Write to stdout (Kubernetes captura esto)
  console.log(JSON.stringify(logEntry));
}

// Usage
log("info", "Order created", { orderId: "123", userId: "u456" });
log("error", "Payment failed", { orderId: "123", error: "Insufficient funds" });
```

**Structured Logging**:

```python
import logging
import json

# ✅ Structured logs (JSON) para fácil parsing
class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            'timestamp': self.formatTime(record),
            'level': record.levelname,
            'message': record.getMessage(),
            'logger': record.name,
        }

        if hasattr(record, 'user_id'):
            log_data['user_id'] = record.user_id

        if hasattr(record, 'order_id'):
            log_data['order_id'] = record.order_id

        return json.dumps(log_data)

# Configure logger
handler = logging.StreamHandler()  # stdout
handler.setFormatter(JSONFormatter())
logger = logging.getLogger()
logger.addHandler(handler)

# Usage
logger.info('Order created', extra={'user_id': 'u123', 'order_id': 'o456'})
```

**Log Aggregation (Kubernetes + ELK/Loki)**:

```yaml
# Kubernetes captura stdout/stderr automáticamente
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
          # App escribe a stdout
          # Kubernetes captura y envía a log aggregator (ELK, Loki, CloudWatch)
```

**Query Logs**:

```bash
# Kubernetes logs
kubectl logs deployment/myapp --tail=100 --follow

# Loki query (LogQL)
{app="myapp"} |= "error" | json | level="error"

# Elasticsearch query
GET /logs-*/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "level": "error" } },
        { "range": { "timestamp": { "gte": "now-1h" } } }
      ]
    }
  }
}
```

---

### Factor 12: Admin Processes (Procesos Administrativos)

**Principio**: Ejecutar tareas admin/management como one-off processes.

**Ejemplos**: Database migrations, REPL/console, one-time scripts.

**✅ Kubernetes Job para migration**:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp:1.0.0
          command: ["npm", "run", "migrate"] # One-off command
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
      restartPolicy: Never
  backoffLimit: 3
```

**Database Migration**:

```typescript
// migrations/001-create-users-table.ts
export async function up(db: Database): Promise<void> {
  await db.query(`
        CREATE TABLE users (
            id UUID PRIMARY KEY,
            email VARCHAR(255) UNIQUE NOT NULL,
            created_at TIMESTAMP DEFAULT NOW()
        )
    `);
}

export async function down(db: Database): Promise<void> {
  await db.query("DROP TABLE users");
}
```

```bash
# Run migration as one-off job
kubectl apply -f db-migration.yaml
kubectl wait --for=condition=complete job/db-migration

# Or locally with same codebase
docker-compose run --rm app npm run migrate
```

**Admin Console**:

```python
# manage.py - Django-style management commands
import click

@click.group()
def cli():
    pass

@cli.command()
def migrate():
    """Run database migrations"""
    run_migrations()

@cli.command()
@click.argument('email')
def create_admin(email: str):
    """Create admin user"""
    user = User(email=email, is_admin=True)
    db.save(user)
    print(f"Admin user {email} created")

@cli.command()
def shell():
    """Interactive Python shell with app context"""
    import code
    code.interact(local=globals())

if __name__ == '__main__':
    cli()
```

```bash
# Run admin commands
python manage.py migrate
python manage.py create-admin admin@example.com
python manage.py shell
```

---

## Docker: Containerización

### Dockerfile Optimizado

```dockerfile
# Multi-stage build para optimizar tamaño
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files first (caching)
COPY package*.json ./
RUN npm ci --only=production

# Copy source code
COPY . .

# Build
RUN npm run build

# ========== Production Stage ==========
FROM node:18-alpine

# Run as non-root user (security)
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy only production dependencies and build
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist

USER nodejs

EXPOSE 3000

CMD ["node", "dist/main.js"]
```

**Optimizaciones**:

1. ✅ Multi-stage build (imagen final más pequeña)
2. ✅ Copy dependencies antes de código (mejor caching)
3. ✅ Non-root user (seguridad)
4. ✅ Alpine Linux (imagen base pequeña)

### Docker Compose para Development

```yaml
version: "3.8"

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - .:/app # Hot reload
      - /app/node_modules # Don't override node_modules
    environment:
      DATABASE_URL: postgres://user:pass@postgres:5432/myapp
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 5s
      retries: 5
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

volumes:
  postgres-data:
  redis-data:
```

---

## Resumen: 12-Factor App

| Factor               | Principio                     | Implementación                   |
| -------------------- | ----------------------------- | -------------------------------- |
| 1. Codebase          | Un repo, múltiples deploys    | Git + branches                   |
| 2. Dependencies      | Declarar explícitamente       | package.json, requirements.txt   |
| 3. Config            | En environment, no en código  | Environment variables            |
| 4. Backing Services  | Recursos adjuntos             | Abstracción + DIP                |
| 5. Build/Release/Run | Separar etapas                | CI/CD pipeline                   |
| 6. Processes         | Stateless                     | Redis sessions, S3 storage       |
| 7. Port Binding      | Auto-contenido                | Express, FastAPI embedded server |
| 8. Concurrency       | Scale horizontally            | Kubernetes HPA                   |
| 9. Disposability     | Fast start, graceful shutdown | SIGTERM handling                 |
| 10. Dev/Prod Parity  | Mismas herramientas           | Docker Compose                   |
| 11. Logs             | Stdout/stderr streams         | Structured logging               |
| 12. Admin Processes  | One-off jobs                  | Kubernetes Jobs                  |

**Próximo tema**: Exploraremos Kubernetes en profundidad: Deployments, Services, ConfigMaps, Secrets, y patrones avanzados.
