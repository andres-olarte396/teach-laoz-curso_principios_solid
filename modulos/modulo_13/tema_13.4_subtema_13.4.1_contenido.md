# Tema 13.4: Observability y Monitoring en Arquitecturas Cloud-Native

---

## Introducción

En arquitecturas distribuidas modernas (microservicios, serverless, Kubernetes), un sistema puede tener cientos de componentes ejecutándose simultáneamente. Cuando algo falla, necesitamos **visibilidad completa** para diagnosticar problemas rápidamente.

**Observability** (observabilidad) es la capacidad de entender el **estado interno** de un sistema basándose en sus **salidas externas** (logs, metrics, traces).

---

## Los Tres Pilares de la Observabilidad

```
Observability = Metrics + Logs + Traces
```

### 1. Metrics (Métricas)

**Números agregados** que representan el comportamiento del sistema en el tiempo.

**Ejemplos**:

- CPU usage: 75%
- Request rate: 1000 req/seg
- Error rate: 0.5%
- Response time (p95): 200ms

**Ventaja**: Eficiente para almacenar y consultar. Ideal para dashboards y alertas.

### 2. Logs (Registros)

**Eventos discretos** que describen lo que ocurrió en un momento específico.

**Ejemplo**:

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "ERROR",
  "service": "payment-service",
  "message": "Payment failed: Insufficient funds",
  "userId": "12345",
  "amount": 99.99
}
```

**Ventaja**: Contexto detallado. Útil para debugging.

### 3. Traces (Trazas)

**Ruta completa** de una request a través de múltiples servicios.

**Ejemplo**:

```
User Request (500ms total)
    ↓ 10ms
Frontend Service
    ↓ 200ms
Backend API
    ↓ 50ms
Auth Service
    ↓ 240ms
Database Query
```

**Ventaja**: Muestra dependencias y cuellos de botella.

---

## Prometheus: Metrics Collection

**Prometheus** es el estándar de facto para métricas en Kubernetes.

### Architecture

```
Prometheus Server
    ↓ (pull metrics every 15s)
Service Metrics Endpoint (:9090/metrics)
    ↑ (expose metrics)
Application (instrumented)
```

**Prometheus pull model**: Prometheus **scrape** (extrae) métricas de endpoints HTTP.

### Tipos de Métricas

#### 1. Counter

**Valor que solo incrementa**. Se resetea a 0 al reiniciar.

**Ejemplos**:

- Total requests: `http_requests_total`
- Total errors: `http_errors_total`

**Uso**: Calcular **rate** (tasa de cambio).

```promql
# Request rate (requests por segundo)
rate(http_requests_total[5m])

# Error rate
rate(http_errors_total[5m]) / rate(http_requests_total[5m])
```

#### 2. Gauge

**Valor que puede subir o bajar**.

**Ejemplos**:

- CPU usage: `node_cpu_usage`
- Memory usage: `node_memory_used_bytes`
- Active connections: `http_connections_active`

**Uso**: Valor actual.

```promql
# Memory usage
node_memory_used_bytes / node_memory_total_bytes * 100
```

#### 3. Histogram

**Distribución de valores** en buckets predefinidos.

**Ejemplo**: Response time en buckets (10ms, 50ms, 100ms, 500ms, 1s).

```
http_request_duration_seconds_bucket{le="0.01"} 100
http_request_duration_seconds_bucket{le="0.05"} 450
http_request_duration_seconds_bucket{le="0.1"} 800
http_request_duration_seconds_bucket{le="0.5"} 950
http_request_duration_seconds_bucket{le="1.0"} 990
```

**Uso**: Calcular percentiles (p95, p99).

```promql
# p95 latency
histogram_quantile(0.95, http_request_duration_seconds_bucket)
```

#### 4. Summary

Similar a Histogram pero calcula percentiles en el **cliente** (no en Prometheus).

**Uso**: Cuando necesitas percentiles exactos.

---

## Instrumentar Aplicación con Prometheus

### Node.js (TypeScript)

```typescript
// src/metrics.ts
import promClient from "prom-client";

const register = new promClient.Registry();

// Default metrics (CPU, memory, etc.)
promClient.collectDefaultMetrics({ register });

// Custom counter
export const httpRequestsTotal = new promClient.Counter({
  name: "http_requests_total",
  help: "Total HTTP requests",
  labelNames: ["method", "route", "status"],
  registers: [register],
});

// Custom histogram
export const httpRequestDuration = new promClient.Histogram({
  name: "http_request_duration_seconds",
  help: "HTTP request duration in seconds",
  labelNames: ["method", "route", "status"],
  buckets: [0.01, 0.05, 0.1, 0.5, 1.0, 2.0],
  registers: [register],
});

// Endpoint para Prometheus
export const metricsHandler = async (req, res) => {
  res.set("Content-Type", register.contentType);
  res.end(await register.metrics());
};
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

// Middleware para capturar métricas
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

// API endpoints
app.get("/api/products", async (req, res) => {
  const products = await getProducts();
  res.json(products);
});

// Metrics endpoint
app.get("/metrics", metricsHandler);

app.listen(8080, () => console.log("Server running on :8080"));
```

**Prometheus scrape config**:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: "myapp"
    static_configs:
      - targets: ["myapp-service:8080"]
    metrics_path: "/metrics"
    scrape_interval: 15s
```

**Resultado**: Prometheus extrae métricas de `http://myapp-service:8080/metrics` cada 15 segundos.

---

## PromQL: Prometheus Query Language

### Queries Básicos

```promql
# Tasa de requests por segundo
rate(http_requests_total[5m])

# Error rate (porcentaje)
rate(http_errors_total[5m]) / rate(http_requests_total[5m]) * 100

# p95 latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Memory usage
node_memory_used_bytes / node_memory_total_bytes * 100

# CPU usage por core
rate(node_cpu_seconds_total{mode!="idle"}[5m])
```

### Filtering y Aggregation

```promql
# Requests por route
sum by (route) (rate(http_requests_total[5m]))

# Error rate por servicio
sum by (service) (rate(http_errors_total[5m])) / sum by (service) (rate(http_requests_total[5m]))

# p99 latency por route
histogram_quantile(0.99, sum by (le, route) (rate(http_request_duration_seconds_bucket[5m])))
```

---

## Grafana: Visualización de Métricas

**Grafana** convierte queries de Prometheus en dashboards visuales.

### Instalación en Kubernetes

```yaml
# grafana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:latest
          ports:
            - containerPort: 3000
          env:
            - name: GF_SECURITY_ADMIN_PASSWORD
              value: "admin"

---
apiVersion: v1
kind: Service
metadata:
  name: grafana
spec:
  type: NodePort
  selector:
    app: grafana
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30030
```

### Crear Dashboard

1. **Add Data Source**: Prometheus (http://prometheus:9090)
2. **Create Dashboard** → **Add Panel**
3. **Query**:

```promql
rate(http_requests_total[5m])
```

4. **Visualización**: Time series graph
5. **Legend**: `{{method}} - {{route}}`

**Panel types**:

- Time series: Gráfico de línea temporal
- Gauge: Medidor (CPU, memory)
- Stat: Número grande (total requests)
- Table: Tabla con datos

---

## Distributed Tracing con Jaeger

**Jaeger** captura el flujo completo de una request a través de múltiples servicios.

### Architecture

```
Application (instrumented)
    ↓ (send spans)
Jaeger Agent (sidecar)
    ↓ (forward spans)
Jaeger Collector
    ↓ (store traces)
Storage (Cassandra, Elasticsearch)
    ↑ (query traces)
Jaeger UI
```

### Instrumentar con OpenTelemetry

**OpenTelemetry** es el estándar para tracing (reemplaza a OpenTracing).

```typescript
// src/tracing.ts
import { NodeTracerProvider } from "@opentelemetry/sdk-trace-node";
import { JaegerExporter } from "@opentelemetry/exporter-jaeger";
import { registerInstrumentations } from "@opentelemetry/instrumentation";
import { HttpInstrumentation } from "@opentelemetry/instrumentation-http";
import { ExpressInstrumentation } from "@opentelemetry/instrumentation-express";

const provider = new NodeTracerProvider();

const jaegerExporter = new JaegerExporter({
  endpoint: "http://jaeger-collector:14268/api/traces",
});

provider.addSpanProcessor(
  new SimpleSpanProcessor(jaegerExporter)
);

provider.register();

// Auto-instrument HTTP and Express
registerInstrumentations({
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
  ],
});

console.log("Tracing initialized");
```

```typescript
// src/app.ts
import "./tracing"; // Inicializar antes de imports
import express from "express";
import fetch from "node-fetch";

const app = express();

app.get("/api/orders/:id", async (req, res) => {
  const orderId = req.params.id;

  // Call inventory service
  const inventory = await fetch(
    `http://inventory-service/check/${orderId}`
  ).then((r) => r.json());

  // Call payment service
  const payment = await fetch(
    `http://payment-service/process/${orderId}`
  ).then((r) => r.json());

  res.json({ orderId, inventory, payment });
});

app.listen(8080);
```

**Resultado**: Jaeger captura automáticamente:

```
Trace ID: abc123 (total: 450ms)
    ↓
Span 1. GET /api/orders/123 (frontend-service) - 450ms
    ↓
Span 2: GET /check/123 (inventory-service) - 80ms
    ↓
Span 3: GET /process/123 (payment-service) - 320ms
        ↓
        Span 4: Database query (payment-service) - 280ms
```

**En Jaeger UI** podemos ver visualmente el trace completo y detectar que `payment-service` tiene un query lento.

---

## Log Aggregation: ELK Stack

**ELK** = **Elasticsearch** + **Logstash** + **Kibana**

**Alternativa moderna**: **Loki** (como Prometheus pero para logs).

### Structured Logging

**Mal** (log no estructurado):

```javascript
console.log("User 12345 logged in at 2024-01-15");
```

**Bien** (log estructurado JSON):

```javascript
logger.info({
  event: "user_login",
  userId: "12345",
  timestamp: new Date().toISOString(),
  ip: "192.168.1.100",
});
```

**Ventaja**: Los logs JSON se pueden consultar eficientemente en Elasticsearch.

### Winston Logger (Node.js)

```typescript
// src/logger.ts
import winston from "winston";

export const logger = winston.createLogger({
  level: "info",
  format: winston.format.json(),
  defaultMeta: { service: "backend-api" },
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: "app.log" }),
  ],
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

  // ... process order

  res.json({ success: true });
});
```

**Output**:

```json
{
  "level": "info",
  "message": "",
  "event": "order_created",
  "orderId": "12345",
  "userId": "67890",
  "amount": 99.99,
  "service": "backend-api",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

### Loki con Promtail

**Loki**: Sistema de agregación de logs (push model).

**Promtail**: Agente que envía logs a Loki.

```yaml
# promtail-config.yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
```

**Query logs en Grafana**:

```logql
{app="backend-api"} |= "error"
```

Muestra todos los logs del `backend-api` que contienen "error".

---

## Alerting con Prometheus AlertManager

### Definir Alertas

```yaml
# prometheus-alerts.yaml
groups:
  - name: api_alerts
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: |
          rate(http_errors_total[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} on {{ $labels.instance }}"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High p95 latency"
          description: "p95 latency is {{ $value }}s on {{ $labels.route }}"

      - alert: PodDown
        expr: |
          up{job="myapp"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Pod is down"
          description: "{{ $labels.instance }} has been down for 2 minutes"
```

### AlertManager Configuration

```yaml
# alertmanager.yaml
global:
  resolve_timeout: 5m

route:
  group_by: ["alertname"]
  receiver: "slack"
  routes:
    - match:
        severity: critical
      receiver: pagerduty

receivers:
  - name: "slack"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/XXX/YYY/ZZZ"
        channel: "#alerts"
        text: "{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}"

  - name: "pagerduty"
    pagerduty_configs:
      - service_key: "YOUR_PAGERDUTY_KEY"
```

**Comportamiento**:

- Alertas con `severity: critical` → PagerDuty (despierta al on-call)
- Otras alertas → Slack

---

## SLO/SLI/SLA

### Service Level Indicator (SLI)

**Métrica** que mide aspecto del servicio.

**Ejemplos**:

- Availability: 99.9% de requests exitosas
- Latency: p95 < 200ms
- Durability: 99.999% de datos no perdidos

### Service Level Objective (SLO)

**Target** para un SLI.

**Ejemplo**:

- SLI: Availability
- SLO: 99.9% de requests exitosas en ventana de 30 días

### Service Level Agreement (SLA)

**Contrato** con el cliente. Si no cumples SLO, hay penalización.

**Ejemplo**:

- SLA: 99.9% uptime mensual
- Si no cumples: 10% reembolso

### Error Budget

**Error budget** = Tiempo permitido de downtime según SLO.

**Ejemplo**:

- SLO: 99.9% uptime
- Error budget: 0.1% downtime = 43 minutos/mes

Si consumes todo el error budget, **congelas deploys** hasta próximo mes.

---

## Ejemplo Completo: Observability Stack

```yaml
# observability-stack.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring

---
# Prometheus
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus:latest
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
          ports:
            - containerPort: 9090

---
# Grafana
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:latest
          ports:
            - containerPort: 3000

---
# Jaeger
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:latest
          ports:
            - containerPort: 16686 # UI
            - containerPort: 14268 # Collector
```

---

## Conclusión

**Observability** es esencial en sistemas distribuidos modernos.

**Tres pilares**:

1. **Metrics** (Prometheus): Números agregados, dashboards, alertas
2. **Logs** (Loki, ELK): Eventos detallados, debugging
3. **Traces** (Jaeger): Flujo completo de requests, dependencias

**Best practices**:

✅ Instrumenta todas las aplicaciones (métricas, logs, traces)  
✅ Usa structured logging (JSON)  
✅ Define SLOs y error budgets  
✅ Alerta solo en violaciones de SLO (evita alert fatigue)  
✅ Dashboards en Grafana para todo el equipo

**Próximo paso**: Implementar observability desde día 1 en tus proyectos.
