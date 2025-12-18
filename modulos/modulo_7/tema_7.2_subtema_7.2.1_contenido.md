# Trade-offs y Casos Límite

## Over-engineering vs Pragmatismo

### ❌ Over-engineering

```java
// Exceso: 3 interfaces para una operación simple
interface Reader { String read(); }
interface Validator { boolean validate(String data); }
interface Processor { void process(String data); }

class DataHandler {
    private Reader reader;
    private Validator validator;
    private Processor processor;
    
    public void handle() {
        String data = reader.read();
        if (validator.validate(data)) {
            processor.process(data);
        }
    }
}
```

### ✅ Pragmático

```java
// Simplicidad cuando no hay variabilidad
class DataHandler {
    public void handle(String data) {
        if (isValid(data)) {
            process(data);
        }
    }
    
    private boolean isValid(String data) { /* ... */ }
    private void process(String data) { /* ... */ }
}
```

**Regla**: Aplicar SOLID cuando hay variabilidad real o anticipada, no especulativamente.

## Performance vs Flexibilidad

```java
// ✅ Flexibilidad (OCP + DIP)
interface Cache { Object get(String key); }
class RedisCache implements Cache { /* ... */ }

// ❌ Menos flexible pero más rápido
class InlineCache {
    private Map<String, Object> cache = new HashMap<>();
    public Object get(String key) { return cache.get(key); }
}
```

**Trade-off**: Abstracciones tienen costo (indirección), evaluar caso por caso.

## Cuándo Romper Reglas

1. **Performance crítico**: Inline code en lugar de Strategy
2. **Prototipo**: YAGNI (You Aren't Gonna Need It)
3. **Dominio simple**: No añadir complejidad innecesaria

## Resumen

**Pragmatismo** = Aplicar SOLID con criterio, evitar dogmatismo.
