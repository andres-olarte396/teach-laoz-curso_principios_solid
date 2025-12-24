# Definición: Abierto para Extensión, Cerrado para Modificación

## Principio Original (Bertrand Meyer, 1988)

> "Las entidades de software (clases, módulos, funciones) deben estar abiertas para extensión, pero cerradas para modificación."

**Interpretación**: Debes poder añadir nuevo comportamiento sin cambiar el código existente.

## Problema que Resuelve

### ❌ Violación de OCP

```java
public class PaymentProcessor {
    public void processPayment(Payment payment) {
        if (payment.getType().equals("CREDIT_CARD")) {
            // Lógica de tarjeta de crédito
            System.out.println("Processing credit card: " + payment.getAmount());
        } else if (payment.getType().equals("PAYPAL")) {
            // Lógica de PayPal
            System.out.println("Processing PayPal: " + payment.getAmount());
        } else if (payment.getType().equals("BITCOIN")) {
            // Lógica de Bitcoin
            System.out.println("Processing Bitcoin: " + payment.getAmount());
        }
        // Cada nuevo método de pago requiere modificar esta clase
    }
}
```

**Problemas**:
1. Cada nuevo método de pago requiere modificar `PaymentProcessor`
2. Alto riesgo de introducir bugs en código existente
3. Viola SRP (múltiples razones para cambiar)
4. Dificulta testing (necesitas todos los métodos de pago)

### ✅ Solución con OCP

```java
// Abstracción: cerrada para modificación
public interface PaymentMethod {
    void process(double amount);
}

// Implementaciones: extensión sin modificación
public class CreditCardPayment implements PaymentMethod {
    @Override
    public void process(double amount) {
        System.out.println("Processing credit card: " + amount);
        // Lógica específica de tarjeta
    }
}

public class PayPalPayment implements PaymentMethod {
    @Override
    public void process(double amount) {
        System.out.println("Processing PayPal: " + amount);
        // Lógica específica de PayPal
    }
}

public class BitcoinPayment implements PaymentMethod {
    @Override
    public void process(double amount) {
        System.out.println("Processing Bitcoin: " + amount);
        // Lógica específica de Bitcoin
    }
}

// Procesador: nunca necesita modificarse
public class PaymentProcessor {
    public void processPayment(PaymentMethod paymentMethod, double amount) {
        paymentMethod.process(amount);
    }
}

// Uso
PaymentProcessor processor = new PaymentProcessor();
processor.processPayment(new CreditCardPayment(), 100.0);
processor.processPayment(new PayPalPayment(), 200.0);
processor.processPayment(new BitcoinPayment(), 0.5);
// Agregar Apple Pay solo requiere crear ApplePayPayment, sin tocar PaymentProcessor
```

**Beneficios**:
- Añadir `ApplePayPayment` no requiere modificar `PaymentProcessor`
- Cada método de pago es independiente (SRP)
- Testing: mock de `PaymentMethod` sin necesitar implementaciones reales

## Mecanismos de Extensión

### 1. Interfaces y Polimorfismo

```java
public interface DiscountStrategy {
    double calculate(double price);
}

public class NoDiscount implements DiscountStrategy {
    public double calculate(double price) { return price; }
}

public class PercentageDiscount implements DiscountStrategy {
    private double percentage;
    
    public double calculate(double price) {
        return price * (1 - percentage / 100);
    }
}

public class FixedAmountDiscount implements DiscountStrategy {
    private double amount;
    
    public double calculate(double price) {
        return Math.max(0, price - amount);
    }
}
```

### 2. Clases Abstractas

```java
public abstract class Report {
    // Template Method (cerrado para modificación)
    public final void generate() {
        String data = fetchData();
        String formatted = format(data);
        export(formatted);
    }
    
    // Puntos de extensión (abiertos para extensión)
    protected abstract String fetchData();
    protected abstract String format(String data);
    protected abstract void export(String formattedData);
}

public class PdfReport extends Report {
    protected String fetchData() { return "Sales data"; }
    protected String format(String data) { return "<PDF>" + data + "</PDF>"; }
    protected void export(String formattedData) { /* PDF export */ }
}

public class ExcelReport extends Report {
    protected String fetchData() { return "Sales data"; }
    protected String format(String data) { return "<XLSX>" + data + "</XLSX>"; }
    protected void export(String formattedData) { /* Excel export */ }
}
```

### 3. Composición

```java
public class Order {
    private List<OrderProcessor> processors = new ArrayList<>();
    
    public void addProcessor(OrderProcessor processor) {
        processors.add(processor);
    }
    
    public void process() {
        for (OrderProcessor processor : processors) {
            processor.process(this);
        }
    }
}

// Extensión sin modificar Order
order.addProcessor(new ValidateOrderProcessor());
order.addProcessor(new CalculateTaxProcessor());
order.addProcessor(new SendEmailProcessor());
```

## Caso de Estudio: Generador de Facturas

### Antes (Violación OCP)

```java
public class InvoiceGenerator {
    public String generate(Invoice invoice, String format) {
        if (format.equals("PDF")) {
            return "PDF: " + invoice.getTotal();
        } else if (format.equals("HTML")) {
            return "<html>" + invoice.getTotal() + "</html>";
        } else if (format.equals("XML")) {
            return "<invoice>" + invoice.getTotal() + "</invoice>";
        }
        throw new IllegalArgumentException("Unknown format");
    }
}
```

**Cambios recientes**:
- Semana 1. Añadido soporte HTML
- Semana 3: Añadido soporte XML
- Semana 5: Añadido soporte JSON (requirió modificar el if-else nuevamente)

### Después (Cumple OCP)

```java
public interface InvoiceFormatter {
    String format(Invoice invoice);
}

public class PdfInvoiceFormatter implements InvoiceFormatter {
    public String format(Invoice invoice) {
        return "PDF: " + invoice.getTotal();
    }
}

public class HtmlInvoiceFormatter implements InvoiceFormatter {
    public String format(Invoice invoice) {
        return "<html>" + invoice.getTotal() + "</html>";
    }
}

public class XmlInvoiceFormatter implements InvoiceFormatter {
    public String format(Invoice invoice) {
        return "<invoice>" + invoice.getTotal() + "</invoice>";
    }
}

public class InvoiceGenerator {
    private InvoiceFormatter formatter;
    
    public InvoiceGenerator(InvoiceFormatter formatter) {
        this.formatter = formatter;
    }
    
    public String generate(Invoice invoice) {
        return formatter.format(invoice);
    }
}

// Uso
InvoiceGenerator pdfGenerator = new InvoiceGenerator(new PdfInvoiceFormatter());
InvoiceGenerator htmlGenerator = new InvoiceGenerator(new HtmlInvoiceFormatter());

// Añadir JSON: crear JsonInvoiceFormatter, sin tocar InvoiceGenerator
```

## OCP y Cambios Legítimos

**Pregunta**: ¿Nunca se debe modificar código existente?

**Respuesta**: Se debe evitar modificar lógica de negocio estable. Los cambios legítimos incluyen:
- **Bugs**: Siempre se corrigen
- **Refactoring**: Mejorar estructura interna sin cambiar comportamiento
- **Cambios de requisitos**: Si el comportamiento base cambia, es válido modificar

**Regla práctica**: Si el cambio es **añadir** nuevo comportamiento (nueva funcionalidad), usa extensión. Si es **corregir** comportamiento existente, modifica.

## Métricas de Adherencia a OCP

### Frecuencia de Modificación

```bash
git log --all --pretty=format: --name-only | sort | uniq -c | sort -rg
```

Si una clase aparece frecuentemente, probablemente viola OCP.

### Número de if/else o switch/case

Muchos `if (type == ...)` indican violación de OCP.

### Cobertura de Tests

Si añadir funcionalidad requiere modificar tests existentes (no solo agregar nuevos), viola OCP.

## Resumen

**OCP** = Extender comportamiento sin modificar código existente.

**Mecanismos**: Interfaces, clases abstractas, composición.

**Beneficio principal**: Reducir riesgo al añadir funcionalidades (código existente no se toca = no se rompe).
