# Ejercicios: Casos de Estudio de SRP en Sistemas Reales

## Nivel 1. Análisis de Código Real ⭐

### Ejercicio 1.1. Identificar Violaciones en Código Legacy

Analiza el siguiente fragmento inspirado en un sistema bancario real y responde:

```java
public class AccountManager {
    private Connection dbConn;
    
    public void transferMoney(String fromAccount, String toAccount, double amount) {
        // Validar
        if (amount <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        
        // Verificar saldo
        String sql1 = "SELECT balance FROM accounts WHERE id = ?";
        double balance = executeQuery(sql1, fromAccount);
        if (balance < amount) {
            throw new InsufficientFundsException();
        }
        
        // Debitar
        String sql2 = "UPDATE accounts SET balance = balance - ? WHERE id = ?";
        executeUpdate(sql2, amount, fromAccount);
        
        // Acreditar
        String sql3 = "UPDATE accounts SET balance = balance + ? WHERE id = ?";
        executeUpdate(sql3, amount, toAccount);
        
        // Registrar auditoría
        String sql4 = "INSERT INTO audit_log (action, account, amount, timestamp) VALUES (?, ?, ?, ?)";
        executeUpdate(sql4, "TRANSFER", fromAccount, amount, LocalDateTime.now());
        
        // Enviar notificación
        EmailService emailService = new EmailService();
        emailService.send(getAccountEmail(fromAccount), "Transfer Completed", 
                          "You transferred $" + amount);
        
        // Generar recibo
        String receipt = "RECEIPT\n" +
                        "From: " + fromAccount + "\n" +
                        "To: " + toAccount + "\n" +
                        "Amount: $" + amount;
        System.out.println(receipt);
    }
}
```

**Preguntas:**
1. ¿Cuántas responsabilidades tiene este método?
2. ¿Qué actores pedirían cambios a este código?
3. Lista 5 razones por las que este código podría cambiar
4. ¿Qué clases extraerías para aplicar SRP?

---

**SOLUCIÓN:**

**1. Responsabilidades identificadas (7):**
- Validación de entrada
- Verificación de saldo (lógica de negocio)
- Persistencia de datos (débito/crédito)
- Auditoría
- Notificaciones por email
- Generación de recibos
- Manejo directo de SQL

**2. Actores que pedirían cambios:**
- **Compliance Team**: Cambios en auditoría (nuevas regulaciones)
- **DBAs**: Optimización de queries, cambio de BD
- **Marketing**: Cambio de formato de emails
- **Product Owner**: Nueva regla de negocio (límites de transferencia)
- **Operations**: Logging adicional, métricas
- **Finance**: Formato de recibos, cálculo de comisiones

**3. Razones para cambiar:**
1. Nueva regulación requiere más datos en auditoría
2. Migración de MySQL a PostgreSQL
3. Cambio de proveedor de email
4. Agregación de límites diarios de transferencia
5. Generación de recibos en PDF en lugar de texto
6. Implementar transferencias programadas
7. Agregar autenticación de dos factores

**4. Clases a extraer:**
```java
// Validación
public class TransferValidator {
    ValidationResult validate(TransferRequest request);
}

// Lógica de negocio
public class TransferService {
    TransferResult execute(TransferRequest request);
}

// Persistencia
public interface AccountRepository {
    void debit(String accountId, double amount);
    void credit(String accountId, double amount);
    double getBalance(String accountId);
}

// Auditoría
public class AuditLogService {
    void logTransfer(String fromAccount, String toAccount, double amount);
}

// Notificaciones
public class TransferNotificationService {
    void sendConfirmation(String accountId, double amount);
}

// Recibos
public class ReceiptGenerator {
    Receipt generate(TransferRequest request);
}
```

---

### Ejercicio 1.2: Comparar Arquitecturas

Compara estas dos arquitecturas para un sistema de autenticación:

**Arquitectura A:**
```
AuthenticationManager (850 LOC)
├── validateCredentials()
├── hashPassword()
├── checkPasswordStrength()
├── saveToDatabase()
├── sendWelcomeEmail()
├── createSession()
├── logAuthAttempt()
└── generateToken()
```

**Arquitectura B:**
```
CredentialValidator (120 LOC)
PasswordHasher (80 LOC)
PasswordStrengthChecker (95 LOC)
UserRepository (110 LOC)
UserNotificationService (75 LOC)
SessionManager (140 LOC)
AuthAuditLogger (60 LOC)
TokenGenerator (90 LOC)
AuthenticationService (150 LOC) [orquestador]
```

**Preguntas:**
1. ¿Cuál arquitectura viola SRP? ¿Por qué?
2. Calcula el LOC promedio por clase en cada arquitectura
3. ¿Qué arquitectura es más fácil de testear? Justifica
4. Si debes cambiar el algoritmo de hashing, ¿cuántas clases modificas en cada arquitectura?

**Respuestas:**
1. **Arquitectura A** viola SRP (todas las responsabilidades en una clase)
2. **Arquitectura A**: 850 LOC/clase | **Arquitectura B**: ~102 LOC/clase promedio
3. **Arquitectura B** es más testeable (mocks simples, tests aislados)
4. **A**: 1 clase (pero riesgo alto de romper otras funciones) | **B**: 1 clase (PasswordHasher, cero riesgo)

---

## Nivel 2: Refactorización Guiada ⭐⭐

### Ejercicio 2.1. Refactorizar Sistema de Notificaciones

Dado este sistema de notificaciones que viola SRP, refactorízalo:

```java
public class NotificationManager {
    public void sendNotification(User user, String message, String type) {
        // Validar
        if (user.getEmail() == null || user.getEmail().isEmpty()) {
            throw new IllegalArgumentException("User must have email");
        }
        
        if (message == null || message.length() > 500) {
            throw new IllegalArgumentException("Invalid message");
        }
        
        // Decidir canal según tipo
        if (type.equals("URGENT")) {
            // SMS
            TwilioAPI twilioAPI = new TwilioAPI("key", "secret");
            twilioAPI.sendSMS(user.getPhone(), message);
            
            // Email
            SmtpClient smtp = new SmtpClient("smtp.gmail.com");
            smtp.send(user.getEmail(), "URGENT: " + message, message);
            
            // Push notification
            if (user.hasApp()) {
                FirebaseAPI firebase = new FirebaseAPI();
                firebase.sendPush(user.getDeviceToken(), message);
            }
        } else if (type.equals("NORMAL")) {
            // Solo email
            SmtpClient smtp = new SmtpClient("smtp.gmail.com");
            smtp.send(user.getEmail(), message, message);
        } else if (type.equals("LOW")) {
            // Solo in-app
            Connection conn = DriverManager.getConnection("jdbc:mysql://localhost/app");
            String sql = "INSERT INTO notifications (user_id, message, read) VALUES (?, ?, false)";
            PreparedStatement stmt = conn.prepareStatement(sql);
            stmt.setLong(1, user.getId());
            stmt.setString(2, message);
            stmt.executeUpdate();
        }
        
        // Guardar log
        Connection conn = DriverManager.getConnection("jdbc:mysql://localhost/app");
        String sql = "INSERT INTO notification_log (user_id, message, type, sent_at) VALUES (?, ?, ?, ?)";
        PreparedStatement stmt = conn.prepareStatement(sql);
        stmt.setLong(1, user.getId());
        stmt.setString(2, message);
        stmt.setString(3, type);
        stmt.setTimestamp(4, new Timestamp(System.currentTimeMillis()));
        stmt.executeUpdate();
    }
}
```

**Tareas:**
1. Identificar responsabilidades (mínimo 6)
2. Diseñar arquitectura con SRP
3. Implementar 4 clases completas
4. Escribir tests unitarios para al menos 2 clases

---

**SOLUCIÓN MODELO:**

**1. Responsabilidades identificadas:**
- Validación de usuario y mensaje
- Envío de SMS (Twilio)
- Envío de email (SMTP)
- Envío de push notifications (Firebase)
- Almacenamiento de notificaciones in-app
- Registro de auditoría
- Decisión de canal según prioridad

**2. Arquitectura con SRP:**

```java
// ✅ Validador
public class NotificationValidator {
    public ValidationResult validate(User user, String message) {
        List<String> errors = new ArrayList<>();
        
        if (user.getEmail() == null || user.getEmail().isEmpty()) {
            errors.add("User must have email");
        }
        
        if (message == null || message.trim().isEmpty()) {
            errors.add("Message cannot be empty");
        }
        
        if (message != null && message.length() > 500) {
            errors.add("Message exceeds 500 characters");
        }
        
        return errors.isEmpty() 
            ? ValidationResult.success() 
            : ValidationResult.failure(errors);
    }
}

// ✅ Interfaces de canales
public interface NotificationChannel {
    void send(User user, String message);
    boolean supports(User user);
}

public class EmailNotificationChannel implements NotificationChannel {
    private final EmailService emailService;
    
    public EmailNotificationChannel(EmailService emailService) {
        this.emailService = emailService;
    }
    
    @Override
    public void send(User user, String message) {
        emailService.send(user.getEmail(), "Notification", message);
    }
    
    @Override
    public boolean supports(User user) {
        return user.getEmail() != null && !user.getEmail().isEmpty();
    }
}

public class SMSNotificationChannel implements NotificationChannel {
    private final SMSService smsService;
    
    public SMSNotificationChannel(SMSService smsService) {
        this.smsService = smsService;
    }
    
    @Override
    public void send(User user, String message) {
        if (message.length() > 160) {
            message = message.substring(0, 157) + "...";
        }
        smsService.send(user.getPhone(), message);
    }
    
    @Override
    public boolean supports(User user) {
        return user.getPhone() != null && !user.getPhone().isEmpty();
    }
}

public class PushNotificationChannel implements NotificationChannel {
    private final PushNotificationService pushService;
    
    public PushNotificationChannel(PushNotificationService pushService) {
        this.pushService = pushService;
    }
    
    @Override
    public void send(User user, String message) {
        pushService.sendPush(user.getDeviceToken(), message);
    }
    
    @Override
    public boolean supports(User user) {
        return user.hasApp() && user.getDeviceToken() != null;
    }
}

public class InAppNotificationChannel implements NotificationChannel {
    private final NotificationRepository repository;
    
    public InAppNotificationChannel(NotificationRepository repository) {
        this.repository = repository;
    }
    
    @Override
    public void send(User user, String message) {
        Notification notification = new Notification(user.getId(), message, false);
        repository.save(notification);
    }
    
    @Override
    public boolean supports(User user) {
        return true; // Siempre soportado
    }
}

// ✅ Estrategia de canales según prioridad
public interface NotificationStrategy {
    List<NotificationChannel> selectChannels(NotificationPriority priority);
}

public class DefaultNotificationStrategy implements NotificationStrategy {
    private final EmailNotificationChannel emailChannel;
    private final SMSNotificationChannel smsChannel;
    private final PushNotificationChannel pushChannel;
    private final InAppNotificationChannel inAppChannel;
    
    public DefaultNotificationStrategy(EmailNotificationChannel emailChannel,
                                       SMSNotificationChannel smsChannel,
                                       PushNotificationChannel pushChannel,
                                       InAppNotificationChannel inAppChannel) {
        this.emailChannel = emailChannel;
        this.smsChannel = smsChannel;
        this.pushChannel = pushChannel;
        this.inAppChannel = inAppChannel;
    }
    
    @Override
    public List<NotificationChannel> selectChannels(NotificationPriority priority) {
        switch (priority) {
            case URGENT:
                return Arrays.asList(smsChannel, emailChannel, pushChannel);
            case NORMAL:
                return Arrays.asList(emailChannel, inAppChannel);
            case LOW:
                return Arrays.asList(inAppChannel);
            default:
                return Arrays.asList(inAppChannel);
        }
    }
}

// ✅ Servicio de auditoría
public class NotificationAuditService {
    private final NotificationLogRepository logRepository;
    
    public NotificationAuditService(NotificationLogRepository logRepository) {
        this.logRepository = logRepository;
    }
    
    public void logNotification(User user, String message, 
                                NotificationPriority priority, 
                                List<String> channelsUsed) {
        NotificationLog log = new NotificationLog(
            user.getId(),
            message,
            priority,
            channelsUsed,
            LocalDateTime.now()
        );
        
        logRepository.save(log);
    }
}

// ✅ Servicio principal (orquestador)
public class NotificationService {
    private final NotificationValidator validator;
    private final NotificationStrategy strategy;
    private final NotificationAuditService auditService;
    
    public NotificationService(NotificationValidator validator,
                               NotificationStrategy strategy,
                               NotificationAuditService auditService) {
        this.validator = validator;
        this.strategy = strategy;
        this.auditService = auditService;
    }
    
    public NotificationResult send(User user, String message, NotificationPriority priority) {
        // 1. Validar
        ValidationResult validation = validator.validate(user, message);
        if (!validation.isValid()) {
            return NotificationResult.failure(validation.getErrors());
        }
        
        // 2. Seleccionar canales
        List<NotificationChannel> channels = strategy.selectChannels(priority);
        
        // 3. Enviar por cada canal soportado
        List<String> usedChannels = new ArrayList<>();
        for (NotificationChannel channel : channels) {
            if (channel.supports(user)) {
                try {
                    channel.send(user, message);
                    usedChannels.add(channel.getClass().getSimpleName());
                } catch (Exception e) {
                    // Log error pero continuar con otros canales
                }
            }
        }
        
        // 4. Auditar
        auditService.logNotification(user, message, priority, usedChannels);
        
        return NotificationResult.success(usedChannels);
    }
}
```

**3. Tests Unitarios:**

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class NotificationValidatorTest {
    private NotificationValidator validator;
    
    @BeforeEach
    void setUp() {
        validator = new NotificationValidator();
    }
    
    @Test
    void shouldAcceptValidNotification() {
        User user = new User("user@example.com");
        String message = "Valid message";
        
        ValidationResult result = validator.validate(user, message);
        
        assertTrue(result.isValid());
    }
    
    @Test
    void shouldRejectUserWithoutEmail() {
        User user = new User(null);
        String message = "Valid message";
        
        ValidationResult result = validator.validate(user, message);
        
        assertFalse(result.isValid());
        assertTrue(result.getErrors().contains("User must have email"));
    }
    
    @Test
    void shouldRejectTooLongMessage() {
        User user = new User("user@example.com");
        String message = "a".repeat(501);
        
        ValidationResult result = validator.validate(user, message);
        
        assertFalse(result.isValid());
        assertTrue(result.getErrors().contains("Message exceeds 500 characters"));
    }
}

class EmailNotificationChannelTest {
    private EmailNotificationChannel channel;
    private EmailService mockEmailService;
    
    @BeforeEach
    void setUp() {
        mockEmailService = mock(EmailService.class);
        channel = new EmailNotificationChannel(mockEmailService);
    }
    
    @Test
    void shouldSendEmailWhenUserHasEmail() {
        User user = new User("user@example.com");
        String message = "Test message";
        
        channel.send(user, message);
        
        verify(mockEmailService).send("user@example.com", "Notification", message);
    }
    
    @Test
    void shouldSupportUserWithEmail() {
        User user = new User("user@example.com");
        
        assertTrue(channel.supports(user));
    }
    
    @Test
    void shouldNotSupportUserWithoutEmail() {
        User user = new User(null);
        
        assertFalse(channel.supports(user));
    }
}

class NotificationServiceTest {
    private NotificationService service;
    private NotificationValidator mockValidator;
    private NotificationStrategy mockStrategy;
    private NotificationAuditService mockAuditService;
    private NotificationChannel mockChannel;
    
    @BeforeEach
    void setUp() {
        mockValidator = mock(NotificationValidator.class);
        mockStrategy = mock(NotificationStrategy.class);
        mockAuditService = mock(NotificationAuditService.class);
        mockChannel = mock(NotificationChannel.class);
        
        service = new NotificationService(mockValidator, mockStrategy, mockAuditService);
    }
    
    @Test
    void shouldSendNotificationSuccessfully() {
        User user = new User("user@example.com");
        String message = "Test";
        
        when(mockValidator.validate(user, message)).thenReturn(ValidationResult.success());
        when(mockStrategy.selectChannels(NotificationPriority.NORMAL))
            .thenReturn(Arrays.asList(mockChannel));
        when(mockChannel.supports(user)).thenReturn(true);
        
        NotificationResult result = service.send(user, message, NotificationPriority.NORMAL);
        
        assertTrue(result.isSuccess());
        verify(mockChannel).send(user, message);
        verify(mockAuditService).logNotification(eq(user), eq(message), 
                                                  eq(NotificationPriority.NORMAL), anyList());
    }
}
```

---

## Nivel 3: Diseño de Arquitectura ⭐⭐⭐

### Ejercicio 3.1. Sistema de Procesamiento de Pagos

Diseña un sistema de procesamiento de pagos que soporte múltiples gateways (Stripe, PayPal, Square) y cumpla con SRP.

**Requisitos:**
1. Validación de datos de pago
2. Procesamiento con diferentes gateways
3. Manejo de webhooks de cada gateway
4. Almacenamiento de transacciones
5. Reconciliación contable
6. Generación de reportes
7. Manejo de reembolsos
8. Detección de fraude

**Entregables:**
- Diagrama de clases identificando responsabilidades
- Implementación de 5 clases completas
- Tests con >75% cobertura

---

## Nivel 4: Análisis de Open Source ⭐⭐⭐⭐

### Ejercicio 4.1. Análisis de Spring Security

Investiga la arquitectura de Spring Security y documenta:

1. ¿Cómo aplica SRP en la autenticación?
2. Identifica al menos 10 clases con responsabilidades únicas
3. ¿Qué patrones de diseño usa para mantener SRP?
4. Presenta diagrama de interacción entre componentes

**Componentes a analizar:**
- `AuthenticationManager`
- `UserDetailsService`
- `PasswordEncoder`
- `SecurityFilterChain`
- `AuthenticationProvider`

---

## Rúbrica de Evaluación

### Nivel 1 (20 puntos)
- [ ] Identifica correctamente responsabilidades (10 pts)
- [ ] Justifica actores que piden cambios (5 pts)
- [ ] Compara arquitecturas con métricas (5 pts)

### Nivel 2 (30 puntos)
- [ ] Extrae al menos 6 clases con responsabilidades únicas (12 pts)
- [ ] Implementa 4 clases completas funcionales (10 pts)
- [ ] Tests cubren casos principales (>70%) (8 pts)

### Nivel 3 (25 puntos)
- [ ] Diagrama muestra separación clara de responsabilidades (10 pts)
- [ ] Implementa 5 clases con SRP (10 pts)
- [ ] Tests con >75% cobertura (5 pts)

### Nivel 4 (25 puntos)
- [ ] Análisis profundo de Spring Security (10 pts)
- [ ] Identifica >10 clases y sus responsabilidades (8 pts)
- [ ] Diagrama de interacción correcto (7 pts)

**Total: 100 puntos**
