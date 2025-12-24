# Ejercicios: Serverless y Service Mesh

## ⭐ Ejercicio 1. AWS Lambda Function Simple

**Objetivo**: Crear una función Lambda que procese eventos HTTP y se conecte a DynamoDB.

### Contexto

Necesitas una API serverless para gestionar una lista de TODOs.

### Tareas

**1.1** Crea una función Lambda con TypeScript que:

- `GET /todos` - Liste todos los TODOs
- `POST /todos` - Cree un nuevo TODO
- `PUT /todos/{id}` - Actualice un TODO
- `DELETE /todos/{id}` - Elimine un TODO

**1.2** Usa DynamoDB como base de datos.

**1.3** Despliega con AWS SAM.

### Solución

```typescript
// src/handlers/todos.ts
import { DynamoDB } from "aws-sdk";
import {
  APIGatewayProxyEvent,
  APIGatewayProxyResult,
} from "aws-lambda";

const dynamoDB = new DynamoDB.DocumentClient();
const TABLE_NAME = process.env.TABLE_NAME!;

export const getTodos = async (): Promise<APIGatewayProxyResult> => {
  const result = await dynamoDB
    .scan({
      TableName: TABLE_NAME,
    })
    .promise();

  return {
    statusCode: 200,
    headers: {
      "Content-Type": "application/json",
      "Access-Control-Allow-Origin": "*",
    },
    body: JSON.stringify(result.Items),
  };
};

export const createTodo = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  const body = JSON.parse(event.body || "{}");
  const { title, description } = body;

  if (!title) {
    return {
      statusCode: 400,
      body: JSON.stringify({ error: "Title is required" }),
    };
  }

  const todo = {
    id: Date.now().toString(),
    title,
    description: description || "",
    completed: false,
    createdAt: new Date().toISOString(),
  };

  await dynamoDB
    .put({
      TableName: TABLE_NAME,
      Item: todo,
    })
    .promise();

  return {
    statusCode: 201,
    headers: {
      "Content-Type": "application/json",
      "Access-Control-Allow-Origin": "*",
    },
    body: JSON.stringify(todo),
  };
};

export const updateTodo = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  const id = event.pathParameters?.id;
  const body = JSON.parse(event.body || "{}");

  const updateExpression: string[] = [];
  const expressionAttributeValues: any = {};
  const expressionAttributeNames: any = {};

  if (body.title !== undefined) {
    updateExpression.push("#title = :title");
    expressionAttributeNames["#title"] = "title";
    expressionAttributeValues[":title"] = body.title;
  }

  if (body.completed !== undefined) {
    updateExpression.push("completed = :completed");
    expressionAttributeValues[":completed"] = body.completed;
  }

  if (updateExpression.length === 0) {
    return {
      statusCode: 400,
      body: JSON.stringify({ error: "No fields to update" }),
    };
  }

  const result = await dynamoDB
    .update({
      TableName: TABLE_NAME,
      Key: { id },
      UpdateExpression: `SET ${updateExpression.join(", ")}`,
      ExpressionAttributeNames:
        Object.keys(expressionAttributeNames).length > 0
          ? expressionAttributeNames
          : undefined,
      ExpressionAttributeValues: expressionAttributeValues,
      ReturnValues: "ALL_NEW",
    })
    .promise();

  return {
    statusCode: 200,
    headers: {
      "Content-Type": "application/json",
      "Access-Control-Allow-Origin": "*",
    },
    body: JSON.stringify(result.Attributes),
  };
};

export const deleteTodo = async (
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
    headers: {
      "Access-Control-Allow-Origin": "*",
    },
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
        TABLE_NAME: !Ref TodosTable

Resources:
  TodosTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: todos
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH

  GetTodosFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./
      Handler: dist/handlers/todos.getTodos
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref TodosTable
      Events:
        Api:
          Type: Api
          Properties:
            Path: /todos
            Method: get

  CreateTodoFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./
      Handler: dist/handlers/todos.createTodo
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref TodosTable
      Events:
        Api:
          Type: Api
          Properties:
            Path: /todos
            Method: post

  UpdateTodoFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./
      Handler: dist/handlers/todos.updateTodo
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref TodosTable
      Events:
        Api:
          Type: Api
          Properties:
            Path: /todos/{id}
            Method: put

  DeleteTodoFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./
      Handler: dist/handlers/todos.deleteTodo
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref TodosTable
      Events:
        Api:
          Type: Api
          Properties:
            Path: /todos/{id}
            Method: delete

Outputs:
  ApiUrl:
    Description: API Gateway endpoint URL
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod"
```

**Comandos**:

```bash
# Build
npm run build
sam build

# Deploy
sam deploy --guided

# Test
API_URL=$(aws cloudformation describe-stacks \
  --stack-name todos-api \
  --query 'Stacks[0].Outputs[?OutputKey==`ApiUrl`].OutputValue' \
  --output text)

# Create TODO
curl -X POST $API_URL/todos \
  -H "Content-Type: application/json" \
  -d '{"title":"Buy groceries","description":"Milk, eggs, bread"}'

# Get all TODOs
curl $API_URL/todos

# Update TODO
curl -X PUT $API_URL/todos/1234567890 \
  -H "Content-Type: application/json" \
  -d '{"completed":true}'

# Delete TODO
curl -X DELETE $API_URL/todos/1234567890
```

**Resultado esperado**:

✅ API serverless funcional  
✅ Auto-scaling automático (0 → 1000 invocaciones)  
✅ Pay-per-request (solo pagas cuando hay tráfico)  
✅ Sin servidores que gestionar

---

## ⭐⭐ Ejercicio 2: Event-Driven Processing con S3 y Lambda

**Objetivo**: Crear un pipeline de procesamiento de imágenes serverless.

### Contexto

Cuando un usuario sube una imagen a S3, automáticamente:

1. Generar thumbnail (200x200)
2. Generar versión medium (800x800)
3. Guardar metadata en DynamoDB

### Arquitectura

```
S3 Upload (original/)
    ↓ trigger
Lambda (processImage)
    ├→ S3 (thumbnails/)
    ├→ S3 (medium/)
    └→ DynamoDB (metadata)
```

### Solución

```typescript
// src/handlers/imageProcessor.ts
import { S3Event } from "aws-lambda";
import AWS from "aws-sdk";
import sharp from "sharp";

const s3 = new AWS.S3();
const dynamoDB = new AWS.DynamoDB.DocumentClient();

const BUCKET_NAME = process.env.BUCKET_NAME!;
const TABLE_NAME = process.env.TABLE_NAME!;

export const handler = async (event: S3Event): Promise<void> => {
  for (const record of event.Records) {
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, " "));

    console.log(`Processing image: ${key}`);

    // Download original image
    const original = await s3
      .getObject({
        Bucket: BUCKET_NAME,
        Key: key,
      })
      .promise();

    const imageBuffer = original.Body as Buffer;

    // Get image metadata
    const metadata = await sharp(imageBuffer).metadata();

    // Generate thumbnail (200x200)
    const thumbnail = await sharp(imageBuffer)
      .resize(200, 200, { fit: "cover" })
      .jpeg({ quality: 80 })
      .toBuffer();

    await s3
      .putObject({
        Bucket: BUCKET_NAME,
        Key: `thumbnails/${key}`,
        Body: thumbnail,
        ContentType: "image/jpeg",
      })
      .promise();

    console.log(`Thumbnail created: thumbnails/${key}`);

    // Generate medium (800x800)
    const medium = await sharp(imageBuffer)
      .resize(800, 800, { fit: "inside" })
      .jpeg({ quality: 90 })
      .toBuffer();

    await s3
      .putObject({
        Bucket: BUCKET_NAME,
        Key: `medium/${key}`,
        Body: medium,
        ContentType: "image/jpeg",
      })
      .promise();

    console.log(`Medium created: medium/${key}`);

    // Save metadata to DynamoDB
    await dynamoDB
      .put({
        TableName: TABLE_NAME,
        Item: {
          id: key,
          originalKey: key,
          thumbnailKey: `thumbnails/${key}`,
          mediumKey: `medium/${key}`,
          width: metadata.width,
          height: metadata.height,
          format: metadata.format,
          size: imageBuffer.length,
          uploadedAt: new Date().toISOString(),
        },
      })
      .promise();

    console.log(`Metadata saved for ${key}`);
  }
};
```

**SAM Template**:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31

Resources:
  ImageBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-images-bucket

  ImageMetadataTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: image-metadata
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH

  ImageProcessorFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./
      Handler: dist/handlers/imageProcessor.handler
      Runtime: nodejs18.x
      MemorySize: 1024
      Timeout: 60
      Environment:
        Variables:
          BUCKET_NAME: !Ref ImageBucket
          TABLE_NAME: !Ref ImageMetadataTable
      Policies:
        - S3CrudPolicy:
            BucketName: !Ref ImageBucket
        - DynamoDBCrudPolicy:
            TableName: !Ref ImageMetadataTable
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
                    Value: original/
```

**Test**:

```bash
# Upload image
aws s3 cp photo.jpg s3://my-images-bucket/original/photo.jpg

# Lambda se ejecuta automáticamente

# Ver thumbnails creados
aws s3 ls s3://my-images-bucket/thumbnails/
aws s3 ls s3://my-images-bucket/medium/

# Ver metadata en DynamoDB
aws dynamodb scan --table-name image-metadata
```

**Resultado esperado**:

✅ Pipeline automático de procesamiento de imágenes  
✅ Event-driven (solo se ejecuta cuando hay uploads)  
✅ Metadata guardada en DynamoDB  
✅ Escalado automático (100 uploads simultáneos → 100 Lambdas)

---

## ⭐⭐⭐ Ejercicio 3: Istio Traffic Management

**Objetivo**: Implementar canary deployment con Istio para desplegar una nueva versión gradualmente.

### Contexto

Tienes una app en v1 y quieres desplegar v2 con el siguiente plan:

1. Inicialmente: 100% tráfico a v1
2. Canary: 90% v1, 10% v2
3. Si v2 está OK: 50% v1, 50% v2
4. Finalmente: 100% v2

### Tareas

**3.1** Despliega v1 y v2 en Kubernetes.

**3.2** Configura VirtualService de Istio para canary deployment.

**3.3** Gradualmente incrementa tráfico a v2.

**3.4** Verifica distribución de tráfico.

### Solución

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
          env:
            - name: VERSION
              value: "1.0.0"

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
          env:
            - name: VERSION
              value: "2.0.0"

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

---
# virtual-service-canary-10.yaml
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
```

**Comandos**:

```bash
# 1. Instalar Istio si no está instalado
istioctl install --set profile=demo -y

# Habilitar sidecar injection
kubectl label namespace default istio-injection=enabled

# 2. Desplegar v1 y v2
kubectl apply -f deployment-v1.yaml
kubectl apply -f deployment-v2.yaml
kubectl apply -f service.yaml

# 3. Configurar Istio
kubectl apply -f destination-rule.yaml
kubectl apply -f virtual-service-canary-10.yaml

# 4. Test: 100 requests
for i in $(seq 1 100); do
  kubectl exec -it <test-pod> -- curl -s http://myapp-service/version | jq -r '.version'
done | sort | uniq -c

# Resultado esperado:
#  90 1.0.0
#  10 2.0.0

# 5. Incrementar a 50/50
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

# Test de nuevo
for i in $(seq 1 100); do
  kubectl exec -it <test-pod> -- curl -s http://myapp-service/version | jq -r '.version'
done | sort | uniq -c

# Resultado esperado:
#  50 1.0.0
#  50 2.0.0

# 6. Finalmente 100% v2
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
            subset: v2
          weight: 100
EOF

# 7. Eliminar deployment v1 (ya no se usa)
kubectl delete deployment myapp-v1
```

**Resultado esperado**:

✅ Canary deployment gradual (10% → 50% → 100%)  
✅ Zero-downtime durante todo el proceso  
✅ Control preciso de distribución de tráfico  
✅ Rollback instant si v2 falla (cambiar weights a 100% v1)

---

## ⭐⭐⭐⭐ Ejercicio 4: Sistema Completo Serverless + Service Mesh

**Objetivo**: Crear un sistema híbrido con funciones serverless (AWS Lambda) y microservicios en Kubernetes con Istio.

### Arquitectura

```
Internet
    ↓
API Gateway
    ↓
┌──────────────────────────────────────┐
│ Lambda Functions (Serverless)       │
│ - User authentication                │
│ - Order processing                   │
└──────────────────────────────────────┘
    ↓ invoke K8s API
┌──────────────────────────────────────┐
│ Kubernetes + Istio (Microservices)  │
│ - Product Service                    │
│ - Inventory Service                  │
│ - Notification Service               │
└──────────────────────────────────────┘
```

### Componentes

**Lambda: Order Processing**

```typescript
// src/lambda/processOrder.ts
import { APIGatewayProxyEvent, APIGatewayProxyResult } from "aws-lambda";
import axios from "axios";

const K8S_API_URL = process.env.K8S_API_URL!;

export const handler = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  const body = JSON.parse(event.body || "{}");
  const { userId, productId, quantity } = body;

  try {
    // 1. Check inventory (Kubernetes service)
    const inventoryResponse = await axios.get(
      `${K8S_API_URL}/api/inventory/${productId}`
    );

    if (inventoryResponse.data.stock < quantity) {
      return {
        statusCode: 400,
        body: JSON.stringify({ error: "Insufficient stock" }),
      };
    }

    // 2. Reserve stock
    await axios.post(`${K8S_API_URL}/api/inventory/reserve`, {
      productId,
      quantity,
    });

    // 3. Create order
    const order = {
      id: Date.now().toString(),
      userId,
      productId,
      quantity,
      status: "pending",
      createdAt: new Date().toISOString(),
    };

    // Save to DynamoDB (omitido para brevedad)

    // 4. Trigger notification (Kubernetes service)
    await axios.post(`${K8S_API_URL}/api/notifications`, {
      userId,
      message: `Order ${order.id} created successfully`,
    });

    return {
      statusCode: 201,
      body: JSON.stringify(order),
    };
  } catch (error) {
    console.error("Error processing order:", error);
    return {
      statusCode: 500,
      body: JSON.stringify({ error: "Internal server error" }),
    };
  }
};
```

**Kubernetes: Inventory Service**

```typescript
// services/inventory/src/index.ts
import express from "express";
import { Pool } from "pg";

const app = express();
app.use(express.json());

const pool = new Pool({
  host: process.env.DATABASE_HOST,
  database: "inventory",
  user: "postgres",
  password: process.env.DATABASE_PASSWORD,
});

app.get("/api/inventory/:productId", async (req, res) => {
  const { productId } = req.params;

  const result = await pool.query(
    "SELECT stock FROM inventory WHERE product_id = $1",
    [productId]
  );

  if (result.rows.length === 0) {
    return res.status(404).json({ error: "Product not found" });
  }

  res.json({ productId, stock: result.rows[0].stock });
});

app.post("/api/inventory/reserve", async (req, res) => {
  const { productId, quantity } = req.body;

  await pool.query(
    "UPDATE inventory SET stock = stock - $1 WHERE product_id = $2",
    [quantity, productId]
  );

  res.json({ success: true });
});

const PORT = 8080;
app.listen(PORT, () => {
  console.log(`Inventory service running on port ${PORT}`);
});
```

**Istio: Virtual Service con Circuit Breaker**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: inventory-circuit-breaker
spec:
  host: inventory-service
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
      maxEjectionPercent: 50
```

**Resultado esperado**:

✅ Lambda procesa orders (serverless, pay-per-request)  
✅ Kubernetes ejecuta servicios con estado (inventory, notifications)  
✅ Istio gestiona tráfico entre servicios (circuit breaking, retries)  
✅ Sistema híbrido optimizado (serverless donde tiene sentido, containers para servicios stateful)

---

## Resumen de Ejercicios

| Ejercicio | Dificultad | Conceptos                                                |
| --------- | ---------- | -------------------------------------------------------- |
| 1         | ⭐         | AWS Lambda, DynamoDB, API Gateway                        |
| 2         | ⭐⭐       | Event-driven processing, S3 triggers, image processing   |
| 3         | ⭐⭐⭐     | Istio, canary deployment, traffic management             |
| 4         | ⭐⭐⭐⭐   | Sistema híbrido serverless + Kubernetes + Istio          |
