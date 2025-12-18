# Contratos de Servicio y Liskov Substitution Principle

## Objetivos
- Aplicar LSP en contratos entre microservicios
- Implementar Consumer-Driven Contracts
- Usar OpenAPI/Swagger para design-by-contract
- Realizar contract testing con Pact

## 1. LSP en Microservicios

### Principio de Sustitución en Servicios Distribuidos

En microservicios, LSP se traduce a: **un servicio puede ser reemplazado por otro que implemente el mismo contrato sin afectar a los consumidores**.

```java
// ❌ MAL: Breaking change viola LSP
// V1 del servicio
@RestController
public class OrderServiceV1 {
    @PostMapping("/orders")
    public OrderResponse createOrder(@RequestBody OrderRequest request) {
        return new OrderResponse(orderId, "PENDING");
    }
}

// V2 rompe el contrato (viola LSP)
@RestController
public class OrderServiceV2 {
    @PostMapping("/orders")
    public OrderResponse createOrder(@RequestBody OrderRequest request) {
        // ❌ Ahora requiere campo adicional obligatorio
        if (request.getPaymentMethod() == null) {
            throw new IllegalArgumentException("paymentMethod is required");
        }
        
        // ❌ Cambia el formato del response
        return new OrderResponse(
            orderId,
            new OrderStatus("PENDING", LocalDateTime.now()),  // Estructura diferente
            request.getPaymentMethod()  // Campo nuevo
        );
    }
}
```

```java
// ✅ BIEN: Backward compatible evolution (respeta LSP)
@RestController
public class OrderServiceV2 {
    @PostMapping("/orders")
    public OrderResponse createOrder(@RequestBody OrderRequest request) {
        // ✅ Campo opcional (default value)
        String paymentMethod = request.getPaymentMethod();
        if (paymentMethod == null) {
            paymentMethod = "CASH";  // Default
        }
        
        // ✅ Response compatible con V1 (solo campos adicionales opcionales)
        OrderResponse response = new OrderResponse(orderId, "PENDING");
        response.setPaymentMethod(paymentMethod);  // Campo nuevo opcional
        response.setCreatedAt(LocalDateTime.now());  // Metadata adicional
        
        return response;
    }
}
```

## 2. Consumer-Driven Contracts

### Definición de Contratos

```typescript
// consumer-contract.ts (definido por el consumidor)
export interface UserServiceContract {
    // Contrato mínimo que el consumidor necesita
    getUser(userId: string): Promise<UserDTO>;
    updateUser(userId: string, data: Partial<UserDTO>): Promise<void>;
}

export interface UserDTO {
    id: string;
    name: string;
    email: string;
    // El proveedor puede incluir campos adicionales,
    // pero estos son los mínimos requeridos por el consumidor
}

// Provider debe respetar este contrato
export class UserServiceClient implements UserServiceContract {
    async getUser(userId: string): Promise<UserDTO> {
        const response = await fetch(`/api/users/${userId}`);
        const data = await response.json();
        
        // Validar que cumple el contrato
        if (!data.id || !data.name || !data.email) {
            throw new ContractViolationError('Missing required fields');
        }
        
        return {
            id: data.id,
            name: data.name,
            email: data.email
        };
    }
    
    async updateUser(userId: string, data: Partial<UserDTO>): Promise<void> {
        await fetch(`/api/users/${userId}`, {
            method: 'PUT',
            body: JSON.stringify(data)
        });
    }
}
```

### Pact Contract Testing

```javascript
// consumer-pact.test.js
const { Pact } = require('@pact-foundation/pact');
const { UserServiceClient } = require('./user-service-client');

describe('User Service Contract', () => {
    const provider = new Pact({
        consumer: 'OrderService',
        provider: 'UserService',
        port: 8080
    });
    
    beforeAll(() => provider.setup());
    afterAll(() => provider.finalize());
    
    describe('GET /users/:id', () => {
        const userId = '123';
        const expectedUser = {
            id: userId,
            name: 'John Doe',
            email: 'john@example.com'
        };
        
        beforeEach(() => {
            return provider.addInteraction({
                state: 'user 123 exists',
                uponReceiving: 'a request for user 123',
                withRequest: {
                    method: 'GET',
                    path: `/api/users/${userId}`
                },
                willRespondWith: {
                    status: 200,
                    headers: { 'Content-Type': 'application/json' },
                    body: expectedUser
                }
            });
        });
        
        it('returns user data matching the contract', async () => {
            const client = new UserServiceClient('http://localhost:8080');
            const user = await client.getUser(userId);
            
            expect(user).toEqual(expectedUser);
        });
    });
    
    describe('GET /users/:id when user does not exist', () => {
        const userId = '999';
        
        beforeEach(() => {
            return provider.addInteraction({
                state: 'user 999 does not exist',
                uponReceiving: 'a request for user 999',
                withRequest: {
                    method: 'GET',
                    path: `/api/users/${userId}`
                },
                willRespondWith: {
                    status: 404,
                    headers: { 'Content-Type': 'application/json' },
                    body: {
                        error: 'User not found',
                        code: 'USER_NOT_FOUND'
                    }
                }
            });
        });
        
        it('throws appropriate error', async () => {
            const client = new UserServiceClient('http://localhost:8080');
            
            await expect(client.getUser(userId))
                .rejects
                .toThrow('User not found');
        });
    });
});
```

## 3. OpenAPI/Swagger para Design-by-Contract

```yaml
# user-service-api.yaml
openapi: 3.0.0
info:
  title: User Service API
  version: 2.0.0
  description: User management service contract
  
paths:
  /users/{userId}:
    get:
      summary: Get user by ID
      operationId: getUser
      parameters:
        - name: userId
          in: path
          required: true
          schema:
            type: string
            pattern: '^[a-zA-Z0-9-]+$'
      responses:
        '200':
          description: User found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          description: User not found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
                
    put:
      summary: Update user
      operationId: updateUser
      parameters:
        - name: userId
          in: path
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserUpdate'
      responses:
        '200':
          description: User updated
        '400':
          description: Invalid request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

components:
  schemas:
    User:
      type: object
      required:
        - id
        - name
        - email
      properties:
        id:
          type: string
          readOnly: true
        name:
          type: string
          minLength: 1
          maxLength: 100
        email:
          type: string
          format: email
        phone:
          type: string
          nullable: true  # Campo opcional
        avatarUrl:
          type: string
          format: uri
          nullable: true
        createdAt:
          type: string
          format: date-time
          readOnly: true
        # Nuevos campos deben ser opcionales para mantener LSP
        
    UserUpdate:
      type: object
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 100
        email:
          type: string
          format: email
        phone:
          type: string
          nullable: true
          
    Error:
      type: object
      required:
        - error
        - code
      properties:
        error:
          type: string
        code:
          type: string
        details:
          type: object
```

### Validación Automática de Contratos

```java
@RestController
@Validated
public class UserController {
    
    // Validación automática contra OpenAPI spec
    @GetMapping("/users/{userId}")
    @Operation(summary = "Get user by ID")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "User found",
            content = @Content(schema = @Schema(implementation = UserDTO.class))),
        @ApiResponse(responseCode = "404", description = "User not found",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    })
    public ResponseEntity<UserDTO> getUser(
        @PathVariable 
        @Pattern(regexp = "^[a-zA-Z0-9-]+$", message = "Invalid userId format")
        String userId
    ) {
        return userService.findById(userId)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PutMapping("/users/{userId}")
    public ResponseEntity<Void> updateUser(
        @PathVariable String userId,
        @RequestBody @Valid UserUpdateDTO update
    ) {
        userService.update(userId, update);
        return ResponseEntity.ok().build();
    }
}

// DTO con validación Bean Validation
public class UserUpdateDTO {
    @NotBlank(message = "Name is required")
    @Size(min = 1, max = 100, message = "Name must be between 1 and 100 characters")
    private String name;
    
    @Email(message = "Invalid email format")
    private String email;
    
    @Pattern(regexp = "^\\+?[0-9]{10,15}$", message = "Invalid phone format")
    private String phone;  // Nullable
    
    // Getters/setters
}
```

## 4. Evolución de Contratos Sin Breaking Changes

### Estrategias de Compatibilidad

```csharp
// V1 del contrato
public class OrderV1
{
    public string Id { get; set; }
    public string Status { get; set; }
    public decimal Total { get; set; }
}

// V2: Agregar campos (compatible hacia atrás)
public class OrderV2
{
    public string Id { get; set; }
    public string Status { get; set; }
    public decimal Total { get; set; }
    
    // Nuevos campos opcionales
    [JsonProperty(NullValueHandling = NullValueHandling.Ignore)]
    public string PaymentMethod { get; set; }
    
    [JsonProperty(NullValueHandling = NullValueHandling.Ignore)]
    public DateTime? CreatedAt { get; set; }
    
    [JsonProperty(NullValueHandling = NullValueHandling.Ignore)]
    public List<string> Tags { get; set; }
}

// V3: Cambiar estructura (requiere versioning)
public class OrderV3
{
    public string Id { get; set; }
    
    // ❌ Cambio estructural - requiere nueva versión
    public OrderStatusInfo Status { get; set; }  // Era string, ahora objeto
    
    public MoneyAmount Total { get; set; }  // Era decimal, ahora objeto
}

// Solución: Mantener ambas versiones
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersV1Controller : ControllerBase
{
    [HttpGet("{id}")]
    public ActionResult<OrderV1> GetOrder(string id)
    {
        var order = _orderService.GetOrder(id);
        return new OrderV1
        {
            Id = order.Id,
            Status = order.Status.Code,  // Mapping de v3 a v1
            Total = order.Total.Amount
        };
    }
}

[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersV2Controller : ControllerBase
{
    [HttpGet("{id}")]
    public ActionResult<OrderV2> GetOrder(string id)
    {
        var order = _orderService.GetOrder(id);
        return new OrderV2
        {
            Id = order.Id,
            Status = order.Status.Code,
            Total = order.Total.Amount,
            PaymentMethod = order.PaymentMethod,
            CreatedAt = order.CreatedAt,
            Tags = order.Tags
        };
    }
}

[ApiVersion("3.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersV3Controller : ControllerBase
{
    [HttpGet("{id}")]
    public ActionResult<OrderV3> GetOrder(string id)
    {
        return _orderService.GetOrder(id);  // Modelo nativo v3
    }
}
```

## 5. Contract Testing en CI/CD

```yaml
# .github/workflows/contract-tests.yml
name: Contract Tests

on:
  pull_request:
    branches: [main]

jobs:
  consumer-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Run Consumer Contract Tests
        run: npm run test:pact
      
      - name: Publish Pacts to Broker
        run: |
          npm run pact:publish -- \
            --consumer-app-version=${{ github.sha }} \
            --broker-base-url=${{ secrets.PACT_BROKER_URL }} \
            --broker-token=${{ secrets.PACT_BROKER_TOKEN }}
  
  provider-verification:
    runs-on: ubuntu-latest
    needs: consumer-tests
    steps:
      - uses: actions/checkout@v2
      
      - name: Start Provider Service
        run: docker-compose up -d user-service
      
      - name: Verify Provider Against Pacts
        run: |
          npm run pact:verify -- \
            --provider=UserService \
            --provider-base-url=http://localhost:8080 \
            --broker-base-url=${{ secrets.PACT_BROKER_URL }} \
            --broker-token=${{ secrets.PACT_BROKER_TOKEN }} \
            --publish-verification-result \
            --provider-version-tag=${{ github.sha }}
      
      - name: Can I Deploy?
        run: |
          pact-broker can-i-deploy \
            --pacticipant=OrderService \
            --version=${{ github.sha }} \
            --to-environment=production \
            --broker-base-url=${{ secrets.PACT_BROKER_URL }} \
            --broker-token=${{ secrets.PACT_BROKER_TOKEN }}
```

## 6. Ejemplo Completo: E-commerce Service Contracts

```python
# contracts/payment_service_contract.py
from typing import Protocol
from decimal import Decimal

class PaymentServiceContract(Protocol):
    """Contrato que todos los payment providers deben cumplir"""
    
    def authorize_payment(
        self,
        amount: Decimal,
        currency: str,
        payment_method: str,
        customer_id: str
    ) -> PaymentAuthorizationResult:
        """
        Preconditions:
        - amount > 0
        - currency in ['USD', 'EUR', 'GBP']
        - payment_method not empty
        
        Postconditions:
        - Returns PaymentAuthorizationResult with transaction_id
        - If declined, returns result with declined=True
        - Never raises exception for business logic failures
        """
        ...
    
    def capture_payment(
        self,
        transaction_id: str,
        amount: Decimal
    ) -> PaymentCaptureResult:
        """
        Preconditions:
        - transaction_id must be from previous authorization
        - amount <= authorized amount
        
        Postconditions:
        - Returns capture result
        - Idempotent (same transaction_id = same result)
        """
        ...

# Implementation: Stripe Provider
class StripePaymentService:
    """Implementación que respeta el contrato"""
    
    def authorize_payment(
        self,
        amount: Decimal,
        currency: str,
        payment_method: str,
        customer_id: str
    ) -> PaymentAuthorizationResult:
        # Validar precondiciones
        assert amount > 0, "Amount must be positive"
        assert currency in ['USD', 'EUR', 'GBP'], "Unsupported currency"
        
        try:
            stripe_result = stripe.PaymentIntent.create(
                amount=int(amount * 100),
                currency=currency.lower(),
                payment_method=payment_method,
                customer=customer_id,
                capture_method='manual'
            )
            
            return PaymentAuthorizationResult(
                transaction_id=stripe_result.id,
                authorized=stripe_result.status == 'requires_capture',
                declined=stripe_result.status == 'declined',
                message=stripe_result.get('status_message', '')
            )
        except stripe.error.CardError as e:
            # No raise - devolver resultado estructurado
            return PaymentAuthorizationResult(
                transaction_id=None,
                authorized=False,
                declined=True,
                message=str(e)
            )

# Contract Test
def test_payment_authorization_contract():
    service = StripePaymentService()
    
    # Test caso exitoso
    result = service.authorize_payment(
        amount=Decimal('100.00'),
        currency='USD',
        payment_method='pm_card_visa',
        customer_id='cus_123'
    )
    
    assert result.transaction_id is not None
    assert result.authorized == True
    assert result.declined == False
    
    # Test caso declinado
    result = service.authorize_payment(
        amount=Decimal('100.00'),
        currency='USD',
        payment_method='pm_card_chargeDeclined',
        customer_id='cus_123'
    )
    
    assert result.authorized == False
    assert result.declined == True
    
    # Test precondition violation
    with pytest.raises(AssertionError):
        service.authorize_payment(
            amount=Decimal('-10.00'),  # Invalid
            currency='USD',
            payment_method='pm_card_visa',
            customer_id='cus_123'
        )
```

## Resumen

### Principios LSP en Microservicios

1. **Backward Compatibility**: Nuevas versiones deben aceptar requests de versiones antiguas
2. **Precondiciones no más estrictas**: No agregar validaciones que rechacen requests válidos anteriormente
3. **Postcondiciones no más débiles**: Garantías del servicio no deben disminuir
4. **Contratos explícitos**: OpenAPI, Pact, Protocol Buffers
5. **Contract Testing**: Validación automática de contratos

### Breaking Changes vs Non-Breaking Changes

**Non-Breaking (LSP compliant)**:
- ✅ Agregar campos opcionales al response
- ✅ Agregar nuevos endpoints
- ✅ Hacer campos obligatorios opcionales en request
- ✅ Relajar validaciones

**Breaking (LSP violation)**:
- ❌ Remover campos del response
- ❌ Cambiar tipos de datos
- ❌ Hacer campos opcionales obligatorios
- ❌ Endurecer validaciones
- ❌ Cambiar semántica de operaciones

**Solución para Breaking Changes**: Versioning explícito (/v1/, /v2/)
