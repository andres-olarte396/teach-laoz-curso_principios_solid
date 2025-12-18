# CQRS Pattern Avanzado

## Objetivos
- Dominar CQRS (Command Query Responsibility Segregation)
- Implementar read models optimizados
- Escalar lecturas y escrituras independientemente
- Aplicar SOLID a arquitecturas CQRS

## 1. ¿Qué es CQRS?

CQRS separa el modelo de **escritura** (Commands) del modelo de **lectura** (Queries) en dos modelos distintos.

```
Traditional Architecture:
┌──────────────────────────────────┐
│     Single Model (CRUD)          │
│                                  │
│  Create  ┐                       │
│  Read    ├─→ Same Model          │
│  Update  │                       │
│  Delete  ┘                       │
└──────────────────────────────────┘
   ❌ Un modelo para todo

CQRS Architecture:
┌─────────────────┐    ┌────────────────┐
│  Command Model  │    │  Query Model   │
│  (Write)        │    │  (Read)        │
│                 │    │                │
│  - Validations  │    │  - Optimized   │
│  - Business     │    │  - Denormalized│
│  - Events       │    │  - Fast reads  │
└─────────────────┘    └────────────────┘
   ✅ Modelos especializados
```

### Motivación: Requisitos Diferentes

```typescript
// Write Side: Necesita validaciones estrictas
class CreateOrderCommandHandler {
    async handle(command: CreateOrderCommand): Promise<void> {
        // 1. Validate customer exists
        const customer = await this.customerRepo.findById(command.customerId);
        if (!customer) throw new Error('Customer not found');
        
        // 2. Validate stock availability
        for (const item of command.items) {
            const product = await this.productRepo.findById(item.productId);
            if (product.stock < item.quantity) {
                throw new Error(`Insufficient stock for ${product.name}`);
            }
        }
        
        // 3. Apply business rules
        if (command.items.length > 100) {
            throw new Error('Maximum 100 items per order');
        }
        
        // 4. Generate event
        const event = new OrderCreatedEvent(...);
        await this.eventStore.save(event);
    }
}

// Read Side: Necesita queries rápidas con joins
class OrderQueryService {
    async getOrderWithDetails(orderId: string): Promise<OrderDetailView> {
        // Query denormalizada (SIN joins, todo pre-calculado)
        return await this.readDb.queryOne(`
            SELECT 
                o.order_id,
                o.customer_name,        -- Denormalizado
                o.customer_email,       -- Denormalizado
                o.items_json,           -- Pre-serializado
                o.total_amount,         -- Pre-calculado
                o.status,
                o.created_at
            FROM order_views o
            WHERE o.order_id = ?
        `, [orderId]);
    }
    
    // Query compleja optimizada
    async getCustomerOrderHistory(
        customerId: string,
        filters: OrderFilters
    ): Promise<OrderSummary[]> {
        // Read model tiene índices específicos para esto
        return await this.readDb.query(`
            SELECT * FROM order_summaries
            WHERE customer_id = ?
              AND status IN (?)
              AND created_at BETWEEN ? AND ?
            ORDER BY created_at DESC
            LIMIT ? OFFSET ?
        `, [
            customerId,
            filters.statuses,
            filters.startDate,
            filters.endDate,
            filters.pageSize,
            filters.offset
        ]);
    }
}
```

## 2. Implementación CQRS Completa

### 2.1 Command Side (Write Model)

```java
// Command (inmutable)
public record CreateProductCommand(
    String name,
    String category,
    BigDecimal price,
    int initialStock
) {}

public record UpdateStockCommand(
    String productId,
    int quantity,
    StockOperation operation  // ADD or REMOVE
) {}

// Domain Model (solo lógica de negocio)
public class Product {
    private String id;
    private String name;
    private int stock;
    private BigDecimal price;
    private int version;
    
    // Business logic
    public ProductCreatedEvent create(CreateProductCommand command) {
        // Validate
        if (command.price().compareTo(BigDecimal.ZERO) <= 0) {
            throw new DomainException("Price must be positive");
        }
        
        if (command.initialStock() < 0) {
            throw new DomainException("Stock cannot be negative");
        }
        
        // Generate event
        return new ProductCreatedEvent(
            UUID.randomUUID().toString(),
            command.name(),
            command.category(),
            command.price(),
            command.initialStock(),
            this.version + 1,
            Instant.now()
        );
    }
    
    public StockUpdatedEvent updateStock(UpdateStockCommand command) {
        int delta = command.operation() == StockOperation.ADD
            ? command.quantity()
            : -command.quantity();
        
        int newStock = this.stock + delta;
        
        if (newStock < 0) {
            throw new DomainException("Insufficient stock");
        }
        
        return new StockUpdatedEvent(
            this.id,
            this.stock,
            newStock,
            delta,
            this.version + 1,
            Instant.now()
        );
    }
    
    // Event application (state changes)
    void apply(DomainEvent event) {
        if (event instanceof ProductCreatedEvent e) {
            this.id = e.productId();
            this.name = e.name();
            this.stock = e.initialStock();
            this.price = e.price();
        } else if (event instanceof StockUpdatedEvent e) {
            this.stock = e.newStock();
        }
        this.version = event.version();
    }
}

// Command Handler
@Service
public class ProductCommandHandler {
    
    private final EventStore eventStore;
    private final EventBus eventBus;
    
    @Transactional
    public String handle(CreateProductCommand command) {
        // Create aggregate
        Product product = new Product();
        ProductCreatedEvent event = product.create(command);
        
        // Persist event
        eventStore.save(event.productId(), event);
        
        // Publish for projections
        eventBus.publish(event);
        
        return event.productId();
    }
    
    @Transactional
    public void handle(UpdateStockCommand command) {
        // Load from event store
        List<DomainEvent> events = eventStore.load(command.productId());
        Product product = Product.fromEvents(events);
        
        // Execute command
        StockUpdatedEvent event = product.updateStock(command);
        
        // Save and publish
        eventStore.save(command.productId(), event);
        eventBus.publish(event);
    }
}
```

### 2.2 Query Side (Read Model)

```typescript
// Read Models (denormalizados para queries específicas)

// Vista principal de productos
interface ProductView {
    productId: string;
    name: string;
    category: string;
    stock: number;
    price: number;
    stockStatus: 'OUT_OF_STOCK' | 'LOW' | 'NORMAL' | 'HIGH';
    lastUpdated: Date;
}

// Vista para analytics
interface ProductSalesView {
    productId: string;
    name: string;
    totalSales: number;
    totalRevenue: number;
    averageOrderValue: number;
    lastSaleDate: Date;
}

// Vista para inventory management
interface InventoryView {
    productId: string;
    name: string;
    currentStock: number;
    reservedStock: number;      // En órdenes pendientes
    availableStock: number;     // current - reserved
    reorderPoint: number;
    needsReorder: boolean;
}

// Projections (construyen read models desde eventos)
class ProductViewProjection {
    constructor(
        private readonly readDb: Database
    ) {}
    
    async on(event: ProductCreatedEvent): Promise<void> {
        await this.readDb.insert('product_views', {
            product_id: event.productId,
            name: event.name,
            category: event.category,
            stock: event.initialStock,
            price: event.price,
            stock_status: this.calculateStockStatus(event.initialStock),
            last_updated: event.timestamp
        });
    }
    
    async on(event: StockUpdatedEvent): Promise<void> {
        await this.readDb.update(
            'product_views',
            { product_id: event.productId },
            {
                stock: event.newStock,
                stock_status: this.calculateStockStatus(event.newStock),
                last_updated: event.timestamp
            }
        );
    }
    
    private calculateStockStatus(stock: number): string {
        if (stock === 0) return 'OUT_OF_STOCK';
        if (stock < 10) return 'LOW';
        if (stock < 50) return 'NORMAL';
        return 'HIGH';
    }
}

class ProductSalesProjection {
    async on(event: OrderCreatedEvent): Promise<void> {
        for (const item of event.items) {
            // Upsert (insert or update)
            await this.readDb.execute(`
                INSERT INTO product_sales_views (
                    product_id, name, total_sales, total_revenue, last_sale_date
                ) VALUES (?, ?, 1, ?, ?)
                ON CONFLICT (product_id) DO UPDATE SET
                    total_sales = total_sales + 1,
                    total_revenue = total_revenue + ?,
                    last_sale_date = ?
            `, [
                item.productId,
                item.productName,
                item.price * item.quantity,
                event.timestamp,
                item.price * item.quantity,
                event.timestamp
            ]);
        }
    }
}

// Query Services (leen desde read models)
class ProductQueryService {
    constructor(private readonly readDb: Database) {}
    
    async getProduct(productId: string): Promise<ProductView> {
        return await this.readDb.queryOne(
            'SELECT * FROM product_views WHERE product_id = ?',
            [productId]
        );
    }
    
    async getLowStockProducts(): Promise<ProductView[]> {
        return await this.readDb.query(`
            SELECT * FROM product_views 
            WHERE stock_status IN ('LOW', 'OUT_OF_STOCK')
            ORDER BY stock ASC
        `);
    }
    
    async getProductsByCategory(
        category: string,
        pagination: Pagination
    ): Promise<PagedResult<ProductView>> {
        const products = await this.readDb.query(`
            SELECT * FROM product_views 
            WHERE category = ?
            ORDER BY name
            LIMIT ? OFFSET ?
        `, [category, pagination.pageSize, pagination.offset]);
        
        const total = await this.readDb.queryScalar(
            'SELECT COUNT(*) FROM product_views WHERE category = ?',
            [category]
        );
        
        return {
            items: products,
            total,
            page: pagination.page,
            pageSize: pagination.pageSize
        };
    }
    
    async getTopSellingProducts(limit: number): Promise<ProductSalesView[]> {
        return await this.readDb.query(`
            SELECT * FROM product_sales_views
            ORDER BY total_sales DESC
            LIMIT ?
        `, [limit]);
    }
}
```

## 3. SOLID en CQRS

### 3.1 Single Responsibility Principle

```python
# ✅ Cada componente una responsabilidad

# Command Handler: Solo orquestación
class CreateOrderCommandHandler:
    def __init__(self, event_store: EventStore, event_bus: EventBus):
        self.event_store = event_store
        self.event_bus = event_bus
    
    async def handle(self, command: CreateOrderCommand) -> str:
        # Solo orquestación, no lógica de negocio
        order = Order()
        event = order.create(command)
        
        await self.event_store.save(event.order_id, event)
        await self.event_bus.publish(event)
        
        return event.order_id

# Aggregate: Solo lógica de negocio
class Order:
    def create(self, command: CreateOrderCommand) -> OrderCreatedEvent:
        # Solo validaciones y reglas de negocio
        if not command.items:
            raise ValueError("Order must have items")
        
        if len(command.items) > 100:
            raise ValueError("Maximum 100 items per order")
        
        total = sum(item.price * item.quantity for item in command.items)
        
        return OrderCreatedEvent(
            order_id=str(uuid4()),
            customer_id=command.customer_id,
            items=command.items,
            total=total,
            version=1,
            timestamp=datetime.now()
        )

# Projection: Solo construcción de read model
class OrderViewProjection:
    def __init__(self, read_db: Database, customer_service: CustomerService):
        self.read_db = read_db
        self.customer_service = customer_service
    
    async def project(self, event: OrderCreatedEvent):
        # Solo construcción de vista denormalizada
        customer = await self.customer_service.get(event.customer_id)
        
        await self.read_db.insert('order_views', {
            'order_id': event.order_id,
            'customer_name': customer.name,  # Denormalizado
            'customer_email': customer.email,
            'item_count': len(event.items),
            'total': event.total,
            'status': 'PENDING',
            'created_at': event.timestamp
        })

# Query Service: Solo queries
class OrderQueryService:
    def __init__(self, read_db: Database):
        self.read_db = read_db
    
    async def get_order(self, order_id: str) -> OrderView:
        # Solo lectura desde read model
        row = await self.read_db.query_one(
            "SELECT * FROM order_views WHERE order_id = ?",
            order_id
        )
        return OrderView(**row)
```

### 3.2 Open/Closed Principle

```csharp
// Extensible sin modificar código existente

// Base projection
public interface IProjection
{
    Task Project(DomainEvent @event);
}

// Original projection
public class OrderViewProjection : IProjection
{
    public async Task Project(DomainEvent @event)
    {
        if (@event is OrderCreatedEvent e)
            await HandleOrderCreated(e);
    }
}

// Add new projection WITHOUT modifying existing code
public class OrderAnalyticsProjection : IProjection
{
    public async Task Project(DomainEvent @event)
    {
        if (@event is OrderCreatedEvent e)
            await TrackOrderMetrics(e);
    }
}

public class CustomerLoyaltyProjection : IProjection
{
    public async Task Project(DomainEvent @event)
    {
        if (@event is OrderCreatedEvent e)
            await UpdateLoyaltyPoints(e);
    }
}

// Projection Manager (open for extension)
public class ProjectionManager
{
    private readonly List<IProjection> _projections;
    
    public ProjectionManager(IEnumerable<IProjection> projections)
    {
        _projections = projections.ToList();
    }
    
    public async Task ProjectAll(DomainEvent @event)
    {
        // Run all projections in parallel
        await Task.WhenAll(
            _projections.Select(p => p.Project(@event))
        );
    }
}

// Register projections (extension point)
services.AddSingleton<IProjection, OrderViewProjection>();
services.AddSingleton<IProjection, OrderAnalyticsProjection>();
services.AddSingleton<IProjection, CustomerLoyaltyProjection>();  // Added!
```

### 3.3 Dependency Inversion Principle

```kotlin
// Abstractions (interfaces)
interface CommandHandler<C, R> {
    suspend fun handle(command: C): R
}

interface QueryHandler<Q, R> {
    suspend fun handle(query: Q): R
}

interface EventStore {
    suspend fun save(aggregateId: String, event: DomainEvent)
    suspend fun load(aggregateId: String): List<DomainEvent>
}

interface ReadModelStore {
    suspend fun save(view: Any)
    suspend fun <T> query(sql: String, params: List<Any>): List<T>
}

// Implementations depend on abstractions
class CreateOrderCommandHandler(
    private val eventStore: EventStore,  // Abstraction
    private val eventBus: EventBus       // Abstraction
) : CommandHandler<CreateOrderCommand, String> {
    
    override suspend fun handle(command: CreateOrderCommand): String {
        val order = Order()
        val event = order.create(command)
        
        eventStore.save(event.orderId, event)
        eventBus.publish(event)
        
        return event.orderId
    }
}

class GetOrderQueryHandler(
    private val readStore: ReadModelStore  // Abstraction
) : QueryHandler<GetOrderQuery, OrderView> {
    
    override suspend fun handle(query: GetOrderQuery): OrderView {
        val results = readStore.query<OrderView>(
            "SELECT * FROM order_views WHERE order_id = ?",
            listOf(query.orderId)
        )
        
        return results.firstOrNull()
            ?: throw OrderNotFoundException(query.orderId)
    }
}

// Easy to swap implementations
class InMemoryEventStore : EventStore { ... }
class PostgresEventStore : EventStore { ... }
class EventStoreDBEventStore : EventStore { ... }

class InMemoryReadStore : ReadModelStore { ... }
class MongoDBReadStore : ReadModelStore { ... }
class ElasticsearchReadStore : ReadModelStore { ... }
```

## 4. Escalabilidad y Performance

### 4.1 Diferentes Bases de Datos

```typescript
// Write DB: Optimizado para transacciones ACID
const writeDb = new PostgresDatabase({
    host: 'write-db.example.com',
    replication: 'master',
    consistency: 'strong'
});

// Read DB: Optimizado para queries complejas
const readDb = new MongoDBDatabase({
    host: 'read-db.example.com',
    replication: 'replica-set',
    consistency: 'eventual',
    indices: [
        { fields: ['customer_id', 'created_at'] },
        { fields: ['status', 'total'] },
        { fields: ['category', 'price'] }
    ]
});

// Analytics DB: Optimizado para agregaciones
const analyticsDb = new ClickHouseDatabase({
    host: 'analytics-db.example.com',
    compression: true,
    columnOriented: true
});
```

### 4.2 Caching Strategies

```java
@Service
public class CachedProductQueryService {
    
    private final ReadModelStore readStore;
    private final Cache cache;
    
    public ProductView getProduct(String productId) {
        // Try cache first
        return cache.get("product:" + productId, () -> {
            // Cache miss, query read model
            return readStore.queryOne(
                "SELECT * FROM product_views WHERE product_id = ?",
                productId
            );
        }, Duration.ofMinutes(5));
    }
    
    @EventListener
    public void on(StockUpdatedEvent event) {
        // Invalidate cache when stock changes
        cache.evict("product:" + event.productId());
    }
}
```

## Resumen

### CQRS Benefits

| Benefit | Description |
|---------|-------------|
| **Performance** | Optimize reads/writes independently |
| **Scalability** | Scale read/write databases separately |
| **Flexibility** | Multiple read models from same events |
| **Simplicity** | Each side focused on its purpose |

### CQRS Trade-offs

| Challenge | Mitigation |
|-----------|------------|
| **Complexity** | Start simple, add complexity as needed |
| **Eventual Consistency** | Acceptable for most business cases |
| **Data Duplication** | Storage is cheap, performance is expensive |
| **Synchronization** | Projections must be idempotent and reliable |

### When to Use CQRS

✅ **Use When**:
- Complex read requirements (reports, dashboards)
- High read/write ratio difference (90% reads, 10% writes)
- Need to scale reads and writes independently
- Multiple views of same data

❌ **Avoid When**:
- Simple CRUD applications
- Strong consistency required everywhere
- Small team without event experience
- Low traffic applications
