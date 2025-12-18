# Integración SOLID: Aplicación de Múltiples Principios

## Los 5 Principios Trabajando Juntos

### SRP + DIP: Responsabilidades claras con abstracciones

```java
// ✅ SRP: Cada clase una responsabilidad
interface OrderRepository { void save(Order order); }
interface PaymentProcessor { PaymentResult process(Payment payment); }
interface EmailService { void send(String to, String message); }

// ✅ DIP: Depender de abstracciones
class OrderService {
    private final OrderRepository repository;
    private final PaymentProcessor paymentProcessor;
    private final EmailService emailService;
    
    public OrderService(OrderRepository repository,
                       PaymentProcessor paymentProcessor,
                       EmailService emailService) {
        this.repository = repository;
        this.paymentProcessor = paymentProcessor;
        this.emailService = emailService;
    }
    
    public void placeOrder(Order order) {
        PaymentResult result = paymentProcessor.process(order.getPayment());
        if (result.isSuccess()) {
            repository.save(order);
            emailService.send(order.getCustomerEmail(), "Order confirmed");
        }
    }
}
```

### OCP + LSP: Extensibilidad con sustitución segura

```java
// ✅ OCP: Abierto a extensión
interface DiscountStrategy {
    double calculate(double amount);
}

// ✅ LSP: Todos los subtipos sustituibles
class PercentageDiscount implements DiscountStrategy {
    public double calculate(double amount) {
        return amount * 0.9; // Siempre retorna valor positivo
    }
}

class FixedDiscount implements DiscountStrategy {
    public double calculate(double amount) {
        return Math.max(0, amount - 10); // Garantiza >= 0
    }
}
```

### ISP + DIP: Interfaces mínimas con inversión

```java
// ✅ ISP: Interfaces segregadas
interface OrderCreator { void create(Order order); }
interface OrderFinder { Order findById(Long id); }
interface OrderCanceller { void cancel(Long id); }

// ✅ DIP: Clientes dependen solo de lo necesario
class OrderController {
    private final OrderCreator creator;
    private final OrderCanceller canceller;
    
    // No depende de OrderFinder (ISP + DIP)
}

class ReportService {
    private final OrderFinder finder;
    
    // Solo lectura (ISP + DIP)
}
```

## Ejemplo Completo: Sistema de Pedidos

```java
// DOMAIN (SRP + LSP)
class Order {
    private OrderStatus status;
    private List<OrderItem> items;
    
    public void complete() { // Una responsabilidad
        if (!canComplete()) throw new IllegalStateException();
        status = OrderStatus.COMPLETED;
    }
}

// ABSTRACCIONES (DIP + ISP)
interface OrderRepository { void save(Order order); }
interface PaymentGateway { PaymentResult charge(Payment payment); }
interface NotificationService { void notify(String message); }

// USE CASE (SRP + DIP)
class CompleteOrderUseCase {
    private final OrderRepository repository;
    private final PaymentGateway gateway;
    private final NotificationService notifications;
    
    public void execute(Long orderId) { // Una responsabilidad
        Order order = repository.findById(orderId);
        PaymentResult result = gateway.charge(order.getPayment());
        
        if (result.isSuccess()) {
            order.complete();
            repository.save(order);
            notifications.notify("Order " + orderId + " completed");
        }
    }
}

// IMPLEMENTACIONES (OCP + LSP)
class StripePaymentGateway implements PaymentGateway {
    public PaymentResult charge(Payment payment) {
        // Implementación Stripe (sustituible)
    }
}

class EmailNotificationService implements NotificationService {
    public void notify(String message) {
        // Envío email (sustituible)
    }
}

// CONFIGURACIÓN (DIP)
@Configuration
class AppConfig {
    @Bean
    public PaymentGateway paymentGateway() {
        return new StripePaymentGateway(); // Fácil cambiar
    }
    
    @Bean
    public NotificationService notificationService() {
        return new EmailNotificationService();
    }
}
```

## Resumen

**SOLID integrado** = Principios se complementan para código flexible, mantenible y testeable.
