# Serverless y Service Mesh

## Introducción

En este tema exploraremos dos paradigmas fundamentales para arquitecturas cloud-native modernas:

1. **Serverless**: Event-driven, pay-per-use, sin gestión de infraestructura
2. **Service Mesh**: Comunicación avanzada entre microservicios con observabilidad, seguridad y control de tráfico

---

## Parte 1. Serverless Computing

### ¿Qué es Serverless?

**Serverless** NO significa "sin servidores". Significa que **tú no gestionas los servidores**.

**Características**:

- ✅ **Event-driven**: Las funciones se ejecutan en respuesta a eventos
- ✅ **Auto-scaling**: De 0 a 1000 instancias automáticamente
- ✅ **Pay-per-use**: Solo pagas por el tiempo de ejecución (milisegundos)
- ✅ **Stateless**: Cada invocación es independiente
- ✅ **Managed**: El cloud provider gestiona infraestructura, OS, runtime

**Comparación**:

| Aspecto          | Tradicional (EC2, VMs)                | Serverless (Lambda, Functions)          |
| ---------------- | ------------------------------------- | --------------------------------------- |
| Infraestructura  | Tú gestionas VMs, OS, patches         | Managed por cloud provider              |
| Escalado         | Manual o auto-scaling groups          | Automático e instantáneo                |
| Costos           | Pagas por uptime (24/7)               | Pagas por ejecución (ms)                |
| Idle time        | Pagas aunque no haya requests         | $0 cuando no hay invocaciones           |
| Límite ejecución | Sin límite                            | Típicamente 15 min (AWS Lambda)         |

---

## AWS Lambda

### Conceptos Básicos

**AWS Lambda** ejecuta tu código en respuesta a eventos:

- HTTP requests (API Gateway)
- S3 uploads (nuevo archivo subido)
- DynamoDB streams (cambio en tabla)
- CloudWatch Events (schedule)
- SQS/SNS messages

**Límites clave**:

| Límite                | Valor                           |
| --------------------- | ------------------------------- |
| Timeout máximo        | 15 minutos                      |
| Memoria               | 128 MB - 10 GB                  |
| Deployment package    | 50 MB (zip), 250 MB (unzipped)  |
| Concurrent executions | 1000 (soft limit, aumentable)   |

### Función Lambda Simple (TypeScript)

```typescript
// handler.ts
import { APIGatewayProxyEvent, APIGatewayProxyResult } from "aws-lambda";

export const handler = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  console.log("Event:", JSON.stringify(event, null, 2));

  const name = event.queryStringParameters?.name || "World";

  return {
    statusCode: 200,
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      message: `Hello, ${name}!`,
      timestamp: new Date().toISOString(),
    }),
  };
};
```

**Deployment (AWS SAM)**:

```yaml
# template.yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31

Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: HelloWorldFunction
      Runtime: nodejs18.x
      Handler: dist/handler.handler
      CodeUri: ./
      MemorySize: 256
      Timeout: 10
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /hello
            Method: get
```

```bash
# Build y deploy
sam build
sam deploy --guided

# Test
curl https://abc123.execute-api.us-east-1.amazonaws.com/Prod/hello?name=Alice
# {"message":"Hello, Alice!","timestamp":"2024-01-15T10:30:00Z"}
```

---

### Patrones Serverless Comunes

#### 1. API Backend (Lambda + API Gateway)

```
Client
  ↓ HTTP
API Gateway
  ↓ invoke
Lambda Function
  ↓ query
DynamoDB
```

**Ventajas**:

- Auto-scaling completo (API Gateway + Lambda)
- Sin servidores que gestionar
- Pay-per-request

**Ejemplo: CRUD de Productos**

```typescript
// src/handlers/products.ts
import { DynamoDB } from "aws-sdk";
import { APIGatewayProxyEvent, APIGatewayProxyResult } from "aws-lambda";

const dynamoDB = new DynamoDB.DocumentClient();
const TABLE_NAME = process.env.TABLE_NAME!;

// GET /products
export const getProducts = async (): Promise<APIGatewayProxyResult> => {
  const result = await dynamoDB
    .scan({
      TableName: TABLE_NAME,
    })
    .promise();

  return {
    statusCode: 200,
    body: JSON.stringify(result.Items),
  };
};

// GET /products/{id}
export const getProduct = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  const id = event.pathParameters?.id;

  const result = await dynamoDB
    .get({
      TableName: TABLE_NAME,
      Key: { id },
    })
    .promise();

  if (!result.Item) {
    return {
      statusCode: 404,
      body: JSON.stringify({ error: "Product not found" }),
    };
  }

  return {
    statusCode: 200,
    body: JSON.stringify(result.Item),
  };
};

// POST /products
export const createProduct = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  const body = JSON.parse(event.body || "{}");
  const { name, price } = body;

  const product = {
    id: Date.now().toString(),
    name,
    price,
    createdAt: new Date().toISOString(),
  };

  await dynamoDB
    .put({
      TableName: TABLE_NAME,
      Item: product,
    })
    .promise();

  return {
    statusCode: 201,
    body: JSON.stringify(product),
  };
};

// DELETE /products/{id}
export const deleteProduct = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  const id = event.pathParameters?.id;

  await dynamoDB
    .delete({
      TableName: TABLE_NAME,
      Key: { id },
    })
    .promise();

  return {
    statusCode: 204,
    body: "",
  };
};
```

**SAM Template**:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31

Resources:
  # DynamoDB Table
  ProductsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: products
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH

  # Lambda Functions
  GetProductsFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: nodejs18.x
      Handler: dist/handlers/products.getProducts
      Environment:
        Variables:
          TABLE_NAME: !Ref ProductsTable
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref ProductsTable
      Events:
        Api:
          Type: Api
          Properties:
            Path: /products
            Method: get

  GetProductFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: nodejs18.x
      Handler: dist/handlers/products.getProduct
      Environment:
        Variables:
          TABLE_NAME: !Ref ProductsTable
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref ProductsTable
      Events:
        Api:
          Type: Api
          Properties:
            Path: /products/{id}
            Method: get

  CreateProductFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: nodejs18.x
      Handler: dist/handlers/products.createProduct
      Environment:
        Variables:
          TABLE_NAME: !Ref ProductsTable
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref ProductsTable
      Events:
        Api:
          Type: Api
          Properties:
            Path: /products
            Method: post

  DeleteProductFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: nodejs18.x
      Handler: dist/handlers/products.deleteProduct
      Environment:
        Variables:
          TABLE_NAME: !Ref ProductsTable
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref ProductsTable
      Events:
        Api:
          Type: Api
          Properties:
            Path: /products/{id}
            Method: delete

Outputs:
  ApiUrl:
    Description: API Gateway endpoint URL
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod"
```

---

#### 2. Event Processing (S3 → Lambda → DB)

```
S3 Bucket (upload image)
  ↓ event
Lambda (resize image)
  ↓ save
S3 Bucket (thumbnails)
  ↓ metadata
DynamoDB
```

**Función de procesamiento**:

```typescript
// src/handlers/imageProcessor.ts
import { S3Event } from "aws-lambda";
import AWS from "aws-sdk";
import sharp from "sharp";

const s3 = new AWS.S3();

export const handler = async (event: S3Event): Promise<void> => {
  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, " "));

    console.log(`Processing ${bucket}/${key}`);

    // Download image
    const originalImage = await s3
      .getObject({
        Bucket: bucket,
        Key: key,
      })
      .promise();

    // Resize to thumbnail
    const thumbnail = await sharp(originalImage.Body as Buffer)
      .resize(200, 200, { fit: "cover" })
      .toBuffer();

    // Upload thumbnail
    const thumbnailKey = `thumbnails/${key}`;
    await s3
      .putObject({
        Bucket: bucket,
        Key: thumbnailKey,
        Body: thumbnail,
        ContentType: "image/jpeg",
      })
      .promise();

    console.log(`Thumbnail created: ${thumbnailKey}`);
  }
};
```

**SAM Template**:

```yaml
Resources:
  ImageBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-images-bucket

  ImageProcessorFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: nodejs18.x
      Handler: dist/handlers/imageProcessor.handler
      MemorySize: 1024
      Timeout: 60
      Policies:
        - S3CrudPolicy:
            BucketName: !Ref ImageBucket
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
                  - Name: suffix
                    Value: .jpg
```

**Uso**:

```bash
# Upload imagen
aws s3 cp photo.jpg s3://my-images-bucket/uploads/photo.jpg

# Lambda se ejecuta automáticamente
# Thumbnail creado en s3://my-images-bucket/thumbnails/photo.jpg
```

---

#### 3. Scheduled Tasks (CloudWatch Events → Lambda)

```typescript
// src/handlers/cleanupOldData.ts
import { ScheduledEvent } from "aws-lambda";
import AWS from "aws-sdk";

const dynamoDB = new AWS.DynamoDB.DocumentClient();
const TABLE_NAME = process.env.TABLE_NAME!;

export const handler = async (event: ScheduledEvent): Promise<void> => {
  console.log("Running scheduled cleanup...", event.time);

  const cutoffDate = new Date();
  cutoffDate.setDate(cutoffDate.getDate() - 30); // 30 días atrás

  // Scan para encontrar items viejos
  const result = await dynamoDB
    .scan({
      TableName: TABLE_NAME,
      FilterExpression: "createdAt < :cutoff",
      ExpressionAttributeValues: {
        ":cutoff": cutoffDate.toISOString(),
      },
    })
    .promise();

  // Eliminar items
  for (const item of result.Items || []) {
    await dynamoDB
      .delete({
        TableName: TABLE_NAME,
        Key: { id: item.id },
      })
      .promise();
    console.log(`Deleted item: ${item.id}`);
  }

  console.log(`Cleanup complete. Deleted ${result.Items?.length || 0} items`);
};
```

**SAM Template**:

```yaml
CleanupFunction:
  Type: AWS::Serverless::Function
  Properties:
    Runtime: nodejs18.x
    Handler: dist/handlers/cleanupOldData.handler
    Environment:
      Variables:
        TABLE_NAME: !Ref ProductsTable
    Policies:
      - DynamoDBCrudPolicy:
          TableName: !Ref ProductsTable
    Events:
      Schedule:
        Type: Schedule
        Properties:
          Schedule: cron(0 2 * * ? *) # Todos los días a las 2 AM UTC
```

---

## Azure Functions

**Azure Functions** es el equivalente de AWS Lambda en Microsoft Azure.

### Función HTTP Simple (TypeScript)

```typescript
// src/functions/httpTrigger.ts
import { app, HttpRequest, HttpResponseInit, InvocationContext } from "@azure/functions";

export async function httpTrigger(
  request: HttpRequest,
  context: InvocationContext
): Promise<HttpResponseInit> {
  context.log("HTTP trigger function processed a request");

  const name = request.query.get("name") || (await request.text()) || "World";

  return {
    status: 200,
    jsonBody: {
      message: `Hello, ${name}!`,
      timestamp: new Date().toISOString(),
    },
  };
}

app.http("httpTrigger", {
  methods: ["GET", "POST"],
  authLevel: "anonymous",
  handler: httpTrigger,
});
```

### Timer Trigger (Scheduled)

```typescript
// src/functions/timerTrigger.ts
import { app, InvocationContext, Timer } from "@azure/functions";

export async function timerTrigger(myTimer: Timer, context: InvocationContext): Promise<void> {
  context.log("Timer trigger function ran!", myTimer.scheduleStatus);

  // Ejecutar tarea
  await processScheduledTask();
}

async function processScheduledTask() {
  console.log("Processing scheduled task...");
  // Lógica aquí
}

app.timer("timerTrigger", {
  schedule: "0 */5 * * * *", // Cada 5 minutos
  handler: timerTrigger,
});
```

### Blob Storage Trigger

```typescript
// src/functions/blobTrigger.ts
import { app, InvocationContext } from "@azure/functions";

export async function blobTrigger(blob: Buffer, context: InvocationContext): Promise<void> {
  context.log(`Blob trigger processed blob "${context.triggerMetadata?.name}" with size ${blob.length} bytes`);

  // Procesar blob (ej: resize imagen)
  const processedBlob = await processBlobData(blob);

  // Guardar resultado en otro container
  // ...
}

app.storageBlob("blobTrigger", {
  path: "uploads/{name}",
  connection: "AzureWebJobsStorage",
  handler: blobTrigger,
});
```

---

## Google Cloud Functions

### HTTP Function (TypeScript)

```typescript
// src/index.ts
import { HttpFunction } from "@google-cloud/functions-framework/build/src/functions";

export const helloWorld: HttpFunction = (req, res) => {
  const name = req.query.name || req.body.name || "World";

  res.status(200).json({
    message: `Hello, ${name}!`,
    timestamp: new Date().toISOString(),
  });
};
```

### Pub/Sub Trigger

```typescript
// src/pubsubHandler.ts
import { CloudEvent } from "@google-cloud/functions-framework";

export const processPubSubMessage = (cloudEvent: CloudEvent) => {
  const message = cloudEvent.data.message;
  const decodedData = Buffer.from(message.data, "base64").toString();

  console.log("Received message:", decodedData);

  // Procesar mensaje
  processMessage(JSON.parse(decodedData));
};

function processMessage(data: any) {
  console.log("Processing:", data);
  // Lógica aquí
}
```

**Deploy**:

```bash
gcloud functions deploy processPubSubMessage \
  --runtime nodejs18 \
  --trigger-topic my-topic \
  --entry-point processPubSubMessage
```

---

## Parte 2: Service Mesh

### ¿Qué es Service Mesh?

**Service Mesh** es una capa de infraestructura dedicada para gestionar comunicación entre microservicios.

**Problema que resuelve**:

En arquitecturas de microservicios, cada servicio necesita:

- ✅ Load balancing
- ✅ Service discovery
- ✅ Circuit breaking
- ✅ Retries y timeouts
- ✅ Mutual TLS (mTLS)
- ✅ Distributed tracing
- ✅ Metrics

**Sin Service Mesh**: Cada microservicio implementa esto en código (duplicación, complejidad).

**Con Service Mesh**: Todo esto se gestiona de forma declarativa fuera del código de la app.

---

## Istio (Service Mesh para Kubernetes)

### Arquitectura

```
┌─────────────────────────────────────────────┐
│         Control Plane (istiod)              │
│  - Service discovery                        │
│  - Configuration                            │
│  - Certificate management                   │
└─────────────────────────────────────────────┘
                    ↓ config
┌─────────────────────────────────────────────┐
│         Data Plane (Envoy Sidecars)         │
│                                             │
│  Pod 1            Pod 2            Pod 3    │
│  ┌──────────┐    ┌──────────┐    ┌──────┐  │
│  │  App     │    │  App     │    │ App  │  │
│  │          │    │          │    │      │  │
│  │  Envoy ←─┼────┼→ Envoy ←─┼────┼→Envoy│  │
│  └──────────┘    └──────────┘    └──────┘  │
│                                             │
└─────────────────────────────────────────────┘
```

**Componente clave**: **Envoy sidecar proxy** intercepta todo el tráfico de entrada y salida del pod.

---

### Instalación de Istio

```bash
# Descargar Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.20.0

# Instalar
istioctl install --set profile=demo -y

# Habilitar sidecar injection automático en namespace
kubectl label namespace default istio-injection=enabled

# Verificar
kubectl get pods -n istio-system
```

---

### Traffic Management: Canary Deployment

**Escenario**: Quieres desplegar v2 pero solo enviarle 10% del tráfico inicialmente.

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
          ports:
            - containerPort: 8080

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
          ports:
            - containerPort: 8080

---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080

---
# virtual-service.yaml (Istio)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-vs
spec:
  hosts:
    - myapp-service
  http:
    - match:
        - headers:
            user-agent:
              regex: ".*Mobile.*"
      route:
        - destination:
            host: myapp-service
            subset: v2
          weight: 100
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

**Resultado**:

- 90% del tráfico → v1
- 10% del tráfico → v2
- Usuarios mobile → 100% v2 (A/B testing)

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
# Verás aprox:
#  90 v1.0.0
#  10 v2.0.0
```

---

### Mutual TLS (mTLS)

Istio puede asegurar toda la comunicación entre servicios con mTLS automáticamente.

```yaml
# peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT # Requiere mTLS para toda comunicación
```

```bash
kubectl apply -f peer-authentication.yaml
```

**Resultado**: Todo el tráfico entre pods en el namespace `default` está cifrado con mTLS.

**Verificar**:

```bash
# Ver certificados
istioctl proxy-config secret <pod-name> -o json | jq '.dynamicActiveSecrets'
```

---

### Circuit Breaking

**Prevenir sobrecarga** de un servicio fallido.

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
        http2MaxRequests: 100
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

**Comportamiento**:

- Si un pod falla 5 requests consecutivos → se expulsa del pool por 30s
- Máximo 50% de pods pueden estar expulsados simultáneamente

---

### Observability: Distributed Tracing

Istio integra con **Jaeger** para distributed tracing.

```bash
# Desplegar Jaeger
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/jaeger.yaml

# Port-forward
kubectl port-forward -n istio-system svc/jaeger 16686:16686

# Abrir en browser
open http://localhost:16686
```

**Trace automático**:

```
User Request
    ↓
Frontend Service (100ms)
    ↓
Backend Service (50ms)
    ↓
Database Query (30ms)
```

Istio captura este trace completo sin cambios en el código de la app.

---

## Resumen

| Concepto         | Tecnología               | Uso Principal                                        |
| ---------------- | ------------------------ | ---------------------------------------------------- |
| **Serverless**   | AWS Lambda               | Event-driven, pay-per-use, auto-scaling              |
|                  | Azure Functions          | Mismo concepto en Azure                              |
|                  | Google Cloud Functions   | Mismo concepto en GCP                                |
| **Service Mesh** | Istio                    | Traffic management, mTLS, observability              |
|                  | Linkerd                  | Más simple que Istio, menos features                 |
|                  | Consul                   | Service mesh + service discovery                     |

**Próximo tema**: Observability y Monitoring con Prometheus, Grafana y distributed tracing.
