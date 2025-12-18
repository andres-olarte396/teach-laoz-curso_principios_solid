# Patrones de Diseño y SOLID

## Strategy Pattern

**Cumple**: OCP (extensible), SRP (algoritmos separados), DIP (depende de abstracción)

```java
interface PaymentStrategy { void pay(double amount); }
class CreditCardStrategy implements PaymentStrategy { /* ... */ }
class PayPalStrategy implements PaymentStrategy { /* ... */ }

class ShoppingCart {
    private PaymentStrategy paymentStrategy; // DIP
    
    public void checkout(double amount) {
        paymentStrategy.pay(amount); // OCP: agregar estrategias sin modificar
    }
}
```

## Factory Pattern

**Cumple**: OCP (nuevos productos sin modificar factory), DIP (retorna abstracciones)

```java
interface Product {}
class ConcreteProductA implements Product {}
class ConcreteProductB implements Product {}

abstract class Factory {
    abstract Product createProduct(); // OCP: subclases extienden
}
```

## Observer Pattern

**Cumple**: OCP (nuevos observadores), ISP (interfaces específicas), DIP

```java
interface Observer { void update(Event event); }
class Subject {
    private List<Observer> observers;
    
    public void attach(Observer observer) { observers.add(observer); }
    public void notifyObservers(Event event) {
        observers.forEach(o -> o.update(event));
    }
}
```

## Decorator Pattern

**Cumple**: OCP (añadir responsabilidades), LSP (decoradores sustituibles), SRP

```java
interface Component { void operation(); }
class ConcreteComponent implements Component {}

class Decorator implements Component {
    private Component wrapped;
    
    public void operation() {
        wrapped.operation();
        // Responsabilidad adicional
    }
}
```

## Resumen

**Patrones GoF** = Aplicaciones concretas de principios SOLID.
