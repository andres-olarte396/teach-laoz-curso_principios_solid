# Guion de Video: Serverless y Service Mesh

**Duración**: 35 minutos  
**Objetivo**: Explicar serverless computing (AWS Lambda) y service mesh (Istio) con demos prácticas

---

## [00:00-01:30] Introducción

Bienvenidos al módulo de **Serverless y Service Mesh**, dos paradigmas que están revolucionando las arquitecturas cloud-native modernas.

**Serverless** NO significa "sin servidores". Significa que **tú no gestionas servidores**. El cloud provider se encarga de infraestructura, OS, patches, scaling.

**Service Mesh** es una capa de infraestructura que gestiona comunicación entre microservicios: load balancing, security, observability.

**Agenda**:

1. AWS Lambda: Event-driven functions
2. Patrones serverless
3. Istio: Traffic management
4. Canary deployments
5. mTLS y security

---

## [01:30-06:00] AWS Lambda Basics

### ¿Qué es Lambda?

**AWS Lambda** ejecuta código en respuesta a eventos sin gestionar servidores.

**Triggers comunes**:

- HTTP requests (API Gateway)
- S3 uploads
- DynamoDB streams
- CloudWatch Events (schedule)

**Demo: Hello World Lambda**

```typescript
// handler.ts
import { APIGatewayProxyEvent, APIGatewayProxyResult } from "aws-lambda";

export const handler = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  const name = event.queryStringParameters?.name || "World";

  return {
    statusCode: 200,
    body: JSON.stringify({
      message: `Hello, ${name}!`,
      timestamp: new Date().toISOString(),
    }),
  };
};
```

**SAM Template**:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31

Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: nodejs18.x
      Handler: dist/handler.handler
      CodeUri: ./
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /hello
            Method: get
```

**Deploy**:

```bash
sam build
sam deploy --guided

# Test
curl https://abc123.execute-api.us-east-1.amazonaws.com/Prod/hello?name=Alice
# {"message":"Hello, Alice!","timestamp":"2024-01-15T10:30:00Z"}
```

**Key features**:

✅ Auto-scaling: 0 → 1000 invocaciones automáticamente  
✅ Pay-per-use: Solo pagas por tiempo de ejecución (milisegundos)  
✅ Managed: AWS gestiona todo

---

## [06:00-12:00] Patrones Serverless

### Patrón 1: API Backend (CRUD)

```
Client → API Gateway → Lambda → DynamoDB
```

**Demo: CRUD de Productos**

```typescript
// src/handlers/products.ts
import { DynamoDB } from "aws-sdk";
const dynamoDB = new DynamoDB.DocumentClient();

export const getProducts = async () => {
  const result = await dynamoDB
    .scan({
      TableName: process.env.TABLE_NAME!,
    })
    .promise();

  return {
    statusCode: 200,
    body: JSON.stringify(result.Items),
  };
};

export const createProduct = async (event) => {
  const { name, price } = JSON.parse(event.body);

  const product = {
    id: Date.now().toString(),
    name,
    price,
    createdAt: new Date().toISOString(),
  };

  await dynamoDB
    .put({
      TableName: process.env.TABLE_NAME!,
      Item: product,
    })
    .promise();

  return {
    statusCode: 201,
    body: JSON.stringify(product),
  };
};
```

**Test**:

```bash
# Create product
curl -X POST $API_URL/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Laptop","price":999.99}'

# Get products
curl $API_URL/products
# [{"id":"1234567890","name":"Laptop","price":999.99,"createdAt":"..."}]
```

### Patrón 2: Event Processing (S3 → Lambda)

```
S3 Upload → Lambda (resize image) → S3 (thumbnails)
```

**Demo: Image Processor**

```typescript
// src/handlers/imageProcessor.ts
import { S3Event } from "aws-lambda";
import AWS from "aws-sdk";
import sharp from "sharp";

const s3 = new AWS.S3();

export const handler = async (event: S3Event) => {
  for (const record of event.Records) {
    const key = record.s3.object.key;

    // Download image
    const original = await s3
      .getObject({
        Bucket: "my-bucket",
        Key: key,
      })
      .promise();

    // Resize to thumbnail
    const thumbnail = await sharp(original.Body as Buffer)
      .resize(200, 200)
      .toBuffer();

    // Upload thumbnail
    await s3
      .putObject({
        Bucket: "my-bucket",
        Key: `thumbnails/${key}`,
        Body: thumbnail,
      })
      .promise();

    console.log(`Thumbnail created: thumbnails/${key}`);
  }
};
```

**Trigger configurado en SAM**:

```yaml
ImageProcessorFunction:
  Type: AWS::Serverless::Function
  Events:
    S3Event:
      Type: S3
      Properties:
        Bucket: !Ref ImageBucket
        Events: s3:ObjectCreated:*
        Filter:
          S3Key:
            Rules:
              - Name: prefix
                Value: uploads/
```

**Test**:

```bash
# Upload imagen
aws s3 cp photo.jpg s3://my-bucket/uploads/photo.jpg

# Lambda se ejecuta automáticamente
# Thumbnail en s3://my-bucket/thumbnails/photo.jpg
```

**Ventaja**: Event-driven, solo se ejecuta cuando hay uploads.

---

## [12:00-17:00] Introducción a Service Mesh

### ¿Qué es Service Mesh?

En microservicios, cada servicio necesita:

- Load balancing
- Service discovery
- Retries, timeouts
- Circuit breaking
- mTLS
- Distributed tracing

**Problema**: Implementar esto en cada servicio duplica código.

**Solución**: Service Mesh gestiona todo esto **fuera del código** de la app.

### Istio Architecture

```
Control Plane (istiod)
    ↓ config
Data Plane (Envoy Sidecars)

Pod 1            Pod 2
┌──────────┐    ┌──────────┐
│  App     │    │  App     │
│          │    │          │
│  Envoy ←─┼────┼→ Envoy  │
└──────────┘    └──────────┘
```

**Envoy sidecar** intercepta todo el tráfico de entrada y salida.

**Instalación**:

```bash
# Descargar Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.20.0

# Instalar
istioctl install --set profile=demo -y

# Habilitar sidecar injection
kubectl label namespace default istio-injection=enabled
```

**Verificar**:

```bash
kubectl get pods -n istio-system
# NAME                                    READY   STATUS    RESTARTS   AGE
# istiod-7d8f5c9b7f-abc12                 1/1     Running   0          2m
# istio-ingressgateway-6f9d8c7b8f-def34   1/1     Running   0          2m
```

---

## [17:00-23:00] Traffic Management: Canary Deployment

**Escenario**: Tienes app v1 en producción. Quieres desplegar v2 pero solo a 10% de usuarios inicialmente.

### Deployments

```yaml
# deployment-v1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
        - name: myapp
          image: myapp:1.0.0

---
# deployment-v2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
        - name: myapp
          image: myapp:2.0.0
```

### Istio Configuration

```yaml
# virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-vs
spec:
  hosts:
    - myapp-service
  http:
    - route:
        - destination:
            host: myapp-service
            subset: v1
          weight: 90
        - destination:
            host: myapp-service
            subset: v2
          weight: 10

---
# destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp-dr
spec:
  host: myapp-service
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

**Deploy**:

```bash
kubectl apply -f deployment-v1.yaml
kubectl apply -f deployment-v2.yaml
kubectl apply -f service.yaml
kubectl apply -f virtual-service.yaml
kubectl apply -f destination-rule.yaml

# Test: 100 requests
for i in $(seq 1 100); do
  curl -s http://myapp-service/version | jq -r '.version'
done | sort | uniq -c
```

**Resultado**:

```
 90 1.0.0
 10 2.0.0
```

**Istio distribuyó tráfico exactamente como configuramos**.

**Incrementar a 50/50**:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-vs
spec:
  hosts:
    - myapp-service
  http:
    - route:
        - destination:
            host: myapp-service
            subset: v1
          weight: 50
        - destination:
            host: myapp-service
            subset: v2
          weight: 50
EOF
```

**Finalmente 100% v2**:

```yaml
weight: 0 # v1
weight: 100 # v2
```

**Ventaja**: Canary deployment sin cambios en el código, todo declarativo.

---

## [23:00-27:00] Mutual TLS (mTLS)

**mTLS**: Autenticación bidireccional con certificados.

- **TLS tradicional**: Solo server tiene certificado (HTTPS)
- **mTLS**: Cliente y server se autentican mutuamente

**Habilitar mTLS en Istio**:

```yaml
# peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT
```

```bash
kubectl apply -f peer-authentication.yaml
```

**Resultado**: Todo el tráfico entre pods en `default` namespace está cifrado con mTLS.

**Verificar certificados**:

```bash
istioctl proxy-config secret <pod-name> -o json | jq '.dynamicActiveSecrets'
```

Veremos certificados X.509 generados automáticamente por Istio.

**Beneficios**:

✅ Zero-trust security  
✅ Rotación automática de certificados  
✅ Sin cambios en código de la app

---

## [27:00-31:00] Circuit Breaking

**Problema**: Un servicio lento puede saturar todo el sistema.

**Solución**: Circuit breaker expulsa pods fallidos temporalmente.

```yaml
# destination-rule-circuit-breaker.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: backend-circuit-breaker
spec:
  host: backend-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

**Comportamiento**:

- Si un pod falla 5 requests consecutivos → se expulsa del pool por 30s
- Máximo 50% de pods pueden estar expulsados

**Test**: Simular servicio lento:

```bash
# Inyectar delay en un pod
kubectl exec -it <backend-pod> -- killall -STOP node

# Hacer requests
for i in $(seq 1 100); do
  curl -s http://backend-service/api/data
done

# Istio detecta fallos y deja de enviar tráfico a ese pod
```

---

## [31:00-35:00] Observability con Jaeger

Istio integra con **Jaeger** para distributed tracing.

```bash
# Deploy Jaeger
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/jaeger.yaml

# Port-forward
kubectl port-forward -n istio-system svc/jaeger 16686:16686

# Abrir browser
open http://localhost:16686
```

**Hacer requests para generar traces**:

```bash
for i in $(seq 1 20); do
  curl http://myapp-service/api/products
done
```

**En Jaeger UI**:

Veremos trace completo:

```
User Request (200ms total)
    ↓
Frontend Service (10ms)
    ↓
Backend Service (50ms)
    ↓
Database Query (140ms)
```

**Sin cambios en código**: Istio captura traces automáticamente via sidecar proxies.

---

## [35:00-37:00] Resumen

**Serverless (AWS Lambda)**:

✅ Event-driven, pay-per-use  
✅ Auto-scaling automático  
✅ Sin gestión de servidores  
✅ Ideal para cargas variables  
❌ Cold start latency  
❌ Timeout máximo (15 min)

**Service Mesh (Istio)**:

✅ Traffic management (canary, A/B testing)  
✅ mTLS automático  
✅ Circuit breaking  
✅ Distributed tracing  
✅ Todo declarativo, sin cambios en código  
❌ Complejidad adicional  
❌ Overhead de sidecars

**Casos de uso**:

- **Serverless**: APIs, event processing, scheduled tasks
- **Service Mesh**: Microservicios complejos, multi-cloud, zero-trust security

**Próximo tema**: Observability y Monitoring con Prometheus, Grafana y OpenTelemetry.

**Gracias por ver este video.**
