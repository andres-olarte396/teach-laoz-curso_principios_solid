# Saga Patterns: Transacciones Distribuidas con Eventos

## Introducción

En arquitecturas de microservicios, una transacción puede requerir actualizar datos en múltiples servicios. Sin embargo, **no existe ACID distribuido** sin complejidades significativas. Necesitamos un patrón que garantice consistencia eventual mientras permite rollback (compensación) en caso de fallos.

**Saga Pattern** es la solución: una secuencia de transacciones locales coordinadas mediante eventos o comandos, donde cada paso puede ser compensado si algo falla posteriormente.

---

## El Problema: Transacciones Distribuidas

### Escenario: Crear Pedido en E-Commerce

```typescript
// ❌ Esto NO funciona en microservicios distribuidos
async function createOrder(orderData: CreateOrderData): Promise<Order> {
    await db.beginTransaction();  // ❌ Solo funciona en una DB
    
    try {
        // Servicio 1: Orders
        const order = await orderService.create(orderData);
        
        // Servicio 2: Inventory (otra DB)
        await inventoryService.reserveStock(orderData.items);
        
        // Servicio 3: Payments (otra DB)
        await paymentService.charge(orderData.customerId, order.total);
        
        // Servicio 4: Shipping (otra DB)
        await shippingService.schedule(order.id);
        
        await db.commit();  // ❌ No puede hacer commit en 4 DBs distintas
        
        return order;
    } catch (error) {
        await db.rollback();  // ❌ No puede rollback servicios externos
        throw error;
    }
}
```

**Problemas**:
1. **No hay transacciones distribuidas**: Cada servicio tiene su propia base de datos
2. **Rollback imposible**: Si `paymentService` falla, ¿cómo deshaces cambios en `orderService` e `inventoryService`?
3. **Acoplamiento**: Servicio Order necesita conocer todos los demás
4. **Bloqueos**: Si uno falla, todos esperan

---

## Saga Pattern: La Solución

Una **Saga** es una secuencia de transacciones locales donde:
- Cada transacción actualiza datos en un solo servicio
- Cada transacción puede ser **compensada** si algo falla después

Hay dos implementaciones principales:

1. **Choreography**: Servicios reaccionan a eventos sin coordinador central
2. **Orchestration**: Un coordinador dirige toda la saga

---

## Choreography Saga

### Concepto

Los servicios se coordinan mediante eventos sin un coordinador central. Cada servicio:
- Escucha eventos
- Ejecuta su transacción local
- Publica nuevo evento
- Define compensación si algo falla

### Implementación: Pedido E-Commerce

```typescript
// ========== Events ==========

interface OrderCreatedEvent {
    orderId: string;
    customerId: string;
    items: OrderItem[];
    total: number;
    timestamp: Date;
}

interface StockReservedEvent {
    orderId: string;
    reservationId: string;
    items: OrderItem[];
    timestamp: Date;
}

interface PaymentChargedEvent {
    orderId: string;
    paymentId: string;
    amount: number;
    timestamp: Date;
}

interface ShippingScheduledEvent {
    orderId: string;
    trackingNumber: string;
    estimatedDelivery: Date;
}

// Compensation Events
interface StockReservationFailedEvent {
    orderId: string;
    reason: string;
}

interface PaymentFailedEvent {
    orderId: string;
    reason: string;
}

// ========== Service 1: Order Service ==========

class OrderService {
    async createOrder(command: CreateOrderCommand): Promise<void> {
        // Transacción local 1
        const order = await this.orderRepo.save({
            id: generateId(),
            customerId: command.customerId,
            items: command.items,
            status: 'PENDING',
            total: calculateTotal(command.items)
        });
        
        // Publica evento para siguiente paso
        await this.eventBus.publish({
            type: 'OrderCreated',
            orderId: order.id,
            customerId: order.customerId,
            items: order.items,
            total: order.total,
            timestamp: new Date()
        });
    }
    
    // Compensación: Cancelar orden si algo falla después
    @EventHandler('PaymentFailed')
    async onPaymentFailed(event: PaymentFailedEvent): Promise<void> {
        await this.orderRepo.update(event.orderId, {
            status: 'CANCELLED',
            cancelReason: event.reason
        });
        
        console.log(`Order ${event.orderId} cancelled due to payment failure`);
    }
}

// ========== Service 2: Inventory Service ==========

class InventoryService {
    @EventHandler('OrderCreated')
    async onOrderCreated(event: OrderCreatedEvent): Promise<void> {
        try {
            // Transacción local 2: Reservar stock
            const reservation = await this.reserveStock(event.items);
            
            // Publica evento de éxito
            await this.eventBus.publish({
                type: 'StockReserved',
                orderId: event.orderId,
                reservationId: reservation.id,
                items: event.items,
                timestamp: new Date()
            });
            
        } catch (error) {
            // Stock insuficiente → Publica evento de fallo
            await this.eventBus.publish({
                type: 'StockReservationFailed',
                orderId: event.orderId,
                reason: error.message,
                timestamp: new Date()
            });
        }
    }
    
    // Compensación: Liberar stock si payment falla
    @EventHandler('PaymentFailed')
    async onPaymentFailed(event: PaymentFailedEvent): Promise<void> {
        await this.releaseReservation(event.orderId);
        console.log(`Stock reservation released for order ${event.orderId}`);
    }
    
    private async reserveStock(items: OrderItem[]): Promise<Reservation> {
        // Verificar disponibilidad
        for (const item of items) {
            const stock = await this.stockRepo.findByProductId(item.productId);
            
            if (stock.available < item.quantity) {
                throw new Error(`Insufficient stock for ${item.productId}`);
            }
        }
        
        // Reservar (no decrementar aún)
        const reservation = await this.reservationRepo.save({
            id: generateId(),
            items: items,
            status: 'RESERVED',
            expiresAt: new Date(Date.now() + 15 * 60 * 1000) // 15 min
        });
        
        return reservation;
    }
}

// ========== Service 3: Payment Service ==========

class PaymentService {
    @EventHandler('StockReserved')
    async onStockReserved(event: StockReservedEvent): Promise<void> {
        try {
            // Transacción local 3: Procesar pago
            const payment = await this.processPayment(
                event.orderId,
                event.items
            );
            
            // Publica evento de éxito
            await this.eventBus.publish({
                type: 'PaymentCharged',
                orderId: event.orderId,
                paymentId: payment.id,
                amount: payment.amount,
                timestamp: new Date()
            });
            
        } catch (error) {
            // Pago falló → Publica evento de fallo
            await this.eventBus.publish({
                type: 'PaymentFailed',
                orderId: event.orderId,
                reason: error.message,
                timestamp: new Date()
            });
        }
    }
    
    // NO hay compensación aquí (el pago no se revierte automáticamente)
    // Requerir refund manual o proceso separado
}

// ========== Service 4: Shipping Service ==========

class ShippingService {
    @EventHandler('PaymentCharged')
    async onPaymentCharged(event: PaymentChargedEvent): Promise<void> {
        // Transacción local 4: Programar envío
        const shipping = await this.scheduleShipment(event.orderId);
        
        await this.eventBus.publish({
            type: 'ShippingScheduled',
            orderId: event.orderId,
            trackingNumber: shipping.trackingNumber,
            estimatedDelivery: shipping.estimatedDelivery,
            timestamp: new Date()
        });
        
        console.log(`Order ${event.orderId} completed successfully! 🎉`);
    }
}
```

### Flujo de Ejecución

**Caso Exitoso**:
```
1. User → CreateOrder
2. OrderService: Save order (PENDING) → Publish OrderCreated
3. InventoryService: Reserve stock → Publish StockReserved
4. PaymentService: Charge payment → Publish PaymentCharged
5. ShippingService: Schedule shipping → Publish ShippingScheduled
6. ✅ Order completed
```

**Caso con Fallo** (payment falla):
```
1. User → CreateOrder
2. OrderService: Save order (PENDING) → Publish OrderCreated
3. InventoryService: Reserve stock → Publish StockReserved
4. PaymentService: Charge FAILS → Publish PaymentFailed
5. InventoryService: onPaymentFailed → Release reservation
6. OrderService: onPaymentFailed → Cancel order (status = CANCELLED)
7. ❌ Order cancelled, state consistent
```

### SOLID en Choreography

```python
# ========== SRP: Cada handler una responsabilidad ==========

class OrderEventHandler:
    """Solo maneja eventos relacionados con Order"""
    
    def on_payment_failed(self, event: PaymentFailedEvent):
        # Solo cancela orden, nada más
        self.order_repo.update(event.order_id, {
            'status': 'CANCELLED',
            'cancel_reason': event.reason
        })

class InventoryEventHandler:
    """Solo maneja eventos relacionados con Inventory"""
    
    def on_payment_failed(self, event: PaymentFailedEvent):
        # Solo libera stock, nada más
        self.release_reservation(event.order_id)

# ========== OCP: Extensible sin modificar existente ==========

# Agregar nuevo step (NotificationService) sin cambiar servicios existentes
class NotificationService:
    """Nuevo servicio agregado SIN modificar Order/Inventory/Payment"""
    
    @event_handler('OrderCreated')
    def on_order_created(self, event: OrderCreatedEvent):
        self.send_email(event.customer_id, "Order created!")
    
    @event_handler('PaymentCharged')
    def on_payment_charged(self, event: PaymentChargedEvent):
        self.send_sms(event.customer_id, "Payment successful!")
    
    @event_handler('PaymentFailed')
    def on_payment_failed(self, event: PaymentFailedEvent):
        self.send_email(event.customer_id, "Payment failed, order cancelled")

# ✅ No modificamos OrderService, InventoryService, PaymentService
# ✅ Solo agregamos nuevo subscriber
```

### Ventajas de Choreography

✅ **Desacoplamiento**: Servicios no se conocen entre sí  
✅ **Simplicidad**: No hay coordinador central  
✅ **Escalabilidad**: Cada servicio escala independientemente  
✅ **Resiliencia**: Si un servicio cae, eventos en queue esperan  

### Desventajas de Choreography

❌ **Difícil de rastrear**: No hay vista única del flujo completo  
❌ **Dependencias cíclicas**: Eventos pueden generar loops  
❌ **Testing complejo**: Requiere toda la cadena de eventos  
❌ **No hay timeout global**: Saga puede quedar en estado intermedio indefinidamente  

---

## Orchestration Saga

### Concepto

Un **Saga Orchestrator** coordina toda la transacción:
- Conoce todos los pasos
- Envía comandos (no eventos)
- Maneja compensaciones
- Tiene timeout y retry logic

### Implementación

```typescript
// ========== Commands (no Events) ==========

interface ReserveStockCommand {
    orderId: string;
    items: OrderItem[];
}

interface ChargePaymentCommand {
    orderId: string;
    customerId: string;
    amount: number;
}

interface ScheduleShippingCommand {
    orderId: string;
    address: Address;
}

// ========== Saga Orchestrator ==========

class CreateOrderSagaOrchestrator {
    constructor(
        private orderService: OrderService,
        private inventoryService: InventoryService,
        private paymentService: PaymentService,
        private shippingService: ShippingService,
        private sagaRepo: SagaRepository
    ) {}
    
    async execute(command: CreateOrderCommand): Promise<SagaResult> {
        const sagaId = generateId();
        
        // Guardar estado de la saga
        await this.sagaRepo.save({
            id: sagaId,
            type: 'CreateOrder',
            status: 'STARTED',
            data: command,
            steps: [],
            startedAt: new Date()
        });
        
        try {
            // Step 1: Create Order
            const orderId = await this.executeStep(sagaId, 'CreateOrder', async () => {
                return await this.orderService.create(command);
            });
            
            // Step 2: Reserve Stock
            await this.executeStep(sagaId, 'ReserveStock', async () => {
                await this.inventoryService.reserveStock({
                    orderId: orderId,
                    items: command.items
                });
            });
            
            // Step 3: Charge Payment
            await this.executeStep(sagaId, 'ChargePayment', async () => {
                await this.paymentService.charge({
                    orderId: orderId,
                    customerId: command.customerId,
                    amount: command.total
                });
            });
            
            // Step 4: Schedule Shipping
            await this.executeStep(sagaId, 'ScheduleShipping', async () => {
                await this.shippingService.schedule({
                    orderId: orderId,
                    address: command.shippingAddress
                });
            });
            
            // ✅ Saga completada
            await this.sagaRepo.update(sagaId, {
                status: 'COMPLETED',
                completedAt: new Date()
            });
            
            return { success: true, orderId };
            
        } catch (error) {
            // ❌ Algo falló → Compensar
            await this.compensate(sagaId);
            
            return { success: false, error: error.message };
        }
    }
    
    private async executeStep(
        sagaId: string,
        stepName: string,
        action: () => Promise<any>
    ): Promise<any> {
        console.log(`[Saga ${sagaId}] Executing step: ${stepName}`);
        
        const stepStartTime = Date.now();
        
        try {
            const result = await action();
            
            // Guardar step exitoso
            await this.sagaRepo.addStep(sagaId, {
                name: stepName,
                status: 'COMPLETED',
                result: result,
                duration: Date.now() - stepStartTime
            });
            
            return result;
            
        } catch (error) {
            // Guardar step fallido
            await this.sagaRepo.addStep(sagaId, {
                name: stepName,
                status: 'FAILED',
                error: error.message,
                duration: Date.now() - stepStartTime
            });
            
            throw error;  // Propagar para compensar
        }
    }
    
    private async compensate(sagaId: string): Promise<void> {
        console.log(`[Saga ${sagaId}] Starting compensation...`);
        
        const saga = await this.sagaRepo.findById(sagaId);
        
        // Compensar steps en ORDEN INVERSO
        const completedSteps = saga.steps
            .filter(s => s.status === 'COMPLETED')
            .reverse();
        
        for (const step of completedSteps) {
            try {
                await this.compensateStep(step);
            } catch (error) {
                console.error(`Failed to compensate step ${step.name}:`, error);
                // Continue compensating other steps
            }
        }
        
        await this.sagaRepo.update(sagaId, {
            status: 'COMPENSATED',
            compensatedAt: new Date()
        });
    }
    
    private async compensateStep(step: SagaStep): Promise<void> {
        switch (step.name) {
            case 'CreateOrder':
                await this.orderService.cancel(step.result);
                break;
                
            case 'ReserveStock':
                await this.inventoryService.releaseReservation(step.result);
                break;
                
            case 'ChargePayment':
                // ⚠️ Payment compensation puede requerir refund manual
                await this.paymentService.initiateRefund(step.result);
                break;
                
            case 'ScheduleShipping':
                await this.shippingService.cancelShipment(step.result);
                break;
        }
        
        console.log(`[Compensation] ${step.name} compensated`);
    }
}
```

### Saga State Storage

```typescript
// Almacenar estado de la saga para recovery
interface SagaState {
    id: string;
    type: string;
    status: 'STARTED' | 'COMPLETED' | 'FAILED' | 'COMPENSATED';
    data: any;
    steps: SagaStep[];
    startedAt: Date;
    completedAt?: Date;
    compensatedAt?: Date;
}

interface SagaStep {
    name: string;
    status: 'COMPLETED' | 'FAILED';
    result?: any;
    error?: string;
    duration: number;
}

class PostgresSagaRepository {
    async save(saga: SagaState): Promise<void> {
        await this.db.query(
            'INSERT INTO sagas (id, type, status, data, steps, started_at) VALUES ($1, $2, $3, $4, $5, $6)',
            [saga.id, saga.type, saga.status, JSON.stringify(saga.data), JSON.stringify(saga.steps), saga.startedAt]
        );
    }
    
    async addStep(sagaId: string, step: SagaStep): Promise<void> {
        await this.db.query(
            'UPDATE sagas SET steps = steps || $1::jsonb WHERE id = $2',
            [JSON.stringify(step), sagaId]
        );
    }
    
    async findById(sagaId: string): Promise<SagaState> {
        const result = await this.db.query(
            'SELECT * FROM sagas WHERE id = $1',
            [sagaId]
        );
        return result.rows[0];
    }
}
```

### Timeout y Retry

```java
// ========== Timeout en Saga ==========

public class CreateOrderSagaOrchestrator {
    private static final Duration SAGA_TIMEOUT = Duration.ofMinutes(5);
    private static final int MAX_RETRIES = 3;
    
    public SagaResult execute(CreateOrderCommand command) {
        var sagaId = UUID.randomUUID().toString();
        var startTime = Instant.now();
        
        try {
            // Step 1 con retry
            var orderId = executeStepWithRetry(
                sagaId,
                "CreateOrder",
                () -> orderService.create(command),
                MAX_RETRIES
            );
            
            // Check timeout
            if (Duration.between(startTime, Instant.now()).compareTo(SAGA_TIMEOUT) > 0) {
                throw new SagaTimeoutException("Saga exceeded 5 minutes");
            }
            
            // Step 2 con retry
            executeStepWithRetry(
                sagaId,
                "ReserveStock",
                () -> inventoryService.reserveStock(orderId, command.items()),
                MAX_RETRIES
            );
            
            // ... más steps
            
            return SagaResult.success(orderId);
            
        } catch (SagaTimeoutException e) {
            compensate(sagaId);
            return SagaResult.timeout();
            
        } catch (Exception e) {
            compensate(sagaId);
            return SagaResult.failure(e.getMessage());
        }
    }
    
    private <T> T executeStepWithRetry(
        String sagaId,
        String stepName,
        Supplier<T> action,
        int maxRetries
    ) throws Exception {
        int attempt = 0;
        Exception lastException = null;
        
        while (attempt < maxRetries) {
            try {
                return action.get();
                
            } catch (TransientException e) {
                // Error temporal (network, timeout) → Retry
                attempt++;
                lastException = e;
                
                if (attempt < maxRetries) {
                    Thread.sleep(1000 * attempt);  // Exponential backoff
                    logger.info("[Saga {}] Retrying step {} (attempt {}/{})",
                        sagaId, stepName, attempt + 1, maxRetries);
                }
                
            } catch (BusinessException e) {
                // Error de negocio (stock insuficiente) → No retry
                throw e;
            }
        }
        
        throw new MaxRetriesExceededException(
            "Step " + stepName + " failed after " + maxRetries + " attempts",
            lastException
        );
    }
}
```

### SOLID en Orchestration

```kotlin
// ========== DIP: Orchestrator depende de abstracciones ==========

interface OrderService {
    suspend fun create(command: CreateOrderCommand): String
    suspend fun cancel(orderId: String)
}

interface InventoryService {
    suspend fun reserveStock(command: ReserveStockCommand): String
    suspend fun releaseReservation(reservationId: String)
}

interface PaymentService {
    suspend fun charge(command: ChargePaymentCommand): String
    suspend fun initiateRefund(paymentId: String)
}

// Orchestrator depende de interfaces, no implementaciones
class CreateOrderSagaOrchestrator(
    private val orderService: OrderService,        // ✅ Interface
    private val inventoryService: InventoryService, // ✅ Interface
    private val paymentService: PaymentService,     // ✅ Interface
    private val sagaRepo: SagaRepository            // ✅ Interface
) {
    // ...
}

// ========== OCP: Extensible agregando nuevos steps ==========

// Podemos extender la saga sin modificar código existente
class ExtendedCreateOrderSagaOrchestrator(
    orderService: OrderService,
    inventoryService: InventoryService,
    paymentService: PaymentService,
    sagaRepo: SagaRepository,
    private val loyaltyService: LoyaltyService  // ✅ Nuevo servicio
) : CreateOrderSagaOrchestrator(orderService, inventoryService, paymentService, sagaRepo) {
    
    override suspend fun execute(command: CreateOrderCommand): SagaResult {
        val result = super.execute(command)  // Ejecuta saga original
        
        if (result.success) {
            // ✅ Agregar step adicional sin modificar original
            loyaltyService.addPoints(command.customerId, command.total)
        }
        
        return result
    }
}
```

### Ventajas de Orchestration

✅ **Vista centralizada**: Flujo completo en un lugar  
✅ **Fácil debugging**: Logs y estado en orchestrator  
✅ **Timeout global**: Control preciso de tiempo  
✅ **Retry logic**: Reintentos automáticos con backoff  
✅ **Testing simple**: Mock servicios, test orchestrator  

### Desventajas de Orchestration

❌ **Acoplamiento**: Orchestrator conoce todos los servicios  
❌ **Single point of failure**: Si orchestrator cae, sagas se bloquean  
❌ **Menos flexible**: Agregar steps requiere modificar orchestrator  

---

## Comparación: Choreography vs Orchestration

| Aspecto | Choreography | Orchestration |
|---------|--------------|---------------|
| **Coordinación** | Descentralizada (eventos) | Centralizada (orchestrator) |
| **Acoplamiento** | Bajo (servicios no se conocen) | Alto (orchestrator conoce todos) |
| **Visibilidad** | Difícil rastrear flujo completo | Fácil ver flujo en orchestrator |
| **Testing** | Complejo (requiere eventos reales) | Simple (mock servicios) |
| **Timeout** | No hay global | Fácil implementar |
| **Retry** | Difícil coordinar | Fácil en orchestrator |
| **Escalabilidad** | Alta (sin bottleneck) | Media (orchestrator puede ser bottleneck) |
| **Flexibilidad** | Alta (agregar listeners) | Media (modificar orchestrator) |
| **Debugging** | Difícil (eventos distribuidos) | Fácil (logs centralizados) |
| **Mejor para** | Workflows simples, eventos domain | Workflows complejos, transacciones críticas |

---

## Cuándo Usar Cada Patrón

### Usa Choreography si:
✅ Workflow es simple (2-4 steps)  
✅ Servicios son independientes  
✅ Tolerancia a eventual consistency alta  
✅ No hay requisitos estrictos de timeout  
✅ Equipo prefiere desacoplamiento  

**Ejemplo**: Notificaciones, analytics, logs distribuidos

### Usa Orchestration si:
✅ Workflow es complejo (5+ steps)  
✅ Requieres timeout y retry preciso  
✅ Necesitas visibilidad completa del flujo  
✅ Transacción es crítica (pagos, finanzas)  
✅ Debugging y monitoring son prioritarios  

**Ejemplo**: Pedidos e-commerce, reservas, procesos bancarios

---

## Compensación: Patrones Avanzados

### Forward Recovery

En lugar de rollback, **completar la saga de forma alternativa**.

```python
class OrderSagaWithForwardRecovery:
    async def execute(self, command: CreateOrderCommand):
        try:
            # Step 1: Reserve stock
            await self.inventory_service.reserve_stock(command.items)
            
        except InsufficientStockException as e:
            # ✅ Forward recovery: Permitir backorder
            await self.inventory_service.create_backorder(command.items)
            
            await self.notification_service.notify_customer(
                command.customer_id,
                "Some items backordered, estimated delivery: +7 days"
            )
            
            # Continuar saga con backorder
            return await self.continue_with_backorder(command)
```

### Semantic Lock

Prevenir conflictos usando "locks" semánticos.

```java
class InventoryService {
    public ReservationResult reserveStock(List<OrderItem> items) {
        for (OrderItem item : items) {
            // Semantic lock: Marcar como "reserved" sin decrementar
            var updated = stockRepo.updateIf(
                item.productId(),
                stock -> stock.available >= item.quantity(),
                stock -> stock.withReserved(stock.reserved + item.quantity())
            );
            
            if (!updated) {
                // Rollback reservations previas
                rollbackReservations(items.subList(0, items.indexOf(item)));
                throw new InsufficientStockException();
            }
        }
        
        return ReservationResult.success();
    }
    
    // Cuando payment OK → Convert reservation to committed
    public void commitReservation(String reservationId) {
        var reservation = reservationRepo.findById(reservationId);
        
        for (OrderItem item : reservation.items()) {
            stockRepo.update(
                item.productId(),
                stock -> stock
                    .withReserved(stock.reserved - item.quantity())
                    .withAvailable(stock.available - item.quantity())
            );
        }
    }
}
```

---

## Resumen

| Concepto | Descripción |
|----------|-------------|
| **Saga Pattern** | Transacción distribuida como secuencia de transacciones locales |
| **Choreography** | Servicios coordinados por eventos, sin coordinador central |
| **Orchestration** | Coordinador central dirige toda la saga |
| **Compensación** | Rollback mediante transacciones inversas |
| **Forward Recovery** | Completar saga de forma alternativa en lugar de rollback |
| **Semantic Lock** | Reservar recursos sin commitear hasta confirmar toda la saga |

**Próximo tema**: Implementaremos Saga Patterns con frameworks reales (Axon, Temporal, Camunda) y exploraremos messaging technologies (Kafka, RabbitMQ).
