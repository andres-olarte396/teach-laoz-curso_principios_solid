# Abstracciones Estables

## Principio de Estabilidad

**Abstracción estable**: No cambia frecuentemente, base sólida para dependencias.

## Características

1. **Cohesiva**: Un solo propósito claro
2. **Completa**: Suficiente para clientes
3. **Minimal**: Solo lo necesario (ISP)
4. **Independiente**: No depende de detalles

## Ejemplo

```java
// ❌ Abstracción inestable (detalles técnicos)
interface UserStorage {
    void saveToMySQL(User user);
    ResultSet executeQuery(String sql);
    Connection getConnection();
}

// ✅ Abstracción estable (concepto de negocio)
interface UserRepository {
    void save(User user);
    User findById(Long id);
    List<User> findAll();
}
```

## Nivel de Abstracción

```java
// Alto nivel (estable)
interface PaymentProcessor {
    PaymentResult process(Payment payment);
}

// Nivel medio
interface PaymentGateway {
    Transaction charge(Amount amount, CreditCard card);
}

// Bajo nivel (inestable)
class StripeHTTPClient {
    HttpResponse post(String url, Map<String, String> params);
}
```

**Regla**: Depender de abstracciones de alto nivel, no de detalles de bajo nivel.

## Resumen

**Abstracciones estables** = Conceptos de negocio, no detalles técnicos.
