# Definición: Subtipos Deben ser Sustituibles

## Principio de Sustitución de Liskov (Barbara Liskov, 1987)

> "Los objetos de una clase derivada deben poder sustituir a objetos de la clase base sin alterar el correcto funcionamiento del programa."

**Interpretación**: Si `S` es subtipo de `T`, entonces objetos de tipo `T` pueden ser reemplazados por objetos de tipo `S` sin romper el programa.

## Ejemplo Clásico: Problema Rectángulo-Cuadrado

### ❌ Violación de LSP

```java
class Rectangle {
    protected int width;
    protected int height;
    
    public void setWidth(int width) {
        this.width = width;
    }
    
    public void setHeight(int height) {
        this.height = height;
    }
    
    public int getArea() {
        return width * height;
    }
}

class Square extends Rectangle {
    @Override
    public void setWidth(int width) {
        this.width = width;
        this.height = width; // Mantener cuadrado
    }
    
    @Override
    public void setHeight(int height) {
        this.width = height;
        this.height = height; // Mantener cuadrado
    }
}

// Cliente que espera comportamiento de Rectangle
void testRectangle(Rectangle rect) {
    rect.setWidth(5);
    rect.setHeight(4);
    
    assert rect.getArea() == 20; // ¡Falla si rect es Square! (25 en lugar de 20)
}

testRectangle(new Rectangle()); // ✅ Pasa
testRectangle(new Square());    // ❌ Falla - Viola LSP
```

**Problema**: `Square` no es sustituible por `Rectangle` porque cambia invariantes (width y height independientes).

### ✅ Solución: Composición en lugar de Herencia

```java
interface Shape {
    int getArea();
}

class Rectangle implements Shape {
    private int width;
    private int height;
    
    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }
    
    public void setWidth(int width) { this.width = width; }
    public void setHeight(int height) { this.height = height; }
    
    public int getArea() {
        return width * height;
    }
}

class Square implements Shape {
    private int side;
    
    public Square(int side) {
        this.side = side;
    }
    
    public void setSide(int side) { this.side = side; }
    
    public int getArea() {
        return side * side;
    }
}

// Cliente trabaja con Shape
void printArea(Shape shape) {
    System.out.println("Area: " + shape.getArea());
}

printArea(new Rectangle(5, 4)); // ✅ 20
printArea(new Square(5));       // ✅ 25
```

## Contratos y Precondiciones/Postcondiciones

### Reglas de Sustitución (Design by Contract)

1. **Precondiciones**: No pueden ser más fuertes en subtipos
2. **Postcondiciones**: No pueden ser más débiles en subtipos
3. **Invariantes**: Deben mantenerse en subtipos

### Ejemplo: Violación de Precondiciones

```java
class BankAccount {
    protected double balance;
    
    // Precondición: amount > 0
    public void withdraw(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        if (balance < amount) {
            throw new IllegalArgumentException("Insufficient funds");
        }
        balance -= amount;
    }
}

class PremiumAccount extends BankAccount {
    @Override
    public void withdraw(double amount) {
        // ❌ Precondición más fuerte: amount > 100
        if (amount <= 100) {
            throw new IllegalArgumentException("Minimum withdrawal: $100");
        }
        super.withdraw(amount);
    }
}

// Cliente
void processWithdrawal(BankAccount account, double amount) {
    account.withdraw(amount); // Espera funcionar con amount > 0
}

processWithdrawal(new BankAccount(), 50);     // ✅ OK
processWithdrawal(new PremiumAccount(), 50);  // ❌ Falla - Viola LSP
```

**Solución**: No fortalecer precondiciones en subtipos.

```java
class PremiumAccount extends BankAccount {
    @Override
    public void withdraw(double amount) {
        // ✅ Misma precondición: amount > 0
        if (amount <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        
        // Lógica adicional sin cambiar contrato
        if (amount < 100) {
            double fee = 5.0;
            super.withdraw(amount + fee);
        } else {
            super.withdraw(amount);
        }
    }
}
```

## Invariantes de Clase

```java
class Stack<T> {
    private List<T> items = new ArrayList<>();
    
    // Invariante: size() >= 0
    public void push(T item) {
        items.add(item);
    }
    
    public T pop() {
        if (items.isEmpty()) {
            throw new EmptyStackException();
        }
        return items.remove(items.size() - 1);
    }
    
    public int size() {
        return items.size();
    }
}

class BoundedStack<T> extends Stack<T> {
    private int maxSize;
    
    public BoundedStack(int maxSize) {
        this.maxSize = maxSize;
    }
    
    @Override
    public void push(T item) {
        if (size() >= maxSize) {
            throw new IllegalStateException("Stack full");
        }
        super.push(item);
    }
    
    // ✅ Invariante size() >= 0 se mantiene
}
```

## Métodos No Implementados

### ❌ Violación: UnsupportedOperationException

```java
interface List<T> {
    void add(T item);
    void remove(int index);
}

class ImmutableList<T> implements List<T> {
    public void add(T item) {
        // ❌ Viola LSP: cliente espera que add() funcione
        throw new UnsupportedOperationException("Cannot modify immutable list");
    }
    
    public void remove(int index) {
        throw new UnsupportedOperationException("Cannot modify immutable list");
    }
}
```

### ✅ Solución: Interfaces Segregadas

```java
interface ReadableList<T> {
    T get(int index);
    int size();
}

interface WritableList<T> extends ReadableList<T> {
    void add(T item);
    void remove(int index);
}

class ImmutableList<T> implements ReadableList<T> {
    public T get(int index) { /* ... */ }
    public int size() { /* ... */ }
    // No implementa WritableList - No viola LSP
}

class MutableList<T> implements WritableList<T> {
    public T get(int index) { /* ... */ }
    public int size() { /* ... */ }
    public void add(T item) { /* ... */ }
    public void remove(int index) { /* ... */ }
}
```

## Resumen

**LSP** = Subtipos deben ser sustituibles por sus tipos base sin romper el programa.

**Reglas**:
1. No fortalecer precondiciones
2. No debilitar postcondiciones
3. Mantener invariantes
4. No lanzar excepciones inesperadas
5. No modificar comportamiento esperado

**Detección**: Tests que pasan con tipo base pero fallan con subtipos → violación LSP.
