# SRP en Java y C#

## Características del Lenguaje que Facilitan SRP

### Java

**Interfaces**: Permiten abstracciones claras
```java
public interface OrderRepository {
    void save(Order order);
    Order findById(Long id);
}

public class JpaOrderRepository implements OrderRepository {
    // Implementación específica
}
```

**Clases internas/anidadas**: Encapsular responsabilidades auxiliares
```java
public class OrderProcessor {
    public void process(Order order) {
        Validator validator = new Validator();
        validator.validate(order);
    }
    
    private static class Validator {
        void validate(Order order) { /* ... */ }
    }
}
```

**Records (Java 14+)**: Clases de datos inmutables
```java
public record OrderPrice(double subtotal, double tax, double total) { }
```

### C#

**Properties**: Encapsulación elegante
```c#
public class User {
    private string _email;
    public string Email {
        get => _email;
        set {
            if (!value.Contains("@")) 
                throw new ArgumentException("Invalid email");
            _email = value;
        }
    }
}
```

**Extension Methods**: Añadir funcionalidad sin modificar clases
```c#
public static class StringExtensions {
    public static bool IsValidEmail(this string email) {
        return email.Contains("@");
    }
}

// Uso
string email = "test@example.com";
if (email.IsValidEmail()) { /* ... */ }
```

**Delegates y Events**: Desacoplar notificaciones
```c#
public class OrderService {
    public event EventHandler<OrderEventArgs> OrderProcessed;
    
    public void ProcessOrder(Order order) {
        // Procesar...
        OrderProcessed?.Invoke(this, new OrderEventArgs(order));
    }
}
```

## Patrones Idiomáticos

### Java: Builder Pattern

```java
public class User {
    private final String name;
    private final String email;
    private final String phone;
    
    private User(Builder builder) {
        this.name = builder.name;
        this.email = builder.email;
        this.phone = builder.phone;
    }
    
    public static class Builder {
        private String name;
        private String email;
        private String phone;
        
        public Builder name(String name) {
            this.name = name;
            return this;
        }
        
        public Builder email(String email) {
            this.email = email;
            return this;
        }
        
        public Builder phone(String phone) {
            this.phone = phone;
            return this;
        }
        
        public User build() {
            return new User(this);
        }
    }
}

// Uso
User user = new User.Builder()
    .name("John")
    .email("john@example.com")
    .build();
```

### C#: LINQ para Transformaciones

```c#
// Separar lógica de filtrado/transformación
public class OrderService {
    public IEnumerable<Order> GetHighValueOrders(IEnumerable<Order> orders) {
        return orders.Where(o => o.Total > 1000)
                    .OrderByDescending(o => o.Total);
    }
}
```

## Frameworks que Promueven SRP

### Java: Spring Framework

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    private final UserValidator userValidator;
    
    @Autowired
    public UserService(UserRepository userRepository, UserValidator userValidator) {
        this.userRepository = userRepository;
        this.userValidator = userValidator;
    }
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> { }

@Component
public class UserValidator {
    public void validate(User user) { /* ... */ }
}
```

### C#: ASP.NET Core

```c#
public class Startup {
    public void ConfigureServices(IServiceCollection services) {
        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<UserService>();
        services.AddScoped<UserValidator>();
    }
}

public class UserController : ControllerBase {
    private readonly UserService _userService;
    
    public UserController(UserService userService) {
        _userService = userService;
    }
}
```

## Anti-Patrones Específicos del Lenguaje

### Java: Utility Classes Incorrectas

```java
// ❌ MAL: Todo estático
public class Utils {
    public static void sendEmail(String to, String subject) { }
    public static double calculateTax(double amount) { }
    public static void logError(Exception e) { }
}

// ✅ BIEN: Clases especializadas
public class EmailSender {
    public void send(String to, String subject) { }
}

public class TaxCalculator {
    public double calculate(double amount) { }
}

public class ErrorLogger {
    public void log(Exception e) { }
}
```

### C#: Abuso de Extension Methods

```c#
// ❌ MAL: Extensions que no extienden comportamiento natural
public static class OrderExtensions {
    public static void SaveToDatabase(this Order order) {
        // Lógica de persistencia no pertenece aquí
    }
}

// ✅ BIEN: Usar servicios apropiados
public class OrderRepository {
    public void Save(Order order) { /* ... */ }
}
```

## Resumen

Tanto Java como C# ofrecen herramientas para SRP. La clave es usarlas apropiadamente: interfaces para abstracciones, DI para desacoplamiento, patterns idiomáticos para claridad.
