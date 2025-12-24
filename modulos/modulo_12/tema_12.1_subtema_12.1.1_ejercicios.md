# Ejercicios: Events vs Messages

## ⭐ Ejercicio 1. Identificar Commands vs Events

Clasifica los siguientes mensajes como Command o Event:

1. `CreateUser`
2. `UserRegistered`
3. `SendEmail`
4. `PaymentProcessed`
5. `CancelOrder`
6. `OrderCancelled`
7. `UpdateInventory`
8. `InventoryUpdated`

<details>
<summary>💡 Solución</summary>

| Mensaje | Tipo | Razón |
|---------|------|-------|
| `CreateUser` | ❌ Command | Imperativo, intención de crear |
| `UserRegistered` | ✅ Event | Pasado, usuario ya registrado |
| `SendEmail` | ❌ Command | Imperativo, solicita envío |
| `PaymentProcessed` | ✅ Event | Pasado, pago ya procesado |
| `CancelOrder` | ❌ Command | Imperativo, intención de cancelar |
| `OrderCancelled` | ✅ Event | Pasado, orden ya cancelada |
| `UpdateInventory` | ❌ Command | Imperativo, solicita actualización |
| `InventoryUpdated` | ✅ Event | Pasado, inventario ya actualizado |

**Regla simple**: 
- Command: Verbo imperativo (Create, Send, Update, Cancel)
- Event: Pasado participio (Created, Sent, Updated, Cancelled)

</details>

---

## ⭐⭐ Ejercicio 2: Implementar Event Bus con SRP

Crea un event bus que permita registrar múltiples handlers para el mismo evento.

**Requisitos**:
- Interface `EventHandler<T>`
- Class `EventBus` con métodos `register()` y `publish()`
- Cada handler debe tener una sola responsabilidad
- Manejar errores en handlers sin afectar otros

<details>
<summary>💡 Solución</summary>

```typescript
// Event base
interface DomainEvent {
    readonly eventId: string;
    readonly timestamp: Date;
}

// Event handler interface
interface EventHandler<T extends DomainEvent> {
    handle(event: T): Promise<void>;
}

// Event bus implementation
class EventBus {
    private handlers: Map<string, EventHandler<any>[]> = new Map();
    private logger: Logger;
    
    constructor(logger: Logger) {
        this.logger = logger;
    }
    
    register<T extends DomainEvent>(
        eventType: string,
        handler: EventHandler<T>
    ): void {
        if (!this.handlers.has(eventType)) {
            this.handlers.set(eventType, []);
        }
        this.handlers.get(eventType)!.push(handler);
        
        this.logger.info(`Registered handler for ${eventType}`);
    }
    
    async publish<T extends DomainEvent>(event: T): Promise<void> {
        const eventType = event.constructor.name;
        const handlers = this.handlers.get(eventType) || [];
        
        this.logger.info(`Publishing ${eventType} to ${handlers.length} handlers`);
        
        // Execute all handlers in parallel
        const results = await Promise.allSettled(
            handlers.map(handler => this.executeHandler(handler, event))
        );
        
        // Log failures without throwing
        results.forEach((result, index) => {
            if (result.status === 'rejected') {
                this.logger.error(
                    `Handler ${index} failed for ${eventType}`,
                    result.reason
                );
            }
        });
    }
    
    private async executeHandler<T extends DomainEvent>(
        handler: EventHandler<T>,
        event: T
    ): Promise<void> {
        try {
            await handler.handle(event);
        } catch (error) {
            // Re-throw to be caught by Promise.allSettled
            throw error;
        }
    }
}

// Example events
class OrderCreatedEvent implements DomainEvent {
    readonly eventId: string;
    readonly timestamp: Date;
    
    constructor(
        public readonly orderId: string,
        public readonly customerId: string,
        public readonly total: number
    ) {
        this.eventId = crypto.randomUUID();
        this.timestamp = new Date();
    }
}

// Example handlers (each with single responsibility)
class InventoryHandler implements EventHandler<OrderCreatedEvent> {
    constructor(private inventoryService: InventoryService) {}
    
    async handle(event: OrderCreatedEvent): Promise<void> {
        // Only responsible for inventory
        await this.inventoryService.reserve(event.orderId);
    }
}

class NotificationHandler implements EventHandler<OrderCreatedEvent> {
    constructor(private emailService: EmailService) {}
    
    async handle(event: OrderCreatedEvent): Promise<void> {
        // Only responsible for notifications
        await this.emailService.sendOrderConfirmation(event.orderId);
    }
}

class AnalyticsHandler implements EventHandler<OrderCreatedEvent> {
    constructor(private analytics: AnalyticsService) {}
    
    async handle(event: OrderCreatedEvent): Promise<void> {
        // Only responsible for analytics
        await this.analytics.recordOrder(event.total);
    }
}

// Usage
const eventBus = new EventBus(logger);

eventBus.register('OrderCreatedEvent', new InventoryHandler(inventoryService));
eventBus.register('OrderCreatedEvent', new NotificationHandler(emailService));
eventBus.register('OrderCreatedEvent', new AnalyticsHandler(analyticsService));

// Publish event
const event = new OrderCreatedEvent('order-123', 'customer-456', 99.99);
await eventBus.publish(event);
```

**Test**:
```typescript
describe('EventBus', () => {
    it('should execute all handlers for an event', async () => {
        const handler1 = mock<EventHandler<OrderCreatedEvent>>();
        const handler2 = mock<EventHandler<OrderCreatedEvent>>();
        
        const eventBus = new EventBus(logger);
        eventBus.register('OrderCreatedEvent', handler1);
        eventBus.register('OrderCreatedEvent', handler2);
        
        const event = new OrderCreatedEvent('1', '2', 100);
        await eventBus.publish(event);
        
        expect(handler1.handle).toHaveBeenCalledWith(event);
        expect(handler2.handle).toHaveBeenCalledWith(event);
    });
    
    it('should not fail when one handler throws', async () => {
        const failingHandler = {
            handle: jest.fn().mockRejectedValue(new Error('Handler failed'))
        };
        const successHandler = {
            handle: jest.fn().mockResolvedValue(undefined)
        };
        
        const eventBus = new EventBus(logger);
        eventBus.register('OrderCreatedEvent', failingHandler);
        eventBus.register('OrderCreatedEvent', successHandler);
        
        const event = new OrderCreatedEvent('1', '2', 100);
        
        // Should not throw
        await expect(eventBus.publish(event)).resolves.toBeUndefined();
        
        // Success handler should still execute
        expect(successHandler.handle).toHaveBeenCalled();
    });
});
```

</details>

---

## ⭐⭐⭐ Ejercicio 3: Implementar Outbox Pattern

Implementa outbox pattern para garantizar que eventos se publiquen después de guardar entidades.

**Requisitos**:
1. Guardar orden + evento en misma transacción
2. Background job procesa outbox cada 5 segundos
3. Reintentos automáticos en caso de fallo
4. Idempotencia (no duplicar eventos)

<details>
<summary>💡 Solución</summary>

```java
// Outbox Entity
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id
    private String id;
    
    private String eventType;
    
    @Column(columnDefinition = "TEXT")
    private String payload;
    
    private Instant createdAt;
    
    private boolean processed;
    
    private Instant processedAt;
    
    private int retryCount;
    
    // Constructors, getters, setters
}

// Repository
public interface OutboxRepository extends JpaRepository<OutboxEvent, String> {
    
    @Query("SELECT e FROM OutboxEvent e WHERE e.processed = false ORDER BY e.createdAt")
    List<OutboxEvent> findPending();
}

// Order Service with Outbox
@Service
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    private final ObjectMapper objectMapper;
    
    @Transactional
    public Order createOrder(CreateOrderCommand command) {
        // 1. Create and save order
        Order order = new Order(
            UUID.randomUUID().toString(),
            command.getCustomerId(),
            command.getItems(),
            OrderStatus.PENDING
        );
        orderRepository.save(order);
        
        // 2. Create event
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getItems(),
            order.getTotal(),
            Instant.now()
        );
        
        // 3. Save to outbox (same transaction!)
        OutboxEvent outboxEvent = new OutboxEvent();
        outboxEvent.setId(UUID.randomUUID().toString());
        outboxEvent.setEventType("OrderCreatedEvent");
        outboxEvent.setPayload(objectMapper.writeValueAsString(event));
        outboxEvent.setCreatedAt(Instant.now());
        outboxEvent.setProcessed(false);
        outboxEvent.setRetryCount(0);
        
        outboxRepository.save(outboxEvent);
        
        // Both saved atomically
        return order;
    }
}

// Outbox Processor (Background Job)
@Component
public class OutboxProcessor {
    
    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;
    
    private static final int MAX_RETRIES = 5;
    private static final Logger log = LoggerFactory.getLogger(OutboxProcessor.class);
    
    @Scheduled(fixedDelay = 5000)  // Every 5 seconds
    public void processOutbox() {
        List<OutboxEvent> pending = outboxRepository.findPending();
        
        log.info("Processing {} pending outbox events", pending.size());
        
        for (OutboxEvent event : pending) {
            processEvent(event);
        }
    }
    
    @Transactional
    private void processEvent(OutboxEvent event) {
        try {
            // Publish to Kafka
            kafkaTemplate.send(
                "domain-events",
                event.getId(),  // Key for idempotency
                event.getPayload()
            ).get(10, TimeUnit.SECONDS);
            
            // Mark as processed
            event.setProcessed(true);
            event.setProcessedAt(Instant.now());
            outboxRepository.save(event);
            
            log.info("Successfully processed outbox event {}", event.getId());
            
        } catch (Exception e) {
            log.error("Failed to process outbox event {}", event.getId(), e);
            
            // Increment retry count
            event.setRetryCount(event.getRetryCount() + 1);
            
            if (event.getRetryCount() >= MAX_RETRIES) {
                log.error("Max retries reached for event {}. Moving to DLQ", event.getId());
                // Move to dead letter queue or alert
                event.setProcessed(true);  // Don't retry anymore
            }
            
            outboxRepository.save(event);
        }
    }
}

// Kafka Consumer (with idempotency)
@Component
public class EventConsumer {
    
    private final Set<String> processedEventIds = ConcurrentHashMap.newKeySet();
    
    @KafkaListener(topics = "domain-events")
    public void handleEvent(ConsumerRecord<String, String> record) {
        String eventId = record.key();
        
        // Idempotency check
        if (processedEventIds.contains(eventId)) {
            log.info("Event {} already processed. Skipping.", eventId);
            return;
        }
        
        try {
            // Process event
            DomainEvent event = objectMapper.readValue(
                record.value(),
                DomainEvent.class
            );
            
            eventHandler.handle(event);
            
            // Mark as processed
            processedEventIds.add(eventId);
            
        } catch (Exception e) {
            log.error("Failed to handle event {}", eventId, e);
            throw e;  // Kafka will retry
        }
    }
}
```

**Test**:
```java
@SpringBootTest
class OutboxPatternTest {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private OutboxRepository outboxRepository;
    
    @Autowired
    private OutboxProcessor outboxProcessor;
    
    @Test
    void shouldSaveOrderAndEventAtomically() {
        // When
        CreateOrderCommand command = new CreateOrderCommand(
            "customer-123",
            List.of(new OrderItem("product-1", 2))
        );
        
        Order order = orderService.createOrder(command);
        
        // Then
        List<OutboxEvent> events = outboxRepository.findPending();
        assertThat(events).hasSize(1);
        assertThat(events.get(0).getEventType()).isEqualTo("OrderCreatedEvent");
        assertThat(events.get(0).isProcessed()).isFalse();
    }
    
    @Test
    void shouldProcessOutboxAndPublishToKafka() throws Exception {
        // Given
        CreateOrderCommand command = new CreateOrderCommand(
            "customer-123",
            List.of(new OrderItem("product-1", 2))
        );
        orderService.createOrder(command);
        
        // When
        outboxProcessor.processOutbox();
        
        // Then
        List<OutboxEvent> pending = outboxRepository.findPending();
        assertThat(pending).isEmpty();
        
        // Verify Kafka received event
        ConsumerRecords<String, String> records = kafkaConsumer.poll(Duration.ofSeconds(5));
        assertThat(records).hasSize(1);
    }
}
```

</details>

---

## ⭐⭐⭐⭐ Ejercicio 4: Event-Carried State Transfer

Diseña eventos que contengan toda la información necesaria para evitar queries adicionales.

**Escenario**: Sistema de reservas de vuelos
- Cuando se crea reserva, enviar email con todos los detalles
- Notification service NO debe consultar otros servicios

**Tareas**:
1. Diseñar `BookingCreatedEvent` con state transfer completo
2. Implementar handler que solo use datos del evento
3. Comparar con enfoque sin state transfer (queries adicionales)

<details>
<summary>💡 Solución</summary>

```typescript
// ❌ MAL: Event sin state transfer (requiere queries)
interface BookingCreatedEventBad {
    bookingId: string;
    customerId: string;
    flightId: string;
    timestamp: Date;
}

class NotificationHandlerBad {
    constructor(
        private customerService: CustomerService,
        private flightService: FlightService,
        private emailService: EmailService
    ) {}
    
    async handle(event: BookingCreatedEventBad): Promise<void> {
        // Multiple queries required
        const customer = await this.customerService.getCustomer(event.customerId);
        const flight = await this.flightService.getFlight(event.flightId);
        
        await this.emailService.send({
            to: customer.email,
            subject: 'Booking Confirmation',
            body: `Your flight ${flight.number} is confirmed`
        });
    }
}

// ✅ BIEN: Event con state transfer completo
interface BookingCreatedEvent {
    bookingId: string;
    timestamp: Date;
    
    // Customer data (embedded)
    customer: {
        id: string;
        name: string;
        email: string;
        phone: string;
        loyaltyTier: 'GOLD' | 'SILVER' | 'BRONZE';
    };
    
    // Flight data (embedded)
    flight: {
        id: string;
        number: string;
        airline: string;
        departure: {
            airport: string;
            city: string;
            time: Date;
            terminal: string;
            gate: string;
        };
        arrival: {
            airport: string;
            city: string;
            time: Date;
            terminal: string;
        };
        duration: string;
        class: 'ECONOMY' | 'BUSINESS' | 'FIRST';
    };
    
    // Booking data
    seat: string;
    baggageAllowance: {
        checked: number;
        carry: number;
    };
    totalPrice: {
        amount: number;
        currency: string;
    };
    
    // Payment data
    paymentMethod: {
        type: 'CREDIT_CARD' | 'DEBIT_CARD';
        last4: string;
    };
}

// Handler no necesita queries
class NotificationHandlerGood {
    constructor(private emailService: EmailService) {}
    
    async handle(event: BookingCreatedEvent): Promise<void> {
        // All data available in event
        const emailHtml = this.buildEmailTemplate(event);
        
        await this.emailService.send({
            to: event.customer.email,
            subject: `Booking Confirmation - Flight ${event.flight.number}`,
            html: emailHtml
        });
    }
    
    private buildEmailTemplate(event: BookingCreatedEvent): string {
        return `
            <h1>Booking Confirmed</h1>
            <p>Dear ${event.customer.name},</p>
            
            <h2>Flight Details</h2>
            <p><strong>Flight:</strong> ${event.flight.airline} ${event.flight.number}</p>
            <p><strong>From:</strong> ${event.flight.departure.city} (${event.flight.departure.airport})</p>
            <p><strong>Departure:</strong> ${event.flight.departure.time.toLocaleString()}</p>
            <p><strong>Terminal:</strong> ${event.flight.departure.terminal}, Gate: ${event.flight.departure.gate}</p>
            
            <p><strong>To:</strong> ${event.flight.arrival.city} (${event.flight.arrival.airport})</p>
            <p><strong>Arrival:</strong> ${event.flight.arrival.time.toLocaleString()}</p>
            <p><strong>Duration:</strong> ${event.flight.duration}</p>
            
            <h2>Booking Details</h2>
            <p><strong>Booking ID:</strong> ${event.bookingId}</p>
            <p><strong>Class:</strong> ${event.flight.class}</p>
            <p><strong>Seat:</strong> ${event.seat}</p>
            <p><strong>Baggage:</strong> ${event.baggageAllowance.checked} checked, ${event.baggageAllowance.carry} carry-on</p>
            
            <h2>Payment</h2>
            <p><strong>Total:</strong> ${event.totalPrice.currency} ${event.totalPrice.amount}</p>
            <p><strong>Paid with:</strong> ${event.paymentMethod.type} ending in ${event.paymentMethod.last4}</p>
            
            ${event.customer.loyaltyTier === 'GOLD' ? '<p><strong>As a Gold member, you earn 2x miles!</strong></p>' : ''}
        `;
    }
}

// Comparison: Performance impact
// Bad approach: 1 event handler + 2 API calls (customer + flight) = ~150ms
// Good approach: 1 event handler + 0 API calls = ~5ms
// 30x faster!

// Trade-off: Event size
// Bad: ~200 bytes
// Good: ~2KB
// Still acceptable for most message brokers (Kafka supports up to 1MB)
```

**Benefits**:
1. ✅ No coupling to other services
2. ✅ Faster processing (no queries)
3. ✅ Works even if other services are down
4. ✅ Event is self-contained documentation

**Trade-offs**:
1. ❌ Larger event size
2. ❌ Data duplication
3. ❌ Potential stale data (if customer updates email after booking)

</details>

---

## Proyecto Final: Event-Driven E-commerce

Implementa sistema de e-commerce con events:

1. **Commands**: CreateOrder, ProcessPayment, ShipOrder
2. **Events**: OrderCreated, PaymentProcessed, OrderShipped
3. **Handlers**: Inventory, Notification, Analytics, Loyalty
4. **Outbox Pattern** para guaranteed delivery
5. **Event-Carried State Transfer** para evitar queries

**Rúbrica**:
- ⭐ Commands y Events correctamente separados
- ⭐⭐ Event bus con múltiples handlers (SRP)
- ⭐⭐⭐ Outbox pattern implementado
- ⭐⭐⭐⭐ State transfer + monitoring + retry logic

## Recursos

- **Event-Driven Microservices** - Chris Richardson
- **Kafka Documentation**: https://kafka.apache.org/documentation/
- **Martin Fowler - Event Sourcing**: https://martinfowler.com/eaaDev/EventSourcing.html
