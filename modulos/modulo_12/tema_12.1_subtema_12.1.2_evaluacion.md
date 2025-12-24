# Evaluación: Event Sourcing y CQRS

## Parte 1. Preguntas Conceptuales (25 puntos)

### Pregunta 1 (8 puntos)
Explica cómo Event Sourcing difiere de persistencia tradicional. ¿Cuáles son las ventajas y desventajas?

**Criterios de evaluación**:
- Diferencia clave: guardar eventos vs estado actual
- Ventajas: audit trail, time travel, debugging
- Desventajas: complejidad, eventual consistency
- Ejemplos concretos

---

### Pregunta 2 (8 puntos)
¿Qué problema resuelve CQRS? ¿Por qué separar modelos de lectura y escritura?

**Criterios de evaluación**:
- Problema: requisitos diferentes para lectura/escritura
- Beneficio: optimización independiente
- Escalabilidad: escalar reads vs writes por separado
- Trade-off: complejidad, eventual consistency

---

### Pregunta 3 (9 puntos)
Explica el propósito de los Snapshots en Event Sourcing. ¿Cuándo son necesarios?

**Criterios de evaluación**:
- Problema: aggregates con miles de eventos son lentos
- Solución: guardar snapshot del estado cada N eventos
- Cálculo: snapshot + eventos posteriores
- Trade-off: storage vs performance

---

## Parte 2: Análisis y Refactoring (30 puntos)

### Ejercicio 1 (15 puntos)
Este Event Store tiene problemas de performance. Identifícalos y optimízalo:

```python
class EventStore:
    def load_aggregate(self, aggregate_id: str):
        # Load ALL events every time
        events = self.db.query(
            "SELECT * FROM events WHERE aggregate_id = ?",
            aggregate_id
        )
        
        # Rebuild from scratch
        aggregate = BankAccount()
        for event in events:
            aggregate.apply(event)
        
        return aggregate
```

**Problemas**:
- ❌ Carga TODOS los eventos siempre (lento con muchos eventos)
- ❌ No usa snapshots
- ❌ No cachea aggregates

**Solución esperada**:

```python
class EventStore:
    SNAPSHOT_INTERVAL = 100
    
    def __init__(self):
        self.cache = {}  # In-memory cache
        self.snapshots = {}
    
    def load_aggregate(self, aggregate_id: str):
        # 1. Check cache
        if aggregate_id in self.cache:
            return self.cache[aggregate_id]
        
        # 2. Load latest snapshot
        snapshot = self.load_snapshot(aggregate_id)
        
        if snapshot:
            # Restore from snapshot
            aggregate = self.hydrate_snapshot(snapshot)
            
            # Load only events AFTER snapshot
            events = self.db.query(
                """
                SELECT * FROM events 
                WHERE aggregate_id = ? AND version > ?
                ORDER BY version
                """,
                aggregate_id, snapshot.version
            )
        else:
            # No snapshot, load all events
            aggregate = BankAccount()
            events = self.db.query(
                "SELECT * FROM events WHERE aggregate_id = ? ORDER BY version",
                aggregate_id
            )
        
        # 3. Replay events
        for event in events:
            aggregate.apply(event)
        
        # 4. Cache result
        self.cache[aggregate_id] = aggregate
        
        return aggregate
    
    def save_event(self, aggregate_id: str, event: DomainEvent):
        self.db.execute(
            "INSERT INTO events (aggregate_id, version, event_data) VALUES (?, ?, ?)",
            aggregate_id, event.version, event
        )
        
        # Create snapshot every N events
        if event.version % self.SNAPSHOT_INTERVAL == 0:
            aggregate = self.load_aggregate(aggregate_id)
            self.save_snapshot(aggregate_id, aggregate)
        
        # Invalidate cache
        self.cache.pop(aggregate_id, None)
```

**Mejoras de Performance**:
- ✅ Snapshots: Reduce eventos a replayar
- ✅ Carga incremental: Solo eventos después del snapshot
- ✅ Cache: Evita reconstruir aggregates frecuentes
- ✅ Invalidación de cache apropiada

---

### Ejercicio 2 (15 puntos)
Este código mezcla write y read models. Refactorízalo usando CQRS:

```typescript
class OrderService {
    async createOrder(customerId: string, items: any[]) {
        const order = new Order(customerId, items);
        await db.save(order);
        return order;
    }
    
    async getOrder(orderId: string) {
        return await db.findById(orderId);
    }
    
    async getCustomerOrders(customerId: string) {
        return await db.query(
            'SELECT * FROM orders WHERE customer_id = ? ORDER BY created_at DESC',
            customerId
        );
    }
}
```

**Problemas**:
- ❌ No separa write/read
- ❌ No usa eventos
- ❌ Read model no optimizado (joins costosos)

**Solución esperada**:

```typescript
// ========== WRITE SIDE ==========

// Command
interface CreateOrderCommand {
    customerId: string;
    items: OrderItem[];
}

// Event
interface OrderCreatedEvent {
    orderId: string;
    customerId: string;
    customerName: string;  // Denormalized
    items: OrderItem[];
    total: number;
    version: number;
    timestamp: Date;
}

// Command Handler (Write)
class OrderCommandHandler {
    constructor(
        private eventStore: EventStore,
        private eventBus: EventBus
    ) {}
    
    async handle(command: CreateOrderCommand): Promise<string> {
        // 1. Load customer (for denormalization)
        const customer = await customerService.get(command.customerId);
        
        // 2. Create aggregate
        const order = new Order();
        const event = order.create(command, customer);
        
        // 3. Save event
        await this.eventStore.save(event.orderId, event);
        
        // 4. Publish
        await this.eventBus.publish(event);
        
        return event.orderId;
    }
}

// ========== READ SIDE ==========

// Read Model (Denormalized)
interface OrderView {
    orderId: string;
    customerId: string;
    customerName: string;      // Denormalized
    customerEmail: string;     // Denormalized
    itemCount: number;         // Pre-calculated
    total: number;             // Pre-calculated
    status: string;
    createdAt: Date;
}

// Projection (builds read model from events)
class OrderProjection {
    async project(event: OrderCreatedEvent): Promise<void> {
        // Build denormalized view
        const view: OrderView = {
            orderId: event.orderId,
            customerId: event.customerId,
            customerName: event.customerName,
            customerEmail: event.customerEmail,
            itemCount: event.items.length,
            total: event.total,
            status: 'PENDING',
            createdAt: event.timestamp
        };
        
        // Save to read database (can be different DB)
        await readDb.insert('order_views', view);
    }
}

// Query Service (Read)
class OrderQueryService {
    async getOrder(orderId: string): Promise<OrderView> {
        // Fast read from denormalized view
        return await readDb.findOne('order_views', { orderId });
    }
    
    async getCustomerOrders(customerId: string): Promise<OrderView[]> {
        // Optimized query (no joins needed)
        return await readDb.find(
            'order_views',
            { customerId },
            { sort: { createdAt: -1 } }
        );
    }
}
```

**Beneficios CQRS**:
- ✅ Write: Enfocado en validaciones y eventos
- ✅ Read: Denormalizado para queries rápidas
- ✅ Escalabilidad: Diferentes DBs para read/write
- ✅ Performance: Sin joins en queries

---

## Parte 3: Implementación (35 puntos)

### Proyecto: Sistema de Gestión de Inventario con Event Sourcing + CQRS

Implementa un sistema de inventario para un warehouse con:

#### Funcionalidades:

1. **Recepción de Stock**
   - Command: `ReceiveStockCommand(productId, quantity, supplier, cost)`
   - Event: `StockReceivedEvent`

2. **Venta de Productos**
   - Command: `SellProductCommand(productId, quantity, orderId)`
   - Event: `ProductSoldEvent`

3. **Ajuste de Inventario**
   - Command: `AdjustInventoryCommand(productId, quantity, reason)`
   - Event: `InventoryAdjustedEvent` (pérdidas, roturas, etc.)

4. **Queries**
   - Obtener stock actual de producto
   - Listar productos con stock bajo (<10 unidades)
   - Obtener historial completo de movimientos
   - Calcular valor total del inventario

#### Requisitos Técnicos:

**Event Sourcing**:
- Aggregate `Product` reconstruye desde eventos
- EventStore con optimistic concurrency
- Snapshots cada 50 eventos

**CQRS**:
- Separación completa write/read
- Projection para `ProductStockView`
- Projection para `InventoryValueView`
- Projection para `LowStockView`

**SOLID**:
- SRP: Cada handler una responsabilidad
- OCP: Sistema extensible para nuevos eventos
- LSP: Event handlers intercambiables
- DIP: Interfaces para EventStore, Projections

#### Código Base:

```typescript
// Events
interface StockReceivedEvent {
    productId: string;
    quantity: number;
    supplier: string;
    cost: number;
    version: number;
    timestamp: Date;
}

interface ProductSoldEvent {
    productId: string;
    quantity: number;
    orderId: string;
    version: number;
    timestamp: Date;
}

interface InventoryAdjustedEvent {
    productId: string;
    quantity: number;  // Can be negative
    reason: string;
    version: number;
    timestamp: Date;
}

// Aggregate
class Product {
    private id: string;
    private currentStock: number = 0;
    private version: number = 0;
    private movements: Movement[] = [];
    
    static fromEvents(events: DomainEvent[]): Product {
        const product = new Product();
        events.forEach(event => product.apply(event));
        return product;
    }
    
    receiveStock(command: ReceiveStockCommand): StockReceivedEvent {
        // TODO: Implement
    }
    
    sell(command: SellProductCommand): ProductSoldEvent {
        // TODO: Validate sufficient stock
        // TODO: Generate event
    }
    
    adjust(command: AdjustInventoryCommand): InventoryAdjustedEvent {
        // TODO: Implement
    }
    
    private apply(event: DomainEvent): void {
        // TODO: Update state based on event
    }
}

// Read Models
interface ProductStockView {
    productId: string;
    productName: string;
    currentStock: number;
    lastMovement: Date;
    stockStatus: 'OK' | 'LOW' | 'OUT';
}

interface InventoryValueView {
    productId: string;
    productName: string;
    quantity: number;
    averageCost: number;
    totalValue: number;
}
```

#### Tests Requeridos:

```typescript
describe('Event Sourcing', () => {
    it('should reconstruct product from events', () => {
        const events = [
            new StockReceivedEvent('prod-1', 100, 'Supplier A', 10, 1, new Date()),
            new ProductSoldEvent('prod-1', 30, 'order-1', 2, new Date()),
            new InventoryAdjustedEvent('prod-1', -5, 'damaged', 3, new Date())
        ];
        
        const product = Product.fromEvents(events);
        
        expect(product.currentStock).toBe(65);  // 100 - 30 - 5
        expect(product.version).toBe(3);
    });
    
    it('should prevent negative stock', () => {
        const product = new Product();
        product.apply(new StockReceivedEvent('prod-1', 10, ...));
        
        expect(() => {
            product.sell(new SellProductCommand('prod-1', 20, 'order-1'));
        }).toThrow('Insufficient stock');
    });
});

describe('CQRS Projections', () => {
    it('should build stock view from events', async () => {
        const projection = new ProductStockProjection();
        
        await projection.project(new StockReceivedEvent(...));
        
        const view = await queryService.getProductStock('prod-1');
        
        expect(view.currentStock).toBe(100);
        expect(view.stockStatus).toBe('OK');
    });
    
    it('should list low stock products', async () => {
        // Given: Products with stock 5, 15, 8
        // When: Query low stock (<10)
        // Then: Returns products with stock 5 and 8
    });
});

describe('Snapshots', () => {
    it('should create snapshot every 50 events', async () => {
        // Add 100 events
        for (let i = 0; i < 100; i++) {
            await eventStore.save('prod-1', new StockReceivedEvent(...));
        }
        
        // Should have 2 snapshots (at v50, v100)
        const snapshots = await eventStore.getSnapshots('prod-1');
        expect(snapshots).toHaveLength(2);
    });
    
    it('should load from snapshot + remaining events', async () => {
        // Given: Snapshot at v50, total 75 events
        // When: Load aggregate
        // Then: Load snapshot + replay 25 events only
        
        const spy = jest.spyOn(eventStore, 'loadEvents');
        const product = await eventStore.loadAggregate('prod-1');
        
        expect(spy).toHaveBeenCalledWith('prod-1', 50);  // Only events after v50
    });
});
```

#### Rúbrica (35 puntos):

| Criterio | Puntos | Descripción |
|----------|--------|-------------|
| **Event Sourcing** | 10 | Aggregate reconstruye correctamente, apply() bien implementado, eventos inmutables |
| **CQRS** | 10 | Separación write/read, projections correctas, queries optimizadas |
| **Snapshots** | 5 | Implementación correcta, performance mejorada |
| **SOLID** | 5 | SRP, OCP, DIP aplicados correctamente |
| **Tests** | 5 | >80% cobertura, casos edge, integration tests |

---

## Parte 4: Análisis Avanzado (10 puntos)

### Escenario Real

Tu sistema procesa 1 millón de transacciones/día. Cada `Product` aggregate tiene en promedio 10,000 eventos.

**Preguntas**:

1. **Performance** (3 puntos): ¿Cuánto tiempo toma reconstruir un aggregate sin snapshots vs con snapshots cada 100 eventos? Asume 0.1ms por evento aplicado.

   **Respuesta esperada**:
   - Sin snapshots: 10,000 × 0.1ms = 1,000ms (1 segundo)
   - Con snapshots: 100 × 0.1ms = 10ms (100x más rápido)

2. **Storage** (3 puntos): Si cada evento ocupa 1KB, ¿cuánto storage necesitas para 1 año?

   **Respuesta esperada**:
   - 1M eventos/día × 365 días × 1KB = 365GB/año
   - Con compresión (50%): ~183GB/año
   - Estrategia: Archivar eventos >1 año a cold storage

3. **Eventual Consistency** (4 puntos): Un usuario hace una compra, pero la proyección de stock tarda 2 segundos en actualizarse. Otro usuario intenta comprar el mismo producto en ese window. ¿Cómo prevenir overselling?

   **Respuesta esperada**:
   - Validar stock en Command Handler (write side), no en read side
   - Usar optimistic locking con version numbers
   - Write side es fuente de verdad, read side es eventual
   - Ejemplo código:
   ```typescript
   async handle(command: SellProductCommand) {
       // Load from event store (source of truth)
       const product = await eventStore.loadAggregate(command.productId);
       
       // Validate (will throw if insufficient)
       const event = product.sell(command);
       
       // Save with optimistic locking
       await eventStore.save(command.productId, event);  // Fails if version conflict
   }
   ```

---

## Bonus: Event Schema Evolution (10 puntos extra)

Implementa event upcasting para este escenario:

**v1** (deprecated):
```typescript
interface ProductAddedEventV1 {
    productId: string;
    name: string;
    price: number;  // Simple number in USD
}
```

**v2** (current):
```typescript
interface ProductAddedEventV2 {
    productId: string;
    name: string;
    price: Money;  // { amount: number, currency: string }
}

interface Money {
    amount: number;
    currency: string;
}
```

**Tareas**:
1. Implementar `EventUpcaster` que convierta v1 → v2
2. EventStore debe retornar siempre v2
3. Tests que validen compatibilidad

---

## Criterios de Aprobación

- **0-59 puntos**: Reprobado
- **60-69 puntos**: Aprobado (C)
- **70-79 puntos**: Bueno (B)
- **80-89 puntos**: Muy Bueno (A)
- **90-100 puntos**: Excelente (A+)

## Entregables

1. Código completo del proyecto de inventario
2. Tests con >80% cobertura
3. Performance benchmarks (con/sin snapshots)
4. Documento con respuestas a preguntas teóricas
5. README con arquitectura y decisiones de diseño

**Tiempo estimado**: 6-8 horas

**Formato**: Repositorio Git con CI/CD (GitHub Actions)
