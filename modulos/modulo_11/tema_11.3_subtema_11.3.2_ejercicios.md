# Ejercicios: Shared Libraries y DIP

## ⭐ Ejercicio 1. Identificar Violaciones DIP en Shared Library

Analiza esta shared library y encuentra violaciones de DIP:

```java
// common-library/src/main/java/OrderValidator.java
public class OrderValidator {
    private final PostgresConnection db;
    private final StripeAPI stripeClient;
    
    public ValidationResult validate(Order order) {
        // Check customer exists in database
        String query = "SELECT * FROM customers WHERE id = ?";
        Customer customer = db.query(query, order.getCustomerId());
        
        if (customer == null) {
            return ValidationResult.error("Customer not found");
        }
        
        // Validate payment method with Stripe
        boolean validCard = stripeClient.validateCard(order.getPaymentMethod());
        
        if (!validCard) {
            return ValidationResult.error("Invalid payment method");
        }
        
        return ValidationResult.success();
    }
}
```

**Tareas**:
1. Identificar todas las violaciones DIP
2. Refactorizar usando abstractions
3. Mover implementaciones fuera de shared library

<details>
<summary>💡 Solución</summary>

**Violaciones**:
1. ❌ Dependencia directa a `PostgresConnection` (implementación concreta)
2. ❌ Dependencia directa a `StripeAPI` (implementación concreta)
3. ❌ Lógica de validación mezclada con acceso a datos
4. ❌ Shared library contiene lógica de negocio

**Refactorización**:

```java
// shared-contracts/src/main/java/CustomerRepository.java
public interface CustomerRepository {
    Optional<Customer> findById(String customerId);
}

// shared-contracts/src/main/java/PaymentValidator.java
public interface PaymentValidator {
    boolean isValid(PaymentMethod method);
}

// shared-contracts/src/main/java/OrderValidator.java
public class OrderValidator {
    // Depends on abstractions, not implementations
    private final CustomerRepository customerRepository;
    private final PaymentValidator paymentValidator;
    
    public OrderValidator(
        CustomerRepository customerRepository,
        PaymentValidator paymentValidator
    ) {
        this.customerRepository = customerRepository;
        this.paymentValidator = paymentValidator;
    }
    
    public ValidationResult validate(Order order) {
        // Check customer exists
        Optional<Customer> customer = customerRepository.findById(order.getCustomerId());
        if (customer.isEmpty()) {
            return ValidationResult.error("Customer not found");
        }
        
        // Validate payment method
        if (!paymentValidator.isValid(order.getPaymentMethod())) {
            return ValidationResult.error("Invalid payment method");
        }
        
        return ValidationResult.success();
    }
}

// order-service/src/main/java/PostgresCustomerRepository.java
// Implementation in service, not in shared library
public class PostgresCustomerRepository implements CustomerRepository {
    private final DataSource dataSource;
    
    @Override
    public Optional<Customer> findById(String customerId) {
        String query = "SELECT * FROM customers WHERE id = ?";
        // PostgreSQL-specific implementation
        return Optional.ofNullable(
            jdbcTemplate.queryForObject(query, new CustomerMapper(), customerId)
        );
    }
}

// payment-service/src/main/java/StripePaymentValidator.java
public class StripePaymentValidator implements PaymentValidator {
    private final StripeAPI stripeClient;
    
    @Override
    public boolean isValid(PaymentMethod method) {
        return stripeClient.validateCard(method);
    }
}
```

</details>

---

## ⭐⭐ Ejercicio 2: Diseñar Value Object Library

Crea shared library con value objects para e-commerce.

**Requisitos**:
- `Money` (amount, currency)
- `Address` (street, city, state, zip, country)
- `Email` (validated)
- `PhoneNumber` (validated, formatted)
- Immutables, con validación
- Sin dependencias externas

<details>
<summary>💡 Solución</summary>

```kotlin
// shared-domain/src/main/kotlin/Money.kt
@JvmInline
value class Money private constructor(private val cents: Long) : Comparable<Money> {
    
    init {
        require(cents >= 0) { "Money cannot be negative" }
    }
    
    val amount: BigDecimal
        get() = BigDecimal(cents).divide(BigDecimal(100), 2, RoundingMode.HALF_UP)
    
    operator fun plus(other: Money): Money = Money(cents + other.cents)
    operator fun minus(other: Money): Money {
        require(cents >= other.cents) { "Insufficient funds" }
        return Money(cents - other.cents)
    }
    
    operator fun times(multiplier: Int): Money = Money(cents * multiplier)
    
    fun multiply(multiplier: BigDecimal): Money {
        val newCents = amount.multiply(multiplier)
            .multiply(BigDecimal(100))
            .toLong()
        return Money(newCents)
    }
    
    override fun compareTo(other: Money): Int = cents.compareTo(other.cents)
    
    override fun toString(): String = "$$amount"
    
    companion object {
        fun zero(): Money = Money(0)
        
        fun of(dollars: Int, cents: Int = 0): Money {
            require(cents in 0..99) { "Cents must be 0-99" }
            return Money(dollars * 100L + cents)
        }
        
        fun of(amount: BigDecimal): Money {
            require(amount >= BigDecimal.ZERO) { "Amount cannot be negative" }
            val cents = amount.multiply(BigDecimal(100)).toLong()
            return Money(cents)
        }
    }
}

// shared-domain/src/main/kotlin/Address.kt
data class Address(
    val street: String,
    val city: String,
    val state: String,
    val zipCode: String,
    val country: String
) {
    init {
        require(street.isNotBlank()) { "Street cannot be blank" }
        require(city.isNotBlank()) { "City cannot be blank" }
        require(state.matches(Regex("[A-Z]{2}"))) { "State must be 2-letter code" }
        require(zipCode.matches(Regex("\\d{5}(-\\d{4})?"))) { "Invalid ZIP code" }
        require(country.matches(Regex("[A-Z]{2}"))) { "Country must be 2-letter ISO code" }
    }
    
    fun format(): String = """
        $street
        $city, $state $zipCode
        $country
    """.trimIndent()
}

// shared-domain/src/main/kotlin/Email.kt
@JvmInline
value class Email private constructor(val value: String) {
    
    init {
        require(isValid(value)) { "Invalid email format: $value" }
    }
    
    val domain: String
        get() = value.substringAfter('@')
    
    val localPart: String
        get() = value.substringBefore('@')
    
    override fun toString(): String = value
    
    companion object {
        private val EMAIL_REGEX = Regex(
            "^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}$"
        )
        
        fun of(value: String): Email {
            return Email(value.lowercase())
        }
        
        fun isValid(value: String): Boolean {
            return EMAIL_REGEX.matches(value)
        }
    }
}

// shared-domain/src/main/kotlin/PhoneNumber.kt
@JvmInline
value class PhoneNumber private constructor(private val digits: String) {
    
    init {
        require(digits.length in 10..15) { "Phone number must be 10-15 digits" }
        require(digits.all { it.isDigit() }) { "Phone number must contain only digits" }
    }
    
    fun format(): String = when (digits.length) {
        10 -> "(${digits.substring(0, 3)}) ${digits.substring(3, 6)}-${digits.substring(6)}"
        11 -> "+${digits[0]} (${digits.substring(1, 4)}) ${digits.substring(4, 7)}-${digits.substring(7)}"
        else -> "+$digits"
    }
    
    override fun toString(): String = format()
    
    companion object {
        fun of(value: String): PhoneNumber {
            // Remove all non-digit characters
            val digits = value.filter { it.isDigit() }
            return PhoneNumber(digits)
        }
    }
}

// Tests
class MoneyTest {
    @Test
    fun `should add money correctly`() {
        val money1 = Money.of(10, 50)
        val money2 = Money.of(5, 25)
        
        assertThat(money1 + money2).isEqualTo(Money.of(15, 75))
    }
    
    @Test
    fun `should throw on negative amount`() {
        assertThrows<IllegalArgumentException> {
            Money.of(-10, 0)
        }
    }
}

class AddressTest {
    @Test
    fun `should validate state code`() {
        assertThrows<IllegalArgumentException> {
            Address("123 Main St", "Boston", "Massachusetts", "02101", "US")
        }
    }
    
    @Test
    fun `should accept valid address`() {
        val address = Address("123 Main St", "Boston", "MA", "02101", "US")
        assertThat(address.format()).contains("Boston, MA 02101")
    }
}
```

</details>

---

## ⭐⭐⭐ Ejercicio 3: Implementar Versioning con Backward Compatibility

Tienes esta API v1:

```java
public interface NotificationService {
    void sendEmail(String to, String subject, String body);
}
```

**Necesitas agregar**:
1. Soporte para HTML emails
2. Attachments
3. CC/BCC recipients
4. Templates

**Tareas**:
1. Crear v2 con nuevas features
2. Mantener v1 compatible
3. Proporcionar migration path
4. Versioning plan (6 meses)

<details>
<summary>💡 Solución</summary>

```java
// shared-contracts v2.0.0

// V1 - Deprecated but still supported
@Deprecated(since = "2.0.0", forRemoval = true)
public interface NotificationService {
    /**
     * @deprecated Use {@link NotificationServiceV2#send(EmailRequest)} instead.
     * This method will be removed in v3.0.0 (July 2024).
     */
    @Deprecated
    void sendEmail(String to, String subject, String body);
}

// V2 - New interface
public interface NotificationServiceV2 {
    
    /**
     * Send email with full configuration.
     * @since 2.0.0
     */
    EmailResult send(EmailRequest request);
    
    /**
     * Send email using template.
     * @since 2.1.0
     */
    EmailResult sendFromTemplate(TemplateEmailRequest request);
}

// EmailRequest (builder pattern for backward compatibility)
public class EmailRequest {
    private final String to;
    private final String subject;
    private final String body;
    private final EmailFormat format;  // TEXT or HTML
    private final List<String> cc;
    private final List<String> bcc;
    private final List<Attachment> attachments;
    
    private EmailRequest(Builder builder) {
        this.to = builder.to;
        this.subject = builder.subject;
        this.body = builder.body;
        this.format = builder.format;
        this.cc = builder.cc;
        this.bcc = builder.bcc;
        this.attachments = builder.attachments;
    }
    
    public static Builder builder(String to, String subject, String body) {
        return new Builder(to, subject, body);
    }
    
    public static class Builder {
        private final String to;
        private final String subject;
        private final String body;
        private EmailFormat format = EmailFormat.TEXT;  // Default
        private List<String> cc = List.of();
        private List<String> bcc = List.of();
        private List<Attachment> attachments = List.of();
        
        private Builder(String to, String subject, String body) {
            this.to = to;
            this.subject = subject;
            this.body = body;
        }
        
        public Builder html() {
            this.format = EmailFormat.HTML;
            return this;
        }
        
        public Builder cc(String... recipients) {
            this.cc = List.of(recipients);
            return this;
        }
        
        public Builder bcc(String... recipients) {
            this.bcc = List.of(recipients);
            return this;
        }
        
        public Builder attach(Attachment... files) {
            this.attachments = List.of(files);
            return this;
        }
        
        public EmailRequest build() {
            return new EmailRequest(this);
        }
    }
}

// Compatibility adapter
public class NotificationServiceAdapter implements NotificationService {
    private final NotificationServiceV2 delegate;
    
    public NotificationServiceAdapter(NotificationServiceV2 delegate) {
        this.delegate = delegate;
    }
    
    @Override
    public void sendEmail(String to, String subject, String body) {
        // Adapt V1 call to V2
        EmailRequest request = EmailRequest.builder(to, subject, body).build();
        delegate.send(request);
    }
}

// Migration Guide
/**
 * MIGRATION GUIDE: NotificationService v1 -> v2
 * 
 * Timeline:
 * - v2.0.0 (Jan 2024): V2 released, V1 deprecated with WARNING
 * - v2.5.0 (Apr 2024): V1 shows ERROR-level deprecation
 * - v3.0.0 (Jul 2024): V1 removed
 * 
 * Migration Examples:
 * 
 * BEFORE (v1):
 * notificationService.sendEmail(
 *     "user@example.com",
 *     "Welcome",
 *     "Welcome to our service"
 * );
 * 
 * AFTER (v2 - simple):
 * EmailRequest request = EmailRequest.builder(
 *     "user@example.com",
 *     "Welcome",
 *     "Welcome to our service"
 * ).build();
 * notificationService.send(request);
 * 
 * AFTER (v2 - with HTML and CC):
 * EmailRequest request = EmailRequest.builder(
 *     "user@example.com",
 *     "Welcome",
 *     "<h1>Welcome</h1><p>Welcome to our service</p>"
 * )
 * .html()
 * .cc("manager@example.com")
 * .build();
 * notificationService.send(request);
 */
```

**Versioning Plan Documentation**:

```markdown
# Notification Service Versioning

## Version 2.0.0 (January 2024)

### New Features
- HTML email support
- CC/BCC recipients
- File attachments
- Email templates

### Deprecated
- `NotificationService.sendEmail(String, String, String)` 
  - Use `NotificationServiceV2.send(EmailRequest)` instead
  - Will be removed in v3.0.0 (July 2024)

### Migration Period
- **January - April 2024**: WARNING-level deprecation
- **April - July 2024**: ERROR-level deprecation
- **July 2024**: Complete removal in v3.0.0

### Backward Compatibility
All v1 functionality available via `NotificationServiceAdapter`

## Version 2.1.0 (March 2024)

### New Features
- Template-based emails via `sendFromTemplate()`

## Version 3.0.0 (July 2024) - BREAKING

### Removed
- `NotificationService` interface (v1)
- All v1 methods

### Required Actions
- Migrate all code to use `NotificationServiceV2`
- Update dependency: `shared-contracts:3.0.0`
```

</details>

---

## ⭐⭐⭐⭐ Ejercicio 4: Shared Library con Contract Tests

Crea shared library de pagos con:
1. `PaymentGateway` interface
2. Contract test suite
3. Implementaciones: Stripe, PayPal, Mock
4. Todas las implementaciones pasan contract tests

<details>
<summary>💡 Solución</summary>

```java
// shared-contracts/src/main/java/PaymentGateway.java
public interface PaymentGateway {
    
    /**
     * Process payment. Must be idempotent.
     * 
     * @param request Payment request with idempotency key
     * @return Payment result (Success, Declined, or Error)
     */
    PaymentResult charge(PaymentRequest request);
    
    /**
     * Refund a previously processed payment.
     * 
     * @param paymentId Original payment ID
     * @param amount Amount to refund (can be partial)
     * @return Refund result
     */
    RefundResult refund(String paymentId, Money amount);
    
    /**
     * Retrieve payment status.
     * 
     * @param paymentId Payment ID
     * @return Payment details
     */
    PaymentDetails getPayment(String paymentId);
}

// Contract test suite
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
public abstract class PaymentGatewayContractTest {
    
    /**
     * Implementations must provide a PaymentGateway instance.
     */
    protected abstract PaymentGateway createGateway();
    
    /**
     * Implementations must provide a valid payment method for testing.
     */
    protected abstract PaymentMethod getValidPaymentMethod();
    
    private PaymentGateway gateway;
    
    @BeforeAll
    void setup() {
        gateway = createGateway();
    }
    
    @Test
    @DisplayName("Should successfully charge valid payment method")
    void shouldChargeSuccessfully() {
        PaymentRequest request = PaymentRequest.builder()
            .orderId("test-order-1")
            .amount(Money.of(99, 99))
            .paymentMethod(getValidPaymentMethod())
            .idempotencyKey("test-key-1")
            .build();
        
        PaymentResult result = gateway.charge(request);
        
        assertThat(result).isInstanceOf(PaymentResult.Success.class);
        PaymentResult.Success success = (PaymentResult.Success) result;
        assertThat(success.paymentId()).isNotBlank();
        assertThat(success.processedAt()).isBeforeOrEqualTo(LocalDateTime.now());
    }
    
    @Test
    @DisplayName("Should be idempotent - same idempotency key returns same result")
    void shouldBeIdempotent() {
        String idempotencyKey = "idempotent-key-" + UUID.randomUUID();
        
        PaymentRequest request = PaymentRequest.builder()
            .orderId("test-order-2")
            .amount(Money.of(50, 0))
            .paymentMethod(getValidPaymentMethod())
            .idempotencyKey(idempotencyKey)
            .build();
        
        PaymentResult result1 = gateway.charge(request);
        PaymentResult result2 = gateway.charge(request);  // Same key
        
        assertThat(result1).isEqualTo(result2);
        if (result1 instanceof PaymentResult.Success s1 && 
            result2 instanceof PaymentResult.Success s2) {
            assertThat(s1.paymentId()).isEqualTo(s2.paymentId());
        }
    }
    
    @Test
    @DisplayName("Should decline payment with insufficient funds")
    void shouldDeclineInsufficientFunds() {
        // Use test card that triggers insufficient funds
        PaymentMethod testCard = PaymentMethod.creditCard(
            "4000000000000341",  // Stripe test card for insufficient funds
            12, 2025,
            "123"
        );
        
        PaymentRequest request = PaymentRequest.builder()
            .orderId("test-order-3")
            .amount(Money.of(1000, 0))
            .paymentMethod(testCard)
            .idempotencyKey("test-key-3")
            .build();
        
        PaymentResult result = gateway.charge(request);
        
        assertThat(result).isInstanceOf(PaymentResult.Declined.class);
        PaymentResult.Declined declined = (PaymentResult.Declined) result;
        assertThat(declined.reason()).containsIgnoringCase("insufficient");
    }
    
    @Test
    @DisplayName("Should successfully refund payment")
    void shouldRefundPayment() {
        // First, charge
        PaymentRequest chargeRequest = PaymentRequest.builder()
            .orderId("test-order-4")
            .amount(Money.of(100, 0))
            .paymentMethod(getValidPaymentMethod())
            .idempotencyKey("test-key-4")
            .build();
        
        PaymentResult chargeResult = gateway.charge(chargeRequest);
        assumeTrue(chargeResult instanceof PaymentResult.Success);
        
        String paymentId = ((PaymentResult.Success) chargeResult).paymentId();
        
        // Then, refund
        RefundResult refundResult = gateway.refund(paymentId, Money.of(100, 0));
        
        assertThat(refundResult).isInstanceOf(RefundResult.Success.class);
        RefundResult.Success success = (RefundResult.Success) refundResult;
        assertThat(success.refundId()).isNotBlank();
    }
    
    @Test
    @DisplayName("Should support partial refunds")
    void shouldSupportPartialRefunds() {
        // Charge $100
        PaymentRequest chargeRequest = PaymentRequest.builder()
            .orderId("test-order-5")
            .amount(Money.of(100, 0))
            .paymentMethod(getValidPaymentMethod())
            .idempotencyKey("test-key-5")
            .build();
        
        PaymentResult chargeResult = gateway.charge(chargeRequest);
        assumeTrue(chargeResult instanceof PaymentResult.Success);
        
        String paymentId = ((PaymentResult.Success) chargeResult).paymentId();
        
        // Refund $30
        RefundResult refundResult = gateway.refund(paymentId, Money.of(30, 0));
        
        assertThat(refundResult).isInstanceOf(RefundResult.Success.class);
        
        // Check payment details
        PaymentDetails details = gateway.getPayment(paymentId);
        assertThat(details.refundedAmount()).isEqualTo(Money.of(30, 0));
        assertThat(details.status()).isEqualTo(PaymentStatus.PARTIALLY_REFUNDED);
    }
    
    @Test
    @DisplayName("Should handle network errors gracefully")
    void shouldHandleNetworkErrors() {
        // This test depends on implementation - may need to inject fault
        // Implementation should catch network exceptions and return Error result
    }
    
    @Test
    @DisplayName("Should retrieve payment details")
    void shouldRetrievePaymentDetails() {
        PaymentRequest request = PaymentRequest.builder()
            .orderId("test-order-6")
            .amount(Money.of(75, 50))
            .paymentMethod(getValidPaymentMethod())
            .idempotencyKey("test-key-6")
            .build();
        
        PaymentResult result = gateway.charge(request);
        assumeTrue(result instanceof PaymentResult.Success);
        
        String paymentId = ((PaymentResult.Success) result).paymentId();
        PaymentDetails details = gateway.getPayment(paymentId);
        
        assertThat(details.amount()).isEqualTo(Money.of(75, 50));
        assertThat(details.status()).isEqualTo(PaymentStatus.COMPLETED);
    }
}

// Stripe implementation
public class StripePaymentGatewayTest extends PaymentGatewayContractTest {
    
    private Stripe stripe;
    
    @Override
    protected PaymentGateway createGateway() {
        stripe = new Stripe("sk_test_...");
        return new StripePaymentGateway(stripe);
    }
    
    @Override
    protected PaymentMethod getValidPaymentMethod() {
        // Stripe test card
        return PaymentMethod.creditCard("4242424242424242", 12, 2025, "123");
    }
}

// PayPal implementation
public class PayPalPaymentGatewayTest extends PaymentGatewayContractTest {
    
    private PayPalAPI paypal;
    
    @Override
    protected PaymentGateway createGateway() {
        paypal = new PayPalAPI("client-id", "secret", PayPalAPI.Environment.SANDBOX);
        return new PayPalPaymentGateway(paypal);
    }
    
    @Override
    protected PaymentMethod getValidPaymentMethod() {
        return PaymentMethod.paypal("test@example.com");
    }
}

// Mock implementation (for testing)
public class MockPaymentGatewayTest extends PaymentGatewayContractTest {
    
    @Override
    protected PaymentGateway createGateway() {
        return new MockPaymentGateway();
    }
    
    @Override
    protected PaymentMethod getValidPaymentMethod() {
        return PaymentMethod.creditCard("4111111111111111", 12, 2025, "123");
    }
}
```

</details>

---

## Proyecto Final: Shared Libraries Suite

Crea suite completa de shared libraries:
1. **domain-contracts**: Interfaces y DTOs
2. **domain-values**: Value objects (Money, Address, Email, etc.)
3. **observability**: Logging, tracing, metrics
4. **testing-contracts**: Contract test suites

**Incluir**:
- BOM para version management
- Contract tests para todas las interfaces
- Documentation con migration guides
- CI/CD pipeline para publish a artifact repository

**Rúbrica**:
- ⭐ Libraries con DIP correcto
- ⭐⭐ Value objects + versioning
- ⭐⭐⭐ Contract tests + BOM
- ⭐⭐⭐⭐ CI/CD + artifact repository + documentation completa

## Recursos

- **Semantic Versioning**: https://semver.org/
- **Contract Testing**: https://martinfowler.com/bliki/ContractTest.html
- **Domain-Driven Design**: Eric Evans
