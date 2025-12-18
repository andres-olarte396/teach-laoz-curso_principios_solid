# Ejercicios: Event Sourcing y CQRS

## ⭐ Ejercicio 1: Implementar Event Replay

Dado un aggregate con estos eventos:

```
1. AccountCreatedEvent (balance: 0)
2. MoneyDepositedEvent (amount: 100)
3. MoneyWithdrawnEvent (amount: 30)
4. MoneyDepositedEvent (amount: 50)
5. MoneyWithdrawnEvent (amount: 20)
```

**Tareas**:
1. Implementar clase `BankAccount` que se reconstruya desde eventos
2. Método `apply(event)` para cada tipo de evento
3. Calcular balance final

<details>
<summary>💡 Solución</summary>

```typescript
// Events
interface DomainEvent {
    version: number;
    timestamp: Date;
}

class AccountCreatedEvent implements DomainEvent {
    constructor(
        public readonly accountId: string,
        public readonly ownerId: string,
        public readonly version: number,
        public readonly timestamp: Date
    ) {}
}

class MoneyDepositedEvent implements DomainEvent {
    constructor(
        public readonly accountId: string,
        public readonly amount: number,
        public readonly version: number,
        public readonly timestamp: Date
    ) {}
}

class MoneyWithdrawnEvent implements DomainEvent {
    constructor(
        public readonly accountId: string,
        public readonly amount: number,
        public readonly version: number,
        public readonly timestamp: Date
    ) {}
}

// Aggregate
class BankAccount {
    private id: string = '';
    private ownerId: string = '';
    private balance: number = 0;
    private version: number = 0;
    private transactions: Transaction[] = [];
    
    // Reconstruct from events
    static fromEvents(events: DomainEvent[]): BankAccount {
        const account = new BankAccount();
        events.forEach(event => account.apply(event));
        return account;
    }
    
    // Apply event to update state
    apply(event: DomainEvent): void {
        if (event instanceof AccountCreatedEvent) {
            this.id = event.accountId;
            this.ownerId = event.ownerId;
            this.balance = 0;
        } else if (event instanceof MoneyDepositedEvent) {
            this.balance += event.amount;
            this.transactions.push({
                type: 'DEPOSIT',
                amount: event.amount,
                timestamp: event.timestamp
            });
        } else if (event instanceof MoneyWithdrawnEvent) {
            this.balance -= event.amount;
            this.transactions.push({
                type: 'WITHDRAWAL',
                amount: event.amount,
                timestamp: event.timestamp
            });
        }
        
        this.version = event.version;
    }
    
    getBalance(): number {
        return this.balance;
    }
    
    getTransactionHistory(): Transaction[] {
        return [...this.transactions];  // Defensive copy
    }
}

// Usage
const events: DomainEvent[] = [
    new AccountCreatedEvent('acc-123', 'user-456', 1, new Date('2024-01-01')),
    new MoneyDepositedEvent('acc-123', 100, 2, new Date('2024-01-02')),
    new MoneyWithdrawnEvent('acc-123', 30, 3, new Date('2024-01-03')),
    new MoneyDepositedEvent('acc-123', 50, 4, new Date('2024-01-04')),
    new MoneyWithdrawnEvent('acc-123', 20, 5, new Date('2024-01-05'))
];

const account = BankAccount.fromEvents(events);

console.log(account.getBalance());  // 100
console.log(account.getTransactionHistory().length);  // 4
```

**Test**:
```typescript
describe('BankAccount Event Replay', () => {
    it('should reconstruct correct state from events', () => {
        const events = [
            new AccountCreatedEvent('acc-1', 'user-1', 1, new Date()),
            new MoneyDepositedEvent('acc-1', 100, 2, new Date()),
            new MoneyWithdrawnEvent('acc-1', 30, 3, new Date()),
        ];
        
        const account = BankAccount.fromEvents(events);
        
        expect(account.getBalance()).toBe(70);
        expect(account.getTransactionHistory()).toHaveLength(2);
    });
    
    it('should handle empty event stream', () => {
        const account = BankAccount.fromEvents([]);
        expect(account.getBalance()).toBe(0);
    });
});
```

</details>

---

## ⭐⭐ Ejercicio 2: Implementar CQRS básico

Crea sistema de gestión de inventario con CQRS.

**Write Side**:
- Command: `AddProductCommand`, `UpdateStockCommand`
- Events: `ProductAddedEvent`, `StockUpdatedEvent`

**Read Side**:
- Query: `GetProductsQuery`, `GetLowStockProductsQuery`
- Read Model: Denormalized product view

<details>
<summary>💡 Solución</summary>

```java
// ========== WRITE SIDE ==========

// Commands
public record AddProductCommand(
    String name,
    String category,
    int initialStock,
    BigDecimal price
) {}

public record UpdateStockCommand(
    String productId,
    int quantity,
    StockOperation operation  // ADD or REMOVE
) {}

public enum StockOperation {
    ADD, REMOVE
}

// Events
public record ProductAddedEvent(
    String productId,
    String name,
    String category,
    int stock,
    BigDecimal price,
    int version,
    Instant timestamp
) implements DomainEvent {}

public record StockUpdatedEvent(
    String productId,
    int previousStock,
    int newStock,
    int delta,
    int version,
    Instant timestamp
) implements DomainEvent {}

// Aggregate (Write Model)
public class Product {
    private String id;
    private String name;
    private String category;
    private int stock;
    private BigDecimal price;
    private int version;
    
    public static Product fromEvents(List<DomainEvent> events) {
        Product product = new Product();
        events.forEach(product::apply);
        return product;
    }
    
    public ProductAddedEvent add(AddProductCommand command) {
        if (command.initialStock() < 0) {
            throw new IllegalArgumentException("Stock cannot be negative");
        }
        
        String productId = UUID.randomUUID().toString();
        
        ProductAddedEvent event = new ProductAddedEvent(
            productId,
            command.name(),
            command.category(),
            command.initialStock(),
            command.price(),
            this.version + 1,
            Instant.now()
        );
        
        apply(event);
        return event;
    }
    
    public StockUpdatedEvent updateStock(UpdateStockCommand command) {
        int previousStock = this.stock;
        int delta = command.operation() == StockOperation.ADD
            ? command.quantity()
            : -command.quantity();
        int newStock = previousStock + delta;
        
        if (newStock < 0) {
            throw new IllegalStateException("Insufficient stock");
        }
        
        StockUpdatedEvent event = new StockUpdatedEvent(
            this.id,
            previousStock,
            newStock,
            delta,
            this.version + 1,
            Instant.now()
        );
        
        apply(event);
        return event;
    }
    
    private void apply(DomainEvent event) {
        if (event instanceof ProductAddedEvent e) {
            this.id = e.productId();
            this.name = e.name();
            this.category = e.category();
            this.stock = e.stock();
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
    
    public String handle(AddProductCommand command) {
        Product product = new Product();
        ProductAddedEvent event = product.add(command);
        
        eventStore.save(event.productId(), event);
        eventBus.publish(event);
        
        return event.productId();
    }
    
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

// ========== READ SIDE ==========

// Read Model
public record ProductView(
    String productId,
    String name,
    String category,
    int stock,
    BigDecimal price,
    String stockStatus,  // LOW, MEDIUM, HIGH
    Instant lastUpdated
) {}

// Projection (builds read model)
@Component
public class ProductProjection {
    
    private final JdbcTemplate jdbcTemplate;
    
    @EventListener
    public void on(ProductAddedEvent event) {
        jdbcTemplate.update(
            """
            INSERT INTO product_views 
            (product_id, name, category, stock, price, stock_status, last_updated)
            VALUES (?, ?, ?, ?, ?, ?, ?)
            """,
            event.productId(),
            event.name(),
            event.category(),
            event.stock(),
            event.price(),
            calculateStockStatus(event.stock()),
            event.timestamp()
        );
    }
    
    @EventListener
    public void on(StockUpdatedEvent event) {
        jdbcTemplate.update(
            """
            UPDATE product_views
            SET stock = ?, stock_status = ?, last_updated = ?
            WHERE product_id = ?
            """,
            event.newStock(),
            calculateStockStatus(event.newStock()),
            event.timestamp(),
            event.productId()
        );
    }
    
    private String calculateStockStatus(int stock) {
        if (stock < 10) return "LOW";
        if (stock < 50) return "MEDIUM";
        return "HIGH";
    }
}

// Query Service
@Service
public class ProductQueryService {
    
    private final JdbcTemplate jdbcTemplate;
    
    public ProductView getProduct(String productId) {
        return jdbcTemplate.queryForObject(
            "SELECT * FROM product_views WHERE product_id = ?",
            this::mapRow,
            productId
        );
    }
    
    public List<ProductView> getProductsByCategory(String category) {
        return jdbcTemplate.query(
            "SELECT * FROM product_views WHERE category = ? ORDER BY name",
            this::mapRow,
            category
        );
    }
    
    public List<ProductView> getLowStockProducts() {
        return jdbcTemplate.query(
            "SELECT * FROM product_views WHERE stock_status = 'LOW' ORDER BY stock",
            this::mapRow
        );
    }
    
    private ProductView mapRow(ResultSet rs, int rowNum) throws SQLException {
        return new ProductView(
            rs.getString("product_id"),
            rs.getString("name"),
            rs.getString("category"),
            rs.getInt("stock"),
            rs.getBigDecimal("price"),
            rs.getString("stock_status"),
            rs.getTimestamp("last_updated").toInstant()
        );
    }
}
```

**Test**:
```java
@SpringBootTest
class CQRSTest {
    
    @Autowired
    private ProductCommandHandler commandHandler;
    
    @Autowired
    private ProductQueryService queryService;
    
    @Test
    void shouldSeparateCommandsAndQueries() {
        // Command (write)
        AddProductCommand command = new AddProductCommand(
            "Laptop",
            "Electronics",
            5,
            new BigDecimal("999.99")
        );
        
        String productId = commandHandler.handle(command);
        
        // Wait for projection (eventual consistency)
        await().atMost(Duration.ofSeconds(5))
            .untilAsserted(() -> {
                // Query (read)
                ProductView product = queryService.getProduct(productId);
                
                assertThat(product.name()).isEqualTo("Laptop");
                assertThat(product.stock()).isEqualTo(5);
                assertThat(product.stockStatus()).isEqualTo("LOW");
            });
    }
    
    @Test
    void shouldQueryLowStockProducts() {
        // Add products with low stock
        commandHandler.handle(new AddProductCommand("Product A", "Cat1", 3, ...));
        commandHandler.handle(new AddProductCommand("Product B", "Cat1", 8, ...));
        commandHandler.handle(new AddProductCommand("Product C", "Cat1", 50, ...));
        
        await().atMost(Duration.ofSeconds(5))
            .untilAsserted(() -> {
                List<ProductView> lowStock = queryService.getLowStockProducts();
                
                assertThat(lowStock).hasSize(2);
                assertThat(lowStock).extracting("name")
                    .containsExactlyInAnyOrder("Product A", "Product B");
            });
    }
}
```

</details>

---

## ⭐⭐⭐ Ejercicio 3: Implementar Snapshots para Performance

Crea sistema de chat con snapshots para optimizar carga de conversaciones largas.

**Requisitos**:
1. Aggregate `ChatConversation` con miles de mensajes
2. Crear snapshot cada 100 mensajes
3. Cargar desde snapshot + eventos posteriores
4. Comparar performance con/sin snapshots

<details>
<summary>💡 Solución</summary>

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import List, Optional
import time

# Events
@dataclass
class ConversationStartedEvent:
    conversation_id: str
    participants: List[str]
    version: int
    timestamp: datetime

@dataclass
class MessageSentEvent:
    conversation_id: str
    sender_id: str
    message: str
    version: int
    timestamp: datetime

# Snapshot
@dataclass
class ConversationSnapshot:
    conversation_id: str
    participants: List[str]
    messages: List[dict]
    version: int
    timestamp: datetime

# Aggregate
class ChatConversation:
    def __init__(self, conversation_id: str):
        self.id = conversation_id
        self.participants: List[str] = []
        self.messages: List[dict] = []
        self.version = 0
    
    @staticmethod
    def from_events(events: List) -> 'ChatConversation':
        conversation = ChatConversation(events[0].conversation_id)
        for event in events:
            conversation.apply(event)
        return conversation
    
    @staticmethod
    def from_snapshot(snapshot: ConversationSnapshot) -> 'ChatConversation':
        conversation = ChatConversation(snapshot.conversation_id)
        conversation.participants = snapshot.participants.copy()
        conversation.messages = snapshot.messages.copy()
        conversation.version = snapshot.version
        return conversation
    
    def apply(self, event):
        if isinstance(event, ConversationStartedEvent):
            self.participants = event.participants
        elif isinstance(event, MessageSentEvent):
            self.messages.append({
                'sender': event.sender_id,
                'message': event.message,
                'timestamp': event.timestamp
            })
        
        self.version = event.version
    
    def to_snapshot(self) -> ConversationSnapshot:
        return ConversationSnapshot(
            conversation_id=self.id,
            participants=self.participants.copy(),
            messages=self.messages.copy(),
            version=self.version,
            timestamp=datetime.now()
        )

# Event Store with Snapshots
class EventStoreWithSnapshots:
    SNAPSHOT_INTERVAL = 100
    
    def __init__(self):
        self.events: dict[str, List] = {}
        self.snapshots: dict[str, ConversationSnapshot] = {}
    
    def save_event(self, aggregate_id: str, event):
        if aggregate_id not in self.events:
            self.events[aggregate_id] = []
        
        self.events[aggregate_id].append(event)
        
        # Create snapshot every N events
        if event.version % self.SNAPSHOT_INTERVAL == 0:
            conversation = self.load_aggregate(aggregate_id)
            self.save_snapshot(aggregate_id, conversation.to_snapshot())
    
    def save_snapshot(self, aggregate_id: str, snapshot: ConversationSnapshot):
        self.snapshots[aggregate_id] = snapshot
        print(f"📸 Snapshot created at version {snapshot.version}")
    
    def load_aggregate(self, aggregate_id: str) -> ChatConversation:
        snapshot = self.snapshots.get(aggregate_id)
        
        if snapshot:
            print(f"✅ Loading from snapshot (version {snapshot.version})")
            # Load from snapshot
            conversation = ChatConversation.from_snapshot(snapshot)
            
            # Load only events after snapshot
            remaining_events = [
                e for e in self.events.get(aggregate_id, [])
                if e.version > snapshot.version
            ]
            
            print(f"📚 Replaying {len(remaining_events)} events after snapshot")
            for event in remaining_events:
                conversation.apply(event)
        else:
            print(f"⚠️ No snapshot found, loading all events")
            # Load all events
            all_events = self.events.get(aggregate_id, [])
            print(f"📚 Replaying {len(all_events)} events")
            conversation = ChatConversation.from_events(all_events)
        
        return conversation

# Performance Test
def test_snapshot_performance():
    event_store = EventStoreWithSnapshots()
    conversation_id = "conv-123"
    
    # Create conversation with 500 messages
    print("Creating conversation with 500 messages...")
    
    # Initial event
    event_store.save_event(
        conversation_id,
        ConversationStartedEvent(
            conversation_id, ["user1", "user2"], 1, datetime.now()
        )
    )
    
    # Add 500 messages
    for i in range(2, 502):
        event_store.save_event(
            conversation_id,
            MessageSentEvent(
                conversation_id,
                f"user{i % 2 + 1}",
                f"Message {i}",
                i,
                datetime.now()
            )
        )
    
    print("\n=== Performance Test ===")
    
    # Test 1: Load with snapshots
    start = time.time()
    conversation = event_store.load_aggregate(conversation_id)
    duration_with_snapshot = time.time() - start
    
    print(f"\n✅ With snapshots: {duration_with_snapshot:.4f}s")
    print(f"   Messages loaded: {len(conversation.messages)}")
    print(f"   Current version: {conversation.version}")
    
    # Test 2: Load without snapshots (clear snapshots)
    event_store.snapshots.clear()
    
    start = time.time()
    conversation = event_store.load_aggregate(conversation_id)
    duration_without_snapshot = time.time() - start
    
    print(f"\n❌ Without snapshots: {duration_without_snapshot:.4f}s")
    
    # Improvement
    improvement = ((duration_without_snapshot - duration_with_snapshot) 
                   / duration_without_snapshot * 100)
    print(f"\n🚀 Performance improvement: {improvement:.1f}%")

if __name__ == "__main__":
    test_snapshot_performance()
```

**Output Example**:
```
Creating conversation with 500 messages...
📸 Snapshot created at version 100
📸 Snapshot created at version 200
📸 Snapshot created at version 300
📸 Snapshot created at version 400
📸 Snapshot created at version 500

=== Performance Test ===
✅ Loading from snapshot (version 500)
📚 Replaying 0 events after snapshot

✅ With snapshots: 0.0012s
   Messages loaded: 500
   Current version: 501

⚠️ No snapshot found, loading all events
📚 Replaying 501 events

❌ Without snapshots: 0.0234s

🚀 Performance improvement: 94.9%
```

</details>

---

## ⭐⭐⭐⭐ Ejercicio 4: Event Schema Evolution

Implementa evolución de esquema de eventos sin romper compatibilidad.

**Escenario**:
- v1: `OrderCreatedEvent` tiene solo `total: number`
- v2: Necesitas cambiar a `total: { amount: number, currency: string }`
- Sistema debe leer eventos v1 y v2

**Tareas**:
1. Implementar upcasting (v1 → v2)
2. Proyecciones deben manejar ambas versiones
3. No perder datos históricos

<details>
<summary>💡 Solución</summary>

```typescript
// ========== Version 1 ==========
interface OrderCreatedEventV1 {
    orderId: string;
    customerId: string;
    total: number;  // Simple number
    version: number;
    timestamp: Date;
    eventVersion: 1;  // Schema version
}

// ========== Version 2 ==========
interface Money {
    amount: number;
    currency: string;
}

interface OrderCreatedEventV2 {
    orderId: string;
    customerId: string;
    total: Money;  // Structured money
    version: number;
    timestamp: Date;
    eventVersion: 2;
}

// Union type
type OrderCreatedEvent = OrderCreatedEventV1 | OrderCreatedEventV2;

// Upcaster (v1 → v2)
class EventUpcaster {
    upcast(event: any): OrderCreatedEventV2 {
        if (event.eventVersion === 1) {
            // Convert v1 to v2
            return {
                orderId: event.orderId,
                customerId: event.customerId,
                total: {
                    amount: event.total,
                    currency: 'USD'  // Default assumption
                },
                version: event.version,
                timestamp: event.timestamp,
                eventVersion: 2
            };
        }
        
        // Already v2
        return event as OrderCreatedEventV2;
    }
}

// Event Store with Upcasting
class EventStore {
    private upcaster = new EventUpcaster();
    private events: Map<string, any[]> = new Map();
    
    save(aggregateId: string, event: OrderCreatedEventV2): void {
        // Always save as latest version
        if (!this.events.has(aggregateId)) {
            this.events.set(aggregateId, []);
        }
        this.events.get(aggregateId)!.push(event);
    }
    
    load(aggregateId: string): OrderCreatedEventV2[] {
        const rawEvents = this.events.get(aggregateId) || [];
        
        // Upcast all events to latest version
        return rawEvents.map(event => this.upcaster.upcast(event));
    }
}

// Aggregate handles only v2
class Order {
    private id: string = '';
    private customerId: string = '';
    private total: Money = { amount: 0, currency: 'USD' };
    
    static fromEvents(events: OrderCreatedEventV2[]): Order {
        const order = new Order();
        events.forEach(event => order.apply(event));
        return order;
    }
    
    apply(event: OrderCreatedEventV2): void {
        this.id = event.orderId;
        this.customerId = event.customerId;
        this.total = event.total;  // Always Money type
    }
}

// Test
describe('Event Schema Evolution', () => {
    it('should upcast v1 events to v2', () => {
        const eventStore = new EventStore();
        const upcaster = new EventUpcaster();
        
        // Old event (v1)
        const oldEvent: OrderCreatedEventV1 = {
            orderId: 'order-1',
            customerId: 'cust-1',
            total: 99.99,  // Simple number
            version: 1,
            timestamp: new Date(),
            eventVersion: 1
        };
        
        // Upcast
        const newEvent = upcaster.upcast(oldEvent);
        
        expect(newEvent.eventVersion).toBe(2);
        expect(newEvent.total).toEqual({
            amount: 99.99,
            currency: 'USD'
        });
    });
    
    it('should load mix of v1 and v2 events', () => {
        const eventStore = new EventStore();
        
        // Save v1 event (simulating old data)
        (eventStore as any).events.set('order-1', [
            {
                orderId: 'order-1',
                customerId: 'cust-1',
                total: 50.0,  // v1
                version: 1,
                timestamp: new Date('2023-01-01'),
                eventVersion: 1
            }
        ]);
        
        // Save v2 event (new data)
        eventStore.save('order-1', {
            orderId: 'order-1',
            customerId: 'cust-1',
            total: { amount: 30.0, currency: 'EUR' },  // v2
            version: 2,
            timestamp: new Date('2024-01-01'),
            eventVersion: 2
        });
        
        // Load - all should be v2
        const events = eventStore.load('order-1');
        
        expect(events).toHaveLength(2);
        expect(events[0].total).toEqual({ amount: 50.0, currency: 'USD' });
        expect(events[1].total).toEqual({ amount: 30.0, currency: 'EUR' });
        
        // Aggregate handles both seamlessly
        const order = Order.fromEvents(events);
        expect(order).toBeDefined();
    });
});
```

**Migration Strategy**:
```typescript
// Background job to migrate old events
class EventMigration {
    async migrateToV2(eventStore: EventStore): Promise<void> {
        console.log('Starting event migration v1 → v2...');
        
        const allAggregates = await eventStore.getAllAggregateIds();
        
        for (const aggregateId of allAggregates) {
            const events = await eventStore.loadRaw(aggregateId);
            
            const migratedEvents = events.map(event => {
                if (event.eventVersion === 1) {
                    // Upcast
                    return {
                        ...event,
                        total: {
                            amount: event.total,
                            currency: 'USD'
                        },
                        eventVersion: 2
                    };
                }
                return event;
            });
            
            await eventStore.replaceEvents(aggregateId, migratedEvents);
        }
        
        console.log('Migration complete!');
    }
}
```

</details>

---

## Proyecto Final: Sistema Bancario con Event Sourcing + CQRS

Implementa sistema bancario completo:

**Write Side**:
- Commands: OpenAccount, Deposit, Withdraw, Transfer
- Events: AccountOpened, MoneyDeposited, MoneyWithdrawn, MoneyTransferred
- Business rules: No negative balance, daily withdrawal limit

**Read Side**:
- Queries: GetBalance, GetTransactionHistory, GetMonthlyStatement
- Projections: AccountView, TransactionView, AnalyticsView

**Features**:
- Snapshots cada 100 transacciones
- Event upcasting para schema evolution
- Audit trail completo
- Performance testing (con/sin snapshots)

**Rúbrica**:
- ⭐ Event sourcing básico
- ⭐⭐ CQRS con projections
- ⭐⭐⭐ Snapshots + upcasting
- ⭐⭐⭐⭐ Performance optimization + monitoring + compliance reporting

## Recursos

- **Event Sourcing**: https://martinfowler.com/eaaDev/EventSourcing.html
- **CQRS**: https://martinfowler.com/bliki/CQRS.html
- **EventStoreDB**: https://www.eventstore.com/
