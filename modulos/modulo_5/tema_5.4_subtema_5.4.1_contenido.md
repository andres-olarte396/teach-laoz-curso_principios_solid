# ISP en Java: Default Methods

## Default Methods (Java 8+)

Permiten evolucionar interfaces sin romper implementaciones:

```java
interface Collection<E> {
    boolean add(E e);
    boolean remove(Object o);
    
    // ✅ Default method: no rompe implementaciones existentes
    default Stream<E> stream() {
        return StreamSupport.stream(spliterator(), false);
    }
}

// Implementaciones antiguas no necesitan cambiar
class MyCollection implements Collection {
    // Solo implementa add() y remove()
    // Hereda stream() automáticamente
}
```

## ISP y Default Methods

```java
// Interfaz base mínima
interface Repository<T> {
    void save(T entity);
    T findById(Long id);
}

// ✅ Extensión opcional vía default methods
interface CacheableRepository<T> extends Repository<T> {
    default void saveWithCache(T entity) {
        save(entity);
        cache.put(entity.getId(), entity);
    }
}

// Implementaciones eligen qué usar
class SimpleRepository implements Repository<User> {
    // Solo métodos básicos
}

class CachedRepository implements CacheableRepository<User> {
    // Métodos básicos + cache
}
```

## Resumen

**Default methods** = Añadir funcionalidad a interfaces sin romper ISP.
