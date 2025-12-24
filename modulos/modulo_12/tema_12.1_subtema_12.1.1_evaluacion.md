# Evaluación: Events vs Commands vs Queries

## Parte 1. Preguntas Teóricas (30 puntos)

### Pregunta 1 (10 puntos)
Explica la diferencia fundamental entre un Command y un Event. Proporciona un ejemplo de cada uno en el contexto de un sistema de reservas de vuelos.

**Criterios de evaluación**:
- Definición correcta de Command (imperativo, puede fallar)
- Definición correcta de Event (hecho inmutable, ya ocurrió)
- Ejemplos contextualizados y claros
- Mención de características (sincronía/asincronía, handlers)

---

### Pregunta 2 (10 puntos)
¿Por qué el Outbox Pattern es importante en arquitecturas event-driven? ¿Qué problema resuelve?

**Criterios de evaluación**:
- Identificación del problema (inconsistencia DB + Message Broker)
- Explicación de la solución (transacción atómica)
- Mención de garantías (at-least-once delivery)
- Comprensión de trade-offs

---

### Pregunta 3 (10 puntos)
Compara Event Notification vs Event-Carried State Transfer. ¿Cuándo usarías cada uno?

**Criterios de evaluación**:
- Descripción de Event Notification (solo IDs)
- Descripción de Event-Carried State Transfer (datos embebidos)
- Trade-offs: acoplamiento vs performance
- Casos de uso apropiados para cada patrón

---

## Parte 2: Análisis de Código (30 puntos)

### Ejercicio 1 (15 puntos)
Identifica los problemas en este código y refactorízalo aplicando SOLID:

```typescript
class OrderSystem {
    async createOrder(customerId: string, items: any[]) {
        // Save to database
        const order = await db.query(
            'INSERT INTO orders (customer_id, items) VALUES (?, ?)',
            [customerId, JSON.stringify(items)]
        );
        
        // Send to Kafka
        await kafka.send('orders', {
            orderId: order.id,
            customerId: customerId,
            items: items
        });
        
        // Send email
        await emailService.send(
            customerId,
            'Order Created',
            `Your order ${order.id} was created`
        );
        
        return order;
    }
}
```

**Problemas a identificar**:
- ❌ Viola SRP (múltiples responsabilidades)
- ❌ No usa Outbox Pattern (inconsistencia posible)
- ❌ Acoplamiento directo a Kafka y Email
- ❌ No maneja errores parciales

**Solución esperada**:
```typescript
// Command
interface CreateOrderCommand {
    customerId: string;
    items: OrderItem[];
}

// Event
interface OrderCreatedEvent {
    orderId: string;
    customerId: string;
    items: OrderItem[];
    timestamp: Date;
}

// SRP: Aggregate solo lógica de negocio
class Order {
    create(command: CreateOrderCommand): OrderCreatedEvent {
        // Validate
        if (command.items.length === 0) {
            throw new Error('Order must have items');
        }
        
        return {
            orderId: generateId(),
            customerId: command.customerId,
            items: command.items,
            timestamp: new Date()
        };
    }
}

// SRP: Command Handler orquesta
class OrderCommandHandler {
    constructor(
        private orderRepo: OrderRepository,
        private eventBus: EventBus
    ) {}
    
    async handle(command: CreateOrderCommand): Promise<string> {
        const order = new Order();
        const event = order.create(command);
        
        // Atomic: Save order + outbox event
        await this.orderRepo.saveWithEvent(order, event);
        
        // Publish after commit
        await this.eventBus.publish(event);
        
        return event.orderId;
    }
}

// SRP: Event Handler solo notificaciones
class OrderNotificationHandler {
    async handle(event: OrderCreatedEvent): Promise<void> {
        await this.emailService.send(
            event.customerId,
            'Order Created',
            `Your order ${event.orderId} was created`
        );
    }
}
```

---

### Ejercicio 2 (15 puntos)
Este código tiene un bug de concurrencia. Identifícalo y arréglalo:

```typescript
class EventStore {
    async save(aggregateId: string, event: DomainEvent): Promise<void> {
        const events = await this.load(aggregateId);
        events.push(event);
        await this.db.save(aggregateId, events);
    }
}
```

**Problema**: Race condition (dos requests simultáneos pueden sobrescribirse)

**Solución esperada**:
```typescript
class EventStore {
    async save(aggregateId: string, event: DomainEvent): Promise<void> {
        const currentVersion = await this.getVersion(aggregateId);
        
        // Optimistic concurrency check
        if (event.version !== currentVersion + 1) {
            throw new ConcurrencyException(
                `Expected version ${currentVersion + 1}, got ${event.version}`
            );
        }
        
        // Use database constraint
        await this.db.execute(
            `INSERT INTO events (aggregate_id, version, event_data)
             VALUES (?, ?, ?)`,
            [aggregateId, event.version, event]
        );
        // Unique constraint on (aggregate_id, version) prevents duplicates
    }
}
```

---

## Parte 3: Implementación Práctica (40 puntos)

### Proyecto: Sistema de Gestión de Biblioteca

Implementa un sistema event-driven para una biblioteca con los siguientes requisitos:

#### Requisitos Funcionales:

1. **Préstamo de Libros**
   - Command: `BorrowBookCommand(userId, bookId, dueDate)`
   - Event: `BookBorrowedEvent(loanId, userId, bookId, borrowedAt, dueDate)`
   - Reglas: Máximo 3 libros por usuario, libro debe estar disponible

2. **Devolución de Libros**
   - Command: `ReturnBookCommand(loanId)`
   - Event: `BookReturnedEvent(loanId, returnedAt, lateDays?)`
   - Reglas: Calcular multa si está atrasado

3. **Notificaciones**
   - Enviar email cuando se presta libro
   - Enviar recordatorio 2 días antes del vencimiento
   - Enviar notificación de multa si hay retraso

#### Requisitos Técnicos:

1. **SOLID Principles**
   - SRP: Separar aggregates, handlers, projections
   - OCP: Event Bus extensible
   - DIP: Abstracciones para EventStore, EmailService

2. **Patterns**
   - Outbox Pattern para garantías transaccionales
   - Event-Carried State Transfer (incluir datos de usuario/libro en eventos)

3. **Testing**
   - Tests unitarios para aggregates
   - Tests de integración para command handlers
   - Tests de proyecciones

#### Estructura Esperada:

```
library/
├── domain/
│   ├── Book.ts (aggregate)
│   ├── Loan.ts (aggregate)
│   └── events.ts
├── application/
│   ├── commands/
│   │   ├── BorrowBookCommand.ts
│   │   └── ReturnBookCommand.ts
│   └── handlers/
│       ├── BorrowBookHandler.ts
│       └── ReturnBookHandler.ts
├── infrastructure/
│   ├── EventStore.ts
│   ├── OutboxProcessor.ts
│   └── EmailService.ts
└── tests/
    ├── Book.test.ts
    └── BorrowBookHandler.test.ts
```

#### Rúbrica (40 puntos):

| Criterio | Puntos | Descripción |
|----------|--------|-------------|
| **Commands/Events** | 8 | Commands bien definidos, Events inmutables, naming correcto |
| **Aggregates** | 8 | Lógica de negocio encapsulada, validaciones, métodos que retornan eventos |
| **SOLID** | 8 | SRP en handlers, OCP en EventBus, DIP con interfaces |
| **Outbox Pattern** | 8 | Implementación correcta, transaccionalidad, background processor |
| **Tests** | 8 | Cobertura >80%, casos edge, integration tests |

#### Casos de Prueba Obligatorios:

```typescript
describe('Library System', () => {
    it('should allow borrowing available book', async () => {
        // Given: Available book, user with < 3 loans
        // When: BorrowBookCommand
        // Then: BookBorrowedEvent published, book marked as unavailable
    });
    
    it('should reject borrowing when user has 3 active loans', async () => {
        // Given: User already has 3 active loans
        // When: BorrowBookCommand
        // Then: Throw exception, no event published
    });
    
    it('should calculate late fee correctly', async () => {
        // Given: Loan overdue by 5 days
        // When: ReturnBookCommand
        // Then: BookReturnedEvent with lateDays=5
    });
    
    it('should send notification email after borrowing', async () => {
        // Given: BookBorrowedEvent
        // When: Event handler processes event
        // Then: Email sent to user with book details
    });
    
    it('should use outbox pattern for transactional guarantees', async () => {
        // Given: Database transaction fails after saving loan
        // When: BorrowBookHandler executes
        // Then: No event published (rollback)
    });
});
```

---

## Parte 4: Preguntas de Diseño (Bonus: 10 puntos)

### Pregunta Bonus 1 (5 puntos)
Tu sistema de e-commerce procesa 10,000 pedidos/día. Cada evento `OrderCreatedEvent` incluye información completa del cliente y productos (2KB). ¿Es esto un problema? ¿Cómo optimizarías?

**Respuesta esperada**:
- Análisis: 10,000 × 2KB = 20MB/día (NO es problema)
- Kafka soporta hasta 1MB por mensaje (2KB está bien)
- Optimizaciones posibles: comprimir eventos, separar en eventos más pequeños
- Trade-off: tamaño vs autonomía de servicios

---

### Pregunta Bonus 2 (5 puntos)
¿Cómo manejarías el caso donde necesitas cambiar el esquema de un evento que ya tiene millones de instancias en producción?

**Respuesta esperada**:
- Event Upcasting: transformar eventos viejos al nuevo esquema al leerlos
- Versionado de eventos: `eventVersion: 1` vs `eventVersion: 2`
- Backward compatibility: nuevos campos opcionales
- Migración batch en background (opcional)
- Nunca eliminar campos (solo agregar)

---

## Criterios de Aprobación

- **0-59 puntos**: Reprobado
- **60-69 puntos**: Aprobado (C)
- **70-79 puntos**: Bueno (B)
- **80-89 puntos**: Muy Bueno (A)
- **90-100 puntos**: Excelente (A+)

## Entregables

1. Código fuente completo del proyecto
2. Tests con >80% cobertura
3. README con instrucciones de ejecución
4. Documento PDF con respuestas a preguntas teóricas
5. Diagrama de arquitectura (opcional, +5 puntos bonus)

**Fecha de entrega**: 7 días después de la evaluación

**Formato de entrega**: Repositorio Git con estructura clara
