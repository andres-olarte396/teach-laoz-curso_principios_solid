# Extract Class y Extract Method - Refactoring hacia SRP

## Introducción

**Extract Class** y **Extract Method** son las dos técnicas de refactoring más fundamentales para lograr el Single Responsibility Principle. Permiten descomponer clases y métodos complejos en componentes cohesivos.

## Extract Method

### Cuándo Aplicar

- Método >20 líneas
- Lógica compleja o anidada
- Comentarios explicando secciones
- Código duplicado

### Pasos

1. Identifica fragmento cohesivo
2. Crea método nuevo con nombre descriptivo
3. Mueve código al nuevo método
4. Reemplaza con llamada

### Ejemplo Básico

```java
// Antes
public void processOrder(Order order) {
    // Validación
    if (order.getItems().isEmpty()) throw new IllegalArgumentException();
    if (order.getCustomer() == null) throw new IllegalArgumentException();
    
    // Cálculo
    double total = 0;
    for (OrderItem item : order.getItems()) {
        total += item.getPrice() * item.getQuantity();
    }
    double tax = total * 0.15;
    total += tax;
    
    // Persistencia
    database.save(order);
}

// Después
public void processOrder(Order order) {
    validateOrder(order);
    double total = calculateTotal(order);
    saveOrder(order);
}

private void validateOrder(Order order) {
    if (order.getItems().isEmpty()) 
        throw new IllegalArgumentException("No items");
    if (order.getCustomer() == null) 
        throw new IllegalArgumentException("No customer");
}

private double calculateTotal(Order order) {
    double subtotal = order.getItems().stream()
        .mapToDouble(item -> item.getPrice() * item.getQuantity())
        .sum();
    return subtotal * 1.15; // +15% tax
}

private void saveOrder(Order order) {
    database.save(order);
}
```

**Beneficios**: Método principal legible, métodos reutilizables, testeo granular.

## Extract Class

### Cuándo Aplicar

- Clase >200 líneas
- LCOM > 0
- Variables usadas solo por subset de métodos
- Dos o más conceptos en una clase

### Pasos

1. Identifica grupo cohesivo de métodos/variables
2. Crea nueva clase
3. Mueve variables y métodos relacionados
4. Establece relación entre clases (composición/delegación)
5. Actualiza clientes

### Ejemplo: De God Class a Clases Especializadas

```java
// ANTES: God Class
public class User {
    // Datos de perfil
    private String name;
    private String email;
    private String phone;
    
    // Datos de autenticación
    private String passwordHash;
    private String salt;
    private LocalDateTime lastLogin;
    
    // Preferencias
    private String language;
    private String timezone;
    private boolean emailNotifications;
    
    // Métodos de perfil
    public void updateProfile(String name, String phone) { }
    public String getFullContactInfo() { }
    
    // Métodos de autenticación
    public boolean authenticate(String password) { }
    public void updatePassword(String newPassword) { }
    
    // Métodos de preferencias
    public void setLanguage(String lang) { }
    public void toggleNotifications() { }
}

// DESPUÉS: Clases especializadas

// Clase 1. Perfil
public class UserProfile {
    private String name;
    private String email;
    private String phone;
    
    public void update(String name, String phone) {
        this.name = name;
        this.phone = phone;
    }
    
    public String getContactInfo() {
        return String.format("%s\n%s\n%s", name, email, phone);
    }
}

// Clase 2: Credenciales
public class UserCredentials {
    private String passwordHash;
    private String salt;
    private LocalDateTime lastLogin;
    
    public boolean authenticate(String password) {
        String hash = hashPassword(password, salt);
        return hash.equals(passwordHash);
    }
    
    public void updatePassword(String newPassword) {
        this.salt = generateSalt();
        this.passwordHash = hashPassword(newPassword, salt);
    }
}

// Clase 3: Preferencias
public class UserPreferences {
    private String language;
    private String timezone;
    private boolean emailNotifications;
    
    public void setLanguage(String language) {
        this.language = language;
    }
    
    public void toggleNotifications() {
        this.emailNotifications = !this.emailNotifications;
    }
}

// Clase agregadora (si es necesaria)
public class User {
    private final UserProfile profile;
    private final UserCredentials credentials;
    private final UserPreferences preferences;
    
    public User(UserProfile profile, UserCredentials credentials, UserPreferences preferences) {
        this.profile = profile;
        this.credentials = credentials;
        this.preferences = preferences;
    }
    
    // Delegación
    public boolean authenticate(String password) {
        return credentials.authenticate(password);
    }
    
    public void updateProfile(String name, String phone) {
        profile.update(name, phone);
    }
}
```

## Patrón: Extract Service Class

Cuando extraemos lógica de negocio compleja:

```java
// Antes
public class Order {
    private List<OrderItem> items;
    private Customer customer;
    
    public double calculateTotal() {
        double subtotal = items.stream()
            .mapToDouble(i -> i.getPrice() * i.getQuantity())
            .sum();
        
        double discount = 0;
        if (customer.isVip()) discount = subtotal * 0.1;
        if (subtotal > 100) discount += 10;
        
        double tax = (subtotal - discount) * 0.15;
        double shipping = calculateShipping();
        
        return subtotal - discount + tax + shipping;
    }
    
    private double calculateShipping() {
        // Lógica compleja...
        return 0;
    }
}

// Después: Extract Service
public class OrderPricingService {
    private final DiscountCalculator discountCalculator;
    private final TaxCalculator taxCalculator;
    private final ShippingCalculator shippingCalculator;
    
    public OrderPrice calculate(Order order) {
        double subtotal = calculateSubtotal(order);
        double discount = discountCalculator.calculate(order);
        double tax = taxCalculator.calculate(subtotal - discount);
        double shipping = shippingCalculator.calculate(order);
        
        return new OrderPrice(subtotal, discount, tax, shipping);
    }
}
```

## Extract Parameter Object

Cuando métodos tienen muchos parámetros:

```java
// Antes
public void createUser(
    String name, 
    String email, 
    String phone, 
    String address, 
    String city, 
    String country, 
    String zip
) { }

// Después
public class UserRegistrationData {
    private final String name;
    private final String email;
    private final String phone;
    private final Address address;
}

public void createUser(UserRegistrationData data) { }
```

## Resumen de Técnicas

| Técnica | Cuándo | Resultado |
|---------|--------|-----------|
| Extract Method | Método largo/complejo | Métodos cohesivos |
| Extract Class | Clase con múltiples responsabilidades | Clases especializadas |
| Extract Service | Lógica de negocio compleja | Servicios reutilizables |
| Extract Parameter Object | Muchos parámetros | Objetos de valor |

Cada técnica reduce complejidad y mejora adherencia a SRP.
