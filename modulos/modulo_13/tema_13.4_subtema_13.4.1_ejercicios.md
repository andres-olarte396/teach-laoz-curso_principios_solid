# Ejercicios: Observability y Monitoring

---

## ⭐ Ejercicio 1. Instrumentar API con Prometheus

**Objetivo**: Exponer métricas de una API Node.js y visualizarlas en Prometheus.

### Requisitos

1. API REST con endpoints `/products`, `/orders`
2. Métricas custom:
   - `http_requests_total` (counter)
   - `http_request_duration_seconds` (histogram)
   - `active_connections` (gauge)
3. Endpoint `/metrics` para Prometheus
4. Prometheus scrapeando cada 15 segundos

### Implementación

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
  help: "HTTP request duration",
  labelNames: ["method", "route", "status"],
  buckets: [0.01, 0.05, 0.1, 0.5, 1.0, 2.0, 5.0],
  registers: [register],
});

export const activeConnections = new promClient.Gauge({
  name: "http_active_connections",
  help: "Active HTTP connections",
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
  activeConnections,
  metricsHandler,
} from "./metrics";

const app = express();

let connections = 0;

app.use((req, res, next) => {
  connections++;
  activeConnections.set(connections);

  const start = Date.now();

  res.on("finish", () => {
    connections--;
    activeConnections.set(connections);

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

app.get("/api/products", async (req, res) => {
  // Simular delay variable
  await new Promise((resolve) => setTimeout(resolve, Math.random() * 500));
  res.json([
    { id: 1, name: "Laptop" },
    { id: 2, name: "Mouse" },
  ]);
});

app.get("/api/orders", async (req, res) => {
  // Simular fallo ocasional
  if (Math.random() < 0.1) {
    res.status(500).json({ error: "Database connection failed" });
    return;
  }

  await new Promise((resolve) => setTimeout(resolve, Math.random() * 300));
  res.json([{ id: 1, total: 99.99 }]);
});

app.get("/metrics", metricsHandler);

app.listen(8080, () => console.log("Server on :8080"));
```

**Prometheus config**:

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "api"
    static_configs:
      - targets: ["localhost:8080"]
    metrics_path: "/metrics"
```

**Ejecutar**:

```bash
# Terminal 1. API
npm install express prom-client
ts-node src/app.ts

# Terminal 2: Prometheus
docker run -p 9090:9090 -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus

# Terminal 3: Generar tráfico
for i in {1..100}; do
  curl http://localhost:8080/api/products
  curl http://localhost:8080/api/orders
done

# Abrir Prometheus
open http://localhost:9090
```

**Queries en Prometheus**:

```promql
# Request rate
rate(http_requests_total[1m])

# Error rate
rate(http_requests_total{status=~"5.."}[1m]) / rate(http_requests_total[1m])

# p95 latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[1m]))

# Active connections
http_active_connections
```

---

## ⭐⭐ Ejercicio 2: Dashboard en Grafana

**Objetivo**: Crear dashboard completo en Grafana con alertas.

### Requisitos

1. 4 paneles:
   - Request rate (time series)
   - Error rate (gauge)
   - p95 latency (time series)
   - Top 5 slowest routes (table)
2. Alertas:
   - Error rate > 5%
   - p95 latency > 1s

### Setup Grafana

```bash
docker run -d -p 3000:3000 --name=grafana grafana/grafana

# Abrir Grafana
open http://localhost:3000
# Login: admin / admin
```

### Panel 1. Request Rate

1. **Add Panel** → **Add new panel**
2. **Query**:

```promql
sum(rate(http_requests_total[1m])) by (route)
```

3. **Panel options**:
   - Title: "Request Rate"
   - Unit: "reqps" (requests per second)
   - Legend: `{{route}}`

### Panel 2: Error Rate

1. **Add Panel**
2. **Query**:

```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100
```

3. **Visualization**: Gauge
4. **Thresholds**:
   - Green: 0-2%
   - Yellow: 2-5%
   - Red: >5%

### Panel 3: p95 Latency

1. **Add Panel**
2. **Query**:

```promql
histogram_quantile(0.95, sum by (le, route) (rate(http_request_duration_seconds_bucket[1m])))
```

3. **Unit**: seconds
4. **Legend**: `p95 {{route}}`

### Panel 4: Slowest Routes

1. **Add Panel**
2. **Query**:

```promql
topk(5, histogram_quantile(0.95, sum by (le, route) (rate(http_request_duration_seconds_bucket[5m]))))
```

3. **Visualization**: Table
4. **Columns**: Route, Latency

### Crear Alerta

1. Panel "Error Rate" → **Alert** tab
2. **Condition**:

```
WHEN avg() OF query(A, 5m, now) IS ABOVE 5
```

3. **Contact point**: Email o Slack
4. **Summary**: "High error rate detected"

---

## ⭐⭐⭐ Ejercicio 3: Distributed Tracing con Jaeger

**Objetivo**: Instrumentar sistema de 3 microservicios con OpenTelemetry y visualizar traces en Jaeger.

### Arquitectura

```
Frontend Service
    ↓
Backend API Service
    ↓ (parallel)
    ├─→ Database Service
    └─→ Cache Service
```

### Frontend Service

```typescript
// frontend/src/tracing.ts
import { NodeTracerProvider } from "@opentelemetry/sdk-trace-node";
import { JaegerExporter } from "@opentelemetry/exporter-jaeger";
import { SimpleSpanProcessor } from "@opentelemetry/sdk-trace-base";
import { registerInstrumentations } from "@opentelemetry/instrumentation";
import { HttpInstrumentation } from "@opentelemetry/instrumentation-http";
import { ExpressInstrumentation } from "@opentelemetry/instrumentation-express";

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
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
  ],
});
```

```typescript
// frontend/src/app.ts
import "./tracing";
import express from "express";
import fetch from "node-fetch";

const app = express();

app.get("/api/dashboard", async (req, res) => {
  const userId = req.query.userId;

  const data = await fetch(
    `http://backend-service:8081/user/${userId}`
  ).then((r) => r.json());

  res.json(data);
});

app.listen(8080);
```

### Backend API Service

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

app.get("/user/:userId", async (req, res) => {
  const userId = req.params.userId;

  // Parallel calls
  const [userData, cacheData] = await Promise.all([
    fetch(`http://database-service:8082/user/${userId}`).then((r) =>
      r.json()
    ),
    fetch(`http://cache-service:8083/cache/${userId}`).then((r) => r.json()),
  ]);

  res.json({ user: userData, cache: cacheData });
});

app.listen(8081);
```

### Database Service

```typescript
// database/src/app.ts
import "./tracing";
import express from "express";

const app = express();

app.get("/user/:userId", async (req, res) => {
  // Simular query lento
  await new Promise((resolve) => setTimeout(resolve, 200));

  res.json({
    id: req.params.userId,
    name: "Alice",
    email: "alice@example.com",
  });
});

app.listen(8082);
```

### Cache Service

```typescript
// cache/src/app.ts
import "./tracing";
import express from "express";

const app = express();

app.get("/cache/:userId", async (req, res) => {
  // Simular cache hit rápido
  await new Promise((resolve) => setTimeout(resolve, 10));

  res.json({ cached: true, timestamp: Date.now() });
});

app.listen(8083);
```

**Ejecutar**:

```bash
# Jaeger
docker run -d -p 16686:16686 -p 14268:14268 jaegertracing/all-in-one:latest

# Services
cd frontend && npm start &
cd backend && npm start &
cd database && npm start &
cd cache && npm start &

# Test
curl "http://localhost:8080/api/dashboard?userId=123"

# Abrir Jaeger
open http://localhost:16686
```

**En Jaeger UI**:

1. Service: `frontend-service`
2. Buscar traces
3. Verás trace completo:

```
frontend-service: GET /api/dashboard (220ms)
    ↓
backend-service: GET /user/123 (210ms)
    ↓ (parallel)
    ├─→ database-service: GET /user/123 (200ms)
    └─→ cache-service: GET /cache/123 (10ms)
```

**Insight**: `database-service` es el cuello de botella (200ms).

---

## ⭐⭐⭐⭐ Ejercicio 4: Observability Stack Completo

**Objetivo**: Implementar Prometheus + Grafana + Jaeger + Loki en Kubernetes.

### Arquitectura

```
Application Pods
    ↓ metrics → Prometheus
    ↓ logs → Loki (via Promtail)
    ↓ traces → Jaeger

Grafana (visualiza todo)
```

### Kubernetes Deployment

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring

---
# prometheus.yaml
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
          volumeMounts:
            - name: config
              mountPath: /etc/prometheus
      volumes:
        - name: config
          configMap:
            name: prometheus-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true

---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  selector:
    app: prometheus
  ports:
    - port: 9090
      targetPort: 9090

---
# grafana.yaml
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
          env:
            - name: GF_SECURITY_ADMIN_PASSWORD
              value: "admin"

---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: grafana
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30300

---
# jaeger.yaml
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

---
apiVersion: v1
kind: Service
metadata:
  name: jaeger
  namespace: monitoring
spec:
  selector:
    app: jaeger
  ports:
    - name: ui
      port: 16686
      targetPort: 16686
    - name: collector
      port: 14268
      targetPort: 14268

---
# loki.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: loki
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: loki
  template:
    metadata:
      labels:
        app: loki
    spec:
      containers:
        - name: loki
          image: grafana/loki:latest
          args:
            - "-config.file=/etc/loki/local-config.yaml"
          ports:
            - containerPort: 3100

---
apiVersion: v1
kind: Service
metadata:
  name: loki
  namespace: monitoring
spec:
  selector:
    app: loki
  ports:
    - port: 3100
      targetPort: 3100
```

### Application with Observability

```yaml
# app-deployment.yaml
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
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: myapp
          image: myapp:latest
          ports:
            - containerPort: 8080
          env:
            - name: JAEGER_ENDPOINT
              value: "http://jaeger.monitoring.svc.cluster.local:14268/api/traces"
```

**Deploy**:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f prometheus.yaml
kubectl apply -f grafana.yaml
kubectl apply -f jaeger.yaml
kubectl apply -f loki.yaml
kubectl apply -f app-deployment.yaml

# Port-forward Grafana
kubectl port-forward -n monitoring svc/grafana 3000:3000

# Abrir Grafana
open http://localhost:3000

# Add Data Sources:
# 1. Prometheus: http://prometheus.monitoring.svc.cluster.local:9090
# 2. Loki: http://loki.monitoring.svc.cluster.local:3100
# 3. Jaeger: http://jaeger.monitoring.svc.cluster.local:16686
```

**Grafana Dashboard**:

1. **Metrics Panel**:

```promql
rate(http_requests_total{app="myapp"}[5m])
```

2. **Logs Panel**:

```logql
{app="myapp"} |= "error"
```

3. **Traces**: Jaeger data source, query `myapp`

**Resultado**: Dashboard unificado con métricas, logs y traces.

---

## Resumen

| Ejercicio | Tecnologías                     | Dificultad |
| --------- | ------------------------------- | ---------- |
| 1         | Prometheus, prom-client         | ⭐          |
| 2         | Grafana, dashboards, alertas    | ⭐⭐         |
| 3         | OpenTelemetry, Jaeger, tracing  | ⭐⭐⭐        |
| 4         | Kubernetes, stack completo      | ⭐⭐⭐⭐       |
