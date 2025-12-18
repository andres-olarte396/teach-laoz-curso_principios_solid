# IoC Containers

## Concepto

**IoC Container**: Framework que gestiona creación y ciclo de vida de objetos.

## Spring Framework

```java
@Component
class MySQLOrderRepository implements OrderRepository {
    // Spring crea instancia automáticamente
}

@Service
class OrderService {
    private final OrderRepository repository;
    
    @Autowired // Inyección automática
    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}

// Application context
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(App.class, args);
        OrderService service = context.getBean(OrderService.class);
    }
}
```

## Scopes

```java
@Component
@Scope("singleton") // Una instancia compartida (default)
class ConfigService {}

@Component
@Scope("prototype") // Nueva instancia cada vez
class RequestHandler {}

@Component
@Scope("request") // Una por HTTP request (Spring Web)
class UserSession {}
```

## Qualifiers

```java
interface PaymentGateway {}

@Component
@Qualifier("stripe")
class StripeGateway implements PaymentGateway {}

@Component
@Qualifier("paypal")
class PayPalGateway implements PaymentGateway {}

@Service
class PaymentService {
    @Autowired
    @Qualifier("stripe") // Selecciona implementación específica
    private PaymentGateway gateway;
}
```

## Resumen

**IoC Container** = Framework que gestiona objetos y dependencias automáticamente.
