# Evaluación: Serverless y Service Mesh

**Duración**: 3 horas  
**Puntuación total**: 100 puntos

---

## Parte 1: Preguntas Teóricas (20 puntos)

### Pregunta 1 (5 puntos)

Explica las ventajas y desventajas de Serverless (AWS Lambda) comparado con contenedores tradicionales (ECS, Kubernetes).

**Respuesta esperada**:

**Ventajas Serverless**:

- ✅ Auto-scaling automático (0 → 1000 instancias)
- ✅ Pay-per-use (solo pagas por tiempo de ejecución)
- ✅ Sin gestión de infraestructura
- ✅ Ideal para cargas variables

**Desventajas Serverless**:

- ❌ Cold start latency (primer request lento)
- ❌ Timeout máximo (15 min en Lambda)
- ❌ Vendor lock-in
- ❌ No ideal para cargas sostenidas 24/7

---

### Pregunta 2 (5 puntos)

¿Qué es un Service Mesh y qué problemas resuelve en arquitecturas de microservicios?

**Respuesta esperada**:

**Service Mesh**: Capa de infraestructura dedicada para gestionar comunicación entre microservicios.

**Problemas que resuelve**:

- ✅ Load balancing inteligente
- ✅ Service discovery automático
- ✅ Circuit breaking (prevenir cascadas de fallos)
- ✅ Mutual TLS (mTLS) automático
- ✅ Distributed tracing
- ✅ Traffic management (canary, A/B testing)
- ✅ Retries y timeouts configurables

**Sin Service Mesh**: Cada microservicio implementa esto en código (duplicación).

---

### Pregunta 3 (5 puntos)

Describe el flujo de un **canary deployment** con Istio. ¿Cómo se controla la distribución de tráfico?

**Respuesta esperada**:

1. Desplegar v2 con pocas réplicas
2. Configurar VirtualService con weights:
   - 90% tráfico → v1
   - 10% tráfico → v2
3. Monitorear métricas de v2 (error rate, latency)
4. Si v2 está OK, incrementar gradualmente:
   - 50% v1, 50% v2
   - 100% v2
5. Eliminar deployment v1

**Control**: VirtualService de Istio con `weight` en destinations.

---

### Pregunta 4 (5 puntos)

¿Qué es **mTLS** (Mutual TLS) en Istio y cómo mejora la seguridad?

**Respuesta esperada**:

**mTLS**: Autenticación bidireccional con certificados TLS.

- **TLS tradicional**: Solo el server tiene certificado (HTTPS)
- **mTLS**: Ambos (cliente y server) se autentican con certificados

**En Istio**:

- Istio inyecta certificados en cada sidecar proxy
- Todo el tráfico entre pods está cifrado
- Rotación automática de certificados
- Zero-trust security: Cada servicio valida identidad del otro

---

## Parte 2: Implementación Serverless (40 puntos)

### Proyecto: API Serverless para E-Commerce

Implementa una API serverless completa con AWS Lambda para gestionar un sistema de e-commerce.

#### Requisitos

**2.1** Funciones Lambda (20 puntos):

- `POST /products` - Crear producto
- `GET /products` - Listar productos (con paginación)
- `GET /products/{id}` - Obtener producto
- `PUT /products/{id}` - Actualizar producto
- `DELETE /products/{id}` - Eliminar producto

**2.2** Base de datos (5 puntos):

- Usar DynamoDB con índices secundarios para búsquedas

**2.3** Event processing (10 puntos):

- Cuando se crea un producto, trigger Lambda que:
  - Genere thumbnail de imagen (si tiene)
  - Envíe notificación SNS

**2.4** Deployment (5 puntos):

- AWS SAM template completo
- Scripts de deploy

### Solución de Referencia

```typescript
// src/handlers/products.ts
import { DynamoDB } from "aws-sdk";
import { SNS } from "aws-sdk";
import { APIGatewayProxyEvent, APIGatewayProxyResult } from "aws-lambda";

const dynamoDB = new DynamoDB.DocumentClient();
const sns = new SNS();

const TABLE_NAME = process.env.TABLE_NAME!;
const TOPIC_ARN = process.env.TOPIC_ARN!;

export const createProduct = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  const body = JSON.parse(event.body || "{}");
  const { name, description, price, category } = body;

  if (!name || !price) {
    return {
      statusCode: 400,
      body: JSON.stringify({ error: "Name and price are required" }),
    };
  }

  const product = {
    id: Date.now().toString(),
    name,
    description: description || "",
    price: parseFloat(price),
    category: category || "general",
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString(),
  };

  await dynamoDB
    .put({
      TableName: TABLE_NAME,
      Item: product,
    })
    .promise();

  // Trigger SNS notification
  await sns
    .publish({
      TopicArn: TOPIC_ARN,
      Message: JSON.stringify({
        event: "product_created",
        product,
      }),
      Subject: `New Product: ${name}`,
    })
    .promise();

  return {
    statusCode: 201,
    body: JSON.stringify(product),
  };
};

export const getProducts = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  const limit = parseInt(event.queryStringParameters?.limit || "20");
  const lastKey = event.queryStringParameters?.lastKey;

  const params: any = {
    TableName: TABLE_NAME,
    Limit: limit,
  };

  if (lastKey) {
    params.ExclusiveStartKey = JSON.parse(
      Buffer.from(lastKey, "base64").toString()
    );
  }

  const result = await dynamoDB.scan(params).promise();

  const nextKey = result.LastEvaluatedKey
    ? Buffer.from(JSON.stringify(result.LastEvaluatedKey)).toString("base64")
    : null;

  return {
    statusCode: 200,
    body: JSON.stringify({
      items: result.Items,
      nextKey,
    }),
  };
};

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

export const updateProduct = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  const id = event.pathParameters?.id;
  const body = JSON.parse(event.body || "{}");

  const updateExpression: string[] = [];
  const expressionAttributeValues: any = {};

  if (body.name) {
    updateExpression.push("name = :name");
    expressionAttributeValues[":name"] = body.name;
  }

  if (body.price !== undefined) {
    updateExpression.push("price = :price");
    expressionAttributeValues[":price"] = parseFloat(body.price);
  }

  if (body.description !== undefined) {
    updateExpression.push("description = :description");
    expressionAttributeValues[":description"] = body.description;
  }

  updateExpression.push("updatedAt = :updatedAt");
  expressionAttributeValues[":updatedAt"] = new Date().toISOString();

  const result = await dynamoDB
    .update({
      TableName: TABLE_NAME,
      Key: { id },
      UpdateExpression: `SET ${updateExpression.join(", ")}`,
      ExpressionAttributeValues: expressionAttributeValues,
      ReturnValues: "ALL_NEW",
    })
    .promise();

  return {
    statusCode: 200,
    body: JSON.stringify(result.Attributes),
  };
};

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

Globals:
  Function:
    Runtime: nodejs18.x
    MemorySize: 256
    Timeout: 10
    Environment:
      Variables:
        TABLE_NAME: !Ref ProductsTable
        TOPIC_ARN: !Ref ProductNotificationTopic

Resources:
  ProductsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: products
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
        - AttributeName: category
          AttributeType: S
        - AttributeName: createdAt
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      GlobalSecondaryIndexes:
        - IndexName: category-index
          KeySchema:
            - AttributeName: category
              KeyType: HASH
            - AttributeName: createdAt
              KeyType: RANGE
          Projection:
            ProjectionType: ALL

  ProductNotificationTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: product-notifications

  # Lambda Functions
  CreateProductFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./
      Handler: dist/handlers/products.createProduct
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref ProductsTable
        - SNSPublishMessagePolicy:
            TopicName: !GetAtt ProductNotificationTopic.TopicName
      Events:
        Api:
          Type: Api
          Properties:
            Path: /products
            Method: post

  GetProductsFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./
      Handler: dist/handlers/products.getProducts
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
      CodeUri: ./
      Handler: dist/handlers/products.getProduct
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref ProductsTable
      Events:
        Api:
          Type: Api
          Properties:
            Path: /products/{id}
            Method: get

  UpdateProductFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./
      Handler: dist/handlers/products.updateProduct
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref ProductsTable
      Events:
        Api:
          Type: Api
          Properties:
            Path: /products/{id}
            Method: put

  DeleteProductFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./
      Handler: dist/handlers/products.deleteProduct
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

## Parte 3: Service Mesh con Istio (40 puntos)

### Proyecto: Microservicios con Traffic Management

Implementa un sistema de microservicios en Kubernetes con Istio para traffic management avanzado.

#### Requisitos

**3.1** Microservicios (15 puntos):

- Frontend Service (React)
- Backend API Service (Node.js)
- Database Service (PostgreSQL)

**3.2** Istio Configuration (15 puntos):

- Canary deployment: 90% v1, 10% v2
- Circuit breaker configurado
- mTLS habilitado

**3.3** Observability (10 puntos):

- Distributed tracing con Jaeger
- Dashboards en Kiali

### Solución de Referencia

```yaml
# deployments.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
      version: v1
  template:
    metadata:
      labels:
        app: backend
        version: v1
    spec:
      containers:
        - name: backend
          image: backend:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: VERSION
              value: "1.0.0"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
      version: v2
  template:
    metadata:
      labels:
        app: backend
        version: v2
    spec:
      containers:
        - name: backend
          image: backend:2.0.0
          ports:
            - containerPort: 8080
          env:
            - name: VERSION
              value: "2.0.0"

---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080

---
# istio-config.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: backend-vs
spec:
  hosts:
    - backend-service
  http:
    - route:
        - destination:
            host: backend-service
            subset: v1
          weight: 90
        - destination:
            host: backend-service
            subset: v2
          weight: 10

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: backend-dr
spec:
  host: backend-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2

---
# mtls.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT
```

**Deployment**:

```bash
# 1. Instalar Istio
istioctl install --set profile=demo -y

# Enable sidecar injection
kubectl label namespace default istio-injection=enabled

# 2. Deploy microservices
kubectl apply -f deployments.yaml
kubectl apply -f service.yaml

# 3. Configure Istio
kubectl apply -f istio-config.yaml
kubectl apply -f mtls.yaml

# 4. Deploy observability tools
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/jaeger.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml

# 5. Test canary deployment
for i in $(seq 1 100); do
  kubectl exec -it <test-pod> -- curl -s http://backend-service/version | jq -r '.version'
done | sort | uniq -c

# Esperado:
#  90 1.0.0
#  10 2.0.0

# 6. Access Kiali dashboard
istioctl dashboard kiali
```

---

## Criterios de Evaluación

| Sección                        | Puntos | Criterios                                          |
| ------------------------------ | ------ | -------------------------------------------------- |
| Parte 1: Teoría                | 20     | Comprensión de serverless y service mesh           |
| Parte 2: Lambda API            | 40     | API funcional, DynamoDB, SNS, deployment           |
| Parte 3: Istio                 | 40     | Canary deployment, circuit breaker, mTLS           |
| **Total**                      | **100** |                                                   |
