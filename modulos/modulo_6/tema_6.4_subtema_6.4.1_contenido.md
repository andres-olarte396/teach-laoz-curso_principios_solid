# Arquitectura Hexagonal (Ports & Adapters)

## Concepto

**Hexagonal Architecture**: Aislar lógica de negocio de detalles técnicos mediante puertos y adaptadores.

```
         [UI/REST]           [Database]
             ↓                    ↓
         [Adapter]           [Adapter]
             ↓                    ↓
         [Port]              [Port]
             ↓                    ↓
        ┌─────────────────────────────┐
        │   CORE (Business Logic)     │
        │   - Domain Models           │
        │   - Use Cases               │
        │   - Business Rules          │
        └─────────────────────────────┘
```

## Estructura

```java
// CORE - Domain
class Order {
    private Long id;
    private List<OrderItem> items;
    private OrderStatus status;
    
    public void complete() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException();
        }
        status = OrderStatus.COMPLETED;
    }
}

// CORE - Port (abstracción)
interface OrderRepository {
    void save(Order order);
    Order findById(Long id);
}

interface NotificationService {
    void sendOrderConfirmation(Order order);
}

// CORE - Use Case
class CompleteOrderUseCase {
    private final OrderRepository repository;
    private final NotificationService notifications;
    
    public CompleteOrderUseCase(OrderRepository repository, 
                                 NotificationService notifications) {
        this.repository = repository;
        this.notifications = notifications;
    }
    
    public void execute(Long orderId) {
        Order order = repository.findById(orderId);
        order.complete();
        repository.save(order);
        notifications.sendOrderConfirmation(order);
    }
}

// ADAPTERS - Infraestructura
class MySQLOrderRepository implements OrderRepository {
    public void save(Order order) {
        // SQL implementation
    }
}

class SMTPNotificationService implements NotificationService {
    public void sendOrderConfirmation(Order order) {
        // SMTP implementation
    }
}

// ADAPTERS - UI
@RestController
class OrderController {
    private final CompleteOrderUseCase useCase;
    
    @PostMapping("/orders/{id}/complete")
    public void completeOrder(@PathVariable Long id) {
        useCase.execute(id);
    }
}
```

## Beneficios

1. **Testabilidad**: Core sin dependencias técnicas
2. **Flexibilidad**: Cambiar UI o DB sin afectar core
3. **DIP**: Core depende de abstracciones (ports)

## Resumen

**Hexagonal** = Core con puertos (abstracciones), adaptadores implementan detalles.
