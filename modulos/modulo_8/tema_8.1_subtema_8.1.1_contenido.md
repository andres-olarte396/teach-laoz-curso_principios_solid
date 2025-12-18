# Code Smells y SOLID

## Long Method

**Viola**: SRP

```java
// ❌ Método largo (múltiples responsabilidades)
void processOrder(Order order) {
    // Validación (20 líneas)
    // Cálculo de precio (30 líneas)
    // Persistencia (15 líneas)
    // Notificación (10 líneas)
}

// ✅ Extraer métodos
void processOrder(Order order) {
    validate(order);
    double total = calculateTotal(order);
    save(order);
    notifyCustomer(order);
}
```

## Large Class

**Viola**: SRP, ISP

```java
// ❌ Clase grande (God Object)
class OrderManager {
    // 50+ campos
    // 100+ métodos
}

// ✅ Segregar responsabilidades
class OrderService { /* ... */ }
class OrderValidator { /* ... */ }
class OrderRepository { /* ... */ }
```

## Feature Envy

**Viola**: SRP (método usa más datos de otra clase)

```java
// ❌ Feature Envy
class OrderService {
    double calculateTotal(Order order) {
        double total = 0;
        for (Item item : order.getItems()) {
            total += item.getPrice() * item.getQuantity();
        }
        return total;
    }
}

// ✅ Mover lógica a Order
class Order {
    double calculateTotal() {
        return items.stream()
            .mapToDouble(item -> item.getPrice() * item.getQuantity())
            .sum();
    }
}
```

## Shotgun Surgery

**Viola**: OCP (un cambio requiere tocar muchas clases)

```java
// ❌ Cambiar tasa de impuesto requiere modificar 10 clases
class OrderCalculator { double tax = 0.19; }
class InvoiceGenerator { double tax = 0.19; }
class ReportBuilder { double tax = 0.19; }

// ✅ Centralizar
interface TaxConfiguration {
    double getTaxRate();
}
```

## Resumen

**Code Smells** = Señales de violaciones SOLID que requieren refactoring.
