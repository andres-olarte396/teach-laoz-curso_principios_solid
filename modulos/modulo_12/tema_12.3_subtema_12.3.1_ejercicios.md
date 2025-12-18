# Ejercicios: Saga Patterns

## Ejercicio 1: Identificar Problemas ⭐

### Problema

Este código intenta implementar una transacción distribuida de forma incorrecta:

```typescript
class BookingService {
    async bookTrip(
        userId: string,
        flightId: string,
        hotelId: string,
        carId: string
    ): Promise<Booking> {
        // ❌ Intento de transacción distribuida
        const booking = await this.db.transaction(async (trx) => {
            // Servicio 1: Flight
            const flight = await this.flightService.book(flightId, userId);
            
            // Servicio 2: Hotel (otra DB)
            const hotel = await this.hotelService.book(hotelId, userId);
            
            // Servicio 3: Car (otra DB)
            const car = await this.carService.book(carId, userId);
            
            return {
                flightBooking: flight,
                hotelBooking: hotel,
                carBooking: car,
                status: 'CONFIRMED'
            };
        });
        
        return booking;
    }
}
```

### Tareas

1. **Identifica 3 problemas** con este código
2. **Explica** qué sucede si `hotelService.book()` falla después de que `flightService.book()` tuvo éxito
3. **Propón** si usarías Choreography o Orchestration para este caso (justifica)

<details>
<summary>💡 Solución</summary>

### Problemas Identificados

**1. No hay transacción distribuida real**
```typescript
// La transacción DB solo cubre la DB local de BookingService
// No puede hacer rollback de flightService, hotelService, carService
await this.db.transaction(async (trx) => {
    // ❌ Cada servicio usa su propia DB
    await this.flightService.book(flightId, userId);  // DB1
    await this.hotelService.book(hotelId, userId);    // DB2
    await this.carService.book(carId, userId);        // DB3
});
```

**2. Sin compensación**
```typescript
// Si hotelService falla, NO hay rollback de flightService
const flight = await this.flightService.book(flightId, userId);  // ✅ OK
const hotel = await this.hotelService.book(hotelId, userId);     // ❌ FAIL

// flight queda reservado permanentemente, datos inconsistentes
```

**3. Alto acoplamiento**
```typescript
// BookingService conoce y depende directamente de todos los servicios
class BookingService {
    constructor(
        private flightService: FlightService,   // ❌ Acoplado
        private hotelService: HotelService,     // ❌ Acoplado
        private carService: CarService          // ❌ Acoplado
    ) {}
}
```

### Qué Sucede si Hotel Falla

```
Timeline:
1. flightService.book(flightId) → ✅ Success (flight reservado en DB1)
2. hotelService.book(hotelId) → ❌ Fail (hotel no disponible)
3. Transaction rollback → ❌ Solo rollback de DB local, NO de flight
4. Usuario ve error, pero flight SIGUE reservado
5. ⚠️ Estado inconsistente: flight reservado sin booking completo
```

### Solución: Usar Orchestration Saga

**¿Por qué Orchestration?**
- Workflow complejo (3 servicios, orden específico)
- Requiere compensación precisa (cancelar reservas en orden inverso)
- Timeout crítico (no dejar reservas indefinidas)
- Debugging importante (viajes son transacciones de alto valor)

```typescript
class BookTripSagaOrchestrator {
    async execute(command: BookTripCommand): Promise<SagaResult> {
        const sagaId = generateId();
        
        try {
            // Step 1: Book flight
            const flightBooking = await this.executeStep(sagaId, 'BookFlight', 
                () => this.flightService.book(command.flightId, command.userId)
            );
            
            // Step 2: Book hotel
            const hotelBooking = await this.executeStep(sagaId, 'BookHotel',
                () => this.hotelService.book(command.hotelId, command.userId)
            );
            
            // Step 3: Book car
            const carBooking = await this.executeStep(sagaId, 'BookCar',
                () => this.carService.book(command.carId, command.userId)
            );
            
            return SagaResult.success({ flightBooking, hotelBooking, carBooking });
            
        } catch (error) {
            // Compensate en orden inverso
            await this.compensate(sagaId);
            return SagaResult.failure(error.message);
        }
    }
    
    private async compensate(sagaId: string): Promise<void> {
        const saga = await this.sagaRepo.findById(sagaId);
        const completedSteps = saga.steps.filter(s => s.status === 'COMPLETED').reverse();
        
        for (const step of completedSteps) {
            switch (step.name) {
                case 'BookFlight':
                    await this.flightService.cancel(step.result.bookingId);
                    break;
                case 'BookHotel':
                    await this.hotelService.cancel(step.result.bookingId);
                    break;
                case 'BookCar':
                    await this.carService.cancel(step.result.bookingId);
                    break;
            }
        }
    }
}
```

</details>

---

## Ejercicio 2: Implementar Choreography Saga ⭐⭐

### Escenario

Sistema de delivery de comida con 3 servicios:

1. **OrderService**: Crea orden
2. **RestaurantService**: Acepta/rechaza orden
3. **DeliveryService**: Asigna repartidor

### Requisitos

Implementa Choreography Saga con:

**Events**:
- `OrderPlacedEvent`
- `OrderAcceptedEvent` / `OrderRejectedEvent`
- `DriverAssignedEvent` / `NoDriverAvailableEvent`

**Compensaciones**:
- Si restaurant rechaza → Cancelar orden
- Si no hay driver disponible → Rechazar orden en restaurant, cancelar orden

### Implementación

```typescript
// ========== Events ==========

interface OrderPlacedEvent {
    orderId: string;
    restaurantId: string;
    customerId: string;
    items: OrderItem[];
    deliveryAddress: string;
    timestamp: Date;
}

interface OrderAcceptedEvent {
    orderId: string;
    restaurantId: string;
    estimatedPrepTime: number; // minutos
    timestamp: Date;
}

interface OrderRejectedEvent {
    orderId: string;
    restaurantId: string;
    reason: string;
    timestamp: Date;
}

interface DriverAssignedEvent {
    orderId: string;
    driverId: string;
    estimatedPickupTime: Date;
    timestamp: Date;
}

interface NoDriverAvailableEvent {
    orderId: string;
    reason: string;
    timestamp: Date;
}

// ========== IMPLEMENTA ESTOS SERVICIOS ==========

class OrderService {
    async placeOrder(command: PlaceOrderCommand): Promise<void> {
        // TODO: Implementar
        // 1. Guardar orden con status 'PENDING'
        // 2. Publicar OrderPlacedEvent
    }
    
    @EventHandler('OrderRejected')
    async onOrderRejected(event: OrderRejectedEvent): Promise<void> {
        // TODO: Implementar compensación
        // 1. Update order status to 'CANCELLED'
        // 2. Refund customer (if payment already charged)
    }
    
    @EventHandler('NoDriverAvailable')
    async onNoDriverAvailable(event: NoDriverAvailableEvent): Promise<void> {
        // TODO: Implementar compensación
        // 1. Cancel order
        // 2. Notify customer
    }
}

class RestaurantService {
    @EventHandler('OrderPlaced')
    async onOrderPlaced(event: OrderPlacedEvent): Promise<void> {
        // TODO: Implementar
        // 1. Check if restaurant can accept (opening hours, capacity)
        // 2. If yes → Publish OrderAcceptedEvent
        // 3. If no → Publish OrderRejectedEvent
    }
    
    @EventHandler('NoDriverAvailable')
    async onNoDriverAvailable(event: NoDriverAvailableEvent): Promise<void> {
        // TODO: Implementar compensación
        // 1. Mark order as rejected in restaurant system
        // 2. Free kitchen capacity
    }
}

class DeliveryService {
    @EventHandler('OrderAccepted')
    async onOrderAccepted(event: OrderAcceptedEvent): Promise<void> {
        // TODO: Implementar
        // 1. Find available driver near restaurant
        // 2. If found → Assign and publish DriverAssignedEvent
        // 3. If not found → Publish NoDriverAvailableEvent
    }
}
```

### Tests a Implementar

```typescript
describe('Food Delivery Choreography Saga', () => {
    it('should complete order when all steps succeed', async () => {
        // TODO: Implementar test
        // 1. Place order
        // 2. Verify OrderPlaced event published
        // 3. Simulate restaurant accepts
        // 4. Verify OrderAccepted event published
        // 5. Simulate driver assigned
        // 6. Verify order status = 'CONFIRMED'
    });
    
    it('should cancel order when restaurant rejects', async () => {
        // TODO: Implementar test
        // 1. Place order
        // 2. Simulate restaurant rejects (closed)
        // 3. Verify order status = 'CANCELLED'
        // 4. Verify customer notified
    });
    
    it('should compensate when no driver available', async () => {
        // TODO: Implementar test
        // 1. Place order
        // 2. Restaurant accepts
        // 3. No driver available
        // 4. Verify restaurant compensated
        // 5. Verify order cancelled
    });
});
```

<details>
<summary>💡 Solución</summary>

```typescript
// ========== Order Service ==========

class OrderService {
    constructor(
        private orderRepo: OrderRepository,
        private eventBus: EventBus,
        private paymentService: PaymentService
    ) {}
    
    async placeOrder(command: PlaceOrderCommand): Promise<string> {
        const orderId = generateId();
        
        // 1. Save order
        await this.orderRepo.save({
            id: orderId,
            restaurantId: command.restaurantId,
            customerId: command.customerId,
            items: command.items,
            deliveryAddress: command.deliveryAddress,
            status: 'PENDING',
            createdAt: new Date()
        });
        
        // 2. Publish event
        await this.eventBus.publish({
            type: 'OrderPlaced',
            orderId: orderId,
            restaurantId: command.restaurantId,
            customerId: command.customerId,
            items: command.items,
            deliveryAddress: command.deliveryAddress,
            timestamp: new Date()
        });
        
        return orderId;
    }
    
    @EventHandler('OrderRejected')
    async onOrderRejected(event: OrderRejectedEvent): Promise<void> {
        await this.orderRepo.update(event.orderId, {
            status: 'CANCELLED',
            cancelReason: event.reason,
            cancelledAt: new Date()
        });
        
        // Refund if payment was charged
        const order = await this.orderRepo.findById(event.orderId);
        if (order.paymentStatus === 'CHARGED') {
            await this.paymentService.refund(order.paymentId);
        }
        
        console.log(`Order ${event.orderId} cancelled: ${event.reason}`);
    }
    
    @EventHandler('NoDriverAvailable')
    async onNoDriverAvailable(event: NoDriverAvailableEvent): Promise<void> {
        await this.orderRepo.update(event.orderId, {
            status: 'CANCELLED',
            cancelReason: 'No driver available',
            cancelledAt: new Date()
        });
        
        console.log(`Order ${event.orderId} cancelled: No drivers`);
    }
}

// ========== Restaurant Service ==========

class RestaurantService {
    constructor(
        private restaurantRepo: RestaurantRepository,
        private eventBus: EventBus
    ) {}
    
    @EventHandler('OrderPlaced')
    async onOrderPlaced(event: OrderPlacedEvent): Promise<void> {
        const restaurant = await this.restaurantRepo.findById(event.restaurantId);
        
        // Check if can accept
        const canAccept = await this.canAcceptOrder(restaurant, event);
        
        if (canAccept) {
            // Accept order
            const estimatedPrepTime = this.calculatePrepTime(event.items);
            
            await this.restaurantRepo.addOrder(event.restaurantId, {
                orderId: event.orderId,
                items: event.items,
                status: 'ACCEPTED',
                estimatedPrepTime: estimatedPrepTime
            });
            
            await this.eventBus.publish({
                type: 'OrderAccepted',
                orderId: event.orderId,
                restaurantId: event.restaurantId,
                estimatedPrepTime: estimatedPrepTime,
                timestamp: new Date()
            });
            
        } else {
            // Reject order
            const reason = this.getRejectionReason(restaurant);
            
            await this.eventBus.publish({
                type: 'OrderRejected',
                orderId: event.orderId,
                restaurantId: event.restaurantId,
                reason: reason,
                timestamp: new Date()
            });
        }
    }
    
    @EventHandler('NoDriverAvailable')
    async onNoDriverAvailable(event: NoDriverAvailableEvent): Promise<void> {
        // Free kitchen capacity
        await this.restaurantRepo.updateOrderStatus(
            event.orderId,
            'CANCELLED'
        );
        
        console.log(`Restaurant: Order ${event.orderId} cancelled (no driver)`);
    }
    
    private async canAcceptOrder(
        restaurant: Restaurant,
        event: OrderPlacedEvent
    ): Promise<boolean> {
        // Check opening hours
        if (!restaurant.isOpen()) {
            return false;
        }
        
        // Check capacity
        const currentOrders = await this.restaurantRepo.getActiveOrders(restaurant.id);
        if (currentOrders.length >= restaurant.maxConcurrentOrders) {
            return false;
        }
        
        // Check if all items available
        for (const item of event.items) {
            const available = await this.restaurantRepo.isItemAvailable(
                restaurant.id,
                item.menuItemId
            );
            if (!available) {
                return false;
            }
        }
        
        return true;
    }
    
    private getRejectionReason(restaurant: Restaurant): string {
        if (!restaurant.isOpen()) {
            return 'Restaurant closed';
        }
        return 'Restaurant at capacity';
    }
    
    private calculatePrepTime(items: OrderItem[]): number {
        // Simple calculation: 5 min per item
        return items.length * 5;
    }
}

// ========== Delivery Service ==========

class DeliveryService {
    constructor(
        private driverRepo: DriverRepository,
        private eventBus: EventBus,
        private geocodingService: GeocodingService
    ) {}
    
    @EventHandler('OrderAccepted')
    async onOrderAccepted(event: OrderAcceptedEvent): Promise<void> {
        // Find restaurant location
        const restaurantLocation = await this.geocodingService.getLocation(
            event.restaurantId
        );
        
        // Find available driver nearby (within 5km)
        const driver = await this.driverRepo.findAvailableNearby(
            restaurantLocation,
            5000  // 5km radius
        );
        
        if (driver) {
            // Assign driver
            await this.driverRepo.assignOrder(driver.id, event.orderId);
            
            const estimatedPickupTime = new Date(
                Date.now() + event.estimatedPrepTime * 60 * 1000
            );
            
            await this.eventBus.publish({
                type: 'DriverAssigned',
                orderId: event.orderId,
                driverId: driver.id,
                estimatedPickupTime: estimatedPickupTime,
                timestamp: new Date()
            });
            
        } else {
            // No driver available
            await this.eventBus.publish({
                type: 'NoDriverAvailable',
                orderId: event.orderId,
                reason: 'No drivers nearby',
                timestamp: new Date()
            });
        }
    }
}

// ========== Tests ==========

describe('Food Delivery Choreography Saga', () => {
    let orderService: OrderService;
    let restaurantService: RestaurantService;
    let deliveryService: DeliveryService;
    let eventBus: InMemoryEventBus;
    
    beforeEach(() => {
        eventBus = new InMemoryEventBus();
        orderService = new OrderService(new InMemoryOrderRepo(), eventBus, mockPaymentService);
        restaurantService = new RestaurantService(new InMemoryRestaurantRepo(), eventBus);
        deliveryService = new DeliveryService(new InMemoryDriverRepo(), eventBus, mockGeocodingService);
        
        // Register event handlers
        eventBus.subscribe('OrderPlaced', (e) => restaurantService.onOrderPlaced(e));
        eventBus.subscribe('OrderAccepted', (e) => deliveryService.onOrderAccepted(e));
        eventBus.subscribe('OrderRejected', (e) => orderService.onOrderRejected(e));
        eventBus.subscribe('NoDriverAvailable', (e) => orderService.onNoDriverAvailable(e));
        eventBus.subscribe('NoDriverAvailable', (e) => restaurantService.onNoDriverAvailable(e));
    });
    
    it('should complete order when all steps succeed', async () => {
        // Setup: Restaurant open, driver available
        await restaurantRepo.save({ id: 'r1', isOpen: true, maxConcurrentOrders: 10 });
        await driverRepo.save({ id: 'd1', status: 'AVAILABLE', location: { lat: 0, lng: 0 } });
        
        // Place order
        const orderId = await orderService.placeOrder({
            restaurantId: 'r1',
            customerId: 'c1',
            items: [{ menuItemId: 'item1', quantity: 2 }],
            deliveryAddress: '123 Main St'
        });
        
        // Wait for all events to process
        await eventBus.waitForProcessing();
        
        // Verify order confirmed
        const order = await orderRepo.findById(orderId);
        expect(order.status).toBe('CONFIRMED');
        
        // Verify events published
        const events = eventBus.getPublishedEvents();
        expect(events).toContainEqual(expect.objectContaining({ type: 'OrderPlaced' }));
        expect(events).toContainEqual(expect.objectContaining({ type: 'OrderAccepted' }));
        expect(events).toContainEqual(expect.objectContaining({ type: 'DriverAssigned' }));
    });
    
    it('should cancel order when restaurant rejects', async () => {
        // Setup: Restaurant closed
        await restaurantRepo.save({ id: 'r1', isOpen: false });
        
        const orderId = await orderService.placeOrder({
            restaurantId: 'r1',
            customerId: 'c1',
            items: [{ menuItemId: 'item1', quantity: 2 }],
            deliveryAddress: '123 Main St'
        });
        
        await eventBus.waitForProcessing();
        
        const order = await orderRepo.findById(orderId);
        expect(order.status).toBe('CANCELLED');
        expect(order.cancelReason).toBe('Restaurant closed');
        
        const events = eventBus.getPublishedEvents();
        expect(events).toContainEqual(expect.objectContaining({
            type: 'OrderRejected',
            reason: 'Restaurant closed'
        }));
    });
    
    it('should compensate when no driver available', async () => {
        // Setup: Restaurant open, NO drivers available
        await restaurantRepo.save({ id: 'r1', isOpen: true, maxConcurrentOrders: 10 });
        // No drivers in repo
        
        const orderId = await orderService.placeOrder({
            restaurantId: 'r1',
            customerId: 'c1',
            items: [{ menuItemId: 'item1', quantity: 2 }],
            deliveryAddress: '123 Main St'
        });
        
        await eventBus.waitForProcessing();
        
        // Verify order cancelled
        const order = await orderRepo.findById(orderId);
        expect(order.status).toBe('CANCELLED');
        expect(order.cancelReason).toBe('No driver available');
        
        // Verify restaurant compensated
        const restaurantOrder = await restaurantRepo.findOrder(orderId);
        expect(restaurantOrder.status).toBe('CANCELLED');
        
        // Verify events
        const events = eventBus.getPublishedEvents();
        expect(events).toContainEqual(expect.objectContaining({ type: 'OrderAccepted' }));
        expect(events).toContainEqual(expect.objectContaining({ type: 'NoDriverAvailable' }));
    });
});
```

</details>

---

## Ejercicio 3: Implementar Orchestration Saga ⭐⭐⭐

### Escenario

Sistema bancario para transferencias entre cuentas que pueden estar en diferentes bancos.

**Servicios**:
1. **AccountService**: Valida cuentas
2. **DebitService**: Debita cuenta origen
3. **CreditService**: Acredita cuenta destino
4. **NotificationService**: Notifica a usuarios
5. **AuditService**: Registra operación

### Requisitos

Implementa Orchestration Saga con:

**Features**:
- Timeout de 30 segundos
- Retry automático (máx 3 intentos) para errores transitorios
- Compensación en orden inverso si falla
- Idempotencia (misma transferencia no se procesa 2 veces)

**Casos de Prueba**:
1. Transferencia exitosa
2. Cuenta destino inválida (compensar débito)
3. Timeout en credit (compensar débito)
4. Retry exitoso después de fallo temporal

### Esqueleto

```typescript
// ========== Commands ==========

interface TransferMoneyCommand {
    transferId: string;  // Idempotency key
    fromAccountId: string;
    toAccountId: string;
    amount: number;
    currency: string;
}

// ========== Saga Orchestrator ==========

class TransferMoneySagaOrchestrator {
    private static readonly SAGA_TIMEOUT = 30_000; // 30 seconds
    private static readonly MAX_RETRIES = 3;
    
    constructor(
        private accountService: AccountService,
        private debitService: DebitService,
        private creditService: CreditService,
        private notificationService: NotificationService,
        private auditService: AuditService,
        private sagaRepo: SagaRepository
    ) {}
    
    async execute(command: TransferMoneyCommand): Promise<SagaResult> {
        // TODO: Implementar
        // 1. Check idempotency (si ya se procesó, retornar resultado previo)
        // 2. Guardar saga state
        // 3. Ejecutar steps con timeout
        // 4. Si falla, compensar en orden inverso
        // 5. Guardar resultado final
    }
    
    private async executeStepWithRetry<T>(
        sagaId: string,
        stepName: string,
        action: () => Promise<T>
    ): Promise<T> {
        // TODO: Implementar retry logic
        // 1. Intentar ejecutar action
        // 2. Si falla con error transitorio (network timeout), retry con backoff
        // 3. Si falla con error de negocio (invalid account), no retry
        // 4. Max 3 intentos
    }
    
    private async compensate(sagaId: string): Promise<void> {
        // TODO: Implementar compensación
        // 1. Obtener steps completados
        // 2. Compensar en orden INVERSO
        // 3. Manejar fallos en compensación (log pero continuar)
    }
}

// ========== Services (stubs para implementar) ==========

interface AccountService {
    validate(accountId: string): Promise<Account>;
}

interface DebitService {
    debit(accountId: string, amount: number, transferId: string): Promise<DebitResult>;
    refund(accountId: string, debitId: string): Promise<void>;
}

interface CreditService {
    credit(accountId: string, amount: number, transferId: string): Promise<CreditResult>;
}
```

<details>
<summary>💡 Solución Completa (~900 líneas)</summary>

```typescript
// ========== Errors ==========

class TransientError extends Error {
    constructor(message: string) {
        super(message);
        this.name = 'TransientError';
    }
}

class BusinessError extends Error {
    constructor(message: string) {
        super(message);
        this.name = 'BusinessError';
    }
}

class SagaTimeoutError extends Error {
    constructor(message: string) {
        super(message);
        this.name = 'SagaTimeoutError';
    }
}

// ========== Domain Models ==========

interface Account {
    id: string;
    balance: number;
    currency: string;
    status: 'ACTIVE' | 'FROZEN' | 'CLOSED';
}

interface DebitResult {
    debitId: string;
    accountId: string;
    amount: number;
    balanceAfter: number;
}

interface CreditResult {
    creditId: string;
    accountId: string;
    amount: number;
    balanceAfter: number;
}

interface SagaResult {
    success: boolean;
    transferId?: string;
    error?: string;
}

interface SagaState {
    id: string;
    transferId: string;
    type: 'TransferMoney';
    status: 'STARTED' | 'COMPLETED' | 'FAILED' | 'COMPENSATED' | 'TIMEOUT';
    command: TransferMoneyCommand;
    steps: SagaStep[];
    startedAt: Date;
    completedAt?: Date;
}

interface SagaStep {
    name: string;
    status: 'COMPLETED' | 'FAILED';
    result?: any;
    error?: string;
    attempts: number;
    duration: number;
}

// ========== Saga Orchestrator ==========

class TransferMoneySagaOrchestrator {
    private static readonly SAGA_TIMEOUT = 30_000;
    private static readonly MAX_RETRIES = 3;
    private static readonly RETRY_DELAYS = [1000, 2000, 4000]; // Exponential backoff
    
    constructor(
        private accountService: AccountService,
        private debitService: DebitService,
        private creditService: CreditService,
        private notificationService: NotificationService,
        private auditService: AuditService,
        private sagaRepo: SagaRepository
    ) {}
    
    async execute(command: TransferMoneyCommand): Promise<SagaResult> {
        // 1. Check idempotency
        const existingSaga = await this.sagaRepo.findByTransferId(command.transferId);
        if (existingSaga) {
            if (existingSaga.status === 'COMPLETED') {
                return { success: true, transferId: command.transferId };
            } else if (existingSaga.status === 'FAILED' || existingSaga.status === 'COMPENSATED') {
                return { success: false, error: 'Transfer already failed' };
            }
            // If STARTED → Continue (possible retry after crash)
        }
        
        const sagaId = generateId();
        const startTime = Date.now();
        
        // 2. Save saga state
        await this.sagaRepo.save({
            id: sagaId,
            transferId: command.transferId,
            type: 'TransferMoney',
            status: 'STARTED',
            command: command,
            steps: [],
            startedAt: new Date()
        });
        
        try {
            // 3. Execute steps with timeout
            await this.executeWithTimeout(async () => {
                // Step 1: Validate from account
                const fromAccount = await this.executeStepWithRetry(
                    sagaId,
                    'ValidateFromAccount',
                    () => this.accountService.validate(command.fromAccountId)
                );
                
                if (fromAccount.status !== 'ACTIVE') {
                    throw new BusinessError(`Account ${command.fromAccountId} is ${fromAccount.status}`);
                }
                
                if (fromAccount.balance < command.amount) {
                    throw new BusinessError('Insufficient funds');
                }
                
                if (fromAccount.currency !== command.currency) {
                    throw new BusinessError('Currency mismatch');
                }
                
                // Step 2: Validate to account
                const toAccount = await this.executeStepWithRetry(
                    sagaId,
                    'ValidateToAccount',
                    () => this.accountService.validate(command.toAccountId)
                );
                
                if (toAccount.status !== 'ACTIVE') {
                    throw new BusinessError(`Account ${command.toAccountId} is ${toAccount.status}`);
                }
                
                // Step 3: Debit from account
                const debitResult = await this.executeStepWithRetry(
                    sagaId,
                    'DebitAccount',
                    () => this.debitService.debit(
                        command.fromAccountId,
                        command.amount,
                        command.transferId
                    )
                );
                
                // Step 4: Credit to account
                const creditResult = await this.executeStepWithRetry(
                    sagaId,
                    'CreditAccount',
                    () => this.creditService.credit(
                        command.toAccountId,
                        command.amount,
                        command.transferId
                    )
                );
                
                // Step 5: Notify users (non-critical, no compensation needed)
                try {
                    await this.notificationService.notifyTransferComplete(
                        command.fromAccountId,
                        command.toAccountId,
                        command.amount
                    );
                } catch (error) {
                    // Log but don't fail saga
                    console.error('Notification failed:', error);
                }
                
                // Step 6: Audit log (non-critical)
                try {
                    await this.auditService.logTransfer({
                        transferId: command.transferId,
                        fromAccountId: command.fromAccountId,
                        toAccountId: command.toAccountId,
                        amount: command.amount,
                        debitId: debitResult.debitId,
                        creditId: creditResult.creditId,
                        timestamp: new Date()
                    });
                } catch (error) {
                    console.error('Audit log failed:', error);
                }
            }, TransferMoneySagaOrchestrator.SAGA_TIMEOUT);
            
            // 4. Success
            await this.sagaRepo.update(sagaId, {
                status: 'COMPLETED',
                completedAt: new Date()
            });
            
            return { success: true, transferId: command.transferId };
            
        } catch (error) {
            // 5. Compensate
            if (error instanceof SagaTimeoutError) {
                await this.sagaRepo.update(sagaId, { status: 'TIMEOUT' });
            } else {
                await this.sagaRepo.update(sagaId, { status: 'FAILED' });
            }
            
            await this.compensate(sagaId);
            
            return {
                success: false,
                error: error.message
            };
        }
    }
    
    private async executeStepWithRetry<T>(
        sagaId: string,
        stepName: string,
        action: () => Promise<T>
    ): Promise<T> {
        let lastError: Error | null = null;
        
        for (let attempt = 0; attempt < TransferMoneySagaOrchestrator.MAX_RETRIES; attempt++) {
            const stepStartTime = Date.now();
            
            try {
                console.log(`[Saga ${sagaId}] Executing ${stepName} (attempt ${attempt + 1}/${TransferMoneySagaOrchestrator.MAX_RETRIES})`);
                
                const result = await action();
                
                // Save successful step
                await this.sagaRepo.addStep(sagaId, {
                    name: stepName,
                    status: 'COMPLETED',
                    result: result,
                    attempts: attempt + 1,
                    duration: Date.now() - stepStartTime
                });
                
                return result;
                
            } catch (error) {
                lastError = error;
                
                // Business errors → No retry
                if (error instanceof BusinessError) {
                    await this.sagaRepo.addStep(sagaId, {
                        name: stepName,
                        status: 'FAILED',
                        error: error.message,
                        attempts: attempt + 1,
                        duration: Date.now() - stepStartTime
                    });
                    throw error;
                }
                
                // Transient errors → Retry
                if (error instanceof TransientError) {
                    if (attempt < TransferMoneySagaOrchestrator.MAX_RETRIES - 1) {
                        const delay = TransferMoneySagaOrchestrator.RETRY_DELAYS[attempt];
                        console.log(`[Saga ${sagaId}] Transient error, retrying in ${delay}ms...`);
                        await sleep(delay);
                        continue;
                    }
                }
                
                // Max retries exceeded
                await this.sagaRepo.addStep(sagaId, {
                    name: stepName,
                    status: 'FAILED',
                    error: error.message,
                    attempts: attempt + 1,
                    duration: Date.now() - stepStartTime
                });
                
                throw error;
            }
        }
        
        throw lastError || new Error(`${stepName} failed after ${TransferMoneySagaOrchestrator.MAX_RETRIES} attempts`);
    }
    
    private async compensate(sagaId: string): Promise<void> {
        console.log(`[Saga ${sagaId}] Starting compensation...`);
        
        const saga = await this.sagaRepo.findById(sagaId);
        const completedSteps = saga.steps
            .filter(s => s.status === 'COMPLETED')
            .reverse(); // Compensate in reverse order
        
        for (const step of completedSteps) {
            try {
                await this.compensateStep(saga.command, step);
            } catch (error) {
                // Log but continue compensating other steps
                console.error(`[Saga ${sagaId}] Failed to compensate ${step.name}:`, error);
            }
        }
        
        await this.sagaRepo.update(sagaId, {
            status: 'COMPENSATED',
            completedAt: new Date()
        });
        
        console.log(`[Saga ${sagaId}] Compensation completed`);
    }
    
    private async compensateStep(
        command: TransferMoneyCommand,
        step: SagaStep
    ): Promise<void> {
        switch (step.name) {
            case 'DebitAccount':
                // Refund the debit
                await this.debitService.refund(
                    command.fromAccountId,
                    step.result.debitId
                );
                console.log(`[Compensation] Refunded debit ${step.result.debitId}`);
                break;
                
            case 'CreditAccount':
                // Normally would reverse credit, but in banking this may require manual intervention
                console.log(`[Compensation] Credit ${step.result.creditId} needs manual reversal`);
                break;
                
            // ValidateFromAccount, ValidateToAccount don't need compensation (read-only)
        }
    }
    
    private async executeWithTimeout<T>(
        action: () => Promise<T>,
        timeoutMs: number
    ): Promise<T> {
        return Promise.race([
            action(),
            new Promise<T>((_, reject) =>
                setTimeout(
                    () => reject(new SagaTimeoutError(`Saga timeout after ${timeoutMs}ms`)),
                    timeoutMs
                )
            )
        ]);
    }
}

// ========== Service Implementations ==========

class AccountServiceImpl implements AccountService {
    constructor(private accountRepo: AccountRepository) {}
    
    async validate(accountId: string): Promise<Account> {
        const account = await this.accountRepo.findById(accountId);
        
        if (!account) {
            throw new BusinessError(`Account ${accountId} not found`);
        }
        
        return account;
    }
}

class DebitServiceImpl implements DebitService {
    constructor(
        private accountRepo: AccountRepository,
        private transactionRepo: TransactionRepository
    ) {}
    
    async debit(
        accountId: string,
        amount: number,
        transferId: string
    ): Promise<DebitResult> {
        // Check idempotency
        const existingDebit = await this.transactionRepo.findByTransferId(transferId, 'DEBIT');
        if (existingDebit) {
            return existingDebit;
        }
        
        const account = await this.accountRepo.findByIdForUpdate(accountId);
        
        if (account.balance < amount) {
            throw new BusinessError('Insufficient funds');
        }
        
        // Debit account
        const newBalance = account.balance - amount;
        await this.accountRepo.update(accountId, { balance: newBalance });
        
        // Record transaction
        const debitId = generateId();
        await this.transactionRepo.save({
            id: debitId,
            accountId: accountId,
            type: 'DEBIT',
            amount: amount,
            balanceBefore: account.balance,
            balanceAfter: newBalance,
            transferId: transferId,
            timestamp: new Date()
        });
        
        return {
            debitId: debitId,
            accountId: accountId,
            amount: amount,
            balanceAfter: newBalance
        };
    }
    
    async refund(accountId: string, debitId: string): Promise<void> {
        const debitTx = await this.transactionRepo.findById(debitId);
        
        if (!debitTx) {
            throw new Error(`Debit transaction ${debitId} not found`);
        }
        
        // Credit back the amount
        const account = await this.accountRepo.findByIdForUpdate(accountId);
        const newBalance = account.balance + debitTx.amount;
        await this.accountRepo.update(accountId, { balance: newBalance });
        
        // Record refund transaction
        await this.transactionRepo.save({
            id: generateId(),
            accountId: accountId,
            type: 'REFUND',
            amount: debitTx.amount,
            balanceBefore: account.balance,
            balanceAfter: newBalance,
            relatedTransactionId: debitId,
            timestamp: new Date()
        });
        
        console.log(`Refunded ${debitTx.amount} to account ${accountId}`);
    }
}

class CreditServiceImpl implements CreditService {
    constructor(
        private accountRepo: AccountRepository,
        private transactionRepo: TransactionRepository
    ) {}
    
    async credit(
        accountId: string,
        amount: number,
        transferId: string
    ): Promise<CreditResult> {
        // Check idempotency
        const existingCredit = await this.transactionRepo.findByTransferId(transferId, 'CREDIT');
        if (existingCredit) {
            return existingCredit;
        }
        
        const account = await this.accountRepo.findByIdForUpdate(accountId);
        
        // Credit account
        const newBalance = account.balance + amount;
        await this.accountRepo.update(accountId, { balance: newBalance });
        
        // Record transaction
        const creditId = generateId();
        await this.transactionRepo.save({
            id: creditId,
            accountId: accountId,
            type: 'CREDIT',
            amount: amount,
            balanceBefore: account.balance,
            balanceAfter: newBalance,
            transferId: transferId,
            timestamp: new Date()
        });
        
        return {
            creditId: creditId,
            accountId: accountId,
            amount: amount,
            balanceAfter: newBalance
        };
    }
}

// ========== Tests ==========

describe('Transfer Money Orchestration Saga', () => {
    let orchestrator: TransferMoneySagaOrchestrator;
    let accountRepo: InMemoryAccountRepository;
    let transactionRepo: InMemoryTransactionRepository;
    let sagaRepo: InMemorySagaRepository;
    
    beforeEach(() => {
        accountRepo = new InMemoryAccountRepository();
        transactionRepo = new InMemoryTransactionRepository();
        sagaRepo = new InMemorySagaRepository();
        
        const accountService = new AccountServiceImpl(accountRepo);
        const debitService = new DebitServiceImpl(accountRepo, transactionRepo);
        const creditService = new CreditServiceImpl(accountRepo, transactionRepo);
        const notificationService = mockNotificationService();
        const auditService = mockAuditService();
        
        orchestrator = new TransferMoneySagaOrchestrator(
            accountService,
            debitService,
            creditService,
            notificationService,
            auditService,
            sagaRepo
        );
    });
    
    it('should complete transfer successfully', async () => {
        // Setup accounts
        await accountRepo.save({ id: 'acc1', balance: 1000, currency: 'USD', status: 'ACTIVE' });
        await accountRepo.save({ id: 'acc2', balance: 500, currency: 'USD', status: 'ACTIVE' });
        
        // Execute transfer
        const result = await orchestrator.execute({
            transferId: 'tx1',
            fromAccountId: 'acc1',
            toAccountId: 'acc2',
            amount: 100,
            currency: 'USD'
        });
        
        // Verify success
        expect(result.success).toBe(true);
        
        // Verify balances
        const acc1 = await accountRepo.findById('acc1');
        const acc2 = await accountRepo.findById('acc2');
        expect(acc1.balance).toBe(900);
        expect(acc2.balance).toBe(600);
        
        // Verify saga completed
        const sagas = await sagaRepo.findByTransferId('tx1');
        expect(sagas[0].status).toBe('COMPLETED');
    });
    
    it('should compensate when to-account is invalid', async () => {
        // Setup: only from account exists
        await accountRepo.save({ id: 'acc1', balance: 1000, currency: 'USD', status: 'ACTIVE' });
        // acc2 doesn't exist
        
        const result = await orchestrator.execute({
            transferId: 'tx2',
            fromAccountId: 'acc1',
            toAccountId: 'acc2',  // Invalid
            amount: 100,
            currency: 'USD'
        });
        
        // Verify failure
        expect(result.success).toBe(false);
        expect(result.error).toContain('not found');
        
        // Verify balance unchanged (no debit happened)
        const acc1 = await accountRepo.findById('acc1');
        expect(acc1.balance).toBe(1000);
        
        // Verify saga compensated
        const sagas = await sagaRepo.findByTransferId('tx2');
        expect(sagas[0].status).toBe('COMPENSATED');
    });
    
    it('should retry and succeed after transient error', async () => {
        await accountRepo.save({ id: 'acc1', balance: 1000, currency: 'USD', status: 'ACTIVE' });
        await accountRepo.save({ id: 'acc2', balance: 500, currency: 'USD', status: 'ACTIVE' });
        
        // Mock credit service to fail once then succeed
        let creditAttempts = 0;
        const originalCredit = CreditServiceImpl.prototype.credit;
        CreditServiceImpl.prototype.credit = async function(...args) {
            creditAttempts++;
            if (creditAttempts === 1) {
                throw new TransientError('Network timeout');
            }
            return originalCredit.apply(this, args);
        };
        
        const result = await orchestrator.execute({
            transferId: 'tx3',
            fromAccountId: 'acc1',
            toAccountId: 'acc2',
            amount: 100,
            currency: 'USD'
        });
        
        // Verify success after retry
        expect(result.success).toBe(true);
        expect(creditAttempts).toBe(2);
        
        // Verify balances correct
        const acc1 = await accountRepo.findById('acc1');
        const acc2 = await accountRepo.findById('acc2');
        expect(acc1.balance).toBe(900);
        expect(acc2.balance).toBe(600);
    });
    
    it('should be idempotent (same transferId returns same result)', async () => {
        await accountRepo.save({ id: 'acc1', balance: 1000, currency: 'USD', status: 'ACTIVE' });
        await accountRepo.save({ id: 'acc2', balance: 500, currency: 'USD', status: 'ACTIVE' });
        
        const command: TransferMoneyCommand = {
            transferId: 'tx4',
            fromAccountId: 'acc1',
            toAccountId: 'acc2',
            amount: 100,
            currency: 'USD'
        };
        
        // Execute first time
        const result1 = await orchestrator.execute(command);
        expect(result1.success).toBe(true);
        
        // Execute again with same transferId
        const result2 = await orchestrator.execute(command);
        expect(result2.success).toBe(true);
        
        // Verify money only transferred once
        const acc1 = await accountRepo.findById('acc1');
        const acc2 = await accountRepo.findById('acc2');
        expect(acc1.balance).toBe(900);  // Not 800
        expect(acc2.balance).toBe(600);  // Not 700
    });
});
```

</details>

---

## Ejercicio 4: Saga con Forward Recovery ⭐⭐⭐⭐

### Escenario Avanzado

Sistema de reserva de vuelos con múltiples aerolíneas. Si la aerolínea preferida no tiene asientos, intentar con aerolíneas alternativas en lugar de cancelar toda la reserva.

### Requisitos

- **Happy Path**: Reservar con aerolínea preferida
- **Forward Recovery**: Si preferida falla → Intentar alternativas
- **Compensación**: Si todas fallan → Cancelar hotel y auto ya reservados
- **Preferencias**: Usuario configura prioridad de aerolíneas

### Implementación

```typescript
interface BookFlightWithAlternativesCommand {
    bookingId: string;
    userId: string;
    route: FlightRoute;
    preferredAirline: string;
    alternativeAirlines: string[];  // Ordenadas por preferencia
    hotelId: string;
    carRentalId: string;
}

class BookTripWithForwardRecoverySaga {
    async execute(command: BookFlightWithAlternativesCommand): Promise<SagaResult> {
        // TODO: Implementar forward recovery
        // 1. Reservar hotel
        // 2. Reservar auto
        // 3. Intentar aerolínea preferida
        // 4. Si falla → Intentar alternativas en orden
        // 5. Si todas fallan → Compensar hotel y auto
    }
}
```

<details>
<summary>💡 Solución con Forward Recovery</summary>

```typescript
class BookTripWithForwardRecoverySaga {
    constructor(
        private hotelService: HotelService,
        private carRentalService: CarRentalService,
        private airlineServices: Map<string, AirlineService>,
        private sagaRepo: SagaRepository
    ) {}
    
    async execute(command: BookFlightWithAlternativesCommand): Promise<SagaResult> {
        const sagaId = generateId();
        
        await this.sagaRepo.save({
            id: sagaId,
            type: 'BookTripWithAlternatives',
            status: 'STARTED',
            command: command,
            steps: [],
            startedAt: new Date()
        });
        
        try {
            // Step 1: Book hotel
            const hotelBooking = await this.executeStep(
                sagaId,
                'BookHotel',
                () => this.hotelService.book(command.hotelId, command.userId)
            );
            
            // Step 2: Book car
            const carBooking = await this.executeStep(
                sagaId,
                'BookCar',
                () => this.carRentalService.book(command.carRentalId, command.userId)
            );
            
            // Step 3: Book flight with forward recovery
            const flightBooking = await this.bookFlightWithFallback(
                sagaId,
                command.route,
                command.userId,
                [command.preferredAirline, ...command.alternativeAirlines]
            );
            
            if (!flightBooking) {
                // All airlines failed → Compensate
                throw new Error('No flights available from any airline');
            }
            
            await this.sagaRepo.update(sagaId, {
                status: 'COMPLETED',
                completedAt: new Date()
            });
            
            return {
                success: true,
                bookingId: command.bookingId,
                hotelBooking: hotelBooking,
                carBooking: carBooking,
                flightBooking: flightBooking
            };
            
        } catch (error) {
            await this.compensate(sagaId);
            
            return {
                success: false,
                error: error.message
            };
        }
    }
    
    private async bookFlightWithFallback(
        sagaId: string,
        route: FlightRoute,
        userId: string,
        airlines: string[]
    ): Promise<FlightBooking | null> {
        for (const airlineCode of airlines) {
            try {
                console.log(`Trying airline: ${airlineCode}`);
                
                const airlineService = this.airlineServices.get(airlineCode);
                if (!airlineService) {
                    console.warn(`Airline ${airlineCode} not available`);
                    continue;
                }
                
                const booking = await this.executeStep(
                    sagaId,
                    `BookFlight_${airlineCode}`,
                    () => airlineService.bookFlight(route, userId)
                );
                
                console.log(`✅ Successfully booked with ${airlineCode}`);
                return booking;
                
            } catch (error) {
                console.warn(`❌ Airline ${airlineCode} failed: ${error.message}`);
                
                // Try next airline (forward recovery)
                continue;
            }
        }
        
        // All airlines failed
        return null;
    }
    
    private async compensate(sagaId: string): Promise<void> {
        const saga = await this.sagaRepo.findById(sagaId);
        const completedSteps = saga.steps.filter(s => s.status === 'COMPLETED').reverse();
        
        for (const step of completedSteps) {
            try {
                await this.compensateStep(step);
            } catch (error) {
                console.error(`Failed to compensate ${step.name}:`, error);
            }
        }
        
        await this.sagaRepo.update(sagaId, { status: 'COMPENSATED' });
    }
    
    private async compensateStep(step: SagaStep): Promise<void> {
        if (step.name === 'BookHotel') {
            await this.hotelService.cancel(step.result.bookingId);
        } else if (step.name === 'BookCar') {
            await this.carRentalService.cancel(step.result.bookingId);
        } else if (step.name.startsWith('BookFlight_')) {
            const airlineCode = step.name.split('_')[1];
            const airlineService = this.airlineServices.get(airlineCode);
            await airlineService?.cancelFlight(step.result.bookingId);
        }
    }
}

// ========== Test ==========

it('should use alternative airline when preferred fails', async () => {
    // Setup: Preferred airline full, alternative has seats
    await mockAirlineService('UA', { hasSeats: false });
    await mockAirlineService('AA', { hasSeats: true });
    
    const result = await saga.execute({
        bookingId: 'b1',
        userId: 'u1',
        route: { from: 'LAX', to: 'JFK' },
        preferredAirline: 'UA',        // Fails
        alternativeAirlines: ['AA'],   // Succeeds
        hotelId: 'h1',
        carRentalId: 'c1'
    });
    
    expect(result.success).toBe(true);
    expect(result.flightBooking.airline).toBe('AA');  // Alternative used
});
```

</details>

---

## Resumen de Ejercicios

| Ejercicio | Patrón | Dificultad | Conceptos Clave |
|-----------|--------|------------|-----------------|
| 1 | Análisis | ⭐ | Identificar problemas distribuidos, justificar patrón |
| 2 | Choreography | ⭐⭐ | Eventos, compensación, tests end-to-end |
| 3 | Orchestration | ⭐⭐⭐ | Coordinador, retry, timeout, idempotencia |
| 4 | Forward Recovery | ⭐⭐⭐⭐ | Alternativas, fallback, compensación parcial |

**Tiempo estimado total**: 10-12 horas
