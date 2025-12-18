# Ejercicios: Violaciones if/else/switch

## ⭐ Nivel 1
Identifica violaciones de OCP:
```java
public double getPrice(Product product) {
    if (product.getCategory().equals("BOOK")) return 9.99;
    else if (product.getCategory().equals("ELECTRONICS")) return 299.99;
    else if (product.getCategory().equals("CLOTHING")) return 49.99;
}
```

## ⭐⭐ Nivel 2
Refactoriza usando polimorfismo:
```java
public void sendNotification(String type, String message) {
    if (type.equals("EMAIL")) sendEmail(message);
    else if (type.equals("SMS")) sendSMS(message);
    else if (type.equals("PUSH")) sendPush(message);
}
```

## ⭐⭐⭐ Nivel 3
Sistema de descuentos con 5 tipos (NoDiscount, Percentage, Fixed, BuyOneGetOne, Seasonal). Elimina todos los if/else usando Strategy.

## ⭐⭐⭐⭐ Nivel 4
Analiza proyecto legacy, identifica todos los switch/if-else sobre tipos con PMD/SonarQube. Refactoriza 3 casos complejos, mide mejora de complejidad ciclomática.
