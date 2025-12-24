# God Classes y Rigidez de Código

## Relación OCP - God Classes

Las **God Classes** (clases dios) violan OCP porque:
1. Tienen demasiadas responsabilidades → cualquier nueva funcionalidad las modifica
2. Son difíciles de extender sin romper código existente
3. Alto acoplamiento → cambios en cascada

## Características de God Classes

```java
// ❌ God Class
public class OrderManager {
    // 1. Validación
    public boolean validateOrder(Order order) { /* ... */ }
    public boolean validateCustomer(Customer customer) { /* ... */ }
    public boolean validateInventory(Order order) { /* ... */ }
    
    // 2. Cálculos
    public double calculateSubtotal(Order order) { /* ... */ }
    public double calculateTax(Order order) { /* ... */ }
    public double calculateShipping(Order order) { /* ... */ }
    public double calculateDiscount(Order order) { /* ... */ }
    
    // 3. Persistencia
    public void saveOrder(Order order) { /* ... */ }
    public Order loadOrder(Long id) { /* ... */ }
    public void updateOrder(Order order) { /* ... */ }
    
    // 4. Notificaciones
    public void sendConfirmationEmail(Order order) { /* ... */ }
    public void sendSMS(Order order) { /* ... */ }
    public void notifyWarehouse(Order order) { /* ... */ }
    
    // 5. Reporting
    public Report generateSalesReport() { /* ... */ }
    public Report generateInventoryReport() { /* ... */ }
    
    // 6. Integraciones
    public void syncWithERP(Order order) { /* ... */ }
    public void syncWithCRM(Order order) { /* ... */ }
}
```

**Violación OCP**: Cualquier nuevo requisito (nuevo método de pago, nuevo tipo de descuento, nueva integración) requiere modificar `OrderManager`.

## Rigidez: Imposibilidad de Extensión

### Problema 1. Métodos Privados Cerrados

```java
public class InvoiceGenerator {
    public String generateInvoice(Order order) {
        String header = createHeader(order);  // Privado, no extensible
        String items = createItems(order);    // Privado, no extensible
        String footer = createFooter(order);  // Privado, no extensible
        return header + items + footer;
    }
    
    private String createHeader(Order order) {
        return "INVOICE\n";
    }
    
    private String createItems(Order order) {
        // Lógica fija
    }
    
    private String createFooter(Order order) {
        return "Thank you";
    }
}
```

**Limitación**: No puedes personalizar el header para clientes VIP sin modificar la clase.

### ✅ Solución: Métodos Protected + Template Method

```java
public abstract class InvoiceGenerator {
    public final String generateInvoice(Order order) {
        String header = createHeader(order);
        String items = createItems(order);
        String footer = createFooter(order);
        return header + items + footer;
    }
    
    protected String createHeader(Order order) {
        return "INVOICE\n";
    }
    
    protected abstract String createItems(Order order);
    
    protected String createFooter(Order order) {
        return "Thank you";
    }
}

public class VipInvoiceGenerator extends InvoiceGenerator {
    @Override
    protected String createHeader(Order order) {
        return "VIP INVOICE\n★★★★★\n";
    }
    
    @Override
    protected String createItems(Order order) {
        // Lógica específica
    }
}
```

### Problema 2: Lógica Hardcoded

```java
public class PriceCalculator {
    public double calculatePrice(Product product) {
        double basePrice = product.getBasePrice();
        
        // Descuento hardcoded
        if (product.getCategory().equals("ELECTRONICS")) {
            basePrice *= 0.9; // 10% off
        }
        
        // Impuesto hardcoded
        double tax = basePrice * 0.21; // 21% VAT
        
        return basePrice + tax;
    }
}
```

**Limitación**: No puedes añadir nuevas reglas de descuento o impuestos sin modificar `PriceCalculator`.

### ✅ Solución: Strategy Pattern

```java
public interface DiscountStrategy {
    double apply(double price);
}

public interface TaxStrategy {
    double calculate(double price);
}

public class PriceCalculator {
    private DiscountStrategy discountStrategy;
    private TaxStrategy taxStrategy;
    
    public PriceCalculator(DiscountStrategy discountStrategy, TaxStrategy taxStrategy) {
        this.discountStrategy = discountStrategy;
        this.taxStrategy = taxStrategy;
    }
    
    public double calculatePrice(Product product) {
        double basePrice = product.getBasePrice();
        double discounted = discountStrategy.apply(basePrice);
        double tax = taxStrategy.calculate(discounted);
        return discounted + tax;
    }
}

// Extensión sin modificación
DiscountStrategy electronicDiscount = price -> price * 0.9;
TaxStrategy vatTax = price -> price * 0.21;
PriceCalculator calculator = new PriceCalculator(electronicDiscount, vatTax);
```

## Caso de Estudio: Sistema de Reportes Rígido

### Antes (Rígido)

```java
public class ReportGenerator {
    public void generateReport(String type) {
        List<Sale> sales = database.query("SELECT * FROM sales");
        
        if (type.equals("PDF")) {
            // Lógica PDF fija (500 líneas)
            generatePdfReport(sales);
        } else if (type.equals("EXCEL")) {
            // Lógica Excel fija (400 líneas)
            generateExcelReport(sales);
        }
    }
    
    private void generatePdfReport(List<Sale> sales) {
        // Formato fijo, no personalizable
        // Colores fijos, fuente fija, layout fijo
    }
}
```

**Problemas**:
- No puedes personalizar el formato PDF
- No puedes añadir nuevo tipo de reporte sin modificar `ReportGenerator`
- 900 líneas en una clase

### Después (Extensible)

```java
// 1. Separar responsabilidades
public interface DataSource {
    List<Sale> fetchData();
}

public interface ReportFormatter {
    void format(List<Sale> sales, OutputStream output);
}

// 2. Implementaciones extensibles
public class SalesDataSource implements DataSource {
    public List<Sale> fetchData() {
        return database.query("SELECT * FROM sales");
    }
}

public class PdfReportFormatter implements ReportFormatter {
    private PdfConfig config; // Configurable
    
    public void format(List<Sale> sales, OutputStream output) {
        // Usar config para personalizar
    }
}

public class ExcelReportFormatter implements ReportFormatter {
    public void format(List<Sale> sales, OutputStream output) {
        // Lógica Excel
    }
}

// 3. Generador pequeño y extensible
public class ReportGenerator {
    private DataSource dataSource;
    private ReportFormatter formatter;
    
    public ReportGenerator(DataSource dataSource, ReportFormatter formatter) {
        this.dataSource = dataSource;
        this.formatter = formatter;
    }
    
    public void generate(OutputStream output) {
        List<Sale> data = dataSource.fetchData();
        formatter.format(data, output);
    }
}

// Uso
DataSource salesData = new SalesDataSource();
ReportFormatter pdfFormatter = new PdfReportFormatter(new PdfConfig(font, colors));
ReportGenerator generator = new ReportGenerator(salesData, pdfFormatter);
generator.generate(outputStream);

// Extensión: Nuevo tipo de reporte (CSV)
ReportFormatter csvFormatter = new CsvReportFormatter();
ReportGenerator csvGenerator = new ReportGenerator(salesData, csvFormatter);
```

## Refactoring: Decomposición de God Classes

### Paso 1. Identificar Responsabilidades

Usar técnica de "Actor" o "Razón de Cambio":
- **Validación**: Cambia cuando cambian reglas de negocio
- **Cálculos**: Cambia cuando cambian fórmulas
- **Persistencia**: Cambia cuando cambia DB
- **Notificaciones**: Cambia cuando cambian canales de comunicación

### Paso 2: Extraer Clases

```java
// De OrderManager (1 God Class)
// A múltiples clases cohesivas:

public class OrderValidator {
    public ValidationResult validate(Order order) { /* ... */ }
}

public class OrderPricingService {
    public double calculateTotal(Order order) { /* ... */ }
}

public class OrderRepository {
    public void save(Order order) { /* ... */ }
    public Order findById(Long id) { /* ... */ }
}

public class OrderNotificationService {
    public void sendConfirmation(Order order) { /* ... */ }
}

public class OrderReportGenerator {
    public Report generate(DateRange range) { /* ... */ }
}

// Orquestador (delgado)
public class OrderService {
    private OrderValidator validator;
    private OrderPricingService pricingService;
    private OrderRepository repository;
    private OrderNotificationService notificationService;
    
    public void placeOrder(Order order) {
        validator.validate(order);
        double total = pricingService.calculateTotal(order);
        order.setTotal(total);
        repository.save(order);
        notificationService.sendConfirmation(order);
    }
}
```

## Métricas de Rigidez

### 1. Número de Modificaciones

```bash
git log --all --pretty=format: --name-only -- OrderManager.java | wc -l
```

Si una clase se modifica frecuentemente, es rígida (no extensible, solo modificable).

### 2. Tamaño de Clase

- **Lines of Code**: >500 LOC = probable God Class
- **Number of Methods**: >20 métodos = probable God Class
- **Dependencies**: >10 dependencias = alto acoplamiento

### 3. Change Coupling

Si cambiar clase A siempre requiere cambiar clases B, C, D → rigidez.

## Resumen

**God Classes** violan OCP porque:
- Concentran muchas responsabilidades
- Cualquier cambio las modifica
- Dificultan extensión sin modificación

**Solución**: Descomponer en clases cohesivas, usar interfaces y composición.

**Objetivo**: Clases pequeñas (<200 LOC), con única responsabilidad, fácilmente extensibles.
