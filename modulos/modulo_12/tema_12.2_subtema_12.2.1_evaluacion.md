# Evaluación: CQRS Pattern

## Parte 1. Conceptos Fundamentales (20 puntos)

### Pregunta 1 (7 puntos)
Explica qué problema resuelve CQRS y por qué no es apropiado para todas las aplicaciones.

**Criterios**:
- Problema: requisitos diferentes para read/write
- Beneficios: escalabilidad, optimización independiente
- Trade-offs: complejidad, eventual consistency
- Cuándo NO usarlo (CRUD simple, strong consistency requerida)

---

### Pregunta 2 (7 puntos)
Compara estos dos enfoques. ¿Cuál sigue CQRS correctamente?

**Opción A**:
```java
class OrderService {
    Order createOrder(OrderDto dto) {
        Order order = new Order(dto);
        repository.save(order);
        return order;  // Retorna entidad
    }
    
    List<Order> getOrders(String customerId) {
        return repository.findByCustomerId(customerId);
    }
}
```

**Opción B**:
```java
class OrderCommandHandler {
    String handle(CreateOrderCommand cmd) {
        // ...
        return orderId;  // Solo ID
    }
}

class OrderQueryService {
    OrderView getOrder(String id) {
        return readDb.findById(id);  // Desde read model
    }
}
```

**Criterios**:
- Identificar que B sigue CQRS
- Explicar por qué A no separa read/write
- Mencionar que commands retornan void/ID, no entidades

---

### Pregunta 3 (6 puntos)
¿Qué es eventual consistency en CQRS? ¿Cómo manejarías un caso donde el usuario crea una orden y quiere verla inmediatamente?

**Criterios**:
- Definición: read models se actualizan asíncronamente
- Soluciones: correlation IDs, polling, WebSockets, wait patterns
- Trade-off entre UX y complejidad

---

## Parte 2: Diseño de Arquitectura (30 puntos)

### Ejercicio 1 (15 puntos)

Diseña arquitectura CQRS para sistema de reservas de hotel con estos requisitos:

**Funcionalidades**:
1. Crear reserva (validar disponibilidad)
2. Cancelar reserva
3. Ver disponibilidad de habitaciones (query frecuente)
4. Ver historial de reservas del cliente
5. Dashboard del hotel (ocupación, ingresos)

**Tareas**:
- Definir Commands y Events
- Diseñar al menos 2 read models diferentes
- Explicar qué DB usarías para write/read (justificar)

**Rúbrica**:
| Criterio | Puntos |
|----------|--------|
| Commands bien definidos | 3 |
| Events correctos | 3 |
| Read models optimizados | 4 |
| Elección de DBs justificada | 3 |
| Separación clara write/read | 2 |

---

### Ejercicio 2 (15 puntos)

Este código mezcla concerns. Refactorízalo usando CQRS:

```python
class ProductService:
    def create_product(self, name: str, price: float) -> Product:
        product = Product(name=name, price=price)
        self.db.save(product)
        
        # Send email
        self.email.send(f"Product {name} created")
        
        # Update analytics
        self.analytics.track("product_created", product.id)
        
        return product
    
    def get_products_with_sales(self, category: str) -> List[Dict]:
        # Complex query with joins
        return self.db.query("""
            SELECT p.*, COUNT(s.id) as sales_count
            FROM products p
            LEFT JOIN sales s ON p.id = s.product_id
            WHERE p.category = ?
            GROUP BY p.id
        """, category)
```

**Solución esperada**:
- Separar command handler y event handler
- Crear projection para ProductSalesView
- Eliminar joins en queries (denormalizar)

---

## Parte 3: Implementación (40 puntos)

### Proyecto: Sistema de Tickets de Soporte con CQRS

Implementa sistema de tickets con:

**Commands**:
- `CreateTicketCommand(customerId, subject, description, priority)`
- `AssignTicketCommand(ticketId, agentId)`
- `ResolveTicketCommand(ticketId, resolution)`
- `AddCommentCommand(ticketId, userId, comment)`

**Events**:
- `TicketCreatedEvent`
- `TicketAssignedEvent`
- `TicketResolvedEvent`
- `CommentAddedEvent`

**Read Models** (3 requeridos):

1. **TicketListView**:
   - Para listar tickets con filtros (status, priority, agent)
   - Campos: ticketId, subject, customerName, agentName, status, priority, createdAt

2. **TicketDetailView**:
   - Para mostrar ticket completo con comentarios
   - Campos: todos los campos + comments[], attachments[]

3. **AgentDashboardView**:
   - Para dashboard del agente
   - Campos: agentId, assignedTickets, resolvedToday, avgResolutionTime

**Requisitos Técnicos**:

1. **Write Side**:
   ```typescript
   class Ticket {
       assign(command: AssignTicketCommand): TicketAssignedEvent {
           // Business rules:
           // - Cannot assign if already resolved
           // - Cannot assign to non-existent agent
       }
   }
   ```

2. **Projections** (implementar las 3):
   ```typescript
   class TicketListProjection {
       async on(event: TicketCreatedEvent): Promise<void> {
           // Insert denormalized view
       }
       
       async on(event: TicketAssignedEvent): Promise<void> {
           // Update agent name
       }
   }
   ```

3. **Query Services**:
   ```typescript
   class TicketQueryService {
       async listTickets(filters: TicketFilters): Promise<TicketListView[]> {
           // Optimized query from read model
       }
       
       async getTicketDetail(ticketId: string): Promise<TicketDetailView> {
           // Single query, no joins
       }
   }
   ```

4. **Tests**:
   ```typescript
   describe('Ticket CQRS', () => {
       it('should update all projections when ticket created');
       it('should prevent assigning resolved ticket');
       it('should query tickets by status efficiently');
       it('should calculate agent stats correctly');
   });
   ```

#### Casos de Prueba Obligatorios:

```typescript
it('should create ticket and update list view', async () => {
    const ticketId = await commandHandler.handle(new CreateTicketCommand(
        'cust-1', 'Login issue', 'Cannot login...', 'HIGH'
    ));
    
    await waitForProjection();
    
    const tickets = await queryService.listTickets({ status: 'OPEN' });
    expect(tickets).toHaveLength(1);
    expect(tickets[0].priority).toBe('HIGH');
});

it('should update agent dashboard when ticket assigned', async () => {
    const ticketId = await createTicket();
    
    await commandHandler.handle(new AssignTicketCommand(ticketId, 'agent-1'));
    
    await waitForProjection();
    
    const dashboard = await queryService.getAgentDashboard('agent-1');
    expect(dashboard.assignedTickets).toBe(1);
});

it('should include all comments in detail view', async () => {
    const ticketId = await createTicket();
    
    await commandHandler.handle(new AddCommentCommand(ticketId, 'user-1', 'Comment 1'));
    await commandHandler.handle(new AddCommentCommand(ticketId, 'user-2', 'Comment 2'));
    
    await waitForProjection();
    
    const detail = await queryService.getTicketDetail(ticketId);
    expect(detail.comments).toHaveLength(2);
});
```

#### Rúbrica (40 puntos):

| Criterio | Puntos | Descripción |
|----------|--------|-------------|
| **Commands/Events** | 8 | Commands inmutables, Events con datos completos |
| **Write Model** | 8 | Aggregate con validaciones, métodos retornan eventos |
| **Projections** | 10 | 3 projections correctas, idempotentes |
| **Query Services** | 6 | Queries optimizadas, sin joins |
| **SOLID** | 4 | SRP en handlers, DIP con interfaces |
| **Tests** | 4 | Cobertura >80%, tests de integración |

---

## Parte 4: Análisis de Performance (10 puntos)

### Escenario Real

Tu sistema de tickets procesa 100,000 tickets/mes. Queries frecuentes:
- Listar tickets del agente (50 req/s)
- Ver detalle de ticket (20 req/s)
- Dashboard del agente (10 req/s)

**Preguntas**:

1. **(3 puntos)** ¿Por qué CQRS mejora performance aquí vs arquitectura tradicional con joins?

   **Respuesta esperada**:
   - Tradicional: JOIN entre tickets, agents, comments → lento
   - CQRS: Datos denormalizados, query simple sin joins → 10x más rápido
   - Read DB puede tener índices específicos para cada query

2. **(3 puntos)** ¿Qué DB usarías para cada read model? Justifica.

   **Respuesta esperada**:
   - TicketListView: PostgreSQL (queries estructuradas, índices)
   - TicketDetailView: MongoDB (documento denormalizado)
   - AgentDashboardView: Redis (agregaciones pre-calculadas, cache)

3. **(4 puntos)** Un agente asigna ticket pero no aparece en su lista inmediatamente. ¿Cómo manejas esto en la UI?

   **Respuesta esperada**:
   - **Opción 1**: Optimistic UI (mostrar inmediatamente, sincronizar después)
   - **Opción 2**: Correlation ID + polling cada 500ms
   - **Opción 3**: WebSocket notification cuando projection completa
   - **Trade-off**: UX vs complejidad técnica

---

## Bonus: Múltiples Read Models (10 puntos extra)

Implementa un 4º read model: **CustomerTicketHistoryView**

```typescript
interface CustomerTicketHistoryView {
    customerId: string;
    customerName: string;
    totalTickets: number;
    openTickets: number;
    resolvedTickets: number;
    averageResolutionTime: number;  // En horas
    lastTicketDate: Date;
    commonIssues: string[];  // Top 3 categorías
}
```

**Requisitos**:
- Proyección que calcula todas las agregaciones
- Query eficiente (sin cálculos en runtime)
- Tests verificando cálculos correctos

---

## Criterios de Aprobación

- **0-59 puntos**: Reprobado
- **60-69 puntos**: Aprobado (C)
- **70-79 puntos**: Bueno (B)
- **80-89 puntos**: Muy Bueno (A)
- **90-110 puntos**: Excelente (A+)

## Entregables

1. Código completo del sistema de tickets
2. Tests con >80% cobertura
3. Documento con respuestas a preguntas teóricas
4. Diagrama de arquitectura CQRS
5. README con decisiones de diseño

**Tiempo estimado**: 8-10 horas

**Formato**: Repositorio Git con estructura clara
