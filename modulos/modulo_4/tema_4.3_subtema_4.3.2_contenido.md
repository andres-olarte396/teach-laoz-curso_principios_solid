# Refactoring de Jerarquías Problemáticas

## Identificación de Violaciones

### Code Smell: Refused Bequest
Subclase que no usa herencia del padre:

```java
// ❌ Violación
class Bird {
    void fly() {}
    void layEggs() {}
}

class Ostrich extends Bird {
    @Override
    void fly() {
        throw new UnsupportedOperationException(); // Refused bequest
    }
}
```

## Técnicas de Refactoring

### 1. Push Down Method
Mover comportamiento específico a subclases:

```java
// Antes
class Animal {
    void fly() { /* solo para aves */ }
    void swim() { /* solo para peces */ }
}

// Después
class Animal {}
class Bird extends Animal {
    void fly() {}
}
class Fish extends Animal {
    void swim() {}
}
```

### 2. Replace Inheritance with Delegation

```java
// Antes
class Stack extends ArrayList {
    void push(Object o) { add(o); }
    Object pop() { return remove(size() - 1); }
}

// Después
class Stack {
    private List items = new ArrayList();
    void push(Object o) { items.add(o); }
    Object pop() { return items.remove(items.size() - 1); }
}
```

### 3. Extract Interface

```java
// Antes
class Rectangle {
    void setWidth(int w) {}
    void setHeight(int h) {}
}
class Square extends Rectangle { /* problemas */ }

// Después
interface Shape {
    int getArea();
}
class Rectangle implements Shape {}
class Square implements Shape {}
```

## Resumen

**Refactoring LSP**:
- Identificar refused bequest
- Push down métodos específicos
- Replace inheritance with delegation
- Extract interface para contratos comunes
