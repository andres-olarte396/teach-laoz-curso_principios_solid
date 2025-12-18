# Catálogo de Refactorings

## Extract Method (SRP)

```java
// Antes
void printOwing() {
    printBanner();
    
    // Print details
    System.out.println("name: " + name);
    System.out.println("amount: " + getOutstanding());
}

// Después
void printOwing() {
    printBanner();
    printDetails(getOutstanding());
}

void printDetails(double outstanding) {
    System.out.println("name: " + name);
    System.out.println("amount: " + outstanding);
}
```

## Replace Conditional with Polymorphism (OCP)

```java
// Antes
double getSpeed() {
    switch (type) {
        case EUROPEAN: return baseSpeed;
        case AFRICAN: return baseSpeed - loadFactor;
        case NORWEGIAN: return (isNailed) ? 0 : baseSpeed;
    }
}

// Después
abstract class Bird {
    abstract double getSpeed();
}

class European extends Bird {
    double getSpeed() { return baseSpeed; }
}

class African extends Bird {
    double getSpeed() { return baseSpeed - loadFactor; }
}
```

## Replace Inheritance with Delegation (LSP)

```java
// Antes
class Stack extends ArrayList { /* ... */ }

// Después
class Stack {
    private List items = new ArrayList();
    
    void push(Object o) { items.add(o); }
    Object pop() { return items.remove(items.size() - 1); }
}
```

## Extract Interface (ISP + DIP)

```java
// Antes
class Client {
    private ConcreteService service;
}

// Después
interface Service {
    void operation();
}

class Client {
    private Service service; // Abstracción
}
```

## Resumen

**Refactorings** = Transformaciones que mejoran diseño aplicando SOLID.
