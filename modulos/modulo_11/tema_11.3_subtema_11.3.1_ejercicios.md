# Ejercicios: Database per Service y Sagas

## ⭐ Ejercicio 1. Identificar Violaciones de Database per Service

Analiza este código y encuentra las violaciones:

```java
@Service
public class OrderService {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public OrderDTO getOrderDetails(String orderId) {
        // Query directa a tabla de otro servicio
        String sql = """
            SELECT o.id, o.total, u.name, u.email, p.name as product_name
            FROM orders o
            JOIN users u ON o.customer_id = u.id
            JOIN order_items oi ON o.id = oi.order_id
            JOIN products p ON oi.product_id = p.id
            WHERE o.id = ?
        """;
        
        return jdbcTemplate.queryForObject(sql, 
            new OrderDTOMapper(), 
            orderId
        );
    }
}
```

**Tareas**:
1. Identificar todas las violaciones
2. Refactorizar usando Database per Service
3. Implementar API calls o eventos

<details>
<summary>💡 Solución</summary>

**Violaciones**:
1. ❌ Join directo a `users` table (User Service data)
2. ❌ Join directo a `products` table (Product Service data)
3. ❌ Acoplamiento fuerte a esquema de otros servicios

**Refactorización**:

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final UserServiceClient userClient;
    private final ProductServiceClient productClient;
    
    public OrderDetailsDTO getOrderDetails(String orderId) {
        // 1. Get order (own database)
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new NotFoundException("Order not found"));
        
        // 2. Get customer data (User Service API)
        CustomerDTO customer = userClient.getCustomer(order.getCustomerId());
        
        // 3. Get product data (Product Service API)
        List<String> productIds = order.getItems().stream()
            .map(OrderItem::getProductId)
            .collect(Collectors.toList());
        
        Map<String, ProductDTO> products = productClient.getProducts(productIds);
        
        // 4. Compose response
        return OrderDetailsDTO.builder()
            .orderId(order.getId())
            .total(order.getTotal())
            .customerName(customer.getName())
            .customerEmail(customer.getEmail())
            .items(order.getItems().stream()
                .map(item -> new OrderItemDTO(
                    products.get(item.getProductId()).getName(),
                    item.getQuantity(),
                    item.getPrice()
                ))
                .collect(Collectors.toList()))
            .build();
    }
}

// User Service Client
@FeignClient(name = "user-service")
public interface UserServiceClient {
    @GetMapping("/customers/{id}")
    CustomerDTO getCustomer(@PathVariable String id);
}

// Product Service Client
@FeignClient(name = "product-service")
public interface ProductServiceClient {
    @PostMapping("/products/batch")
    Map<String, ProductDTO> getProducts(@RequestBody List<String> ids);
}
```

**Optimización con Cache/Read Model**:
```java
// Para queries frecuentes, usar read model
@Service
public class OrderQueryService {
    private final RedisTemplate<String, OrderView> redis;
    
    public OrderView getOrderView(String orderId) {
        // Try cache first
        OrderView cached = redis.opsForValue().get("order_view:" + orderId);
        if (cached != null) {
            return cached;
        }
        
        // Rebuild from source services
        OrderView view = buildOrderView(orderId);
        redis.opsForValue().set("order_view:" + orderId, view, 1, TimeUnit.HOURS);
        return view;
    }
}
```

</details>

---

## ⭐⭐ Ejercicio 2: Implementar Saga con Compensación

Implementa Saga choreography para este flujo:
1. Create Order
2. Reserve Inventory
3. Process Payment
4. Schedule Shipping

Si cualquier paso falla, compensar los pasos previos.

<details>
<summary>💡 Solución</summary>

```typescript
// Event definitions
interface OrderCreatedEvent {
    orderId: string;
    customerId: string;
    items: Array<{productId: string, quantity: number}>;
    total: Money;
}

interface InventoryReservedEvent {
    orderId: string;
    reservationId: string;
}

interface PaymentProcessedEvent {
    orderId: string;
    paymentId: string;
}

interface PaymentFailedEvent {
    orderId: string;
    reservationId: string;
    reason: string;
}

// Order Service
export class OrderService {
    async createOrder(command: CreateOrderCommand): Promise<string> {
        const order = new Order({
            id: generateId(),
            customerId: command.customerId,
            items: command.items,
            status: OrderStatus.PENDING
        });
        
        await this.repository.save(order);
        
        // Emit event
        await this.eventBus.publish(new OrderCreatedEvent({
            orderId: order.id,
            customerId: order.customerId,
            items: order.items,
            total: order.total
        }));
        
        return order.id;
    }
    
    @EventHandler(OrderFailedEvent)
    async handleOrderFailed(event: OrderFailedEvent): Promise<void> {
        const order = await this.repository.findById(event.orderId);
        order.cancel(event.reason);
        await this.repository.save(order);
        
        // Notify customer
        await this.notificationService.sendOrderCancelled(
            order.customerId, 
            order.id, 
            event.reason
        );
    }
}

// Inventory Service
export class InventoryService {
    @EventHandler(OrderCreatedEvent)
    async handleOrderCreated(event: OrderCreatedEvent): Promise<void> {
        try {
            // Reserve inventory
            const reservationId = await this.reserve(event.items);
            
            await this.eventBus.publish(new InventoryReservedEvent({
                orderId: event.orderId,
                reservationId
            }));
        } catch (error) {
            // Emit failure - triggers compensation
            await this.eventBus.publish(new InventoryReservationFailedEvent({
                orderId: event.orderId,
                reason: error.message
            }));
        }
    }
    
    @EventHandler(PaymentFailedEvent)
    async compensateReservation(event: PaymentFailedEvent): Promise<void> {
        // Compensating transaction
        await this.releaseReservation(event.reservationId);
        console.log(`Compensated: Released reservation ${event.reservationId}`);
    }
    
    private async reserve(items: OrderItem[]): Promise<string> {
        const reservationId = generateId();
        
        for (const item of items) {
            const available = await this.checkAvailability(item.productId);
            if (available < item.quantity) {
                throw new Error(`Insufficient inventory for ${item.productId}`);
            }
            
            await this.updateStock(item.productId, -item.quantity);
            await this.saveReservation(reservationId, item);
        }
        
        return reservationId;
    }
}

// Payment Service
export class PaymentService {
    @EventHandler(InventoryReservedEvent)
    async handleInventoryReserved(event: InventoryReservedEvent): Promise<void> {
        try {
            const paymentId = await this.processPayment(event.orderId);
            
            await this.eventBus.publish(new PaymentProcessedEvent({
                orderId: event.orderId,
                paymentId
            }));
        } catch (error) {
            // Payment failed - trigger full compensation
            await this.eventBus.publish(new PaymentFailedEvent({
                orderId: event.orderId,
                reservationId: event.reservationId,
                reason: error.message
            }));
        }
    }
    
    @EventHandler(ShippingFailedEvent)
    async compensatePayment(event: ShippingFailedEvent): Promise<void> {
        // Refund payment
        await this.refund(event.paymentId);
        console.log(`Compensated: Refunded payment ${event.paymentId}`);
    }
}

// Shipping Service
export class ShippingService {
    @EventHandler(PaymentProcessedEvent)
    async handlePaymentProcessed(event: PaymentProcessedEvent): Promise<void> {
        try {
            const shipmentId = await this.scheduleShipment(event.orderId);
            
            await this.eventBus.publish(new ShippingScheduledEvent({
                orderId: event.orderId,
                shipmentId,
                estimatedDelivery: new Date(Date.now() + 7 * 24 * 3600 * 1000)
            }));
        } catch (error) {
            // Shipping failed - trigger compensation
            await this.eventBus.publish(new ShippingFailedEvent({
                orderId: event.orderId,
                paymentId: event.paymentId,
                reason: error.message
            }));
        }
    }
}
```

**Test de Compensación**:
```typescript
describe('Saga Compensation', () => {
    it('should compensate when payment fails', async () => {
        // Arrange
        const paymentService = mock<PaymentService>();
        paymentService.processPayment.mockRejectedValue(new Error('Card declined'));
        
        // Act
        const orderId = await orderService.createOrder({
            customerId: 'cust-123',
            items: [{productId: 'prod-1', quantity: 2}]
        });
        
        // Wait for saga to complete
        await waitFor(() => eventBus.getEvents(OrderFailedEvent).length > 0);
        
        // Assert
        const order = await orderRepository.findById(orderId);
        expect(order.status).toBe(OrderStatus.CANCELLED);
        
        // Verify compensation
        const inventoryReleased = await inventoryService.checkStock('prod-1');
        expect(inventoryReleased).toBe(100); // Original stock restored
    });
});
```

</details>

---

## ⭐⭐⭐ Ejercicio 3: Saga Orchestration con Timeout

Implementa Saga orchestrator que:
- Ejecuta pasos secuencialmente
- Timeout de 30 segundos por paso
- Retries automáticos (max 3 intentos)
- Compensación completa en rollback

<details>
<summary>💡 Solución</summary>

```java
@Service
public class OrderSagaOrchestrator {
    
    private final SagaStateRepository stateRepository;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final ShippingService shippingService;
    
    @Transactional
    public SagaResult executeSaga(CreateOrderCommand command) {
        String sagaId = UUID.randomUUID().toString();
        SagaState state = new SagaState(sagaId, command);
        stateRepository.save(state);
        
        try {
            // Step 1. Reserve Inventory (with timeout & retry)
            String reservationId = executeWithRetry(
                "reserveInventory",
                () -> inventoryService.reserve(command.getItems()),
                state,
                Duration.ofSeconds(30),
                3
            );
            
            // Step 2: Process Payment
            String paymentId = executeWithRetry(
                "processPayment",
                () -> paymentService.charge(command.getPaymentMethod(), command.getTotal()),
                state,
                Duration.ofSeconds(30),
                3
            );
            
            // Step 3: Schedule Shipping
            String shipmentId = executeWithRetry(
                "scheduleShipping",
                () -> shippingService.schedule(command.getShippingAddress()),
                state,
                Duration.ofSeconds(30),
                3
            );
            
            state.setStatus(SagaStatus.COMPLETED);
            stateRepository.save(state);
            
            return SagaResult.success(state);
            
        } catch (Exception e) {
            log.error("Saga {} failed: {}", sagaId, e.getMessage());
            compensate(state);
            
            state.setStatus(SagaStatus.COMPENSATED);
            state.setFailureReason(e.getMessage());
            stateRepository.save(state);
            
            return SagaResult.failure(e.getMessage());
        }
    }
    
    private <T> T executeWithRetry(
        String stepName,
        Supplier<T> action,
        SagaState state,
        Duration timeout,
        int maxAttempts
    ) throws Exception {
        state.addStep(stepName, StepStatus.STARTED);
        stateRepository.save(state);
        
        int attempt = 0;
        Exception lastException = null;
        
        while (attempt < maxAttempts) {
            try {
                // Execute with timeout
                CompletableFuture<T> future = CompletableFuture.supplyAsync(action);
                T result = future.get(timeout.toMillis(), TimeUnit.MILLISECONDS);
                
                // Success
                state.completeStep(stepName, result.toString());
                stateRepository.save(state);
                return result;
                
            } catch (TimeoutException e) {
                attempt++;
                lastException = new Exception("Timeout after " + timeout.toSeconds() + "s");
                log.warn("Step {} timed out (attempt {}/{})", stepName, attempt, maxAttempts);
                
            } catch (Exception e) {
                attempt++;
                lastException = e;
                log.warn("Step {} failed (attempt {}/{}): {}", 
                    stepName, attempt, maxAttempts, e.getMessage());
            }
            
            if (attempt < maxAttempts) {
                // Exponential backoff
                Thread.sleep((long) Math.pow(2, attempt) * 1000);
            }
        }
        
        // All retries exhausted
        state.failStep(stepName, lastException.getMessage());
        stateRepository.save(state);
        throw lastException;
    }
    
    private void compensate(SagaState state) {
        List<SagaStep> completedSteps = state.getCompletedSteps();
        Collections.reverse(completedSteps);
        
        for (SagaStep step : completedSteps) {
            try {
                log.info("Compensating step: {}", step.getName());
                
                switch (step.getName()) {
                    case "scheduleShipping":
                        shippingService.cancelShipment(step.getData());
                        break;
                    case "processPayment":
                        paymentService.refund(step.getData());
                        break;
                    case "reserveInventory":
                        inventoryService.releaseReservation(step.getData());
                        break;
                }
                
                step.setStatus(StepStatus.COMPENSATED);
                
            } catch (Exception e) {
                log.error("Compensation failed for step {}: {}", 
                    step.getName(), e.getMessage());
                step.setStatus(StepStatus.COMPENSATION_FAILED);
                // Trigger alert for manual intervention
                alertService.sendAlert(
                    AlertLevel.CRITICAL,
                    "Manual compensation required for saga " + state.getSagaId()
                );
            }
        }
        
        stateRepository.save(state);
    }
}

// Saga State Entity
@Entity
public class SagaState {
    @Id
    private String sagaId;
    
    @Enumerated(EnumType.STRING)
    private SagaStatus status;
    
    @OneToMany(cascade = CascadeType.ALL)
    private List<SagaStep> steps = new ArrayList<>();
    
    private String failureReason;
    
    @Column(columnDefinition = "TEXT")
    private String commandData;  // JSON del comando original
    
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    public void addStep(String name, StepStatus status) {
        steps.add(new SagaStep(name, status, LocalDateTime.now()));
        this.updatedAt = LocalDateTime.now();
    }
    
    public List<SagaStep> getCompletedSteps() {
        return steps.stream()
            .filter(s -> s.getStatus() == StepStatus.COMPLETED)
            .collect(Collectors.toList());
    }
}
```

**Monitoreo de Sagas**:
```java
@RestController
@RequestMapping("/admin/sagas")
public class SagaMonitoringController {
    
    private final SagaStateRepository repository;
    
    @GetMapping("/failed")
    public List<SagaState> getFailedSagas() {
        return repository.findByStatus(SagaStatus.COMPENSATED);
    }
    
    @GetMapping("/{sagaId}")
    public SagaDetailsDTO getSagaDetails(@PathVariable String sagaId) {
        SagaState state = repository.findById(sagaId)
            .orElseThrow(() -> new NotFoundException());
        
        return SagaDetailsDTO.builder()
            .sagaId(state.getSagaId())
            .status(state.getStatus())
            .steps(state.getSteps())
            .duration(Duration.between(state.getCreatedAt(), 
                state.getUpdatedAt()))
            .build();
    }
    
    @PostMapping("/{sagaId}/retry")
    public SagaResult retrySaga(@PathVariable String sagaId) {
        // Retry failed saga
        SagaState state = repository.findById(sagaId)
            .orElseThrow(() -> new NotFoundException());
        
        CreateOrderCommand originalCommand = 
            objectMapper.readValue(state.getCommandData(), CreateOrderCommand.class);
        
        return orchestrator.executeSaga(originalCommand);
    }
}
```

</details>

---

## ⭐⭐⭐⭐ Ejercicio 4: Consistency Boundary Design

Diseña boundaries de consistencia para un sistema de reservas de hotel.

**Requisitos**:
- Reserva de habitación (fuerte consistencia)
- Actualización de puntos de lealtad (eventual)
- Envío de confirmación por email (eventual)
- Cobro de tarjeta (fuerte consistencia)
- Actualización de estadísticas (eventual)

**Tareas**:
1. Definir aggregates y boundaries
2. Decidir qué necesita transacción ACID
3. Diseñar eventos para consistencia eventual
4. Implementar saga para proceso completo

<details>
<summary>💡 Solución</summary>

```
Consistency Boundaries:

┌─────────────────────────────────────┐
│   Reservation Aggregate             │ ← STRONG CONSISTENCY
│  ┌──────────────────────────────┐   │
│  │ Reservation                  │   │
│  │ - Room availability check    │   │
│  │ - Price calculation          │   │
│  │ - Payment processing         │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
                │
                │ ReservationCreatedEvent
                ▼
    ┌───────────┴───────────┬──────────────────┐
    │                       │                  │
┌───▼──────────┐   ┌───────▼──────┐   ┌──────▼──────┐
│ Loyalty      │   │ Notification │   │ Analytics   │
│ Service      │   │ Service      │   │ Service     │
│ (Eventual)   │   │ (Eventual)   │   │ (Eventual)  │
└──────────────┘   └──────────────┘   └─────────────┘
```

```typescript
// 1. Strong Consistency: Reservation Aggregate
export class Reservation {
    constructor(
        public readonly id: string,
        private roomId: string,
        private guestId: string,
        private checkIn: Date,
        private checkOut: Date,
        private payment: Payment,
        private status: ReservationStatus
    ) {}
    
    // Invariants enforced within aggregate
    static async create(
        command: CreateReservationCommand,
        roomAvailability: RoomAvailability,
        paymentGateway: PaymentGateway
    ): Promise<Reservation> {
        // Check availability (within transaction)
        if (!roomAvailability.isAvailable(command.roomId, command.checkIn, command.checkOut)) {
            throw new Error('Room not available');
        }
        
        // Process payment (within transaction)
        const payment = await paymentGateway.charge(
            command.paymentMethod,
            command.totalAmount
        );
        
        // Create reservation (atomic)
        const reservation = new Reservation(
            generateId(),
            command.roomId,
            command.guestId,
            command.checkIn,
            command.checkOut,
            payment,
            ReservationStatus.CONFIRMED
        );
        
        // Block room
        roomAvailability.block(command.roomId, command.checkIn, command.checkOut);
        
        return reservation;
    }
}

// Repository with transaction
export class ReservationRepository {
    async saveWithTransaction(
        reservation: Reservation,
        roomBlock: RoomBlock
    ): Promise<void> {
        await this.db.transaction(async (trx) => {
            // Save reservation
            await trx('reservations').insert({
                id: reservation.id,
                room_id: reservation.roomId,
                guest_id: reservation.guestId,
                check_in: reservation.checkIn,
                check_out: reservation.checkOut,
                payment_id: reservation.payment.id,
                status: reservation.status
            });
            
            // Block room (same transaction)
            await trx('room_blocks').insert({
                room_id: roomBlock.roomId,
                start_date: roomBlock.startDate,
                end_date: roomBlock.endDate,
                reservation_id: reservation.id
            });
            
            // Emit event (outbox pattern)
            await trx('event_outbox').insert({
                event_type: 'ReservationCreated',
                payload: JSON.stringify(reservation),
                created_at: new Date()
            });
        });
    }
}

// 2. Eventual Consistency: Event Handlers

// Loyalty Service
export class LoyaltyService {
    @EventHandler(ReservationCreatedEvent)
    async handleReservationCreated(event: ReservationCreatedEvent): Promise<void> {
        try {
            // Award loyalty points (idempotent)
            await this.awardPoints(
                event.guestId,
                this.calculatePoints(event.totalAmount),
                event.reservationId  // Deduplication key
            );
        } catch (error) {
            // Retry will be handled by message broker
            log.error('Failed to award loyalty points', error);
            throw error;  // Trigger retry
        }
    }
    
    private async awardPoints(
        guestId: string,
        points: number,
        idempotencyKey: string
    ): Promise<void> {
        // Check if already processed (idempotency)
        const existing = await this.db('loyalty_transactions')
            .where({idempotency_key: idempotencyKey})
            .first();
        
        if (existing) {
            log.info('Points already awarded for reservation', idempotencyKey);
            return;  // Already processed
        }
        
        await this.db.transaction(async (trx) => {
            // Update points balance
            await trx('guest_loyalty')
                .where({guest_id: guestId})
                .increment('points', points);
            
            // Record transaction
            await trx('loyalty_transactions').insert({
                id: generateId(),
                guest_id: guestId,
                points: points,
                reason: 'RESERVATION',
                idempotency_key: idempotencyKey,
                created_at: new Date()
            });
        });
    }
}

// Notification Service (eventual)
export class NotificationService {
    @EventHandler(ReservationCreatedEvent)
    async handleReservationCreated(event: ReservationCreatedEvent): Promise<void> {
        // Send confirmation email (idempotent via message deduplication)
        await this.emailService.send({
            to: event.guestEmail,
            template: 'reservation-confirmation',
            data: {
                reservationId: event.reservationId,
                checkIn: event.checkIn,
                checkOut: event.checkOut,
                roomType: event.roomType
            }
        });
    }
}

// Analytics Service (eventual)
export class AnalyticsService {
    @EventHandler(ReservationCreatedEvent)
    async handleReservationCreated(event: ReservationCreatedEvent): Promise<void> {
        // Update statistics (eventually consistent)
        await this.db('daily_stats')
            .where({date: event.createdAt.toDateString()})
            .increment('reservations_count', 1)
            .increment('revenue', event.totalAmount);
    }
}
```

</details>

---

## Proyecto Final: E-commerce con Database per Service

Implementa sistema completo con:
- Order Service (PostgreSQL)
- Product Service (MongoDB)
- User Service (MySQL)
- Inventory Service (Redis + PostgreSQL)
- Saga orchestrator para checkout
- Read models para queries complejas

**Rúbrica**:
- ⭐ Database isolation implementado
- ⭐⭐ Saga básica con compensación
- ⭐⭐⭐ Read models + consistencia eventual
- ⭐⭐⭐⭐ Saga orchestration + monitoring + chaos testing

## Recursos

- **Microservices Patterns** - Chris Richardson
- **Saga Pattern**: https://microservices.io/patterns/data/saga.html
- **Event Sourcing**: https://martinfowler.com/eaaDev/EventSourcing.html
