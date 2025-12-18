# Guion de Video: Events vs Commands vs Queries

**Duración**: 25 minutos  
**Nivel**: Intermedio  
**Requisitos previos**: Conocimientos básicos de arquitectura de software

---

## [00:00 - 01:00] Introducción

**[PANTALLA: Título animado]**

🎬 **Narración**:
"Bienvenidos a este módulo sobre Event-Driven Architecture. Hoy vamos a explorar uno de los conceptos más importantes en arquitecturas modernas: la diferencia entre Events, Commands y Queries.

Si alguna vez te has preguntado por qué Netflix puede manejar millones de usuarios simultáneos, o cómo Amazon procesa miles de pedidos por segundo sin perder consistencia, la respuesta está en arquitecturas event-driven.

Al final de este video, entenderás cuándo usar Events vs Commands, cómo aplicar principios SOLID a sistemas event-driven, y patrones cruciales como Outbox Pattern y Event-Carried State Transfer."

**[VISUAL: Logos de Netflix, Amazon, Uber]**

---

## [01:00 - 05:00] Problema: ¿Por qué necesitamos Events?

**[PANTALLA: Código de arquitectura monolítica]**

🎬 **Narración**:
"Comencemos con un problema real. Imagina que estás construyendo un sistema de e-commerce tradicional."

**[VISUAL: Código Java]**

```java
public class OrderService {
    public void createOrder(Order order) {
        orderRepository.save(order);
        inventoryService.reduceStock(order.items);
        emailService.sendConfirmation(order);
        analyticsService.trackSale(order);
    }
}
```

🎬 **Narración**:
"¿Ves el problema? Este código viola el principio de Single Responsibility. ¿Qué pasa si el servicio de email falla? ¿Se debe cancelar todo el pedido? ¿Qué pasa si necesitas agregar notificaciones SMS?"

**[VISUAL: Diagrama mostrando acoplamiento]**

"Además, todo está **fuertemente acoplado**. Agregar una nueva funcionalidad requiere modificar este método, violando el principio Open/Closed."

**[ANIMACIÓN: Mostrar cascada de fallos]**

"Y lo peor: si cualquier servicio falla, el usuario tiene que esperar mientras todo se ejecuta **sincrónicamente**. Esto es lento e ineficiente."

---

## [05:00 - 10:00] Solución: Commands vs Events

**[TRANSICIÓN: Arquitectura Event-Driven]**

🎬 **Narración**:
"Aquí es donde entran Commands y Events. Veamos las diferencias fundamentales."

**[PANTALLA: Tabla comparativa animada]**

| Aspecto | Command | Event |
|---------|---------|-------|
| **Tiempo Verbal** | Imperativo (CreateOrder) | Pasado (OrderCreated) |
| **Puede Fallar** | ✅ Sí | ❌ No (ya ocurrió) |
| **Handlers** | 1 (exactamente uno) | N (cero o más) |
| **Intención** | "Haz esto" | "Esto ocurrió" |

🎬 **Narración**:
"Un **Command** es una instrucción: 'CreateOrder'. Puede fallar si el stock no existe. Tiene exactamente un handler que lo procesa."

**[VISUAL: Animación de Command flow]**

```
Usuario → CreateOrderCommand → OrderHandler → ✅/❌
```

"Un **Event** es un hecho inmutable: 'OrderCreated'. Ya ocurrió, no puede fallar. Puede tener múltiples handlers interesados."

**[VISUAL: Animación de Event flow]**

```
OrderCreatedEvent → ┬→ InventoryHandler
                    ├→ EmailHandler
                    ├→ AnalyticsHandler
                    └→ NotificationHandler
```

---

## [10:00 - 15:00] SOLID en Event-Driven

**[PANTALLA: Código refactorizado]**

🎬 **Narración**:
"Ahora refactoricemos nuestro código aplicando SOLID. Primero, el **Single Responsibility Principle**."

**[VISUAL: Código TypeScript con animación]**

```typescript
// SRP: Cada handler una responsabilidad
class InventoryEventHandler implements EventHandler<OrderCreatedEvent> {
    async handle(event: OrderCreatedEvent): Promise<void> {
        await this.inventory.reduceStock(event.items);
    }
}

class NotificationEventHandler implements EventHandler<OrderCreatedEvent> {
    async handle(event: OrderCreatedEvent): Promise<void> {
        await this.email.send(event.customerId, 'Order created!');
    }
}
```

🎬 **Narración**:
"Cada handler tiene **una sola responsabilidad**. Si el email falla, el inventario ya se redujo. Están desacoplados."

**[TRANSICIÓN: Open/Closed Principle]**

```typescript
// OCP: Extensible sin modificar EventBus
class EventBus {
    private handlers = new Map<string, EventHandler[]>();
    
    register<T>(eventType: string, handler: EventHandler<T>) {
        // Agregar handler sin modificar código existente
    }
}

// Agregar nuevo handler SIN cambiar EventBus
eventBus.register('OrderCreated', new SMSHandler());  // ✅
```

🎬 **Narración**:
"El EventBus es **extensible** sin modificar su código. Esto es Open/Closed Principle en acción."

**[VISUAL: Dependency Inversion]**

```kotlin
// DIP: Depender de abstracciones
interface EventPublisher {
    suspend fun publish(event: DomainEvent)
}

class KafkaEventPublisher : EventPublisher { ... }
class RabbitMQEventPublisher : EventPublisher { ... }
```

---

## [15:00 - 20:00] Patrones Cruciales

**[PANTALLA: Título "Outbox Pattern"]**

🎬 **Narración**:
"Ahora hablemos del **Outbox Pattern**. Este patrón resuelve un problema crítico: ¿qué pasa si guardas el pedido en la base de datos pero Kafka se cae antes de publicar el evento?"

**[VISUAL: Diagrama del problema]**

```
❌ Problema:
   1. db.save(order)      ✅
   2. kafka.publish()     ❌ (Kafka caído)
   
   Resultado: Pedido guardado, pero nadie se enteró!
```

🎬 **Narración**:
"La solución: guardar el evento en una tabla `outbox` en la **misma transacción** que el pedido."

**[ANIMACIÓN: Outbox Pattern flow]**

```java
@Transactional
public void createOrder(Order order) {
    // 1. Guardar pedido
    orderRepo.save(order);
    
    // 2. Guardar evento en outbox (misma transacción)
    outboxRepo.save(new OutboxEvent(
        "OrderCreated",
        toJson(new OrderCreatedEvent(order))
    ));
    
    // Commit atómico: ambos o ninguno
}

// 3. Proceso en background lee outbox y publica
@Scheduled(fixedDelay = 5000)
public void processOutbox() {
    List<OutboxEvent> pending = outboxRepo.findUnprocessed();
    
    for (OutboxEvent event : pending) {
        kafka.publish(event);
        event.markProcessed();
    }
}
```

🎬 **Narración**:
"Con esto garantizamos **at-least-once delivery**. Si Kafka falla, el evento permanece en outbox y se reintentará."

**[TRANSICIÓN: Event-Carried State Transfer]**

🎬 **Narración**:
"Otro patrón importante: **Event-Carried State Transfer**. En lugar de enviar solo IDs, embebe los datos necesarios."

**[VISUAL: Comparación lado a lado]**

```typescript
// ❌ Event Notification (solo IDs)
interface BookingCreatedEventBad {
    bookingId: string;
    customerId: string;    // Solo ID
    flightId: string;      // Solo ID
}

// Consumidor necesita llamar APIs:
const customer = await customerService.get(event.customerId);  // 50ms
const flight = await flightService.get(event.flightId);        // 100ms
// Total: 150ms + acoplamiento

// ✅ Event-Carried State Transfer (datos embebidos)
interface BookingCreatedEvent {
    bookingId: string;
    customer: {
        name: string;
        email: string;
        loyaltyTier: string;
    },
    flight: {
        number: string;
        departure: FlightDetails;
        arrival: FlightDetails;
    }
}

// Consumidor tiene TODO:
sendEmail(event.customer.email, renderTemplate(event));
// Total: 5ms, sin dependencias externas
```

🎬 **Narración**:
"El trade-off: eventos más grandes (2KB vs 200 bytes), pero **30x más rápido** y sin acoplamiento."

---

## [20:00 - 23:00] Demostración Práctica

**[PANTALLA: Terminal + IDE]**

🎬 **Narración**:
"Veamos esto en acción. Voy a ejecutar nuestro sistema de órdenes con eventos."

**[DEMO: Ejecutar código]**

```bash
$ npm test

✓ EventBus supports multiple handlers
✓ Handlers run in parallel
✓ Error in one handler doesn't affect others
✓ Outbox pattern guarantees delivery

Performance Test:
  Event Notification:        150ms
  Event-Carried Transfer:      5ms
  Improvement:               30x faster ✅
```

🎬 **Narración**:
"Como ven, los handlers corren en **paralelo**, los errores están **aislados**, y el Outbox Pattern garantiza **entrega confiable**."

---

## [23:00 - 25:00] Resumen y Siguientes Pasos

**[PANTALLA: Puntos clave animados]**

🎬 **Narración**:
"Recapitulemos los puntos clave:

1. **Commands**: Imperativos, pueden fallar, 1 handler
2. **Events**: Pasado, inmutables, N handlers
3. **SOLID**: SRP en handlers, OCP en EventBus, DIP en publishers
4. **Outbox Pattern**: Garantías transaccionales
5. **State Transfer**: Performance vs tamaño

En el próximo video exploraremos **Event Sourcing**: cómo guardar eventos en lugar de estado actual, y por qué esto te da superpoderes como time travel y audit trail completo.

Hasta la próxima!"

**[PANTALLA: Call to action]**

- 📚 Material complementario en el repositorio
- 💻 Ejercicios prácticos disponibles
- 🎯 Próximo tema: Event Sourcing y CQRS

---

## Recursos Visuales

### Animaciones Clave:
1. **Command flow**: Usuario → Command → Handler (linear)
2. **Event flow**: Event → Múltiples handlers (fan-out)
3. **Outbox Pattern**: DB transaction con outbox + background processor
4. **Performance comparison**: Event Notification vs State Transfer

### Código a Mostrar:
- TypeScript: EventBus implementation
- Java: Outbox Pattern con Spring Boot
- Python: Event handlers con SRP
- Kotlin: DIP con EventPublisher interface

### Diagramas:
1. Arquitectura monolítica vs Event-Driven
2. Outbox Pattern sequence diagram
3. Event-Carried State Transfer comparison

## Notas para el Editor

- Usar syntax highlighting para código
- Animaciones suaves para transiciones
- Zoom en puntos clave del código
- Background music sutil (no distractiva)
- Subtítulos para términos técnicos
