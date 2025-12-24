# Ejercicios: Acoplamiento y Cohesión

## ⭐ Nivel 1. Cálculo de Métricas

Calcula LCOM4, Ce, Ca para estas clases. Identifica si cumplen SRP.

```java
public class EmailSender {
    private SMTPConfig config;
    public void send(String to, String subject, String body) { }
}

public class OrderService {
    private OrderRepository repo;
    private PaymentService payment;
    private EmailSender emailSender;
    public void processOrder(Order order) { }
}
```

## ⭐⭐ Nivel 2: Refactorización

Reduce acoplamiento en este código aplicando Dependency Inversion:

```java
public class ReportGenerator {
    private MySQLDatabase database = new MySQLDatabase();
    private PDFGenerator pdfGenerator = new PDFGenerator();
    
    public void generateReport() {
        List<Data> data = database.query("SELECT...");
        pdfGenerator.create(data);
    }
}
```

## ⭐⭐⭐ Nivel 3: Proyecto

Diseña un sistema de gestión de inventario con:
- LCOM4 = 1 para todas las clases
- Ce < 4 por clase
- Diagrama de dependencias mostrando Ca/Ce

## ⭐⭐⭐⭐ Nivel 4: Análisis con JDepend

Analiza un proyecto real con JDepend, genera reporte de métricas, identifica clases con peor cohesión/acoplamiento, refactoriza y vuelve a medir.
