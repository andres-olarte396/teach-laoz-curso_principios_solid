# Refactoring de Sistema Legacy

## Escenario: E-commerce Monolítico

### Estado Inicial

```java
// ❌ God Class con todas las violaciones SOLID
class OrderManager {
    private Connection dbConnection;
    private SmtpClient emailClient;
    
    public void processOrder(Map<String, Object> orderData) {
        // Validación (50 líneas)
        if (!orderData.containsKey("items")) throw new Exception();
        // ...
        
        // Cálculo (80 líneas)
        double total = 0;
        List items = (List) orderData.get("items");
        for (Object item : items) {
            Map itemData = (Map) item;
            double price = (Double) itemData.get("price");
            int quantity = (Integer) itemData.get("quantity");
            
            // Descuentos hardcoded
            if (quantity > 10) price *= 0.9;
            if (quantity > 50) price *= 0.8;
            
            total += price * quantity;
        }
        
        // Persistencia (60 líneas)
        Statement stmt = dbConnection.createStatement();
        stmt.execute("INSERT INTO orders...");
        
        // Notificación (40 líneas)
        emailClient.send(...);
    }
}
```

### Plan de Refactoring

#### Fase 1: Escribir Tests de Caracterización

```java
@Test
void testCurrentBehavior() {
    OrderManager manager = new OrderManager();
    Map<String, Object> orderData = createTestOrder();
    
    manager.processOrder(orderData);
    
    // Verificar estado actual
}
```

#### Fase 2: Extraer Responsabilidades (SRP)

```java
// Validación
class OrderValidator {
    public void validate(Order order) {
        if (order.getItems().isEmpty()) {
            throw new InvalidOrderException();
        }
    }
}

// Cálculo
class PriceCalculator {
    public double calculate(Order order) {
        return order.getItems().stream()
            .mapToDouble(item -> item.getPrice() * item.getQuantity())
            .sum();
    }
}

// Persistencia
interface OrderRepository {
    void save(Order order);
}

class JdbcOrderRepository implements OrderRepository {
    public void save(Order order) {
        // JDBC logic
    }
}

// Notificación
interface NotificationService {
    void sendOrderConfirmation(Order order);
}

class EmailNotificationService implements NotificationService {
    public void sendOrderConfirmation(Order order) {
        // Email logic
    }
}
```

#### Fase 3: Introducir Abstracciones (DIP + OCP)

```java
// Estrategia de descuentos (OCP)
interface DiscountStrategy {
    double apply(double price, int quantity);
}

class VolumeDiscountStrategy implements DiscountStrategy {
    public double apply(double price, int quantity) {
        if (quantity > 50) return price * 0.8;
        if (quantity > 10) return price * 0.9;
        return price;
    }
}

// Refactored calculator
class PriceCalculator {
    private DiscountStrategy discountStrategy;
    
    public double calculate(Order order) {
        return order.getItems().stream()
            .mapToDouble(item -> {
                double price = item.getPrice();
                price = discountStrategy.apply(price, item.getQuantity());
                return price * item.getQuantity();
            })
            .sum();
    }
}
```

#### Fase 4: Orquestación con Use Case

```java
class ProcessOrderUseCase {
    private final OrderValidator validator;
    private final PriceCalculator calculator;
    private final OrderRepository repository;
    private final NotificationService notificationService;
    
    @Autowired
    public ProcessOrderUseCase(OrderValidator validator,
                               PriceCalculator calculator,
                               OrderRepository repository,
                               NotificationService notificationService) {
        this.validator = validator;
        this.calculator = calculator;
        this.repository = repository;
        this.notificationService = notificationService;
    }
    
    public void execute(Order order) {
        validator.validate(order);
        double total = calculator.calculate(order);
        order.setTotal(total);
        repository.save(order);
        notificationService.sendOrderConfirmation(order);
    }
}
```

### Resultado Final

**Métricas**:
- Clase original: 500 LOC, CC=25, LCOM=0.8
- Refactorizado: 5 clases cohesivas, 50-100 LOC cada una, CC<5, LCOM<0.2

**Beneficios**:
- ✅ Testeable (mocks fáciles)
- ✅ Extensible (nuevos descuentos sin modificar)
- ✅ Mantenible (responsabilidades claras)

## Resumen

**Legacy refactoring** = Tests → Extract responsibilities → Abstractions → Use cases.
