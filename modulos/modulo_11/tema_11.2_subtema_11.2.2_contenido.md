# Service Mesh y Dependency Inversion Principle

## Objetivos
- Aplicar DIP en arquitecturas con Service Mesh
- Configurar Istio/Linkerd para separación de concerns
- Implementar Sidecar pattern
- Configurar observability y resilience patterns

## 1. Service Mesh y Separación de Concerns (DIP)

El Service Mesh aplica DIP separando la lógica de infraestructura del código de negocio.

```
Sin Service Mesh (acoplamiento):
┌────────────────────────────┐
│   Order Service            │
│  ┌──────────────────────┐  │
│  │ Business Logic       │  │
│  │ + Circuit Breaker    │  │ ← Mezclado
│  │ + Retry Logic        │  │ ← Mezclado
│  │ + Metrics            │  │ ← Mezclado
│  │ + Tracing            │  │ ← Mezclado
│  └──────────────────────┘  │
└────────────────────────────┘

Con Service Mesh (DIP):
┌────────────────┐    ┌─────────────┐
│ Order Service  │───▶│   Sidecar   │
│ (solo negocio) │    │ (infraestr.)│
└────────────────┘    └─────────────┘
                           │
                      ┌────┴────┐
                      │  Istio  │
                      │ Control │
                      │  Plane  │
                      └─────────┘
```

### Sin Service Mesh (Código Acoplado)

```java
// ❌ MAL: Lógica de negocio mezclada con infraestructura
@Service
public class OrderService {
    
    private final CircuitBreaker circuitBreaker;
    private final MeterRegistry metrics;
    private final Tracer tracer;
    
    public Order createOrder(OrderRequest request) {
        // Tracing manual
        Span span = tracer.buildSpan("createOrder").start();
        
        try {
            // Metrics manual
            metrics.counter("orders.created").increment();
            
            // Circuit breaker manual
            return circuitBreaker.executeSupplier(() -> {
                // Retry manual
                return Retry.decorateSupplier(
                    Retry.ofDefaults("orderRetry"),
                    () -> {
                        // Lógica de negocio enterrada
                        Order order = new Order(request);
                        orderRepository.save(order);
                        
                        // Llamada a servicio con timeout manual
                        CompletableFuture<Void> payment = CompletableFuture
                            .runAsync(() -> paymentClient.process(order))
                            .orTimeout(5, TimeUnit.SECONDS);
                        
                        payment.join();
                        return order;
                    }
                ).get();
            });
        } finally {
            span.finish();
        }
    }
}
```

### Con Service Mesh (DIP Aplicado)

```java
// ✅ BIEN: Solo lógica de negocio
@Service
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final PaymentClient paymentClient;  // Client simple
    
    public Order createOrder(OrderRequest request) {
        // Solo lógica de dominio
        Order order = new Order(request);
        orderRepository.save(order);
        
        // Llamada simple - Sidecar maneja:
        // - Circuit breaking
        // - Retries
        // - Timeouts
        // - Tracing
        // - Metrics
        paymentClient.process(order);
        
        return order;
    }
}

// Client HTTP simple
@FeignClient(name = "payment-service")
public interface PaymentClient {
    @PostMapping("/payments")
    void process(@RequestBody Order order);
}
```

## 2. Istio Configuration (Infraestructura como Configuración)

### Virtual Service (Routing)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - match:
    - headers:
        x-api-version:
          exact: "v2"
    route:
    - destination:
        host: payment-service
        subset: v2
  - route:
    - destination:
        host: payment-service
        subset: v1
```

### Destination Rule (Circuit Breaker, Load Balancing)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 40
    loadBalancer:
      simple: LEAST_CONN
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 50
```

### Retry Policy

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - route:
    - destination:
        host: order-service
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,reset,connect-failure,refused-stream
```

### Timeout Configuration

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: inventory-service
spec:
  hosts:
  - inventory-service
  http:
  - route:
    - destination:
        host: inventory-service
    timeout: 10s
```

## 3. Observability (Metrics, Tracing, Logging)

### Automatic Metrics (Prometheus)

```yaml
# Istio genera métricas automáticamente
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-grafana
data:
  dashboards.yaml: |
    apiVersion: 1
    providers:
    - name: 'Istio'
      folder: 'Istio'
      type: file
      options:
        path: /var/lib/grafana/dashboards/istio
```

**Métricas automáticas**:
- `istio_requests_total`: Total de requests
- `istio_request_duration_milliseconds`: Latencia
- `istio_request_bytes`: Tamaño de request
- `istio_response_bytes`: Tamaño de response

### Distributed Tracing (Jaeger)

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: tracing-default
spec:
  tracing:
  - providers:
    - name: jaeger
    randomSamplingPercentage: 100
    customTags:
      environment:
        literal:
          value: production
```

**El código no necesita cambios**:
```java
// El sidecar inyecta headers automáticamente:
// - x-request-id
// - x-b3-traceid
// - x-b3-spanid
// - x-b3-parentspanid

@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable String id) {
    // Tracing automático - no código manual
    return orderRepository.findById(id)
        .orElseThrow(() -> new NotFoundException());
}
```

### Access Logging

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: access-logging
spec:
  accessLogging:
  - providers:
    - name: envoy
    filter:
      expression: response.code >= 400
```

## 4. Security (mTLS, Authorization)

### Automatic mTLS

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # Todas las comunicaciones requieren mTLS
```

### Authorization Policy

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-authz
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/payment-service"]
    to:
    - operation:
        methods: ["POST"]
        paths: ["/orders/*/confirm"]
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/api-gateway"]
    to:
    - operation:
        methods: ["GET", "POST"]
```

## 5. Resilience Patterns

### Circuit Breaking Example

```python
# Sin Istio: Código manual
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=30)
def call_payment_service(order_id):
    response = requests.post(
        f"{PAYMENT_SERVICE_URL}/payments",
        json={"orderId": order_id},
        timeout=5
    )
    response.raise_for_status()
    return response.json()

# Con Istio: Solo configuración
# order_service.py
def call_payment_service(order_id):
    # Istio sidecar maneja circuit breaking
    response = requests.post(
        "http://payment-service/payments",
        json={"orderId": order_id}
    )
    return response.json()
```

```yaml
# istio-config.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-circuit-breaker
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

### Rate Limiting

```yaml
apiVersion: networking.istio.io/v1beta1
kind: EnvoyFilter
metadata:
  name: rate-limit-filter
spec:
  workloadSelector:
    labels:
      app: order-service
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        filterChain:
          filter:
            name: "envoy.filters.network.http_connection_manager"
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.filters.http.ratelimit
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.http.ratelimit.v3.RateLimit
          domain: order-service-ratelimit
          rate_limit_service:
            grpc_service:
              envoy_grpc:
                cluster_name: rate_limit_service
```

## 6. Traffic Management

### Canary Deployment

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service-canary
spec:
  hosts:
  - order-service
  http:
  - match:
    - headers:
        x-user-type:
          exact: "beta-tester"
    route:
    - destination:
        host: order-service
        subset: v2
      weight: 100
  - route:
    - destination:
        host: order-service
        subset: v1
      weight: 90
    - destination:
        host: order-service
        subset: v2
      weight: 10  # 10% de tráfico a nueva versión
```

### A/B Testing

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: recommendation-ab-test
spec:
  hosts:
  - recommendation-service
  http:
  - match:
    - headers:
        cookie:
          regex: "^(.*?;)?(experiment=new-algorithm)(;.*)?$"
    route:
    - destination:
        host: recommendation-service
        subset: ml-v2
  - route:
    - destination:
        host: recommendation-service
        subset: ml-v1
```

## 7. Ejemplo Completo: E-commerce con Istio

```yaml
# ============================================
# Service Mesh Configuration
# ============================================

# Namespace configuration
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
  labels:
    istio-injection: enabled  # Auto-inject sidecars

---
# Order Service with Resilience
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service
  namespace: ecommerce
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 100
        maxRequestsPerConnection: 5
    outlierDetection:
      consecutiveErrors: 3
      interval: 10s
      baseEjectionTime: 30s
    loadBalancer:
      consistentHash:
        httpCookie:
          name: user-session
          ttl: 3600s

---
# Payment Service with Circuit Breaker
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
  namespace: ecommerce
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 60s
      maxEjectionPercent: 50

---
# Retry policy for transient failures
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: inventory-service
  namespace: ecommerce
spec:
  hosts:
  - inventory-service
  http:
  - route:
    - destination:
        host: inventory-service
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,reset,connect-failure
    timeout: 10s

---
# Security: Only allow specific service-to-service communication
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-to-payment
  namespace: ecommerce
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/ecommerce/sa/order-service"
        - "cluster.local/ns/ecommerce/sa/admin-service"
```

## Resumen

### Beneficios del Service Mesh (DIP)

1. **Separación de Concerns**: Negocio vs Infraestructura
2. **Reutilización**: Configuración aplicable a todos los servicios
3. **Testabilidad**: Servicios sin dependencias de infraestructura
4. **Observability**: Automática sin código
5. **Security**: mTLS transparente
6. **Resilience**: Configuración declarativa

### Service Mesh vs Library

| Aspecto | Library (Resilience4j) | Service Mesh (Istio) |
|---------|------------------------|----------------------|
| **Acoplamiento** | Alto (código) | Bajo (configuración) |
| **Lenguaje** | Específico | Agnóstico |
| **Testing** | Requiere mocks | Independiente |
| **Deployment** | Con app | Infraestructura |
| **Upgrades** | Rebuild app | Rolling update sidecar |
| **Observability** | Manual | Automática |

### Cuándo Usar Service Mesh

✅ **Usar cuando**:
- 10+ microservicios
- Múltiples lenguajes
- Requisitos de observability estrictos
- Comunicación service-to-service compleja
- mTLS requerido

❌ **No usar cuando**:
- < 5 microservicios
- Single language stack
- Equipo pequeño (< 5 personas)
- Complejidad operacional limitada
