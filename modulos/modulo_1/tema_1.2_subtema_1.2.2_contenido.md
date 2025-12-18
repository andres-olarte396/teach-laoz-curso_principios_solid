# Subtema 1.2.2: SRP y Testing - La Relación Fundamental

## 1. Introducción

El testing es el mejor indicador de SRP. Si una clase es difícil de testear, probablemente viola SRP.

**Relación clave:**
- **SRP correcto** → Testing simple y aislado
- **SRP violado** → Testing complejo con muchos mocks

## 2. Por Qué SRP Facilita el Testing

### 2.1 Comparación Directa

**❌ Sin SRP: Clase Difícil de Testear**

```java
public class UserService {
    public void registerUser(String email, String password) {
        // Validación
        if (!email.contains("@")) {
            throw new IllegalArgumentException("Invalid email");
        }
        
        // Hash de contraseña
        String hashedPassword = BCrypt.hashpw(password, BCrypt.gensalt());
        
        // Guardar en BD
        Connection conn = DriverManager.getConnection("jdbc:mysql://localhost/users");
        String sql = "INSERT INTO users (email, password) VALUES (?, ?)";
        PreparedStatement stmt = conn.prepareStatement(sql);
        stmt.setString(1, email);
        stmt.setString(2, hashedPassword);
        stmt.executeUpdate();
        
        // Enviar email de bienvenida
        EmailSender sender = new EmailSender("smtp.gmail.com");
        sender.send(email, "Welcome!", "Thanks for registering");
    }
}
```

**Problemas para testing:**
```java
@Test
void testRegisterUser() {
    UserService service = new UserService();
    
    // ❌ Problema 1: Necesitas BD MySQL corriendo
    // ❌ Problema 2: Necesitas servidor SMTP
    // ❌ Problema 3: No puedes verificar hash sin lógica acoplada
    // ❌ Problema 4: Test lento (red, BD, bcrypt)
    // ❌ Problema 5: Test frágil (falla si BD está caída)
    
    service.registerUser("test@example.com", "password123");
    
    // ¿Cómo verificar que funcionó sin consultar BD real?
}
```

**✅ Con SRP: Clases Fáciles de Testear**

```java
// Validador (puro, sin dependencias)
public class EmailValidator {
    public boolean isValid(String email) {
        return email != null && email.contains("@") && email.contains(".");
    }
}

// Password hasher (sin estado)
public class PasswordHasher {
    public String hash(String password) {
        return BCrypt.hashpw(password, BCrypt.gensalt());
    }
    
    public boolean verify(String password, String hash) {
        return BCrypt.checkpw(password, hash);
    }
}

// Repositorio (interfaz inyectable)
public interface UserRepository {
    void save(User user);
    User findByEmail(String email);
}

// Servicio de email (interfaz inyectable)
public interface EmailService {
    void send(String to, String subject, String body);
}

// Servicio principal (orquestador con dependencias inyectadas)
public class UserRegistrationService {
    private final EmailValidator emailValidator;
    private final PasswordHasher passwordHasher;
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    public UserRegistrationService(EmailValidator emailValidator,
                                   PasswordHasher passwordHasher,
                                   UserRepository userRepository,
                                   EmailService emailService) {
        this.emailValidator = emailValidator;
        this.passwordHasher = passwordHasher;
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
    
    public void registerUser(String email, String password) {
        if (!emailValidator.isValid(email)) {
            throw new IllegalArgumentException("Invalid email");
        }
        
        String hashedPassword = passwordHasher.hash(password);
        
        User user = new User(email, hashedPassword);
        userRepository.save(user);
        
        emailService.send(email, "Welcome!", "Thanks for registering");
    }
}
```

**Tests ahora son simples:**

```java
class EmailValidatorTest {
    private EmailValidator validator;
    
    @BeforeEach
    void setUp() {
        validator = new EmailValidator();
    }
    
    @Test
    void shouldAcceptValidEmail() {
        assertTrue(validator.isValid("user@example.com"));
    }
    
    @Test
    void shouldRejectEmailWithoutAt() {
        assertFalse(validator.isValid("userexample.com"));
    }
    
    @Test
    void shouldRejectEmailWithoutDot() {
        assertFalse(validator.isValid("user@examplecom"));
    }
}

class UserRegistrationServiceTest {
    private UserRegistrationService service;
    private EmailValidator mockValidator;
    private PasswordHasher mockHasher;
    private UserRepository mockRepository;
    private EmailService mockEmailService;
    
    @BeforeEach
    void setUp() {
        mockValidator = mock(EmailValidator.class);
        mockHasher = mock(PasswordHasher.class);
        mockRepository = mock(UserRepository.class);
        mockEmailService = mock(EmailService.class);
        
        service = new UserRegistrationService(
            mockValidator, mockHasher, mockRepository, mockEmailService
        );
    }
    
    @Test
    void shouldRegisterValidUser() {
        String email = "user@example.com";
        String password = "password123";
        String hashedPassword = "hashed123";
        
        when(mockValidator.isValid(email)).thenReturn(true);
        when(mockHasher.hash(password)).thenReturn(hashedPassword);
        
        service.registerUser(email, password);
        
        verify(mockRepository).save(argThat(user -> 
            user.getEmail().equals(email) && 
            user.getPassword().equals(hashedPassword)
        ));
        verify(mockEmailService).send(email, "Welcome!", "Thanks for registering");
    }
    
    @Test
    void shouldRejectInvalidEmail() {
        when(mockValidator.isValid("invalid")).thenReturn(false);
        
        assertThrows(IllegalArgumentException.class, 
            () -> service.registerUser("invalid", "password123")
        );
        
        verify(mockRepository, never()).save(any());
        verify(mockEmailService, never()).send(anyString(), anyString(), anyString());
    }
}
```

**Ventajas:**
- ✅ **Rápido**: Sin BD, sin red, sin bcrypt real
- ✅ **Aislado**: Cada test verifica UNA cosa
- ✅ **Confiable**: No depende de infraestructura externa
- ✅ **Legible**: Test expresa intención claramente

## 3. Test-Driven Development (TDD) y SRP

### 3.1 Ciclo Red-Green-Refactor con SRP

TDD naturalmente conduce a SRP porque:
- Cada test verifica UNA funcionalidad
- Funcionalidades complejas requieren mocks complejos
- Complejidad de mocks indica violación de SRP

**Ejemplo: Desarrollo TDD de un Validador**

**Test 1 (Red):**
```java
@Test
void shouldValidateMinimumLength() {
    PasswordValidator validator = new PasswordValidator();
    
    assertFalse(validator.isValid("12345")); // < 8 caracteres
    assertTrue(validator.isValid("12345678")); // >= 8 caracteres
}
```

**Implementación (Green):**
```java
public class PasswordValidator {
    public boolean isValid(String password) {
        return password != null && password.length() >= 8;
    }
}
```

**Test 2 (Red):**
```java
@Test
void shouldRequireUppercaseLetter() {
    PasswordValidator validator = new PasswordValidator();
    
    assertFalse(validator.isValid("password123"));
    assertTrue(validator.isValid("Password123"));
}
```

**Implementación (Green):**
```java
public class PasswordValidator {
    public boolean isValid(String password) {
        if (password == null || password.length() < 8) {
            return false;
        }
        
        return password.chars().anyMatch(Character::isUpperCase);
    }
}
```

**Observación**: `PasswordValidator` tiene UNA responsabilidad clara: validar contraseñas.

### 3.2 Cuándo NO Separar

**❌ Sobre-fragmentación innecesaria:**

```java
// Demasiado granular
public class PasswordLengthValidator {
    public boolean hasMinimumLength(String password) {
        return password.length() >= 8;
    }
}

public class PasswordUppercaseValidator {
    public boolean hasUppercase(String password) {
        return password.chars().anyMatch(Character::isUpperCase);
    }
}

public class PasswordLowercaseValidator {
    public boolean hasLowercase(String password) {
        return password.chars().anyMatch(Character::isLowerCase);
    }
}

public class PasswordDigitValidator {
    public boolean hasDigit(String password) {
        return password.chars().anyMatch(Character::isDigit);
    }
}

// Orquestador que es más complejo que la lógica
public class PasswordValidator {
    private final PasswordLengthValidator lengthValidator;
    private final PasswordUppercaseValidator uppercaseValidator;
    private final PasswordLowercaseValidator lowercaseValidator;
    private final PasswordDigitValidator digitValidator;
    
    // Constructor gigante
    
    public boolean isValid(String password) {
        return lengthValidator.hasMinimumLength(password) &&
               uppercaseValidator.hasUppercase(password) &&
               lowercaseValidator.hasLowercase(password) &&
               digitValidator.hasDigit(password);
    }
}
```

**✅ Granularidad adecuada:**

```java
public class PasswordValidator {
    private static final int MIN_LENGTH = 8;
    
    public ValidationResult validate(String password) {
        List<String> errors = new ArrayList<>();
        
        if (password == null || password.length() < MIN_LENGTH) {
            errors.add("Password must be at least " + MIN_LENGTH + " characters");
        }
        
        if (password != null && !password.chars().anyMatch(Character::isUpperCase)) {
            errors.add("Password must contain at least one uppercase letter");
        }
        
        if (password != null && !password.chars().anyMatch(Character::isLowerCase)) {
            errors.add("Password must contain at least one lowercase letter");
        }
        
        if (password != null && !password.chars().anyMatch(Character::isDigit)) {
            errors.add("Password must contain at least one digit");
        }
        
        return errors.isEmpty() 
            ? ValidationResult.success() 
            : ValidationResult.failure(errors);
    }
}
```

**Test simple:**
```java
@Test
void shouldRejectWeakPassword() {
    PasswordValidator validator = new PasswordValidator();
    
    ValidationResult result = validator.validate("weak");
    
    assertFalse(result.isValid());
    assertEquals(3, result.getErrors().size());
}
```

## 4. Patrones de Testing para SRP

### 4.1 Test Doubles (Mocks, Stubs, Fakes)

**Mock**: Verifica interacciones

```java
@Test
void shouldNotifyUserWhenOrderConfirmed() {
    NotificationService mockNotificationService = mock(NotificationService.class);
    OrderService orderService = new OrderService(mockNotificationService);
    
    orderService.confirmOrder(order);
    
    verify(mockNotificationService).sendConfirmation(order.getCustomerId());
}
```

**Stub**: Proporciona respuestas predefinidas

```java
@Test
void shouldCalculatePriceWithDiscount() {
    DiscountCalculator stubCalculator = mock(DiscountCalculator.class);
    when(stubCalculator.calculate(any())).thenReturn(10.0);
    
    PricingService pricingService = new PricingService(stubCalculator);
    
    double total = pricingService.calculateTotal(order);
    
    assertEquals(90.0, total); // 100 - 10
}
```

**Fake**: Implementación simplificada

```java
public class InMemoryUserRepository implements UserRepository {
    private Map<String, User> users = new HashMap<>();
    
    @Override
    public void save(User user) {
        users.put(user.getId(), user);
    }
    
    @Override
    public User findById(String id) {
        return users.get(id);
    }
}

@Test
void shouldSaveAndRetrieveUser() {
    UserRepository fakeRepo = new InMemoryUserRepository();
    UserService userService = new UserService(fakeRepo);
    
    User user = new User("1", "John");
    userService.register(user);
    
    User retrieved = userService.findById("1");
    assertEquals("John", retrieved.getName());
}
```

### 4.2 Testing Pyramid y SRP

```
        /\
       /E2E\        Pocos tests E2E (integración completa)
      /------\
     /Integration\  Tests de integración (2-3 servicios)
    /------------\
   /  Unit Tests  \ Muchos tests unitarios (clase individual)
  /________________\
```

**SRP facilita la base de la pirámide (Unit Tests):**
- Clases pequeñas → Tests unitarios simples
- Responsabilidades aisladas → Menos mocks
- Alta cobertura → Confianza en refactoring

**Ejemplo de distribución:**

```java
// Unit Tests (70-80% del total)
EmailValidatorTest (10 tests)
PasswordHasherTest (8 tests)
OrderPricingServiceTest (15 tests)
InventoryServiceTest (12 tests)

// Integration Tests (15-25% del total)
OrderProcessingIntegrationTest (8 tests)
  - Valida interacción entre OrderService, InventoryService, PaymentService

// E2E Tests (5-10% del total)
CompleteOrderFlowTest (3 tests)
  - Simula usuario comprando desde UI hasta confirmación
```

## 5. Cobertura de Código y SRP

### 5.1 Métricas de Cobertura

**Clase con SRP:**
```java
public class TaxCalculator {
    private static final double DEFAULT_RATE = 0.07;
    
    public double calculate(double amount, Address address) {
        double rate = getRateForState(address.getState());
        return amount * rate;
    }
    
    private double getRateForState(String state) {
        switch (state) {
            case "CA": return 0.0725;
            case "NY": return 0.08;
            case "TX": return 0.0625;
            default: return DEFAULT_RATE;
        }
    }
}
```

**Cobertura 100% con pocos tests:**
```java
@Test
void shouldCalculateTaxForCalifornia() {
    TaxCalculator calculator = new TaxCalculator();
    Address address = new Address("CA");
    
    double tax = calculator.calculate(100.0, address);
    
    assertEquals(7.25, tax, 0.01);
}

@Test
void shouldCalculateTaxForNewYork() {
    // Similar
}

@Test
void shouldCalculateTaxForTexas() {
    // Similar
}

@Test
void shouldUseDefaultRateForUnknownState() {
    TaxCalculator calculator = new TaxCalculator();
    Address address = new Address("ZZ");
    
    double tax = calculator.calculate(100.0, address);
    
    assertEquals(7.0, tax, 0.01);
}
```

**Clase sin SRP:**
```java
public class OrderManager {
    // 1200 líneas, 50+ métodos, 7 responsabilidades
}
```

**Cobertura parcial con muchos tests:**
- 200+ tests para llegar a 60% cobertura
- Tests frágiles que fallan frecuentemente
- Dificultad para aislar fallos

### 5.2 Mutation Testing y SRP

**Mutation testing** introduce bugs intencionales para verificar que tests detectan cambios.

**Con SRP:**
```java
public class DiscountCalculator {
    public double calculate(double amount) {
        if (amount > 100) {
            return amount * 0.1; // 10% descuento
        }
        return 0;
    }
}
```

**Mutación: Cambiar 0.1 a 0.2**
```java
return amount * 0.2; // Test debería fallar
```

**Test que detecta mutación:**
```java
@Test
void shouldCalculate10PercentDiscount() {
    DiscountCalculator calculator = new DiscountCalculator();
    
    double discount = calculator.calculate(200.0);
    
    assertEquals(20.0, discount, 0.01); // Falla si es 0.2 (40.0)
}
```

**Sin SRP**: Mutaciones pueden pasar desapercibidas porque tests complejos no cubren todos los casos.

## 6. Refactoring Guiado por Tests

### 6.1 Estrategia: Characterization Tests

Cuando heredas código legacy sin tests:

**Paso 1**: Escribir tests que describan comportamiento actual

```java
@Test
void characterizeCurrentBehavior() {
    LegacyOrderProcessor processor = new LegacyOrderProcessor();
    
    // Capturar comportamiento actual (aunque sea incorrecto)
    Order result = processor.process(testOrder);
    
    assertEquals(123.45, result.getTotal()); // Lo que hace AHORA
}
```

**Paso 2**: Refactorizar hacia SRP manteniendo tests verdes

```java
// Extraer validación
OrderValidator validator = new OrderValidator();
// Tests pasan (mismo comportamiento)

// Extraer cálculo de precios
PricingService pricingService = new PricingService();
// Tests siguen pasando

// Actualizar LegacyOrderProcessor para usar servicios
// Tests SIGUEN pasando
```

**Paso 3**: Mejorar tests ahora que hay SRP

```java
@Test
void shouldValidateOrderCorrectly() {
    OrderValidator validator = new OrderValidator();
    
    ValidationResult result = validator.validate(invalidOrder);
    
    assertFalse(result.isValid());
    assertTrue(result.getErrors().contains("Invalid email"));
}
```

## 7. Herramientas de Testing y SRP

### 7.1 JUnit 5 + Mockito

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock
    private OrderRepository repository;
    
    @Mock
    private PaymentService paymentService;
    
    @InjectMocks
    private OrderService orderService;
    
    @Test
    void shouldProcessOrder() {
        when(paymentService.process(any())).thenReturn(PaymentResult.success());
        
        orderService.processOrder(order);
        
        verify(repository).save(order);
    }
}
```

### 7.2 Test Containers (Integration Tests)

```java
@Testcontainers
class OrderRepositoryIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:14");
    
    @Test
    void shouldSaveAndRetrieveOrder() {
        OrderRepository repository = new JdbcOrderRepository(postgres.getJdbcUrl());
        
        repository.save(order);
        Order retrieved = repository.findById(order.getId());
        
        assertEquals(order.getId(), retrieved.getId());
    }
}
```

## 8. Resumen Ejecutivo

**Relación SRP ↔ Testing:**
- SRP correcto → Tests unitarios simples
- Violación de SRP → Tests complejos con muchos mocks

**Indicadores de buen SRP:**
- ✅ Test de 10-20 líneas
- ✅ 0-2 mocks por test
- ✅ Cobertura >80% fácilmente alcanzable
- ✅ Tests rápidos (<100ms)

**Indicadores de SRP violado:**
- ❌ Test de 50+ líneas
- ❌ 5+ mocks por test
- ❌ Cobertura <60% con 200+ tests
- ❌ Tests lentos (>1s)

## 9. Puntos Clave

✅ **SRP facilita testing** reduciendo dependencias  
✅ **TDD conduce naturalmente a SRP**  
✅ **Mocks complejos** indican violación de SRP  
✅ **Test pyramid** funciona mejor con SRP  
✅ **Cobertura alta** es más fácil con clases pequeñas  
✅ **Mutation testing** valida calidad de tests  
❌ Evitar **sobre-fragmentación** que complica tests  
❌ No confundir **muchos tests con buenos tests**
