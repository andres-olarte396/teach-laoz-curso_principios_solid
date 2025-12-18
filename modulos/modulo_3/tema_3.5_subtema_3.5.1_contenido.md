# OCP en Java

## Características del Lenguaje

### Interfaces

```java
public interface PaymentProcessor {
    void process(Payment payment);
}

public class CreditCardProcessor implements PaymentProcessor {
    public void process(Payment payment) {
        // Lógica específica
    }
}

public class PayPalProcessor implements PaymentProcessor {
    public void process(Payment payment) {
        // Lógica específica
    }
}
```

**OCP**: Añadir `BitcoinProcessor` no modifica código existente.

### Default Methods (Java 8+)

```java
public interface Logger {
    void log(String message);
    
    default void logError(String message) {
        log("ERROR: " + message);
    }
    
    default void logWarning(String message) {
        log("WARNING: " + message);
    }
}

public class ConsoleLogger implements Logger {
    public void log(String message) {
        System.out.println(message);
    }
    // Hereda logError() y logWarning() por defecto
}

public class FileLogger implements Logger {
    public void log(String message) {
        writeToFile(message);
    }
    
    @Override
    public void logError(String message) {
        // Personalizar para archivos
        writeToFile("[ERROR] " + message);
    }
}
```

**Ventaja**: Puedes añadir métodos a interfaces sin romper implementaciones existentes.

### Functional Interfaces y Lambdas

```java
@FunctionalInterface
public interface Validator<T> {
    boolean validate(T value);
}

// Extensión mediante lambdas
Validator<String> emailValidator = email -> email.contains("@");
Validator<String> lengthValidator = str -> str.length() >= 5;
Validator<Integer> positiveValidator = num -> num > 0;

// Composición
public class ValidatorChain<T> {
    private List<Validator<T>> validators = new ArrayList<>();
    
    public ValidatorChain<T> add(Validator<T> validator) {
        validators.add(validator);
        return this;
    }
    
    public boolean validate(T value) {
        return validators.stream().allMatch(v -> v.validate(value));
    }
}

// Uso
ValidatorChain<String> emailChain = new ValidatorChain<String>()
    .add(email -> email.contains("@"))
    .add(email -> email.length() >= 5)
    .add(email -> email.endsWith(".com"));
```

### Streams API para Extensibilidad

```java
public interface OrderFilter {
    boolean test(Order order);
}

public class OrderService {
    public List<Order> filterOrders(List<Order> orders, OrderFilter filter) {
        return orders.stream()
                     .filter(filter::test)
                     .collect(Collectors.toList());
    }
}

// Extensión sin modificar OrderService
OrderFilter highValueFilter = order -> order.getTotal() > 1000;
OrderFilter premiumCustomerFilter = order -> order.getCustomer().isPremium();

List<Order> highValueOrders = orderService.filterOrders(orders, highValueFilter);
```

### Reflection para Plugin Loading

```java
public interface Plugin {
    String getName();
    void execute();
}

public class PluginLoader {
    public static List<Plugin> loadPlugins(String packageName) {
        Reflections reflections = new Reflections(packageName);
        Set<Class<? extends Plugin>> pluginClasses = reflections.getSubTypesOf(Plugin.class);
        
        return pluginClasses.stream()
                .map(clazz -> {
                    try {
                        return clazz.getDeclaredConstructor().newInstance();
                    } catch (Exception e) {
                        throw new RuntimeException("Failed to load plugin", e);
                    }
                })
                .collect(Collectors.toList());
    }
}

// Uso
List<Plugin> plugins = PluginLoader.loadPlugins("com.example.plugins");
plugins.forEach(Plugin::execute);
```

## Frameworks Java que Promueven OCP

### Spring Framework

```java
// Extensión mediante beans
@Configuration
public class AppConfig {
    @Bean
    public PaymentProcessor creditCardProcessor() {
        return new CreditCardProcessor();
    }
    
    @Bean
    public PaymentProcessor payPalProcessor() {
        return new PayPalProcessor();
    }
}

// Spring auto-inyecta todos los PaymentProcessor
@Service
public class PaymentService {
    private List<PaymentProcessor> processors;
    
    @Autowired
    public PaymentService(List<PaymentProcessor> processors) {
        this.processors = processors;
    }
    
    public void processPayment(Payment payment) {
        for (PaymentProcessor processor : processors) {
            if (processor.supports(payment)) {
                processor.process(payment);
                return;
            }
        }
    }
}
```

### JUnit 5 Extensions

```java
// Extension Point
public interface TestExecutionListener {
    void beforeEach(ExtensionContext context);
    void afterEach(ExtensionContext context);
}

// Implementación
public class TimingExtension implements TestExecutionListener {
    public void beforeEach(ExtensionContext context) {
        long start = System.currentTimeMillis();
        context.getStore(NAMESPACE).put("start", start);
    }
    
    public void afterEach(ExtensionContext context) {
        long start = context.getStore(NAMESPACE).get("start", Long.class);
        long duration = System.currentTimeMillis() - start;
        System.out.println("Test took: " + duration + "ms");
    }
}

// Uso
@ExtendWith(TimingExtension.class)
class MyTest {
    @Test
    void testSomething() {
        // ...
    }
}
```

## Patrones Java-Específicos

### Builder Pattern con Fluent API

```java
public class HttpClient {
    private String baseUrl;
    private int timeout;
    private Map<String, String> headers;
    
    private HttpClient(Builder builder) {
        this.baseUrl = builder.baseUrl;
        this.timeout = builder.timeout;
        this.headers = builder.headers;
    }
    
    public static class Builder {
        private String baseUrl;
        private int timeout = 30000;
        private Map<String, String> headers = new HashMap<>();
        
        public Builder baseUrl(String baseUrl) {
            this.baseUrl = baseUrl;
            return this;
        }
        
        public Builder timeout(int timeout) {
            this.timeout = timeout;
            return this;
        }
        
        public Builder header(String key, String value) {
            headers.put(key, value);
            return this;
        }
        
        public HttpClient build() {
            return new HttpClient(this);
        }
    }
}

// Extensión
HttpClient client = new HttpClient.Builder()
    .baseUrl("https://api.example.com")
    .timeout(10000)
    .header("Authorization", "Bearer token")
    .header("Content-Type", "application/json")
    .build();
```

### Annotations para Extensibilidad

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Cacheable {
    int ttl() default 60;
}

public class CacheAspect {
    private Map<String, CachedValue> cache = new HashMap<>();
    
    public Object handleCacheable(ProceedingJoinPoint joinPoint) throws Throwable {
        Cacheable cacheable = getAnnotation(joinPoint);
        String key = generateKey(joinPoint);
        
        CachedValue cached = cache.get(key);
        if (cached != null && !cached.isExpired()) {
            return cached.getValue();
        }
        
        Object result = joinPoint.proceed();
        cache.put(key, new CachedValue(result, cacheable.ttl()));
        return result;
    }
}

// Uso
public class UserService {
    @Cacheable(ttl = 300)
    public User getUser(Long id) {
        return database.findUser(id);
    }
}
```

## Resumen

**Java** facilita OCP mediante:
- **Interfaces**: Contratos extensibles
- **Default methods**: Añadir funcionalidad sin romper implementaciones
- **Lambdas**: Comportamiento extensible inline
- **Reflection**: Descubrimiento dinámico de plugins
- **Annotations**: Metadata extensible
- **Frameworks** (Spring, JUnit): Extension points bien definidos
