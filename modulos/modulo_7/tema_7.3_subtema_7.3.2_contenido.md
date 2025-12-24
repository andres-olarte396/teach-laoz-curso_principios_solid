# Métricas de Calidad SOLID

## Métricas de Acoplamiento

### Afferent Coupling (Ca)
Número de clases externas que dependen de esta clase.

### Efferent Coupling (Ce)
Número de clases externas de las que depende esta clase.

### Instability (I)
```
I = Ce / (Ca + Ce)
```
- I = 0: Muy estable (muchas dependencias entrantes)
- I = 1. Muy inestable (muchas dependencias salientes)

## Métricas de Cohesión

### LCOM (Lack of Cohesion of Methods)
Mide si métodos usan variables de instancia comunes.

**Alto LCOM** = Baja cohesión (viola SRP)

## Métricas de Complejidad

### Cyclomatic Complexity
Número de caminos independientes en código.

**Alta complejidad** = Posible violación OCP (muchos if/else)

## Herramientas

```bash
# JDepend: análisis de dependencias
jdepend src/

# SonarQube: métricas completas
sonar-scanner

# Metrics Reloaded (IntelliJ IDEA)
# Analyze → Calculate Metrics
```

## Ejemplo Análisis

```java
// ❌ Alto Ce (depende de muchas clases)
class OrderController {
    private OrderService orderService;
    private UserService userService;
    private ProductService productService;
    private PaymentService paymentService;
    private EmailService emailService;
    private LoggingService loggingService;
    // Ce = 6, alta inestabilidad
}

// ✅ Bajo Ce (depende de facade)
class OrderController {
    private OrderFacade facade; // Ce = 1
}
```

## Resumen

**Métricas** = Acoplamiento (Ca, Ce, I), cohesión (LCOM), complejidad (CC).
