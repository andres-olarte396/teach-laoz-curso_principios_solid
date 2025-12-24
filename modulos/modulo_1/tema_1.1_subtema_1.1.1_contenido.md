# Subtema 1.1.1. Definición y Fundamentos del SRP

## 1. Contexto y Motivación

El **Single Responsibility Principle (SRP)** es el primero y posiblemente el más importante de los principios SOLID. Robert C. Martin lo define así:

> "Una clase debe tener una, y solo una, razón para cambiar."

Este principio es fundamental porque:
- **Reduce complejidad**: Clases más pequeñas y enfocadas son más fáciles de entender
- **Facilita mantenimiento**: Cambios en un requisito afectan menos clases
- **Mejora testabilidad**: Clases con una responsabilidad son más fáciles de testear
- **Aumenta reusabilidad**: Clases enfocadas se pueden reutilizar en más contextos

## 2. Fundamentos Teóricos

### 2.1 ¿Qué es una "Responsabilidad"?

Una **responsabilidad** es una razón para cambiar. En términos más concretos:
- Un actor o stakeholder que puede requerir cambios
- Un conjunto cohesivo de funcionalidades
- Una fuente de cambio en el sistema

**Ejemplo:**
```java
// ❌ VIOLA SRP: Esta clase tiene 3 responsabilidades (3 razones para cambiar)
public class Employee {
    // Responsabilidad 1. Lógica de negocio (CFO puede pedir cambios)
    public double calculatePay() {
        // Cálculo de salario
    }
    
    // Responsabilidad 2: Persistencia (DBA puede pedir cambios)
    public void save() {
        // Guardar en base de datos
    }
    
    // Responsabilidad 3: Reportes (COO puede pedir cambios)
    public String generateReport() {
        // Generar reporte de horas
    }
}
```

Cada método responde a un actor diferente:
- `calculatePay()` → CFO (políticas de compensación)
- `save()` → DBA (estructura de base de datos)
- `generateReport()` → COO (formato de reportes)

### 2.2 Cohesión vs Acoplamiento

**Cohesión:** Grado en que los elementos de un módulo pertenecen juntos.
- ✅ **Alta cohesión** (BUENO): Elementos relacionados agrupados
- ❌ **Baja cohesión** (MALO): Elementos no relacionados mezclados

**Acoplamiento:** Grado de interdependencia entre módulos.
- ✅ **Bajo acoplamiento** (BUENO): Módulos independientes
- ❌ **Alto acoplamiento** (MALO): Módulos muy dependientes

**SRP busca:** Alta cohesión + Bajo acoplamiento

### 2.3 Razón para Cambiar

Una clase tiene **múltiples razones para cambiar** cuando:
- Combina lógica de negocio con persistencia
- Mezcla presentación con procesamiento
- Agrupa funcionalidades de diferentes dominios
- Tiene métodos que sirven a diferentes actores

**Señales de violación de SRP:**
- Clases con muchos métodos (>10-15)
- Nombres con "Manager", "Handler", "Utility", "Helper"
- Dificultad para describir la clase en una frase
- Cambios en un requisito afectan múltiples métodos no relacionados

## 3. Ejemplos Prácticos

### 3.1 Violación Clásica de SRP

```java
// ❌ VIOLA SRP: Clase con múltiples responsabilidades
public class User {
    private String username;
    private String password;
    private String email;
    
    // Responsabilidad 1. Validación de datos
    public boolean isValidEmail() {
        return email.matches("^[A-Za-z0-9+_.-]+@(.+)$");
    }
    
    public boolean isPasswordStrong() {
        return password.length() >= 8 && 
               password.matches(".*[A-Z].*") &&
               password.matches(".*[0-9].*");
    }
    
    // Responsabilidad 2: Persistencia
    public void saveToDatabase() {
        Connection conn = DriverManager.getConnection("jdbc:mysql://...");
        PreparedStatement stmt = conn.prepareStatement(
            "INSERT INTO users (username, password, email) VALUES (?, ?, ?)"
        );
        stmt.setString(1, username);
        stmt.setString(2, password);
        stmt.setString(3, email);
        stmt.executeUpdate();
    }
    
    // Responsabilidad 3: Autenticación
    public boolean authenticate(String inputPassword) {
        return BCrypt.checkpw(inputPassword, this.password);
    }
    
    // Responsabilidad 4: Notificaciones
    public void sendWelcomeEmail() {
        EmailService service = new EmailService();
        service.send(email, "Welcome!", "Thanks for signing up!");
    }
    
    // Responsabilidad 5: Formateo/Presentación
    public String toJSON() {
        return String.format("{\"username\":\"%s\",\"email\":\"%s\"}", 
                             username, email);
    }
}
```

**Problemas:**
1. Cambios en validación afectan la clase User
2. Cambios en base de datos afectan la clase User
3. Cambios en autenticación afectan la clase User
4. Cambios en notificaciones afectan la clase User
5. Cambios en formato de salida afectan la clase User

### 3.2 Solución Aplicando SRP

```java
// ✅ CUMPLE SRP: Clase con única responsabilidad (modelo de dominio)
public class User {
    private final String username;
    private final String password;
    private final String email;
    
    public User(String username, String password, String email) {
        this.username = username;
        this.password = password;
        this.email = email;
    }
    
    // Solo getters - representación de datos
    public String getUsername() { return username; }
    public String getPassword() { return password; }
    public String getEmail() { return email; }
}

// ✅ Responsabilidad: Validación
public class UserValidator {
    public ValidationResult validate(User user) {
        ValidationResult result = new ValidationResult();
        
        if (!isValidEmail(user.getEmail())) {
            result.addError("Invalid email format");
        }
        
        if (!isPasswordStrong(user.getPassword())) {
            result.addError("Password must be at least 8 characters with uppercase and numbers");
        }
        
        return result;
    }
    
    private boolean isValidEmail(String email) {
        return email.matches("^[A-Za-z0-9+_.-]+@(.+)$");
    }
    
    private boolean isPasswordStrong(String password) {
        return password.length() >= 8 && 
               password.matches(".*[A-Z].*") &&
               password.matches(".*[0-9].*");
    }
}

// ✅ Responsabilidad: Persistencia
public class UserRepository {
    private final DataSource dataSource;
    
    public UserRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    public void save(User user) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(
                 "INSERT INTO users (username, password, email) VALUES (?, ?, ?)")) {
            
            stmt.setString(1, user.getUsername());
            stmt.setString(2, user.getPassword());
            stmt.setString(3, user.getEmail());
            stmt.executeUpdate();
            
        } catch (SQLException e) {
            throw new PersistenceException("Failed to save user", e);
        }
    }
    
    public User findByUsername(String username) {
        // Implementación de búsqueda
    }
}

// ✅ Responsabilidad: Autenticación
public class AuthenticationService {
    public boolean authenticate(User user, String inputPassword) {
        return BCrypt.checkpw(inputPassword, user.getPassword());
    }
    
    public String hashPassword(String plainPassword) {
        return BCrypt.hashpw(plainPassword, BCrypt.gensalt());
    }
}

// ✅ Responsabilidad: Notificaciones
public class UserNotificationService {
    private final EmailService emailService;
    
    public UserNotificationService(EmailService emailService) {
        this.emailService = emailService;
    }
    
    public void sendWelcomeEmail(User user) {
        emailService.send(
            user.getEmail(),
            "Welcome!",
            "Thanks for signing up, " + user.getUsername() + "!"
        );
    }
}

// ✅ Responsabilidad: Serialización
public class UserSerializer {
    public String toJSON(User user) {
        return String.format("{\"username\":\"%s\",\"email\":\"%s\"}", 
                             user.getUsername(), user.getEmail());
    }
    
    public String toXML(User user) {
        return String.format("<user><username>%s</username><email>%s</email></user>",
                             user.getUsername(), user.getEmail());
    }
}

// Orquestación (Use Case / Application Service)
public class UserRegistrationService {
    private final UserValidator validator;
    private final UserRepository repository;
    private final AuthenticationService authService;
    private final UserNotificationService notificationService;
    
    public UserRegistrationService(
            UserValidator validator,
            UserRepository repository,
            AuthenticationService authService,
            UserNotificationService notificationService) {
        this.validator = validator;
        this.repository = repository;
        this.authService = authService;
        this.notificationService = notificationService;
    }
    
    public void registerUser(String username, String password, String email) {
        // 1. Crear usuario
        String hashedPassword = authService.hashPassword(password);
        User user = new User(username, hashedPassword, email);
        
        // 2. Validar
        ValidationResult validationResult = validator.validate(user);
        if (!validationResult.isValid()) {
            throw new ValidationException(validationResult.getErrors());
        }
        
        // 3. Guardar
        repository.save(user);
        
        // 4. Notificar
        notificationService.sendWelcomeEmail(user);
    }
}
```

### 3.3 Comparación Lado a Lado

| Aspecto | Sin SRP (1 clase) | Con SRP (6 clases) |
|---------|-------------------|-------------------|
| **Líneas por clase** | ~80 | ~20-30 cada una |
| **Testabilidad** | Difícil (múltiples dependencias) | Fácil (una responsabilidad) |
| **Reusabilidad** | Baja (todo o nada) | Alta (usar solo lo necesario) |
| **Mantenibilidad** | Cambios arriesgados | Cambios localizados |
| **Legibilidad** | Confusa (hace demasiado) | Clara (nombre describe función) |

## 4. Código Ejecutable con Tests

```java
// Tests para UserValidator
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import static org.junit.jupiter.api.Assertions.*;

class UserValidatorTest {
    
    private UserValidator validator;
    
    @BeforeEach
    void setUp() {
        validator = new UserValidator();
    }
    
    @Test
    void validUserShouldPassValidation() {
        User user = new User("john_doe", "SecurePass123", "john@example.com");
        ValidationResult result = validator.validate(user);
        assertTrue(result.isValid());
    }
    
    @Test
    void invalidEmailShouldFailValidation() {
        User user = new User("john_doe", "SecurePass123", "invalid-email");
        ValidationResult result = validator.validate(user);
        assertFalse(result.isValid());
        assertTrue(result.getErrors().stream()
            .anyMatch(error -> error.contains("email")));
    }
    
    @Test
    void weakPasswordShouldFailValidation() {
        User user = new User("john_doe", "weak", "john@example.com");
        ValidationResult result = validator.validate(user);
        assertFalse(result.isValid());
        assertTrue(result.getErrors().stream()
            .anyMatch(error -> error.contains("Password")));
    }
}

// Tests para UserRepository (con mock de database)
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import static org.mockito.Mockito.*;

class UserRepositoryTest {
    
    @Test
    void saveShouldInsertUserIntoDatabase() throws SQLException {
        // Arrange
        DataSource mockDataSource = mock(DataSource.class);
        Connection mockConnection = mock(Connection.class);
        PreparedStatement mockStatement = mock(PreparedStatement.class);
        
        when(mockDataSource.getConnection()).thenReturn(mockConnection);
        when(mockConnection.prepareStatement(anyString())).thenReturn(mockStatement);
        
        UserRepository repository = new UserRepository(mockDataSource);
        User user = new User("john_doe", "hashed_password", "john@example.com");
        
        // Act
        repository.save(user);
        
        // Assert
        verify(mockStatement).setString(1, "john_doe");
        verify(mockStatement).setString(2, "hashed_password");
        verify(mockStatement).setString(3, "john@example.com");
        verify(mockStatement).executeUpdate();
    }
}

// Tests para AuthenticationService
class AuthenticationServiceTest {
    
    @Test
    void authenticateShouldReturnTrueForCorrectPassword() {
        AuthenticationService authService = new AuthenticationService();
        String plainPassword = "MyPassword123";
        String hashedPassword = authService.hashPassword(plainPassword);
        
        User user = new User("john", hashedPassword, "john@example.com");
        
        assertTrue(authService.authenticate(user, plainPassword));
    }
    
    @Test
    void authenticateShouldReturnFalseForIncorrectPassword() {
        AuthenticationService authService = new AuthenticationService();
        String hashedPassword = authService.hashPassword("CorrectPassword");
        
        User user = new User("john", hashedPassword, "john@example.com");
        
        assertFalse(authService.authenticate(user, "WrongPassword"));
    }
}
```

## 5. Casos de Uso Reales

### 5.1 Sistema de E-commerce

```java
// ❌ VIOLA SRP
class Product {
    void calculatePrice() { }
    void saveToDatabase() { }
    void generateHTML() { }
    void sendStockAlert() { }
}

// ✅ CUMPLE SRP
class Product {
    // Solo datos del producto
}

class PriceCalculator {
    double calculate(Product product, DiscountRules rules) { }
}

class ProductRepository {
    void save(Product product) { }
}

class ProductPresenter {
    String toHTML(Product product) { }
}

class StockAlertService {
    void checkAndNotify(Product product) { }
}
```

### 5.2 Sistema de Reservas

```java
// ✅ Cada clase una responsabilidad
class Booking {
    // Modelo de dominio: datos de la reserva
}

class BookingValidator {
    // Validar disponibilidad, fechas, capacidad
}

class BookingRepository {
    // Persistencia de reservas
}

class BookingPriceCalculator {
    // Calcular precio según temporada, duración, etc.
}

class BookingConfirmationService {
    // Enviar confirmaciones por email/SMS
}

class BookingCalendarSync {
    // Sincronizar con calendarios externos (Google Calendar, etc.)
}
```

## 6. Granularidad: ¿Cuándo es "demasiado"?

### 6.1 El Peligro de la Sobre-Fragmentación

```java
// ❌ DEMASIADO GRANULAR
class UserFirstNameValidator { }
class UserLastNameValidator { }
class UserEmailValidator { }
class UserPhoneValidator { }
class UserAddressValidator { }
// ... 15 clases más para cada campo

// ✅ GRANULARIDAD APROPIADA
class UserValidator {
    private EmailValidator emailValidator;
    private PhoneValidator phoneValidator;
    // Agrupación lógica de validaciones relacionadas
}
```

### 6.2 Guías para Encontrar el Balance

**Pregunta clave:** ¿Estos métodos/propiedades cambiarían por la misma razón?

**Señales de granularidad correcta:**
- ✅ Clase se puede describir en una frase sin "y"
- ✅ Cambios en requisitos afectan una clase
- ✅ Tests de la clase son cohesivos
- ✅ Nombre de clase es sustantivo claro (no genérico)

**Señales de sobre-fragmentación:**
- ❌ Clases con 1-2 métodos triviales
- ❌ Muchas clases siempre usadas juntas
- ❌ Navegación compleja entre clases relacionadas

## 7. Mejores Prácticas

### 7.1 Identificar Responsabilidades

1. **Método de los actores:** ¿Quién pediría cambios?
   - CFO → Lógica financiera
   - DBA → Persistencia
   - UX → Presentación

2. **Método de descripción:** Si usas "y" al describir la clase, probablemente viola SRP
   - ❌ "Esta clase valida usuarios **y** los guarda **y** envía emails"
   - ✅ "Esta clase valida usuarios"

3. **Análisis de cambios:** ¿Qué cambios de requisitos afectarían esta clase?

### 7.2 Nombres Revelan Responsabilidad

```java
// ❌ Nombres vagos (señal de múltiples responsabilidades)
class UserManager { }
class DataHandler { }
class SystemUtility { }

// ✅ Nombres específicos (indican responsabilidad única)
class UserValidator { }
class UserRepository { }
class UserRegistrationService { }
class EmailNotificationSender { }
```

### 7.3 Composición sobre Clases Grandes

```java
// En lugar de una clase grande, componer servicios
public class UserRegistrationService {
    private final UserValidator validator;
    private final UserRepository repository;
    private final EmailService emailService;
    
    // Constructor con inyección de dependencias
    public UserRegistrationService(
            UserValidator validator,
            UserRepository repository,
            EmailService emailService) {
        this.validator = validator;
        this.repository = repository;
        this.emailService = emailService;
    }
    
    public void register(UserDTO dto) {
        // Orquestar llamadas a servicios especializados
        validator.validate(dto);
        User user = repository.save(dto);
        emailService.sendWelcome(user);
    }
}
```

## 8. Errores Comunes

### 8.1 God Class (Clase Dios)

```java
// ❌ Clase que hace TODO
class Application {
    void handleHTTPRequest() { }
    void queryDatabase() { }
    void sendEmail() { }
    void processPayment() { }
    void generatePDF() { }
    void validateInput() { }
    // ... 50 métodos más
}
```

### 8.2 Confundir Cohesión con SRP

```java
// ❌ Métodos cohesivos pero múltiples responsabilidades
class ReportGenerator {
    // Cohesivos (todos sobre reportes) pero diferentes responsabilidades
    void fetchDataFromDatabase() { }  // Persistencia
    void calculateStatistics() { }     // Lógica de negocio
    void formatAsHTML() { }            // Presentación
    void sendByEmail() { }             // Notificaciones
}
```

### 8.3 Anemic Domain Model

```java
// ❌ Modelo anémico (solo getters/setters, toda lógica en "Services")
class Order {
    private List<Item> items;
    // Solo getters/setters
}

class OrderService {
    double calculateTotal(Order order) { }
    boolean canCheckout(Order order) { }
    // Toda la lógica aquí
}

// ✅ Domain model con lógica de dominio
class Order {
    private List<Item> items;
    
    public double calculateTotal() {
        return items.stream()
            .mapToDouble(Item::getPrice)
            .sum();
    }
    
    public boolean canCheckout() {
        return !items.isEmpty() && calculateTotal() > 0;
    }
}
```

## 9. Resumen Ejecutivo

**Single Responsibility Principle:**
- Una clase debe tener **una, y solo una, razón para cambiar**
- Razón para cambiar = Actor/Stakeholder que puede requerir modificaciones
- Promueve **alta cohesión** y **bajo acoplamiento**

**Beneficios:**
- ✅ Código más fácil de entender y mantener
- ✅ Tests más simples y enfocados
- ✅ Mayor reusabilidad de componentes
- ✅ Cambios localizados (menos efectos secundarios)

**Cómo aplicarlo:**
1. Identificar responsabilidades por actor
2. Separar en clases especializadas
3. Usar nombres claros que indiquen la responsabilidad
4. Componer servicios de alto nivel con servicios especializados

## 10. Puntos Clave

✅ **Una clase = Una responsabilidad = Un motivo para cambiar**  
✅ Separar **lógica de negocio**, **persistencia** y **presentación**  
✅ Nombres claros revelan responsabilidad (`UserValidator` > `UserManager`)  
✅ Preferir **múltiples clases pequeñas** sobre una clase grande  
✅ SRP facilita **testing**, **mantenimiento** y **reusabilidad**  
❌ Evitar **God Classes** que hacen demasiado  
❌ Evitar nombres vagos como "Manager", "Handler", "Utility"  
❌ No sobre-fragmentar (encontrar balance)
