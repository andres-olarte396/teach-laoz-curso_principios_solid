# Guion de Video: Event Sourcing y CQRS

**Duración**: 28 minutos  
**Nivel**: Avanzado  
**Requisitos previos**: Eventos, Commands, arquitectura event-driven

---

## [00:00 - 01:30] Introducción

**[PANTALLA: Título animado "Event Sourcing + CQRS"]**

🎬 **Narración**:
"Bienvenidos al siguiente nivel de arquitecturas event-driven: Event Sourcing y CQRS.

Imagina poder viajar en el tiempo en tu aplicación. Ver exactamente qué pasó hace 3 meses. Reproducir bugs con precisión exacta. Reconstruir el estado de tu sistema en cualquier momento del pasado.

Esto no es ciencia ficción. Es Event Sourcing.

Y cuando lo combinas con CQRS, obtienes un sistema que puede escalar a millones de usuarios mientras mantiene un audit trail perfecto.

Hoy veremos cómo empresas como GitHub, Uber y bancos implementan estas arquitecturas para manejar cargas masivas sin perder un solo evento."

**[VISUAL: Logos de GitHub, Uber, bancos]**

---

## [01:30 - 07:00] El Problema con Persistencia Tradicional

**[PANTALLA: Base de datos tradicional]**

🎬 **Narración**:
"Primero, entendamos qué está mal con la persistencia tradicional. Miremos esta tabla de órdenes."

**[VISUAL: Tabla SQL animada]**

```sql
Orders Table:
┌────────┬────────┬──────────────┬───────┐
│ id     │ status │ last_updated │ total │
├────────┼────────┼──────────────┼───────┤
│ 123    │ SHIPPED│ 2024-03-15   │ 99.99 │
└────────┴────────┴──────────────┴───────┘
```

🎬 **Narración**:
"¿Qué vemos aquí? Solo el **estado actual**: SHIPPED. Pero... ¿cuándo se creó el pedido? ¿Cuándo se pagó? ¿Cuándo se envió? **Todo ese historial se perdió.**

Si necesitas investigar por qué un pedido tardó 5 días en enviarse, estás en problemas. No tienes datos.

Si un bug causó que 1000 pedidos se marcaran incorrectamente como 'CANCELLED' el mes pasado, no puedes detectarlo. La información fue sobrescrita."

**[ANIMACIÓN: UPDATE sobrescribiendo datos]**

```sql
UPDATE orders SET status = 'CANCELLED' WHERE id = 123;
-- El estado anterior se perdió para siempre ❌
```

---

## [07:00 - 12:00] Event Sourcing: La Solución

**[TRANSICIÓN: Event Store]**

🎬 **Narración**:
"Event Sourcing invierte este paradigma. En lugar de guardar el estado actual, guardamos **la secuencia de eventos** que llevaron a ese estado."

**[VISUAL: Event Store animado]**

```
Event Store:
┌─────────────────────────────────────────┐
│ OrderCreated      v1   10:00 AM         │
│ PaymentProcessed  v2   10:01 AM         │
│ OrderShipped      v3   11:30 AM         │
└─────────────────────────────────────────┘

Estado actual = Replay de todos los eventos
```

🎬 **Narración**:
"Ahora tenemos el **historial completo**. Podemos ver exactamente qué pasó y cuándo.

Pero aquí viene la parte poderosa: podemos **reconstruir** el estado del pedido en cualquier momento replayando los eventos."

**[ANIMACIÓN: Replay de eventos]**

```typescript
class Order {
    private status: OrderStatus;
    
    static fromEvents(events: DomainEvent[]): Order {
        const order = new Order();
        
        // Replay eventos secuencialmente
        events.forEach(event => order.apply(event));
        
        return order;
    }
    
    private apply(event: DomainEvent): void {
        if (event instanceof OrderCreatedEvent) {
            this.status = OrderStatus.PENDING;
        } else if (event instanceof PaymentProcessedEvent) {
            this.status = OrderStatus.PAID;
        } else if (event instanceof OrderShippedEvent) {
            this.status = OrderStatus.SHIPPED;
        }
    }
}
```

**[DEMO: Ejecutar código]**

🎬 **Narración**:
"Miren esto. Cargamos los 3 eventos y reconstruimos el pedido. El estado final es SHIPPED, pero podemos ver todo el journey."

---

## [12:00 - 17:00] CQRS: Optimización Read/Write

**[PANTALLA: Título "CQRS"]**

🎬 **Narración**:
"Ahora, Event Sourcing es poderoso, pero tiene un problema: leer datos puede ser lento. Si un pedido tiene 100 eventos, tenemos que replayarlos todos cada vez.

Aquí entra CQRS: Command Query Responsibility Segregation. Separamos completamente el modelo de **escritura** del modelo de **lectura**."

**[VISUAL: Arquitectura CQRS animada]**

```
Write Side (Commands):          Read Side (Queries):
┌─────────────────┐            ┌──────────────────┐
│  CreateOrder    │            │  GetOrder        │
│  ShipOrder      │            │  ListOrders      │
└────────┬────────┘            └────────▲─────────┘
         │                              │
         ▼                              │
    ┌─────────┐                         │
    │  Event  │─────Events─────┐        │
    │  Store  │                │        │
    └─────────┘                ▼        │
                         ┌──────────────┴────┐
                         │   Projection      │
                         │  (Read Model)     │
                         └───────────────────┘
                                │
                         ┌──────▼────────┐
                         │   Read DB     │
                         │ (Denormalized)│
                         └───────────────┘
```

🎬 **Narración**:
"El **Write Side** solo maneja commands y genera eventos. Es simple y enfocado en validaciones de negocio.

El **Read Side** escucha eventos y construye **vistas denormalizadas** optimizadas para queries.

La magia: podemos tener **múltiples read models** para diferentes necesidades."

**[ANIMACIÓN: Múltiples projections]**

```python
# Projection 1: Para UI (datos básicos)
class OrderSummaryProjection:
    async def project(self, event: OrderCreatedEvent):
        await db.insert('order_summaries', {
            'order_id': event.order_id,
            'customer_name': event.customer_name,  # Denormalizado
            'total': event.total,
            'status': 'PENDING'
        })

# Projection 2: Para analytics (agregaciones)
class SalesAnalyticsProjection:
    async def project(self, event: OrderCreatedEvent):
        await db.execute(
            "UPDATE daily_sales SET total = total + ? WHERE date = ?",
            event.total, event.timestamp.date()
        )

# Projection 3: Para reporting (histórico completo)
class AuditProjection:
    async def project(self, event: DomainEvent):
        await db.insert('audit_log', {
            'event_type': event.__class__.__name__,
            'data': event.to_json(),
            'timestamp': event.timestamp
        })
```

🎬 **Narración**:
"Tres projections diferentes del **mismo stream de eventos**. Cada una optimizada para su propósito."

---

## [17:00 - 22:00] Snapshots: Optimización de Performance

**[PANTALLA: Problema de performance]**

🎬 **Narración**:
"Ahora, imagina una cuenta bancaria con 10,000 transacciones. Replayar 10,000 eventos cada vez que necesitas el balance es **extremadamente lento**."

**[VISUAL: Benchmark animado]**

```
Sin Snapshots:
  Load account → Replay 10,000 events → 1,000ms ❌

Con Snapshots (cada 100 eventos):
  Load snapshot v10,000 → Replay 0 eventos → 10ms ✅
  
Improvement: 100x faster!
```

🎬 **Narración**:
"Los **snapshots** resuelven esto. Cada N eventos, guardamos una foto del estado actual."

**[ANIMACIÓN: Snapshot process]**

```typescript
class EventStore {
    SNAPSHOT_INTERVAL = 100;
    
    async loadAggregate(aggregateId: string): Promise<Order> {
        // 1. Load latest snapshot
        const snapshot = await this.loadSnapshot(aggregateId);
        
        let order: Order;
        let events: DomainEvent[];
        
        if (snapshot) {
            // Restore from snapshot
            order = Order.fromSnapshot(snapshot);
            
            // Load ONLY events after snapshot
            events = await this.loadEventsSince(
                aggregateId, 
                snapshot.version
            );
        } else {
            // No snapshot, load all events
            order = new Order();
            events = await this.loadAllEvents(aggregateId);
        }
        
        // Replay remaining events
        events.forEach(e => order.apply(e));
        
        return order;
    }
}
```

**[DEMO: Performance comparison]**

🎬 **Narración**:
"Veamos el impacto. Voy a cargar una cuenta con 500 transacciones."

```bash
$ npm run benchmark

Creating account with 500 transactions...
📸 Snapshot created at version 100
📸 Snapshot created at version 200
📸 Snapshot created at version 300
📸 Snapshot created at version 400
📸 Snapshot created at version 500

Performance Test:
✅ With snapshots:    12ms
❌ Without snapshots: 234ms

🚀 Performance improvement: 94.9%
```

---

## [22:00 - 25:00] SOLID en Event Sourcing

**[PANTALLA: Principios SOLID]**

🎬 **Narración**:
"Event Sourcing no significa abandonar SOLID. De hecho, lo refuerza."

**[VISUAL: Código Kotlin]**

```kotlin
// SRP: Aggregate solo lógica de negocio
class Order(val id: String) {
    private var status = OrderStatus.PENDING
    
    // Solo business logic
    fun ship(trackingNumber: String): OrderShippedEvent {
        require(status == OrderStatus.PAID) {
            "Only paid orders can ship"
        }
        
        return OrderShippedEvent(id, trackingNumber, version + 1)
    }
    
    // Solo state changes
    fun apply(event: DomainEvent) {
        when (event) {
            is OrderShippedEvent -> status = OrderStatus.SHIPPED
        }
    }
}

// SRP: EventStore solo persistencia
class EventStore(private val db: Database) {
    suspend fun save(aggregateId: String, event: DomainEvent) {
        db.insert("events", event)
    }
}

// SRP: Projection solo construcción de read model
class OrderProjection(private val readDb: Database) {
    suspend fun project(event: DomainEvent) {
        when (event) {
            is OrderShippedEvent -> updateView(event)
        }
    }
}
```

🎬 **Narración**:
"Cada clase tiene **una sola responsabilidad**. El Aggregate maneja reglas de negocio. El EventStore maneja persistencia. Las Projections construyen vistas.

Esto hace que cada componente sea **fácil de testear** y **fácil de mantener**."

---

## [25:00 - 28:00] Resumen y Trade-offs

**[PANTALLA: Comparación final]**

🎬 **Narración**:
"Recapitulemos lo que aprendimos hoy."

**[VISUAL: Tabla de beneficios]**

```
✅ Beneficios:
   • Audit trail completo automático
   • Time travel (rebuild estado en cualquier momento)
   • Debugging perfecto (replay eventos)
   • Escalabilidad (read/write independientes)
   • Múltiples read models del mismo stream

❌ Trade-offs:
   • Mayor complejidad inicial
   • Eventual consistency (projections)
   • Necesita manejo de schema evolution
   • Más storage (todos los eventos)
```

🎬 **Narración**:
"Event Sourcing no es para todo. Úsalo cuando:

1. Necesitas **audit trail** completo (bancos, healthcare)
2. Tu dominio es **naturalmente event-driven** (trading, booking)
3. Necesitas **escalar reads y writes** independientemente
4. Requieres **debugging** profundo de comportamiento histórico

Evítalo cuando:
- Tu aplicación es muy simple (CRUD básico)
- No puedes tolerar eventual consistency
- El equipo no tiene experiencia con eventos

En el próximo módulo exploraremos **Saga Patterns**: cómo manejar transacciones distribuidas en microservicios usando eventos.

¡Hasta la próxima!"

**[PANTALLA: Recursos]**

- 📚 Ejercicios: Implementar sistema bancario con Event Sourcing
- 💡 Proyecto: Inventario con CQRS + Snapshots
- 🔗 Recursos: EventStoreDB, Axon Framework
- ▶️ Próximo: Saga Patterns (Choreography vs Orchestration)

---

## Recursos Visuales

### Animaciones Clave:
1. **Event Replay**: Eventos aplicándose secuencialmente
2. **CQRS Architecture**: Separación write/read con projections
3. **Snapshots**: Proceso de creación y carga
4. **Multiple Projections**: Mismo stream → diferentes vistas

### Código a Mostrar:
- TypeScript: Order aggregate con fromEvents
- Python: Projections para diferentes read models
- Java: EventStore con optimistic concurrency
- Kotlin: SOLID en Event Sourcing

### Demos en Vivo:
1. Reconstruir aggregate desde eventos
2. Benchmark snapshots (con/sin)
3. Múltiples projections del mismo evento
4. Time travel: rebuild estado del pasado

### Diagramas:
1. Traditional DB vs Event Store
2. CQRS architecture completa
3. Snapshot loading process
4. Event evolution (v1 → v2 upcasting)

## Notas para el Editor

- Resaltar visualmente cuando eventos se aplican (apply)
- Usar colores diferentes para Write Side (azul) vs Read Side (verde)
- Animación smooth para replay de eventos
- Mostrar timestamps en eventos para enfatizar historial
- Performance benchmarks con números grandes y claros
- Transiciones fluidas entre conceptos relacionados

## B-Roll Sugerido

- Código de repositorios reales (GitHub, EventStoreDB)
- Dashboards mostrando event streams
- Gráficos de performance (snapshots vs sin snapshots)
- Arquitecturas de sistemas reales usando Event Sourcing
