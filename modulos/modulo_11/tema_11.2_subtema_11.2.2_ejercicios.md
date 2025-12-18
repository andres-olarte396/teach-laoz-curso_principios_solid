# Ejercicios: Service Mesh y DIP

## ⭐ Ejercicio 1: Migrar de Library a Service Mesh

Tienes este código con Resilience4j:

```java
@Service
public class OrderService {
    
    @CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
    @Retry(name = "payment", maxAttempts = 3)
    @Bulkhead(name = "payment", maxConcurrentCalls = 10)
    public PaymentResult processPayment(Order order) {
        return paymentClient.charge(order.getTotal());
    }
    
    private PaymentResult paymentFallback(Order order, Exception ex) {
        log.error("Payment failed for order {}", order.getId(), ex);
        return PaymentResult.failed("Service unavailable");
    }
}
```

**Tareas**:
1. Eliminar anotaciones de Resilience4j
2. Crear configuración Istio equivalente
3. Simplificar código a solo lógica de negocio

<details>
<summary>💡 Solución</summary>

```java
// Código simplificado (solo negocio)
@Service
public class OrderService {
    
    private final PaymentClient paymentClient;
    
    public PaymentResult processPayment(Order order) {
        // Istio sidecar maneja circuit breaking, retries, bulkhead
        return paymentClient.charge(order.getTotal());
    }
}

// Client simple sin resiliencia
@FeignClient(name = "payment-service")
public interface PaymentClient {
    @PostMapping("/charges")
    PaymentResult charge(@RequestBody BigDecimal amount);
}
```

```yaml
# istio/destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    # Circuit Breaker (equivalente a @CircuitBreaker)
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 40
    
    # Bulkhead (equivalente a @Bulkhead)
    connectionPool:
      http:
        http1MaxPendingRequests: 10  # maxConcurrentCalls
        http2MaxRequests: 10
        maxRequestsPerConnection: 2

---
# istio/virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - route:
    - destination:
        host: payment-service
    # Retry (equivalente a @Retry)
    retries:
      attempts: 3
      perTryTimeout: 5s
      retryOn: 5xx,reset,connect-failure
    timeout: 15s
```

**Despliegue**:
```bash
kubectl apply -f istio/destination-rule.yaml
kubectl apply -f istio/virtual-service.yaml

# Verificar configuración
istioctl proxy-config routes order-service-pod-xyz
```

</details>

---

## ⭐⭐ Ejercicio 2: Configurar Observability Completa

Configura Istio para capturar métricas, traces y logs de un servicio de inventario.

**Requisitos**:
1. Exportar métricas a Prometheus
2. Enviar traces a Jaeger (100% sampling)
3. Logs de access solo para errores 4xx/5xx
4. Custom tags en traces: `customer_id`, `region`

<details>
<summary>💡 Solución</summary>

```yaml
# observability/telemetry.yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: inventory-observability
spec:
  selector:
    matchLabels:
      app: inventory-service
  
  # Distributed Tracing
  tracing:
  - providers:
    - name: jaeger
    randomSamplingPercentage: 100  # 100% sampling
    customTags:
      customer_id:
        header:
          name: x-customer-id
          defaultValue: "unknown"
      region:
        header:
          name: x-region
          defaultValue: "us-east-1"
      http_method:
        literal:
          value: "request.method"
  
  # Metrics
  metrics:
  - providers:
    - name: prometheus
    overrides:
    - match:
        metric: REQUEST_COUNT
      tagOverrides:
        customer_tier:
          value: "request.headers['x-customer-tier']"
    - match:
        metric: REQUEST_DURATION
      tagOverrides:
        operation:
          value: "request.url_path"
  
  # Access Logging (solo errores)
  accessLogging:
  - providers:
    - name: envoy
    filter:
      expression: "response.code >= 400"

---
# observability/service-monitor.yaml (Prometheus)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: inventory-service-metrics
spec:
  selector:
    matchLabels:
      app: inventory-service
  endpoints:
  - port: http-envoy-prom
    interval: 15s
    path: /stats/prometheus

---
# observability/grafana-dashboard.json
{
  "dashboard": {
    "title": "Inventory Service Metrics",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "sum(rate(istio_requests_total{destination_service=\"inventory-service\"}[1m])) by (response_code)"
          }
        ]
      },
      {
        "title": "P95 Latency",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(istio_request_duration_milliseconds_bucket{destination_service=\"inventory-service\"}[1m])) by (le))"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "sum(rate(istio_requests_total{destination_service=\"inventory-service\",response_code=~\"5..\"}[1m])) / sum(rate(istio_requests_total{destination_service=\"inventory-service\"}[1m]))"
          }
        ]
      }
    ]
  }
}
```

**Verificación**:
```bash
# Ver traces en Jaeger
kubectl port-forward -n istio-system svc/jaeger-query 16686:16686
# Abrir http://localhost:16686

# Ver métricas en Prometheus
kubectl port-forward -n istio-system svc/prometheus 9090:9090
# Query: istio_requests_total{destination_service="inventory-service"}

# Ver logs
kubectl logs -n production inventory-service-pod-xyz -c istio-proxy --tail=100
```

</details>

---

## ⭐⭐⭐ Ejercicio 3: Implementar Canary Deployment con Automatic Rollback

Implementa canary deployment para nueva versión de recommendation service.

**Requisitos**:
1. Empezar con 10% tráfico a v2
2. Monitorear error rate cada 2 minutos
3. Si error rate > 5%, rollback automático a v1
4. Si success, incrementar gradualmente: 10% → 25% → 50% → 100%

<details>
<summary>💡 Solución</summary>

```yaml
# canary/virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: recommendation-service
spec:
  hosts:
  - recommendation-service
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: recommendation-service
        subset: v2
      weight: 100
  - route:
    - destination:
        host: recommendation-service
        subset: v1
      weight: 90  # START: 90% v1
    - destination:
        host: recommendation-service
        subset: v2
      weight: 10  # START: 10% v2

---
# canary/destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: recommendation-service
spec:
  host: recommendation-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2

---
# canary/prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: canary-alerts
spec:
  groups:
  - name: canary
    interval: 30s
    rules:
    - alert: CanaryHighErrorRate
      expr: |
        sum(rate(istio_requests_total{
          destination_service="recommendation-service",
          destination_version="v2",
          response_code=~"5.."
        }[2m])) 
        / 
        sum(rate(istio_requests_total{
          destination_service="recommendation-service",
          destination_version="v2"
        }[2m])) 
        > 0.05
      labels:
        severity: critical
      annotations:
        summary: "Canary v2 error rate > 5%"
        description: "Rolling back to v1"
```

```python
# canary/canary-controller.py
import time
from kubernetes import client, config
from prometheus_api_client import PrometheusConnect

config.load_kube_config()
v1 = client.NetworkingV1beta1Api()
prom = PrometheusConnect(url="http://prometheus:9090")

STAGES = [10, 25, 50, 75, 100]
STAGE_DURATION = 120  # 2 minutes

def get_error_rate(version):
    query = f'''
    sum(rate(istio_requests_total{{
        destination_service="recommendation-service",
        destination_version="{version}",
        response_code=~"5.."
    }}[2m])) 
    / 
    sum(rate(istio_requests_total{{
        destination_service="recommendation-service",
        destination_version="{version}"
    }}[2m]))
    '''
    result = prom.custom_query(query=query)
    if result:
        return float(result[0]['value'][1])
    return 0.0

def update_traffic_split(v2_percent):
    v1_percent = 100 - v2_percent
    
    vs = v1.read_namespaced_virtual_service(
        name="recommendation-service",
        namespace="production"
    )
    
    vs.spec.http[0].route = [
        {
            "destination": {"host": "recommendation-service", "subset": "v1"},
            "weight": v1_percent
        },
        {
            "destination": {"host": "recommendation-service", "subset": "v2"},
            "weight": v2_percent
        }
    ]
    
    v1.patch_namespaced_virtual_service(
        name="recommendation-service",
        namespace="production",
        body=vs
    )
    print(f"✅ Traffic split updated: v1={v1_percent}%, v2={v2_percent}%")

def rollback():
    print("🚨 ROLLBACK: Reverting to 100% v1")
    update_traffic_split(v2_percent=0)

def canary_deploy():
    for stage in STAGES:
        print(f"🚀 Stage: {stage}% to v2")
        update_traffic_split(v2_percent=stage)
        
        # Wait for metrics
        time.sleep(STAGE_DURATION)
        
        # Check health
        error_rate = get_error_rate("v2")
        print(f"📊 Error rate: {error_rate:.2%}")
        
        if error_rate > 0.05:
            rollback()
            return False
    
    print("✅ Canary deployment successful!")
    return True

if __name__ == "__main__":
    success = canary_deploy()
    exit(0 if success else 1)
```

**Ejecución**:
```bash
# Deploy v2 (sin tráfico)
kubectl apply -f k8s/recommendation-v2.yaml

# Iniciar canary
python canary/canary-controller.py

# Monitorear en tiempo real
watch -n 5 'kubectl get vs recommendation-service -o yaml | grep weight'
```

</details>

---

## ⭐⭐⭐⭐ Ejercicio 4: Multi-Cluster Service Mesh

Configura Istio multi-cluster para disaster recovery.

**Arquitectura**:
- Cluster US-East (primary)
- Cluster EU-West (secondary)
- Failover automático si US-East cae
- Load balancing cross-cluster en operación normal

<details>
<summary>💡 Solución</summary>

```yaml
# cluster-us-east/service-entry.yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: payment-service-multicluster
spec:
  hosts:
  - payment-service.global
  location: MESH_INTERNAL
  ports:
  - number: 8080
    name: http
    protocol: HTTP
  resolution: DNS
  endpoints:
  - address: payment-service.us-east.svc.cluster.local
    locality: us-east/zone1
    labels:
      cluster: us-east
    weight: 100  # Primary
  - address: payment-service.eu-west.svc.cluster.local
    locality: eu-west/zone1
    labels:
      cluster: eu-west
    weight: 0  # Standby (solo failover)

---
# Locality Load Balancing
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service-multicluster
spec:
  host: payment-service.global
  trafficPolicy:
    loadBalancer:
      localityLbSetting:
        enabled: true
        distribute:
        - from: us-east/*
          to:
            "us-east/*": 80
            "eu-west/*": 20
        - from: eu-west/*
          to:
            "eu-west/*": 80
            "us-east/*": 20
    outlierDetection:
      consecutiveGatewayErrors: 3
      interval: 30s
      baseEjectionTime: 60s

---
# Failover configuration
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service-failover
spec:
  host: payment-service.global
  trafficPolicy:
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 10s
      baseEjectionTime: 30s
    loadBalancer:
      localityLbSetting:
        enabled: true
        failover:
        - from: us-east
          to: eu-west
        - from: eu-west
          to: us-east
```

```bash
# Install Istio en ambos clusters
# Cluster 1 (us-east)
istioctl install --set profile=default \
  --set values.global.multiCluster.clusterName=us-east \
  --set values.global.network=network1

# Cluster 2 (eu-west)
istioctl install --set profile=default \
  --set values.global.multiCluster.clusterName=eu-west \
  --set values.global.network=network2

# Create remote secrets
istioctl x create-remote-secret \
  --context=us-east \
  --name=us-east | \
  kubectl apply -f - --context=eu-west

istioctl x create-remote-secret \
  --context=eu-west \
  --name=eu-west | \
  kubectl apply -f - --context=us-east

# Verify multi-cluster setup
istioctl proxy-config endpoints payment-service-pod.us-east | grep payment-service
# Debería mostrar endpoints de ambos clusters
```

**Test de Failover**:
```bash
# Simular falla en US-East
kubectl scale deployment payment-service --replicas=0 -n us-east

# Verificar tráfico va a EU-West
kubectl logs -f order-service-pod -n us-east | grep payment-service
# Verás requests yendo a EU-West

# Restaurar US-East
kubectl scale deployment payment-service --replicas=3 -n us-east

# Tráfico vuelve gradualmente a US-East
```

</details>

---

## Proyecto Final: Service Mesh Completo

Implementa service mesh para un sistema de e-commerce con:

1. **Servicios**: Order, Payment, Inventory, Shipping, Notification
2. **Resilience**: Circuit breakers, retries, timeouts
3. **Security**: mTLS, authorization policies
4. **Observability**: Prometheus, Jaeger, Grafana dashboards
5. **Traffic Management**: Canary deployment para Order Service v2
6. **Testing**: Chaos testing con fault injection

**Rúbrica**:
- ⭐ Configuración básica de Istio
- ⭐⭐ Resilience patterns configurados
- ⭐⭐⭐ Observability completa + dashboards
- ⭐⭐⭐⭐ Canary + chaos testing + multi-cluster

## Recursos Adicionales

- **Istio Documentation**: https://istio.io/latest/docs/
- **Linkerd Documentation**: https://linkerd.io/docs/
- **Service Mesh Comparison**: https://servicemesh.io/
- **Book**: "Istio in Action" - Christian Posta
