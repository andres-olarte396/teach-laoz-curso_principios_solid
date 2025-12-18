# Decorator Pattern para Añadir Funcionalidad

## Definición

**Decorator Pattern** permite añadir responsabilidades a un objeto dinámicamente, sin modificar su clase original. Es una alternativa flexible a la herencia.

## Estructura

```java
// Componente base
public interface Coffee {
    String getDescription();
    double getCost();
}

// Implementación concreta
public class SimpleCoffee implements Coffee {
    public String getDescription() {
        return "Simple coffee";
    }
    
    public double getCost() {
        return 2.0;
    }
}

// Decorador abstracto
public abstract class CoffeeDecorator implements Coffee {
    protected Coffee decoratedCoffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.decoratedCoffee = coffee;
    }
    
    public String getDescription() {
        return decoratedCoffee.getDescription();
    }
    
    public double getCost() {
        return decoratedCoffee.getCost();
    }
}

// Decoradores concretos
public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }
    
    public String getDescription() {
        return decoratedCoffee.getDescription() + ", Milk";
    }
    
    public double getCost() {
        return decoratedCoffee.getCost() + 0.5;
    }
}

public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }
    
    public String getDescription() {
        return decoratedCoffee.getDescription() + ", Sugar";
    }
    
    public double getCost() {
        return decoratedCoffee.getCost() + 0.2;
    }
}

public class WhipDecorator extends CoffeeDecorator {
    public WhipDecorator(Coffee coffee) {
        super(coffee);
    }
    
    public String getDescription() {
        return decoratedCoffee.getDescription() + ", Whip";
    }
    
    public double getCost() {
        return decoratedCoffee.getCost() + 0.7;
    }
}

// Uso
Coffee coffee = new SimpleCoffee();
System.out.println(coffee.getDescription() + " $" + coffee.getCost());
// Output: Simple coffee $2.0

coffee = new MilkDecorator(coffee);
System.out.println(coffee.getDescription() + " $" + coffee.getCost());
// Output: Simple coffee, Milk $2.5

coffee = new SugarDecorator(coffee);
System.out.println(coffee.getDescription() + " $" + coffee.getCost());
// Output: Simple coffee, Milk, Sugar $2.7

coffee = new WhipDecorator(coffee);
System.out.println(coffee.getDescription() + " $" + coffee.getCost());
// Output: Simple coffee, Milk, Sugar, Whip $3.4
```

## OCP con Decorator

**Cerrado para modificación**: `SimpleCoffee` nunca cambia.

**Abierto para extensión**: Añadir `CaramelDecorator` no requiere modificar clases existentes.

## Caso de Estudio: Sistema de Notificaciones

### Problema

Necesitas enviar notificaciones por múltiples canales (email, SMS, Slack) con opciones de cifrado, compresión, logging.

### Solución con Decorator

```java
public interface Notifier {
    void send(String message);
}

// Componente base
public class EmailNotifier implements Notifier {
    private String email;
    
    public EmailNotifier(String email) {
        this.email = email;
    }
    
    public void send(String message) {
        System.out.println("Sending email to " + email + ": " + message);
    }
}

// Decoradores
public abstract class NotifierDecorator implements Notifier {
    protected Notifier wrappee;
    
    public NotifierDecorator(Notifier notifier) {
        this.wrappee = notifier;
    }
    
    public void send(String message) {
        wrappee.send(message);
    }
}

public class SmsNotifier extends NotifierDecorator {
    private String phone;
    
    public SmsNotifier(Notifier notifier, String phone) {
        super(notifier);
        this.phone = phone;
    }
    
    public void send(String message) {
        super.send(message); // Envía vía canal anterior
        System.out.println("Sending SMS to " + phone + ": " + message);
    }
}

public class SlackNotifier extends NotifierDecorator {
    private String channel;
    
    public SlackNotifier(Notifier notifier, String channel) {
        super(notifier);
        this.channel = channel;
    }
    
    public void send(String message) {
        super.send(message);
        System.out.println("Sending Slack to " + channel + ": " + message);
    }
}

public class EncryptionDecorator extends NotifierDecorator {
    public EncryptionDecorator(Notifier notifier) {
        super(notifier);
    }
    
    public void send(String message) {
        String encrypted = encrypt(message);
        super.send(encrypted);
    }
    
    private String encrypt(String message) {
        return "ENCRYPTED(" + message + ")";
    }
}

public class LoggingDecorator extends NotifierDecorator {
    public LoggingDecorator(Notifier notifier) {
        super(notifier);
    }
    
    public void send(String message) {
        System.out.println("LOG: Sending message at " + LocalDateTime.now());
        super.send(message);
        System.out.println("LOG: Message sent successfully");
    }
}

// Uso flexible
Notifier notifier = new EmailNotifier("admin@example.com");

// Añadir SMS
notifier = new SmsNotifier(notifier, "+1234567890");

// Añadir cifrado
notifier = new EncryptionDecorator(notifier);

// Añadir logging
notifier = new LoggingDecorator(notifier);

notifier.send("Hello World");

// Output:
// LOG: Sending message at 2024-01-15T10:30:00
// Sending email to admin@example.com: ENCRYPTED(Hello World)
// Sending SMS to +1234567890: ENCRYPTED(Hello World)
// LOG: Message sent successfully
```

**Extensión sin modificación**: Añadir `CompressionDecorator` o `FacebookNotifier` no requiere tocar clases existentes.

## Decorator vs Herencia

### Con Herencia (inflexible)

```java
class EmailNotifier { }
class EmailSmsNotifier extends EmailNotifier { }
class EmailSmsSlackNotifier extends EmailSmsNotifier { }
class EncryptedEmailSmsSlackNotifier extends EmailSmsSlackNotifier { }
```

**Explosión de clases**: Para n características, necesitas 2^n clases.

### Con Decorator (flexible)

```java
Notifier n = new EmailNotifier();
n = new SmsNotifier(n);
n = new SlackNotifier(n);
n = new EncryptionDecorator(n);
```

**Composición dinámica**: Combina características en runtime.

## Decorators en Java Estándar

```java
// java.io usa decorators
InputStream input = new FileInputStream("file.txt");
input = new BufferedInputStream(input); // Decorator: buffering
input = new GZIPInputStream(input);      // Decorator: descompresión
input = new DataInputStream(input);      // Decorator: tipos primitivos

// Puedes apilar tantos decorators como necesites
```

## Decorator con Múltiples Métodos

```java
public interface DataSource {
    void writeData(String data);
    String readData();
}

public class FileDataSource implements DataSource {
    private String filename;
    
    public void writeData(String data) {
        // Escribir a archivo
    }
    
    public String readData() {
        // Leer de archivo
        return "file data";
    }
}

public class CompressionDecorator extends DataSourceDecorator {
    public void writeData(String data) {
        String compressed = compress(data);
        super.writeData(compressed);
    }
    
    public String readData() {
        String data = super.readData();
        return decompress(data);
    }
}

public class EncryptionDecorator extends DataSourceDecorator {
    public void writeData(String data) {
        String encrypted = encrypt(data);
        super.writeData(encrypted);
    }
    
    public String readData() {
        String data = super.readData();
        return decrypt(data);
    }
}

// Uso: compression + encryption
DataSource source = new FileDataSource("data.txt");
source = new CompressionDecorator(source);
source = new EncryptionDecorator(source);

source.writeData("Sensitive data");
// Flujo: encrypt → compress → write to file

String data = source.readData();
// Flujo: read from file → decompress → decrypt
```

## Resumen

**Decorator** = Añadir responsabilidades a objetos dinámicamente mediante composición.

**OCP**: Nuevas funcionalidades = nuevos decoradores, sin modificar componentes existentes.

**Ventaja sobre herencia**: Combinaciones flexibles en runtime sin explosión de clases.

**Cuándo usar**: Múltiples características opcionales y combinables.
