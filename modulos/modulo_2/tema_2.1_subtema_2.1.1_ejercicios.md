# Ejercicios: Definición y Fundamentos del SRP

## Objetivo
Practicar la identificación de responsabilidades en clases existentes y aplicar el "Test del Actor" para detectar violaciones del Single Responsibility Principle.

---

## ⭐ Nivel 1: Identificación de Responsabilidades

### Ejercicio 1.1: Análisis de Clase Report

Analiza la siguiente clase y responde las preguntas:

```java
public class Report {
    private String title;
    private List<String> data;
    private String author;
    
    public void generateReport() {
        // Lógica para generar el contenido del reporte
        System.out.println("Generating report: " + title);
    }
    
    public void saveToFile(String filename) {
        // Guardar reporte en archivo
        try (FileWriter writer = new FileWriter(filename)) {
            writer.write(title + "\n");
            for (String line : data) {
                writer.write(line + "\n");
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
    public void sendByEmail(String recipient) {
        // Enviar reporte por email
        EmailService service = new EmailService();
        service.send(recipient, "Report: " + title, generateContent());
    }
    
    public String formatAsHtml() {
        StringBuilder html = new StringBuilder();
        html.append("<html><body>");
        html.append("<h1>").append(title).append("</h1>");
        for (String line : data) {
            html.append("<p>").append(line).append("</p>");
        }
        html.append("</body></html>");
        return html.toString();
    }
    
    public String formatAsJson() {
        return "{\"title\":\"" + title + "\",\"data\":" + data + "}";
    }
    
    private String generateContent() {
        return String.join("\n", data);
    }
}
```

**Preguntas**:

1. ¿Cuántas responsabilidades identificas en esta clase?
2. ¿Qué actores del negocio podrían solicitar cambios en cada método?
3. ¿Qué métodos cambiarían si:
   - El diseñador pide cambiar el formato HTML?
   - El administrador de sistemas cambia el sistema de archivos?
   - El equipo de marketing pide agregar SMS además de email?

### Ejercicio 1.2: Test del Actor

Para la clase `Customer` siguiente, identifica qué actor solicita cada funcionalidad:

```java
public class Customer {
    private String name;
    private String email;
    private String phone;
    private List<Order> orders;
    private double creditLimit;
    
    // Método A
    public boolean canPurchase(double amount) {
        return getTotalDebt() + amount <= creditLimit;
    }
    
    // Método B
    public void saveToDatabase() {
        Database.execute("INSERT INTO customers...");
    }
    
    // Método C
    public void sendPromotionalEmail() {
        EmailService.send(email, "Special offers...");
    }
    
    // Método D
    public double calculateLoyaltyPoints() {
        return orders.stream()
                     .mapToDouble(Order::getTotal)
                     .sum() * 0.1;
    }
    
    // Método E
    public String generateInvoicePdf() {
        // Genera PDF con todas las órdenes
        return "PDF content...";
    }
    
    private double getTotalDebt() {
        return orders.stream()
                     .mapToDouble(Order::getTotal)
                     .sum();
    }
}
```

**Tabla a completar**:

| Método | Actor del Negocio | Razón del Cambio Potencial |
|--------|-------------------|----------------------------|
| `canPurchase` | | |
| `saveToDatabase` | | |
| `sendPromotionalEmail` | | |
| `calculateLoyaltyPoints` | | |
| `generateInvoicePdf` | | |

---

## ⭐⭐ Nivel 2: Refactorización Básica

### Ejercicio 2.1: Separar Responsabilidades en OrderProcessor

Refactoriza la siguiente clase aplicando SRP:

```java
public class OrderProcessor {
    private Database database;
    private EmailService emailService;
    
    public void processOrder(Order order) {
        // Validación
        if (order.getItems().isEmpty()) {
            throw new IllegalArgumentException("Order must have items");
        }
        
        if (order.getCustomer() == null) {
            throw new IllegalArgumentException("Order must have customer");
        }
        
        // Cálculo de total
        double subtotal = 0;
        for (OrderItem item : order.getItems()) {
            subtotal += item.getPrice() * item.getQuantity();
        }
        
        double tax = subtotal * 0.15;
        double total = subtotal + tax;
        order.setTotal(total);
        
        // Aplicar descuento
        if (order.getCustomer().isVip()) {
            double discount = total * 0.1;
            order.setDiscount(discount);
            order.setTotal(total - discount);
        }
        
        // Actualizar inventario
        for (OrderItem item : order.getItems()) {
            Product product = database.getProduct(item.getProductId());
            product.decreaseStock(item.getQuantity());
            database.updateProduct(product);
        }
        
        // Guardar orden
        database.saveOrder(order);
        
        // Enviar confirmación
        String message = "Your order #" + order.getId() + " has been confirmed. " +
                        "Total: $" + order.getTotal();
        emailService.send(order.getCustomer().getEmail(), "Order Confirmation", message);
        
        // Registrar en log
        System.out.println("Order processed: " + order.getId() + " at " + new Date());
    }
}
```

**Tareas**:

1. Identifica las responsabilidades presentes
2. Crea clases separadas para cada responsabilidad
3. Implementa tests unitarios para cada clase creada

**Criterios de éxito**:
- Mínimo 4 clases nuevas con responsabilidades únicas
- Cada clase debe ser testeable sin mocks de base de datos
- La clase `OrderProcessor` debe orquestar, no implementar

### Solución Modelo

<details>
<summary>Ver solución completa</summary>

```java
// Responsabilidad: Validar órdenes
public class OrderValidator {
    public void validate(Order order) {
        List<String> errors = new ArrayList<>();
        
        if (order.getItems() == null || order.getItems().isEmpty()) {
            errors.add("Order must have at least one item");
        }
        
        if (order.getCustomer() == null) {
            errors.add("Order must have a customer");
        }
        
        if (!errors.isEmpty()) {
            throw new ValidationException(errors);
        }
    }
}

// Responsabilidad: Calcular precios
public class OrderPricingService {
    private static final double TAX_RATE = 0.15;
    
    public OrderPrice calculatePrice(Order order) {
        double subtotal = order.getItems().stream()
            .mapToDouble(item -> item.getPrice() * item.getQuantity())
            .sum();
        
        double tax = subtotal * TAX_RATE;
        double total = subtotal + tax;
        
        return new OrderPrice(subtotal, tax, total);
    }
}

// Responsabilidad: Aplicar descuentos
public class DiscountCalculator {
    private static final double VIP_DISCOUNT_RATE = 0.1;
    
    public double calculateDiscount(Customer customer, double total) {
        if (customer.isVip()) {
            return total * VIP_DISCOUNT_RATE;
        }
        return 0.0;
    }
}

// Responsabilidad: Actualizar inventario
public class InventoryService {
    private final ProductRepository productRepository;
    
    public InventoryService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }
    
    public void decreaseStock(List<OrderItem> items) {
        for (OrderItem item : items) {
            Product product = productRepository.findById(item.getProductId());
            product.decreaseStock(item.getQuantity());
            productRepository.save(product);
        }
    }
}

// Responsabilidad: Persistir órdenes
public class OrderRepository {
    private final Database database;
    
    public OrderRepository(Database database) {
        this.database = database;
    }
    
    public void save(Order order) {
        database.saveOrder(order);
    }
}

// Responsabilidad: Notificar clientes
public class OrderNotificationService {
    private final EmailService emailService;
    
    public OrderNotificationService(EmailService emailService) {
        this.emailService = emailService;
    }
    
    public void sendConfirmation(Order order) {
        String message = String.format(
            "Your order #%s has been confirmed. Total: $%.2f",
            order.getId(),
            order.getTotal()
        );
        
        emailService.send(
            order.getCustomer().getEmail(),
            "Order Confirmation",
            message
        );
    }
}

// Responsabilidad: Auditoría
public class OrderAuditLogger {
    public void logProcessed(Order order) {
        System.out.println(String.format(
            "Order processed: %s at %s",
            order.getId(),
            LocalDateTime.now()
        ));
    }
}

// Orquestador (ahora solo coordina)
public class OrderProcessor {
    private final OrderValidator validator;
    private final OrderPricingService pricingService;
    private final DiscountCalculator discountCalculator;
    private final InventoryService inventoryService;
    private final OrderRepository orderRepository;
    private final OrderNotificationService notificationService;
    private final OrderAuditLogger auditLogger;
    
    public OrderProcessor(
        OrderValidator validator,
        OrderPricingService pricingService,
        DiscountCalculator discountCalculator,
        InventoryService inventoryService,
        OrderRepository orderRepository,
        OrderNotificationService notificationService,
        OrderAuditLogger auditLogger
    ) {
        this.validator = validator;
        this.pricingService = pricingService;
        this.discountCalculator = discountCalculator;
        this.inventoryService = inventoryService;
        this.orderRepository = orderRepository;
        this.notificationService = notificationService;
        this.auditLogger = auditLogger;
    }
    
    public void processOrder(Order order) {
        // 1. Validar
        validator.validate(order);
        
        // 2. Calcular precio
        OrderPrice price = pricingService.calculatePrice(order);
        order.setSubtotal(price.getSubtotal());
        order.setTax(price.getTax());
        order.setTotal(price.getTotal());
        
        // 3. Aplicar descuentos
        double discount = discountCalculator.calculateDiscount(
            order.getCustomer(),
            price.getTotal()
        );
        order.setDiscount(discount);
        order.setTotal(price.getTotal() - discount);
        
        // 4. Actualizar inventario
        inventoryService.decreaseStock(order.getItems());
        
        // 5. Guardar
        orderRepository.save(order);
        
        // 6. Notificar
        notificationService.sendConfirmation(order);
        
        // 7. Auditar
        auditLogger.logProcessed(order);
    }
}
```

**Tests Unitarios**:

```java
public class OrderValidatorTest {
    private OrderValidator validator;
    
    @BeforeEach
    void setUp() {
        validator = new OrderValidator();
    }
    
    @Test
    void shouldThrowExceptionWhenOrderHasNoItems() {
        Order order = new Order();
        order.setItems(Collections.emptyList());
        order.setCustomer(new Customer());
        
        assertThrows(ValidationException.class, () -> validator.validate(order));
    }
    
    @Test
    void shouldThrowExceptionWhenOrderHasNoCustomer() {
        Order order = new Order();
        order.setItems(List.of(new OrderItem()));
        order.setCustomer(null);
        
        assertThrows(ValidationException.class, () -> validator.validate(order));
    }
    
    @Test
    void shouldNotThrowExceptionWhenOrderIsValid() {
        Order order = new Order();
        order.setItems(List.of(new OrderItem()));
        order.setCustomer(new Customer());
        
        assertDoesNotThrow(() -> validator.validate(order));
    }
}

public class OrderPricingServiceTest {
    private OrderPricingService pricingService;
    
    @BeforeEach
    void setUp() {
        pricingService = new OrderPricingService();
    }
    
    @Test
    void shouldCalculateCorrectPrice() {
        Order order = new Order();
        OrderItem item1 = new OrderItem(1L, "Product1", 100.0, 2);
        OrderItem item2 = new OrderItem(2L, "Product2", 50.0, 1);
        order.setItems(List.of(item1, item2));
        
        OrderPrice price = pricingService.calculatePrice(order);
        
        assertEquals(250.0, price.getSubtotal()); // 200 + 50
        assertEquals(37.5, price.getTax());       // 250 * 0.15
        assertEquals(287.5, price.getTotal());    // 250 + 37.5
    }
}

public class DiscountCalculatorTest {
    private DiscountCalculator discountCalculator;
    
    @BeforeEach
    void setUp() {
        discountCalculator = new DiscountCalculator();
    }
    
    @Test
    void shouldApplyVipDiscount() {
        Customer vipCustomer = new Customer();
        vipCustomer.setVip(true);
        
        double discount = discountCalculator.calculateDiscount(vipCustomer, 100.0);
        
        assertEquals(10.0, discount); // 10% de 100
    }
    
    @Test
    void shouldNotApplyDiscountForNonVip() {
        Customer regularCustomer = new Customer();
        regularCustomer.setVip(false);
        
        double discount = discountCalculator.calculateDiscount(regularCustomer, 100.0);
        
        assertEquals(0.0, discount);
    }
}
```

**Comparación Antes/Después**:

| Métrica | Antes | Después |
|---------|-------|---------|
| Clases | 1 | 7 + 1 orquestador |
| Responsabilidades por clase | 7 | 1 cada una |
| Líneas por clase | ~80 | 10-25 |
| Tests sin mocks | Imposible | Todos excepto InventoryService y OrderRepository |
| Complejidad ciclomática | 15+ | 2-5 por clase |
| Reutilización | Baja | Alta (DiscountCalculator, OrderValidator) |

</details>

---

## ⭐⭐⭐ Nivel 3: Diseño desde Cero

### Ejercicio 3.1: Sistema de Gestión de Tareas

Diseña un sistema de gestión de tareas (TODO list) aplicando SRP desde el inicio.

**Requisitos funcionales**:

1. Crear, actualizar, eliminar tareas
2. Asignar tareas a usuarios
3. Establecer prioridades (Alta, Media, Baja)
4. Marcar tareas como completadas
5. Enviar recordatorios por email 24h antes del deadline
6. Generar reportes de productividad (tareas completadas por usuario)
7. Exportar tareas a CSV y JSON
8. Validar que las tareas tengan título y fecha válida

**Tareas**:

1. Identifica las responsabilidades necesarias
2. Diseña clases siguiendo SRP (mínimo 8 clases)
3. Crea un diagrama de clases UML
4. Implementa las clases principales con métodos stub
5. Escribe tests para al menos 3 clases

**Guía de diseño**:

Considera separar:
- Entidades de dominio (Task, User)
- Validación
- Persistencia
- Lógica de negocio (asignación, completado)
- Notificaciones
- Generación de reportes
- Exportación/Serialización

### Solución Modelo

<details>
<summary>Ver solución completa</summary>

```java
// ===== ENTIDADES DE DOMINIO =====

public class Task {
    private Long id;
    private String title;
    private String description;
    private LocalDateTime deadline;
    private Priority priority;
    private TaskStatus status;
    private User assignee;
    
    public enum Priority { HIGH, MEDIUM, LOW }
    public enum TaskStatus { PENDING, IN_PROGRESS, COMPLETED }
    
    // Constructores, getters, setters
}

public class User {
    private Long id;
    private String name;
    private String email;
    
    // Constructores, getters, setters
}

// ===== VALIDACIÓN =====

public class TaskValidator {
    public ValidationResult validate(Task task) {
        List<String> errors = new ArrayList<>();
        
        if (task.getTitle() == null || task.getTitle().trim().isEmpty()) {
            errors.add("Title is required");
        }
        
        if (task.getTitle() != null && task.getTitle().length() > 200) {
            errors.add("Title must be 200 characters or less");
        }
        
        if (task.getDeadline() != null && task.getDeadline().isBefore(LocalDateTime.now())) {
            errors.add("Deadline cannot be in the past");
        }
        
        return new ValidationResult(errors.isEmpty(), errors);
    }
}

// ===== REPOSITORIOS =====

public interface TaskRepository {
    void save(Task task);
    void update(Task task);
    void delete(Long taskId);
    Task findById(Long taskId);
    List<Task> findByAssignee(User user);
    List<Task> findByStatus(Task.TaskStatus status);
    List<Task> findTasksDueWithin(Duration duration);
}

public interface UserRepository {
    User findById(Long userId);
    User findByEmail(String email);
}

// ===== SERVICIOS DE DOMINIO =====

public class TaskAssignmentService {
    public void assign(Task task, User user) {
        if (task.getStatus() == Task.TaskStatus.COMPLETED) {
            throw new IllegalStateException("Cannot reassign completed task");
        }
        task.setAssignee(user);
    }
    
    public void unassign(Task task) {
        task.setAssignee(null);
    }
}

public class TaskCompletionService {
    public void complete(Task task) {
        if (task.getStatus() == Task.TaskStatus.COMPLETED) {
            throw new IllegalStateException("Task already completed");
        }
        task.setStatus(Task.TaskStatus.COMPLETED);
    }
    
    public void reopen(Task task) {
        if (task.getStatus() != Task.TaskStatus.COMPLETED) {
            throw new IllegalStateException("Task is not completed");
        }
        task.setStatus(Task.TaskStatus.PENDING);
    }
}

// ===== NOTIFICACIONES =====

public class TaskReminderService {
    private final TaskRepository taskRepository;
    private final EmailService emailService;
    
    public TaskReminderService(TaskRepository taskRepository, EmailService emailService) {
        this.taskRepository = taskRepository;
        this.emailService = emailService;
    }
    
    public void sendDueReminders() {
        List<Task> tasksDueSoon = taskRepository.findTasksDueWithin(Duration.ofHours(24));
        
        for (Task task : tasksDueSoon) {
            if (task.getAssignee() != null && task.getStatus() != Task.TaskStatus.COMPLETED) {
                sendReminder(task);
            }
        }
    }
    
    private void sendReminder(Task task) {
        String message = String.format(
            "Reminder: Task '%s' is due on %s",
            task.getTitle(),
            task.getDeadline()
        );
        
        emailService.send(
            task.getAssignee().getEmail(),
            "Task Reminder",
            message
        );
    }
}

// ===== REPORTES =====

public class ProductivityReportGenerator {
    private final TaskRepository taskRepository;
    
    public ProductivityReportGenerator(TaskRepository taskRepository) {
        this.taskRepository = taskRepository;
    }
    
    public ProductivityReport generateReport(User user, LocalDateTime from, LocalDateTime to) {
        List<Task> userTasks = taskRepository.findByAssignee(user);
        
        long completedTasks = userTasks.stream()
            .filter(t -> t.getStatus() == Task.TaskStatus.COMPLETED)
            .count();
        
        long pendingTasks = userTasks.stream()
            .filter(t -> t.getStatus() != Task.TaskStatus.COMPLETED)
            .count();
        
        return new ProductivityReport(user, completedTasks, pendingTasks, from, to);
    }
}

// ===== EXPORTACIÓN =====

public interface TaskExporter {
    String export(List<Task> tasks);
}

public class CsvTaskExporter implements TaskExporter {
    @Override
    public String export(List<Task> tasks) {
        StringBuilder csv = new StringBuilder();
        csv.append("ID,Title,Description,Deadline,Priority,Status,Assignee\n");
        
        for (Task task : tasks) {
            csv.append(String.format(
                "%d,\"%s\",\"%s\",%s,%s,%s,%s\n",
                task.getId(),
                escapeCsv(task.getTitle()),
                escapeCsv(task.getDescription()),
                task.getDeadline(),
                task.getPriority(),
                task.getStatus(),
                task.getAssignee() != null ? task.getAssignee().getName() : ""
            ));
        }
        
        return csv.toString();
    }
    
    private String escapeCsv(String value) {
        if (value == null) return "";
        return value.replace("\"", "\"\"");
    }
}

public class JsonTaskExporter implements TaskExporter {
    @Override
    public String export(List<Task> tasks) {
        StringBuilder json = new StringBuilder();
        json.append("[");
        
        for (int i = 0; i < tasks.size(); i++) {
            Task task = tasks.get(i);
            json.append(String.format(
                "{\"id\":%d,\"title\":\"%s\",\"deadline\":\"%s\",\"priority\":\"%s\",\"status\":\"%s\"}",
                task.getId(),
                escapeJson(task.getTitle()),
                task.getDeadline(),
                task.getPriority(),
                task.getStatus()
            ));
            
            if (i < tasks.size() - 1) {
                json.append(",");
            }
        }
        
        json.append("]");
        return json.toString();
    }
    
    private String escapeJson(String value) {
        if (value == null) return "";
        return value.replace("\"", "\\\"").replace("\n", "\\n");
    }
}
```

**Tests**:

```java
public class TaskValidatorTest {
    private TaskValidator validator;
    
    @BeforeEach
    void setUp() {
        validator = new TaskValidator();
    }
    
    @Test
    void shouldRejectTaskWithoutTitle() {
        Task task = new Task();
        task.setTitle("");
        
        ValidationResult result = validator.validate(task);
        
        assertFalse(result.isValid());
        assertTrue(result.getErrors().contains("Title is required"));
    }
    
    @Test
    void shouldRejectTaskWithPastDeadline() {
        Task task = new Task();
        task.setTitle("Valid Title");
        task.setDeadline(LocalDateTime.now().minusDays(1));
        
        ValidationResult result = validator.validate(task);
        
        assertFalse(result.isValid());
        assertTrue(result.getErrors().contains("Deadline cannot be in the past"));
    }
    
    @Test
    void shouldAcceptValidTask() {
        Task task = new Task();
        task.setTitle("Valid Title");
        task.setDeadline(LocalDateTime.now().plusDays(1));
        
        ValidationResult result = validator.validate(task);
        
        assertTrue(result.isValid());
        assertTrue(result.getErrors().isEmpty());
    }
}

public class TaskCompletionServiceTest {
    private TaskCompletionService service;
    
    @BeforeEach
    void setUp() {
        service = new TaskCompletionService();
    }
    
    @Test
    void shouldCompleteTask() {
        Task task = new Task();
        task.setStatus(Task.TaskStatus.PENDING);
        
        service.complete(task);
        
        assertEquals(Task.TaskStatus.COMPLETED, task.getStatus());
    }
    
    @Test
    void shouldThrowExceptionWhenCompletingAlreadyCompletedTask() {
        Task task = new Task();
        task.setStatus(Task.TaskStatus.COMPLETED);
        
        assertThrows(IllegalStateException.class, () -> service.complete(task));
    }
    
    @Test
    void shouldReopenCompletedTask() {
        Task task = new Task();
        task.setStatus(Task.TaskStatus.COMPLETED);
        
        service.reopen(task);
        
        assertEquals(Task.TaskStatus.PENDING, task.getStatus());
    }
}

public class CsvTaskExporterTest {
    private CsvTaskExporter exporter;
    
    @BeforeEach
    void setUp() {
        exporter = new CsvTaskExporter();
    }
    
    @Test
    void shouldExportTasksToCsv() {
        Task task1 = new Task();
        task1.setId(1L);
        task1.setTitle("Task 1");
        task1.setPriority(Task.Priority.HIGH);
        task1.setStatus(Task.TaskStatus.PENDING);
        
        Task task2 = new Task();
        task2.setId(2L);
        task2.setTitle("Task 2");
        task2.setPriority(Task.Priority.LOW);
        task2.setStatus(Task.TaskStatus.COMPLETED);
        
        String csv = exporter.export(List.of(task1, task2));
        
        assertTrue(csv.contains("ID,Title,Description"));
        assertTrue(csv.contains("1,\"Task 1\""));
        assertTrue(csv.contains("2,\"Task 2\""));
    }
}
```

**Diagrama de clases**:

```
┌─────────────┐         ┌──────────────────┐
│   Task      │◄────────│ TaskValidator    │
└─────────────┘         └──────────────────┘
       │
       │ uses
       ▼
┌─────────────┐         ┌──────────────────────────┐
│   User      │◄────────│ TaskAssignmentService    │
└─────────────┘         └──────────────────────────┘
                                    │
                        ┌───────────┴───────────┐
                        │                       │
             ┌──────────────────┐   ┌──────────────────────┐
             │ TaskRepository   │   │ UserRepository       │
             └──────────────────┘   └──────────────────────┘
                        │
                        │ uses
                        ▼
             ┌──────────────────────────┐
             │ TaskReminderService      │──────► EmailService
             └──────────────────────────┘
                        │
                        │
             ┌──────────────────────────┐
             │ ProductivityReportGen    │
             └──────────────────────────┘
                        │
                        │
             ┌──────────────────────────┐
             │ TaskExporter (interface) │
             └──────────────────────────┘
                        △
                        │
              ┌─────────┴─────────┐
              │                   │
    ┌─────────────────┐  ┌────────────────┐
    │ CsvTaskExporter │  │ JsonTaskExporter│
    └─────────────────┘  └────────────────┘
```

**Responsabilidades identificadas**:

1. **Task, User**: Entidades de dominio (datos)
2. **TaskValidator**: Validación de reglas de negocio
3. **TaskRepository, UserRepository**: Persistencia
4. **TaskAssignmentService**: Lógica de asignación
5. **TaskCompletionService**: Lógica de completado
6. **TaskReminderService**: Envío de recordatorios
7. **ProductivityReportGenerator**: Generación de reportes
8. **CsvTaskExporter, JsonTaskExporter**: Exportación en diferentes formatos

**Cada clase tiene una única razón para cambiar** ✅

</details>

---

## ⭐⭐⭐⭐ Nivel 4: Análisis de Arquitectura Real

### Ejercicio 4.1: Auditoría de Código Legacy

Se te proporciona un fragmento de un sistema de comercio electrónico legacy:

```java
public class ShoppingCartManager {
    private Map<String, List<Product>> userCarts = new HashMap<>();
    private Database database;
    private PaymentGateway paymentGateway;
    private EmailService emailService;
    private TaxCalculator taxCalculator;
    private ShippingCalculator shippingCalculator;
    
    public void addProductToCart(String userId, Product product, int quantity) {
        // Validar producto
        if (product.getStock() < quantity) {
            throw new IllegalArgumentException("Insufficient stock");
        }
        
        // Validar usuario
        User user = database.findUserById(userId);
        if (user == null) {
            throw new IllegalArgumentException("User not found");
        }
        
        // Agregar al carrito
        List<Product> cart = userCarts.getOrDefault(userId, new ArrayList<>());
        for (int i = 0; i < quantity; i++) {
            cart.add(product);
        }
        userCarts.put(userId, cart);
        
        // Log
        System.out.println("Product added to cart: " + product.getName() + " x" + quantity);
    }
    
    public double calculateCartTotal(String userId) {
        List<Product> cart = userCarts.get(userId);
        if (cart == null) return 0.0;
        
        double subtotal = cart.stream()
            .mapToDouble(Product::getPrice)
            .sum();
        
        User user = database.findUserById(userId);
        double tax = taxCalculator.calculate(subtotal, user.getState());
        double shipping = shippingCalculator.calculate(cart.size(), user.getAddress());
        
        // Aplicar cupón si existe
        if (user.getCoupon() != null) {
            double discount = subtotal * user.getCoupon().getDiscountRate();
            subtotal -= discount;
        }
        
        return subtotal + tax + shipping;
    }
    
    public void checkout(String userId, String paymentMethod) {
        List<Product> cart = userCarts.get(userId);
        User user = database.findUserById(userId);
        
        // Validar carrito
        if (cart == null || cart.isEmpty()) {
            throw new IllegalStateException("Cart is empty");
        }
        
        // Calcular total
        double total = calculateCartTotal(userId);
        
        // Procesar pago
        PaymentResult result = paymentGateway.charge(user.getPaymentInfo(), total, paymentMethod);
        if (!result.isSuccessful()) {
            throw new PaymentFailedException("Payment failed: " + result.getErrorMessage());
        }
        
        // Crear orden
        Order order = new Order();
        order.setUserId(userId);
        order.setProducts(cart);
        order.setTotal(total);
        order.setPaymentId(result.getTransactionId());
        database.saveOrder(order);
        
        // Actualizar stock
        for (Product product : cart) {
            product.decreaseStock(1);
            database.updateProduct(product);
        }
        
        // Limpiar carrito
        userCarts.remove(userId);
        
        // Enviar confirmación
        String message = "Your order #" + order.getId() + " has been confirmed.";
        emailService.send(user.getEmail(), "Order Confirmation", message);
        
        // Enviar factura
        String invoice = generateInvoice(order);
        emailService.send(user.getEmail(), "Invoice", invoice);
        
        // Registrar analytics
        System.out.println("Order completed: " + order.getId() + " - Total: $" + total);
    }
    
    private String generateInvoice(Order order) {
        StringBuilder invoice = new StringBuilder();
        invoice.append("INVOICE\n");
        invoice.append("Order ID: ").append(order.getId()).append("\n");
        invoice.append("Total: $").append(order.getTotal()).append("\n");
        // ... más lógica de formateo
        return invoice.toString();
    }
    
    public Map<String, Integer> getCartSummary(String userId) {
        List<Product> cart = userCarts.get(userId);
        if (cart == null) return Collections.emptyMap();
        
        Map<String, Integer> summary = new HashMap<>();
        for (Product product : cart) {
            summary.put(product.getName(), summary.getOrDefault(product.getName(), 0) + 1);
        }
        return summary;
    }
}
```

**Tareas**:

1. **Análisis de violaciones**:
   - Identifica todas las responsabilidades en `ShoppingCartManager`
   - Para cada responsabilidad, identifica el actor del negocio
   - Calcula el número de razones para cambiar
   - Identifica code smells relacionados con SRP

2. **Propuesta de refactorización**:
   - Diseña una arquitectura con clases que cumplan SRP
   - Crea un plan de refactorización incremental (no Big Bang)
   - Identifica qué partes se pueden testear sin cambios

3. **Implementación**:
   - Implementa al menos 5 clases extraídas
   - Escribe tests para esas clases
   - Demuestra que el código refactorizado es más testeable

**Criterios de evaluación**:

| Criterio | Puntos |
|----------|--------|
| Identificación completa de responsabilidades (mínimo 10) | 20 |
| Diseño de arquitectura con SRP | 25 |
| Plan de refactorización incremental | 20 |
| Implementación de clases extraídas | 20 |
| Tests unitarios (cobertura > 80%) | 15 |

**Rúbrica detallada**:

- **Excelente (90-100)**: 10+ responsabilidades identificadas, arquitectura con 8+ clases cohesivas, plan incremental detallado, implementación completa con tests
- **Bueno (75-89)**: 7-9 responsabilidades, arquitectura con 5-7 clases, plan básico, implementación parcial
- **Suficiente (60-74)**: 5-6 responsabilidades, arquitectura con 3-4 clases, implementación mínima
- **Insuficiente (<60)**: < 5 responsabilidades identificadas, arquitectura confusa

---

## Resumen de Ejercicios

| Nivel | Habilidad Desarrollada | Tiempo Estimado |
|-------|------------------------|-----------------|
| ⭐ | Identificar responsabilidades y aplicar Test del Actor | 30-45 min |
| ⭐⭐ | Refactorizar código existente aplicando SRP | 60-90 min |
| ⭐⭐⭐ | Diseñar sistemas desde cero con SRP | 120-180 min |
| ⭐⭐⭐⭐ | Auditar y refactorizar código legacy complejo | 180-240 min |

## Recursos Adicionales

- **Herramientas de análisis**: SonarQube (LCOM metric), IntelliJ IDEA (Analyze Dependencies)
- **Lecturas**: "Clean Code" Cap. 10, "Agile Software Development" Cap. 8
- **Videos**: "The Single Responsibility Principle" - Uncle Bob Martin

---

**Próximo paso**: Una vez completados estos ejercicios, continúa con el subtema **2.1.2: Identificación de Responsabilidades**, donde profundizaremos en técnicas sistemáticas para detectar múltiples responsabilidades en código existente.
