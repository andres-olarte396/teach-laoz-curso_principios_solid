# Dependency Inversion Principle (DIP)

## Definición

> "A. Los módulos de alto nivel no deben depender de módulos de bajo nivel. Ambos deben depender de abstracciones."
> 
> "B. Las abstracciones no deben depender de los detalles. Los detalles deben depender de las abstracciones."

## Problema: Acoplamiento Directo

```java
// ❌ Alto nivel depende de bajo nivel
class OrderProcessor {
    private MySQLDatabase database; // Acoplamiento directo
    
    public void processOrder(Order order) {
        database.save(order);
    }
}
```

**Problemas**:
- No puedes cambiar MySQL por PostgreSQL sin modificar OrderProcessor
- Difícil testing (necesitas base de datos real)
- Violación de OCP (cerrado a extensión)

## Solución: Invertir Dependencia

```java
// ✅ Abstracción
interface OrderRepository {
    void save(Order order);
}

// Alto nivel depende de abstracción
class OrderProcessor {
    private OrderRepository repository; // Abstracción
    
    public OrderProcessor(OrderRepository repository) {
        this.repository = repository;
    }
    
    public void processOrder(Order order) {
        repository.save(order);
    }
}

// Bajo nivel implementa abstracción
class MySQLOrderRepository implements OrderRepository {
    public void save(Order order) {
        // Implementación MySQL
    }
}

class PostgreSQLOrderRepository implements OrderRepository {
    public void save(Order order) {
        // Implementación PostgreSQL
    }
}
```

## Inversión de Control (IoC)

**Antes DIP**: Alto nivel controla bajo nivel (llama directamente)
**Con DIP**: Abstracción controla (bajo nivel implementa contrato)

```java
// Cliente elige implementación
OrderRepository repo = new MySQLOrderRepository();
OrderProcessor processor = new OrderProcessor(repo);

// Fácil cambiar implementación
OrderRepository repo2 = new PostgreSQLOrderRepository();
OrderProcessor processor2 = new OrderProcessor(repo2);
```

## Beneficios

1. **Flexibilidad**: Cambiar implementaciones sin modificar código
2. **Testabilidad**: Mocks/stubs fáciles
3. **Reusabilidad**: Alto nivel independiente de detalles
4. **Mantenibilidad**: Cambios localizados

## Resumen

**DIP** = Depender de abstracciones, no de concreciones. Invertir flujo de control.
