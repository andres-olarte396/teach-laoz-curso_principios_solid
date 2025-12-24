# Ejercicios: SRP y Testing

## Nivel 1. Identificación de Problemas de Testing ⭐

### Ejercicio 1.1. Analizar Testabilidad

Evalúa la testabilidad de estas clases y explica por qué son difíciles de testear:

**Clase A:**
```java
public class ReportService {
    public void generateReport(String reportType) {
        Connection conn = DriverManager.getConnection("jdbc:mysql://prod-server/db");
        Statement stmt = conn.createStatement();
        ResultSet rs = stmt.executeQuery("SELECT * FROM sales");
        
        if (reportType.equals("PDF")) {
            PDFGenerator pdf = new PDFGenerator();
            pdf.generate(rs, "report.pdf");
        } else if (reportType.equals("EXCEL")) {
            ExcelGenerator excel = new ExcelGenerator();
            excel.generate(rs, "report.xlsx");
        }
        
        EmailSender sender = new EmailSender("smtp.gmail.com");
        sender.send("boss@company.com", "Report Ready", "Your report is ready");
    }
}
```

**Clase B:**
```java
public class UserValidator {
    public boolean isValid(String email) {
        return email != null && 
               email.contains("@") && 
               email.length() > 5 &&
               email.length() < 100;
    }
}
```

**Preguntas:**
1. ¿Qué necesitas para testear la Clase A?
2. ¿Qué necesitas para testear la Clase B?
3. ¿Cuántas responsabilidades tiene cada clase?
4. ¿Cuántos mocks necesitarías para cada clase?

---

**SOLUCIÓN:**

**Clase A - Problemas de Testabilidad:**

1. **Necesitas para testear:**
   - Base de datos MySQL en servidor de producción corriendo
   - Datos de prueba en tabla `sales`
   - Servidor SMTP configurado
   - Permisos de escritura para generar PDFs/Excel
   - Librerías de generación de documentos configuradas

2. **Responsabilidades (5):**
   - Conexión a base de datos
   - Consulta de datos
   - Generación de PDF
   - Generación de Excel
   - Envío de email

3. **Mocks necesarios**: 5-6 (Connection, Statement, ResultSet, PDFGenerator, ExcelGenerator, EmailSender)

4. **Score de testabilidad**: ❌ 2/10 (muy difícil)

**Clase B - Testabilidad:**

1. **Necesitas para testear:**
   - Nada, método puro sin dependencias

2. **Responsabilidades**: 1 (validación de email)

3. **Mocks necesarios**: 0

4. **Score de testabilidad**: ✅ 10/10 (trivial)

**Test de Clase B:**
```java
@Test
void shouldAcceptValidEmail() {
    UserValidator validator = new UserValidator();
    assertTrue(validator.isValid("user@example.com"));
}

@Test
void shouldRejectEmailWithoutAt() {
    UserValidator validator = new UserValidator();
    assertFalse(validator.isValid("userexample.com"));
}
```

---

### Ejercicio 1.2: Comparar Estrategias de Testing

Compara estas dos implementaciones del mismo servicio:

**Implementación 1:**
```java
public class OrderService {
    public void processOrder(Order order) {
        // Validación inline
        if (order.getItems().isEmpty()) {
            throw new IllegalArgumentException("Order is empty");
        }
        
        // Cálculo inline
        double total = 0;
        for (OrderItem item : order.getItems()) {
            total += item.getPrice() * item.getQuantity();
        }
        
        // Persistencia inline
        Connection conn = DriverManager.getConnection("jdbc:...");
        PreparedStatement stmt = conn.prepareStatement("INSERT...");
        stmt.setDouble(1, total);
        stmt.executeUpdate();
    }
}
```

**Implementación 2:**
```java
public class OrderService {
    private final OrderValidator validator;
    private final PriceCalculator calculator;
    private final OrderRepository repository;
    
    public OrderService(OrderValidator validator,
                        PriceCalculator calculator,
                        OrderRepository repository) {
        this.validator = validator;
        this.calculator = calculator;
        this.repository = repository;
    }
    
    public void processOrder(Order order) {
        validator.validate(order);
        double total = calculator.calculate(order);
        order.setTotal(total);
        repository.save(order);
    }
}
```

**Tareas:**
1. Escribe un test para cada implementación
2. Cuenta las líneas de código de cada test
3. ¿Cuál es más rápido de ejecutar?
4. ¿Cuál es más mantenible?

---

**SOLUCIÓN:**

**Test Implementación 1:**
```java
@Test
void testProcessOrder() throws SQLException {
    // Setup (complejo)
    // Necesitas BD real o H2 embedded
    Connection conn = DriverManager.getConnection("jdbc:h2:mem:test");
    Statement stmt = conn.createStatement();
    stmt.execute("CREATE TABLE orders (total DOUBLE)");
    
    OrderService service = new OrderService();
    Order order = new Order();
    order.addItem(new OrderItem("Product", 10.0, 2));
    
    // Ejecutar
    service.processOrder(order);
    
    // Verificar (complejo)
    ResultSet rs = conn.createStatement().executeQuery("SELECT total FROM orders");
    assertTrue(rs.next());
    assertEquals(20.0, rs.getDouble("total"), 0.01);
    
    conn.close();
}
// Líneas: ~20, Tiempo: ~500ms
```

**Test Implementación 2:**
```java
@Test
void testProcessOrder() {
    // Setup (simple)
    OrderValidator mockValidator = mock(OrderValidator.class);
    PriceCalculator mockCalculator = mock(PriceCalculator.class);
    OrderRepository mockRepository = mock(OrderRepository.class);
    
    when(mockCalculator.calculate(any())).thenReturn(20.0);
    
    OrderService service = new OrderService(mockValidator, mockCalculator, mockRepository);
    Order order = new Order();
    
    // Ejecutar
    service.processOrder(order);
    
    // Verificar
    verify(mockValidator).validate(order);
    verify(mockCalculator).calculate(order);
    verify(mockRepository).save(order);
    assertEquals(20.0, order.getTotal());
}
// Líneas: ~15, Tiempo: ~5ms
```

**Comparación:**

| Aspecto | Implementación 1 | Implementación 2 |
|---------|-----------------|------------------|
| Líneas de test | 20 | 15 |
| Tiempo de ejecución | ~500ms | ~5ms |
| Dependencias externas | BD H2 | Ninguna |
| Mantenibilidad | Baja | Alta |
| Claridad | Media | Alta |
| Fragilidad | Alta (falla si BD cambia) | Baja |

---

## Nivel 2: Refactoring para Testabilidad ⭐⭐

### Ejercicio 2.1. Hacer Clase Testeable

Refactoriza esta clase para hacerla completamente testeable:

```java
public class PaymentProcessor {
    public boolean processPayment(String cardNumber, double amount) {
        // Validar tarjeta
        if (cardNumber.length() != 16) {
            return false;
        }
        
        // Conectar a gateway de pago
        StripeAPI stripe = new StripeAPI("sk_live_real_key");
        
        try {
            ChargeResponse response = stripe.charge(cardNumber, amount);
            
            if (response.isSuccess()) {
                // Guardar transacción
                Connection conn = DriverManager.getConnection("jdbc:mysql://prod/payments");
                String sql = "INSERT INTO transactions (card, amount, status) VALUES (?, ?, ?)";
                PreparedStatement stmt = conn.prepareStatement(sql);
                stmt.setString(1, cardNumber);
                stmt.setDouble(2, amount);
                stmt.setString(3, "SUCCESS");
                stmt.executeUpdate();
                
                // Enviar recibo por email
                String email = getEmailFromCard(cardNumber);
                EmailService emailService = new EmailService();
                emailService.send(email, "Payment Receipt", "Charged $" + amount);
                
                return true;
            }
            return false;
        } catch (Exception e) {
            return false;
        }
    }
    
    private String getEmailFromCard(String cardNumber) {
        // Lógica complicada para obtener email
        return "customer@example.com";
    }
}
```

**Tareas:**
1. Identifica las responsabilidades
2. Extrae clases para cada responsabilidad
3. Implementa las clases extraídas con interfaces
4. Escribe tests unitarios para cada clase
5. Escribe test de integración para el flujo completo

---

**SOLUCIÓN MODELO:**

**1. Responsabilidades identificadas:**
- Validación de tarjeta
- Procesamiento de pago con gateway
- Persistencia de transacción
- Envío de recibo por email
- Asociación de tarjeta con email

**2. Clases extraídas:**

```java
// Validación
public class CardValidator {
    private static final int VALID_LENGTH = 16;
    
    public ValidationResult validate(String cardNumber) {
        if (cardNumber == null) {
            return ValidationResult.failure("Card number is required");
        }
        
        if (cardNumber.length() != VALID_LENGTH) {
            return ValidationResult.failure("Card number must be 16 digits");
        }
        
        // Algoritmo de Luhn
        if (!passesLuhnCheck(cardNumber)) {
            return ValidationResult.failure("Invalid card number");
        }
        
        return ValidationResult.success();
    }
    
    private boolean passesLuhnCheck(String cardNumber) {
        // Implementación del algoritmo de Luhn
        int sum = 0;
        boolean alternate = false;
        for (int i = cardNumber.length() - 1; i >= 0; i--) {
            int n = Integer.parseInt(cardNumber.substring(i, i + 1));
            if (alternate) {
                n *= 2;
                if (n > 9) {
                    n -= 9;
                }
            }
            sum += n;
            alternate = !alternate;
        }
        return (sum % 10 == 0);
    }
}

// Gateway de pago
public interface PaymentGateway {
    PaymentResult charge(String cardNumber, double amount);
}

public class StripePaymentGateway implements PaymentGateway {
    private final StripeAPI stripeAPI;
    
    public StripePaymentGateway(StripeAPI stripeAPI) {
        this.stripeAPI = stripeAPI;
    }
    
    @Override
    public PaymentResult charge(String cardNumber, double amount) {
        try {
            ChargeResponse response = stripeAPI.charge(cardNumber, amount);
            
            if (response.isSuccess()) {
                return PaymentResult.success(response.getTransactionId());
            } else {
                return PaymentResult.failure(response.getErrorMessage());
            }
        } catch (Exception e) {
            return PaymentResult.failure("Payment processing error: " + e.getMessage());
        }
    }
}

// Persistencia
public interface TransactionRepository {
    void save(Transaction transaction);
    Transaction findById(String transactionId);
}

public class JdbcTransactionRepository implements TransactionRepository {
    private final DataSource dataSource;
    
    public JdbcTransactionRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    @Override
    public void save(Transaction transaction) {
        String sql = "INSERT INTO transactions (id, card_last4, amount, status, created_at) " +
                     "VALUES (?, ?, ?, ?, ?)";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setString(1, transaction.getId());
            stmt.setString(2, transaction.getCardLast4());
            stmt.setDouble(3, transaction.getAmount());
            stmt.setString(4, transaction.getStatus().name());
            stmt.setTimestamp(5, Timestamp.valueOf(transaction.getCreatedAt()));
            stmt.executeUpdate();
            
        } catch (SQLException e) {
            throw new PersistenceException("Failed to save transaction", e);
        }
    }
    
    @Override
    public Transaction findById(String transactionId) {
        // Implementación
        return null;
    }
}

// Servicio de recibos
public class PaymentReceiptService {
    private final EmailService emailService;
    private final CustomerService customerService;
    
    public PaymentReceiptService(EmailService emailService,
                                 CustomerService customerService) {
        this.emailService = emailService;
        this.customerService = customerService;
    }
    
    public void sendReceipt(String cardNumber, double amount, String transactionId) {
        String email = customerService.getEmailByCard(cardNumber);
        
        String subject = "Payment Receipt - Transaction #" + transactionId;
        String body = buildReceiptBody(amount, transactionId);
        
        emailService.send(email, subject, body);
    }
    
    private String buildReceiptBody(double amount, String transactionId) {
        return String.format(
            "Your payment of $%.2f has been processed successfully.%n" +
            "Transaction ID: %s%n" +
            "Thank you for your purchase!",
            amount, transactionId
        );
    }
}

// Servicio principal (orquestador)
public class PaymentProcessor {
    private final CardValidator cardValidator;
    private final PaymentGateway paymentGateway;
    private final TransactionRepository transactionRepository;
    private final PaymentReceiptService receiptService;
    
    public PaymentProcessor(CardValidator cardValidator,
                            PaymentGateway paymentGateway,
                            TransactionRepository transactionRepository,
                            PaymentReceiptService receiptService) {
        this.cardValidator = cardValidator;
        this.paymentGateway = paymentGateway;
        this.transactionRepository = transactionRepository;
        this.receiptService = receiptService;
    }
    
    public ProcessingResult processPayment(String cardNumber, double amount) {
        // 1. Validar tarjeta
        ValidationResult validation = cardValidator.validate(cardNumber);
        if (!validation.isValid()) {
            return ProcessingResult.failure(validation.getErrors());
        }
        
        // 2. Procesar pago
        PaymentResult paymentResult = paymentGateway.charge(cardNumber, amount);
        if (!paymentResult.isSuccess()) {
            return ProcessingResult.failure(paymentResult.getErrorMessage());
        }
        
        // 3. Guardar transacción
        Transaction transaction = new Transaction(
            paymentResult.getTransactionId(),
            cardNumber.substring(cardNumber.length() - 4),
            amount,
            TransactionStatus.SUCCESS,
            LocalDateTime.now()
        );
        transactionRepository.save(transaction);
        
        // 4. Enviar recibo
        receiptService.sendReceipt(cardNumber, amount, transaction.getId());
        
        return ProcessingResult.success(transaction.getId());
    }
}
```

**4. Tests Unitarios:**

```java
class CardValidatorTest {
    private CardValidator validator;
    
    @BeforeEach
    void setUp() {
        validator = new CardValidator();
    }
    
    @Test
    void shouldAcceptValidCardNumber() {
        ValidationResult result = validator.validate("4532015112830366");
        assertTrue(result.isValid());
    }
    
    @Test
    void shouldRejectNullCardNumber() {
        ValidationResult result = validator.validate(null);
        assertFalse(result.isValid());
        assertTrue(result.getErrors().contains("Card number is required"));
    }
    
    @Test
    void shouldRejectInvalidLength() {
        ValidationResult result = validator.validate("12345");
        assertFalse(result.isValid());
        assertTrue(result.getErrors().contains("Card number must be 16 digits"));
    }
    
    @Test
    void shouldRejectInvalidLuhnChecksum() {
        ValidationResult result = validator.validate("1234567890123456");
        assertFalse(result.isValid());
    }
}

class StripePaymentGatewayTest {
    private StripePaymentGateway gateway;
    private StripeAPI mockStripeAPI;
    
    @BeforeEach
    void setUp() {
        mockStripeAPI = mock(StripeAPI.class);
        gateway = new StripePaymentGateway(mockStripeAPI);
    }
    
    @Test
    void shouldReturnSuccessWhenChargeSucceeds() {
        ChargeResponse successResponse = new ChargeResponse(true, "txn_123", null);
        when(mockStripeAPI.charge(anyString(), anyDouble())).thenReturn(successResponse);
        
        PaymentResult result = gateway.charge("4532015112830366", 100.0);
        
        assertTrue(result.isSuccess());
        assertEquals("txn_123", result.getTransactionId());
    }
    
    @Test
    void shouldReturnFailureWhenChargeDeclined() {
        ChargeResponse failedResponse = new ChargeResponse(false, null, "Insufficient funds");
        when(mockStripeAPI.charge(anyString(), anyDouble())).thenReturn(failedResponse);
        
        PaymentResult result = gateway.charge("4532015112830366", 100.0);
        
        assertFalse(result.isSuccess());
        assertEquals("Insufficient funds", result.getErrorMessage());
    }
    
    @Test
    void shouldHandleExceptionGracefully() {
        when(mockStripeAPI.charge(anyString(), anyDouble()))
            .thenThrow(new RuntimeException("Network error"));
        
        PaymentResult result = gateway.charge("4532015112830366", 100.0);
        
        assertFalse(result.isSuccess());
        assertTrue(result.getErrorMessage().contains("Payment processing error"));
    }
}

class PaymentProcessorTest {
    private PaymentProcessor processor;
    private CardValidator mockValidator;
    private PaymentGateway mockGateway;
    private TransactionRepository mockRepository;
    private PaymentReceiptService mockReceiptService;
    
    @BeforeEach
    void setUp() {
        mockValidator = mock(CardValidator.class);
        mockGateway = mock(PaymentGateway.class);
        mockRepository = mock(TransactionRepository.class);
        mockReceiptService = mock(PaymentReceiptService.class);
        
        processor = new PaymentProcessor(
            mockValidator, mockGateway, mockRepository, mockReceiptService
        );
    }
    
    @Test
    void shouldProcessPaymentSuccessfully() {
        String cardNumber = "4532015112830366";
        double amount = 100.0;
        
        when(mockValidator.validate(cardNumber)).thenReturn(ValidationResult.success());
        when(mockGateway.charge(cardNumber, amount))
            .thenReturn(PaymentResult.success("txn_123"));
        
        ProcessingResult result = processor.processPayment(cardNumber, amount);
        
        assertTrue(result.isSuccess());
        verify(mockRepository).save(any(Transaction.class));
        verify(mockReceiptService).sendReceipt(eq(cardNumber), eq(amount), anyString());
    }
    
    @Test
    void shouldFailWhenCardInvalid() {
        String cardNumber = "invalid";
        
        when(mockValidator.validate(cardNumber))
            .thenReturn(ValidationResult.failure("Invalid card"));
        
        ProcessingResult result = processor.processPayment(cardNumber, 100.0);
        
        assertFalse(result.isSuccess());
        verify(mockGateway, never()).charge(anyString(), anyDouble());
        verify(mockRepository, never()).save(any());
    }
}
```

**5. Test de Integración:**

```java
@Testcontainers
class PaymentProcessorIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:14");
    
    @Test
    void shouldProcessPaymentEndToEnd() {
        // Setup real components
        DataSource dataSource = createDataSource(postgres.getJdbcUrl());
        TransactionRepository repository = new JdbcTransactionRepository(dataSource);
        
        // Mock external services
        PaymentGateway mockGateway = mock(PaymentGateway.class);
        when(mockGateway.charge(anyString(), anyDouble()))
            .thenReturn(PaymentResult.success("txn_integration_123"));
        
        PaymentReceiptService mockReceiptService = mock(PaymentReceiptService.class);
        
        // Real validator
        CardValidator validator = new CardValidator();
        
        PaymentProcessor processor = new PaymentProcessor(
            validator, mockGateway, repository, mockReceiptService
        );
        
        // Execute
        ProcessingResult result = processor.processPayment("4532015112830366", 250.0);
        
        // Verify
        assertTrue(result.isSuccess());
        
        Transaction saved = repository.findById(result.getTransactionId());
        assertNotNull(saved);
        assertEquals(250.0, saved.getAmount());
        assertEquals(TransactionStatus.SUCCESS, saved.getStatus());
    }
}
```

---

## Nivel 3: TDD con SRP ⭐⭐⭐

### Ejercicio 3.1. Desarrollar con TDD

Usa TDD para crear un sistema de gestión de inventario que cumpla SRP.

**Requisitos:**
1. Validar productos antes de agregar
2. Rastrear stock de productos
3. Generar alertas cuando stock < 10
4. Registrar movimientos de inventario
5. Calcular valor total del inventario

**Proceso TDD:**
- Escribe test PRIMERO
- Implementa MÍNIMO código para pasar
- Refactoriza manteniendo tests verdes
- Repite

**Entregables:**
- Mínimo 15 tests (uno por funcionalidad)
- 5+ clases con responsabilidad única
- Cobertura >85%

---

## Nivel 4: Mutation Testing ⭐⭐⭐⭐

### Ejercicio 4.1. Validar Calidad de Tests

Usa PIT (Mutation Testing) para validar la calidad de tus tests.

**Tareas:**
1. Implementa un sistema de descuentos con SRP
2. Escribe tests unitarios
3. Ejecuta PIT mutation testing
4. Alcanza >80% mutation score
5. Documenta qué mutaciones sobrevivieron y por qué

**Ejemplo de configuración PIT (Maven):**
```xml
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.15.0</version>
    <configuration>
        <targetClasses>
            <param>com.example.discounts.*</param>
        </targetClasses>
        <targetTests>
            <param>com.example.discounts.*Test</param>
        </targetTests>
    </configuration>
</plugin>
```

---

## Rúbrica de Evaluación

### Nivel 1 (20 puntos)
- [ ] Identifica problemas de testabilidad correctamente (10 pts)
- [ ] Compara estrategias de testing con métricas (10 pts)

### Nivel 2 (30 puntos)
- [ ] Extrae clases con responsabilidades únicas (10 pts)
- [ ] Tests unitarios aislados y rápidos (10 pts)
- [ ] Test de integración funcional (10 pts)

### Nivel 3 (25 puntos)
- [ ] Sigue ciclo TDD correctamente (10 pts)
- [ ] Implementa >5 clases con SRP (8 pts)
- [ ] Cobertura >85% (7 pts)

### Nivel 4 (25 puntos)
- [ ] Implementa sistema con SRP (8 pts)
- [ ] Alcanza >80% mutation score (12 pts)
- [ ] Documenta mutaciones sobrevivientes (5 pts)

**Total: 100 puntos**
