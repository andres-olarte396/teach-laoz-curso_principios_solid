# Events vs Messages: Fundamentos de Event-Driven Architecture

## Objetivos
- Comprender la diferencia entre Events y Commands
- Diseñar event-driven systems con SOLID
- Implementar event sourcing básico
- Aplicar patterns de mensajería

## 1. Events vs Commands vs Queries

```
┌─────────────────────────────────────────────────┐
│              Message Types                      │
├─────────────────┬─────────────────┬─────────────┤
│    Command      │      Event      │    Query    │
├─────────────────┼─────────────────┼─────────────┤
│ Imperativo      │ Pasado          │ Pregunta    │
│ "CreateOrder"   │ "OrderCreated"  │ "GetOrder"  │
│ "ProcessPayment"│ "PaymentProcessed"│ "GetTotal"│
├─────────────────┼─────────────────┼─────────────┤
│ 1 Handler       │ N Handlers      │ 1 Handler   │
│ Synchronous     │ Asynchronous    │ Synchronous │
│ Can fail        │ Fact (happened) │ Read-only   │
│ Changes state   │ Notifies change │ No side fx  │
└─────────────────┴─────────────────┴─────────────┘
```

### 1.1 Commands (Intención de Cambio)

```typescript
// Command: Imperativo, puede fallar
interface CreateOrderCommand {
    customerId: string;
    items: Array<{
        productId: string;
        quantity: number;
    }>;
    paymentMethod: PaymentMethod;
}

class OrderCommandHandler {
    constructor(
        private orderRepository: OrderRepository,
        private inventoryService: InventoryService,
        private eventBus: EventBus
    ) {}
    
    async handle(command: CreateOrderCommand): Promise<Order> {
        // Validar (puede fallar)
        const customer = await this.validateCustomer(command.customerId);
        
        // Verificar inventario (puede fallar)
        const available = await this.inventoryService.checkAvailability(
            command.items
        );
        
        if (!available) {
            throw new Error('Insufficient inventory');
        }
        
        // Crear orden
        const order = new Order({
            id: generateId(),
            customerId: command.customerId,
            items: command.items,
            status: OrderStatus.PENDING
        });
        
        await this.orderRepository.save(order);
        
        // Emitir evento (notifica el cambio)
        await this.eventBus.publish(new OrderCreatedEvent({
            orderId: order.id,
            customerId: order.customerId,
            items: order.items,
            total: order.total,
            timestamp: new Date()
        }));
        
        return order;
    }
}
```

### 1.2 Events (Hechos Inmutables)

```java
// Event: Pasado, inmutable, representa un hecho
public record OrderCreatedEvent(
    String orderId,
    String customerId,
    List<OrderItem> items,
    Money total,
    Instant timestamp
) implements DomainEvent {
    
    // Events son inmutables - solo constructor
    public OrderCreatedEvent {
        Objects.requireNonNull(orderId, "orderId cannot be null");
        Objects.requireNonNull(timestamp, "timestamp cannot be null");
        items = List.copyOf(items);  // Defensive copy
    }
}

// Múltiples handlers pueden escuchar el mismo evento
@Component
public class InventoryEventHandler {
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Reservar inventario
        inventoryService.reserve(event.items());
    }
}

@Component
public class LoyaltyEventHandler {
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Otorgar puntos de lealtad
        loyaltyService.awardPoints(event.customerId(), event.total());
    }
}

@Component
public class NotificationEventHandler {
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Enviar confirmación
        notificationService.sendOrderConfirmation(event);
    }
}
```

## 2. Event-Driven Architecture con SOLID

### 2.1 Single Responsibility Principle (SRP)

Cada event handler tiene una responsabilidad única.

```python
# ❌ MAL: Handler con múltiples responsabilidades
class OrderEventHandler:
    def handle_order_created(self, event: OrderCreatedEvent):
        # Reservar inventario
        self.inventory_db.execute(
            "UPDATE products SET stock = stock - ? WHERE id = ?",
            (event.quantity, event.product_id)
        )
        
        # Enviar email
        self.email_service.send(
            to=event.customer_email,
            subject="Order Confirmation",
            body=f"Order {event.order_id} created"
        )
        
        # Actualizar analytics
        self.analytics_db.execute(
            "INSERT INTO daily_stats (date, orders) VALUES (?, ?)",
            (datetime.now(), 1)
        )
        
        # Procesar pago
        self.payment_gateway.charge(event.payment_method, event.total)

# ✅ BIEN: Handlers separados con SRP
class InventoryEventHandler:
    def handle_order_created(self, event: OrderCreatedEvent):
        # Solo responsable de inventario
        self.inventory_service.reserve(event.items)

class NotificationEventHandler:
    def handle_order_created(self, event: OrderCreatedEvent):
        # Solo responsable de notificaciones
        self.notification_service.send_order_confirmation(event)

class AnalyticsEventHandler:
    def handle_order_created(self, event: OrderCreatedEvent):
        # Solo responsable de analytics
        self.analytics_service.record_order(event)

class PaymentEventHandler:
    def handle_order_created(self, event: OrderCreatedEvent):
        # Solo responsable de pagos
        self.payment_service.process(event)
```

### 2.2 Open/Closed Principle (OCP)

Sistema abierto a nuevos handlers sin modificar código existente.

```typescript
// Event bus extensible
interface EventHandler<T extends DomainEvent> {
    handle(event: T): Promise<void>;
}

class EventBus {
    private handlers: Map<string, EventHandler<any>[]> = new Map();
    
    // Abierto para extensión
    register<T extends DomainEvent>(
        eventType: string,
        handler: EventHandler<T>
    ): void {
        if (!this.handlers.has(eventType)) {
            this.handlers.set(eventType, []);
        }
        this.handlers.get(eventType)!.push(handler);
    }
    
    async publish<T extends DomainEvent>(event: T): Promise<void> {
        const eventType = event.constructor.name;
        const handlers = this.handlers.get(eventType) || [];
        
        // Ejecutar handlers en paralelo
        await Promise.all(
            handlers.map(handler => handler.handle(event))
        );
    }
}

// Agregar nuevo handler sin modificar EventBus
class FraudDetectionHandler implements EventHandler<OrderCreatedEvent> {
    async handle(event: OrderCreatedEvent): Promise<void> {
        const riskScore = await this.fraudService.analyze(event);
        if (riskScore > 0.8) {
            await this.eventBus.publish(new SuspiciousOrderDetectedEvent({
                orderId: event.orderId,
                riskScore
            }));
        }
    }
}

// Registrar sin modificar código existente
eventBus.register('OrderCreatedEvent', new InventoryEventHandler());
eventBus.register('OrderCreatedEvent', new NotificationEventHandler());
eventBus.register('OrderCreatedEvent', new FraudDetectionHandler());  // Nuevo
```

### 2.3 Liskov Substitution Principle (LSP)

Event handlers intercambiables.

```java
// Base interface
public interface EventHandler<T extends DomainEvent> {
    void handle(T event);
}

// Implementations deben ser intercambiables
public class EmailNotificationHandler implements EventHandler<OrderCreatedEvent> {
    @Override
    public void handle(OrderCreatedEvent event) {
        emailService.send(
            event.customerEmail(),
            "Order Confirmation",
            buildEmailBody(event)
        );
    }
}

public class SMSNotificationHandler implements EventHandler<OrderCreatedEvent> {
    @Override
    public void handle(OrderCreatedEvent event) {
        smsService.send(
            event.customerPhone(),
            "Order " + event.orderId() + " confirmed"
        );
    }
}

// Ambos son intercambiables - LSP
List<EventHandler<OrderCreatedEvent>> handlers = List.of(
    new EmailNotificationHandler(),
    new SMSNotificationHandler()
);

for (EventHandler<OrderCreatedEvent> handler : handlers) {
    handler.handle(orderCreatedEvent);  // Polimorfismo
}
```

### 2.4 Dependency Inversion Principle (DIP)

Depender de abstracciones, no de implementaciones concretas.

```kotlin
// ❌ MAL: Dependencia de implementación concreta
class OrderService(
    private val kafkaProducer: KafkaProducer<String, Event>  // Concrete
) {
    fun createOrder(command: CreateOrderCommand): Order {
        val order = Order(...)
        
        // Acoplado a Kafka
        kafkaProducer.send(ProducerRecord(
            "orders",
            order.id,
            OrderCreatedEvent(...)
        ))
        
        return order
    }
}

// ✅ BIEN: Dependencia de abstracción
interface EventPublisher {
    suspend fun publish(event: DomainEvent)
}

class OrderService(
    private val eventPublisher: EventPublisher  // Abstraction
) {
    suspend fun createOrder(command: CreateOrderCommand): Order {
        val order = Order(...)
        
        // Desacoplado de implementación
        eventPublisher.publish(OrderCreatedEvent(...))
        
        return order
    }
}

// Implementations
class KafkaEventPublisher(
    private val producer: KafkaProducer<String, Event>
) : EventPublisher {
    override suspend fun publish(event: DomainEvent) {
        producer.send(ProducerRecord("events", event.id, event))
    }
}

class RabbitMQEventPublisher(
    private val channel: Channel
) : EventPublisher {
    override suspend fun publish(event: DomainEvent) {
        channel.basicPublish(
            "events-exchange",
            event.javaClass.simpleName,
            null,
            objectMapper.writeValueAsBytes(event)
        )
    }
}

class InMemoryEventPublisher : EventPublisher {
    private val events = mutableListOf<DomainEvent>()
    
    override suspend fun publish(event: DomainEvent) {
        events.add(event)
    }
}
```

## 3. Event Patterns

### 3.1 Event Notification (Simple)

```typescript
// Producer
class OrderService {
    async createOrder(command: CreateOrderCommand): Promise<Order> {
        const order = new Order(command);
        await this.repository.save(order);
        
        // Fire and forget - no espera respuesta
        await this.eventBus.publish(new OrderCreatedEvent({
            orderId: order.id,
            timestamp: new Date()
        }));
        
        return order;
    }
}

// Consumer
class InventoryService {
    @EventHandler(OrderCreatedEvent)
    async handleOrderCreated(event: OrderCreatedEvent): Promise<void> {
        // Procesa de forma asíncrona
        await this.reserveInventory(event.items);
    }
}
```

### 3.2 Event-Carried State Transfer

```java
// Event contiene todos los datos necesarios (no queries adicionales)
public record OrderCreatedEvent(
    String orderId,
    CustomerInfo customer,  // Datos completos del customer
    List<ProductInfo> products,  // Datos completos de productos
    Money total,
    Address shippingAddress,
    Instant createdAt
) implements DomainEvent {}

public class ShippingEventHandler {
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // No necesita consultar otros servicios
        Shipment shipment = new Shipment(
            event.orderId(),
            event.shippingAddress(),
            event.products().stream()
                .map(p -> new ShipmentItem(p.id(), p.name(), p.weight()))
                .toList()
        );
        
        shipmentRepository.save(shipment);
    }
}
```

### 3.3 Event Sourcing (Preview)

```python
# Guardar eventos en lugar de estado
class OrderEventStore:
    def __init__(self, db: Database):
        self.db = db
    
    def save_event(self, event: DomainEvent):
        self.db.execute(
            """
            INSERT INTO event_store (
                aggregate_id, 
                event_type, 
                event_data, 
                version, 
                timestamp
            ) VALUES (?, ?, ?, ?, ?)
            """,
            (
                event.aggregate_id,
                event.__class__.__name__,
                json.dumps(event.to_dict()),
                event.version,
                event.timestamp
            )
        )
    
    def load_events(self, aggregate_id: str) -> List[DomainEvent]:
        rows = self.db.query(
            """
            SELECT event_type, event_data 
            FROM event_store 
            WHERE aggregate_id = ? 
            ORDER BY version
            """,
            (aggregate_id,)
        )
        
        return [self.deserialize_event(row) for row in rows]

# Reconstruir estado desde eventos
class Order:
    def __init__(self, order_id: str):
        self.id = order_id
        self.items = []
        self.status = OrderStatus.PENDING
        self.version = 0
    
    @staticmethod
    def from_events(events: List[DomainEvent]) -> 'Order':
        # Replay events
        order = Order(events[0].aggregate_id)
        for event in events:
            order.apply(event)
        return order
    
    def apply(self, event: DomainEvent):
        if isinstance(event, OrderCreatedEvent):
            self.items = event.items
            self.status = OrderStatus.PENDING
        elif isinstance(event, OrderConfirmedEvent):
            self.status = OrderStatus.CONFIRMED
        elif isinstance(event, OrderShippedEvent):
            self.status = OrderStatus.SHIPPED
        
        self.version += 1
```

## 4. Guaranteed Delivery: Outbox Pattern

```java
// Transactional outbox para garantizar entrega
@Service
public class OrderService {
    
    @Transactional
    public Order createOrder(CreateOrderCommand command) {
        // 1. Guardar orden (en transacción)
        Order order = new Order(command);
        orderRepository.save(order);
        
        // 2. Guardar evento en outbox (misma transacción)
        OutboxEvent outboxEvent = new OutboxEvent(
            UUID.randomUUID().toString(),
            "OrderCreatedEvent",
            objectMapper.writeValueAsString(new OrderCreatedEvent(order)),
            Instant.now()
        );
        outboxRepository.save(outboxEvent);
        
        // Commit atómico: orden + evento
        return order;
    }
}

// Background job procesa outbox
@Component
public class OutboxProcessor {
    
    @Scheduled(fixedDelay = 1000)  // Cada segundo
    public void processOutbox() {
        List<OutboxEvent> pending = outboxRepository.findPending();
        
        for (OutboxEvent event : pending) {
            try {
                // Publicar a Kafka
                kafkaProducer.send(new ProducerRecord<>(
                    "events",
                    event.getId(),
                    event.getPayload()
                ));
                
                // Marcar como procesado
                event.setProcessed(true);
                outboxRepository.save(event);
                
            } catch (Exception e) {
                log.error("Failed to process outbox event", e);
                // Retry en próxima ejecución
            }
        }
    }
}
```

## Resumen

### Events vs Commands

| Aspecto | Command | Event |
|---------|---------|-------|
| **Tiempo** | Futuro (intención) | Pasado (hecho) |
| **Naming** | Imperativo (CreateOrder) | Pasado (OrderCreated) |
| **Handlers** | 1 (single receiver) | N (multiple subscribers) |
| **Falla** | Puede fallar | Ya ocurrió (fact) |
| **Mutabilidad** | Mutable | Inmutable |
| **Ejemplo** | PlaceOrder, ProcessPayment | OrderPlaced, PaymentProcessed |

### SOLID en Event-Driven

1. **SRP**: Un handler, una responsabilidad
2. **OCP**: Nuevos handlers sin modificar código
3. **LSP**: Handlers intercambiables
4. **ISP**: Interfaces específicas por tipo de evento
5. **DIP**: Abstracciones (EventPublisher) sobre implementaciones

### Best Practices

✅ **DO**:
- Events inmutables
- Event naming en pasado
- State transfer en eventos (evitar queries)
- Outbox pattern para garantías
- Idempotent handlers

❌ **DON'T**:
- Eventos mutables
- Lógica de negocio en event bus
- Eventos gigantes (> 1MB)
- Dependencias cíclicas entre handlers
- Asumir orden de procesamiento
