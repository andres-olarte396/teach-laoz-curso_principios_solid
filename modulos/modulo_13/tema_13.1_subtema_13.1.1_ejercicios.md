# Ejercicios: 12-Factor App y Containerización

## ⭐ Ejercicio 1: Identificar Violaciones de 12-Factor

**Descripción**: Analiza el siguiente código e identifica qué factores de 12-Factor App se están violando.

**Código Problemático**:

```javascript
// server.js
const express = require('express');
const fs = require('fs');
const app = express();

// ❌ Problema 1: ¿Qué factor viola?
const config = {
    database: {
        host: 'localhost',
        port: 5432,
        password: 'mysecretpassword'
    },
    apiKey: 'sk-1234567890abcdef'
};

// ❌ Problema 2: ¿Qué factor viola?
let sessionData = {};

app.post('/login', (req, res) => {
    const sessionId = Math.random().toString(36);
    sessionData[sessionId] = { userId: req.body.userId };
    res.json({ sessionId });
});

// ❌ Problema 3: ¿Qué factor viola?
app.post('/upload', (req, res) => {
    const filename = req.file.originalname;
    fs.writeFileSync(`./uploads/${filename}`, req.file.buffer);
    res.json({ url: `/uploads/${filename}` });
});

// ❌ Problema 4: ¿Qué factor viola?
app.get('/logs', (req, res) => {
    const logs = fs.readFileSync('./app.log', 'utf-8');
    res.send(logs);
});

// ❌ Problema 5: ¿Qué factor viola?
const server = require('http').createServer(app);
server.listen(3000);  // Hardcoded port

// No graceful shutdown handling
```

**Tareas**:

1. Identifica los 5 factores violados (uno por problema marcado)
2. Explica por qué cada uno es una violación
3. Describe qué problemas causaría en producción

**Solución**:

<details>
<summary>Ver solución</summary>

**Factores Violados**:

1. **Problema 1 - Factor 3 (Config)**:
   - Violación: Configuración hardcoded en el código
   - Por qué es malo: Password en código fuente, misma config para todos los environments
   - Problema en producción: No se puede cambiar config sin rebuild, secretos expuestos en repo

2. **Problema 2 - Factor 6 (Processes)**:
   - Violación: Estado en memoria (sessionData)
   - Por qué es malo: No funciona con múltiples instancias
   - Problema en producción: Con 3 pods, cada sesión solo existe en 1 pod. Load balancer envía requests a diferentes pods → sesión se pierde

3. **Problema 3 - Factor 6 (Processes)**:
   - Violación: Filesystem local como storage
   - Por qué es malo: Filesystem en containers es efímero
   - Problema en producción: Pod se reinicia → archivos desaparecen. Con múltiples pods, archivo solo existe en 1 pod

4. **Problema 4 - Factor 11 (Logs)**:
   - Violación: Logs a archivo local
   - Por qué es malo: No centralizado, difícil acceso en containers
   - Problema en producción: Logs dispersos en múltiples pods, se pierden al reiniciar pods

5. **Problema 5 - Factor 7 (Port Binding) y Factor 9 (Disposability)**:
   - Violación: Puerto hardcoded, no graceful shutdown
   - Por qué es malo: No configurable, shutdown abrupto
   - Problema en producción: Kubernetes no puede asignar puerto dinámico, requests en proceso se cortan al terminar pod

</details>

---

## ⭐⭐ Ejercicio 2: Refactorizar a 12-Factor App

**Descripción**: Refactoriza la aplicación del Ejercicio 1 para cumplir con 12-Factor App.

**Implementación**:

```typescript
// src/config.ts - Factor 3: Config from environment
interface AppConfig {
    port: number;
    database: {
        host: string;
        port: number;
        password: string;
        database: string;
    };
    redis: {
        url: string;
    };
    s3: {
        bucket: string;
        region: string;
    };
    apiKey: string;
}

function requireEnv(name: string): string {
    const value = process.env[name];
    if (!value) {
        throw new Error(`Environment variable ${name} is required`);
    }
    return value;
}

export function loadConfig(): AppConfig {
    return {
        port: parseInt(process.env.PORT || '3000'),
        database: {
            host: requireEnv('DATABASE_HOST'),
            port: parseInt(requireEnv('DATABASE_PORT')),
            password: requireEnv('DATABASE_PASSWORD'),
            database: requireEnv('DATABASE_NAME'),
        },
        redis: {
            url: requireEnv('REDIS_URL'),
        },
        s3: {
            bucket: requireEnv('S3_BUCKET'),
            region: requireEnv('AWS_REGION'),
        },
        apiKey: requireEnv('API_KEY'),
    };
}
```

```typescript
// src/services/session.service.ts - Factor 6: Stateless with Redis
import { Redis } from 'ioredis';

export class SessionService {
    constructor(private redis: Redis) {}
    
    async createSession(userId: string): Promise<string> {
        const sessionId = this.generateSessionId();
        const sessionData = { userId, createdAt: Date.now() };
        
        // Store in Redis (shared state), not in memory
        await this.redis.setex(
            `session:${sessionId}`,
            3600,  // 1 hour TTL
            JSON.stringify(sessionData)
        );
        
        return sessionId;
    }
    
    async getSession(sessionId: string): Promise<{ userId: string } | null> {
        const data = await this.redis.get(`session:${sessionId}`);
        return data ? JSON.parse(data) : null;
    }
    
    async deleteSession(sessionId: string): Promise<void> {
        await this.redis.del(`session:${sessionId}`);
    }
    
    private generateSessionId(): string {
        return require('crypto').randomBytes(32).toString('hex');
    }
}
```

```typescript
// src/services/storage.service.ts - Factor 6: S3 instead of local filesystem
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { v4 as uuidv4 } from 'uuid';

export class StorageService {
    private s3: S3Client;
    
    constructor(
        private bucket: string,
        private region: string
    ) {
        this.s3 = new S3Client({ region });
    }
    
    async uploadFile(
        filename: string,
        buffer: Buffer,
        contentType: string
    ): Promise<string> {
        const key = `uploads/${uuidv4()}-${filename}`;
        
        await this.s3.send(new PutObjectCommand({
            Bucket: this.bucket,
            Key: key,
            Body: buffer,
            ContentType: contentType,
        }));
        
        return `https://${this.bucket}.s3.${this.region}.amazonaws.com/${key}`;
    }
}
```

```typescript
// src/logger.ts - Factor 11: Structured logging to stdout
interface LogEntry {
    timestamp: string;
    level: string;
    message: string;
    [key: string]: any;
}

export class Logger {
    private log(level: string, message: string, meta?: any): void {
        const entry: LogEntry = {
            timestamp: new Date().toISOString(),
            level,
            message,
            ...meta,
        };
        
        // Write to stdout (Kubernetes captures this)
        console.log(JSON.stringify(entry));
    }
    
    info(message: string, meta?: any): void {
        this.log('info', message, meta);
    }
    
    error(message: string, meta?: any): void {
        this.log('error', message, meta);
    }
    
    warn(message: string, meta?: any): void {
        this.log('warn', message, meta);
    }
}

export const logger = new Logger();
```

```typescript
// src/server.ts - Factors 7, 9: Port binding, Graceful shutdown
import express from 'express';
import { loadConfig } from './config';
import { SessionService } from './services/session.service';
import { StorageService } from './services/storage.service';
import { logger } from './logger';
import { Redis } from 'ioredis';

async function startServer() {
    // Factor 3: Load config from environment
    const config = loadConfig();
    
    // Factor 4: Backing services as attached resources
    const redis = new Redis(config.redis.url);
    const sessionService = new SessionService(redis);
    const storageService = new StorageService(config.s3.bucket, config.s3.region);
    
    const app = express();
    app.use(express.json());
    
    // Health check endpoint
    app.get('/health', (req, res) => {
        res.json({ status: 'healthy' });
    });
    
    // Login endpoint (stateless)
    app.post('/login', async (req, res) => {
        try {
            const sessionId = await sessionService.createSession(req.body.userId);
            logger.info('User logged in', { userId: req.body.userId });
            res.json({ sessionId });
        } catch (error) {
            logger.error('Login failed', { error: error.message });
            res.status(500).json({ error: 'Login failed' });
        }
    });
    
    // Upload endpoint (S3 storage)
    app.post('/upload', async (req, res) => {
        try {
            const url = await storageService.uploadFile(
                req.file.originalname,
                req.file.buffer,
                req.file.mimetype
            );
            logger.info('File uploaded', { filename: req.file.originalname, url });
            res.json({ url });
        } catch (error) {
            logger.error('Upload failed', { error: error.message });
            res.status(500).json({ error: 'Upload failed' });
        }
    });
    
    // Factor 7: Port binding from environment
    const server = app.listen(config.port, () => {
        logger.info(`Server started on port ${config.port}`);
    });
    
    // Factor 9: Graceful shutdown
    const shutdown = async (signal: string) => {
        logger.info(`${signal} received, shutting down gracefully...`);
        
        // Stop accepting new requests
        server.close(() => {
            logger.info('HTTP server closed');
        });
        
        // Close backing service connections
        await redis.quit();
        
        logger.info('Shutdown complete');
        process.exit(0);
    };
    
    process.on('SIGTERM', () => shutdown('SIGTERM'));
    process.on('SIGINT', () => shutdown('SIGINT'));
    
    return server;
}

startServer().catch(error => {
    logger.error('Failed to start server', { error: error.message });
    process.exit(1);
});
```

**Environment Variables** (.env.example):

```bash
# Factor 3: Config in environment
PORT=3000

# Database
DATABASE_HOST=postgres
DATABASE_PORT=5432
DATABASE_PASSWORD=changeme
DATABASE_NAME=myapp

# Redis
REDIS_URL=redis://redis:6379

# S3
S3_BUCKET=my-app-uploads
AWS_REGION=us-east-1

# API
API_KEY=sk-production-key-here
```

**Docker Setup**:

```dockerfile
# Dockerfile - Multi-stage build
FROM node:18-alpine AS builder

WORKDIR /app

# Factor 2: Dependencies declared
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine

# Security: Run as non-root
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/package*.json ./

USER nodejs

# Factor 7: Port binding
EXPOSE 3000

# Factor 9: Graceful shutdown (Node.js handles SIGTERM)
CMD ["node", "dist/server.js"]
```

```yaml
# docker-compose.yml - Factor 10: Dev/Prod parity
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      PORT: 3000
      DATABASE_HOST: postgres
      DATABASE_PORT: 5432
      DATABASE_PASSWORD: devpassword
      DATABASE_NAME: myapp
      REDIS_URL: redis://redis:6379
      S3_BUCKET: dev-uploads
      AWS_REGION: us-east-1
      API_KEY: dev-api-key
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
      POSTGRES_PASSWORD: devpassword
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

**Tests**:

```typescript
// tests/integration.test.ts
import request from 'supertest';
import { Redis } from 'ioredis';

describe('12-Factor App Integration Tests', () => {
    let redis: Redis;
    
    beforeAll(() => {
        redis = new Redis(process.env.REDIS_URL);
    });
    
    afterAll(async () => {
        await redis.quit();
    });
    
    test('Factor 6: Sessions are stateless (stored in Redis)', async () => {
        const response = await request(app)
            .post('/login')
            .send({ userId: 'user123' });
        
        expect(response.status).toBe(200);
        const { sessionId } = response.body;
        
        // Verify session is in Redis, not in memory
        const sessionData = await redis.get(`session:${sessionId}`);
        expect(sessionData).toBeTruthy();
        expect(JSON.parse(sessionData!).userId).toBe('user123');
    });
    
    test('Factor 11: Logs are structured JSON to stdout', () => {
        const consoleSpy = jest.spyOn(console, 'log');
        
        logger.info('Test message', { userId: 'user123' });
        
        expect(consoleSpy).toHaveBeenCalled();
        const logEntry = JSON.parse(consoleSpy.mock.calls[0][0]);
        expect(logEntry.level).toBe('info');
        expect(logEntry.message).toBe('Test message');
        expect(logEntry.userId).toBe('user123');
    });
});
```

**Resultado**: Aplicación completamente 12-Factor compliant, lista para despliegue en Kubernetes.

---

## ⭐⭐⭐ Ejercicio 3: Multi-Stage Dockerfile Optimizado

**Descripción**: Crea un Dockerfile multi-stage optimizado para una aplicación Java Spring Boot.

**Requisitos**:

1. Build con Maven
2. Imagen final mínima (JRE, no JDK)
3. Non-root user
4. Layer caching eficiente
5. Security scanning

**Implementación**:

```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app

# Copy dependency files first (better caching)
COPY pom.xml .
COPY src/main/resources/application.properties src/main/resources/

# Download dependencies (cached if pom.xml unchanged)
RUN mvn dependency:go-offline -B

# Copy source code
COPY src ./src

# Build (skip tests for faster build, tests run in CI)
RUN mvn clean package -DskipTests -B

# Stage 2: Runtime
FROM eclipse-temurin:17-jre-alpine

# Install security updates
RUN apk update && apk upgrade && apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

WORKDIR /app

# Copy JAR from builder stage
COPY --from=builder --chown=appuser:appgroup /app/target/*.jar app.jar

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# Use dumb-init for proper signal handling (Factor 9: Graceful shutdown)
ENTRYPOINT ["/usr/bin/dumb-init", "--"]

# Run application
CMD ["java", \
     "-XX:+UseContainerSupport", \
     "-XX:MaxRAMPercentage=75.0", \
     "-Djava.security.egd=file:/dev/./urandom", \
     "-jar", "app.jar"]
```

**Optimizaciones Aplicadas**:

1. **Multi-stage build**:
   - Builder: 700MB (Maven + JDK)
   - Runtime: 180MB (JRE only)
   - Reducción: ~72%

2. **Layer caching**:

   ```dockerfile
   # ✅ BIEN: Dependencies copied first
   COPY pom.xml .
   RUN mvn dependency:go-offline
   # Cache se reutiliza si pom.xml no cambia
   
   COPY src ./src
   # Solo se rebuilds si src cambia
   ```

3. **Security**:
   - Alpine base (menos superficie de ataque)
   - Non-root user
   - Security updates installed
   - JRE only (no build tools en producción)

4. **Signal handling**:
   - `dumb-init` para proper SIGTERM forwarding
   - Java recibe señal correctamente → graceful shutdown

**docker-compose.yml para desarrollo**:

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/myapp
      SPRING_DATASOURCE_USERNAME: user
      SPRING_DATASOURCE_PASSWORD: password
      SPRING_REDIS_HOST: redis
      SPRING_REDIS_PORT: 6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 3s
      retries: 3
  
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
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

**Build y Test**:

```bash
# Build
docker build -t myapp:1.0.0 .

# Check image size
docker images myapp:1.0.0
# REPOSITORY   TAG     SIZE
# myapp        1.0.0   180MB  ✅ Small!

# Security scan
docker scan myapp:1.0.0

# Run locally
docker-compose up

# Test
curl http://localhost:8080/actuator/health
# {"status":"UP"}

# Test graceful shutdown
docker-compose stop app
# Logs should show:
# "Shutting down ExecutorService 'applicationTaskExecutor'"
# "Closing JPA EntityManagerFactory"
# ✅ Graceful shutdown working
```

**Kubernetes Deployment**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
      - name: app
        image: myapp:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
      terminationGracePeriodSeconds: 30
```

**Resultado**: Imagen optimizada, segura y lista para producción.

---

## ⭐⭐⭐⭐ Ejercicio 4: Sistema Completo Cloud-Native

**Descripción**: Diseña e implementa un sistema de e-commerce cloud-native completo siguiendo 12-Factor App.

**Requisitos**:

1. **Microservicios**:
   - Product Service (catálogo)
   - Order Service (órdenes)
   - Payment Service (pagos)
   - Notification Service (emails)

2. **12-Factor Compliance**:
   - Config en environment
   - Stateless (Redis para sessions)
   - Logs estructurados
   - Graceful shutdown
   - Dev/Prod parity

3. **Backing Services**:
   - PostgreSQL (productos, órdenes)
   - Redis (cache, sessions)
   - RabbitMQ (eventos)
   - S3 (imágenes de productos)

4. **Observabilidad**:
   - Health checks
   - Structured logging
   - Metrics (Prometheus)

**Arquitectura**:

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
┌──────▼──────────┐
│  API Gateway    │
│  (Port 80/443)  │
└──────┬──────────┘
       │
       ├────────────────┬───────────────┬──────────────────┐
       │                │               │                  │
┌──────▼──────┐  ┌──────▼──────┐ ┌─────▼──────┐  ┌────────▼────────┐
│  Product    │  │   Order     │ │  Payment   │  │  Notification   │
│  Service    │  │   Service   │ │  Service   │  │   Service       │
│  :3001      │  │   :3002     │ │  :3003     │  │   :3004         │
└──────┬──────┘  └──────┬──────┘ └─────┬──────┘  └────────┬────────┘
       │                │               │                  │
       │                └───────┬───────┘                  │
       │                        │                          │
┌──────▼──────┐       ┌─────────▼────────┐       ┌────────▼────────┐
│ PostgreSQL  │       │    RabbitMQ      │       │   Email SMTP    │
│ (Products,  │       │   (Events)       │       │   (SendGrid)    │
│  Orders)    │       └──────────────────┘       └─────────────────┘
└─────────────┘
       │
┌──────▼──────┐       ┌──────────────────┐
│   Redis     │       │       S3         │
│  (Cache)    │       │  (Product Imgs)  │
└─────────────┘       └──────────────────┘
```

**1. Product Service**:

```typescript
// product-service/src/server.ts
import express from 'express';
import { loadConfig } from './config';
import { ProductRepository } from './repositories/product.repository';
import { StorageService } from './services/storage.service';
import { CacheService } from './services/cache.service';
import { logger } from './logger';
import { createPrometheusMiddleware } from './metrics';

async function startServer() {
    const config = loadConfig();
    
    // Backing services
    const productRepo = new ProductRepository(config.database);
    const storage = new StorageService(config.s3.bucket, config.s3.region);
    const cache = new CacheService(config.redis.url);
    
    const app = express();
    app.use(express.json());
    app.use(createPrometheusMiddleware());
    
    // Health check
    app.get('/health', async (req, res) => {
        try {
            await productRepo.ping();
            await cache.ping();
            res.json({ status: 'healthy' });
        } catch (error) {
            res.status(503).json({ status: 'unhealthy', error: error.message });
        }
    });
    
    // Get products (with caching)
    app.get('/products', async (req, res) => {
        try {
            // Try cache first
            const cached = await cache.get('products:all');
            if (cached) {
                logger.info('Products fetched from cache');
                return res.json(JSON.parse(cached));
            }
            
            // Fetch from database
            const products = await productRepo.findAll();
            
            // Cache for 5 minutes
            await cache.set('products:all', JSON.stringify(products), 300);
            
            logger.info('Products fetched from database', { count: products.length });
            res.json(products);
        } catch (error) {
            logger.error('Failed to fetch products', { error: error.message });
            res.status(500).json({ error: 'Failed to fetch products' });
        }
    });
    
    // Create product
    app.post('/products', async (req, res) => {
        try {
            const product = await productRepo.create(req.body);
            
            // Invalidate cache
            await cache.delete('products:all');
            
            logger.info('Product created', { productId: product.id });
            res.status(201).json(product);
        } catch (error) {
            logger.error('Failed to create product', { error: error.message });
            res.status(500).json({ error: 'Failed to create product' });
        }
    });
    
    // Upload product image
    app.post('/products/:id/image', async (req, res) => {
        try {
            const url = await storage.uploadFile(
                `products/${req.params.id}/${req.file.originalname}`,
                req.file.buffer,
                req.file.mimetype
            );
            
            await productRepo.updateImageUrl(req.params.id, url);
            
            logger.info('Product image uploaded', { productId: req.params.id, url });
            res.json({ imageUrl: url });
        } catch (error) {
            logger.error('Failed to upload image', { error: error.message });
            res.status(500).json({ error: 'Failed to upload image' });
        }
    });
    
    // Graceful shutdown
    const server = app.listen(config.port, () => {
        logger.info(`Product Service started on port ${config.port}`);
    });
    
    const shutdown = async (signal: string) => {
        logger.info(`${signal} received, shutting down...`);
        server.close();
        await productRepo.close();
        await cache.close();
        logger.info('Product Service shutdown complete');
        process.exit(0);
    };
    
    process.on('SIGTERM', () => shutdown('SIGTERM'));
    process.on('SIGINT', () => shutdown('SIGINT'));
}

startServer();
```

**2. Order Service (con eventos)**:

```typescript
// order-service/src/server.ts
import express from 'express';
import { loadConfig } from './config';
import { OrderRepository } from './repositories/order.repository';
import { EventBus } from './services/eventbus.service';
import { logger } from './logger';

async function startServer() {
    const config = loadConfig();
    
    const orderRepo = new OrderRepository(config.database);
    const eventBus = new EventBus(config.rabbitmq.url);
    
    await eventBus.connect();
    
    const app = express();
    app.use(express.json());
    
    // Create order
    app.post('/orders', async (req, res) => {
        try {
            const order = await orderRepo.create({
                userId: req.body.userId,
                items: req.body.items,
                totalAmount: req.body.totalAmount,
                status: 'PENDING'
            });
            
            // Publish event (otros servicios se suscriben)
            await eventBus.publish('order.created', {
                orderId: order.id,
                userId: order.userId,
                totalAmount: order.totalAmount,
                timestamp: new Date().toISOString()
            });
            
            logger.info('Order created', { orderId: order.id, userId: order.userId });
            res.status(201).json(order);
        } catch (error) {
            logger.error('Failed to create order', { error: error.message });
            res.status(500).json({ error: 'Failed to create order' });
        }
    });
    
    // Listen to payment events
    await eventBus.subscribe('payment.completed', async (event) => {
        try {
            await orderRepo.updateStatus(event.orderId, 'PAID');
            
            // Publish next event
            await eventBus.publish('order.paid', {
                orderId: event.orderId,
                timestamp: new Date().toISOString()
            });
            
            logger.info('Order marked as paid', { orderId: event.orderId });
        } catch (error) {
            logger.error('Failed to process payment event', { error: error.message });
        }
    });
    
    const server = app.listen(config.port, () => {
        logger.info(`Order Service started on port ${config.port}`);
    });
    
    const shutdown = async (signal: string) => {
        logger.info(`${signal} received, shutting down...`);
        server.close();
        await eventBus.close();
        await orderRepo.close();
        logger.info('Order Service shutdown complete');
        process.exit(0);
    };
    
    process.on('SIGTERM', () => shutdown('SIGTERM'));
    process.on('SIGINT', () => shutdown('SIGINT'));
}

startServer();
```

**3. Docker Compose (Dev/Prod Parity)**:

```yaml
version: '3.8'

services:
  # Product Service
  product-service:
    build: ./product-service
    ports:
      - "3001:3001"
    environment:
      PORT: 3001
      DATABASE_URL: postgres://user:pass@postgres:5432/products
      REDIS_URL: redis://redis:6379
      S3_BUCKET: dev-product-images
      AWS_REGION: us-east-1
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
  
  # Order Service
  order-service:
    build: ./order-service
    ports:
      - "3002:3002"
    environment:
      PORT: 3002
      DATABASE_URL: postgres://user:pass@postgres:5432/orders
      RABBITMQ_URL: amqp://rabbitmq:5672
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
  
  # Payment Service
  payment-service:
    build: ./payment-service
    ports:
      - "3003:3003"
    environment:
      PORT: 3003
      RABBITMQ_URL: amqp://rabbitmq:5672
      STRIPE_API_KEY: sk_test_xxx
    depends_on:
      rabbitmq:
        condition: service_healthy
  
  # Notification Service
  notification-service:
    build: ./notification-service
    ports:
      - "3004:3004"
    environment:
      PORT: 3004
      RABBITMQ_URL: amqp://rabbitmq:5672
      SENDGRID_API_KEY: SG.xxx
    depends_on:
      rabbitmq:
        condition: service_healthy
  
  # Backing Services
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 5s
      retries: 5
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
  
  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
  
  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "15672:15672"  # Management UI
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq

volumes:
  postgres-data:
  redis-data:
  rabbitmq-data:
```

**4. Kubernetes Deployment**:

```yaml
# k8s/product-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      containers:
      - name: product-service
        image: myregistry/product-service:1.0.0
        ports:
        - containerPort: 3001
        envFrom:
        - configMapRef:
            name: product-service-config
        - secretRef:
            name: product-service-secrets
        livenessProbe:
          httpGet:
            path: /health
            port: 3001
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3001
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
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  selector:
    app: product-service
  ports:
  - port: 80
    targetPort: 3001
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**Tests End-to-End**:

```typescript
// e2e/ecommerce.test.ts
describe('E-Commerce Cloud-Native System', () => {
    test('Complete order flow', async () => {
        // 1. Create product
        const productResponse = await fetch('http://localhost:3001/products', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                name: 'Laptop',
                price: 999.99,
                stock: 10
            })
        });
        const product = await productResponse.json();
        expect(product.id).toBeDefined();
        
        // 2. Create order
        const orderResponse = await fetch('http://localhost:3002/orders', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                userId: 'user123',
                items: [{ productId: product.id, quantity: 1 }],
                totalAmount: 999.99
            })
        });
        const order = await orderResponse.json();
        expect(order.status).toBe('PENDING');
        
        // 3. Wait for events to propagate
        await new Promise(resolve => setTimeout(resolve, 2000));
        
        // 4. Verify order was paid (via Payment Service event)
        const updatedOrderResponse = await fetch(`http://localhost:3002/orders/${order.id}`);
        const updatedOrder = await updatedOrderResponse.json();
        expect(updatedOrder.status).toBe('PAID');
        
        // 5. Verify notification was sent (check logs or email service)
        // This would be verified via observability tools in production
    });
    
    test('System is stateless - requests can be handled by any pod', async () => {
        // Create 10 concurrent requests
        const requests = Array.from({ length: 10 }, () =>
            fetch('http://localhost:3001/products')
        );
        
        const responses = await Promise.all(requests);
        
        // All should succeed (load balanced across pods)
        responses.forEach(response => {
            expect(response.status).toBe(200);
        });
    });
    
    test('Graceful shutdown - ongoing requests complete', async () => {
        // Start long-running request
        const requestPromise = fetch('http://localhost:3001/products/slow');
        
        // Send SIGTERM to pod
        execSync('kubectl delete pod -l app=product-service --grace-period=30');
        
        // Request should complete successfully
        const response = await requestPromise;
        expect(response.status).toBe(200);
    });
});
```

**Resultado**: Sistema completo cloud-native, 12-Factor compliant, listo para producción en Kubernetes con escalado automático, observabilidad y resiliencia.

---

## Resumen de Ejercicios

| Ejercicio | Nivel | Tema | Duración |
|-----------|-------|------|----------|
| 1 | ⭐ | Identificar violaciones 12-Factor | 30 min |
| 2 | ⭐⭐ | Refactorizar a 12-Factor App | 2 horas |
| 3 | ⭐⭐⭐ | Dockerfile multi-stage optimizado | 1.5 horas |
| 4 | ⭐⭐⭐⭐ | Sistema e-commerce cloud-native completo | 4 horas |

**Próximo**: Evaluación con proyecto de migración cloud-native.
