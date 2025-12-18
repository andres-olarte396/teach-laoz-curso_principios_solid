# Template Method Pattern para Reutilización

## Definición

**Template Method** define el esqueleto de un algoritmo en una clase base, permitiendo que las subclases sobrescriban pasos específicos sin cambiar la estructura general.

## Estructura

```java
public abstract class DataProcessor {
    // Template Method (final = no se puede sobrescribir)
    public final void process() {
        byte[] data = readData();
        byte[] validated = validateData(data);
        byte[] transformed = transformData(validated);
        saveData(transformed);
    }
    
    // Pasos abstractos (deben implementarse)
    protected abstract byte[] readData();
    protected abstract byte[] transformData(byte[] data);
    
    // Pasos con implementación por defecto (pueden sobrescribirse)
    protected byte[] validateData(byte[] data) {
        // Validación básica
        if (data == null || data.length == 0) {
            throw new IllegalArgumentException("Empty data");
        }
        return data;
    }
    
    protected void saveData(byte[] data) {
        // Guardar en archivo por defecto
        Files.write(Paths.get("output.dat"), data);
    }
}

// Implementación concreta 1
public class CsvDataProcessor extends DataProcessor {
    protected byte[] readData() {
        return Files.readAllBytes(Paths.get("input.csv"));
    }
    
    protected byte[] transformData(byte[] data) {
        // Convertir CSV a formato interno
        return parseCsv(data);
    }
}

// Implementación concreta 2
public class XmlDataProcessor extends DataProcessor {
    protected byte[] readData() {
        return Files.readAllBytes(Paths.get("input.xml"));
    }
    
    protected byte[] transformData(byte[] data) {
        // Convertir XML a formato interno
        return parseXml(data);
    }
    
    @Override
    protected void saveData(byte[] data) {
        // Guardar en base de datos en lugar de archivo
        database.save(data);
    }
}

// Uso
DataProcessor csvProcessor = new CsvDataProcessor();
csvProcessor.process(); // Ejecuta readData() → validateData() → transformData() → saveData()

DataProcessor xmlProcessor = new XmlDataProcessor();
xmlProcessor.process();
```

## OCP con Template Method

**Cerrado para modificación**: El algoritmo general (`process()`) está fijo en la clase base.

**Abierto para extensión**: Nuevos procesadores (JSON, YAML) solo requieren heredar y sobrescribir métodos específicos.

## Caso de Estudio: Generador de Reportes

```java
public abstract class ReportGenerator {
    // Template Method
    public final String generateReport() {
        String data = fetchData();
        String processed = processData(data);
        String formatted = formatReport(processed);
        return addHeader() + formatted + addFooter();
    }
    
    // Pasos abstractos
    protected abstract String fetchData();
    protected abstract String formatReport(String data);
    
    // Hooks (métodos opcionales)
    protected String processData(String data) {
        return data; // Por defecto, sin procesamiento
    }
    
    protected String addHeader() {
        return ""; // Sin encabezado por defecto
    }
    
    protected String addFooter() {
        return ""; // Sin pie por defecto
    }
}

public class PdfSalesReport extends ReportGenerator {
    protected String fetchData() {
        return database.query("SELECT * FROM sales WHERE date = ?", today);
    }
    
    protected String formatReport(String data) {
        return "<PDF>" + data + "</PDF>";
    }
    
    @Override
    protected String addHeader() {
        return "Company Inc. - Sales Report\n";
    }
}

public class ExcelInventoryReport extends ReportGenerator {
    protected String fetchData() {
        return database.query("SELECT * FROM inventory");
    }
    
    protected String formatReport(String data) {
        return "<XLSX>" + data + "</XLSX>";
    }
    
    @Override
    protected String processData(String data) {
        // Agregar columna calculada
        return data + ",STOCK_STATUS";
    }
}
```

**Extensión sin modificación**: Añadir `HtmlCustomerReport` no requiere cambiar `ReportGenerator`.

## Hooks vs Abstract Methods

```java
public abstract class OrderProcessor {
    public final void processOrder(Order order) {
        validateOrder(order);
        
        // Hook: comportamiento opcional
        if (shouldApplyDiscount(order)) {
            applyDiscount(order);
        }
        
        calculateTotal(order);
        saveOrder(order);
        
        // Hook: comportamiento opcional
        afterOrderProcessed(order);
    }
    
    // Método abstracto: obligatorio
    protected abstract void calculateTotal(Order order);
    
    // Hook con implementación por defecto: opcional
    protected boolean shouldApplyDiscount(Order order) {
        return false; // Por defecto, sin descuento
    }
    
    protected void applyDiscount(Order order) {
        // Implementación vacía por defecto
    }
    
    protected void afterOrderProcessed(Order order) {
        // Implementación vacía por defecto
    }
    
    private void validateOrder(Order order) {
        if (order.getItems().isEmpty()) {
            throw new IllegalArgumentException("Empty order");
        }
    }
    
    private void saveOrder(Order order) {
        database.save(order);
    }
}

public class PremiumOrderProcessor extends OrderProcessor {
    protected void calculateTotal(Order order) {
        // Lógica de cálculo
    }
    
    @Override
    protected boolean shouldApplyDiscount(Order order) {
        return order.getTotal() > 100; // Descuento si >$100
    }
    
    @Override
    protected void applyDiscount(Order order) {
        order.setTotal(order.getTotal() * 0.9); // 10% off
    }
    
    @Override
    protected void afterOrderProcessed(Order order) {
        emailService.sendConfirmation(order.getCustomer());
    }
}
```

## Template Method vs Strategy

| Aspecto | Template Method | Strategy |
|---------|----------------|----------|
| **Mecanismo** | Herencia | Composición |
| **Cambio en runtime** | No | Sí |
| **Complejidad** | Menor (una clase base) | Mayor (interfaces + múltiples implementaciones) |
| **Flexibilidad** | Limitada (herencia única) | Alta (múltiples estrategias) |
| **Cuándo usar** | Algoritmo con pasos fijos | Algoritmos intercambiables |

## Ejemplo Comparativo

### Template Method
```java
abstract class Report {
    final void generate() { /* pasos fijos */ }
}
class PdfReport extends Report { /* implementa pasos */ }
```
**Limitación**: No puedes cambiar de PDF a Excel en runtime.

### Strategy
```java
interface Formatter { String format(); }
class Report {
    void generate(Formatter formatter) { formatter.format(); }
}
```
**Ventaja**: Puedes cambiar el formateador en cualquier momento.

## Resumen

**Template Method** = Algoritmo fijo en clase base, pasos variables en subclases.

**OCP**: Extensión vía herencia, sin modificar clase base.

**Hooks**: Métodos opcionales con implementación por defecto.

**Cuándo usar**: Algoritmo con estructura estable pero pasos variables.
