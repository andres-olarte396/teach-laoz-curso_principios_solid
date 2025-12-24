# Ejercicios: Aplicación Práctica del SRP

## Nivel 1. Refactorización Guiada ⭐

### Ejercicio 1.1. Identificar Técnicas de Extracción

Analiza el siguiente código y determina qué técnica de extracción (Extract Class, Extract Service, Extract Interface, Move Method) aplicarías para cada responsabilidad:

```java
public class BlogPost {
    private String title;
    private String content;
    private String authorEmail;
    private LocalDateTime publishDate;
    
    // Responsabilidad 1
    public boolean isValid() {
        return title != null && !title.isEmpty() 
            && content != null && content.length() > 100;
    }
    
    // Responsabilidad 2
    public void save() {
        Connection conn = Database.getConnection();
        String sql = "INSERT INTO posts...";
        // ... código de persistencia
    }
    
    // Responsabilidad 3
    public void sendNotification() {
        EmailSender sender = new EmailSender();
        sender.send(authorEmail, "Post published", "...");
    }
    
    // Responsabilidad 4
    public String toHTML() {
        return "<article><h1>" + title + "</h1><p>" + content + "</p></article>";
    }
    
    // Responsabilidad 5
    public String toMarkdown() {
        return "# " + title + "\n\n" + content;
    }
}
```

**Tareas:**
1. Identifica cada responsabilidad
2. Determina la técnica de extracción apropiada
3. Justifica tu elección

**Solución Esperada:**

| Responsabilidad | Técnica | Justificación |
|----------------|---------|---------------|
| Validación | Extract Service | Lógica reutilizable que no es parte del modelo |
| Persistencia | Extract Service/Repository | Patrón Repository para separar persistencia |
| Notificaciones | Extract Service | Infraestructura externa, no dominio |
| Formateo HTML | Extract Class | Múltiples formatos → Extraer FormatterService |
| Formateo Markdown | Extract Interface | Crear PostFormatter interface con implementaciones |

---

### Ejercicio 1.2: Detectar Code Smells

Identifica qué code smell (Large Class, Divergent Change, Shotgun Surgery, Feature Envy) presenta cada fragmento:

**Fragmento A:**
```java
public class CustomerManager {
    // 50 métodos
    // 800 líneas de código
    // Maneja validación, persistencia, cálculos, reportes, notificaciones
}
```

**Fragmento B:**
```java
public class Order {
    public double calculateShipping(Address address) {
        double distance = address.getCoordinates().distanceTo(warehouse);
        double weight = address.getPackageWeight();
        return distance * 0.5 + weight * 0.2;
    }
}
```

**Fragmento C:**
```java
// Para cambiar el formato de fecha, debes modificar:
// UserFormatter.java
// OrderFormatter.java
// ReportGenerator.java
// EmailTemplate.java
// ... 10 clases más
```

**Fragmento D:**
```java
public class Product {
    void updatePrice() { } // Departamento Financiero pide cambios
    void saveToDB() { }    // DBAs piden cambios
    void generateXML() { } // API Team pide cambios
    void sendAlert() { }   // Operaciones pide cambios
}
```

**Solución:**
- A: **Large Class** (clase gigante con múltiples responsabilidades)
- B: **Feature Envy** (Order usa más datos de Address que propios)
- C: **Shotgun Surgery** (un cambio requiere tocar muchas clases)
- D: **Divergent Change** (la clase cambia por múltiples razones)

---

## Nivel 2: Refactorización Completa ⭐⭐

### Ejercicio 2.1. Refactorizar Sistema de Gestión de Tareas

Dado el siguiente código que viola SRP, refactorízalo aplicando los 5 pasos del proceso:

```java
public class Task {
    private String id;
    private String title;
    private String description;
    private String assignedTo;
    private TaskStatus status;
    private LocalDateTime dueDate;
    
    // Responsabilidad 1. Validación
    public boolean isValid() {
        if (title == null || title.isEmpty()) {
            return false;
        }
        if (assignedTo == null || !assignedTo.contains("@")) {
            return false;
        }
        if (dueDate != null && dueDate.isBefore(LocalDateTime.now())) {
            return false;
        }
        return true;
    }
    
    // Responsabilidad 2: Lógica de negocio
    public void complete() {
        if (status == TaskStatus.COMPLETED) {
            throw new IllegalStateException("Task already completed");
        }
        this.status = TaskStatus.COMPLETED;
        sendCompletionNotification();
    }
    
    public boolean isOverdue() {
        return dueDate != null && 
               dueDate.isBefore(LocalDateTime.now()) && 
               status != TaskStatus.COMPLETED;
    }
    
    // Responsabilidad 3: Persistencia
    public void save() {
        try {
            Connection conn = DriverManager.getConnection("jdbc:mysql://localhost/tasks");
            String sql = "INSERT INTO tasks (id, title, description, assigned_to, status, due_date) " +
                         "VALUES (?, ?, ?, ?, ?, ?)";
            PreparedStatement stmt = conn.prepareStatement(sql);
            stmt.setString(1, id);
            stmt.setString(2, title);
            stmt.setString(3, description);
            stmt.setString(4, assignedTo);
            stmt.setString(5, status.name());
            stmt.setTimestamp(6, Timestamp.valueOf(dueDate));
            stmt.executeUpdate();
        } catch (SQLException e) {
            throw new RuntimeException("Failed to save task", e);
        }
    }
    
    public static Task load(String id) {
        // Similar código JDBC para cargar
        return null;
    }
    
    // Responsabilidad 4: Notificaciones
    private void sendCompletionNotification() {
        EmailSender sender = new EmailSender();
        String message = "Task '" + title + "' has been completed";
        sender.send(assignedTo, "Task Completed", message);
    }
    
    public void sendReminderIfOverdue() {
        if (isOverdue()) {
            EmailSender sender = new EmailSender();
            sender.send(assignedTo, "Overdue Task", "Task '" + title + "' is overdue");
        }
    }
    
    // Responsabilidad 5: Formateo
    public String toJSON() {
        return String.format(
            "{\"id\":\"%s\",\"title\":\"%s\",\"status\":\"%s\"}",
            id, title, status
        );
    }
    
    public String toHTML() {
        return String.format(
            "<div class='task'><h3>%s</h3><p>%s</p><span>%s</span></div>",
            title, description, status
        );
    }
}
```

**Tareas:**
1. Aplicar Paso 1. Identificar responsabilidades y sus actores
2. Aplicar Paso 2: Analizar dependencias de datos
3. Aplicar Paso 3: Extraer clases nuevas
4. Aplicar Paso 4: Escribir código cliente actualizado
5. Aplicar Paso 5: Escribir tests unitarios para cada clase

---

**SOLUCIÓN MODELO:**

**PASO 1. Identificar Responsabilidades**

| Responsabilidad | Métodos | Actor que pide cambios |
|----------------|---------|------------------------|
| Modelo de dominio | Getters, constructores | Product Owner |
| Validación | isValid() | Equipo de Validación |
| Lógica de negocio | complete(), isOverdue() | Product Owner |
| Persistencia | save(), load() | DBAs |
| Notificaciones | sendCompletionNotification(), sendReminderIfOverdue() | Equipo de Comunicaciones |
| Formateo | toJSON(), toHTML() | Frontend Team |

**PASO 2: Analizar Dependencias**

```
isValid() → usa title, assignedTo, dueDate
complete() → usa status, necesita sendCompletionNotification()
save() → usa TODOS los campos
sendCompletionNotification() → usa title, assignedTo
toJSON() → usa id, title, status
```

**PASO 3: Extraer Clases**

```java
// ✅ Clase 1. Modelo de dominio puro
public class Task {
    private final String id;
    private String title;
    private String description;
    private String assignedTo;
    private TaskStatus status;
    private LocalDateTime dueDate;
    
    public Task(String id, String title, String description, 
                String assignedTo, LocalDateTime dueDate) {
        this.id = id;
        this.title = title;
        this.description = description;
        this.assignedTo = assignedTo;
        this.status = TaskStatus.PENDING;
        this.dueDate = dueDate;
    }
    
    // Solo cambio de estado (lógica mínima de dominio)
    public void markAsCompleted() {
        if (this.status == TaskStatus.COMPLETED) {
            throw new IllegalStateException("Task already completed");
        }
        this.status = TaskStatus.COMPLETED;
    }
    
    // Getters y setters básicos
    public String getId() { return id; }
    public String getTitle() { return title; }
    public String getDescription() { return description; }
    public String getAssignedTo() { return assignedTo; }
    public TaskStatus getStatus() { return status; }
    public LocalDateTime getDueDate() { return dueDate; }
}

// ✅ Clase 2: Validador
public class TaskValidator {
    public ValidationResult validate(Task task) {
        List<String> errors = new ArrayList<>();
        
        if (task.getTitle() == null || task.getTitle().isEmpty()) {
            errors.add("Title cannot be empty");
        }
        
        if (task.getAssignedTo() == null || !task.getAssignedTo().contains("@")) {
            errors.add("Invalid email address");
        }
        
        if (task.getDueDate() != null && task.getDueDate().isBefore(LocalDateTime.now())) {
            errors.add("Due date cannot be in the past");
        }
        
        return errors.isEmpty() 
            ? ValidationResult.success() 
            : ValidationResult.failure(errors);
    }
}

public class ValidationResult {
    private final boolean valid;
    private final List<String> errors;
    
    private ValidationResult(boolean valid, List<String> errors) {
        this.valid = valid;
        this.errors = errors;
    }
    
    public static ValidationResult success() {
        return new ValidationResult(true, Collections.emptyList());
    }
    
    public static ValidationResult failure(List<String> errors) {
        return new ValidationResult(false, errors);
    }
    
    public boolean isValid() { return valid; }
    public List<String> getErrors() { return errors; }
}

// ✅ Clase 3: Servicio de lógica de negocio
public class TaskService {
    private final TaskNotificationService notificationService;
    
    public TaskService(TaskNotificationService notificationService) {
        this.notificationService = notificationService;
    }
    
    public void completeTask(Task task) {
        task.markAsCompleted();
        notificationService.sendCompletionNotification(task);
    }
    
    public boolean isOverdue(Task task) {
        return task.getDueDate() != null &&
               task.getDueDate().isBefore(LocalDateTime.now()) &&
               task.getStatus() != TaskStatus.COMPLETED;
    }
    
    public void checkAndNotifyOverdue(Task task) {
        if (isOverdue(task)) {
            notificationService.sendOverdueReminder(task);
        }
    }
}

// ✅ Clase 4: Repositorio de persistencia
public interface TaskRepository {
    void save(Task task);
    Task findById(String id);
    List<Task> findByAssignedTo(String email);
}

public class JdbcTaskRepository implements TaskRepository {
    private final DataSource dataSource;
    
    public JdbcTaskRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    @Override
    public void save(Task task) {
        String sql = "INSERT INTO tasks (id, title, description, assigned_to, status, due_date) " +
                     "VALUES (?, ?, ?, ?, ?, ?) " +
                     "ON DUPLICATE KEY UPDATE title=?, description=?, status=?, due_date=?";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setString(1, task.getId());
            stmt.setString(2, task.getTitle());
            stmt.setString(3, task.getDescription());
            stmt.setString(4, task.getAssignedTo());
            stmt.setString(5, task.getStatus().name());
            stmt.setTimestamp(6, Timestamp.valueOf(task.getDueDate()));
            stmt.setString(7, task.getTitle());
            stmt.setString(8, task.getDescription());
            stmt.setString(9, task.getStatus().name());
            stmt.setTimestamp(10, Timestamp.valueOf(task.getDueDate()));
            
            stmt.executeUpdate();
        } catch (SQLException e) {
            throw new PersistenceException("Failed to save task: " + task.getId(), e);
        }
    }
    
    @Override
    public Task findById(String id) {
        String sql = "SELECT * FROM tasks WHERE id = ?";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setString(1, id);
            ResultSet rs = stmt.executeQuery();
            
            if (rs.next()) {
                return new Task(
                    rs.getString("id"),
                    rs.getString("title"),
                    rs.getString("description"),
                    rs.getString("assigned_to"),
                    rs.getTimestamp("due_date").toLocalDateTime()
                );
            }
            return null;
        } catch (SQLException e) {
            throw new PersistenceException("Failed to load task: " + id, e);
        }
    }
    
    @Override
    public List<Task> findByAssignedTo(String email) {
        // Similar implementación
        return new ArrayList<>();
    }
}

// ✅ Clase 5: Servicio de notificaciones
public class TaskNotificationService {
    private final EmailService emailService;
    
    public TaskNotificationService(EmailService emailService) {
        this.emailService = emailService;
    }
    
    public void sendCompletionNotification(Task task) {
        String subject = "Task Completed";
        String body = String.format("Task '%s' has been completed", task.getTitle());
        emailService.send(task.getAssignedTo(), subject, body);
    }
    
    public void sendOverdueReminder(Task task) {
        String subject = "Overdue Task";
        String body = String.format("Task '%s' is overdue. Please complete it ASAP.", 
                                     task.getTitle());
        emailService.send(task.getAssignedTo(), subject, body);
    }
    
    public void sendAssignmentNotification(Task task) {
        String subject = "New Task Assigned";
        String body = String.format("You have been assigned: %s", task.getTitle());
        emailService.send(task.getAssignedTo(), subject, body);
    }
}

// ✅ Clase 6: Formateadores
public interface TaskFormatter {
    String format(Task task);
}

public class JSONTaskFormatter implements TaskFormatter {
    @Override
    public String format(Task task) {
        return String.format(
            "{\"id\":\"%s\",\"title\":\"%s\",\"status\":\"%s\",\"assignedTo\":\"%s\"}",
            task.getId(),
            task.getTitle(),
            task.getStatus(),
            task.getAssignedTo()
        );
    }
}

public class HTMLTaskFormatter implements TaskFormatter {
    @Override
    public String format(Task task) {
        return String.format(
            "<div class='task' data-id='%s'>" +
            "<h3>%s</h3>" +
            "<p>%s</p>" +
            "<span class='status %s'>%s</span>" +
            "<span class='assigned'>%s</span>" +
            "</div>",
            task.getId(),
            task.getTitle(),
            task.getDescription(),
            task.getStatus().name().toLowerCase(),
            task.getStatus(),
            task.getAssignedTo()
        );
    }
}
```

**PASO 4: Código Cliente Actualizado**

```java
// Antes (todo en una clase)
Task task = new Task("T001", "Fix bug", "Critical bug", "dev@example.com", 
                     LocalDateTime.now().plusDays(2));
if (task.isValid()) {
    task.save();
    task.sendReminderIfOverdue();
}

// Después (usando servicios especializados)
Task task = new Task("T001", "Fix bug", "Critical bug", "dev@example.com",
                     LocalDateTime.now().plusDays(2));

TaskValidator validator = new TaskValidator();
ValidationResult result = validator.validate(task);

if (result.isValid()) {
    TaskRepository repository = new JdbcTaskRepository(dataSource);
    repository.save(task);
    
    TaskService taskService = new TaskService(notificationService);
    taskService.checkAndNotifyOverdue(task);
    
    JSONTaskFormatter formatter = new JSONTaskFormatter();
    String json = formatter.format(task);
}
```

**PASO 5: Tests Unitarios**

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import static org.junit.jupiter.api.Assertions.*;

class TaskValidatorTest {
    private TaskValidator validator;
    
    @BeforeEach
    void setUp() {
        validator = new TaskValidator();
    }
    
    @Test
    void shouldAcceptValidTask() {
        Task task = new Task("T001", "Valid Task", "Description",
                             "user@example.com", LocalDateTime.now().plusDays(1));
        
        ValidationResult result = validator.validate(task);
        
        assertTrue(result.isValid());
        assertTrue(result.getErrors().isEmpty());
    }
    
    @Test
    void shouldRejectEmptyTitle() {
        Task task = new Task("T001", "", "Description",
                             "user@example.com", LocalDateTime.now().plusDays(1));
        
        ValidationResult result = validator.validate(task);
        
        assertFalse(result.isValid());
        assertTrue(result.getErrors().contains("Title cannot be empty"));
    }
    
    @Test
    void shouldRejectInvalidEmail() {
        Task task = new Task("T001", "Task", "Description",
                             "invalid-email", LocalDateTime.now().plusDays(1));
        
        ValidationResult result = validator.validate(task);
        
        assertFalse(result.isValid());
        assertTrue(result.getErrors().contains("Invalid email address"));
    }
    
    @Test
    void shouldRejectPastDueDate() {
        Task task = new Task("T001", "Task", "Description",
                             "user@example.com", LocalDateTime.now().minusDays(1));
        
        ValidationResult result = validator.validate(task);
        
        assertFalse(result.isValid());
        assertTrue(result.getErrors().contains("Due date cannot be in the past"));
    }
}

class TaskServiceTest {
    private TaskService taskService;
    private TaskNotificationService mockNotificationService;
    
    @BeforeEach
    void setUp() {
        mockNotificationService = mock(TaskNotificationService.class);
        taskService = new TaskService(mockNotificationService);
    }
    
    @Test
    void shouldCompleteTaskAndSendNotification() {
        Task task = new Task("T001", "Task", "Description",
                             "user@example.com", LocalDateTime.now().plusDays(1));
        
        taskService.completeTask(task);
        
        assertEquals(TaskStatus.COMPLETED, task.getStatus());
        verify(mockNotificationService).sendCompletionNotification(task);
    }
    
    @Test
    void shouldDetectOverdueTask() {
        Task task = new Task("T001", "Task", "Description",
                             "user@example.com", LocalDateTime.now().minusDays(1));
        
        boolean isOverdue = taskService.isOverdue(task);
        
        assertTrue(isOverdue);
    }
    
    @Test
    void shouldNotDetectOverdueForCompletedTask() {
        Task task = new Task("T001", "Task", "Description",
                             "user@example.com", LocalDateTime.now().minusDays(1));
        task.markAsCompleted();
        
        boolean isOverdue = taskService.isOverdue(task);
        
        assertFalse(isOverdue);
    }
}

class JSONTaskFormatterTest {
    private JSONTaskFormatter formatter;
    
    @BeforeEach
    void setUp() {
        formatter = new JSONTaskFormatter();
    }
    
    @Test
    void shouldFormatTaskAsJSON() {
        Task task = new Task("T001", "Test Task", "Description",
                             "user@example.com", LocalDateTime.now().plusDays(1));
        
        String json = formatter.format(task);
        
        assertTrue(json.contains("\"id\":\"T001\""));
        assertTrue(json.contains("\"title\":\"Test Task\""));
        assertTrue(json.contains("\"status\":\"PENDING\""));
    }
}
```

---

## Nivel 3: Diseño desde Cero ⭐⭐⭐

### Ejercicio 3.1. Sistema de Gestión de Inventario

Diseña un sistema de gestión de inventario para una tienda online que cumpla con SRP. El sistema debe:

**Requisitos Funcionales:**
1. **Productos**: Gestionar productos (nombre, SKU, precio, stock)
2. **Validación**: Validar productos antes de agregar al inventario
3. **Persistencia**: Guardar productos en base de datos
4. **Movimientos de Stock**: Registrar entradas y salidas de inventario
5. **Alertas**: Notificar cuando stock esté bajo (< 10 unidades)
6. **Reportes**: Generar reportes de inventario en JSON y CSV
7. **Búsqueda**: Buscar productos por diferentes criterios

**Tareas:**
1. Identificar responsabilidades separadas
2. Diseñar clases aplicando SRP
3. Definir interfaces entre clases
4. Implementar al menos 3 clases completas
5. Escribir tests unitarios

**Guía de Solución:**

Responsabilidades identificadas:
- **Product** (modelo): Datos del producto
- **ProductValidator**: Validaciones de negocio
- **ProductRepository**: Persistencia CRUD
- **InventoryMovementService**: Gestión de movimientos de stock
- **StockAlertService**: Monitoreo y alertas
- **InventoryReportGenerator**: Generación de reportes
- **ProductSearchService**: Búsquedas y filtros

---

## Nivel 4: Proyecto Integrador ⭐⭐⭐⭐

### Ejercicio 4.1. Sistema de Biblioteca Digital

Diseña e implementa un sistema completo de biblioteca digital que permita:

**Funcionalidades:**
1. **Gestión de Libros**: Agregar, actualizar, eliminar libros
2. **Gestión de Usuarios**: Registro, autenticación, perfiles
3. **Préstamos**: Prestar y devolver libros
4. **Reservas**: Sistema de reservas cuando libro no está disponible
5. **Multas**: Calcular multas por retrasos
6. **Notificaciones**: Recordatorios de devolución, libros disponibles
7. **Búsqueda Avanzada**: Por título, autor, género, ISBN
8. **Reportes**: Estadísticas de préstamos, libros populares
9. **Auditoría**: Registro de todas las operaciones

**Restricciones:**
- Aplicar SRP rigurosamente
- Mínimo 15 clases bien definidas
- Usar patrones: Repository, Service, Strategy, Observer
- Tests unitarios con >80% cobertura
- Documentación JavaDoc completa

**Entregables:**
1. Diagrama de clases UML
2. Código fuente completo
3. Suite de tests JUnit
4. Documento de decisiones de diseño (por qué se separó cada responsabilidad)

---

**SOLUCIÓN MODELO (Esquema):**

```java
// Capa de Dominio (Modelos)
public class Book { }
public class User { }
public class Loan { }
public class Reservation { }
public class Fine { }

// Capa de Validación
public class BookValidator { }
public class UserValidator { }
public class LoanValidator { }

// Capa de Persistencia (Repositories)
public interface BookRepository { }
public interface UserRepository { }
public interface LoanRepository { }
public interface ReservationRepository { }
public interface FineRepository { }

// Capa de Servicios (Lógica de Negocio)
public class LoanService { }
public class ReservationService { }
public class FineCalculationService { }
public class BookSearchService { }

// Capa de Infraestructura
public class LibraryNotificationService { }
public class AuditLogService { }
public class LibraryReportGenerator { }

// Ejemplo de implementación completa de una clase:
public class LoanService {
    private final LoanRepository loanRepository;
    private final BookRepository bookRepository;
    private final LoanValidator loanValidator;
    private final LibraryNotificationService notificationService;
    private final AuditLogService auditService;
    
    public LoanService(LoanRepository loanRepository,
                       BookRepository bookRepository,
                       LoanValidator loanValidator,
                       LibraryNotificationService notificationService,
                       AuditLogService auditService) {
        this.loanRepository = loanRepository;
        this.bookRepository = bookRepository;
        this.loanValidator = loanValidator;
        this.notificationService = notificationService;
        this.auditService = auditService;
    }
    
    public LoanResult borrowBook(User user, Book book) {
        // 1. Validar
        ValidationResult validation = loanValidator.canBorrow(user, book);
        if (!validation.isValid()) {
            return LoanResult.failure(validation.getErrors());
        }
        
        // 2. Verificar disponibilidad
        if (!bookRepository.isAvailable(book.getISBN())) {
            return LoanResult.failure("Book not available");
        }
        
        // 3. Crear préstamo
        Loan loan = new Loan(user.getId(), book.getISBN(), LocalDate.now());
        loanRepository.save(loan);
        
        // 4. Actualizar stock
        bookRepository.decrementAvailableCopies(book.getISBN());
        
        // 5. Notificar
        notificationService.sendLoanConfirmation(user, book, loan);
        
        // 6. Auditar
        auditService.log("LOAN_CREATED", user.getId(), book.getISBN());
        
        return LoanResult.success(loan);
    }
    
    // Más métodos...
}
```

---

## Rúbrica de Evaluación

### Nivel 1 (20 puntos)
- [ ] Identifica correctamente 5/5 responsabilidades (10 pts)
- [ ] Justifica adecuadamente técnicas de extracción (5 pts)
- [ ] Detecta correctamente 4/4 code smells (5 pts)

### Nivel 2 (30 puntos)
- [ ] Aplica los 5 pasos de refactorización (10 pts)
- [ ] Extrae al menos 5 clases con responsabilidades claras (10 pts)
- [ ] Tests cubren casos principales (>70% cobertura) (10 pts)

### Nivel 3 (25 puntos)
- [ ] Identifica mínimo 7 responsabilidades separadas (8 pts)
- [ ] Diseño tiene alta cohesión y bajo acoplamiento (10 pts)
- [ ] Implementa 3 clases completas con tests (7 pts)

### Nivel 4 (25 puntos)
- [ ] Sistema tiene >15 clases bien diseñadas (8 pts)
- [ ] Cada clase tiene UNA responsabilidad clara (10 pts)
- [ ] Tests con >80% cobertura (5 pts)
- [ ] Documentación justifica decisiones de diseño (2 pts)

**Total: 100 puntos**
