# Dependency Injection (DI)

## Concepto

**DI** = Técnica para implementar DIP. Dependencias se inyectan externamente.

## Tipos de Inyección

### 1. Constructor Injection (Recomendada)

```java
class OrderService {
    private final OrderRepository repository;
    private final EmailService emailService;
    
    // ✅ Dependencias inmutables
    public OrderService(OrderRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }
}
```

**Ventajas**:
- Dependencias explícitas
- Inmutabilidad
- Fácil testing

### 2. Setter Injection

```java
class OrderService {
    private OrderRepository repository;
    
    // Inyección opcional
    public void setRepository(OrderRepository repository) {
        this.repository = repository;
    }
}
```

**Uso**: Dependencias opcionales.

### 3. Field Injection (Evitar)

```java
class OrderService {
    @Autowired // ❌ Evitar (rompe encapsulación)
    private OrderRepository repository;
}
```

## Manual DI

```java
// Main o configuración
OrderRepository repo = new MySQLOrderRepository();
EmailService email = new SMTPEmailService();
OrderService service = new OrderService(repo, email);
```

## DI Container (Spring)

```java
@Configuration
class AppConfig {
    @Bean
    public OrderRepository orderRepository() {
        return new MySQLOrderRepository();
    }
    
    @Bean
    public EmailService emailService() {
        return new SMTPEmailService();
    }
    
    @Bean
    public OrderService orderService(OrderRepository repo, EmailService email) {
        return new OrderService(repo, email);
    }
}

// Spring resuelve dependencias automáticamente
@Autowired
OrderService service;
```

## Resumen

**DI** = Inyectar dependencias externamente. Constructor injection preferida.
