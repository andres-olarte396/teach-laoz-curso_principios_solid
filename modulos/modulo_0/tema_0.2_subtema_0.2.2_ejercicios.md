# Ejercicios: Introducción a Patrones de Diseño

## BANCO DE EJERCICIOS GRADUADOS

### ⭐ NIVEL 1. Conceptuales (Identificación)

#### Ejercicio 1.1. Identificar Patrones en Código

**Enunciado:**
Identifica qué patrón de diseño se está usando en cada fragmento de código:

```java
// Fragmento 1
public abstract class ReportGenerator {
    public final void generateReport() {
        collectData();
        formatData();
        printReport();
    }
    
    protected abstract void collectData();
    protected abstract void formatData();
    protected void printReport() { /* ... */ }
}

// Fragmento 2
public interface Logger {
    void log(String message);
}

public class FileLogger implements Logger {
    private Logger wrapped;
    
    public FileLogger(Logger logger) {
        this.wrapped = logger;
    }
    
    public void log(String message) {
        System.out.println("Writing to file: " + message);
        if (wrapped != null) wrapped.log(message);
    }
}

// Fragmento 3
public class GameSettings {
    private static GameSettings instance;
    
    private GameSettings() { }
    
    public static GameSettings getInstance() {
        if (instance == null) instance = new GameSettings();
        return instance;
    }
}

// Fragmento 4
public interface SortStrategy {
    void sort(int[] array);
}

public class Sorter {
    private SortStrategy strategy;
    
    public void setStrategy(SortStrategy s) {
        this.strategy = s;
    }
    
    public void sort(int[] array) {
        strategy.sort(array);
    }
}
```

**Solución Modelo:**

1. **Fragmento 1. Template Method Pattern**
   - Descripción: Define el esqueleto de un algoritmo, delegando algunos pasos a subclases
   - Indicadores: Método `final` con estructura fija, métodos `abstract` para pasos variables
   - Propósito: Reutilizar estructura común mientras permite personalización

2. **Fragmento 2: Decorator Pattern**
   - Descripción: Envuelve un objeto para agregar funcionalidad dinámicamente
   - Indicadores: Constructor recibe mismo tipo que implementa, delega llamadas
   - Propósito: Extender funcionalidad sin herencia

3. **Fragmento 3: Singleton Pattern**
   - Descripción: Garantiza una única instancia de la clase
   - Indicadores: Constructor privado, instancia estática, método `getInstance()`
   - Propósito: Control de acceso a recurso único

4. **Fragmento 4: Strategy Pattern**
   - Descripción: Encapsula familia de algoritmos intercambiables
   - Indicadores: Interface de estrategia, contexto que la usa, cambio en runtime
   - Propósito: Algoritmos flexibles e intercambiables

**Rúbrica:**
- Identificación correcta (50%): Nombró el patrón correcto
- Justificación (30%): Explicó características identificadoras
- Propósito (20%): Describió para qué se usa el patrón

---

#### Ejercicio 1.2: Cuándo Usar Cada Patrón

**Enunciado:**
Para cada escenario, indica qué patrón es más apropiado y por qué:

1. Necesitas diferentes algoritmos de compresión de archivos intercambiables
2. Quieres agregar funcionalidad de logging, caching y validación a un servicio web
3. Múltiples componentes de UI necesitan actualizarse cuando cambia el modelo de datos
4. Necesitas crear diferentes tipos de documentos (PDF, Word, HTML) pero la lógica de creación es compleja

**Solución Modelo:**

1. **Strategy Pattern**
   - Razón: Encapsula algoritmos de compresión (ZIP, GZIP, RAR) en estrategias separadas
   - Permite cambiar algoritmo en runtime según necesidad
   - Ejemplo: `CompressionStrategy` con `ZipStrategy`, `GzipStrategy`, `RarStrategy`

2. **Decorator Pattern**
   - Razón: Cada funcionalidad (logging, caching, validación) se puede agregar/quitar independientemente
   - Permite combinar múltiples decoradores en cualquier orden
   - Ejemplo: `LoggingDecorator`, `CachingDecorator`, `ValidationDecorator` envolviendo `WebService`

3. **Observer Pattern**
   - Razón: Modelo de datos es el Subject, componentes UI son Observers
   - Cuando modelo cambia, notifica automáticamente a todos los observadores
   - Ejemplo: `DataModel.notifyObservers()` → todos los `UIComponent.update()`

4. **Factory Method Pattern** o **Abstract Factory Pattern**
   - Razón: Encapsula lógica compleja de creación de documentos
   - Cada factory conoce cómo crear su tipo específico de documento
   - Ejemplo: `PDFDocumentFactory`, `WordDocumentFactory`, `HTMLDocumentFactory`

---

### ⭐⭐ NIVEL 2: Implementación Básica

#### Ejercicio 2.1. Implementar Strategy Pattern - Sistema de Envío

**Enunciado:**
Implementa un sistema de cálculo de costos de envío que soporte múltiples estrategias:
- Envío estándar: $5 base
- Envío express: $15 base
- Envío internacional: $30 base + $2 por kg

**Requisitos:**
1. Interface `ShippingStrategy` con método `calculateCost(double weight)`
2. Tres implementaciones concretas
3. Clase `ShippingCalculator` que use la estrategia
4. Tests para cada estrategia

**Solución Modelo:**

```java
// Strategy interface
public interface ShippingStrategy {
    double calculateCost(double weight);
    String getDescription();
}

// Concrete Strategies
public class StandardShipping implements ShippingStrategy {
    private static final double BASE_COST = 5.0;
    
    @Override
    public double calculateCost(double weight) {
        return BASE_COST;
    }
    
    @Override
    public String getDescription() {
        return "Standard Shipping (3-5 business days)";
    }
}

public class ExpressShipping implements ShippingStrategy {
    private static final double BASE_COST = 15.0;
    
    @Override
    public double calculateCost(double weight) {
        return BASE_COST;
    }
    
    @Override
    public String getDescription() {
        return "Express Shipping (1-2 business days)";
    }
}

public class InternationalShipping implements ShippingStrategy {
    private static final double BASE_COST = 30.0;
    private static final double COST_PER_KG = 2.0;
    
    @Override
    public double calculateCost(double weight) {
        return BASE_COST + (weight * COST_PER_KG);
    }
    
    @Override
    public String getDescription() {
        return "International Shipping (7-14 business days)";
    }
}

// Context
public class ShippingCalculator {
    private ShippingStrategy strategy;
    
    public void setShippingStrategy(ShippingStrategy strategy) {
        this.strategy = strategy;
    }
    
    public double calculateShippingCost(double weight) {
        if (strategy == null) {
            throw new IllegalStateException("Shipping strategy not set");
        }
        if (weight <= 0) {
            throw new IllegalArgumentException("Weight must be positive");
        }
        return strategy.calculateCost(weight);
    }
    
    public String getShippingDescription() {
        if (strategy == null) {
            throw new IllegalStateException("Shipping strategy not set");
        }
        return strategy.getDescription();
    }
}

// Tests
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import static org.junit.jupiter.api.Assertions.*;

class ShippingStrategyTest {
    
    private ShippingCalculator calculator;
    
    @BeforeEach
    void setUp() {
        calculator = new ShippingCalculator();
    }
    
    @Test
    void testStandardShipping() {
        calculator.setShippingStrategy(new StandardShipping());
        assertEquals(5.0, calculator.calculateShippingCost(1.0), 0.01);
        assertEquals(5.0, calculator.calculateShippingCost(10.0), 0.01);
    }
    
    @Test
    void testExpressShipping() {
        calculator.setShippingStrategy(new ExpressShipping());
        assertEquals(15.0, calculator.calculateShippingCost(1.0), 0.01);
    }
    
    @Test
    void testInternationalShipping() {
        calculator.setShippingStrategy(new InternationalShipping());
        assertEquals(30.0, calculator.calculateShippingCost(0.0), 0.01); // Solo base
        assertEquals(32.0, calculator.calculateShippingCost(1.0), 0.01); // 30 + 2*1
        assertEquals(40.0, calculator.calculateShippingCost(5.0), 0.01); // 30 + 2*5
    }
    
    @Test
    void testStrategyNotSet() {
        assertThrows(IllegalStateException.class, () -> {
            calculator.calculateShippingCost(1.0);
        });
    }
    
    @Test
    void testInvalidWeight() {
        calculator.setShippingStrategy(new StandardShipping());
        assertThrows(IllegalArgumentException.class, () -> {
            calculator.calculateShippingCost(-1.0);
        });
    }
    
    @Test
    void testChangeStrategyAtRuntime() {
        calculator.setShippingStrategy(new StandardShipping());
        double cost1 = calculator.calculateShippingCost(5.0);
        
        calculator.setShippingStrategy(new ExpressShipping());
        double cost2 = calculator.calculateShippingCost(5.0);
        
        assertNotEquals(cost1, cost2);
    }
}
```

**Rúbrica:**
- Interface correcta (20%)
- Estrategias implementadas correctamente (30%)
- Context funcional (20%)
- Tests completos (30%)

---

#### Ejercicio 2.2: Implementar Observer Pattern - Sistema de Notificaciones

**Enunciado:**
Implementa un sistema donde múltiples observadores se suscriben a cambios de temperatura:
- `TemperatureSensor` (Subject) publica cambios de temperatura
- `Display`, `Logger`, `AlertSystem` (Observers) reaccionan a cambios
- Alerta si temperatura > 30°C

**Solución Modelo:**

```java
// Observer interface
public interface TemperatureObserver {
    void update(double temperature);
}

// Subject interface
public interface TemperatureSubject {
    void attach(TemperatureObserver observer);
    void detach(TemperatureObserver observer);
    void notifyObservers();
}

// Concrete Subject
import java.util.ArrayList;
import java.util.List;

public class TemperatureSensor implements TemperatureSubject {
    private List<TemperatureObserver> observers = new ArrayList<>();
    private double temperature;
    
    @Override
    public void attach(TemperatureObserver observer) {
        if (!observers.contains(observer)) {
            observers.add(observer);
        }
    }
    
    @Override
    public void detach(TemperatureObserver observer) {
        observers.remove(observer);
    }
    
    @Override
    public void notifyObservers() {
        for (TemperatureObserver observer : observers) {
            observer.update(temperature);
        }
    }
    
    public void setTemperature(double temperature) {
        this.temperature = temperature;
        notifyObservers();
    }
    
    public double getTemperature() {
        return temperature;
    }
}

// Concrete Observers
public class TemperatureDisplay implements TemperatureObserver {
    private String name;
    private double currentTemperature;
    
    public TemperatureDisplay(String name) {
        this.name = name;
    }
    
    @Override
    public void update(double temperature) {
        this.currentTemperature = temperature;
        System.out.println(name + " Display: " + temperature + "°C");
    }
    
    public double getCurrentTemperature() {
        return currentTemperature;
    }
}

public class TemperatureLogger implements TemperatureObserver {
    private List<String> log = new ArrayList<>();
    
    @Override
    public void update(double temperature) {
        String entry = "Temperature: " + temperature + "°C at " + 
                       java.time.LocalDateTime.now();
        log.add(entry);
        System.out.println("LOGGED: " + entry);
    }
    
    public List<String> getLog() {
        return new ArrayList<>(log);
    }
}

public class AlertSystem implements TemperatureObserver {
    private static final double THRESHOLD = 30.0;
    private boolean alertActive = false;
    
    @Override
    public void update(double temperature) {
        if (temperature > THRESHOLD) {
            if (!alertActive) {
                System.out.println("🚨 ALERT: Temperature exceeds " + THRESHOLD + "°C!");
                alertActive = true;
            }
        } else {
            if (alertActive) {
                System.out.println("✓ Temperature back to normal");
                alertActive = false;
            }
        }
    }
    
    public boolean isAlertActive() {
        return alertActive;
    }
}

// Tests
class TemperatureSensorTest {
    
    @Test
    void testObserversNotifiedOnChange() {
        TemperatureSensor sensor = new TemperatureSensor();
        TemperatureDisplay display = new TemperatureDisplay("Main");
        
        sensor.attach(display);
        sensor.setTemperature(25.0);
        
        assertEquals(25.0, display.getCurrentTemperature(), 0.01);
    }
    
    @Test
    void testMultipleObservers() {
        TemperatureSensor sensor = new TemperatureSensor();
        TemperatureDisplay display1 = new TemperatureDisplay("Room1");
        TemperatureDisplay display2 = new TemperatureDisplay("Room2");
        TemperatureLogger logger = new TemperatureLogger();
        
        sensor.attach(display1);
        sensor.attach(display2);
        sensor.attach(logger);
        
        sensor.setTemperature(22.0);
        
        assertEquals(22.0, display1.getCurrentTemperature(), 0.01);
        assertEquals(22.0, display2.getCurrentTemperature(), 0.01);
        assertEquals(1, logger.getLog().size());
    }
    
    @Test
    void testDetachObserver() {
        TemperatureSensor sensor = new TemperatureSensor();
        TemperatureDisplay display = new TemperatureDisplay("Main");
        
        sensor.attach(display);
        sensor.setTemperature(20.0);
        assertEquals(20.0, display.getCurrentTemperature(), 0.01);
        
        sensor.detach(display);
        sensor.setTemperature(25.0);
        assertEquals(20.0, display.getCurrentTemperature(), 0.01); // No cambió
    }
    
    @Test
    void testAlertSystem() {
        TemperatureSensor sensor = new TemperatureSensor();
        AlertSystem alert = new AlertSystem();
        
        sensor.attach(alert);
        
        sensor.setTemperature(25.0);
        assertFalse(alert.isAlertActive());
        
        sensor.setTemperature(35.0);
        assertTrue(alert.isAlertActive());
        
        sensor.setTemperature(28.0);
        assertFalse(alert.isAlertActive());
    }
}
```

---

### ⭐⭐⭐ NIVEL 3: Integración de Patrones

#### Ejercicio 3.1. Coffee Shop con Strategy y Decorator

**Enunciado:**
Implementa un sistema de cafetería que combine:
- **Decorator**: Para agregar extras (leche, azúcar, crema)
- **Strategy**: Para calcular precio según tipo de cliente (normal, estudiante, VIP)

**Solución Modelo:**

```java
// Component (Coffee)
public interface Coffee {
    String getDescription();
    double getBaseCost();
}

// Concrete Component
public class Espresso implements Coffee {
    public String getDescription() { return "Espresso"; }
    public double getBaseCost() { return 2.0; }
}

public class Americano implements Coffee {
    public String getDescription() { return "Americano"; }
    public double getBaseCost() { return 2.5; }
}

// Decorator
public abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
    
    public String getDescription() {
        return coffee.getDescription();
    }
    
    public double getBaseCost() {
        return coffee.getBaseCost();
    }
}

public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) { super(coffee); }
    
    public String getDescription() {
        return coffee.getDescription() + " + Milk";
    }
    
    public double getBaseCost() {
        return coffee.getBaseCost() + 0.5;
    }
}

// Pricing Strategy
public interface PricingStrategy {
    double calculateFinalPrice(double basePrice);
}

public class NormalPricing implements PricingStrategy {
    public double calculateFinalPrice(double basePrice) {
        return basePrice;
    }
}

public class StudentPricing implements PricingStrategy {
    public double calculateFinalPrice(double basePrice) {
        return basePrice * 0.85; // 15% descuento
    }
}

public class VIPPricing implements PricingStrategy {
    public double calculateFinalPrice(double basePrice) {
        return basePrice * 0.70; // 30% descuento
    }
}

// Order
public class CoffeeOrder {
    private Coffee coffee;
    private PricingStrategy pricingStrategy;
    
    public CoffeeOrder(Coffee coffee, PricingStrategy strategy) {
        this.coffee = coffee;
        this.pricingStrategy = strategy;
    }
    
    public double getFinalPrice() {
        return pricingStrategy.calculateFinalPrice(coffee.getBaseCost());
    }
    
    public String getDescription() {
        return coffee.getDescription();
    }
}

// Uso
Coffee coffee = new Espresso();
coffee = new MilkDecorator(coffee);
coffee = new SugarDecorator(coffee);

CoffeeOrder order = new CoffeeOrder(coffee, new StudentPricing());
System.out.println(order.getDescription() + ": $" + order.getFinalPrice());
```

---

### ⭐⭐⭐⭐ NIVEL 4: Proyecto Completo

#### Ejercicio 4.1. Sistema de Notificaciones Multi-Canal

Implementa un sistema completo que combine:
- **Observer**: Para múltiples suscriptores
- **Strategy**: Para diferentes canales (Email, SMS, Push)
- **Decorator**: Para agregar funcionalidades (retry, logging, rate-limiting)
- **Factory**: Para crear notificadores según configuración

Este ejercicio requiere diseño arquitectónico avanzado con al menos 15 clases.

---

## CRITERIOS DE EVALUACIÓN

| Criterio | Peso | Descripción |
|----------|------|-------------|
| **Correctness** | 35% | Implementación correcta del patrón |
| **Design** | 25% | Adherencia a principios SOLID |
| **Tests** | 20% | Cobertura y calidad de tests |
| **Code Quality** | 20% | Nombres, estructura, documentación |
