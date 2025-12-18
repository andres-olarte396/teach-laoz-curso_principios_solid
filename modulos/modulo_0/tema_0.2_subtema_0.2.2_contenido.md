# Subtema 0.2.2: Introducción a Patrones de Diseño

## 1. Contexto y Motivación

Los **patrones de diseño** son soluciones probadas a problemas comunes de diseño de software. Son fundamentales para entender los principios SOLID, ya que:

- Los principios SOLID **guían** cómo diseñar clases
- Los patrones de diseño **implementan** estos principios en soluciones concretas
- Muchos patrones **ejemplifican** los principios SOLID en acción

Por ejemplo:
- **Strategy Pattern** → Open/Closed Principle
- **Adapter Pattern** → Dependency Inversion Principle
- **Decorator Pattern** → Single Responsibility + Open/Closed

## 2. Fundamentos Teóricos

### 2.1 ¿Qué son los Patrones de Diseño?

**Definición:** Soluciones reutilizables a problemas recurrentes en el diseño de software, documentadas de manera estándar.

**Características:**
- **No son código**: Son plantillas conceptuales
- **Independientes del lenguaje**: Se pueden implementar en cualquier lenguaje OOP
- **Probados**: Soluciones validadas por la experiencia
- **Comunicación**: Vocabulario común entre desarrolladores

### 2.2 Catálogo Gang of Four (GoF)

El libro "Design Patterns: Elements of Reusable Object-Oriented Software" (1994) documentó 23 patrones en tres categorías:

**1. Patrones Creacionales** (creación de objetos):
- Singleton
- Factory Method
- Abstract Factory
- Builder
- Prototype

**2. Patrones Estructurales** (composición de clases/objetos):
- Adapter
- Decorator
- Facade
- Composite
- Proxy
- Bridge
- Flyweight

**3. Patrones de Comportamiento** (interacción entre objetos):
- Strategy
- Observer
- Command
- Template Method
- Iterator
- State
- Chain of Responsibility
- Visitor
- Mediator
- Memento

### 2.3 Estructura de Documentación de un Patrón

Cada patrón se documenta con:
- **Nombre**: Identificador único
- **Intención**: ¿Qué problema resuelve?
- **Motivación**: Escenario ejemplo
- **Aplicabilidad**: ¿Cuándo usarlo?
- **Estructura**: Diagrama UML
- **Participantes**: Clases y responsabilidades
- **Colaboraciones**: Cómo interactúan
- **Consecuencias**: Ventajas y desventajas
- **Implementación**: Detalles prácticos
- **Código de ejemplo**: Implementación concreta

## 3. Patrones Fundamentales

### 3.1 Strategy Pattern

**Intención:** Definir una familia de algoritmos, encapsular cada uno, y hacerlos intercambiables.

**Problema que resuelve:**
```java
// ❌ PROBLEMA: Lógica condicional compleja y difícil de extender
class PaymentProcessor {
    public void processPayment(String method, double amount) {
        if (method.equals("CREDIT_CARD")) {
            // Lógica tarjeta de crédito
            System.out.println("Processing credit card payment: " + amount);
        } else if (method.equals("PAYPAL")) {
            // Lógica PayPal
            System.out.println("Processing PayPal payment: " + amount);
        } else if (method.equals("BITCOIN")) {
            // Lógica Bitcoin
            System.out.println("Processing Bitcoin payment: " + amount);
        }
        // Agregar nuevo método requiere modificar esta clase (viola OCP)
    }
}
```

**Solución con Strategy:**
```java
// Interface Strategy
public interface PaymentStrategy {
    void pay(double amount);
}

// Concrete Strategies
public class CreditCardPayment implements PaymentStrategy {
    private String cardNumber;
    
    public CreditCardPayment(String cardNumber) {
        this.cardNumber = cardNumber;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("Paid " + amount + " using Credit Card " + cardNumber);
    }
}

public class PayPalPayment implements PaymentStrategy {
    private String email;
    
    public PayPalPayment(String email) {
        this.email = email;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("Paid " + amount + " using PayPal account " + email);
    }
}

public class BitcoinPayment implements PaymentStrategy {
    private String walletAddress;
    
    public BitcoinPayment(String walletAddress) {
        this.walletAddress = walletAddress;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("Paid " + amount + " BTC to wallet " + walletAddress);
    }
}

// Context
public class ShoppingCart {
    private PaymentStrategy paymentStrategy;
    private double total;
    
    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }
    
    public void checkout() {
        if (paymentStrategy == null) {
            throw new IllegalStateException("Payment strategy not set");
        }
        paymentStrategy.pay(total);
    }
    
    public void setTotal(double total) {
        this.total = total;
    }
}

// Uso
ShoppingCart cart = new ShoppingCart();
cart.setTotal(100.0);

// Cambiar estrategia en runtime
cart.setPaymentStrategy(new CreditCardPayment("1234-5678"));
cart.checkout();

cart.setPaymentStrategy(new PayPalPayment("user@example.com"));
cart.checkout();

cart.setPaymentStrategy(new BitcoinPayment("1A2B3C4D"));
cart.checkout();
```

**Ventajas:**
- ✅ Cumple Open/Closed Principle (agregar nueva estrategia sin modificar código existente)
- ✅ Elimina condicionales complejos
- ✅ Algoritmos intercambiables en runtime
- ✅ Código más testeable

**Cuándo usar:**
- Múltiples variantes de un algoritmo
- Necesitas cambiar comportamiento en runtime
- Tienes muchos condicionales if/else o switch relacionados

### 3.2 Observer Pattern

**Intención:** Definir una dependencia uno-a-muchos entre objetos, de manera que cuando un objeto cambia de estado, todos sus dependientes son notificados automáticamente.

**Implementación:**
```java
// Subject (Observable)
import java.util.ArrayList;
import java.util.List;

public interface Subject {
    void attach(Observer observer);
    void detach(Observer observer);
    void notifyObservers();
}

// Observer
public interface Observer {
    void update(String message);
}

// Concrete Subject
public class NewsAgency implements Subject {
    private List<Observer> observers = new ArrayList<>();
    private String news;
    
    @Override
    public void attach(Observer observer) {
        observers.add(observer);
    }
    
    @Override
    public void detach(Observer observer) {
        observers.remove(observer);
    }
    
    @Override
    public void notifyObservers() {
        for (Observer observer : observers) {
            observer.update(news);
        }
    }
    
    public void setNews(String news) {
        this.news = news;
        notifyObservers(); // Notificar automáticamente cuando cambia el estado
    }
}

// Concrete Observers
public class NewsChannel implements Observer {
    private String name;
    
    public NewsChannel(String name) {
        this.name = name;
    }
    
    @Override
    public void update(String news) {
        System.out.println(name + " received news: " + news);
    }
}

public class EmailSubscriber implements Observer {
    private String email;
    
    public EmailSubscriber(String email) {
        this.email = email;
    }
    
    @Override
    public void update(String news) {
        System.out.println("Email sent to " + email + ": " + news);
    }
}

// Uso
NewsAgency agency = new NewsAgency();

Observer channel1 = new NewsChannel("CNN");
Observer channel2 = new NewsChannel("BBC");
Observer subscriber = new EmailSubscriber("user@example.com");

agency.attach(channel1);
agency.attach(channel2);
agency.attach(subscriber);

agency.setNews("Breaking: SOLID principles are awesome!");
// Salida:
// CNN received news: Breaking: SOLID principles are awesome!
// BBC received news: Breaking: SOLID principles are awesome!
// Email sent to user@example.com: Breaking: SOLID principles are awesome!

agency.detach(channel2);
agency.setNews("Update: Design patterns simplify code");
// Solo CNN y EmailSubscriber reciben esta noticia
```

**Ventajas:**
- ✅ Acoplamiento débil entre Subject y Observers
- ✅ Soporte para broadcast
- ✅ Fácil agregar nuevos observers

**Cuándo usar:**
- Cambios en un objeto requieren cambios en otros
- No sabes cuántos objetos necesitan ser notificados
- Event-driven systems (UI, eventos de dominio)

### 3.3 Decorator Pattern

**Intención:** Adjuntar responsabilidades adicionales a un objeto dinámicamente. Proporciona una alternativa flexible a la herencia para extender funcionalidad.

**Implementación:**
```java
// Component interface
public interface Coffee {
    String getDescription();
    double getCost();
}

// Concrete Component
public class SimpleCoffee implements Coffee {
    @Override
    public String getDescription() {
        return "Simple Coffee";
    }
    
    @Override
    public double getCost() {
        return 2.0;
    }
}

// Base Decorator
public abstract class CoffeeDecorator implements Coffee {
    protected Coffee decoratedCoffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.decoratedCoffee = coffee;
    }
    
    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription();
    }
    
    @Override
    public double getCost() {
        return decoratedCoffee.getCost();
    }
}

// Concrete Decorators
public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription() + ", Milk";
    }
    
    @Override
    public double getCost() {
        return decoratedCoffee.getCost() + 0.5;
    }
}

public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription() + ", Sugar";
    }
    
    @Override
    public double getCost() {
        return decoratedCoffee.getCost() + 0.2;
    }
}

public class WhipDecorator extends CoffeeDecorator {
    public WhipDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription() + ", Whipped Cream";
    }
    
    @Override
    public double getCost() {
        return decoratedCoffee.getCost() + 0.7;
    }
}

// Uso
Coffee coffee = new SimpleCoffee();
System.out.println(coffee.getDescription() + " $" + coffee.getCost());
// Simple Coffee $2.0

coffee = new MilkDecorator(coffee);
System.out.println(coffee.getDescription() + " $" + coffee.getCost());
// Simple Coffee, Milk $2.5

coffee = new SugarDecorator(coffee);
System.out.println(coffee.getDescription() + " $" + coffee.getCost());
// Simple Coffee, Milk, Sugar $2.7

coffee = new WhipDecorator(coffee);
System.out.println(coffee.getDescription() + " $" + coffee.getCost());
// Simple Coffee, Milk, Sugar, Whipped Cream $3.4
```

**Ventajas:**
- ✅ Más flexible que herencia estática
- ✅ Evita clases sobrecargadas de funcionalidades
- ✅ Cumple Single Responsibility (cada decorator una responsabilidad)
- ✅ Cumple Open/Closed (extender sin modificar)

**Cuándo usar:**
- Agregar responsabilidades a objetos individuales dinámicamente
- Cuando la herencia es impráctica (muchas combinaciones)

### 3.4 Factory Method Pattern

**Intención:** Definir una interfaz para crear objetos, pero dejar que las subclases decidan qué clase instanciar.

**Implementación:**
```java
// Product interface
public interface Vehicle {
    void drive();
}

// Concrete Products
public class Car implements Vehicle {
    @Override
    public void drive() {
        System.out.println("Driving a car on the road");
    }
}

public class Motorcycle implements Vehicle {
    @Override
    public void drive() {
        System.out.println("Riding a motorcycle");
    }
}

public class Truck implements Vehicle {
    @Override
    public void drive() {
        System.out.println("Driving a truck");
    }
}

// Creator (Factory)
public abstract class VehicleFactory {
    // Factory Method
    public abstract Vehicle createVehicle();
    
    // Template Method que usa el Factory Method
    public void deliverVehicle() {
        Vehicle vehicle = createVehicle();
        System.out.println("Preparing vehicle for delivery...");
        vehicle.drive();
        System.out.println("Vehicle delivered!");
    }
}

// Concrete Creators
public class CarFactory extends VehicleFactory {
    @Override
    public Vehicle createVehicle() {
        return new Car();
    }
}

public class MotorcycleFactory extends VehicleFactory {
    @Override
    public Vehicle createVehicle() {
        return new Motorcycle();
    }
}

public class TruckFactory extends VehicleFactory {
    @Override
    public Vehicle createVehicle() {
        return new Truck();
    }
}

// Uso
VehicleFactory factory = new CarFactory();
factory.deliverVehicle();
// Salida:
// Preparing vehicle for delivery...
// Driving a car on the road
// Vehicle delivered!

factory = new MotorcycleFactory();
factory.deliverVehicle();
```

**Ventajas:**
- ✅ Cumple Dependency Inversion (depende de abstracciones)
- ✅ Cumple Open/Closed (agregar nuevos productos sin modificar código)
- ✅ Encapsula lógica de creación

**Cuándo usar:**
- No sabes exactamente qué clase necesitas crear de antemano
- La lógica de creación es compleja
- Quieres delegar la creación a subclases

### 3.5 Singleton Pattern

**Intención:** Garantizar que una clase tenga solo una instancia y proporcionar un punto de acceso global a ella.

**Implementación (Thread-safe):**
```java
public class DatabaseConnection {
    // Instancia única (eager initialization)
    private static final DatabaseConnection INSTANCE = new DatabaseConnection();
    
    // Constructor privado
    private DatabaseConnection() {
        // Inicialización costosa
        System.out.println("Connecting to database...");
    }
    
    // Punto de acceso global
    public static DatabaseConnection getInstance() {
        return INSTANCE;
    }
    
    public void query(String sql) {
        System.out.println("Executing query: " + sql);
    }
}

// Versión Lazy Initialization (thread-safe)
public class ConfigurationManager {
    private static volatile ConfigurationManager instance;
    
    private ConfigurationManager() {
        // Cargar configuración
    }
    
    public static ConfigurationManager getInstance() {
        if (instance == null) {
            synchronized (ConfigurationManager.class) {
                if (instance == null) {
                    instance = new ConfigurationManager();
                }
            }
        }
        return instance;
    }
}

// Uso
DatabaseConnection db1 = DatabaseConnection.getInstance();
DatabaseConnection db2 = DatabaseConnection.getInstance();

System.out.println(db1 == db2); // true - misma instancia
```

**⚠️ Advertencia:** Singleton es controversial:
- ❌ Estado global (dificulta testing)
- ❌ Viola Single Responsibility
- ❌ Puede ocultar dependencias

**Alternativas modernas:**
- Dependency Injection containers
- Scoped instances

## 4. Relación con Principios SOLID

| Patrón | Principios SOLID que implementa |
|--------|--------------------------------|
| **Strategy** | Open/Closed, Dependency Inversion |
| **Observer** | Open/Closed, Interface Segregation |
| **Decorator** | Single Responsibility, Open/Closed |
| **Factory Method** | Dependency Inversion, Open/Closed |
| **Adapter** | Dependency Inversion, Interface Segregation |
| **Template Method** | Open/Closed, Liskov Substitution |

## 5. Antipatrones (Qué Evitar)

### 5.1 God Object
```java
// ❌ MAL: Clase que hace demasiado
class SystemManager {
    void processPayment() { }
    void sendEmail() { }
    void generateReport() { }
    void validateUser() { }
    void logActivity() { }
    // 50 métodos más...
}
```

### 5.2 Spaghetti Code
```java
// ❌ MAL: Lógica entrelazada sin estructura
void processOrder() {
    if (user.isValid()) {
        if (inventory.check()) {
            if (payment.process()) {
                email.send();
                if (!shipping.available()) {
                    // código anidado 10 niveles...
                }
            }
        }
    }
}
```

### 5.3 Magic Numbers
```java
// ❌ MAL
if (status == 3) { }  // ¿Qué significa 3?

// ✅ BIEN
enum OrderStatus { PENDING, PROCESSING, SHIPPED, DELIVERED }
if (status == OrderStatus.SHIPPED) { }
```

## 6. Resumen Ejecutivo

**Patrones de Diseño:**
- Soluciones probadas a problemas recurrentes
- Vocabulario común entre desarrolladores
- Implementan principios SOLID en código concreto
- 23 patrones GoF en 3 categorías

**Patrones clave para SOLID:**
- **Strategy**: Algoritmos intercambiables (OCP)
- **Observer**: Comunicación uno-a-muchos (OCP)
- **Decorator**: Extender funcionalidad dinámicamente (SRP, OCP)
- **Factory Method**: Creación flexible (DIP, OCP)

## 7. Puntos Clave

✅ Los **patrones** son soluciones, los **principios** son guías de diseño  
✅ **Strategy** elimina condicionales complejos  
✅ **Observer** permite comunicación desacoplada  
✅ **Decorator** es alternativa flexible a herencia  
✅ **Factory Method** encapsula creación de objetos  
✅ Los patrones **implementan** principios SOLID  
✅ No abuses de patrones (KISS - Keep It Simple)  
❌ Evita **Singleton** cuando sea posible (usa DI)  
❌ No uses patrones si no resuelven un problema real
