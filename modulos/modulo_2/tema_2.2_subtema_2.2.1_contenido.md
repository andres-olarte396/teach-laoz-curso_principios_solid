# God Classes y Feature Envy

## Introducción

Las **God Classes** (Clases Dios) y el **Feature Envy** (Envidia de Características) son dos de los code smells más comunes que violan el Single Responsibility Principle. Estos antipatrones indican diseño centralizado y acoplamiento excesivo.

## God Class (Clase Dios)

### Definición

Una **God Class** es una clase que:
- Conoce demasiado sobre el sistema
- Hace demasiado trabajo
- Centraliza funcionalidad que debería estar distribuida
- Tiene muy alta cohesión interna pero baja cohesión con el dominio

### Características

🚨 **Señales de una God Class**:
- \>500 líneas de código
- \>20 métodos públicos
- \>10 variables de instancia
- \>5 dependencias inyectadas
- Nombre genérico: Manager, Controller, Handler, Service
- Tests extremadamente complejos

### Ejemplo Clásico

```java
public class OrderManager {
    private Database database;
    private EmailService emailService;
    private PaymentGateway paymentGateway;
    private InventoryService inventoryService;
    private ShippingCalculator shippingCalculator;
    private TaxCalculator taxCalculator;
    private CouponValidator couponValidator;
    private LoyaltyPointsCalculator loyaltyCalculator;
    private InvoiceGenerator invoiceGenerator;
    private Analytics analytics;
    
    // 50+ métodos...
    public void createOrder(...) { }
    public void updateOrder(...) { }
    public void cancelOrder(...) { }
    public void processPayment(...) { }
    public void refundPayment(...) { }
    public void calculateTax(...) { }
    public void calculateShipping(...) { }
    public void validateCoupon(...) { }
    public void applyDiscount(...) { }
    public void updateInventory(...) { }
    public void reserveStock(...) { }
    public void releaseStock(...) { }
    public void sendConfirmation(...) { }
    public void sendInvoice(...) { }
    public void sendShippingNotification(...) { }
    public void calculateLoyaltyPoints(...) { }
    public void generateInvoice(...) { }
    public void trackOrder(...) { }
    public void exportOrders(...) { }
    // ... 30 métodos más
}
```

**Problemas**:
- **Modificación riesgosa**: Cualquier cambio afecta múltiples funcionalidades
- **Testing imposible**: Requiere mockear 10+ dependencias
- **Merge conflicts**: Múltiples desarrolladores modificando la misma clase
- **Comprensión difícil**: Nadie entiende todo lo que hace

### Refactorización: Descomponer God Class

**Estrategia**: Identificar cohesión y extraer clases especializadas.

```java
// Clase de dominio (solo datos)
public class Order {
    private Long id;
    private Customer customer;
    private List<OrderItem> items;
    private OrderStatus status;
    // Getters/setters
}

// Repositorio (persistencia)
public class OrderRepository {
    private Database database;
    public void save(Order order) { }
    public Order findById(Long id) { }
}

// Servicio de cálculo
public class OrderPricingService {
    private TaxCalculator taxCalculator;
    private ShippingCalculator shippingCalculator;
    
    public OrderPrice calculate(Order order) {
        double subtotal = calculateSubtotal(order);
        double tax = taxCalculator.calculate(subtotal);
        double shipping = shippingCalculator.calculate(order);
        return new OrderPrice(subtotal, tax, shipping);
    }
}

// Servicio de descuentos
public class DiscountService {
    private CouponValidator couponValidator;
    public double applyDiscount(Order order, Coupon coupon) { }
}

// Servicio de pago
public class PaymentService {
    private PaymentGateway paymentGateway;
    public PaymentResult processPayment(Order order) { }
    public void refund(String transactionId) { }
}

// Servicio de inventario
public class InventoryService {
    private InventoryRepository inventoryRepository;
    public void reserveStock(Order order) { }
    public void releaseStock(Order order) { }
}

// Servicio de notificaciones
public class OrderNotificationService {
    private EmailService emailService;
    public void sendConfirmation(Order order) { }
    public void sendInvoice(Order order) { }
    public void sendShippingNotification(Order order) { }
}

// Generador de facturas
public class InvoiceGenerator {
    public Invoice generate(Order order) { }
}

// Calculador de puntos
public class LoyaltyPointsService {
    public int calculatePoints(Order order) { }
}

// Analytics
public class OrderAnalytics {
    public void track(Order order, String event) { }
}

// Orquestador (ahora simple)
public class OrderService {
    private final OrderRepository orderRepository;
    private final OrderPricingService pricingService;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final OrderNotificationService notificationService;
    
    public void processOrder(Order order) {
        OrderPrice price = pricingService.calculate(order);
        order.setTotal(price.getTotal());
        
        PaymentResult payment = paymentService.processPayment(order);
        order.setPaymentId(payment.getTransactionId());
        
        inventoryService.reserveStock(order);
        orderRepository.save(order);
        notificationService.sendConfirmation(order);
    }
}
```

**Mejoras**:
- De 1 clase de 2000 LOC → 10 clases de 50-200 LOC
- Cada clase testeable independientemente
- Cambios localizados
- Reutilización mejorada

## Feature Envy (Envidia de Características)

### Definición

**Feature Envy** ocurre cuando un método está más interesado en otra clase que en la propia. El método "envidia" los datos o métodos de otra clase.

### Señales

```java
// ❌ Feature Envy evidente
public class ShoppingCart {
    private List<Product> products;
    
    // Este método "envidia" Product
    public double calculateTotalPrice() {
        double total = 0;
        for (Product product : products) {
            // Accede repetidamente a Product
            double basePrice = product.getBasePrice();
            double discount = product.getDiscount();
            double tax = product.getTaxRate();
            double finalPrice = basePrice * (1 - discount) * (1 + tax);
            total += finalPrice;
        }
        return total;
    }
}
```

**Problema**: El método accede múltiples veces a datos de `Product`. La lógica de cálculo de precio debería estar en `Product`.

### Refactorización: Move Method

```java
// ✅ Solución: Mover lógica a Product
public class Product {
    private double basePrice;
    private double discount;
    private double taxRate;
    
    // Método movido aquí
    public double calculateFinalPrice() {
        return basePrice * (1 - discount) * (1 + taxRate);
    }
}

public class ShoppingCart {
    private List<Product> products;
    
    public double calculateTotalPrice() {
        return products.stream()
            .mapToDouble(Product::calculateFinalPrice)
            .sum();
    }
}
```

### Casos Avanzados de Feature Envy

#### Caso 1. Envidia de Múltiples Clases

```java
// ❌ Envidia de Customer y Order
public class ReportGenerator {
    public String generateCustomerOrderReport(Customer customer, Order order) {
        String report = "";
        report += "Customer: " + customer.getName() + "\n";
        report += "Email: " + customer.getEmail() + "\n";
        report += "Phone: " + customer.getPhone() + "\n";
        report += "Order ID: " + order.getId() + "\n";
        report += "Total: $" + order.getTotal() + "\n";
        report += "Status: " + order.getStatus() + "\n";
        return report;
    }
}
```

**Solución**: Extract Method + Delegation

```java
// ✅ Cada clase conoce su propia representación
public class Customer {
    public String toReportFormat() {
        return String.format("Customer: %s\nEmail: %s\nPhone: %s\n",
                           name, email, phone);
    }
}

public class Order {
    public String toReportFormat() {
        return String.format("Order ID: %s\nTotal: $%.2f\nStatus: %s\n",
                           id, total, status);
    }
}

public class ReportGenerator {
    public String generateReport(Customer customer, Order order) {
        return customer.toReportFormat() + order.toReportFormat();
    }
}
```

#### Caso 2: Envidia en Cadena (Law of Demeter)

```java
// ❌ Violación del Law of Demeter
public class OrderProcessor {
    public void processOrder(Order order) {
        String city = order.getCustomer().getAddress().getCity();
        double tax = taxCalculator.getTaxRate(city);
        // ...
    }
}
```

**Problema**: Cadena `order.getCustomer().getAddress().getCity()` indica Feature Envy.

**Solución**: Tell, Don't Ask

```java
// ✅ Encapsular navegación
public class Order {
    public String getCustomerCity() {
        return customer.getAddress().getCity();
    }
}

public class OrderProcessor {
    public void processOrder(Order order) {
        String city = order.getCustomerCity();
        double tax = taxCalculator.getTaxRate(city);
    }
}
```

## Relación entre God Class y Feature Envy

Frecuentemente, ambos antipatrones coexisten:

```java
// God Class con Feature Envy
public class UserManager {
    // Envidia de User
    public void updateUserProfile(User user, String newName, String newEmail) {
        user.setName(newName);
        user.setEmail(newEmail);
        user.setLastModified(LocalDateTime.now());
        validateEmail(user.getEmail());
        database.update(user);
    }
    
    // Envidia de Order
    public double calculateUserOrderTotal(User user) {
        List<Order> orders = database.findOrdersByUser(user.getId());
        double total = 0;
        for (Order order : orders) {
            total += order.getSubtotal() + order.getTax() + order.getShipping();
        }
        return total;
    }
}
```

**Refactorización combinada**:

```java
// Lógica movida a User
public class User {
    public void updateProfile(String newName, String newEmail) {
        this.name = newName;
        this.email = newEmail;
        this.lastModified = LocalDateTime.now();
    }
}

// Lógica movida a Order
public class Order {
    public double getTotalWithTaxAndShipping() {
        return subtotal + tax + shipping;
    }
}

// Servicio simplificado
public class UserService {
    public void updateUserProfile(User user, String name, String email) {
        user.updateProfile(name, email);
        userRepository.save(user);
    }
}

public class OrderService {
    public double calculateUserTotal(User user) {
        return orderRepository.findByUser(user).stream()
            .mapToDouble(Order::getTotalWithTaxAndShipping)
            .sum();
    }
}
```

## Detección Automatizada

### Herramientas

**PMD**:
```xml
<rule ref="category/java/design.xml/GodClass"/>
<rule ref="category/java/design.xml/LawOfDemeter"/>
```

**SonarQube**:
- Rule S1448: God Class
- Rule S3776: Cognitive Complexity (indicador de God Class)

### Métricas

| Métrica | God Class | Valor Saludable |
|---------|-----------|-----------------|
| LOC | >500 | <200 |
| Métodos públicos | >20 | <10 |
| Variables de instancia | >10 | <7 |
| Dependencias | >5 | <3 |
| LCOM | >0.8 | <0.5 |
| Complejidad ciclomática | >50 | <15 |

## Estrategias de Prevención

### 1. Code Reviews Enfocados

**Checklist**:
- [ ] ¿La clase hace más de una cosa?
- [ ] ¿El método accede múltiples veces a otra clase?
- [ ] ¿Hay cadenas de llamadas (a.b().c().d())?
- [ ] ¿Puedo describir la responsabilidad en una oración?

### 2. Tests como Indicador

```java
// Test complejo = Posible God Class
@Test
public void testUserManager() {
    Database mockDb = mock(Database.class);
    EmailService mockEmail = mock(EmailService.class);
    SMSService mockSMS = mock(SMSService.class);
    // 7+ mocks → ALERTA
}
```

### 3. Límites Arquitectónicos

```java
// Establecer límites claros
@MaxDependencies(3)
@MaxMethods(10)
@MaxLines(200)
public class OrderService {
    // Si excedes límites → Build falla
}
```

## Resumen

| Antipatrón | Señal | Refactorización |
|------------|-------|-----------------|
| **God Class** | Muchas responsabilidades | Extract Class |
| **Feature Envy** | Acceso excesivo a otra clase | Move Method |

**Regla de oro**: Si un método usa más datos de otra clase que de la suya, probablemente debería estar en esa clase.
