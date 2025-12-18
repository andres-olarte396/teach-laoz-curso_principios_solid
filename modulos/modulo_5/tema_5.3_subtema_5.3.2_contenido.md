# Interfaces Mínimas y Cohesivas

## Principio de Mínima Interfaz

**Regla**: Exponer solo lo absolutamente necesario para el cliente.

```java
// ❌ Interfaz con detalles de implementación
interface UserRepository {
    void save(User user);
    User findById(Long id);
    Connection getConnection(); // ❌ Detalle de implementación
    void clearCache(); // ❌ Detalle de implementación
}

// ✅ Interfaz mínima
interface UserRepository {
    void save(User user);
    User findById(Long id);
}
```

## Cohesión de Interfaz

**Alta cohesión**: Métodos fuertemente relacionados.

```java
// ✅ Alta cohesión
interface Authenticator {
    boolean authenticate(String username, String password);
    void logout(String username);
    boolean isAuthenticated(String username);
}

// ❌ Baja cohesión
interface UserService {
    boolean authenticate(String username, String password); // Autenticación
    void sendEmail(String to, String message); // Email
    void generateReport(); // Reportes
}
```

## Resumen

**Interfaces mínimas** = Solo métodos necesarios para el cliente, alta cohesión.
