# Guion de Video: Observability y Monitoring

**Duración**: 35 minutos  
**Objetivo**: Explicar los tres pilares de observability (metrics, logs, traces) con Prometheus, Grafana y Jaeger

---

## [00:00-01:30] Introducción

Bienvenidos al módulo de **Observability y Monitoring**, el último pilar de arquitecturas cloud-native.

En sistemas monolíticos, debugging era simple: revisar logs del server. En microservicios con cientos de pods, necesitamos **observability**.

**Observability**: Capacidad de entender estado interno de un sistema basándose en sus salidas externas.

**Tres pilares**:

1. **Metrics**: Números agregados (CPU, request rate)
2. **Logs**: Eventos discretos con contexto
3. **Traces**: Flujo completo de requests

**Stack que usaremos**:

- Prometheus: Metrics
- Grafana: Visualización
- Jaeger: Distributed tracing
- Loki: Log aggregation

---

## [01:30-06:00] Prometheus: Metrics Collection

### ¿Qué es Prometheus?

**Prometheus** es el estándar para métricas en Kubernetes.

**Architecture**:

```
Prometheus Server (pull model)
    ↓ (scrape every 15s)
Application :8080/metrics
    ↑ (expose metrics)
prom-client library
```

### Tipos de Métricas

#### 1. Counter

Valor que **solo incrementa**.

**Ejemplo**: `http_requests_total`

```typescript
import promClient from "prom-client";

const httpRequestsTotal = new promClient.Counter({
  name: "http_requests_total",
  help: "Total HTTP requests",
  labelNames: ["method", "route", "status"],
});

// Increment
httpRequestsTotal.inc({ method: "GET", route: "/products", status: 200 });
```

**Uso**: Calcular **rate** (tasa de cambio).

```promql
rate(http_requests_total[5m])
```

#### 2. Gauge

Valor que **sube o baja**.

**Ejemplo**: `memory_used_bytes`

```typescript
const memoryUsed = new promClient.Gauge({
  name: "memory_used_bytes",
  help: "Memory usage in bytes",
});

// Set value
memoryUsed.set(process.memoryUsage().heapUsed);
```

#### 3. Histogram

Distribución en buckets.

**Ejemplo**: Response time.

```typescript
const httpDuration = new promClient.Histogram({
  name: "http_request_duration_seconds",
  help: "HTTP request duration",
  labelNames: ["method", "route"],
  buckets: [0.01, 0.05, 0.1, 0.5, 1.0],
});

// Observe
httpDuration.observe({ method: "GET", route: "/products" }, 0.15);
```

**Uso**: Calcular percentiles.

```promql
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

---

## [06:00-11:00] Instrumentar Aplicación

### Demo: API con Prometheus

```typescript
// src/metrics.ts
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
  help: "Request duration",
  labelNames: ["method", "route"],
  buckets: [0.01, 0.05, 0.1, 0.5, 1.0],
  registers: [register],
});

export async function metricsHandler(req, res) {
  res.set("Content-Type", register.contentType);
  res.end(await register.metrics());
}
```

```typescript
// src/app.ts
import express from "express";
import {
  httpRequestsTotal,
  httpRequestDuration,
  metricsHandler,
} from "./metrics";

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
      },
      duration
    );
  });

  next();
});

app.get("/api/products", (req, res) => {
  res.json([{ id: 1, name: "Laptop" }]);
});

app.get("/metrics", metricsHandler);

app.listen(8080);
```

**Ejecutar**:

```bash
npm install express prom-client
ts-node src/app.ts

# Test
curl http://localhost:8080/api/products

# Metrics endpoint
curl http://localhost:8080/metrics
```

**Output**:

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",route="/api/products",status="200"} 1

# HELP http_request_duration_seconds Request duration
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.01",method="GET",route="/api/products"} 1
http_request_duration_seconds_bucket{le="0.05",method="GET",route="/api/products"} 1
```

**Prometheus config**:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: "api"
    static_configs:
      - targets: ["localhost:8080"]
    metrics_path: "/metrics"
    scrape_interval: 15s
```

**Ejecutar Prometheus**:

```bash
docker run -p 9090:9090 -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus

# Abrir
open http://localhost:9090
```

**Queries**:

```promql
# Request rate
rate(http_requests_total[5m])

# p95 latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

---

## [11:00-16:00] Grafana: Dashboards

**Grafana** visualiza métricas de Prometheus.

### Setup

```bash
docker run -d -p 3000:3000 grafana/grafana

# Abrir
open http://localhost:3000
# Login: admin / admin
```

### Configurar Data Source

1. **Configuration** → **Data Sources** → **Add data source**
2. Seleccionar **Prometheus**
3. URL: `http://host.docker.internal:9090` (Mac/Windows) o `http://prometheus:9090` (Linux)
4. **Save & Test**

### Crear Dashboard

#### Panel 1. Request Rate

1. **Create Dashboard** → **Add Panel**
2. **Query**:

```promql
rate(http_requests_total[1m])
```

3. **Panel options**:
   - Title: "Request Rate"
   - Unit: "reqps"
   - Legend: `{{method}} {{route}}`

#### Panel 2: Error Rate (Gauge)

1. **Add Panel**
2. **Query**:

```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100
```

3. **Visualization**: Gauge
4. **Thresholds**:
   - Green: 0-2
   - Yellow: 2-5
   - Red: >5
5. **Unit**: percent (0-100)

#### Panel 3: p95 Latency

1. **Add Panel**
2. **Query**:

```promql
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[1m]))
```

3. **Unit**: seconds
4. **Legend**: `p95 {{route}}`

**Resultado**: Dashboard con 3 paneles mostrando request rate, error rate y latency en tiempo real.

---

## [16:00-22:00] Distributed Tracing con Jaeger

**Problema**: En microservicios, una request puede pasar por 10+ servicios. ¿Cómo identificar dónde está el cuello de botella?

**Solución**: **Distributed Tracing**.

### OpenTelemetry

**OpenTelemetry** es el estándar (reemplaza OpenTracing y OpenCensus).

### Demo: 3 Microservicios

```
Frontend → Backend → Database
```

#### Frontend Service

```typescript
// frontend/src/tracing.ts
import { NodeTracerProvider } from "@opentelemetry/sdk-trace-node";
import { JaegerExporter } from "@opentelemetry/exporter-jaeger";
import { SimpleSpanProcessor } from "@opentelemetry/sdk-trace-base";
import { registerInstrumentations } from "@opentelemetry/instrumentation";
import { HttpInstrumentation } from "@opentelemetry/instrumentation-http";

const provider = new NodeTracerProvider({
  resource: {
    attributes: {
      "service.name": "frontend-service",
    },
  },
});

const jaegerExporter = new JaegerExporter({
  endpoint: "http://localhost:14268/api/traces",
});

provider.addSpanProcessor(new SimpleSpanProcessor(jaegerExporter));
provider.register();

registerInstrumentations({
  instrumentations: [new HttpInstrumentation()],
});
```

```typescript
// frontend/src/app.ts
import "./tracing"; // IMPORTANTE: antes de imports
import express from "express";
import fetch from "node-fetch";

const app = express();

app.get("/api/data", async (req, res) => {
  const data = await fetch("http://localhost:8081/backend").then((r) =>
    r.json()
  );
  res.json(data);
});

app.listen(8080);
```

#### Backend Service

```typescript
// backend/src/tracing.ts
import { NodeTracerProvider } from "@opentelemetry/sdk-trace-node";
import { JaegerExporter } from "@opentelemetry/exporter-jaeger";
import { SimpleSpanProcessor } from "@opentelemetry/sdk-trace-base";
import { registerInstrumentations } from "@opentelemetry/instrumentation";
import { HttpInstrumentation } from "@opentelemetry/instrumentation-http";

const provider = new NodeTracerProvider({
  resource: {
    attributes: {
      "service.name": "backend-service",
    },
  },
});

const jaegerExporter = new JaegerExporter({
  endpoint: "http://localhost:14268/api/traces",
});

provider.addSpanProcessor(new SimpleSpanProcessor(jaegerExporter));
provider.register();

registerInstrumentations({
  instrumentations: [new HttpInstrumentation()],
});
```

```typescript
// backend/src/app.ts
import "./tracing";
import express from "express";
import fetch from "node-fetch";

const app = express();

app.get("/backend", async (req, res) => {
  const dbData = await fetch("http://localhost:8082/database").then((r) =>
    r.json()
  );
  res.json({ backend: true, data: dbData });
});

app.listen(8081);
```

#### Database Service

```typescript
// database/src/app.ts
import "./tracing";
import express from "express";

const app = express();

app.get("/database", async (req, res) => {
  // Simular query lento
  await new Promise((resolve) => setTimeout(resolve, 200));
  res.json({ database: "PostgreSQL", records: 1000 });
});

app.listen(8082);
```

**Ejecutar Jaeger**:

```bash
docker run -d -p 16686:16686 -p 14268:14268 jaegertracing/all-in-one:latest
```

**Ejecutar servicios**:

```bash
cd frontend && npm start &
cd backend && npm start &
cd database && npm start &

# Test
curl http://localhost:8080/api/data
```

**Abrir Jaeger UI**:

```bash
open http://localhost:16686
```

**Buscar traces**:

1. Service: `frontend-service`
2. Click en trace
3. Verás:

```
Trace ID: abc123 (total: 220ms)
    ↓
frontend-service: GET /api/data (220ms)
    ↓
backend-service: GET /backend (210ms)
    ↓
database-service: GET /database (200ms)
```

**Insight**: `database-service` toma 200ms de 220ms total (90%). Optimizar query.

---

## [22:00-27:00] Structured Logging con Loki

**Problema**: Logs planos dificultan búsqueda.

**Solución**: **Structured logging** (JSON).

### Winston Logger

```typescript
// src/logger.ts
import winston from "winston";

export const logger = winston.createLogger({
  level: "info",
  format: winston.format.json(),
  defaultMeta: { service: "backend-api" },
  transports: [new winston.transports.Console()],
});
```

```typescript
// src/app.ts
import { logger } from "./logger";

app.post("/api/orders", async (req, res) => {
  const order = req.body;

  logger.info({
    event: "order_created",
    orderId: order.id,
    userId: order.userId,
    amount: order.total,
  });

  // ... process

  res.json({ success: true });
});

app.use((err, req, res, next) => {
  logger.error({
    event: "unhandled_error",
    error: err.message,
    stack: err.stack,
    route: req.path,
  });

  res.status(500).json({ error: "Internal server error" });
});
```

**Output**:

```json
{
  "level": "info",
  "event": "order_created",
  "orderId": "12345",
  "userId": "67890",
  "amount": 99.99,
  "service": "backend-api",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

### Loki

**Loki**: Sistema de logs de Grafana (como Prometheus para logs).

```bash
docker run -d -p 3100:3100 grafana/loki:latest
```

**Configurar en Grafana**:

1. **Data Sources** → **Add Loki**
2. URL: `http://localhost:3100`

**Query logs**:

```logql
{service="backend-api"} |= "error"
```

Muestra todos los logs con "error".

**Filter por field**:

```logql
{service="backend-api"} | json | userId="67890"
```

---

## [27:00-31:00] Alertas con Prometheus AlertManager

**Objetivo**: Alerta cuando error rate > 5% o latency > 1s.

### Definir Alertas

```yaml
# alerts.yml
groups:
  - name: api_alerts
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High p95 latency"
          description: "p95 is {{ $value }}s"
```

### AlertManager

```yaml
# alertmanager.yml
route:
  receiver: "slack"

receivers:
  - name: "slack"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/XXX"
        channel: "#alerts"
        text: "{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}"
```

**Resultado**: Cuando error rate > 5% por más de 5 minutos, mensaje en Slack.

---

## [31:00-35:00] SLO y Error Budget

### SLO (Service Level Objective)

**SLO**: Target de calidad.

**Ejemplo**: 99.9% uptime mensual.

### Error Budget

**Error Budget**: Tiempo permitido de downtime.

**Cálculo**:

- SLO: 99.9%
- Error budget: 0.1% = **43 minutos/mes**

**Uso**:

- Si consumes todo el error budget en semana 1 → **congelas deploys** hasta recuperarte.
- Balancea innovación (deploys) con confiabilidad.

### SLI (Service Level Indicator)

**SLI**: Métrica que mide SLO.

**Ejemplos**:

- Availability SLI: `sum(http_requests_total{status!="5xx"}) / sum(http_requests_total)`
- Latency SLI: `histogram_quantile(0.95, http_request_duration_seconds_bucket) < 0.2`

---

## [35:00-37:00] Resumen

**Tres pilares de observability**:

1. **Metrics** (Prometheus):
   - Números agregados
   - Dashboards, alertas
   - **Uso**: Monitoreo en tiempo real

2. **Logs** (Loki):
   - Eventos detallados
   - **Uso**: Debugging, auditoría

3. **Traces** (Jaeger):
   - Flujo completo de requests
   - **Uso**: Identificar cuellos de botella

**Stack completo**:

- **Prometheus**: Métricas
- **Grafana**: Visualización (métricas, logs, traces)
- **Jaeger**: Distributed tracing
- **Loki**: Log aggregation
- **AlertManager**: Alertas

**Best practices**:

✅ Instrumenta desde día 1  
✅ Usa structured logging (JSON)  
✅ Define SLOs y error budgets  
✅ Alerta solo en violaciones de SLO  
✅ Dashboard unificado en Grafana

**Con esto finalizamos el curso de Principios SOLID y Arquitecturas Cloud-Native.**

Has aprendido desde SOLID hasta Kubernetes, Serverless e Observability. Ahora puedes diseñar sistemas escalables, resilientes y mantenibles.

**Gracias por tu tiempo y dedicación.**
