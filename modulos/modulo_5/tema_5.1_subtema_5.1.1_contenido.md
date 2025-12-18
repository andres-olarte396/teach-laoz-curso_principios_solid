# Definición: Clientes No Deben Depender de Interfaces Innecesarias

## Interface Segregation Principle (ISP)

> "Los clientes no deben ser forzados a depender de interfaces que no usan."

## Problema: Fat Interfaces

```java
// ❌ Fat Interface
interface Worker {
    void work();
    void eat();
    void sleep();
    void attendMeeting();
    void writeReport();
}

class Robot implements Worker {
    public void work() { /* ... */ }
    public void eat() { throw new UnsupportedOperationException(); } // ❌
    public void sleep() { throw new UnsupportedOperationException(); } // ❌
    public void attendMeeting() { /* ... */ }
    public void writeReport() { /* ... */ }
}
```

**Problema**: Robot debe implementar métodos irrelevantes.

## Solución: Interfaces Segregadas

```java
// ✅ Interfaces cohesivas
interface Workable {
    void work();
}

interface Eatable {
    void eat();
}

interface Sleepable {
    void sleep();
}

interface Reportable {
    void writeReport();
}

class Human implements Workable, Eatable, Sleepable, Reportable {
    public void work() { /* ... */ }
    public void eat() { /* ... */ }
    public void sleep() { /* ... */ }
    public void writeReport() { /* ... */ }
}

class Robot implements Workable, Reportable {
    public void work() { /* ... */ }
    public void writeReport() { /* ... */ }
    // No implementa Eatable ni Sleepable
}
```

## Beneficios

1. **Flexibilidad**: Clases implementan solo lo necesario
2. **Reusabilidad**: Interfaces pequeñas son más reutilizables
3. **Testing**: Mocks más simples
4. **Mantenibilidad**: Cambios en una interfaz afectan menos clases

## ISP y SRP

**SRP** = Una clase, una responsabilidad
**ISP** = Una interfaz, una responsabilidad

ISP es SRP aplicado a interfaces.

## Resumen

**ISP** = Interfaces pequeñas y cohesivas, clientes solo dependen de lo que usan.
