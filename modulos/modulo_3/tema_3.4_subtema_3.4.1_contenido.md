# if/else/switch como Violación de OCP

## El Problema

El uso excesivo de `if/else` y `switch/case` basados en tipos o categorías indica violación de OCP: cada nuevo tipo requiere modificar las estructuras condicionales existentes.

## Antipatrón: Type Checking

### ❌ Violación con if/else

```java
public class ShapeCalculator {
    public double calculateArea(Shape shape) {
        if (shape.getType().equals("CIRCLE")) {
            Circle circle = (Circle) shape;
            return Math.PI * circle.getRadius() * circle.getRadius();
        } else if (shape.getType().equals("RECTANGLE")) {
            Rectangle rect = (Rectangle) shape;
            return rect.getWidth() * rect.getHeight();
        } else if (shape.getType().equals("TRIANGLE")) {
            Triangle triangle = (Triangle) shape;
            return 0.5 * triangle.getBase() * triangle.getHeight();
        }
        throw new IllegalArgumentException("Unknown shape");
    }
}
```

**Problemas**:
- Añadir `Pentagon` requiere modificar `calculateArea`
- Viola SRP (múltiples razones para cambiar)
- Alto acoplamiento con tipos concretos
- Propenso a errores (olvidar un caso)

### ✅ Solución con Polimorfismo

```java
public interface Shape {
    double calculateArea();
}

public class Circle implements Shape {
    private double radius;
    
    public double calculateArea() {
        return Math.PI * radius * radius;
    }
}

public class Rectangle implements Shape {
    private double width;
    private double height;
    
    public double calculateArea() {
        return width * height;
    }
}

public class Triangle implements Shape {
    private double base;
    private double height;
    
    public double calculateArea() {
        return 0.5 * base * height;
    }
}

// No necesitas ShapeCalculator
Shape shape = getShape();
double area = shape.calculateArea(); // Polimorfismo
```

**Extensión**: Añadir `Pentagon` solo requiere crear la clase, sin tocar código existente.

## Antipatrón: Switch sobre Enums

### ❌ Violación con switch

```java
public enum OrderStatus {
    PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
}

public class OrderProcessor {
    public void processOrder(Order order) {
        switch (order.getStatus()) {
            case PENDING:
                validatePayment(order);
                break;
            case CONFIRMED:
                prepareForShipping(order);
                break;
            case SHIPPED:
                updateTracking(order);
                break;
            case DELIVERED:
                sendFeedbackRequest(order);
                break;
            case CANCELLED:
                refundPayment(order);
                break;
        }
    }
}
```

**Problema**: Añadir `OrderStatus.RETURNED` requiere modificar `processOrder`.

### ✅ Solución con State Pattern

```java
public interface OrderState {
    void process(Order order);
}

public class PendingState implements OrderState {
    public void process(Order order) {
        validatePayment(order);
        order.setState(new ConfirmedState());
    }
}

public class ConfirmedState implements OrderState {
    public void process(Order order) {
        prepareForShipping(order);
        order.setState(new ShippedState());
    }
}

public class ShippedState implements OrderState {
    public void process(Order order) {
        updateTracking(order);
        order.setState(new DeliveredState());
    }
}

public class Order {
    private OrderState state = new PendingState();
    
    public void setState(OrderState state) {
        this.state = state;
    }
    
    public void process() {
        state.process(this);
    }
}

// Uso
Order order = new Order();
order.process(); // PendingState ejecuta
order.process(); // ConfirmedState ejecuta
order.process(); // ShippedState ejecuta
```

**Extensión**: Añadir `ReturnedState` no requiere modificar estados existentes.

## Detección de Violaciones

### Heurística 1: Múltiples if/else Similares

```java
// ❌ MAL: Patrón repetido
public double calculateDiscount(Customer customer) {
    if (customer.getType().equals("REGULAR")) return 0;
    else if (customer.getType().equals("PREMIUM")) return 0.1;
    else if (customer.getType().equals("VIP")) return 0.2;
}

public String getGreeting(Customer customer) {
    if (customer.getType().equals("REGULAR")) return "Hello";
    else if (customer.getType().equals("PREMIUM")) return "Welcome";
    else if (customer.getType().equals("VIP")) return "Welcome VIP";
}
```

**Señal**: Mismo `if/else` en múltiples métodos → usar Strategy/Polymorphism.

### Heurística 2: if/else con instanceof

```java
// ❌ MAL
public void processPayment(Payment payment) {
    if (payment instanceof CreditCardPayment) {
        CreditCardPayment cc = (CreditCardPayment) payment;
        chargeCreditCard(cc.getCardNumber());
    } else if (payment instanceof PayPalPayment) {
        PayPalPayment pp = (PayPalPayment) payment;
        chargePayPal(pp.getEmail());
    }
}

// ✅ BIEN
public interface Payment {
    void process();
}

public class CreditCardPayment implements Payment {
    public void process() {
        chargeCreditCard(cardNumber);
    }
}
```

### Heurística 3: Switch Duplicado

```java
// ❌ MAL: Switch duplicado en múltiples clases
// En OrderProcessor
switch (order.getType()) { /* ... */ }

// En InvoiceGenerator
switch (order.getType()) { /* ... */ }

// En ShippingCalculator
switch (order.getType()) { /* ... */ }
```

**Solución**: Mover lógica a clases específicas por tipo de orden.

## Refactoring: Replace Conditional with Polymorphism

### Paso 1: Identificar Patrón

```java
if (type == A) { doA(); }
else if (type == B) { doB(); }
else if (type == C) { doC(); }
```

### Paso 2: Crear Jerarquía

```java
interface Handler {
    void handle();
}

class AHandler implements Handler { void handle() { doA(); } }
class BHandler implements Handler { void handle() { doB(); } }
class CHandler implements Handler { void handle() { doC(); } }
```

### Paso 3: Usar Polimorfismo

```java
Handler handler = getHandler(type);
handler.handle();
```

## Casos Donde if/else es Aceptable

### 1. Validación Simple

```java
public void setAge(int age) {
    if (age < 0 || age > 150) {
        throw new IllegalArgumentException("Invalid age");
    }
    this.age = age;
}
```

**OK**: Lógica de validación simple, poco probable que cambie.

### 2. Control de Flujo

```java
public void processOrder(Order order) {
    if (order.getItems().isEmpty()) {
        return; // Early return
    }
    // Procesar...
}
```

**OK**: Guardia de condición, no basada en tipos.

### 3. Configuración

```java
if (config.isDebugEnabled()) {
    log.debug("Processing order: " + order);
}
```

**OK**: Flag booleano simple.

## Métricas de Complejidad

### Complejidad Ciclomática

```bash
# Herramienta: SonarQube, PMD
if/else/switch contribuyen +1 cada uno
```

**Límite recomendado**: Complejidad < 10 por método.

### Code Smell: Long Method con if/else

```java
// ❌ MAL: 200 líneas con 15 if/else
public void processOrder(Order order) {
    if (...) { /* 20 líneas */ }
    else if (...) { /* 25 líneas */ }
    else if (...) { /* 30 líneas */ }
    // ... 10 más
}
```

**Refactoring**: Extract Method + Strategy Pattern.

## Resumen

**if/else/switch** basados en tipos = violación de OCP.

**Solución**: Polimorfismo, Strategy, State Pattern.

**Regla**: Si el condicional cambia frecuentemente o se duplica, refactoriza.
