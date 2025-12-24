# Evaluación: Saga Patterns

## Parte 1. Conceptos Fundamentales (25 puntos)

### Pregunta 1 (8 puntos)

Explica la diferencia entre **Choreography** y **Orchestration** Saga. Proporciona un ejemplo de cuándo usarías cada uno.

**Criterios**:
- Definición clara de cada patrón
- Ventajas y desventajas de cada uno
- Ejemplo concreto justificando la elección

---

### Pregunta 2 (9 puntos)

Este código intenta manejar una transacción distribuida:

```java
@Transactional
public Order createOrderWithPayment(OrderRequest request) {
    // Save order
    Order order = orderRepository.save(new Order(request));
    
    // Charge payment (external service)
    PaymentResult payment = paymentService.charge(
        order.getCustomerId(),
        order.getTotal()
    );
    
    if (!payment.isSuccess()) {
        throw new PaymentFailedException();  // Rollback order
    }
    
    // Reserve inventory (external service)
    inventoryService.reserve(order.getItems());
    
    return order;
}
```

**Tareas**:
1. Identifica **3 problemas** con este código
2. Explica qué sucede si `inventoryService.reserve()` lanza una excepción
3. Propón cómo refactorizarlo usando Saga Pattern (elige Choreography u Orchestration y justifica)

**Puntos**: 
- Problemas identificados: 3 puntos
- Explicación de comportamiento: 3 puntos
- Propuesta de refactorización: 3 puntos

---

### Pregunta 3 (8 puntos)

¿Qué es **compensación** en Saga Patterns? ¿Por qué no siempre es posible hacer rollback perfecto?

Proporciona un ejemplo de una operación que **no puede compensarse completamente**.

**Criterios**:
- Definición de compensación
- Limitaciones del rollback distribuido
- Ejemplo realista (ej: email enviado, pago procesado por tercero, stock físico vendido)

---

## Parte 2: Análisis y Diseño (30 puntos)

### Ejercicio 1. Diseñar Choreography Saga (15 puntos)

Diseña una Choreography Saga para un sistema de **préstamos bibliotecarios** con:

**Servicios**:
- UserService: Valida usuario no tenga multas
- BookService: Reserva libro
- NotificationService: Notifica usuario
- FeeService: Calcula multas si devolución tardía

**Escenarios**:
1. **Happy path**: Usuario válido, libro disponible → Préstamo confirmado
2. **Usuario con multas**: Rechazar préstamo
3. **Libro no disponible**: Rechazar después de validar usuario

**Tareas**:
- Definir eventos (mínimo 6)
- Diseñar flujo completo con compensaciones
- Explicar cómo manejas caso donde libro se reserva pero usuario tiene multas (compensar reserva)

**Rúbrica**:
| Criterio | Puntos |
|----------|--------|
| Eventos bien definidos | 4 |
| Flujo happy path correcto | 3 |
| Compensaciones adecuadas | 4 |
| Manejo de edge cases | 4 |

---

### Ejercicio 2: Diseñar Orchestration Saga (15 puntos)

Diseña una Orchestration Saga para **procesamiento de devoluciones** (refunds) en e-commerce:

**Steps requeridos**:
1. Validar devolución (dentro de 30 días, producto sin usar)
2. Crear etiqueta de envío para devolución
3. Esperar confirmación de recepción del producto
4. Inspeccionar producto recibido
5. Procesar reembolso
6. Restaurar inventario

**Requisitos**:
- Timeout global de 7 días (si producto no llega, cancelar devolución)
- Retry automático para steps con fallas transitorias
- Compensación si inspección falla (producto dañado)

**Tareas**:
- Diseñar interface del Orchestrator
- Definir qué steps requieren compensación y cuáles no
- Explicar estrategia de timeout (qué pasa después de 7 días)

**Rúbrica**:
| Criterio | Puntos |
|----------|--------|
| Orchestrator bien diseñado | 4 |
| Steps y orden correcto | 4 |
| Compensaciones apropiadas | 3 |
| Timeout y retry strategy | 4 |

---

## Parte 3: Implementación (35 puntos)

### Proyecto: Sistema de Reserva de Conciertos con Saga

Implementa un sistema completo de reserva de conciertos usando **Orchestration Saga**.

**Contexto**:
- Usuario reserva boletos para concierto
- Debe bloquear asientos, procesar pago, enviar boletos por email

**Commands**:
- `ReserveConcertTicketsCommand(userId, concertId, seatIds[], paymentMethod)`

**Services**:
- **SeatService**: Bloquea asientos (timeout 15 minutos)
- **PaymentService**: Procesa pago
- **TicketService**: Genera boletos PDF
- **EmailService**: Envía boletos por email

**Requisitos de Implementación**:

#### 1. Orchestrator (12 puntos)

```typescript
class ReserveConcertTicketsSagaOrchestrator {
    private static readonly SAGA_TIMEOUT = 60_000;  // 60 seconds
    private static readonly MAX_RETRIES = 3;
    
    async execute(command: ReserveConcertTicketsCommand): Promise<SagaResult> {
        // TODO: Implementar
        // 1. Validar asientos disponibles
        // 2. Bloquear asientos (con timeout de 15 min)
        // 3. Procesar pago
        // 4. Generar boletos PDF
        // 5. Enviar email (non-critical, no compensate)
        
        // Si falla step 3 (payment) → Liberar asientos
        // Si falla step 4 (PDF generation) → Refund payment, liberar asientos
    }
    
    private async compensate(sagaId: string): Promise<void> {
        // TODO: Implementar compensación en orden inverso
    }
}
```

**Criterios**:
- Timeout global implementado: 2 puntos
- Steps ejecutados en orden: 3 puntos
- Compensación correcta: 4 puntos
- Manejo de errores: 3 puntos

#### 2. Seat Blocking con Timeout (8 puntos)

```typescript
class SeatService {
    async blockSeats(
        concertId: string,
        seatIds: string[],
        reservationId: string
    ): Promise<SeatBlockResult> {
        // TODO: Implementar
        // 1. Verificar asientos disponibles
        // 2. Bloquear con estado 'BLOCKED'
        // 3. Guardar expiración (15 minutos)
        // 4. Retornar blockId
    }
    
    async confirmSeats(blockId: string): Promise<void> {
        // TODO: Cambiar estado de BLOCKED → CONFIRMED
    }
    
    async releaseSeats(blockId: string): Promise<void> {
        // TODO: Cambiar estado de BLOCKED → AVAILABLE
    }
    
    // Background job
    async releaseExpiredBlocks(): Promise<void> {
        // TODO: Cada minuto, liberar bloqueos expirados
    }
}
```

**Criterios**:
- Bloqueo con expiración: 3 puntos
- Confirm/Release correcto: 3 puntos
- Cleanup job implementado: 2 puntos

#### 3. Payment con Idempotencia (7 puntos)

```typescript
class PaymentService {
    async charge(
        userId: string,
        amount: number,
        idempotencyKey: string  // reservationId
    ): Promise<PaymentResult> {
        // TODO: Implementar
        // 1. Verificar si payment ya procesado (idempotency)
        // 2. Si ya existe, retornar resultado previo
        // 3. Si no, procesar payment
        // 4. Guardar resultado
    }
    
    async refund(paymentId: string): Promise<void> {
        // TODO: Procesar reembolso
    }
}
```

**Criterios**:
- Idempotencia correcta: 4 puntos
- Refund implementado: 3 puntos

#### 4. Tests Completos (8 puntos)

```typescript
describe('Concert Ticket Reservation Saga', () => {
    it('should complete reservation successfully', async () => {
        // TODO: Happy path test
        // 1. Reserve tickets
        // 2. Verify seats blocked
        // 3. Verify payment charged
        // 4. Verify PDF generated
        // 5. Verify email sent
    });
    
    it('should release seats when payment fails', async () => {
        // TODO: Compensation test
        // 1. Block seats succeeds
        // 2. Payment fails
        // 3. Verify seats released
    });
    
    it('should be idempotent', async () => {
        // TODO: Idempotency test
        // 1. Reserve with reservationId 'r1'
        // 2. Reserve again with same 'r1'
        // 3. Verify payment only charged once
    });
    
    it('should release expired seat blocks', async () => {
        // TODO: Timeout test
        // 1. Block seats
        // 2. Wait 16 minutes (mock time)
        // 3. Run cleanup job
        // 4. Verify seats released
    });
});
```

**Criterios**:
- Happy path: 2 puntos
- Compensación: 2 puntos
- Idempotencia: 2 puntos
- Timeout cleanup: 2 puntos

---

## Parte 4: Análisis Avanzado (10 puntos)

### Escenario: Distributed Saga Failure

Tu Orchestration Saga falla en el step de "Generar PDF" después de procesar el pago. El servicio de PDFs está completamente caído.

**Preguntas**:

1. **(3 puntos)** ¿Qué opciones tienes para manejar esta situación?
   - Opción A: Compensar pago inmediatamente
   - Opción B: Guardar saga en estado "PENDING" y retry más tarde
   - Opción C: Forward recovery (enviar email con link para descargar PDF después)
   
   Analiza pros/cons de cada opción.

2. **(4 puntos)** Si eliges retry, ¿cómo implementarías retry con exponential backoff hasta que PDF service se recupere? Considera:
   - ¿Cuántos reintentos?
   - ¿Qué delay entre reintentos?
   - ¿Qué haces si nunca se recupera?
   - ¿Cómo notificas al usuario?

3. **(3 puntos)** ¿Cómo garantizarías que si el Orchestrator server crashea en medio de la saga, la saga se recupere al reiniciar?

   **Pista**: Persistent saga state, recovery mechanism

---

## Bonus: Saga con Multiple Compensations (10 puntos extra)

Implementa un caso avanzado donde un step puede fallar **durante la compensación**.

**Escenario**:
- Payment service procesa pago exitosamente
- PDF generation falla → Inicia compensación
- Refund (compensación de payment) también falla (servicio caído)

**Tareas**:
1. Implementar retry de compensación
2. Si retry falla, guardar compensación pendiente en "dead letter queue"
3. Implementar proceso manual de revisión
4. Notificar al equipo de ops cuando compensación falla

```typescript
interface FailedCompensation {
    sagaId: string;
    stepName: string;
    compensationAttempts: number;
    lastError: string;
    requiresManualIntervention: boolean;
}

class CompensationDeadLetterQueue {
    async addFailedCompensation(failed: FailedCompensation): Promise<void> {
        // TODO: Guardar compensación fallida
    }
    
    async getFailedCompensations(): Promise<FailedCompensation[]> {
        // TODO: Para dashboard de ops
    }
    
    async retryCompensation(sagaId: string): Promise<void> {
        // TODO: Reintentar manualmente
    }
}
```

---

## Criterios de Aprobación

- **0-59 puntos**: Reprobado
- **60-69 puntos**: Aprobado (C)
- **70-79 puntos**: Bueno (B)
- **80-89 puntos**: Muy Bueno (A)
- **90-110 puntos**: Excelente (A+)

## Entregables

1. **Código completo** del sistema de reserva de conciertos
2. **Tests** con >80% cobertura
3. **Documento** con respuestas a preguntas teóricas
4. **Diagrama de flujo** de la saga (incluir compensaciones)
5. **README** con decisiones de diseño y trade-offs

**Tiempo estimado**: 10-12 horas

**Formato**: Repositorio Git con estructura clara
