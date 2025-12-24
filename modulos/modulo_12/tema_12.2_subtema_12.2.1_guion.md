# Guion de Video: CQRS Pattern

**Duración**: 30 minutos  
**Nivel**: Avanzado  
**Requisitos previos**: Event-Driven Architecture, Event Sourcing básico

---

## [00:00 - 01:30] Introducción - El Problema

**[PANTALLA: Título animado "CQRS: Command Query Responsibility Segregation"]**

🎬 **Narración**:
"Bienvenidos a uno de los patrones más poderosos en arquitecturas modernas: CQRS.

Imagina que estás construyendo Amazon. Por un lado, necesitas procesar millones de pedidos con validaciones estrictas: verificar stock, procesar pagos, aplicar reglas de negocio complejas.

Por otro lado, necesitas mostrar catálogos de productos a millones de usuarios simultáneamente, con búsquedas full-text, filtros por categoría, ordenamiento por precio.

¿El problema? Estos dos requisitos son **completamente diferentes**. Usar el mismo modelo para ambos es como usar un martillo para todo: funciona, pero no es óptimo.

Hoy veremos cómo CQRS separa estas responsabilidades, cómo escalar reads y writes independientemente, y por qué empresas como Netflix, Uber y bancos usan este patrón."

**[VISUAL: Logos de Netflix, Uber, bancos]**

---

## [01:30 - 07:00] El Problema con Modelos Únicos

**[PANTALLA: Código tradicional]**

🎬 **Narración**:
"Empecemos viendo qué está mal con un modelo único. Este es código típico que vemos en aplicaciones tradicionales."

**[VISUAL: Código Java]**

```java
@Entity
public class Order {
    @Id
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "customer_id")
    private Customer customer;  // JOIN necesario
    
    @OneToMany(mappedBy = "order")
    private List<OrderItem> items;  // JOIN necesario
    
    private OrderStatus status;
    private BigDecimal total;
}

@Service
public class OrderService {
    // Write
    public Order createOrder(CreateOrderDto dto) {
        Order order = new Order();
        order.setCustomer(customerRepo.findById(dto.customerId()));
        order.setItems(dto.items());
        // Validaciones...
        return orderRepo.save(order);
    }
    
    // Read
    public List<OrderSummary> getCustomerOrders(Long customerId) {
        // Query compleja con múltiples JOINs
        return orderRepo.findByCustomerWithItems(customerId);
    }
}
```

**[ANIMACIÓN: Mostrar problema de performance]**

🎬 **Narración**:
"¿Ven el problema? Para **escribir** una orden, necesitamos cargar el customer (JOIN). Para **leer** órdenes, necesitamos cargar items (otro JOIN).

Pero aquí está la cuestión: cuando escribes, NO necesitas todos esos datos. Solo necesitas validar que el customer existe.

Y cuando lees para mostrar en UI, NO necesitas la entidad completa con todas sus relaciones. Solo necesitas un resumen denormalizado."

**[VISUAL: Benchmark animado]**

```
Traditional (con JOINs):
  Write: 150ms (carga customer + items)
  Read:  200ms (3 JOINs: order + customer + items)

CQRS (modelos separados):
  Write: 10ms (solo validación de ID)
  Read:  5ms (tabla denormalizada, sin JOINs)
  
Improvement: 15x writes, 40x reads! 🚀
```

---

## [07:00 - 13:00] CQRS: La Solución

**[TRANSICIÓN: Arquitectura CQRS]**

🎬 **Narración**:
"CQRS resuelve esto separando **completamente** el modelo de escritura del modelo de lectura."

**[VISUAL: Diagrama CQRS animado]**

```
┌─────────────────────────────────────────────┐
│           CQRS Architecture                 │
└─────────────────────────────────────────────┘

WRITE SIDE:                    READ SIDE:
┌────────────────┐            ┌─────────────────┐
│  CreateOrder   │            │  GetOrder       │
│  CancelOrder   │            │  ListOrders     │
│  UpdateOrder   │            │  SearchOrders   │
└───────┬────────┘            └────────▲────────┘
        │                              │
        ▼                              │
   ┌─────────┐                         │
   │  Event  │──────Events──────┐      │
   │  Store  │                  │      │
   └─────────┘                  ▼      │
                         ┌──────────────┴────┐
                         │   Projection      │
                         │  (Read Model)     │
                         └───────────────────┘
                                │
                         ┌──────▼──────┐
                         │   Read DB   │
                         │(Denormalized)│
                         └─────────────┘
```

🎬 **Narración**:
"El **Write Side** solo maneja commands: crear, actualizar, cancelar. Está optimizado para validaciones y generación de eventos.

El **Read Side** solo maneja queries: listar, buscar, filtrar. Está optimizado para lecturas rápidas con datos denormalizados.

Lo crucial: pueden usar **bases de datos diferentes**."

**[DEMO: Código TypeScript]**

```typescript
// ========== WRITE SIDE ==========

// Command (inmutable)
interface CreateOrderCommand {
    customerId: string;
    items: OrderItem[];
}

// Write Model (solo validaciones)
class Order {
    create(command: CreateOrderCommand): OrderCreatedEvent {
        // Solo validaciones, NO carga datos completos
        if (command.items.length === 0) {
            throw new Error('Order must have items');
        }
        
        if (command.items.length > 100) {
            throw new Error('Max 100 items');
        }
        
        // Genera evento
        return {
            orderId: generateId(),
            customerId: command.customerId,
            items: command.items,
            total: this.calculateTotal(command.items),
            timestamp: new Date()
        };
    }
}

// Command Handler (orquestación)
class CreateOrderCommandHandler {
    async handle(command: CreateOrderCommand): Promise<string> {
        // 1. Validate customer exists (solo ID check, rápido)
        await this.customerService.validateExists(command.customerId);
        
        // 2. Create aggregate
        const order = new Order();
        const event = order.create(command);
        
        // 3. Save event
        await this.eventStore.save(event.orderId, event);
        
        // 4. Publish for projections
        await this.eventBus.publish(event);
        
        return event.orderId;  // Solo retorna ID
    }
}
```

**[TRANSICIÓN: Read Side]**

```typescript
// ========== READ SIDE ==========

// Read Model (denormalizado)
interface OrderView {
    orderId: string;
    customerName: string;      // ✅ Denormalizado
    customerEmail: string;     // ✅ Denormalizado
    itemCount: number;         // ✅ Pre-calculado
    total: number;
    status: string;
    createdAt: Date;
}

// Projection (construye read model desde eventos)
class OrderViewProjection {
    async on(event: OrderCreatedEvent): Promise<void> {
        // Fetch customer data ONCE (para denormalizar)
        const customer = await this.customerService.get(event.customerId);
        
        // Save denormalized view
        await this.readDb.insert('order_views', {
            order_id: event.orderId,
            customer_name: customer.name,    // Denormalizado
            customer_email: customer.email,  // Denormalizado
            item_count: event.items.length,  // Pre-calculado
            total: event.total,
            status: 'PENDING',
            created_at: event.timestamp
        });
    }
}

// Query Service (lectura simple)
class OrderQueryService {
    async getCustomerOrders(customerId: string): Promise<OrderView[]> {
        // ✅ Query simple, SIN joins
        return await this.readDb.query(
            'SELECT * FROM order_views WHERE customer_id = ? ORDER BY created_at DESC',
            [customerId]
        );
    }
}
```

🎬 **Narración**:
"¿Ven la diferencia? El write side NO carga datos completos. Solo valida y genera eventos.

El read side tiene TODO denormalizado. Customer name, email, item count... todo pre-calculado. Las queries son **trivialmente rápidas**."

---

## [13:00 - 18:00] Múltiples Read Models

**[PANTALLA: Múltiples proyecciones]**

🎬 **Narración**:
"Aquí viene la parte realmente poderosa: puedes tener **múltiples read models** del mismo stream de eventos."

**[ANIMACIÓN: Un evento → 3 proyecciones]**

```typescript
// Mismo evento, 3 read models diferentes

// Read Model 1. Para UI (lista básica)
interface OrderListView {
    orderId: string;
    customerName: string;
    total: number;
    status: string;
}

// Read Model 2: Para analytics (agregaciones)
interface OrderAnalyticsView {
    date: Date;
    totalOrders: number;
    totalRevenue: number;
    averageOrderValue: number;
    topProducts: string[];
}

// Read Model 3: Para reportes (histórico completo)
interface OrderAuditView {
    orderId: string;
    allEvents: DomainEvent[];  // Historia completa
    timeline: TimelineEntry[];
    customerSnapshot: Customer;
}
```

**[DEMO: Proyecciones en acción]**

🎬 **Narración**:
"Cuando publicas un `OrderCreatedEvent`, las 3 proyecciones lo procesan **en paralelo**."

```python
class EventBus:
    async def publish(self, event: DomainEvent):
        # Run all projections in parallel
        await asyncio.gather(
            order_list_projection.project(event),
            analytics_projection.project(event),
            audit_projection.project(event)
        )
```

"Cada una construye SU vista optimizada. La lista UI está en PostgreSQL con índices. Analytics en ClickHouse para agregaciones. Audit en MongoDB para documentos completos.

¿El costo? Storage. Pero storage es **barato**. Performance es **cara**."

---

## [18:00 - 23:00] Diferentes Bases de Datos

**[PANTALLA: Múltiples DBs]**

🎬 **Narración**:
"Ahora la pregunta del millón: ¿qué base de datos usas para cada lado?"

**[VISUAL: Comparación de DBs]**

```
Write DB (Event Store):
✅ PostgreSQL
   - ACID guarantees
   - Optimistic locking
   - Transactional events

Read DB 1 (Structured Queries):
✅ PostgreSQL/MySQL
   - Índices para filtros
   - Joins eficientes si necesarios
   - Queries complejas

Read DB 2 (Full-Text Search):
✅ Elasticsearch
   - Búsqueda fuzzy
   - Relevance scoring
   - Agregaciones

Read DB 3 (High Performance):
✅ Redis
   - Cache de queries frecuentes
   - Contadores en memoria
   - TTL automático
```

**[DEMO: Código con múltiples DBs]**

```typescript
// Write Side: PostgreSQL
const eventStore = new PostgresEventStore({
    host: 'write-db.example.com',
    database: 'events',
    // ACID transactions
});

// Read Side 1. MongoDB (documentos denormalizados)
const orderViewStore = new MongoDBStore({
    host: 'read-db-mongo.example.com',
    database: 'order_views'
});

// Read Side 2: Elasticsearch (búsqueda)
const searchStore = new ElasticsearchStore({
    host: 'search.example.com'
});

// Projections usan diferentes DBs
class OrderViewProjection {
    async on(event: OrderCreatedEvent) {
        // Save to MongoDB
        await orderViewStore.insert({
            _id: event.orderId,
            ...denormalizedData
        });
        
        // Index in Elasticsearch
        await searchStore.index('orders', {
            id: event.orderId,
            searchableFields: extractSearchable(event)
        });
    }
}
```

🎬 **Narración**:
"Cada DB optimizada para su propósito. Esto es **escalabilidad real**."

---

## [23:00 - 27:00] SOLID en CQRS

**[PANTALLA: Principios SOLID]**

🎬 **Narración**:
"CQRS no significa abandonar SOLID. De hecho, lo refuerza dramáticamente."

**[VISUAL: Código Kotlin con SRP]**

```kotlin
// ✅ Single Responsibility Principle

// Command Handler: Solo orquestación
class CreateOrderCommandHandler(
    private val eventStore: EventStore,
    private val eventBus: EventBus
) {
    suspend fun handle(command: CreateOrderCommand): String {
        val order = Order()
        val event = order.create(command)
        
        eventStore.save(event.orderId, event)
        eventBus.publish(event)
        
        return event.orderId
    }
}

// Aggregate: Solo lógica de negocio
class Order {
    fun create(command: CreateOrderCommand): OrderCreatedEvent {
        // Solo validaciones
        require(command.items.isNotEmpty()) { "Must have items" }
        return OrderCreatedEvent(...)
    }
}

// Projection: Solo construcción de vista
class OrderViewProjection(private val readDb: Database) {
    suspend fun project(event: OrderCreatedEvent) {
        // Solo inserción en read model
        readDb.insert("order_views", buildView(event))
    }
}

// Query Service: Solo queries
class OrderQueryService(private val readDb: Database) {
    suspend fun getOrders(): List<OrderView> {
        // Solo lectura
        return readDb.query("SELECT * FROM order_views")
    }
}
```

🎬 **Narración**:
"Cada clase tiene **una responsabilidad**. Command Handler orquesta. Aggregate valida. Projection construye vistas. Query Service lee.

Esto hace testing **trivial**."

**[DEMO: Tests]**

```typescript
describe('CQRS with SOLID', () => {
    it('should test aggregate in isolation', () => {
        const order = new Order();
        
        const event = order.create({
            customerId: 'c1',
            items: []
        });
        
        // NO dependencies, pure logic
        expect(() => event).toThrow('Must have items');
    });
    
    it('should test projection in isolation', async () => {
        const mockDb = createMockDb();
        const projection = new OrderViewProjection(mockDb);
        
        await projection.project(mockEvent);
        
        expect(mockDb.insert).toHaveBeenCalledWith(...);
    });
});
```

---

## [27:00 - 30:00] Resumen y Trade-offs

**[PANTALLA: Comparación final]**

🎬 **Narración**:
"Recapitulemos: CQRS es poderoso, pero no es gratis."

**[VISUAL: Tabla de trade-offs]**

```
✅ BENEFICIOS:
   • Performance: 10-100x más rápido en reads
   • Escalabilidad: Escala reads/writes independientemente
   • Flexibilidad: Múltiples read models del mismo evento
   • Optimización: Cada lado optimizado para su propósito

❌ TRADE-OFFS:
   • Complejidad: Más código, más infraestructura
   • Eventual Consistency: Proyecciones asíncronas
   • Data Duplication: Múltiples copias de datos
   • Learning Curve: Equipo necesita entrenamiento
```

🎬 **Narración**:
"¿Cuándo usar CQRS?

✅ **Úsalo cuando**:
- Requisitos muy diferentes para read/write
- Alta proporción de lecturas vs escrituras (90/10)
- Necesitas escalar reads y writes independientemente
- Queries complejas con joins lentos

❌ **Evítalo cuando**:
- CRUD simple (blog, admin panel básico)
- Equipo sin experiencia en eventos
- Strong consistency requerida en todas partes
- Budget limitado de infraestructura

Recuerda: **No todo necesita CQRS**. Pero cuando lo necesitas, es transformador.

En el próximo video exploraremos **Saga Patterns**: cómo manejar transacciones distribuidas usando eventos con choreography y orchestration.

¡Hasta la próxima!"

**[PANTALLA: Recursos]**

- 📚 Ejercicios: Sistema de tickets con 3 read models
- 💡 Proyecto: E-commerce con PostgreSQL + MongoDB + Elasticsearch
- 🔗 Recursos: Axon Framework, EventStore, Projection patterns
- ▶️ Próximo: Saga Patterns (Choreography vs Orchestration)

---

## Recursos Visuales

### Animaciones Clave:
1. **Problema tradicional**: JOINs lentos, modelo único
2. **CQRS Architecture**: Separación write/read con event bus
3. **Múltiples proyecciones**: 1 evento → N read models
4. **Performance comparison**: Traditional vs CQRS benchmarks

### Código a Mostrar:
- TypeScript: Command/Query separation completa
- Python: Proyecciones con múltiples DBs
- Java: Write model con validaciones
- Kotlin: SOLID en CQRS

### Demos en Vivo:
1. Command handler generando evento
2. 3 proyecciones procesando mismo evento
3. Query performance (con/sin denormalización)
4. Tests unitarios de componentes aislados

### Diagramas:
1. Traditional architecture (single model)
2. CQRS architecture (complete)
3. Multiple read models from same events
4. Different databases for different purposes

## Notas para el Editor

- Resaltar visualmente separación write/read (colores: azul vs verde)
- Animación de eventos fluyendo a múltiples proyecciones
- Benchmarks con números grandes y claros
- Comparación lado a lado: traditional vs CQRS
- Zoom en queries SQL (mostrar diferencia con/sin joins)
- Performance graphs mostrando mejoras

## B-Roll Sugerido

- Dashboards de sistemas reales con CQRS
- Arquitecturas de Netflix, Uber (referencias públicas)
- Diagramas de AWS/Azure con servicios CQRS
- Código de proyectos open source (Axon, EventStore)
