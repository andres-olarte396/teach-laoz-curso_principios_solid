# Ejercicios: Contratos de Servicio y LSP

## ⭐ Ejercicio 1. Detectar Violaciones LSP en APIs

Analiza las siguientes evoluciones de API e identifica cuáles violan LSP:

**API V1**:
```json
GET /api/products/123
Response 200:
{
  "id": "123",
  "name": "Laptop",
  "price": 999.99
}
```

**Cambio A - API V2**:
```json
GET /api/products/123
Response 200:
{
  "id": "123",
  "name": "Laptop",
  "price": {
    "amount": 999.99,
    "currency": "USD"
  }
}
```

**Cambio B - API V2**:
```json
GET /api/products/123
Response 200:
{
  "id": "123",
  "name": "Laptop",
  "price": 999.99,
  "category": "Electronics"
}
```

**Cambio C - API V2**:
```json
POST /api/products
Request:
{
  "name": "Laptop",
  "price": 999.99,
  "category": "Electronics"  // Ahora obligatorio (antes opcional)
}
```

¿Cuáles cambios violan LSP? ¿Cómo los arreglarías?

<details>
<summary>💡 Solución</summary>

**Cambio A**: ❌ **Viola LSP** - Cambia estructura de `price` de number a object. Clientes que esperan `price: number` fallarán.

**Solución**:
```json
{
  "id": "123",
  "name": "Laptop",
  "price": 999.99,  // Mantener retrocompatibilidad
  "priceDetails": {  // Nuevo campo opcional
    "amount": 999.99,
    "currency": "USD"
  }
}
```

**Cambio B**: ✅ **No viola LSP** - Solo agrega campo opcional. Compatible hacia atrás.

**Cambio C**: ❌ **Viola LSP** - Hace campo obligatorio que antes era opcional. Requests válidos en V1 ahora fallan.

**Solución**:
```typescript
// Mantener category opcional con default
interface ProductCreateRequest {
    name: string;
    price: number;
    category?: string;  // Opcional
}

function createProduct(req: ProductCreateRequest): Product {
    const category = req.category ?? 'Uncategorized';  // Default
    // ...
}
```

</details>

---

## ⭐⭐ Ejercicio 2: Implementar Consumer-Driven Contract

Crea un contrato Pact para un servicio de notificaciones.

**Requisitos del consumidor**:
- Enviar email: `POST /notifications/email`
- Body: `{ "to": "user@example.com", "subject": "...", "body": "..." }`
- Response exitoso: `{ "id": "notif-123", "status": "sent" }`
- Error: `400` con `{ "error": "Invalid email" }`

**Tareas**:
1. Escribir Pact test del consumidor
2. Implementar provider que cumpla el contrato
3. Verificar provider contra Pact

<details>
<summary>💡 Solución</summary>

```javascript
// consumer-pact.test.js
const { Pact } = require('@pact-foundation/pact');
const { NotificationClient } = require('./notification-client');

describe('Notification Service Contract', () => {
    const provider = new Pact({
        consumer: 'OrderService',
        provider: 'NotificationService',
        port: 9000
    });
    
    beforeAll(() => provider.setup());
    afterEach(() => provider.verify());
    afterAll(() => provider.finalize());
    
    describe('Send Email', () => {
        it('sends email successfully', async () => {
            await provider.addInteraction({
                state: 'email service is available',
                uponReceiving: 'a request to send email',
                withRequest: {
                    method: 'POST',
                    path: '/notifications/email',
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: {
                        to: 'user@example.com',
                        subject: 'Order Confirmed',
                        body: 'Your order has been confirmed'
                    }
                },
                willRespondWith: {
                    status: 200,
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: {
                        id: Pact.Matchers.like('notif-123'),
                        status: 'sent'
                    }
                }
            });
            
            const client = new NotificationClient('http://localhost:9000');
            const result = await client.sendEmail({
                to: 'user@example.com',
                subject: 'Order Confirmed',
                body: 'Your order has been confirmed'
            });
            
            expect(result.status).toBe('sent');
            expect(result.id).toBeDefined();
        });
        
        it('returns error for invalid email', async () => {
            await provider.addInteraction({
                state: 'email service is available',
                uponReceiving: 'a request with invalid email',
                withRequest: {
                    method: 'POST',
                    path: '/notifications/email',
                    body: {
                        to: 'invalid-email',
                        subject: 'Test',
                        body: 'Test'
                    }
                },
                willRespondWith: {
                    status: 400,
                    body: {
                        error: 'Invalid email'
                    }
                }
            });
            
            const client = new NotificationClient('http://localhost:9000');
            
            await expect(
                client.sendEmail({
                    to: 'invalid-email',
                    subject: 'Test',
                    body: 'Test'
                })
            ).rejects.toThrow('Invalid email');
        });
    });
});

// Provider verification (notification-service)
const { Verifier } = require('@pact-foundation/pact');

describe('Notification Service Provider', () => {
    it('validates the expectations of OrderService', () => {
        return new Verifier({
            provider: 'NotificationService',
            providerBaseUrl: 'http://localhost:8080',
            pactUrls: ['./pacts/orderservice-notificationservice.json'],
            stateHandlers: {
                'email service is available': () => {
                    // Setup: ensure service is ready
                    return Promise.resolve();
                }
            }
        }).verifyProvider();
    });
});
```

</details>

---

## ⭐⭐⭐ Ejercicio 3: OpenAPI Schema Evolution

Tienes esta API OpenAPI v1:

```yaml
paths:
  /orders:
    post:
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [customerId, items]
              properties:
                customerId:
                  type: string
                items:
                  type: array
                  items:
                    type: object
                    properties:
                      productId:
                        type: string
                      quantity:
                        type: integer
      responses:
        '201':
          content:
            application/json:
              schema:
                type: object
                properties:
                  orderId:
                    type: string
                  status:
                    type: string
```

**Necesitas agregar**:
1. Campo `shippingAddress` (obligatorio para nuevos clientes)
2. Campo `promotionCode` (opcional)
3. Response debe incluir `estimatedDelivery`

**Tareas**:
- Evolucionar schema sin romper clientes existentes
- Documentar compatibilidad
- Escribir migration guide

<details>
<summary>💡 Solución</summary>

```yaml
# v2 - Backward compatible
paths:
  /orders:
    post:
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [customerId, items]  # NO agregar shippingAddress aquí
              properties:
                customerId:
                  type: string
                items:
                  type: array
                  items:
                    $ref: '#/components/schemas/OrderItem'
                shippingAddress:
                  type: object
                  nullable: true  # Opcional para retrocompatibilidad
                  properties:
                    street: {type: string}
                    city: {type: string}
                    zipCode: {type: string}
                promotionCode:
                  type: string
                  nullable: true
      responses:
        '201':
          content:
            application/json:
              schema:
                type: object
                properties:
                  orderId: {type: string}
                  status: {type: string}
                  estimatedDelivery:  # Nuevo campo opcional
                    type: string
                    format: date
                    nullable: true

# Migration Guide
# v1 → v2:
# - shippingAddress: Optional field. If omitted, uses customer's default address
# - promotionCode: Optional discount code
# - estimatedDelivery: Added to response (may be null if shipping not scheduled)
# 
# Breaking in v3 (future):
# - shippingAddress will become required
# - Migration window: 6 months
```

</details>

---

## ⭐⭐⭐⭐ Ejercicio 4: Contract Testing Pipeline Completo

Implementa un pipeline CI/CD con contract testing:

**Arquitectura**:
- Consumer: Order Service
- Providers: User Service, Product Service, Payment Service

**Requisitos**:
1. Pact tests en PR de consumer
2. Publish pacts a Pact Broker
3. Provider verification automática
4. Can-I-Deploy check antes de production
5. Webhook para notificar providers de nuevos pacts

<details>
<summary>💡 Solución</summary>

```yaml
# .github/workflows/consumer-ci.yml (Order Service)
name: Consumer Contract Tests

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run Pact Tests
        run: npm run test:pact
      
      - name: Publish Pacts
        if: github.ref == 'refs/heads/main'
        run: |
          npm run pact:publish -- \
            --consumer-app-version=${{ github.sha }} \
            --tag=${{ github.ref_name }} \
            --broker-base-url=${{ secrets.PACT_BROKER_URL }} \
            --broker-token=${{ secrets.PACT_BROKER_TOKEN }}
      
      - name: Can I Deploy to Production?
        if: github.ref == 'refs/heads/main'
        run: |
          docker run --rm \
            pactfoundation/pact-cli:latest \
            broker can-i-deploy \
            --pacticipant=OrderService \
            --version=${{ github.sha }} \
            --to-environment=production \
            --broker-base-url=${{ secrets.PACT_BROKER_URL }} \
            --broker-token=${{ secrets.PACT_BROKER_TOKEN }}

# .github/workflows/provider-ci.yml (User Service)
name: Provider Contract Verification

on:
  push:
    branches: [main]
  repository_dispatch:  # Triggered by Pact Broker webhook
    types: [pact-changed]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Start Provider Service
        run: docker-compose up -d user-service
      
      - name: Wait for Service
        run: |
          timeout 60 bash -c 'until curl -f http://localhost:8080/health; do sleep 2; done'
      
      - name: Verify Pacts
        run: |
          npm run pact:verify -- \
            --provider=UserService \
            --provider-base-url=http://localhost:8080 \
            --broker-base-url=${{ secrets.PACT_BROKER_URL }} \
            --broker-token=${{ secrets.PACT_BROKER_TOKEN }} \
            --publish-verification-results \
            --provider-version-tag=${{ github.sha }} \
            --enable-pending
      
      - name: Record Deployment
        if: success()
        run: |
          docker run --rm \
            pactfoundation/pact-cli:latest \
            broker record-deployment \
            --pacticipant=UserService \
            --version=${{ github.sha }} \
            --environment=production \
            --broker-base-url=${{ secrets.PACT_BROKER_URL }} \
            --broker-token=${{ secrets.PACT_BROKER_TOKEN }}

# pact-broker webhook configuration
{
  "events": [{
    "name": "contract_content_changed"
  }],
  "request": {
    "method": "POST",
    "url": "https://api.github.com/repos/my-org/user-service/dispatches",
    "headers": {
      "Authorization": "Bearer ${{ secrets.GITHUB_TOKEN }}",
      "Accept": "application/vnd.github.v3+json"
    },
    "body": {
      "event_type": "pact-changed",
      "client_payload": {
        "pact_url": "${pactbroker.pactUrl}",
        "consumer": "${pactbroker.consumerName}",
        "provider": "${pactbroker.providerName}"
      }
    }
  }
}
```

</details>

---

## Recursos Adicionales

- **Pact Documentation**: https://docs.pact.io/
- **OpenAPI Specification**: https://spec.openapis.org/oas/latest.html
- **Martin Fowler - Consumer-Driven Contracts**: https://martinfowler.com/articles/consumerDrivenContracts.html
- **Book**: "Testing Microservices with Pact" - Lewis Cowles
