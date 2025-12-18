# Ejercicios: Definición y Fundamentos del SRP

## BANCO DE EJERCICIOS GRADUADOS

### ⭐ NIVEL 1: Identificación de Violaciones

#### Ejercicio 1.1: Detectar Responsabilidades Múltiples

**Enunciado:**
Analiza la siguiente clase e identifica todas las responsabilidades que tiene. Para cada responsabilidad, indica qué actor/stakeholder podría requerir cambios.

```java
public class Invoice {
    private String invoiceNumber;
    private Customer customer;
    private List<LineItem> items;
    private LocalDate date;
    
    public double calculateTotal() {
        return items.stream()
            .mapToDouble(item -> item.getPrice() * item.getQuantity())
            .sum();
    }
    
    public double calculateTax() {
        double subtotal = calculateTotal();
        return subtotal * 0.16; // 16% IVA
    }
    
    public void saveToDatabase() {
        Connection conn = DatabaseConnection.getInstance();
        String sql = "INSERT INTO invoices (number, customer_id, total, date) VALUES (?, ?, ?, ?)";
        PreparedStatement stmt = conn.prepareStatement(sql);
        stmt.setString(1, invoiceNumber);
        stmt.setInt(2, customer.getId());
        stmt.setDouble(3, calculateTotal());
        stmt.setDate(4, Date.valueOf(date));
        stmt.executeUpdate();
    }
    
    public String generateHTML() {
        StringBuilder html = new StringBuilder();
        html.append("<html><body>");
        html.append("<h1>Invoice #").append(invoiceNumber).append("</h1>");
        html.append("<p>Customer: ").append(customer.getName()).append("</p>");
        html.append("<table>");
        for (LineItem item : items) {
            html.append("<tr><td>").append(item.getName()).append("</td>");
            html.append("<td>").append(item.getPrice()).append("</td></tr>");
        }
        html.append("</table>");
        html.append("<p>Total: $").append(calculateTotal()).append("</p>");
        html.append("</body></html>");
        return html.toString();
    }
    
    public void sendByEmail() {
        EmailService emailService = new EmailService();
        String htmlContent = generateHTML();
        emailService.send(customer.getEmail(), "Your Invoice", htmlContent);
    }
    
    public void print() {
        PrinterService printer = new PrinterService();
        String content = generateHTML();
        printer.print(content);
    }
    
    public boolean isOverdue() {
        return LocalDate.now().isAfter(date.plusDays(30));
    }
    
    public String toJSON() {
        return String.format("{\"invoiceNumber\":\"%s\",\"total\":%.2f}", 
                             invoiceNumber, calculateTotal());
    }
}
```

**Solución Modelo:**

**Responsabilidades identificadas:**

1. **Cálculos de negocio** (CFO/Contador)
   - `calculateTotal()`
   - `calculateTax()`
   - `isOverdue()`
   - Actor: CFO, Contador - Pueden cambiar reglas de cálculo de impuestos, totales, políticas de vencimiento

2. **Persistencia de datos** (DBA/Equipo de Infraestructura)
   - `saveToDatabase()`
   - Actor: DBA - Puede cambiar esquema de base de datos, tipo de BD, estrategia de persistencia

3. **Generación de HTML** (Equipo de UX/Frontend)
   - `generateHTML()`
   - Actor: Diseñadores/UX - Pueden cambiar diseño, estructura HTML, estilos

4. **Notificaciones** (Equipo de Comunicaciones)
   - `sendByEmail()`
   - Actor: Marketing/Comunicaciones - Pueden cambiar canal de notificación, contenido, timing

5. **Impresión** (Equipo de Operaciones)
   - `print()`
   - Actor: Operaciones - Pueden cambiar formato de impresión, impresoras, configuración

6. **Serialización** (Equipo de Integración/API)
   - `toJSON()`
   - Actor: Equipo de API - Pueden cambiar formato de intercambio (JSON, XML, protobuf)

**Total: 6 responsabilidades distintas = 6 razones para cambiar**

**Refactorización propuesta:**

```java
// ✅ Modelo de dominio (solo datos y lógica de negocio esencial)
public class Invoice {
    private final String invoiceNumber;
    private final Customer customer;
    private final List<LineItem> items;
    private final LocalDate date;
    
    public Invoice(String invoiceNumber, Customer customer, List<LineItem> items, LocalDate date) {
        this.invoiceNumber = invoiceNumber;
        this.customer = customer;
        this.items = new ArrayList<>(items);
        this.date = date;
    }
    
    public double calculateTotal() {
        return items.stream()
            .mapToDouble(item -> item.getPrice() * item.getQuantity())
            .sum();
    }
    
    public boolean isOverdue() {
        return LocalDate.now().isAfter(date.plusDays(30));
    }
    
    // Getters
    public String getInvoiceNumber() { return invoiceNumber; }
    public Customer getCustomer() { return customer; }
    public List<LineItem> getItems() { return new ArrayList<>(items); }
    public LocalDate getDate() { return date; }
}

// ✅ Responsabilidad: Cálculos fiscales
public class TaxCalculator {
    private static final double VAT_RATE = 0.16;
    
    public double calculateTax(Invoice invoice) {
        return invoice.calculateTotal() * VAT_RATE;
    }
}

// ✅ Responsabilidad: Persistencia
public class InvoiceRepository {
    private final DataSource dataSource;
    
    public void save(Invoice invoice) {
        // Lógica de guardado
    }
}

// ✅ Responsabilidad: Generación HTML
public class InvoiceHTMLGenerator {
    public String generate(Invoice invoice) {
        // Lógica de generación HTML
    }
}

// ✅ Responsabilidad: Notificaciones
public class InvoiceEmailService {
    private final EmailService emailService;
    private final InvoiceHTMLGenerator htmlGenerator;
    
    public void send(Invoice invoice) {
        String html = htmlGenerator.generate(invoice);
        emailService.send(invoice.getCustomer().getEmail(), "Your Invoice", html);
    }
}

// ✅ Responsabilidad: Impresión
public class InvoicePrinter {
    private final PrinterService printerService;
    
    public void print(Invoice invoice) {
        // Lógica de impresión
    }
}

// ✅ Responsabilidad: Serialización
public class InvoiceSerializer {
    public String toJSON(Invoice invoice) {
        // Lógica de serialización
    }
}
```

**Rúbrica:**
- Identificación de responsabilidades (50%): ¿Encontró todas?
- Identificación de actores (30%): ¿Identificó quién pediría cada cambio?
- Propuesta de refactorización (20%): ¿Sugirió separación apropiada?

---

#### Ejercicio 1.2: ¿Viola SRP o No?

**Enunciado:**
Para cada clase, determina si viola SRP. Justifica tu respuesta.

```java
// Clase A
public class EmailValidator {
    public boolean isValid(String email) {
        return email.matches("^[A-Za-z0-9+_.-]+@(.+)$");
    }
    
    public String extractDomain(String email) {
        return email.substring(email.indexOf('@') + 1);
    }
}

// Clase B
public class UserService {
    public void registerUser(User user) { }
    public void updateProfile(User user) { }
    public void deleteUser(String userId) { }
    public void sendPasswordReset(String email) { }
    public List<User> searchUsers(String query) { }
}

// Clase C
public class Order {
    private List<OrderItem> items;
    
    public void addItem(OrderItem item) {
        items.add(item);
    }
    
    public void removeItem(OrderItem item) {
        items.remove(item);
    }
    
    public double getTotal() {
        return items.stream().mapToDouble(OrderItem::getPrice).sum();
    }
}

// Clase D
public class ReportGenerator {
    public Report generate(ReportType type, Date from, Date to) {
        Data data = fetchData(from, to);
        return switch(type) {
            case PDF -> generatePDF(data);
            case EXCEL -> generateExcel(data);
            case HTML -> generateHTML(data);
        };
    }
}
```

**Solución Modelo:**

**Clase A: EmailValidator** - ✅ **NO viola SRP**
- Responsabilidad única: Validación y manipulación de emails
- Ambos métodos están cohesivos (trabajan con el mismo concepto)
- Cambiarían por la misma razón (cambio en reglas de validación de email)

**Clase B: UserService** - ❌ **SÍ viola SRP**
- Múltiples responsabilidades:
  1. Gestión de usuarios (register, update, delete)
  2. Autenticación (sendPasswordReset)
  3. Búsqueda (searchUsers)
- Debería separarse en: `UserRepository`, `UserAuthenticationService`, `UserSearchService`

**Clase C: Order** - ✅ **NO viola SRP**
- Responsabilidad única: Gestionar el estado del pedido
- Todos los métodos son lógica de dominio del pedido
- Son operaciones cohesivas del mismo concepto de negocio

**Clase D: ReportGenerator** - ❌ **SÍ viola SRP**
- Múltiples responsabilidades (generación en diferentes formatos)
- Cada formato es una razón para cambiar
- Debería usar Strategy Pattern: `PDFReportGenerator`, `ExcelReportGenerator`, `HTMLReportGenerator`

---

### ⭐⭐ NIVEL 2: Refactorización Guiada

#### Ejercicio 2.1: Refactorizar Sistema de Biblioteca

**Enunciado:**
Refactoriza la siguiente clase que gestiona libros en una biblioteca para cumplir con SRP:

```java
public class Book {
    private String isbn;
    private String title;
    private String author;
    private boolean available;
    private LocalDate dueDate;
    
    public boolean isAvailable() {
        return available;
    }
    
    public void checkout(String userId) {
        if (!available) {
            throw new IllegalStateException("Book not available");
        }
        available = false;
        dueDate = LocalDate.now().plusDays(14);
        saveToDatabase();
        sendCheckoutEmail(userId);
    }
    
    public void returnBook() {
        available = true;
        dueDate = null;
        saveToDatabase();
        if (hasWaitingList()) {
            notifyNextInQueue();
        }
    }
    
    private void saveToDatabase() {
        Connection conn = DatabaseConnection.get();
        PreparedStatement stmt = conn.prepareStatement(
            "UPDATE books SET available=?, due_date=? WHERE isbn=?"
        );
        stmt.setBoolean(1, available);
        stmt.setDate(2, dueDate != null ? Date.valueOf(dueDate) : null);
        stmt.setString(3, isbn);
        stmt.executeUpdate();
    }
    
    private void sendCheckoutEmail(String userId) {
        EmailService service = new EmailService();
        User user = findUser(userId);
        service.send(user.getEmail(), "Book Checked Out", 
                     "You checked out: " + title);
    }
    
    private boolean hasWaitingList() {
        Connection conn = DatabaseConnection.get();
        PreparedStatement stmt = conn.prepareStatement(
            "SELECT COUNT(*) FROM waiting_list WHERE isbn=?"
        );
        stmt.setString(1, isbn);
        ResultSet rs = stmt.executeQuery();
        return rs.next() && rs.getInt(1) > 0;
    }
    
    private void notifyNextInQueue() {
        // Lógica para notificar siguiente en lista de espera
    }
    
    public String generateBarcode() {
        return "BARCODE-" + isbn;
    }
    
    public String toJSON() {
        return String.format("{\"isbn\":\"%s\",\"title\":\"%s\",\"available\":%b}",
                             isbn, title, available);
    }
}
```

**Solución Modelo:**

```java
// ✅ Modelo de dominio puro
public class Book {
    private final String isbn;
    private final String title;
    private final String author;
    private BookStatus status;
    private LocalDate dueDate;
    
    public Book(String isbn, String title, String author) {
        this.isbn = isbn;
        this.title = title;
        this.author = author;
        this.status = BookStatus.AVAILABLE;
    }
    
    public void markAsCheckedOut(LocalDate dueDate) {
        if (status != BookStatus.AVAILABLE) {
            throw new IllegalStateException("Book is not available");
        }
        this.status = BookStatus.CHECKED_OUT;
        this.dueDate = dueDate;
    }
    
    public void markAsReturned() {
        this.status = BookStatus.AVAILABLE;
        this.dueDate = null;
    }
    
    public boolean isAvailable() {
        return status == BookStatus.AVAILABLE;
    }
    
    // Getters
    public String getIsbn() { return isbn; }
    public String getTitle() { return title; }
    public String getAuthor() { return author; }
    public BookStatus getStatus() { return status; }
    public LocalDate getDueDate() { return dueDate; }
}

public enum BookStatus {
    AVAILABLE, CHECKED_OUT, RESERVED, DAMAGED
}

// ✅ Responsabilidad: Persistencia
public class BookRepository {
    private final DataSource dataSource;
    
    public BookRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    public void save(Book book) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(
                 "UPDATE books SET status=?, due_date=? WHERE isbn=?")) {
            
            stmt.setString(1, book.getStatus().name());
            stmt.setDate(2, book.getDueDate() != null ? Date.valueOf(book.getDueDate()) : null);
            stmt.setString(3, book.getIsbn());
            stmt.executeUpdate();
            
        } catch (SQLException e) {
            throw new PersistenceException("Failed to save book", e);
        }
    }
    
    public Book findByIsbn(String isbn) {
        // Lógica de búsqueda
        return null;
    }
}

// ✅ Responsabilidad: Gestión de préstamos
public class CheckoutService {
    private static final int LOAN_PERIOD_DAYS = 14;
    private final BookRepository bookRepository;
    
    public CheckoutService(BookRepository bookRepository) {
        this.bookRepository = bookRepository;
    }
    
    public void checkout(Book book, String userId) {
        if (!book.isAvailable()) {
            throw new IllegalStateException("Book not available");
        }
        
        LocalDate dueDate = LocalDate.now().plusDays(LOAN_PERIOD_DAYS);
        book.markAsCheckedOut(dueDate);
        bookRepository.save(book);
    }
    
    public void returnBook(Book book) {
        book.markAsReturned();
        bookRepository.save(book);
    }
}

// ✅ Responsabilidad: Lista de espera
public class WaitingListService {
    private final DataSource dataSource;
    
    public boolean hasWaitingList(String isbn) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(
                 "SELECT COUNT(*) FROM waiting_list WHERE isbn=?")) {
            
            stmt.setString(1, isbn);
            ResultSet rs = stmt.executeQuery();
            return rs.next() && rs.getInt(1) > 0;
            
        } catch (SQLException e) {
            throw new PersistenceException("Failed to check waiting list", e);
        }
    }
    
    public String getNextInQueue(String isbn) {
        // Retorna userId del siguiente en lista
        return null;
    }
    
    public void removeFromQueue(String isbn, String userId) {
        // Elimina usuario de la lista de espera
    }
}

// ✅ Responsabilidad: Notificaciones
public class BookNotificationService {
    private final EmailService emailService;
    private final UserRepository userRepository;
    
    public BookNotificationService(EmailService emailService, UserRepository userRepository) {
        this.emailService = emailService;
        this.userRepository = userRepository;
    }
    
    public void sendCheckoutNotification(Book book, String userId) {
        User user = userRepository.findById(userId);
        emailService.send(
            user.getEmail(),
            "Book Checked Out",
            String.format("You checked out: %s by %s", book.getTitle(), book.getAuthor())
        );
    }
    
    public void sendReturnNotification(Book book, String userId) {
        User user = userRepository.findById(userId);
        emailService.send(
            user.getEmail(),
            "Book Returned",
            String.format("Thank you for returning: %s", book.getTitle())
        );
    }
    
    public void sendAvailabilityNotification(Book book, String userId) {
        User user = userRepository.findById(userId);
        emailService.send(
            user.getEmail(),
            "Book Available",
            String.format("The book '%s' is now available!", book.getTitle())
        );
    }
}

// ✅ Responsabilidad: Generación de códigos de barras
public class BarcodeGenerator {
    public String generateBarcode(String isbn) {
        return "BARCODE-" + isbn;
    }
}

// ✅ Responsabilidad: Serialización
public class BookSerializer {
    public String toJSON(Book book) {
        return String.format(
            "{\"isbn\":\"%s\",\"title\":\"%s\",\"author\":\"%s\",\"available\":%b}",
            book.getIsbn(),
            book.getTitle(),
            book.getAuthor(),
            book.isAvailable()
        );
    }
}

// ✅ Orquestación (Application Service)
public class LibraryService {
    private final CheckoutService checkoutService;
    private final BookRepository bookRepository;
    private final BookNotificationService notificationService;
    private final WaitingListService waitingListService;
    
    public LibraryService(
            CheckoutService checkoutService,
            BookRepository bookRepository,
            BookNotificationService notificationService,
            WaitingListService waitingListService) {
        this.checkoutService = checkoutService;
        this.bookRepository = bookRepository;
        this.notificationService = notificationService;
        this.waitingListService = waitingListService;
    }
    
    public void checkoutBook(String isbn, String userId) {
        Book book = bookRepository.findByIsbn(isbn);
        checkoutService.checkout(book, userId);
        notificationService.sendCheckoutNotification(book, userId);
    }
    
    public void returnBook(String isbn, String userId) {
        Book book = bookRepository.findByIsbn(isbn);
        checkoutService.returnBook(book);
        notificationService.sendReturnNotification(book, userId);
        
        // Si hay lista de espera, notificar
        if (waitingListService.hasWaitingList(isbn)) {
            String nextUserId = waitingListService.getNextInQueue(isbn);
            notificationService.sendAvailabilityNotification(book, nextUserId);
            waitingListService.removeFromQueue(isbn, nextUserId);
        }
    }
}
```

**Tests:**

```java
class CheckoutServiceTest {
    
    @Test
    void checkoutShouldMarkBookAsCheckedOut() {
        BookRepository mockRepo = mock(BookRepository.class);
        CheckoutService service = new CheckoutService(mockRepo);
        Book book = new Book("123", "Clean Code", "Martin");
        
        service.checkout(book, "user123");
        
        assertFalse(book.isAvailable());
        assertEquals(BookStatus.CHECKED_OUT, book.getStatus());
        assertNotNull(book.getDueDate());
        verify(mockRepo).save(book);
    }
    
    @Test
    void checkoutUnavailableBookShouldThrowException() {
        BookRepository mockRepo = mock(BookRepository.class);
        CheckoutService service = new CheckoutService(mockRepo);
        Book book = new Book("123", "Clean Code", "Martin");
        book.markAsCheckedOut(LocalDate.now().plusDays(14));
        
        assertThrows(IllegalStateException.class, () -> {
            service.checkout(book, "user123");
        });
    }
}
```

**Rúbrica:**
- Identificación de responsabilidades (20%)
- Separación en clases cohesivas (40%)
- Nombres apropiados (15%)
- Tests (25%)

---

### ⭐⭐⭐ NIVEL 3: Diseño desde Cero

#### Ejercicio 3.1: Sistema de Gestión de Pedidos

**Enunciado:**
Diseña desde cero un sistema de gestión de pedidos para un e-commerce que cumpla con SRP. El sistema debe manejar:

1. Creación y validación de pedidos
2. Cálculo de precios (con descuentos, impuestos, envío)
3. Gestión de inventario
4. Procesamiento de pagos
5. Notificaciones (confirmación, envío, entrega)
6. Generación de facturas
7. Tracking de envíos

**Requisitos:**
- Cada clase debe tener una única responsabilidad
- Incluir tests para cada clase
- Diagramas UML opcionales
- Implementación de al menos 2 casos de uso completos

**Solución Modelo (Parcial):**

```java
// === MODELO DE DOMINIO ===

public class Order {
    private final String orderId;
    private final String customerId;
    private final List<OrderLine> lines;
    private OrderStatus status;
    private LocalDateTime createdAt;
    
    public Order(String orderId, String customerId, List<OrderLine> lines) {
        this.orderId = orderId;
        this.customerId = customerId;
        this.lines = new ArrayList<>(lines);
        this.status = OrderStatus.PENDING;
        this.createdAt = LocalDateTime.now();
    }
    
    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Can only confirm pending orders");
        }
        this.status = OrderStatus.CONFIRMED;
    }
    
    public double getSubtotal() {
        return lines.stream()
            .mapToDouble(OrderLine::getTotal)
            .sum();
    }
    
    // Getters
}

public class OrderLine {
    private final Product product;
    private final int quantity;
    private final double unitPrice;
    
    public double getTotal() {
        return quantity * unitPrice;
    }
}

// === VALIDACIÓN ===

public class OrderValidator {
    private final InventoryService inventoryService;
    
    public ValidationResult validate(Order order) {
        ValidationResult result = new ValidationResult();
        
        if (order.getLines().isEmpty()) {
            result.addError("Order must have at least one item");
        }
        
        for (OrderLine line : order.getLines()) {
            if (!inventoryService.hasStock(line.getProduct().getSku(), line.getQuantity())) {
                result.addError("Insufficient stock for " + line.getProduct().getName());
            }
        }
        
        return result;
    }
}

// === CÁLCULO DE PRECIOS ===

public class PriceCalculator {
    public OrderPrice calculate(Order order, DiscountRules discounts, TaxRules taxes) {
        double subtotal = order.getSubtotal();
        double discount = discounts.calculateDiscount(order);
        double tax = taxes.calculateTax(subtotal - discount);
        double shipping = calculateShipping(order);
        
        return new OrderPrice(subtotal, discount, tax, shipping);
    }
    
    private double calculateShipping(Order order) {
        // Lógica de cálculo de envío
        return 10.0;
    }
}

// === INVENTARIO ===

public class InventoryService {
    private final InventoryRepository repository;
    
    public void reserve(Order order) {
        for (OrderLine line : order.getLines()) {
            repository.reserveStock(line.getProduct().getSku(), line.getQuantity());
        }
    }
    
    public void release(Order order) {
        for (OrderLine line : order.getLines()) {
            repository.releaseStock(line.getProduct().getSku(), line.getQuantity());
        }
    }
    
    public boolean hasStock(String sku, int quantity) {
        return repository.getAvailableStock(sku) >= quantity;
    }
}

// === PROCESAMIENTO DE PAGOS ===

public class PaymentProcessor {
    private final PaymentGateway gateway;
    
    public PaymentResult processPayment(Order order, OrderPrice price, PaymentMethod method) {
        try {
            String transactionId = gateway.charge(price.getTotal(), method);
            return PaymentResult.success(transactionId);
        } catch (PaymentException e) {
            return PaymentResult.failure(e.getMessage());
        }
    }
}

// === NOTIFICACIONES ===

public class OrderNotificationService {
    private final EmailService emailService;
    private final CustomerRepository customerRepository;
    
    public void sendOrderConfirmation(Order order) {
        Customer customer = customerRepository.findById(order.getCustomerId());
        emailService.send(
            customer.getEmail(),
            "Order Confirmation",
            buildConfirmationEmail(order)
        );
    }
    
    public void sendShippingNotification(Order order, String trackingNumber) {
        Customer customer = customerRepository.findById(order.getCustomerId());
        emailService.send(
            customer.getEmail(),
            "Your order has shipped!",
            buildShippingEmail(order, trackingNumber)
        );
    }
}

// === CASO DE USO: Procesar Pedido ===

public class OrderProcessingService {
    private final OrderValidator validator;
    private final PriceCalculator priceCalculator;
    private final InventoryService inventoryService;
    private final PaymentProcessor paymentProcessor;
    private final OrderRepository orderRepository;
    private final OrderNotificationService notificationService;
    
    @Transactional
    public ProcessingResult processOrder(Order order, PaymentMethod paymentMethod) {
        // 1. Validar
        ValidationResult validation = validator.validate(order);
        if (!validation.isValid()) {
            return ProcessingResult.validationFailed(validation.getErrors());
        }
        
        // 2. Calcular precio
        OrderPrice price = priceCalculator.calculate(order, 
                                                      DiscountRules.standard(),
                                                      TaxRules.forRegion(order.getShippingAddress()));
        
        // 3. Reservar inventario
        inventoryService.reserve(order);
        
        // 4. Procesar pago
        PaymentResult payment = paymentProcessor.processPayment(order, price, paymentMethod);
        if (!payment.isSuccessful()) {
            inventoryService.release(order);
            return ProcessingResult.paymentFailed(payment.getErrorMessage());
        }
        
        // 5. Confirmar pedido
        order.confirm();
        orderRepository.save(order);
        
        // 6. Notificar
        notificationService.sendOrderConfirmation(order);
        
        return ProcessingResult.success(order.getOrderId());
    }
}
```

**Rúbrica:**
- Diseño (30%): Separación clara de responsabilidades
- Implementación (30%): Código funcional y limpio
- Tests (25%): Cobertura adecuada
- Documentación (15%): Comentarios, diagramas, README

---

### ⭐⭐⭐⭐ NIVEL 4: Proyecto Arquitectónico

#### Ejercicio 4.1: Sistema de Gestión Hospitalaria

Diseña e implementa un sistema hospitalario completo con:
- Gestión de pacientes y expedientes médicos
- Sistema de citas
- Gestión de medicamentos e inventario
- Facturación y seguros
- Reportes médicos y estadísticas
- Notificaciones y recordatorios
- Integración con laboratorios externos

Debe incluir al menos 30 clases bien diseñadas siguiendo SRP, tests completos, y documentación arquitectónica.

---

## CRITERIOS GENERALES DE EVALUACIÓN

| Criterio | Peso | Descripción |
|----------|------|-------------|
| **Identificación de Responsabilidades** | 25% | Detecta correctamente múltiples responsabilidades |
| **Separación Apropiada** | 30% | Divide en clases cohesivas |
| **Nombres Descriptivos** | 15% | Nombres que reflejan responsabilidad única |
| **Tests** | 20% | Tests enfocados en una responsabilidad |
| **Justificación** | 10% | Explica decisiones de diseño |
