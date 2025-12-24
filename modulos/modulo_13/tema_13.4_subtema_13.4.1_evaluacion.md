# Evaluación: Observability y Monitoring

**Duración**: 3 horas  
**Puntuación total**: 100 puntos

---

## Parte 1. Preguntas Teóricas (20 puntos)

### Pregunta 1 (5 puntos)

Explica los **tres pilares de observability** y cuándo usar cada uno.

**Respuesta esperada**:

1. **Metrics (Métricas)**:
   - Números agregados en el tiempo (CPU, request rate, latency)
   - **Cuándo**: Dashboards, alertas, tendencias
   - **Ejemplo**: `rate(http_requests_total[5m])`

2. **Logs (Registros)**:
   - Eventos discretos con contexto detallado
   - **Cuándo**: Debugging, auditoría
   - **Ejemplo**: `{"event":"payment_failed","userId":"123","reason":"insufficient_funds"}`

3. **Traces (Trazas)**:
   - Flujo completo de una request a través de microservicios
   - **Cuándo**: Identificar cuellos de botella, dependencias
   - **Ejemplo**: Frontend (10ms) → Backend (50ms) → DB (200ms)

---

### Pregunta 2 (5 puntos)

¿Cuál es la diferencia entre **Counter**, **Gauge** e **Histogram** en Prometheus?

**Respuesta esperada**:

| Tipo      | Descripción                      | Ejemplo                     | Operación     |
| --------- | -------------------------------- | --------------------------- | ------------- |
| Counter   | Valor que solo incrementa        | `http_requests_total`       | `rate()`      |
| Gauge     | Valor que sube o baja            | `cpu_usage`, `memory_used`  | Valor actual  |
| Histogram | Distribución en buckets          | `http_request_duration`     | `quantile()`  |

---

### Pregunta 3 (5 puntos)

¿Qué es un **SLO** (Service Level Objective) y cómo se relaciona con **error budget**?

**Respuesta esperada**:

**SLO**: Target de calidad del servicio.

**Ejemplo**: 99.9% de requests exitosas en 30 días.

**Error Budget**: Tiempo permitido de downtime según SLO.

**Cálculo**:

- SLO: 99.9% uptime
- Error budget: 0.1% = 43 minutos/mes

**Uso**: Si consumes todo el error budget, **congelas deploys** hasta recuperarte.

---

### Pregunta 4 (5 puntos)

¿Cómo funciona **distributed tracing** con OpenTelemetry y Jaeger?

**Respuesta esperada**:

1. **Instrumentación**: OpenTelemetry inyecta context propagation (Trace ID, Span ID) en headers HTTP
2. **Spans**: Cada servicio crea span con timing
3. **Export**: Spans se envían a Jaeger Collector
4. **Visualización**: Jaeger UI reconstruye trace completo

**Ventaja**: Automático con auto-instrumentation, sin cambios en código.

---

## Parte 2: PromQL y Queries (25 puntos)

### Pregunta 5 (5 puntos)

Escribe query PromQL para calcular **request rate por segundo** en ventana de 5 minutos.

**Respuesta**:

```promql
rate(http_requests_total[5m])
```

---

### Pregunta 6 (5 puntos)

Calcula **error rate** (porcentaje) de requests con status 5xx.

**Respuesta**:

```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100
```

---

### Pregunta 7 (5 puntos)

Obtén el **p95 latency** de requests HTTP.

**Respuesta**:

```promql
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

---

### Pregunta 8 (5 puntos)

Lista los **top 5 routes más lentos** (p95).

**Respuesta**:

```promql
topk(5, histogram_quantile(0.95, sum by (le, route) (rate(http_request_duration_seconds_bucket[5m]))))
```

---

### Pregunta 9 (5 puntos)

Calcula **memory usage** como porcentaje del total.

**Respuesta**:

```promql
node_memory_used_bytes / node_memory_total_bytes * 100
```

---

## Parte 3: Implementación Práctica (55 puntos)

### Proyecto: Monitoring Stack para E-Commerce

Implementa un sistema de e-commerce con observability completa.

#### Requisitos

**3.1** Microservicios (20 puntos):

Crea 3 microservicios:

1. **API Gateway** (Node.js)
   - `GET /products` → llama a Product Service
   - `POST /orders` → llama a Order Service
2. **Product Service** (Node.js)
   - `GET /products` → retorna lista de productos
3. **Order Service** (Node.js)
   - `POST /orders` → crea orden, llama a Product Service para validar stock

**3.2** Instrumentación (15 puntos):

- Prometheus metrics: `http_requests_total`, `http_request_duration_seconds`
- OpenTelemetry tracing: Propagación de Trace ID entre servicios
- Structured logging (Winston con JSON)

**3.3** Deployment (10 puntos):

- Kubernetes con 3 Deployments
- Prometheus scraping con annotations
- Jaeger para tracing

**3.4** Dashboards (10 puntos):

- Grafana dashboard con:
  - Request rate por servicio
  - Error rate gauge
  - p95 latency por route
  - Logs panel con Loki

### Solución de Referencia

#### API Gateway

```typescript
// api-gateway/src/tracing.ts
import { NodeTracerProvider } from "@opentelemetry/sdk-trace-node";
import { JaegerExporter } from "@opentelemetry/exporter-jaeger";
import { SimpleSpanProcessor } from "@opentelemetry/sdk-trace-base";
import { registerInstrumentations } from "@opentelemetry/instrumentation";
import { HttpInstrumentation } from "@opentelemetry/instrumentation-http";
import { ExpressInstrumentation } from "@opentelemetry/instrumentation-express";

const provider = new NodeTracerProvider({
  resource: {
    attributes: {
      "service.name": "api-gateway",
    },
  },
});

const jaegerExporter = new JaegerExporter({
  endpoint: process.env.JAEGER_ENDPOINT || "http://jaeger:14268/api/traces",
});

provider.addSpanProcessor(new SimpleSpanProcessor(jaegerExporter));
provider.register();

registerInstrumentations({
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
  ],
});
```

```typescript
// api-gateway/src/metrics.ts
import promClient from "prom-client";

const register = new promClient.Registry();
promClient.collectDefaultMetrics({ register });

export const httpRequestsTotal = new promClient.Counter({
  name: "http_requests_total",
  help: "Total HTTP requests",
  labelNames: ["method", "route", "status"],
  registers: [register],
});

export const httpRequestDuration = new promClient.Histogram({
  name: "http_request_duration_seconds",
  help: "HTTP request duration",
  labelNames: ["method", "route", "status"],
  buckets: [0.01, 0.05, 0.1, 0.5, 1.0, 2.0],
  registers: [register],
});

export async function metricsHandler(req, res) {
  res.set("Content-Type", register.contentType);
  res.end(await register.metrics());
}
```

```typescript
// api-gateway/src/logger.ts
import winston from "winston";

export const logger = winston.createLogger({
  level: "info",
  format: winston.format.json(),
  defaultMeta: { service: "api-gateway" },
  transports: [new winston.transports.Console()],
});
```

```typescript
// api-gateway/src/app.ts
import "./tracing";
import express from "express";
import fetch from "node-fetch";
import { httpRequestsTotal, httpRequestDuration, metricsHandler } from "./metrics";
import { logger } from "./logger";

const app = express();
app.use(express.json());

// Metrics middleware
app.use((req, res, next) => {
  const start = Date.now();

  res.on("finish", () => {
    const duration = (Date.now() - start) / 1000;

    httpRequestsTotal.inc({
      method: req.method,
      route: req.route?.path || req.path,
      status: res.statusCode,
    });

    httpRequestDuration.observe(
      {
        method: req.method,
        route: req.route?.path || req.path,
        status: res.statusCode,
      },
      duration
    );

    logger.info({
      method: req.method,
      route: req.route?.path || req.path,
      status: res.statusCode,
      duration,
    });
  });

  next();
});

app.get("/products", async (req, res) => {
  try {
    const products = await fetch("http://product-service:8081/products").then(
      (r) => r.json()
    );

    logger.info({ event: "products_fetched", count: products.length });
    res.json(products);
  } catch (error) {
    logger.error({ event: "products_fetch_failed", error: error.message });
    res.status(500).json({ error: "Failed to fetch products" });
  }
});

app.post("/orders", async (req, res) => {
  try {
    const order = await fetch("http://order-service:8082/orders", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(req.body),
    }).then((r) => r.json());

    logger.info({ event: "order_created", orderId: order.id });
    res.status(201).json(order);
  } catch (error) {
    logger.error({ event: "order_creation_failed", error: error.message });
    res.status(500).json({ error: "Failed to create order" });
  }
});

app.get("/metrics", metricsHandler);

app.listen(8080, () => logger.info({ event: "server_started", port: 8080 }));
```

#### Product Service

```typescript
// product-service/src/app.ts
import "./tracing";
import express from "express";
import { httpRequestsTotal, httpRequestDuration, metricsHandler } from "./metrics";
import { logger } from "./logger";

const app = express();

app.use((req, res, next) => {
  const start = Date.now();

  res.on("finish", () => {
    const duration = (Date.now() - start) / 1000;

    httpRequestsTotal.inc({
      method: req.method,
      route: req.route?.path || req.path,
      status: res.statusCode,
    });

    httpRequestDuration.observe(
      {
        method: req.method,
        route: req.route?.path || req.path,
        status: res.statusCode,
      },
      duration
    );
  });

  next();
});

app.get("/products", async (req, res) => {
  // Simular delay
  await new Promise((resolve) => setTimeout(resolve, Math.random() * 100));

  const products = [
    { id: 1, name: "Laptop", price: 999.99, stock: 10 },
    { id: 2, name: "Mouse", price: 29.99, stock: 50 },
  ];

  logger.info({ event: "products_listed", count: products.length });
  res.json(products);
});

app.get("/products/:id", async (req, res) => {
  const product = { id: req.params.id, name: "Laptop", stock: 10 };
  res.json(product);
});

app.get("/metrics", metricsHandler);

app.listen(8081, () => logger.info({ event: "server_started", port: 8081 }));
```

#### Order Service

```typescript
// order-service/src/app.ts
import "./tracing";
import express from "express";
import fetch from "node-fetch";
import { httpRequestsTotal, httpRequestDuration, metricsHandler } from "./metrics";
import { logger } from "./logger";

const app = express();
app.use(express.json());

app.use((req, res, next) => {
  const start = Date.now();

  res.on("finish", () => {
    const duration = (Date.now() - start) / 1000;

    httpRequestsTotal.inc({
      method: req.method,
      route: req.route?.path || req.path,
      status: res.statusCode,
    });

    httpRequestDuration.observe(
      {
        method: req.method,
        route: req.route?.path || req.path,
        status: res.statusCode,
      },
      duration
    );
  });

  next();
});

app.post("/orders", async (req, res) => {
  const { productId, quantity } = req.body;

  // Validar stock
  const product = await fetch(
    `http://product-service:8081/products/${productId}`
  ).then((r) => r.json());

  if (product.stock < quantity) {
    logger.warn({
      event: "order_rejected",
      productId,
      quantity,
      availableStock: product.stock,
    });
    return res.status(400).json({ error: "Insufficient stock" });
  }

  const order = {
    id: Date.now().toString(),
    productId,
    quantity,
    total: product.price * quantity,
    createdAt: new Date().toISOString(),
  };

  logger.info({ event: "order_created", orderId: order.id, total: order.total });
  res.status(201).json(order);
});

app.get("/metrics", metricsHandler);

app.listen(8082, () => logger.info({ event: "server_started", port: 8082 }));
```

#### Kubernetes Deployment

```yaml
# deployments.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: api-gateway
          image: api-gateway:latest
          ports:
            - containerPort: 8080
          env:
            - name: JAEGER_ENDPOINT
              value: "http://jaeger.monitoring.svc.cluster.local:14268/api/traces"

---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
spec:
  type: LoadBalancer
  selector:
    app: api-gateway
  ports:
    - port: 80
      targetPort: 8080

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8081"
    spec:
      containers:
        - name: product-service
          image: product-service:latest
          ports:
            - containerPort: 8081
          env:
            - name: JAEGER_ENDPOINT
              value: "http://jaeger.monitoring.svc.cluster.local:14268/api/traces"

---
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  selector:
    app: product-service
  ports:
    - port: 8081
      targetPort: 8081

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8082"
    spec:
      containers:
        - name: order-service
          image: order-service:latest
          ports:
            - containerPort: 8082
          env:
            - name: JAEGER_ENDPOINT
              value: "http://jaeger.monitoring.svc.cluster.local:14268/api/traces"

---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
    - port: 8082
      targetPort: 8082
```

**Deploy**:

```bash
kubectl apply -f deployments.yaml

# Generar tráfico
for i in {1..100}; do
  curl http://api-gateway/products
  curl -X POST http://api-gateway/orders \
    -H "Content-Type: application/json" \
    -d '{"productId":1,"quantity":2}'
done
```

**Verificar en Jaeger**:

```
Trace: POST /orders (120ms)
    ↓
API Gateway (10ms)
    ↓
Order Service (110ms)
    ↓
Product Service (50ms)
```

---

## Criterios de Evaluación

| Sección                        | Puntos | Criterios                                          |
| ------------------------------ | ------ | -------------------------------------------------- |
| Parte 1. Teoría                | 20     | Comprensión de pilares y métricas                  |
| Parte 2: PromQL                | 25     | Queries correctos                                  |
| Parte 3: Implementación        | 55     | Microservicios, instrumentación, deployment        |
| **Total**                      | **100** |                                                   |
