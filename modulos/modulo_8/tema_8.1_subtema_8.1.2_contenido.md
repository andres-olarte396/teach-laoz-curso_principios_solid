# Violaciones Comunes de SOLID

## Violaciones SRP

```java
// ❌ Múltiples razones de cambio
class User {
    private String name;
    private String email;
    
    public void save() { /* DB logic */ }
    public void sendEmail() { /* Email logic */ }
    public String toJson() { /* Serialization */ }
}
```

## Violaciones OCP

```java
// ❌ Switch statement (cerrado a extensión)
double calculateDiscount(CustomerType type) {
    switch (type) {
        case REGULAR: return 0.05;
        case PREMIUM: return 0.10;
        case VIP: return 0.20;
    }
}
```

## Violaciones LSP

```java
// ❌ Subtipo rompe contrato
class Rectangle {
    void setWidth(int w) { width = w; }
    void setHeight(int h) { height = h; }
}

class Square extends Rectangle {
    void setWidth(int w) { width = height = w; } // Viola LSP
}
```

## Violaciones ISP

```java
// ❌ Fat interface
interface Worker {
    void work();
    void eat();
    void sleep();
    void attendMeeting();
}

class Robot implements Worker {
    void eat() { throw new UnsupportedOperationException(); }
}
```

## Violaciones DIP

```java
// ❌ Depende de concreción
class OrderService {
    private MySQLDatabase db = new MySQLDatabase(); // Acoplamiento directo
}
```

## Resumen

**Violaciones** = Patrones recurrentes que rompen principios SOLID.
