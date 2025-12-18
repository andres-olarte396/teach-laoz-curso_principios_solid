# Single Responsibility Principle (SRP) - Definición Formal

## Contexto y Motivación

El **Single Responsibility Principle (SRP)** es el primero de los principios SOLID y, paradójicamente, uno de los más malinterpretados. Robert C. Martin lo definió como:

> "Una clase debe tener una, y solo una, razón para cambiar"

Esta definición aparentemente simple esconde profundas implicaciones sobre cómo estructuramos nuestro código. El SRP no significa que una clase debe hacer solo una cosa (eso sería demasiado restrictivo), sino que debe tener **una única responsabilidad** bien definida.

### ¿Por qué es importante?

En sistemas complejos, el cambio es inevitable. Los requisitos evolucionan, se descubren bugs, se agregan características. El SRP nos ayuda a:

1. **Minimizar el impacto del cambio**: Si cada clase tiene una única razón para cambiar, las modificaciones se localizan
2. **Facilitar la comprensión**: Clases con responsabilidades claras son más fáciles de entender
3. **Mejorar la testabilidad**: Responsabilidades únicas se traducen en tests más simples
4. **Reducir el acoplamiento**: Clases enfocadas tienen menos dependencias

## Definición Profunda: "Una Razón para Cambiar"

La clave del SRP está en entender qué significa "una razón para cambiar". No se refiere a cambios técnicos arbitrarios, sino a **cambios impulsados por diferentes actores del negocio**.

### Ejemplo Conceptual

Consideremos una clase `Employee`:

```java
public class Employee {
    private String name;
    private double hourlyRate;
    
    // Responsabilidad 1: Cálculo de salario (Departamento de Finanzas)
    public double calculatePay() {
        // Lógica de cálculo según políticas de RRHH
        return hourlyRate * 160; // 160 horas/mes
    }
    
    // Responsabilidad 2: Persistencia (Departamento de IT)
    public void save() {
        // Lógica de guardado en base de datos
        Database.execute("INSERT INTO employees...");
    }
    
    // Responsabilidad 3: Reportes (Departamento de Auditoría)
    public String describeEmployee() {
        // Formato específico para reportes
        return "Employee Report:\n" +
               "Name: " + name + "\n" +
               "Monthly Pay: " + calculatePay();
    }
}
```

**Problema**: Esta clase tiene **tres razones para cambiar**:

1. **Finanzas** podría cambiar la política de cálculo de salarios (overtime, bonos)
2. **IT** podría migrar de base de datos relacional a NoSQL
3. **Auditoría** podría cambiar el formato de reportes (JSON, XML, PDF)

Cada cambio afecta a la misma clase, incrementando el riesgo de:
- **Conflictos de merge** si múltiples equipos modifican `Employee` simultáneamente
- **Bugs introducidos** al cambiar código para un propósito y romper funcionalidad de otro
- **Dificultad de testing** al tener que mockear bases de datos para testear cálculo de salarios

### La Solución: Separar Responsabilidades

```java
// Responsabilidad única: Representar datos del empleado
public class Employee {
    private String name;
    private double hourlyRate;
    
    // Getters, setters simples
    public String getName() { return name; }
    public double getHourlyRate() { return hourlyRate; }
}

// Responsabilidad única: Cálculo de salarios (Dominio de Finanzas)
public class PayrollCalculator {
    public double calculatePay(Employee employee) {
        // Lógica compleja de cálculo
        return employee.getHourlyRate() * 160;
    }
}

// Responsabilidad única: Persistencia (Dominio de IT)
public class EmployeeRepository {
    public void save(Employee employee) {
        Database.execute("INSERT INTO employees...");
    }
    
    public Employee findById(String id) {
        // Lógica de consulta
        return new Employee();
    }
}

// Responsabilidad única: Generación de reportes (Dominio de Auditoría)
public class EmployeeReportFormatter {
    public String format(Employee employee, double pay) {
        return "Employee Report:\n" +
               "Name: " + employee.getName() + "\n" +
               "Monthly Pay: " + pay;
    }
}
```

**Beneficios inmediatos**:

✅ **Cambios localizados**: Cambiar el formato de reportes no afecta cálculo ni persistencia  
✅ **Testing simplificado**: `PayrollCalculator` se testea sin bases de datos  
✅ **Reutilización**: `EmployeeRepository` puede usarse en otros contextos  
✅ **Comprensión clara**: Cada clase tiene un propósito evidente

## Identificando Responsabilidades: El Test del Actor

Robert C. Martin propone el **"Test del Actor"** para identificar responsabilidades:

> Una responsabilidad es una familia de funciones que sirven a un actor particular

**Actores** son grupos de stakeholders que pueden requerir cambios en el sistema:
- Departamento de RRHH (políticas de personal)
- Departamento Financiero (cálculos contables)
- Departamento Legal (cumplimiento normativo)
- Usuarios finales (interfaces y experiencia)

### Ejercicio Mental: ¿Quién pide el cambio?

Para cada método de una clase, pregúntate:

1. ¿Qué actor del negocio solicitaría cambios en este método?
2. ¿Hay métodos solicitados por diferentes actores?
3. Si respuesta = Sí → **Violación de SRP**

**Ejemplo aplicado**:

```java
public class Invoice {
    private List<LineItem> items;
    
    // Actor: Departamento de Ventas (políticas de descuento)
    public double calculateTotal() { 
        return items.stream()
                    .mapToDouble(LineItem::getPrice)
                    .sum();
    }
    
    // Actor: Departamento de IT (formato de almacenamiento)
    public void saveToDatabase() {
        // SQL insert logic
    }
    
    // Actor: Departamento de Marketing (diseño de facturas)
    public void printInvoice() {
        System.out.println("=== INVOICE ===");
        // Formato específico de impresión
    }
}
```

**Análisis**:
- 3 métodos → 3 actores diferentes → **Violación clara de SRP**
- Solución: Separar en `Invoice` (datos), `InvoiceCalculator`, `InvoiceRepository`, `InvoicePrinter`

## Niveles de Granularidad

El SRP se aplica en múltiples niveles de abstracción:

### 1. Nivel de Método

Un método debe hacer **una cosa** bien definida:

```java
// ❌ MAL: Método hace múltiples cosas
public void processOrder(Order order) {
    validateOrder(order);           // 1. Validación
    double total = calculateTotal(order); // 2. Cálculo
    applyDiscount(order, total);    // 3. Descuentos
    updateInventory(order);         // 4. Inventario
    sendConfirmationEmail(order);   // 5. Notificación
    saveToDatabase(order);          // 6. Persistencia
}

// ✅ BIEN: Método orquesta, delega responsabilidades
public void processOrder(Order order) {
    OrderValidator.validate(order);
    OrderProcessor processor = new OrderProcessor(order);
    processor.process();
}
```

### 2. Nivel de Clase

Una clase debe tener **una responsabilidad cohesiva**:

```java
// ❌ MAL: Clase "God Object" con múltiples responsabilidades
public class UserManager {
    public void createUser(User user) { }      // CRUD
    public void deleteUser(String id) { }      // CRUD
    public void sendWelcomeEmail(User user) { }// Notificación
    public boolean authenticate(String user, String pwd) { } // Seguridad
    public void logActivity(String action) { } // Auditoría
    public void exportUsersToCSV() { }        // Reportes
}

// ✅ BIEN: Clases especializadas
public class UserRepository { /* CRUD */ }
public class UserNotificationService { /* Emails */ }
public class AuthenticationService { /* Seguridad */ }
public class AuditLogger { /* Logs */ }
public class UserReportGenerator { /* Reportes */ }
```

### 3. Nivel de Módulo/Paquete

Un módulo debe agrupar clases con **responsabilidades relacionadas**:

```
com.company.ecommerce.
├── domain/           // Responsabilidad: Lógica de negocio
│   ├── Product.java
│   └── Order.java
├── persistence/      // Responsabilidad: Acceso a datos
│   ├── ProductRepository.java
│   └── OrderRepository.java
└── notification/     // Responsabilidad: Comunicaciones
    ├── EmailService.java
    └── SMSService.java
```

## Cohesión y SRP

El SRP está íntimamente ligado al concepto de **cohesión**. Una clase cohesiva es aquella cuyos métodos y datos están estrechamente relacionados.

### Métrica: LCOM (Lack of Cohesion of Methods)

Mide cuántos métodos comparten variables de instancia:

```java
// ALTA cohesión (LCOM bajo) - ✅ CUMPLE SRP
public class Rectangle {
    private double width;
    private double height;
    
    public double getArea() { 
        return width * height; // Usa width y height
    }
    
    public double getPerimeter() { 
        return 2 * (width + height); // Usa width y height
    }
    
    // Todos los métodos usan todas las variables → Alta cohesión
}

// BAJA cohesión (LCOM alto) - ❌ VIOLA SRP
public class MixedResponsibilities {
    private double width;
    private double height;
    private Connection dbConnection;
    private EmailService emailService;
    
    public double getArea() { 
        return width * height; // Solo usa width, height
    }
    
    public void saveToDb() {
        dbConnection.execute("..."); // Solo usa dbConnection
    }
    
    public void sendNotification() {
        emailService.send("..."); // Solo usa emailService
    }
    
    // Métodos no comparten variables → Baja cohesión → Múltiples responsabilidades
}
```

**Regla práctica**: Si variables de instancia se usan solo por subconjuntos disjuntos de métodos, probablemente hay múltiples responsabilidades.

## Errores Comunes al Aplicar SRP

### 1. Confundir "Una responsabilidad" con "Un método"

```java
// ❌ SOBRE-SIMPLIFICACIÓN
public class EmailSender {
    public void send(Email email) { } // Solo un método, pero ¿es útil?
}

// La clase es tan pequeña que no agrega valor. Mejor:
public class EmailService {
    public void send(Email email) { }
    public void sendBulk(List<Email> emails) { }
    public void scheduleEmail(Email email, LocalDateTime time) { }
    // Todas relacionadas con "envío de emails" - Una responsabilidad cohesiva
}
```

### 2. Fragmentación Excesiva

```java
// ❌ DEMASIADAS CLASES para algo simple
public class PasswordLengthValidator { }
public class PasswordUppercaseValidator { }
public class PasswordNumberValidator { }
public class PasswordSpecialCharValidator { }
// 4 clases para una validación simple

// ✅ MEJOR: Responsabilidad cohesiva
public class PasswordValidator {
    public ValidationResult validate(String password) {
        // Todas las reglas en un lugar - Una responsabilidad: validar passwords
    }
}
```

### 3. Ignorar el Contexto del Negocio

```java
// En una startup pequeña, esto puede ser aceptable:
public class User {
    public void save() { }
    public String toJson() { }
}

// En una empresa grande con múltiples equipos, separar:
public class User { } // Solo datos
public class UserRepository { } // Persistencia
public class UserSerializer { } // Serialización
```

**Principio**: El SRP es pragmático, no dogmático. La "granularidad correcta" depende del contexto.

## Relación con Otros Principios SOLID

El SRP es la base sobre la que se construyen los demás principios:

- **OCP**: Clases con responsabilidades únicas son más fáciles de extender sin modificar
- **LSP**: Responsabilidades claras facilitan contratos de sustitución
- **ISP**: Interfaces cohesivas (una responsabilidad por interfaz) derivan del SRP
- **DIP**: Abstracciones estables emergen de responsabilidades bien definidas

## Testing como Validador de SRP

Una clase que cumple SRP típicamente:

✅ Tiene tests **cortos** y **enfocados**  
✅ Requiere **pocos mocks** (baja dependencia)  
✅ Se testea **rápidamente** (sin setup complejo)  
✅ Los tests **no se rompen juntos** (cambios localizados)

```java
// ✅ FÁCIL de testear (cumple SRP)
@Test
public void testPayrollCalculation() {
    Employee emp = new Employee("John", 50.0);
    PayrollCalculator calc = new PayrollCalculator();
    
    double pay = calc.calculatePay(emp);
    
    assertEquals(8000.0, pay); // Sin mocks, sin base de datos
}

// ❌ DIFÍCIL de testear (viola SRP)
@Test
public void testEmployeeWithDatabase() {
    // Setup de base de datos
    Database mockDb = mock(Database.class);
    EmailService mockEmail = mock(EmailService.class);
    
    Employee emp = new Employee("John", 50.0);
    emp.setDatabase(mockDb);
    emp.setEmailService(mockEmail);
    
    double pay = emp.calculatePay(); // ¿Por qué necesito DB para calcular?
    
    verify(mockDb, never()).execute(any()); // Verificaciones irrelevantes
}
```

## Caso Práctico: Refactorización Real

### Antes (Violación de SRP)

```java
public class BlogPost {
    private String title;
    private String content;
    private LocalDateTime publishedAt;
    private String author;
    
    // Responsabilidad 1: Validación de negocio
    public boolean isValid() {
        return title != null && title.length() > 5 &&
               content != null && content.length() > 100;
    }
    
    // Responsabilidad 2: Formateo HTML
    public String toHtml() {
        return "<article>" +
               "<h1>" + title + "</h1>" +
               "<p>" + content + "</p>" +
               "<footer>By " + author + " on " + publishedAt + "</footer>" +
               "</article>";
    }
    
    // Responsabilidad 3: Formateo JSON
    public String toJson() {
        return "{\"title\":\"" + title + "\"," +
               "\"content\":\"" + content + "\"," +
               "\"author\":\"" + author + "\"}";
    }
    
    // Responsabilidad 4: Cálculo de métricas
    public int estimatedReadingTime() {
        int wordCount = content.split("\\s+").length;
        return wordCount / 200; // 200 palabras por minuto
    }
    
    // Responsabilidad 5: Persistencia
    public void save() {
        Database.execute("INSERT INTO posts VALUES (?, ?, ?)", 
                        title, content, author);
    }
}
```

**Problemas identificados**:
1. **Cambio de formato HTML** (diseñador) afecta clase usada por cálculos
2. **Cambio de base de datos** (DBA) requiere tocar clase de dominio
3. **Testing requiere** mockear DB para validar formateo
4. **Imposible reutilizar** formateo sin arrastrar lógica de persistencia

### Después (Cumple SRP)

```java
// Responsabilidad única: Representar un post de blog
public class BlogPost {
    private String title;
    private String content;
    private LocalDateTime publishedAt;
    private String author;
    
    // Constructor, getters, setters
    public String getTitle() { return title; }
    public String getContent() { return content; }
    public String getAuthor() { return author; }
    public LocalDateTime getPublishedAt() { return publishedAt; }
}

// Responsabilidad única: Validar posts
public class BlogPostValidator {
    private static final int MIN_TITLE_LENGTH = 5;
    private static final int MIN_CONTENT_LENGTH = 100;
    
    public ValidationResult validate(BlogPost post) {
        List<String> errors = new ArrayList<>();
        
        if (post.getTitle() == null || post.getTitle().length() < MIN_TITLE_LENGTH) {
            errors.add("Title must be at least " + MIN_TITLE_LENGTH + " characters");
        }
        
        if (post.getContent() == null || post.getContent().length() < MIN_CONTENT_LENGTH) {
            errors.add("Content must be at least " + MIN_CONTENT_LENGTH + " characters");
        }
        
        return new ValidationResult(errors.isEmpty(), errors);
    }
}

// Responsabilidad única: Formatear a HTML
public class HtmlBlogPostFormatter {
    public String format(BlogPost post) {
        return "<article>" +
               "<h1>" + escapeHtml(post.getTitle()) + "</h1>" +
               "<p>" + escapeHtml(post.getContent()) + "</p>" +
               "<footer>By " + post.getAuthor() + 
               " on " + post.getPublishedAt() + "</footer>" +
               "</article>";
    }
    
    private String escapeHtml(String text) {
        // Implementación de escape HTML
        return text.replace("<", "&lt;").replace(">", "&gt;");
    }
}

// Responsabilidad única: Formatear a JSON
public class JsonBlogPostFormatter {
    public String format(BlogPost post) {
        return String.format(
            "{\"title\":\"%s\",\"content\":\"%s\",\"author\":\"%s\",\"publishedAt\":\"%s\"}",
            escapeJson(post.getTitle()),
            escapeJson(post.getContent()),
            post.getAuthor(),
            post.getPublishedAt()
        );
    }
    
    private String escapeJson(String text) {
        return text.replace("\"", "\\\"").replace("\n", "\\n");
    }
}

// Responsabilidad única: Calcular métricas de lectura
public class ReadingTimeCalculator {
    private static final int WORDS_PER_MINUTE = 200;
    
    public int estimateMinutes(String content) {
        int wordCount = content.split("\\s+").length;
        return Math.max(1, wordCount / WORDS_PER_MINUTE);
    }
}

// Responsabilidad única: Persistir posts
public class BlogPostRepository {
    private final Database database;
    
    public BlogPostRepository(Database database) {
        this.database = database;
    }
    
    public void save(BlogPost post) {
        database.execute(
            "INSERT INTO posts (title, content, author, published_at) VALUES (?, ?, ?, ?)",
            post.getTitle(),
            post.getContent(),
            post.getAuthor(),
            post.getPublishedAt()
        );
    }
    
    public BlogPost findById(Long id) {
        // Lógica de consulta
        return null; // Placeholder
    }
}
```

**Beneficios logrados**:

| Aspecto | Antes | Después |
|---------|-------|---------|
| **Líneas por clase** | ~80 | 10-30 cada una |
| **Dependencias externas** | Database en clase de dominio | Solo en Repository |
| **Testing** | Requiere DB mock | Cada clase se testea aisladamente |
| **Reutilización** | Imposible separar formateo | Formatters reutilizables en otros contextos |
| **Mantenibilidad** | Cambio HTML afecta todo | Cambios localizados |

### Tests Comparativos

```java
// Test ANTES (complejo, acoplado)
@Test
public void testBlogPostHtml() {
    Database mockDb = mock(Database.class);
    Database.setInstance(mockDb); // Configuración global
    
    BlogPost post = new BlogPost("Title", "Content...", "Author");
    String html = post.toHtml();
    
    assertTrue(html.contains("<h1>Title</h1>"));
    verify(mockDb, never()).execute(any()); // ¿Por qué esto es necesario?
}

// Test DESPUÉS (simple, enfocado)
@Test
public void testHtmlFormatter() {
    BlogPost post = new BlogPost("Test Title", "Test content", "John Doe");
    HtmlBlogPostFormatter formatter = new HtmlBlogPostFormatter();
    
    String html = formatter.format(post);
    
    assertTrue(html.contains("<h1>Test Title</h1>"));
    assertTrue(html.contains("<p>Test content</p>"));
    // Sin mocks, sin setup, test rápido
}

@Test
public void testReadingTimeCalculator() {
    ReadingTimeCalculator calculator = new ReadingTimeCalculator();
    String content = String.join(" ", Collections.nCopies(400, "word"));
    
    int minutes = calculator.estimateMinutes(content);
    
    assertEquals(2, minutes); // 400 palabras / 200 por minuto
}
```

## Resumen Ejecutivo

### Puntos Clave

1. **SRP = Una razón para cambiar** (un actor del negocio que solicita modificaciones)
2. **Cohesión alta** = Métodos y datos relacionados = Cumple SRP
3. **Testing fácil** = Indicador de cumplimiento de SRP
4. **Pragmatismo** > Dogmatismo (contexto importa)

### Checklist de Validación SRP

Al diseñar/revisar una clase, pregúntate:

- [ ] ¿Cuántos actores del negocio podrían solicitar cambios en esta clase?
- [ ] ¿Los métodos comparten las mismas variables de instancia?
- [ ] ¿Puedo describir la responsabilidad de la clase en una oración clara?
- [ ] ¿Los tests de esta clase son simples y enfocados?
- [ ] ¿Un cambio en una parte de la clase afecta otras partes no relacionadas?

**Si respondes > 1 actor, o "No" a las demás**: Posible violación de SRP.

### Anti-Patrones a Evitar

❌ **God Classes** (clases que hacen todo)  
❌ **Clases con nombres vagos** (Manager, Handler, Util)  
❌ **Alta dependencia de mocks** en tests  
❌ **Fragmentación excesiva** (micro-clases sin valor)

### Próximos Pasos

En los siguientes subtemas profundizaremos en:
- **Identificación sistemática** de responsabilidades
- **Violaciones comunes** (God Classes, Feature Envy)
- **Técnicas de refactoring** (Extract Class, Extract Method)
- **SRP en diferentes lenguajes** (Java, C#, TypeScript, Python)

---

**Recuerda**: El SRP no es sobre hacer clases pequeñas, es sobre hacer clases **cohesivas** con **una razón clara para existir y cambiar**.
