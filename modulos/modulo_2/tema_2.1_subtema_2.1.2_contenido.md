# Identificación Sistemática de Responsabilidades

## Introducción

Identificar responsabilidades en código existente es una habilidad fundamental para aplicar el Single Responsibility Principle efectivamente. No siempre es obvio cuándo una clase tiene múltiples responsabilidades, especialmente en código legacy que ha crecido orgánicamente.

En este módulo aprenderemos **técnicas sistemáticas** para detectar violaciones de SRP antes de que se conviertan en problemas de mantenibilidad.

## Técnica 1: Análisis de Cohesión (LCOM)

### ¿Qué es LCOM?

**LCOM (Lack of Cohesion of Methods)** es una métrica que mide cuánto comparten los métodos de una clase sus variables de instancia. Un LCOM alto indica baja cohesión y posibles múltiples responsabilidades.

### Cálculo de LCOM (versión simplificada)

```
LCOM = (Pares de métodos que NO comparten variables) - (Pares que SÍ comparten)

Si LCOM > 0: Posible violación de SRP
Si LCOM ≤ 0: Cohesión aceptable
```

### Ejemplo Práctico

```java
public class UserService {
    // Variables de instancia
    private String username;
    private String password;
    private Connection dbConnection;
    private Logger logger;
    
    // Método 1: Usa username, password
    public boolean authenticate() {
        return username.equals("admin") && password.equals("secret");
    }
    
    // Método 2: Usa username, password
    public void changePassword(String newPassword) {
        this.password = newPassword;
    }
    
    // Método 3: Usa dbConnection
    public void saveToDatabase() {
        dbConnection.execute("INSERT INTO users...");
    }
    
    // Método 4: Usa logger
    public void logActivity(String action) {
        logger.log("User " + username + " performed: " + action);
    }
}
```

**Análisis de cohesión**:

| Método | Variables Usadas |
|--------|------------------|
| `authenticate()` | username, password |
| `changePassword()` | password |
| `saveToDatabase()` | dbConnection |
| `logActivity()` | username, logger |

**Pares que comparten variables**:
- `authenticate` ↔ `changePassword`: Comparten `password` ✅
- `authenticate` ↔ `logActivity`: Comparten `username` ✅
- `changePassword` ↔ `logActivity`: Comparten `username` (indirectamente) ✅

**Pares que NO comparten**:
- `authenticate` ↔ `saveToDatabase` ❌
- `changePassword` ↔ `saveToDatabase` ❌
- `logActivity` ↔ `saveToDatabase` ❌
- Y otros pares...

**Conclusión**: Alta cantidad de pares no compartidos → **LCOM alto** → **Múltiples responsabilidades**

### Interpretación Visual

```
Grupos de cohesión detectados:

Grupo 1: Autenticación
├── username
├── password
├── authenticate()
└── changePassword()

Grupo 2: Persistencia
├── dbConnection
└── saveToDatabase()

Grupo 3: Logging
├── logger
└── logActivity()
```

**Refactorización sugerida**:

```java
// Responsabilidad 1: Autenticación
public class Authenticator {
    private String username;
    private String password;
    
    public boolean authenticate() {
        return username.equals("admin") && password.equals("secret");
    }
    
    public void changePassword(String newPassword) {
        this.password = newPassword;
    }
}

// Responsabilidad 2: Persistencia
public class UserRepository {
    private Connection dbConnection;
    
    public void save(User user) {
        dbConnection.execute("INSERT INTO users...");
    }
}

// Responsabilidad 3: Auditoría
public class ActivityLogger {
    private Logger logger;
    
    public void logActivity(String username, String action) {
        logger.log("User " + username + " performed: " + action);
    }
}
```

## Técnica 2: Análisis de Dependencias

### Regla: Una clase no debería depender de múltiples subsistemas desacoplados

Si una clase importa/usa clases de múltiples dominios no relacionados, probablemente tiene múltiples responsabilidades.

### Ejemplo: Análisis de Imports

```java
// ❌ SOSPECHOSO: Imports de múltiples dominios
import java.sql.Connection;              // Persistencia
import javax.mail.EmailService;          // Comunicación
import com.stripe.PaymentGateway;        // Pagos
import org.apache.poi.ExcelGenerator;    // Reportes
import java.util.logging.Logger;         // Logging

public class OrderManager {
    private Connection db;
    private EmailService emailService;
    private PaymentGateway paymentGateway;
    private ExcelGenerator excelGenerator;
    private Logger logger;
    
    // Múltiples responsabilidades evidentes por las dependencias
}
```

**Diagnóstico**:
- **5 dominios diferentes** → **Violación clara de SRP**
- Cada dominio representa una razón potencial para cambiar

### Matriz de Dependencias

Crea una matriz para visualizar acoplamiento:

|  | DB | Email | Payment | Excel | Logger |
|--|-----|-------|---------|-------|--------|
| **processOrder()** | ✓ | ✓ | ✓ | ✗ | ✓ |
| **generateReport()** | ✓ | ✗ | ✗ | ✓ | ✓ |
| **sendInvoice()** | ✓ | ✓ | ✗ | ✗ | ✓ |

**Interpretación**:
- Cada fila es un método
- Cada columna es una dependencia externa
- ✓ = método usa esa dependencia
- Alta dispersión de ✓ → Baja cohesión

## Técnica 3: Análisis de Actores (Actor-Based Analysis)

### Metodología

1. **Lista todos los métodos públicos** de la clase
2. **Identifica el actor** que solicitaría cambios en cada método
3. **Agrupa métodos** por actor
4. **Cuenta actores únicos** → Número de responsabilidades

### Ejemplo Paso a Paso

```java
public class Employee {
    private String name;
    private double hourlyRate;
    private String department;
    
    // Método A
    public double calculatePay() {
        return hourlyRate * 160;
    }
    
    // Método B
    public String generatePaySlip() {
        return "Pay: $" + calculatePay();
    }
    
    // Método C
    public void save() {
        Database.execute("INSERT INTO employees...");
    }
    
    // Método D
    public String toJson() {
        return "{\"name\":\"" + name + "\"}";
    }
    
    // Método E
    public void sendWelcomeEmail() {
        EmailService.send(name + "@company.com", "Welcome!");
    }
    
    // Método F
    public int calculateVacationDays() {
        return (int)(hourlyRate / 10); // Ejemplo simplificado
    }
}
```

**Paso 1: Identificar actores**

| Método | Actor | Razón del Cambio Posible |
|--------|-------|--------------------------|
| `calculatePay()` | CFO (Finanzas) | Cambio en política salarial |
| `generatePaySlip()` | CFO (Finanzas) | Cambio en formato de nómina |
| `save()` | CTO (IT) | Migración de base de datos |
| `toJson()` | CTO (IT) | Cambio en API REST |
| `sendWelcomeEmail()` | CMO (Marketing) | Cambio en campaña de onboarding |
| `calculateVacationDays()` | CHRO (RRHH) | Cambio en política de vacaciones |

**Paso 2: Agrupar por actor**

```
Grupo CFO (Finanzas):
- calculatePay()
- generatePaySlip()

Grupo CTO (IT):
- save()
- toJson()

Grupo CMO (Marketing):
- sendWelcomeEmail()

Grupo CHRO (RRHH):
- calculateVacationDays()
```

**Paso 3: Contar actores**

**4 actores diferentes** → **4 responsabilidades** → **Violación grave de SRP**

### Refactorización Guiada por Actores

```java
// Actor: CFO (Finanzas)
public class PayrollCalculator {
    public double calculatePay(Employee employee) {
        return employee.getHourlyRate() * 160;
    }
    
    public String generatePaySlip(Employee employee, double pay) {
        return "Pay: $" + pay;
    }
}

// Actor: CTO (IT) - Persistencia
public class EmployeeRepository {
    public void save(Employee employee) {
        Database.execute("INSERT INTO employees...");
    }
}

// Actor: CTO (IT) - Serialización
public class EmployeeSerializer {
    public String toJson(Employee employee) {
        return "{\"name\":\"" + employee.getName() + "\"}";
    }
}

// Actor: CMO (Marketing)
public class EmployeeOnboardingService {
    public void sendWelcomeEmail(Employee employee) {
        String email = employee.getName() + "@company.com";
        EmailService.send(email, "Welcome!");
    }
}

// Actor: CHRO (RRHH)
public class VacationCalculator {
    public int calculateVacationDays(Employee employee) {
        return (int)(employee.getHourlyRate() / 10);
    }
}

// Entidad de dominio (sin lógica de negocio)
public class Employee {
    private String name;
    private double hourlyRate;
    private String department;
    
    // Solo getters/setters
}
```

## Técnica 4: Análisis de Frecuencia de Cambio

### Principio

> "Separa las cosas que cambian por razones diferentes en momentos diferentes"

### Metodología

1. Revisa el **historial de commits** (git log)
2. Identifica qué métodos/regiones cambian juntos
3. Identifica qué métodos cambian independientemente
4. Métodos que cambian independientemente → Diferentes responsabilidades

### Ejemplo con Git

```bash
# Analizar cambios en UserManager.java en últimos 6 meses
git log --since="6 months ago" --pretty=format:"%h %s" -- UserManager.java
```

**Resultado**:

```
a1b2c3d Fix email template for welcome message
d4e5f6g Update database connection string
h7i8j9k Change password hashing algorithm
k0l1m2n Fix email SMTP configuration
n3o4p5q Migrate to PostgreSQL from MySQL
```

**Análisis**:

| Commits | Cambios en... | Actor Responsable |
|---------|---------------|-------------------|
| a1b2c3d, k0l1m2n | Email | Marketing |
| d4e5f6g, n3o4p5q | Database | IT |
| h7i8j9k | Autenticación | Seguridad |

**Conclusión**: **3 actores** cambiando la misma clase → **Violación de SRP**

### Heatmap de Cambios

```
UserManager.java:
├── authenticate()         [##########] 10 cambios (Seguridad)
├── changePassword()       [#######] 7 cambios (Seguridad)
├── saveToDatabase()       [#############] 13 cambios (IT)
├── sendWelcomeEmail()     [####] 4 cambios (Marketing)
└── sendPasswordReset()    [######] 6 cambios (Marketing)
```

**Patrón identificado**:
- Métodos de autenticación cambian juntos
- Métodos de base de datos cambian juntos
- Métodos de email cambian juntos
- **Pero NO cambian entre grupos** → Separar en 3 clases

## Técnica 5: Análisis de Nombres (Linguistic Anti-Patterns)

### Regla: El nombre de la clase debe describir una sola cosa

Si el nombre contiene "And", "Manager", "Handler", "Util", "Helper" → **Sospecha de SRP violado**

### Nombres Problemáticos

```java
// ❌ "And" en el nombre
public class UserAndOrderManager { }

// ❌ "Manager" genérico
public class DataManager { }

// ❌ "Handler" vago
public class RequestHandler { }

// ❌ "Util" sin propósito claro
public class SystemUtil { }

// ❌ Múltiples verbos no relacionados
public class ValidateSaveAndNotify { }
```

### Test del Nombre

**Pregunta**: "¿Puedo describir la clase en una oración sin usar 'y', 'o', 'además'?"

```java
// ❌ MAL: Necesita "y" en la descripción
public class UserProcessor {
    // "Valida usuarios Y los guarda en la base de datos Y envía emails"
    // → 3 responsabilidades
}

// ✅ BIEN: Descripción clara y simple
public class UserValidator {
    // "Valida que los datos de usuario cumplan reglas de negocio"
    // → 1 responsabilidad
}
```

### Nombres que Sugieren SRP

```java
// ✅ Nombre específico y enfocado
public class PasswordHasher { }
public class InvoiceGenerator { }
public class EmailSender { }
public class ProductRepository { }
public class TaxCalculator { }
```

## Técnica 6: Análisis de Testing (Test Complexity)

### Indicadores en Tests

Un test complejo suele indicar una clase con múltiples responsabilidades:

```java
// ❌ Test complejo = Clase con múltiples responsabilidades
@Test
public void testOrderProcessing() {
    // Setup requiere MUCHAS dependencias
    Database mockDb = mock(Database.class);
    EmailService mockEmail = mock(EmailService.class);
    PaymentGateway mockPayment = mock(PaymentGateway.class);
    InventoryService mockInventory = mock(InventoryService.class);
    TaxCalculator mockTax = mock(TaxCalculator.class);
    ShippingCalculator mockShipping = mock(ShippingCalculator.class);
    Logger mockLogger = mock(Logger.class);
    
    // Configurar TODOS los mocks
    when(mockDb.findUser(anyString())).thenReturn(new User());
    when(mockPayment.charge(any(), anyDouble())).thenReturn(success());
    // ... 20 líneas más de setup
    
    OrderProcessor processor = new OrderProcessor(
        mockDb, mockEmail, mockPayment, mockInventory, 
        mockTax, mockShipping, mockLogger
    );
    
    // Test real
    processor.processOrder(order);
    
    // Verificaciones de TODOS los mocks
    verify(mockDb, times(3)).execute(any());
    verify(mockEmail).send(any(), any(), any());
    // ... 15 líneas más de verificaciones
}
```

**Problemas identificados**:

| Indicador | Valor | Interpretación |
|-----------|-------|----------------|
| Número de mocks | 7 | ❌ Demasiadas dependencias |
| Líneas de setup | 25+ | ❌ Setup complejo |
| Líneas de verificación | 15+ | ❌ Verificando múltiples comportamientos |
| Tiempo de ejecución | 500ms | ❌ Lento (muchas operaciones) |

**Conclusión**: Test complejo → Clase con múltiples responsabilidades

### Test Simplificado (después de SRP)

```java
// ✅ Test simple = Clase con responsabilidad única
@Test
public void testTaxCalculation() {
    TaxCalculator calculator = new TaxCalculator();
    
    double tax = calculator.calculate(100.0, "CA");
    
    assertEquals(8.25, tax); // Sin mocks, sin setup, rápido
}
```

**Métricas mejoradas**:

| Indicador | Valor | Interpretación |
|-----------|-------|----------------|
| Número de mocks | 0 | ✅ Sin dependencias externas |
| Líneas de setup | 1 | ✅ Setup trivial |
| Líneas de verificación | 1 | ✅ Una aserción clara |
| Tiempo de ejecución | <10ms | ✅ Instantáneo |

## Técnica 7: Matriz de Trazabilidad de Requisitos

### Metodología

Crea una matriz que mapea requisitos de negocio a clases:

|  | UserManager | PaymentService | EmailService | ReportGenerator |
|--|-------------|----------------|--------------|-----------------|
| **Login** | ✓ | ✗ | ✗ | ✗ |
| **Register** | ✓ | ✗ | ✓ | ✗ |
| **Process Payment** | ✓ | ✓ | ✓ | ✗ |
| **Generate Invoice** | ✓ | ✗ | ✓ | ✓ |

**Análisis**:
- `UserManager` aparece en 4 requisitos → **Responsabilidad poco clara**
- `PaymentService` solo en 1 requisito → **Responsabilidad bien definida** ✅

**Refactorización**:

Dividir `UserManager`:
- `AuthenticationService` (Login, Register)
- `OrderProcessor` (Process Payment)
- `InvoiceService` (Generate Invoice)

## Caso Práctico Integrado

### Código Original

```java
public class BlogManager {
    private Database db;
    private EmailService emailService;
    private ImageProcessor imageProcessor;
    private AnalyticsTracker analytics;
    
    public void publishPost(BlogPost post) {
        // Validación
        if (post.getTitle() == null || post.getTitle().isEmpty()) {
            throw new IllegalArgumentException("Title required");
        }
        
        // Procesar imágenes
        if (post.hasImages()) {
            for (Image img : post.getImages()) {
                Image resized = imageProcessor.resize(img, 800, 600);
                Image thumbnail = imageProcessor.createThumbnail(img, 200, 150);
                img.setProcessedVersion(resized);
                img.setThumbnail(thumbnail);
            }
        }
        
        // Guardar
        db.execute("INSERT INTO posts VALUES (?, ?)", post.getTitle(), post.getContent());
        
        // Notificar suscriptores
        List<String> subscribers = db.query("SELECT email FROM subscribers");
        for (String email : subscribers) {
            emailService.send(email, "New post: " + post.getTitle(), post.getSummary());
        }
        
        // Analytics
        analytics.track("post_published", post.getId());
        
        // Log
        System.out.println("Published: " + post.getTitle() + " at " + new Date());
    }
}
```

### Aplicando las 7 Técnicas

#### 1. **Análisis LCOM**

Variables:
- `db`: Usado por persistencia y consulta de suscriptores
- `emailService`: Usado solo por notificación
- `imageProcessor`: Usado solo por procesamiento de imágenes
- `analytics`: Usado solo por tracking

**LCOM alto** → Baja cohesión

#### 2. **Análisis de Dependencias**

Imports de 4 dominios:
- Persistencia (`Database`)
- Comunicación (`EmailService`)
- Procesamiento multimedia (`ImageProcessor`)
- Analytics (`AnalyticsTracker`)

**4 dominios** → **4 responsabilidades**

#### 3. **Análisis de Actores**

| Funcionalidad | Actor |
|---------------|-------|
| Validación | Editores de contenido |
| Procesamiento de imágenes | Equipo de diseño |
| Persistencia | IT/DBA |
| Notificaciones | Marketing |
| Analytics | Data Science |

**5 actores** → **5 responsabilidades**

#### 4. **Análisis de Cambios** (hipotético)

```
Commits recientes:
- "Change image resize dimensions" (Diseño)
- "Migrate to MongoDB" (IT)
- "Update email template" (Marketing)
- "Add Google Analytics integration" (Data Science)
```

**Múltiples equipos** modificando la misma clase

#### 5. **Análisis de Nombre**

"BlogManager" → Nombre genérico "Manager" → **Sospechoso**

#### 6. **Análisis de Testing**

Test requeriría:
- Mock de `Database`
- Mock de `EmailService`
- Mock de `ImageProcessor`
- Mock de `AnalyticsTracker`

**4 mocks** → Complejidad alta

#### 7. **Matriz de Requisitos**

| Requisito | BlogManager |
|-----------|-------------|
| Validar post | ✓ |
| Procesar imágenes | ✓ |
| Persistir post | ✓ |
| Notificar usuarios | ✓ |
| Tracking | ✓ |

**5 requisitos** en 1 clase → **Violación clara**

### Refactorización Final

```java
// Responsabilidad 1: Validación
public class BlogPostValidator {
    public void validate(BlogPost post) {
        if (post.getTitle() == null || post.getTitle().isEmpty()) {
            throw new ValidationException("Title required");
        }
    }
}

// Responsabilidad 2: Procesamiento de imágenes
public class BlogImageProcessor {
    private final ImageProcessor imageProcessor;
    
    public void processImages(BlogPost post) {
        if (!post.hasImages()) return;
        
        for (Image img : post.getImages()) {
            Image resized = imageProcessor.resize(img, 800, 600);
            Image thumbnail = imageProcessor.createThumbnail(img, 200, 150);
            img.setProcessedVersion(resized);
            img.setThumbnail(thumbnail);
        }
    }
}

// Responsabilidad 3: Persistencia
public class BlogPostRepository {
    private final Database db;
    
    public void save(BlogPost post) {
        db.execute("INSERT INTO posts VALUES (?, ?)", 
                  post.getTitle(), post.getContent());
    }
}

// Responsabilidad 4: Notificaciones
public class BlogNotificationService {
    private final EmailService emailService;
    private final SubscriberRepository subscriberRepository;
    
    public void notifySubscribers(BlogPost post) {
        List<String> subscribers = subscriberRepository.findAllEmails();
        for (String email : subscribers) {
            emailService.send(email, 
                            "New post: " + post.getTitle(), 
                            post.getSummary());
        }
    }
}

// Responsabilidad 5: Analytics
public class BlogAnalyticsService {
    private final AnalyticsTracker analytics;
    
    public void trackPublication(BlogPost post) {
        analytics.track("post_published", post.getId());
    }
}

// Responsabilidad 6: Auditoría
public class BlogAuditLogger {
    public void logPublication(BlogPost post) {
        System.out.println("Published: " + post.getTitle() + 
                          " at " + LocalDateTime.now());
    }
}

// Orquestador (ahora solo coordina)
public class BlogPublishingService {
    private final BlogPostValidator validator;
    private final BlogImageProcessor imageProcessor;
    private final BlogPostRepository repository;
    private final BlogNotificationService notificationService;
    private final BlogAnalyticsService analyticsService;
    private final BlogAuditLogger auditLogger;
    
    public void publishPost(BlogPost post) {
        validator.validate(post);
        imageProcessor.processImages(post);
        repository.save(post);
        notificationService.notifySubscribers(post);
        analyticsService.trackPublication(post);
        auditLogger.logPublication(post);
    }
}
```

## Checklist de Identificación de Responsabilidades

Usa este checklist al analizar cualquier clase:

- [ ] **LCOM**: ¿Los métodos comparten variables de instancia?
- [ ] **Dependencias**: ¿Importa clases de múltiples dominios no relacionados?
- [ ] **Actores**: ¿Cuántos stakeholders diferentes podrían solicitar cambios?
- [ ] **Historial**: ¿La clase ha sido modificada por múltiples razones de negocio?
- [ ] **Nombre**: ¿El nombre contiene "Manager", "Handler", "Util" o "And"?
- [ ] **Testing**: ¿Los tests requieren muchos mocks (>3)?
- [ ] **Requisitos**: ¿La clase implementa múltiples casos de uso no relacionados?

**Si respondes "Sí" a 3+ preguntas**: Alta probabilidad de violación de SRP.

## Herramientas Automatizadas

### SonarQube

```bash
# Analizar cohesión
sonar-scanner -Dsonar.projectKey=my-project
# Buscar en dashboard: "LCOM4 metric"
```

### IntelliJ IDEA

```
Analyze → Inspect Code → "Class metrics" → "Lack of cohesion methods"
```

### JDepend

```java
// Analizar dependencias
jdepend.xmlui.JDepend
// Output: Package dependencies y abstractness
```

## Resumen Ejecutivo

### Técnicas de Identificación

| Técnica | Indicador Clave | Umbral de Alarma |
|---------|-----------------|------------------|
| **LCOM** | Cohesión de métodos | LCOM > 0 |
| **Dependencias** | Dominios importados | > 2 dominios |
| **Actores** | Stakeholders únicos | > 1 actor |
| **Frecuencia de cambio** | Commits por actor | > 1 actor |
| **Nombres** | Palabras vagas | "Manager", "Handler", "And" |
| **Testing** | Número de mocks | > 3 mocks |
| **Requisitos** | Casos de uso | > 2 casos de uso |

### Workflow Recomendado

1. **Análisis rápido** (5 min): Nombre + Número de dependencias
2. **Análisis medio** (15 min): Actores + Testing
3. **Análisis profundo** (30 min): LCOM + Historial de cambios

### Señales de Alerta Temprana

🚨 **ALERTA CRÍTICA** si encuentras:
- Clase con >10 métodos públicos
- Clase con >5 dependencias inyectadas
- Tests que requieren >5 mocks
- Nombre de clase con "Manager", "Service", "Handler" sin contexto específico

---

**Próximo paso**: En el siguiente subtema (**2.2.1: God Classes y Feature Envy**) estudiaremos antipatrones específicos que violan SRP y cómo refactorizarlos sistemáticamente.
