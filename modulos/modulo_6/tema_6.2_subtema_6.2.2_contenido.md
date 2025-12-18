# Service Locator Pattern

## Concepto

**Service Locator**: Registro central de servicios, clientes buscan dependencias.

## Implementación

```java
class ServiceLocator {
    private static Map<Class<?>, Object> services = new HashMap<>();
    
    public static <T> void register(Class<T> type, T implementation) {
        services.put(type, implementation);
    }
    
    public static <T> T get(Class<T> type) {
        return (T) services.get(type);
    }
}

// Registro
ServiceLocator.register(OrderRepository.class, new MySQLOrderRepository());
ServiceLocator.register(EmailService.class, new SMTPEmailService());

// Uso
class OrderService {
    public void processOrder(Order order) {
        OrderRepository repo = ServiceLocator.get(OrderRepository.class);
        EmailService email = ServiceLocator.get(EmailService.class);
        
        repo.save(order);
        email.send(order.getCustomerEmail(), "Order processed");
    }
}
```

## Service Locator vs DI

| Aspecto | Service Locator | Dependency Injection |
|---------|----------------|---------------------|
| Dependencias | Ocultas | Explícitas |
| Testing | Más difícil | Más fácil |
| Acoplamiento | Alto (al locator) | Bajo |
| Recomendación | ❌ Evitar | ✅ Preferir |

## Cuándo Usar Service Locator

- Frameworks legacy sin DI
- Código de terceros sin control

**Recomendación general**: Preferir DI sobre Service Locator.

## Resumen

**Service Locator** = Registro central de servicios. Menos preferido que DI.
