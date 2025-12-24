# Ejercicios: Bounded Contexts y SRP

## ⭐ Ejercicio 1. Identificación de Bounded Contexts

**Contexto**: Tienes un sistema monolítico de gestión universitaria con las siguientes funcionalidades:

```java
public class University System {
    // Student management
    void enrollStudent(Student student);
    void updateGrades(StudentId id, List<Grade> grades);
    void calculateGPA(StudentId id);
    
    // Course management
    void createCourse(Course course);
    void assignProfessor(CourseId course, ProfessorId professor);
    void scheduleCourse(CourseId course, TimeSlot slot, Room room);
    
    // Library management
    void checkoutBook(StudentId student, BookId book);
    void returnBook(BookId book);
    void payLateFee(StudentId student, Money amount);
    
    // Financial management
    void chargeTuition(StudentId student, Money amount);
    void processPayment(PaymentId payment);
    void generateInvoice(StudentId student);
    
    // Housing management
    void assignDorm(StudentId student, DormId dorm);
    void reportMaintenance(DormId dorm, MaintenanceRequest request);
    void chargeHousingFee(StudentId student);
}
```

**Tareas**:
1. Identificar bounded contexts apropiados
2. Dibujar context map mostrando relaciones
3. Para cada contexto, definir:
   - Agregados principales
   - Eventos de dominio
   - Lenguaje ubiquitario
4. Identificar qué datos son compartidos y cómo

<details>
<summary>💡 Solución</summary>

```markdown
## Bounded Contexts Identificados

### 1. Academic Context
**Responsabilidad**: Gestión académica (cursos, calificaciones, inscripciones)
**Agregados**:
- Enrollment (root)
- Course
- Grade

**Eventos**:
- StudentEnrolled
- GradeSubmitted
- CourseCompleted

**Lenguaje**: Student, Enrollment, GPA, Transcript, Course, Credits

### 2. Library Context
**Responsabilidad**: Gestión de biblioteca
**Agregados**:
- Checkout (root)
- Book
- LateFee

**Eventos**:
- BookCheckedOut
- BookReturned
- LateFeeIncurred

**Lenguaje**: Patron, Loan, Fine, Catalog

### 3. Finance Context
**Responsabilidad**: Facturación y pagos
**Agregados**:
- Invoice (root)
- Payment

**Eventos**:
- InvoiceGenerated
- PaymentProcessed
- TuitionCharged

**Lenguaje**: Account, Charge, Payment, Balance

### 4. Housing Context
**Responsabilidad**: Alojamiento estudiantil
**Agregados**:
- DormAssignment (root)
- MaintenanceTicket

**Eventos**:
- DormAssigned
- MaintenanceRequested
- HousingFeeCharged

**Lenguaje**: Resident, RoomAssignment, Occupancy

## Context Map

```
┌─────────────┐         ┌──────────────┐
│  Academic   │────────▶│   Finance    │
│   Context   │ Customer│   Context    │
└─────────────┘ Supplier└──────────────┘
      │                         ▲
      │ Shared Kernel           │
      │ (Student ID)            │
      │                         │
┌─────────────┐         ┌──────────────┐
│   Library   │────────▶│   Housing    │
│   Context   │         │   Context    │
└─────────────┘         └──────────────┘
```

## Datos Compartidos

**Student ID**: Shared Kernel entre todos los contextos
- Implementación: UUID generado por Identity Service
- Cada contexto mantiene copia local del Student ID
- Sincronización vía eventos (StudentRegistered)

</details>

---

## ⭐⭐ Ejercicio 2: Refactoring Distributed Monolith

**Contexto**: El siguiente código representa un Distributed Monolith. Refactorízalo para crear microservicios verdaderamente autónomos.

```typescript
// order-service/src/order-controller.ts
export class OrderController {
    constructor(
        private orderRepository: OrderRepository,
        private customerClient: CustomerServiceClient,
        private inventoryClient: InventoryServiceClient,
        private paymentClient: PaymentServiceClient,
        private shippingClient: ShippingServiceClient,
        private notificationClient: NotificationServiceClient
    ) {}
    
    async createOrder(request: CreateOrderRequest): Promise<Order> {
        // Sincrónico: Customer service
        const customer = await this.customerClient.getCustomer(request.customerId);
        if (!customer) throw new Error('Customer not found');
        
        // Sincrónico: Inventory service
        const items = await this.inventoryClient.checkAvailability(request.items);
        if (!items.allAvailable) throw new Error('Items not available');
        
        // Sincrónico: Payment service
        const payment = await this.paymentClient.authorize(request.paymentMethod, items.total);
        if (!payment.authorized) throw new Error('Payment declined');
        
        // Guardar orden
        const order = await this.orderRepository.save(new Order(customer, items, payment));
        
        // Sincrónico: Shipping service
        await this.shippingClient.scheduleShipment(order.id, customer.address);
        
        // Sincrónico: Notification service
        await this.notificationClient.sendEmail(customer.email, 'Order confirmed');
        
        return order;
    }
}
```

**Problemas Identificados**:
1. 5 llamadas síncronas → alta latencia
2. Si cualquier servicio falla, todo falla
3. No hay compensación para fallos parciales
4. Order service no puede funcionar sin los demás

**Tareas**:
1. Rediseñar con comunicación asíncrona
2. Implementar eventual consistency
3. Añadir circuit breakers
4. Crear read models locales

<details>
<summary>💡 Solución</summary>

```typescript
// ============================================
// order-service/src/domain/order.ts
// ============================================
export class Order {
    private constructor(
        public readonly id: OrderId,
        private customerId: CustomerId,
        private items: OrderItem[],
        private status: OrderStatus,
        private events: DomainEvent[] = []
    ) {}
    
    static create(
        customerId: CustomerId,
        items: OrderItem[],
        customerData: CustomerSnapshot  // Read model local
    ): Order {
        const order = new Order(
            OrderId.generate(),
            customerId,
            items,
            OrderStatus.PENDING,
            []
        );
        
        order.addEvent(new OrderCreated(
            order.id,
            customerId,
            items,
            customerData.shippingAddress
        ));
        
        return order;
    }
    
    confirm(paymentId: PaymentId): void {
        if (this.status !== OrderStatus.PENDING) {
            throw new Error('Order cannot be confirmed');
        }
        
        this.status = OrderStatus.CONFIRMED;
        this.addEvent(new OrderConfirmed(this.id, paymentId));
    }
}

// ============================================
// order-service/src/application/order-service.ts
// ============================================
export class OrderApplicationService {
    constructor(
        private orderRepository: OrderRepository,
        private customerReadModel: CustomerReadModel,  // Cache local
        private inventoryReadModel: InventoryReadModel, // Cache local
        private eventBus: EventBus,
        private circuitBreaker: CircuitBreaker
    ) {}
    
    async createOrder(command: CreateOrderCommand): Promise<OrderId> {
        // Datos locales (no requieren llamadas remotas)
        const customer = this.customerReadModel.get(command.customerId);
        if (!customer) {
            throw new CustomerNotFoundError(command.customerId);
        }
        
        const inventory = this.inventoryReadModel.checkAvailability(command.items);
        if (!inventory.sufficient) {
            throw new InsufficientInventoryError(command.items);
        }
        
        // Crear orden optimísticamente
        const order = Order.create(command.customerId, command.items, customer);
        await this.orderRepository.save(order);
        
        // Publicar eventos (asíncrono)
        for (const event of order.getDomainEvents()) {
            await this.eventBus.publish(event);
        }
        
        // Los demás servicios reaccionan a eventos:
        // - Payment escucha OrderCreated → authorizes payment → publica PaymentAuthorized
        // - Inventory escucha OrderCreated → reserves stock → publica StockReserved
        // - Shipping escucha OrderConfirmed → schedules shipment
        // - Notification escucha OrderConfirmed → sends email
        
        return order.id;
    }
}

// ============================================
// order-service/src/infrastructure/event-handlers.ts
// ============================================
export class OrderEventHandlers {
    constructor(
        private orderRepository: OrderRepository,
        private circuitBreaker: CircuitBreaker
    ) {}
    
    @EventHandler(PaymentAuthorized)
    async onPaymentAuthorized(event: PaymentAuthorized): Promise<void> {
        const order = await this.orderRepository.findById(event.orderId);
        if (!order) return;
        
        order.confirm(event.paymentId);
        await this.orderRepository.save(order);
        
        // Esto publica OrderConfirmed
    }
    
    @EventHandler(PaymentFailed)
    async onPaymentFailed(event: PaymentFailed): Promise<void> {
        const order = await this.orderRepository.findById(event.orderId);
        if (!order) return;
        
        order.cancel('Payment failed');
        await this.orderRepository.save(order);
    }
}

// ============================================
// order-service/src/infrastructure/read-models.ts
// ============================================
export class CustomerReadModel {
    constructor(private cache: RedisClient) {}
    
    @EventHandler(CustomerCreated)
    async onCustomerCreated(event: CustomerCreated): Promise<void> {
        await this.cache.set(
            `customer:${event.customerId}`,
            JSON.stringify({
                id: event.customerId,
                name: event.name,
                email: event.email,
                shippingAddress: event.defaultAddress
            }),
            { ttl: 3600 }
        );
    }
    
    @EventHandler(CustomerAddressUpdated)
    async onAddressUpdated(event: CustomerAddressUpdated): Promise<void> {
        const key = `customer:${event.customerId}`;
        const customer = JSON.parse(await this.cache.get(key));
        customer.shippingAddress = event.newAddress;
        await this.cache.set(key, JSON.stringify(customer), { ttl: 3600 });
    }
    
    get(customerId: CustomerId): CustomerSnapshot | null {
        const data = await this.cache.get(`customer:${customerId}`);
        return data ? JSON.parse(data) : null;
    }
}

// ============================================
// order-service/src/infrastructure/circuit-breaker.ts
// ============================================
export class CircuitBreaker {
    private failureCount = 0;
    private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';
    private lastFailureTime: Date | null = null;
    
    async execute<T>(fn: () => Promise<T>, fallback: () => T): Promise<T> {
        if (this.state === 'OPEN') {
            // Check if we should try again
            if (Date.now() - this.lastFailureTime!.getTime() > 60000) {
                this.state = 'HALF_OPEN';
            } else {
                return fallback();
            }
        }
        
        try {
            const result = await fn();
            this.onSuccess();
            return result;
        } catch (error) {
            this.onFailure();
            return fallback();
        }
    }
    
    private onSuccess(): void {
        this.failureCount = 0;
        this.state = 'CLOSED';
    }
    
    private onFailure(): void {
        this.failureCount++;
        this.lastFailureTime = new Date();
        
        if (this.failureCount >= 5) {
            this.state = 'OPEN';
        }
    }
}
```

**Beneficios del rediseño**:
- ✅ Order service es autónomo (read models locales)
- ✅ Comunicación asíncrona vía eventos
- ✅ Circuit breaker para resiliencia
- ✅ Eventual consistency aceptable
- ✅ Cada servicio puede fallar independientemente

</details>

---

## ⭐⭐⭐ Ejercicio 3: Descomposición Completa de Monolito

**Contexto**: Sistema de gestión de hospital con 50,000 líneas de código en un solo repositorio.

**Funcionalidades**:
- Gestión de pacientes
- Citas médicas
- Historiales clínicos
- Facturación y seguros
- Inventario de farmacia
- Gestión de laboratorios
- Recursos humanos (médicos, enfermeras)

**Tareas**:
1. **Event Storming**: Identificar todos los eventos de dominio
2. **Bounded Contexts**: Agrupar eventos en contextos
3. **Context Map**: Definir relaciones (Customer-Supplier, Shared Kernel, etc.)
4. **Priorización**: Qué microservicio extraer primero y por qué
5. **Migración**: Plan de 6 meses usando Strangler Fig
6. **Código**: Implementar 3 microservicios principales

<details>
<summary>💡 Solución Parcial</summary>

```markdown
## Event Storming

### Eventos Identificados (30+ eventos)

**Patient Management**:
- PatientRegistered
- PatientInformationUpdated
- PatientAdmitted
- PatientDischarged

**Appointments**:
- AppointmentScheduled
- AppointmentCancelled
- AppointmentCompleted
- AppointmentRescheduled

**Medical Records**:
- DiagnosisRecorded
- PrescriptionIssued
- LabTestOrdered
- LabResultsReceived

**Billing**:
- ServiceCharged
- InsuranceClaimSubmitted
- PaymentReceived
- InvoiceGenerated

## Bounded Contexts

### 1. Patient Management Context
### 2. Scheduling Context
### 3. Clinical Context
### 4. Billing Context
### 5. Pharmacy Context
### 6. Laboratory Context

## Priorización

**Primero**: Billing Context
**Razón**: 
- Alta carga (facturación 24/7)
- Requisitos de compliance (HIPAA)
- Equipo de finanzas independiente
- Tecnología especializada (ledger database)

**Segundo**: Laboratory Context
**Razón**:
- Integración con equipos externos (LIMS)
- Escalado independiente
- Baja acoplamiento con otros contextos

</details>

---

## ⭐⭐⭐⭐ Ejercicio 4: Anti-Corruption Layer

**Contexto**: Estás integrando con un servicio legacy de Inventory que tiene un modelo de datos complejo y acoplado. Tu microservicio de Orders usa DDD con modelos limpios.

**API Legacy de Inventory**:
```json
{
  "PROD_ID": "12345",
  "SKU_CODE": "ABC-001",
  "QTY_ON_HAND": 100,
  "QTY_ALLOCATED": 50,
  "QTY_AVAILABLE": 50,
  "WAREHOUSE_LIST": [
    {
      "WH_ID": "WH01",
      "WH_NAME": "Main Warehouse",
      "QTY": 30
    },
    {
      "WH_ID": "WH02",
      "WH_NAME": "Secondary",
      "QTY": 20
    }
  ],
  "UNIT_PRICE": 99.99,
  "CURRENCY": "USD",
  "TAX_CATEGORY": "STANDARD",
  "IS_ACTIVE": "Y",
  "LAST_UPDATED": "2025-12-07T10:30:00Z"
}
```

**Tu modelo de dominio**:
```typescript
export class Product {
    constructor(
        public readonly id: ProductId,
        public readonly name: ProductName,
        public readonly price: Money
    ) {}
}

export class InventoryAvailability {
    constructor(
        public readonly productId: ProductId,
        public readonly availableQuantity: Quantity,
        public readonly warehouses: WarehouseStock[]
    ) {}
}
```

**Tareas**:
1. Diseñar Anti-Corruption Layer completo
2. Implementar traducción bidireccional
3. Añadir caching para reducir llamadas al legacy
4. Implementar circuit breaker
5. Testing de integración

<details>
<summary>💡 Solución</summary>

```typescript
// ============================================
// anti-corruption/inventory-translator.ts
// ============================================
export class InventoryTranslator {
    toDomain(legacyProduct: LegacyProduct): InventoryAvailability {
        return new InventoryAvailability(
            ProductId.from(legacyProduct.PROD_ID),
            Quantity.from(legacyProduct.QTY_AVAILABLE),
            legacyProduct.WAREHOUSE_LIST.map(wh => 
                new WarehouseStock(
                    WarehouseId.from(wh.WH_ID),
                    wh.WH_NAME,
                    Quantity.from(wh.QTY)
                )
            )
        );
    }
    
    toLegacy(reservation: StockReservation): LegacyReservationRequest {
        return {
            PROD_ID: reservation.productId.value,
            QTY_TO_ALLOCATE: reservation.quantity.value,
            CUSTOMER_ID: reservation.customerId.value,
            ORDER_ID: reservation.orderId.value,
            REQUESTED_DATE: new Date().toISOString()
        };
    }
}

// ============================================
// anti-corruption/inventory-facade.ts
// ============================================
export class InventoryServiceFacade {
    constructor(
        private legacyClient: LegacyInventoryClient,
        private translator: InventoryTranslator,
        private cache: CacheService,
        private circuitBreaker: CircuitBreaker
    ) {}
    
    async checkAvailability(productId: ProductId): Promise<InventoryAvailability> {
        // Check cache first
        const cached = await this.cache.get(`inventory:${productId.value}`);
        if (cached) {
            return this.translator.toDomain(cached);
        }
        
        // Call legacy service with circuit breaker
        const legacyData = await this.circuitBreaker.execute(
            () => this.legacyClient.getProduct(productId.value),
            () => this.getFallbackData(productId)
        );
        
        // Cache for 5 minutes
        await this.cache.set(`inventory:${productId.value}`, legacyData, 300);
        
        return this.translator.toDomain(legacyData);
    }
    
    private getFallbackData(productId: ProductId): LegacyProduct {
        // Return conservative estimate
        return {
            PROD_ID: productId.value,
            QTY_AVAILABLE: 0,  // Assume out of stock
            WAREHOUSE_LIST: []
        };
    }
}
```

</details>

---

## Recursos Adicionales

- **Libro**: "Domain-Driven Design" - Eric Evans
- **Libro**: "Building Microservices" - Sam Newman
- **Artículo**: "Monolith First" - Martin Fowler
- **Curso**: "Microservices Patterns" - Chris Richardson
