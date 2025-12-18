# Database per Service y Single Responsibility Principle

## Objetivos
- Aplicar SRP a nivel de datos en microservicios
- Implementar Database per Service pattern
- Manejar transacciones distribuidas con Saga pattern
- Diseñar estrategias de consistencia eventual

## 1. Database per Service Pattern

Cada microservicio debe tener su propia base de datos para mantener independencia y SRP.

```
❌ MAL: Shared Database Anti-Pattern
┌─────────────────────────────────────────┐
│        Shared Database                   │
│  ┌──────────────────────────────────┐   │
│  │ Orders | Products | Users | ...  │   │
│  └──────────────────────────────────┘   │
└──────┬───────────┬──────────┬───────────┘
       │           │          │
   ┌───┴──┐   ┌───┴──┐  ┌───┴──┐
   │Order │   │Product│  │User  │
   │Service   │Service│  │Service
   └──────┘   └──────┘  └──────┘

Problemas:
- Acoplamiento fuerte
- Violación de SRP (servicios comparten esquema)
- Scaling independiente imposible
- Tecnología única (no polyglot persistence)
```

```
✅ BIEN: Database per Service
┌──────────┐   ┌──────────┐   ┌──────────┐
│  Order   │   │ Product  │   │   User   │
│ Service  │   │ Service  │   │ Service  │
└────┬─────┘   └────┬─────┘   └────┬─────┘
     │              │              │
┌────┴─────┐   ┌───┴──────┐  ┌───┴──────┐
│ Orders   │   │ Products │  │  Users   │
│   DB     │   │    DB    │  │    DB    │
│(Postgres)│   │  (Mongo) │  │ (MySQL)  │
└──────────┘   └──────────┘  └──────────┘

Beneficios:
- Loose coupling
- Polyglot persistence
- Independent scaling
- Schema evolution independiente
```

## 2. Implementación: Database per Service

### Order Service (PostgreSQL)

```typescript
// order-service/src/domain/Order.ts
export class Order {
    constructor(
        public readonly id: string,
        public readonly customerId: string,  // Reference, no join
        private items: OrderItem[],
        private status: OrderStatus,
        private total: Money
    ) {}
    
    // Solo lógica de Order - SRP
    addItem(item: OrderItem): void {
        this.items.push(item);
        this.recalculateTotal();
    }
    
    confirm(): void {
        if (this.status !== OrderStatus.PENDING) {
            throw new Error('Only pending orders can be confirmed');
        }
        this.status = OrderStatus.CONFIRMED;
    }
    
    private recalculateTotal(): void {
        this.total = this.items.reduce(
            (sum, item) => sum.add(item.price.multiply(item.quantity)),
            Money.zero()
        );
    }
}

// order-service/src/infrastructure/OrderRepository.ts
export class PostgresOrderRepository implements OrderRepository {
    constructor(private pool: Pool) {}
    
    async save(order: Order): Promise<void> {
        const client = await this.pool.connect();
        try {
            await client.query('BEGIN');
            
            await client.query(
                `INSERT INTO orders (id, customer_id, status, total_amount, total_currency)
                 VALUES ($1, $2, $3, $4, $5)
                 ON CONFLICT (id) DO UPDATE SET
                 status = $3, total_amount = $4`,
                [order.id, order.customerId, order.status, order.total.amount, order.total.currency]
            );
            
            await client.query('COMMIT');
        } catch (e) {
            await client.query('ROLLBACK');
            throw e;
        } finally {
            client.release();
        }
    }
}
```

### Product Service (MongoDB)

```typescript
// product-service/src/domain/Product.ts
export class Product {
    constructor(
        public readonly id: string,
        private name: string,
        private description: string,
        private price: Money,
        private inventory: Inventory
    ) {}
    
    // Solo lógica de Product - SRP
    updatePrice(newPrice: Money): void {
        if (newPrice.amount <= 0) {
            throw new Error('Price must be positive');
        }
        this.price = newPrice;
    }
    
    reserve(quantity: number): ReservationResult {
        return this.inventory.reserve(quantity);
    }
}

// product-service/src/infrastructure/ProductRepository.ts
export class MongoProductRepository implements ProductRepository {
    constructor(private collection: Collection<ProductDocument>) {}
    
    async save(product: Product): Promise<void> {
        await this.collection.updateOne(
            { _id: product.id },
            {
                $set: {
                    name: product.name,
                    description: product.description,
                    price: {
                        amount: product.price.amount,
                        currency: product.price.currency
                    },
                    inventory: product.inventory.toDocument()
                }
            },
            { upsert: true }
        );
    }
    
    async findById(id: string): Promise<Product | null> {
        const doc = await this.collection.findOne({ _id: id });
        if (!doc) return null;
        
        return Product.fromDocument(doc);
    }
}
```

## 3. Desafío: Transacciones Distribuidas

No podemos usar transacciones ACID tradicionales. Necesitamos **Saga Pattern**.

### Saga Choreography (Event-Driven)

```typescript
// Scenario: Create Order → Reserve Inventory → Process Payment → Ship Order

// 1. Order Service emits OrderCreated
export class OrderService {
    constructor(
        private orderRepository: OrderRepository,
        private eventBus: EventBus
    ) {}
    
    async createOrder(command: CreateOrderCommand): Promise<Order> {
        const order = new Order(
            generateId(),
            command.customerId,
            command.items,
            OrderStatus.PENDING,
            calculateTotal(command.items)
        );
        
        await this.orderRepository.save(order);
        
        // Emit event
        await this.eventBus.publish(new OrderCreatedEvent({
            orderId: order.id,
            items: order.items.map(item => ({
                productId: item.productId,
                quantity: item.quantity
            }))
        }));
        
        return order;
    }
}

// 2. Inventory Service listens and reserves
export class InventoryEventHandler {
    constructor(
        private inventoryService: InventoryService,
        private eventBus: EventBus
    ) {}
    
    @EventListener(OrderCreatedEvent)
    async handleOrderCreated(event: OrderCreatedEvent): Promise<void> {
        try {
            // Reserve inventory
            const reservationId = await this.inventoryService.reserve(event.items);
            
            // Success: emit InventoryReserved
            await this.eventBus.publish(new InventoryReservedEvent({
                orderId: event.orderId,
                reservationId
            }));
        } catch (error) {
            // Failure: emit InventoryReservationFailed
            await this.eventBus.publish(new InventoryReservationFailedEvent({
                orderId: event.orderId,
                reason: error.message
            }));
        }
    }
}

// 3. Payment Service listens
export class PaymentEventHandler {
    @EventListener(InventoryReservedEvent)
    async handleInventoryReserved(event: InventoryReservedEvent): Promise<void> {
        try {
            const paymentId = await this.paymentService.charge(event.orderId);
            
            await this.eventBus.publish(new PaymentProcessedEvent({
                orderId: event.orderId,
                paymentId
            }));
        } catch (error) {
            // Payment failed: trigger compensating transaction
            await this.eventBus.publish(new PaymentFailedEvent({
                orderId: event.orderId,
                reservationId: event.reservationId,
                reason: error.message
            }));
        }
    }
    
    // Compensating transaction
    @EventListener(PaymentFailedEvent)
    async handlePaymentFailed(event: PaymentFailedEvent): Promise<void> {
        // Release inventory reservation
        await this.inventoryService.release(event.reservationId);
        
        // Mark order as failed
        await this.eventBus.publish(new OrderFailedEvent({
            orderId: event.orderId,
            reason: event.reason
        }));
    }
}

// 4. Order Service updates status
export class OrderEventHandler {
    @EventListener(PaymentProcessedEvent)
    async handlePaymentProcessed(event: PaymentProcessedEvent): Promise<void> {
        const order = await this.orderRepository.findById(event.orderId);
        order.confirm();
        await this.orderRepository.save(order);
    }
    
    @EventListener(OrderFailedEvent)
    async handleOrderFailed(event: OrderFailedEvent): Promise<void> {
        const order = await this.orderRepository.findById(event.orderId);
        order.cancel(event.reason);
        await this.orderRepository.save(order);
    }
}
```

### Saga Orchestration (Coordinator)

```java
// Centralized orchestrator
@Service
public class OrderSaga {
    
    private final OrderService orderService;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final ShippingService shippingService;
    
    @Transactional
    public OrderResult executeCreateOrderSaga(CreateOrderCommand command) {
        String sagaId = UUID.randomUUID().toString();
        SagaState state = new SagaState(sagaId);
        
        try {
            // Step 1: Create Order
            state.addStep("createOrder");
            String orderId = orderService.createOrder(command);
            state.completeStep("createOrder", orderId);
            
            // Step 2: Reserve Inventory
            state.addStep("reserveInventory");
            String reservationId = inventoryService.reserve(command.getItems());
            state.completeStep("reserveInventory", reservationId);
            
            // Step 3: Process Payment
            state.addStep("processPayment");
            String paymentId = paymentService.charge(orderId, command.getPaymentMethod());
            state.completeStep("processPayment", paymentId);
            
            // Step 4: Create Shipment
            state.addStep("createShipment");
            String shipmentId = shippingService.schedule(orderId, command.getShippingAddress());
            state.completeStep("createShipment", shipmentId);
            
            return OrderResult.success(orderId);
            
        } catch (Exception e) {
            // Compensate in reverse order
            compensate(state);
            return OrderResult.failure(e.getMessage());
        }
    }
    
    private void compensate(SagaState state) {
        List<String> completedSteps = state.getCompletedSteps();
        Collections.reverse(completedSteps);
        
        for (String step : completedSteps) {
            try {
                switch (step) {
                    case "createShipment":
                        shippingService.cancel(state.getData("createShipment"));
                        break;
                    case "processPayment":
                        paymentService.refund(state.getData("processPayment"));
                        break;
                    case "reserveInventory":
                        inventoryService.release(state.getData("reserveInventory"));
                        break;
                    case "createOrder":
                        orderService.cancel(state.getData("createOrder"));
                        break;
                }
            } catch (Exception e) {
                // Log compensation failure
                log.error("Compensation failed for step: {}", step, e);
                // Could trigger manual intervention
            }
        }
    }
}

// Persistent saga state
@Entity
@Table(name = "saga_state")
public class SagaState {
    @Id
    private String sagaId;
    
    @ElementCollection
    @CollectionTable(name = "saga_steps")
    private List<SagaStep> steps = new ArrayList<>();
    
    @Enumerated(EnumType.STRING)
    private SagaStatus status;
    
    public void addStep(String name) {
        steps.add(new SagaStep(name, StepStatus.STARTED));
    }
    
    public void completeStep(String name, String data) {
        steps.stream()
            .filter(s -> s.getName().equals(name))
            .findFirst()
            .ifPresent(s -> {
                s.setStatus(StepStatus.COMPLETED);
                s.setData(data);
            });
    }
}
```

## 4. Consistencia Eventual

```python
# Read Model Projection (CQRS)
# Construir vista denormalizada para queries

# Write Model: Order Service
class OrderCreatedHandler:
    def handle(self, event: OrderCreatedEvent):
        # Update read model asynchronously
        asyncio.create_task(
            self.projection_service.project_order_created(event)
        )

# Read Model: Query Service
class OrderProjectionService:
    def __init__(self, redis: Redis):
        self.redis = redis
    
    async def project_order_created(self, event: OrderCreatedEvent):
        # Fetch customer data (from User Service API)
        customer = await self.user_service.get_customer(event.customer_id)
        
        # Fetch product data (from Product Service API)
        products = await self.product_service.get_products(
            [item.product_id for item in event.items]
        )
        
        # Create denormalized view
        order_view = {
            'order_id': event.order_id,
            'customer': {
                'id': customer.id,
                'name': customer.name,
                'email': customer.email
            },
            'items': [
                {
                    'product_id': item.product_id,
                    'product_name': products[item.product_id].name,
                    'quantity': item.quantity,
                    'price': products[item.product_id].price
                }
                for item in event.items
            ],
            'status': event.status,
            'created_at': event.created_at
        }
        
        # Store in Redis for fast queries
        await self.redis.setex(
            f'order_view:{event.order_id}',
            3600,  # 1 hour TTL
            json.dumps(order_view)
        )
    
    async def get_order_view(self, order_id: str) -> dict:
        cached = await self.redis.get(f'order_view:{order_id}')
        if cached:
            return json.loads(cached)
        
        # Fallback: reconstruct from events
        return await self.rebuild_from_events(order_id)
```

## 5. Shared Data: Reference Data Pattern

Para datos compartidos (ej: códigos de país, monedas), usar **Reference Data Service**.

```typescript
// Reference Data Service (read-only)
export class ReferenceDataService {
    private cache: Map<string, any> = new Map();
    
    async getCountries(): Promise<Country[]> {
        if (!this.cache.has('countries')) {
            const countries = await this.repository.findAllCountries();
            this.cache.set('countries', countries);
            
            // Invalidate after 24h
            setTimeout(() => this.cache.delete('countries'), 24 * 3600 * 1000);
        }
        return this.cache.get('countries');
    }
    
    async getCurrencies(): Promise<Currency[]> {
        // Similar caching strategy
    }
}

// Other services use it via API, not direct DB access
export class OrderService {
    constructor(
        private referenceDataClient: ReferenceDataClient
    ) {}
    
    async validateOrder(order: Order): Promise<void> {
        const validCurrencies = await this.referenceDataClient.getCurrencies();
        if (!validCurrencies.some(c => c.code === order.currency)) {
            throw new Error(`Invalid currency: ${order.currency}`);
        }
    }
}
```

## Resumen

### Database per Service Benefits

| Aspecto | Benefit |
|---------|---------|
| **Acoplamiento** | Loose - no shared schema |
| **Escalabilidad** | Independent scaling |
| **Tecnología** | Polyglot persistence |
| **Deployment** | Independent |
| **Ownership** | Clear team boundaries |

### Saga Patterns Comparison

| Pattern | Pros | Cons | Use When |
|---------|------|------|----------|
| **Choreography** | Decentralized, simple | Hard to debug, no central view | Few services (2-4) |
| **Orchestration** | Centralized control, easier debugging | Single point of failure, complexity | Many services (5+) |

### Best Practices

1. ✅ **Un servicio, una base de datos**
2. ✅ **Usar eventos para sincronización**
3. ✅ **Implementar compensating transactions**
4. ✅ **Idempotent operations** (para retries)
5. ✅ **Eventual consistency** como default
6. ❌ **Evitar joins cross-service**
7. ❌ **Evitar distributed transactions (2PC)**
