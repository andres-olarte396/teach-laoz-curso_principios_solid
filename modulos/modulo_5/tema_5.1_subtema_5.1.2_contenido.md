# Role Interfaces

## Concepto

**Role Interface**: Interfaz que representa un rol o capacidad específica del cliente.

## Ejemplo

```java
// En lugar de una interfaz monolítica
interface IOrderService {
    void createOrder(Order order);
    void cancelOrder(Long id);
    List<Order> findOrders();
    void generateInvoice(Long orderId);
    void sendNotification(Long orderId);
}

// ✅ Interfaces por rol
interface OrderCreator {
    void createOrder(Order order);
}

interface OrderCanceller {
    void cancelOrder(Long id);
}

interface OrderFinder {
    List<Order> findOrders();
}

interface InvoiceGenerator {
    void generateInvoice(Long orderId);
}

// Implementación completa
class OrderService implements OrderCreator, OrderCanceller, OrderFinder, InvoiceGenerator {
    // Implementa todos los roles
}

// Clientes dependen solo de su rol
class OrderController {
    private OrderCreator orderCreator;
    private OrderCanceller orderCanceller;
    
    // Solo conoce crear y cancelar
}

class ReportingService {
    private OrderFinder orderFinder;
    
    // Solo conoce buscar
}
```

## Ventajas

1. **Acoplamiento mínimo**: Clientes conocen solo lo que necesitan
2. **Testing fácil**: Mocks pequeños
3. **Evolución**: Cambios en un rol no afectan otros

## Resumen

**Role Interfaces** = Diseñar interfaces según roles del cliente, no según implementación.
