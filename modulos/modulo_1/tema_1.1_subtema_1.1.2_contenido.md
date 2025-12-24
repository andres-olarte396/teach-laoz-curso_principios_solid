# Subtema 1.1.2: Aplicación Práctica del SRP

## 1. Contexto y Motivación

Entender SRP teóricamente es solo el primer paso. La aplicación práctica requiere:
- **Reconocer** violaciones en código existente
- **Refactorizar** sistemáticamente hacia SRP
- **Diseñar** nuevas funcionalidades respetando SRP desde el inicio
- **Evaluar** trade-offs (cuándo es "suficientemente bueno")

Este subtema se enfoca en técnicas concretas para aplicar SRP en proyectos reales.

## 2. Proceso de Refactorización hacia SRP

### 2.1 Metodología de 5 Pasos

**Paso 1. Identificar Responsabilidades**
- Listar todos los métodos de la clase
- Agrupar por concepto/dominio
- Identificar actores que pedirían cambios

**Paso 2: Analizar Dependencias**
- ¿Qué métodos usan qué datos?
- ¿Hay grupos cohesivos de métodos?
- ¿Qué métodos son independientes?

**Paso 3: Extraer Clases**
- Crear nuevas clases para cada responsabilidad
- Mover métodos y datos relacionados
- Mantener interfaces públicas claras

**Paso 4: Actualizar Dependencias**
- Inyectar dependencias extraídas
- Actualizar llamadas a métodos
- Mantener tests pasando

**Paso 5: Verificar y Refinar**
- ¿Cada clase tiene una responsabilidad clara?
- ¿Tests son más simples?
- ¿Nombres reflejan la responsabilidad?

### 2.2 Ejemplo Paso a Paso

**PASO 1. Identificar Responsabilidades**

```java
// Clase original
public class Employee {
    private String name;
    private double salary;
    private String department;
    private int hoursWorked;
    
    // Grupo 1. Cálculos de nómina
    public double calculatePay() {
        return salary + calculateBonus();
    }
    
    private double calculateBonus() {
        return salary * 0.1;
    }
    
    // Grupo 2: Persistencia
    public void save() {
        Database.execute("INSERT INTO employees...");
    }
    
    public static Employee load(int id) {
        return Database.query("SELECT * FROM employees WHERE id = " + id);
    }
    
    // Grupo 3: Reportes
    public String generateReport() {
        return "Employee: " + name + "\nSalary: " + salary;
    }
}
```

**Responsabilidades identificadas:**
1. **Modelo de dominio**: Representar datos del empleado
2. **Cálculos de nómina**: calculatePay(), calculateBonus()
3. **Persistencia**: save(), load()
4. **Reportes**: generateReport()

**PASO 2: Analizar Dependencias**

```
calculatePay() → usa salary
calculateBonus() → usa salary
save() → usa name, salary, department, hoursWorked
load() → crea Employee con todos los datos
generateReport() → usa name, salary
```

**PASO 3: Extraer Clases**

```java
// ✅ Clase 1. Modelo de dominio puro
public class Employee {
    private final String id;
    private final String name;
    private final double salary;
    private final String department;
    private int hoursWorked;
    
    public Employee(String id, String name, double salary, String department) {
        this.id = id;
        this.name = name;
        this.salary = salary;
        this.department = department;
        this.hoursWorked = 0;
    }
    
    public void logHours(int hours) {
        this.hoursWorked += hours;
    }
    
    // Getters
    public String getId() { return id; }
    public String getName() { return name; }
    public double getSalary() { return salary; }
    public String getDepartment() { return department; }
    public int getHoursWorked() { return hoursWorked; }
}

// ✅ Clase 2: Cálculos de nómina
public class PayrollCalculator {
    private static final double BONUS_RATE = 0.1;
    
    public double calculatePay(Employee employee) {
        return employee.getSalary() + calculateBonus(employee);
    }
    
    private double calculateBonus(Employee employee) {
        return employee.getSalary() * BONUS_RATE;
    }
    
    public double calculateYearlyPay(Employee employee) {
        return calculatePay(employee) * 12;
    }
}

// ✅ Clase 3: Persistencia
public class EmployeeRepository {
    private final Connection connection;
    
    public EmployeeRepository(Connection connection) {
        this.connection = connection;
    }
    
    public void save(Employee employee) {
        String sql = "INSERT INTO employees (id, name, salary, department, hours_worked) " +
                     "VALUES (?, ?, ?, ?, ?)";
        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            stmt.setString(1, employee.getId());
            stmt.setString(2, employee.getName());
            stmt.setDouble(3, employee.getSalary());
            stmt.setString(4, employee.getDepartment());
            stmt.setInt(5, employee.getHoursWorked());
            stmt.executeUpdate();
        } catch (SQLException e) {
            throw new PersistenceException("Failed to save employee", e);
        }
    }
    
    public Employee findById(String id) {
        String sql = "SELECT * FROM employees WHERE id = ?";
        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            stmt.setString(1, id);
            ResultSet rs = stmt.executeQuery();
            if (rs.next()) {
                return new Employee(
                    rs.getString("id"),
                    rs.getString("name"),
                    rs.getDouble("salary"),
                    rs.getString("department")
                );
            }
            return null;
        } catch (SQLException e) {
            throw new PersistenceException("Failed to load employee", e);
        }
    }
}

// ✅ Clase 4: Generación de reportes
public class EmployeeReportGenerator {
    public String generateTextReport(Employee employee) {
        return String.format("Employee: %s%nSalary: $%.2f%nDepartment: %s",
                             employee.getName(),
                             employee.getSalary(),
                             employee.getDepartment());
    }
    
    public String generateHTMLReport(Employee employee) {
        return String.format(
            "<div class='employee'>" +
            "<h2>%s</h2>" +
            "<p>Salary: $%.2f</p>" +
            "<p>Department: %s</p>" +
            "</div>",
            employee.getName(),
            employee.getSalary(),
            employee.getDepartment()
        );
    }
}
```

**PASO 4: Actualizar Código Cliente**

```java
// Antes (código cliente acoplado)
Employee emp = new Employee("E001", "John", 50000, "IT");
double pay = emp.calculatePay();
emp.save();
String report = emp.generateReport();

// Después (usando servicios especializados)
Employee emp = new Employee("E001", "John", 50000, "IT");

PayrollCalculator payroll = new PayrollCalculator();
double pay = payroll.calculatePay(emp);

EmployeeRepository repository = new EmployeeRepository(connection);
repository.save(emp);

EmployeeReportGenerator reportGen = new EmployeeReportGenerator();
String report = reportGen.generateTextReport(emp);
```

**PASO 5: Verificar Resultados**

✅ **Employee**: Solo datos y lógica de dominio básica  
✅ **PayrollCalculator**: Solo cálculos de nómina  
✅ **EmployeeRepository**: Solo persistencia  
✅ **EmployeeReportGenerator**: Solo generación de reportes  

Cada clase tiene una razón clara para cambiar.

## 3. Técnicas de Extracción

### 3.1 Extract Class

**Cuándo usar:** Clase con múltiples responsabilidades cohesivas

```java
// Antes
public class Order {
    private String orderId;
    private Customer customer;
    private Address shippingAddress;
    
    public void validateAddress() { }
    public String formatAddress() { }
    // ... más métodos de dirección
}

// Después
public class Order {
    private String orderId;
    private Customer customer;
    private ShippingAddress shippingAddress; // Clase extraída
}

public class ShippingAddress {
    private String street;
    private String city;
    private String zipCode;
    
    public boolean isValid() { }
    public String format() { }
}
```

### 3.2 Extract Service

**Cuándo usar:** Operaciones que no pertenecen al modelo de dominio

```java
// Antes
public class User {
    public void sendWelcomeEmail() {
        EmailService service = new EmailService();
        service.send(this.email, "Welcome!", "...");
    }
}

// Después
public class User {
    // Solo datos y lógica de dominio
}

public class UserNotificationService {
    private final EmailService emailService;
    
    public void sendWelcomeEmail(User user) {
        emailService.send(user.getEmail(), "Welcome!", "...");
    }
}
```

### 3.3 Extract Interface

**Cuándo usar:** Múltiples implementaciones de una responsabilidad

```java
// Antes
public class FileLogger {
    public void log(String message) {
        // Escribir a archivo
    }
}

// Después
public interface Logger {
    void log(String message);
}

public class FileLogger implements Logger {
    public void log(String message) {
        // Escribir a archivo
    }
}

public class DatabaseLogger implements Logger {
    public void log(String message) {
        // Escribir a BD
    }
}

public class ConsoleLogger implements Logger {
    public void log(String message) {
        System.out.println(message);
    }
}
```

### 3.4 Move Method

**Cuándo usar:** Método usa más datos de otra clase que de la propia

```java
// Antes
public class Invoice {
    private Customer customer;
    
    public double calculateDiscount() {
        if (customer.isPremium()) {
            return total * 0.15;
        }
        return total * 0.05;
    }
}

// Después
public class Customer {
    public double getDiscountRate() {
        return isPremium() ? 0.15 : 0.05;
    }
}

public class Invoice {
    private Customer customer;
    
    public double calculateDiscount() {
        return total * customer.getDiscountRate();
    }
}
```

## 4. Patrones de Diseño que Facilitan SRP

### 4.1 Repository Pattern (Persistencia)

```java
// Separa lógica de negocio de persistencia
public interface OrderRepository {
    void save(Order order);
    Order findById(String id);
    List<Order> findByCustomerId(String customerId);
}

public class JdbcOrderRepository implements OrderRepository {
    // Implementación específica de JDBC
}

public class MongoOrderRepository implements OrderRepository {
    // Implementación específica de MongoDB
}
```

### 4.2 Strategy Pattern (Algoritmos Intercambiables)

```java
// Separa lógica de cálculo en estrategias
public interface PricingStrategy {
    double calculatePrice(Order order);
}

public class StandardPricing implements PricingStrategy {
    public double calculatePrice(Order order) {
        return order.getSubtotal();
    }
}

public class SeasonalPricing implements PricingStrategy {
    public double calculatePrice(Order order) {
        return order.getSubtotal() * 0.9; // 10% descuento
    }
}

public class PriceCalculator {
    private PricingStrategy strategy;
    
    public void setStrategy(PricingStrategy strategy) {
        this.strategy = strategy;
    }
    
    public double calculate(Order order) {
        return strategy.calculatePrice(order);
    }
}
```

### 4.3 Facade Pattern (Simplificar Complejidad)

```java
// Fachada que orquesta múltiples servicios con responsabilidades únicas
public class OrderFacade {
    private final OrderValidator validator;
    private final InventoryService inventory;
    private final PaymentService payment;
    private final ShippingService shipping;
    private final NotificationService notification;
    
    public OrderResult placeOrder(OrderRequest request) {
        // Orquesta servicios especializados
        validator.validate(request);
        inventory.reserve(request.getItems());
        payment.process(request.getPaymentInfo());
        shipping.schedule(request.getShippingAddress());
        notification.sendConfirmation(request.getCustomerId());
        
        return OrderResult.success();
    }
}
```

## 5. Code Smells que Indican Violación de SRP

### 5.1 Large Class (Clase Grande)

```java
// ❌ Señal: Más de 200-300 líneas
public class UserManager {
    // 50+ métodos, 500+ líneas
}

// ✅ Refactorizar en clases más pequeñas
public class UserValidator { }
public class UserRepository { }
public class UserAuthenticationService { }
```

### 5.2 Divergent Change (Cambio Divergente)

```java
// ❌ Señal: Cambiar por múltiples razones
public class Product {
    // Cambio 1. Nueva regla de precio
    double calculatePrice() { }
    
    // Cambio 2: Nuevo formato de persistencia
    void saveToDB() { }
    
    // Cambio 3: Nuevo formato de exportación
    String toXML() { }
}
```

### 5.3 Shotgun Surgery (Cirugía de Escopeta)

```java
// ❌ Señal: Un cambio requiere modificar muchas clases
// Para cambiar formato de fecha, modificar 15 clases diferentes

// ✅ Centralizar responsabilidad
public class DateFormatter {
    public String format(LocalDate date) {
        // Un solo lugar para cambiar formato
    }
}
```

### 5.4 Feature Envy (Envidia de Características)

```java
// ❌ Método usa más datos de otra clase
public class Invoice {
    public double calculateDiscount(Customer customer) {
        return customer.getPurchaseHistory().getTotalSpent() > 10000
            ? customer.getPurchaseHistory().getTotalSpent() * 0.1
            : 0;
    }
}

// ✅ Mover lógica a la clase que tiene los datos
public class Customer {
    public double calculateLoyaltyDiscount() {
        return purchaseHistory.getTotalSpent() > 10000
            ? purchaseHistory.getTotalSpent() * 0.1
            : 0;
    }
}
```

## 6. Trade-offs y Decisiones Pragmáticas

### 6.1 Simplicidad vs Pureza

```java
// Opción 1. SRP puro (muchas clases)
public class User { }
public class UserValidator { }
public class UserRepository { }
public class UserEmailSender { }
public class UserPasswordHasher { }
public class UserSessionManager { }
// ... 10 clases más

// Opción 2: Pragmático (equilibrio)
public class User { }
public class UserService {
    // Agrupa operaciones muy relacionadas
    private UserValidator validator;
    private UserRepository repository;
}
```

**Guía:** Si las clases SIEMPRE se usan juntas, considera agruparlas.

### 6.2 Tamaño de Proyecto

**Proyectos pequeños (<1000 LOC):**
- Menos estricto con SRP
- Evitar sobre-ingeniería

**Proyectos medianos (1000-10000 LOC):**
- Aplicar SRP consistentemente
- Comenzar a usar patrones

**Proyectos grandes (>10000 LOC):**
- SRP es crítico
- Arquitectura por capas/módulos

### 6.3 Velocidad de Desarrollo

**Prototipo/MVP:**
```java
// Aceptable: Combinar responsabilidades temporalmente
public class QuickPrototypeService {
    void doEverything() { }
}
```

**Producción:**
```java
// Requerido: Separar responsabilidades
public class ValidationService { }
public class BusinessLogicService { }
public class PersistenceService { }
```

## 7. Métricas para Evaluar SRP

### 7.1 Cohesión (LCOM - Lack of Cohesion of Methods)

```
LCOM bajo = Alta cohesión = Buen SRP
LCOM alto = Baja cohesión = Violación de SRP
```

```java
// LCOM bajo (BUENO)
public class Rectangle {
    private double width;
    private double height;
    
    public double getArea() {
        return width * height; // Usa width y height
    }
    
    public double getPerimeter() {
        return 2 * (width + height); // Usa width y height
    }
}

// LCOM alto (MALO)
public class MixedClass {
    private double width;
    private double height;
    private String email;
    
    public double getArea() {
        return width * height; // Usa width y height
    }
    
    public boolean isValidEmail() {
        return email.contains("@"); // Solo usa email
    }
}
```

### 7.2 Número de Responsabilidades

**Heurística:**
- 1 responsabilidad = ✅ Excelente
- 2 responsabilidades = ⚠️ Considerar separar
- 3+ responsabilidades = ❌ Refactorizar

### 7.3 Tamaño de Clase

**Líneas de código:**
- < 100 líneas = Generalmente bueno
- 100-200 líneas = Revisar responsabilidades
- > 200 líneas = Probablemente viola SRP

**Número de métodos:**
- < 10 métodos = Generalmente bueno
- 10-15 métodos = Evaluar cohesión
- > 15 métodos = Probablemente múltiples responsabilidades

## 8. Resumen Ejecutivo

**Proceso de Refactorización:**
1. Identificar responsabilidades
2. Analizar dependencias
3. Extraer clases
4. Actualizar dependencias
5. Verificar y refinar

**Técnicas Clave:**
- Extract Class
- Extract Service
- Extract Interface
- Move Method

**Patrones Útiles:**
- Repository (persistencia)
- Strategy (algoritmos)
- Facade (orquestación)

**Code Smells:**
- Large Class
- Divergent Change
- Shotgun Surgery
- Feature Envy

## 9. Puntos Clave

✅ Aplicar **refactorización sistemática** hacia SRP  
✅ Usar **patrones de diseño** para separar responsabilidades  
✅ Detectar **code smells** que indican violaciones  
✅ **Equilibrar** pureza con pragmatismo  
✅ Clases pequeñas (<200 LOC) son más manejables  
✅ Cada extracción debe tener **nombre claro**  
❌ Evitar **sobre-fragmentación** innecesaria  
❌ No aplicar SRP mecánicamente sin considerar contexto
