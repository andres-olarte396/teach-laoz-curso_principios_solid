# Herencia vs Composición para LSP

## El Problema de la Herencia

La herencia crea acoplamiento fuerte y puede violar LSP fácilmente:

```java
// ❌ Herencia problemática
class Bird {
    void fly() { System.out.println("Flying"); }
}

class Penguin extends Bird {
    @Override
    void fly() {
        throw new UnsupportedOperationException("Penguins can't fly");
    }
}

// Viola LSP: Penguin no es sustituible por Bird
void makeBirdFly(Bird bird) {
    bird.fly(); // ❌ Falla con Penguin
}
```

## Solución: Composición

```java
// ✅ Composición
interface Bird {
    void move();
}

interface Flyable {
    void fly();
}

class Sparrow implements Bird, Flyable {
    public void move() { fly(); }
    public void fly() { System.out.println("Flying"); }
}

class Penguin implements Bird {
    public void move() { swim(); }
    private void swim() { System.out.println("Swimming"); }
}

// Cliente específico
void makeFly(Flyable flyable) {
    flyable.fly(); // Solo acepta objetos voladores
}

void makeMove(Bird bird) {
    bird.move(); // Funciona con cualquier ave
}
```

## Favor Composición sobre Herencia

### Herencia: Relación "ES-UN"
Solo usar cuando hay sustitución completa.

### Composición: Relación "TIENE-UN"
Más flexible, evita violaciones LSP.

## Ejemplo: Sistema de Empleados

```java
// ❌ Herencia rígida
class Employee {
    void calculatePay() { /* ... */ }
    void generateReport() { /* ... */ }
}

class Contractor extends Employee {
    @Override
    void generateReport() {
        throw new UnsupportedOperationException(); // Contractors no tienen reportes
    }
}

// ✅ Composición flexible
interface Payable {
    void calculatePay();
}

interface Reportable {
    void generateReport();
}

class FullTimeEmployee implements Payable, Reportable {
    public void calculatePay() { /* ... */ }
    public void generateReport() { /* ... */ }
}

class Contractor implements Payable {
    public void calculatePay() { /* ... */ }
    // No implementa Reportable
}
```

## Delegation Pattern

```java
class Stack<T> {
    private List<T> list = new ArrayList<>();
    
    public void push(T item) {
        list.add(item);
    }
    
    public T pop() {
        return list.remove(list.size() - 1);
    }
    
    // No expone métodos de List que rompan abstracción de Stack
}
```

## Resumen

**Herencia**: Solo cuando hay sustitución perfecta (relación "ES-UN" verdadera)

**Composición**: Cuando hay comportamiento compartido pero no sustitución completa

**Regla**: Preferir composición, usar herencia con precaución
