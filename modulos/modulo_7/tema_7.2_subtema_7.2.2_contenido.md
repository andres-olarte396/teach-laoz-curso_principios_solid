# Anti-patrones que Violan SOLID

## God Object

**Viola**: SRP, OCP, ISP

```java
// ❌ God Object
class ApplicationManager {
    void handleUserRequest() {}
    void processPayment() {}
    void sendEmail() {}
    void generateReport() {}
    void validateInput() {}
    void logActivity() {}
    void manageDatabase() {}
    // ... 50 métodos más
}
```

## Spaghetti Code

**Viola**: Todos los principios

```java
// ❌ Sin separación de responsabilidades
void processOrder() {
    // Validación
    if (order.items.size() == 0) return;
    
    // Cálculo
    double total = 0;
    for (Item item : order.items) {
        total += item.price * item.quantity;
    }
    
    // Persistencia
    Connection conn = DriverManager.getConnection("...");
    Statement stmt = conn.createStatement();
    stmt.execute("INSERT INTO orders...");
    
    // Notificación
    SmtpClient smtp = new SmtpClient();
    smtp.send(order.customerEmail, "Order confirmed");
}
```

## Lava Flow

**Viola**: OCP (código muerto impide refactoring)

```java
// ❌ Código comentado que nadie toca
class LegacyService {
    void process() {
        // Old implementation (DO NOT REMOVE)
        // processOldWay();
        
        processNewWay();
        
        // TODO: cleanup old code
        // cleanupOldData();
    }
}
```

## Resumen

**Anti-patrones** = Violaciones sistemáticas de SOLID que degradan código.
