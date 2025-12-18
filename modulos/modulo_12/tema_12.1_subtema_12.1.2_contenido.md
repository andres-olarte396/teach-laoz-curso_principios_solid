# Event Sourcing y CQRS

## Objetivos
- Comprender Event Sourcing como fuente de verdad
- Implementar CQRS (Command Query Responsibility Segregation)
- Diseñar projections y read models
- Aplicar SOLID a arquitecturas event-sourced

## 1. ¿Qué es Event Sourcing?

En lugar de guardar el **estado actual**, guardamos la **secuencia de eventos** que llevaron a ese estado.

```
Traditional Persistence:
┌──────────────────────────┐
│  Orders Table            │
│ ┌──────────────────────┐ │
│ │ id: 123              │ │
│ │ status: SHIPPED      │ │ ← Solo estado actual
│ │ total: 99.99         │ │
│ └──────────────────────┘ │
└──────────────────────────┘
   Perdimos el historial ❌

Event Sourcing:
┌──────────────────────────────────────┐
│  Event Store                         │
│ ┌──────────────────────────────────┐ │
│ │ OrderCreated     (v1) 10:00 AM   │ │
│ │ PaymentProcessed (v2) 10:01 AM   │ │ ← Historial completo
│ │ OrderShipped     (v3) 11:30 AM   │ │
│ └──────────────────────────────────┘ │
└──────────────────────────────────────┘
   Historial completo ✅
   Auditoría natural ✅
   Time travel ✅
```

### Reconstruir Estado desde Eventos

```typescript
// Events (immutable facts)
class OrderCreatedEvent {
    constructor(
        public readonly orderId: string,
        public readonly customerId: string,
        public readonly items: OrderItem[],
        public readonly version: number,
        public readonly timestamp: Date
    ) {}
}

class PaymentProcessedEvent {
    constructor(
        public readonly orderId: string,
        public readonly paymentId: string,
        public readonly amount: number,
        public readonly version: number,
        public readonly timestamp: Date
    ) {}
}

class OrderShippedEvent {
    constructor(
        public readonly orderId: string,
        public readonly trackingNumber: string,
        public readonly version: number,
        public readonly timestamp: Date
    ) {}
}

// Aggregate reconstructs from events
class Order {
    private id: string;
    private customerId: string;
    private items: OrderItem[] = [];
    private status: OrderStatus = OrderStatus.PENDING;
    private paymentId?: string;
    private trackingNumber?: string;
    private version: number = 0;
    
    // Load from events (replay history)
    static fromEvents(events: DomainEvent[]): Order {
        const order = new Order();
        events.forEach(event => order.apply(event));
        return order;
    }
    
    // Apply event to update state
    private apply(event: DomainEvent): void {
        if (event instanceof OrderCreatedEvent) {
            this.id = event.orderId;
            this.customerId = event.customerId;
            this.items = event.items;
            this.status = OrderStatus.PENDING;
        } else if (event instanceof PaymentProcessedEvent) {
            this.paymentId = event.paymentId;
            this.status = OrderStatus.PAID;
        } else if (event instanceof OrderShippedEvent) {
            this.trackingNumber = event.trackingNumber;
            this.status = OrderStatus.SHIPPED;
        }
        
        this.version = event.version;
    }
    
    // Commands generate new events
    ship(trackingNumber: string): OrderShippedEvent {
        if (this.status !== OrderStatus.PAID) {
            throw new Error('Can only ship paid orders');
        }
        
        return new OrderShippedEvent(
            this.id,
            trackingNumber,
            this.version + 1,
            new Date()
        );
    }
}

// Event Store
class EventStore {
    private events: Map<string, DomainEvent[]> = new Map();
    
    async save(aggregateId: string, event: DomainEvent): Promise<void> {
        if (!this.events.has(aggregateId)) {
            this.events.set(aggregateId, []);
        }
        
        const events = this.events.get(aggregateId)!;
        
        // Optimistic concurrency check
        if (event.version !== events.length + 1) {
            throw new Error('Concurrency conflict detected');
        }
        
        events.push(event);
    }
    
    async load(aggregateId: string): Promise<DomainEvent[]> {
        return this.events.get(aggregateId) || [];
    }
}

// Usage
const eventStore = new EventStore();

// Save events
await eventStore.save('order-123', new OrderCreatedEvent(...));
await eventStore.save('order-123', new PaymentProcessedEvent(...));
await eventStore.save('order-123', new OrderShippedEvent(...));

// Load and replay
const events = await eventStore.load('order-123');
const order = Order.fromEvents(events);

console.log(order.status); // SHIPPED
```

## 2. CQRS (Command Query Responsibility Segregation)

Separar modelos de **escritura** (Commands) y **lectura** (Queries).

```
┌─────────────────────────────────────────────┐
│              CQRS Architecture              │
└─────────────────────────────────────────────┘

Commands (Write):                 Queries (Read):
┌─────────────┐                  ┌──────────────┐
│ CreateOrder │                  │ GetOrder     │
│ ShipOrder   │                  │ ListOrders   │
└──────┬──────┘                  └──────▲───────┘
       │                                 │
       ▼                                 │
┌─────────────┐                         │
│ Event Store │───────Events───────┐    │
│ (Write DB)  │                    │    │
└─────────────┘                    ▼    │
                              ┌──────────┴──────┐
                              │   Projection    │
                              │   (Read Model)  │
                              └─────────────────┘
                                      │
                              ┌───────▼─────────┐
                              │   Read DB       │
                              │ (Denormalized)  │
                              └─────────────────┘
```

### 2.1 Write Model (Command Side)

```java
// Command
public record CreateOrderCommand(
    String customerId,
    List<OrderItem> items
) {}

// Command Handler
@Service
public class OrderCommandHandler {
    
    private final EventStore eventStore;
    private final EventBus eventBus;
    
    public String handle(CreateOrderCommand command) {
        // 1. Validate
        if (command.items().isEmpty()) {
            throw new IllegalArgumentException("Order must have items");
        }
        
        // 2. Create aggregate
        String orderId = UUID.randomUUID().toString();
        Order order = new Order(orderId);
        
        // 3. Execute business logic (generates events)
        OrderCreatedEvent event = order.create(
            command.customerId(),
            command.items()
        );
        
        // 4. Save events
        eventStore.save(orderId, event);
        
        // 5. Publish events
        eventBus.publish(event);
        
        return orderId;
    }
}

// Aggregate (only business logic, no queries)
public class Order {
    private String id;
    private String customerId;
    private List<OrderItem> items;
    private OrderStatus status;
    private int version;
    
    public OrderCreatedEvent create(String customerId, List<OrderItem> items) {
        // Business rules
        if (items.isEmpty()) {
            throw new DomainException("Cannot create order without items");
        }
        
        // Generate event
        OrderCreatedEvent event = new OrderCreatedEvent(
            this.id,
            customerId,
            items,
            this.version + 1,
            Instant.now()
        );
        
        // Apply to self
        apply(event);
        
        return event;
    }
    
    private void apply(OrderCreatedEvent event) {
        this.customerId = event.customerId();
        this.items = event.items();
        this.status = OrderStatus.PENDING;
        this.version = event.version();
    }
}
```

### 2.2 Read Model (Query Side)

```python
# Read Model (Denormalized for queries)
@dataclass
class OrderReadModel:
    order_id: str
    customer_id: str
    customer_name: str  # Denormalized
    customer_email: str  # Denormalized
    items: List[OrderItemView]
    total: Decimal
    status: str
    created_at: datetime
    updated_at: datetime

@dataclass
class OrderItemView:
    product_id: str
    product_name: str  # Denormalized
    product_image_url: str  # Denormalized
    quantity: int
    price: Decimal
    subtotal: Decimal

# Projection (builds read model from events)
class OrderProjection:
    def __init__(self, db: Database, customer_service: CustomerService):
        self.db = db
        self.customer_service = customer_service
    
    async def project(self, event: DomainEvent):
        if isinstance(event, OrderCreatedEvent):
            await self.handle_order_created(event)
        elif isinstance(event, PaymentProcessedEvent):
            await self.handle_payment_processed(event)
        elif isinstance(event, OrderShippedEvent):
            await self.handle_order_shipped(event)
    
    async def handle_order_created(self, event: OrderCreatedEvent):
        # Fetch customer data (denormalize)
        customer = await self.customer_service.get_customer(event.customer_id)
        
        # Build denormalized view
        order_view = OrderReadModel(
            order_id=event.order_id,
            customer_id=event.customer_id,
            customer_name=customer.name,
            customer_email=customer.email,
            items=[
                OrderItemView(
                    product_id=item.product_id,
                    product_name=item.product_name,  # From event
                    product_image_url=item.image_url,
                    quantity=item.quantity,
                    price=item.price,
                    subtotal=item.price * item.quantity
                )
                for item in event.items
            ],
            total=sum(item.price * item.quantity for item in event.items),
            status='PENDING',
            created_at=event.timestamp,
            updated_at=event.timestamp
        )
        
        # Save to read database
        await self.db.execute(
            """
            INSERT INTO order_views (
                order_id, customer_id, customer_name, customer_email,
                items_json, total, status, created_at, updated_at
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
            """,
            (
                order_view.order_id,
                order_view.customer_id,
                order_view.customer_name,
                order_view.customer_email,
                json.dumps([asdict(item) for item in order_view.items]),
                order_view.total,
                order_view.status,
                order_view.created_at,
                order_view.updated_at
            )
        )
    
    async def handle_order_shipped(self, event: OrderShippedEvent):
        # Update read model
        await self.db.execute(
            """
            UPDATE order_views 
            SET status = 'SHIPPED', updated_at = ?
            WHERE order_id = ?
            """,
            (event.timestamp, event.order_id)
        )

# Query Service (reads from read model)
class OrderQueryService:
    def __init__(self, db: Database):
        self.db = db
    
    async def get_order(self, order_id: str) -> OrderReadModel:
        row = await self.db.query_one(
            "SELECT * FROM order_views WHERE order_id = ?",
            (order_id,)
        )
        
        return OrderReadModel(
            order_id=row['order_id'],
            customer_id=row['customer_id'],
            customer_name=row['customer_name'],
            customer_email=row['customer_email'],
            items=json.loads(row['items_json']),
            total=row['total'],
            status=row['status'],
            created_at=row['created_at'],
            updated_at=row['updated_at']
        )
    
    async def list_customer_orders(
        self, 
        customer_id: str,
        page: int = 1,
        page_size: int = 20
    ) -> List[OrderReadModel]:
        # Optimized query on denormalized table
        rows = await self.db.query(
            """
            SELECT * FROM order_views 
            WHERE customer_id = ?
            ORDER BY created_at DESC
            LIMIT ? OFFSET ?
            """,
            (customer_id, page_size, (page - 1) * page_size)
        )
        
        return [self.row_to_model(row) for row in rows]
```

## 3. SOLID en Event Sourcing + CQRS

### 3.1 Single Responsibility Principle

```kotlin
// ✅ Aggregate: Solo lógica de negocio
class Order(val id: String) {
    private var status: OrderStatus = OrderStatus.PENDING
    private var items: List<OrderItem> = emptyList()
    private var version: Int = 0
    
    // Only business logic
    fun create(customerId: String, items: List<OrderItem>): OrderCreatedEvent {
        require(items.isNotEmpty()) { "Order must have items" }
        return OrderCreatedEvent(id, customerId, items, version + 1)
    }
    
    fun ship(trackingNumber: String): OrderShippedEvent {
        require(status == OrderStatus.PAID) { "Only paid orders can ship" }
        return OrderShippedEvent(id, trackingNumber, version + 1)
    }
    
    // Apply events (state changes)
    fun apply(event: DomainEvent) {
        when (event) {
            is OrderCreatedEvent -> {
                items = event.items
                status = OrderStatus.PENDING
            }
            is OrderShippedEvent -> {
                status = OrderStatus.SHIPPED
            }
        }
        version = event.version
    }
}

// ✅ EventStore: Solo persistencia de eventos
class EventStore(private val db: Database) {
    suspend fun save(aggregateId: String, event: DomainEvent) {
        db.execute(
            """
            INSERT INTO events (aggregate_id, event_type, event_data, version)
            VALUES (?, ?, ?, ?)
            """,
            aggregateId, event::class.simpleName, toJson(event), event.version
        )
    }
    
    suspend fun load(aggregateId: String): List<DomainEvent> {
        return db.query(
            "SELECT * FROM events WHERE aggregate_id = ? ORDER BY version",
            aggregateId
        ).map { fromJson(it) }
    }
}

// ✅ Projection: Solo construcción de read model
class OrderProjection(private val readDb: Database) {
    suspend fun project(event: DomainEvent) {
        when (event) {
            is OrderCreatedEvent -> createReadModel(event)
            is OrderShippedEvent -> updateStatus(event)
        }
    }
}
```

### 3.2 Open/Closed Principle

```csharp
// Extensible sin modificar código existente
public interface IProjection
{
    Task Project(DomainEvent @event);
}

// Original projection
public class OrderProjection : IProjection
{
    public async Task Project(DomainEvent @event)
    {
        switch (@event)
        {
            case OrderCreatedEvent e:
                await HandleOrderCreated(e);
                break;
            case OrderShippedEvent e:
                await HandleOrderShipped(e);
                break;
        }
    }
}

// Add new projection without modifying existing code
public class AnalyticsProjection : IProjection
{
    public async Task Project(DomainEvent @event)
    {
        switch (@event)
        {
            case OrderCreatedEvent e:
                await RecordOrderMetric(e);
                break;
            case PaymentProcessedEvent e:
                await RecordRevenueMetric(e);
                break;
        }
    }
}

// Register both projections
var projections = new List<IProjection>
{
    new OrderProjection(readDb),
    new AnalyticsProjection(analyticsDb)  // Added without changes
};

foreach (var projection in projections)
{
    await projection.Project(@event);
}
```

## 4. Snapshots (Optimización)

Para aggregates con miles de eventos, usar snapshots para performance.

```typescript
interface Snapshot {
    aggregateId: string;
    version: number;
    state: any;
    timestamp: Date;
}

class EventStore {
    private readonly SNAPSHOT_INTERVAL = 100;
    
    async loadAggregate<T>(
        aggregateId: string,
        factory: (events: DomainEvent[]) => T
    ): Promise<T> {
        // 1. Load latest snapshot
        const snapshot = await this.loadSnapshot(aggregateId);
        
        let events: DomainEvent[];
        let aggregate: T;
        
        if (snapshot) {
            // Load only events after snapshot
            events = await this.loadEventsSince(aggregateId, snapshot.version);
            
            // Restore from snapshot
            aggregate = this.hydrateFromSnapshot(snapshot, factory);
        } else {
            // Load all events
            events = await this.loadEvents(aggregateId);
            aggregate = factory([]);
        }
        
        // Replay remaining events
        events.forEach(event => (aggregate as any).apply(event));
        
        return aggregate;
    }
    
    async save(aggregateId: string, event: DomainEvent): Promise<void> {
        await this.saveEvent(aggregateId, event);
        
        // Create snapshot every N events
        if (event.version % this.SNAPSHOT_INTERVAL === 0) {
            const aggregate = await this.loadAggregate(aggregateId, Order.fromEvents);
            await this.saveSnapshot(aggregateId, event.version, aggregate);
        }
    }
}
```

## Resumen

### Event Sourcing Benefits

| Benefit | Description |
|---------|-------------|
| **Audit Trail** | Complete history automatically |
| **Time Travel** | Rebuild state at any point in time |
| **Debuggability** | Replay events to reproduce bugs |
| **Event Replay** | Fix bugs by replaying with corrected code |
| **Business Insights** | Analyze event patterns |

### CQRS Benefits

| Benefit | Description |
|---------|-------------|
| **Scalability** | Scale reads and writes independently |
| **Optimization** | Optimize each side differently |
| **Flexibility** | Different models for different use cases |
| **Performance** | Denormalized read models are fast |

### Trade-offs

❌ **Challenges**:
- Increased complexity
- Eventual consistency
- Learning curve
- Event schema evolution

✅ **When to Use**:
- Complex domains
- Audit requirements
- Event-driven architecture
- High read/write ratio difference
