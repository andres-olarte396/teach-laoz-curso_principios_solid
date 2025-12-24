# Acoplamiento y Cohesión en el Contexto de SRP

## Introducción

La comprensión de **acoplamiento** y **cohesión** es fundamental para aplicar correctamente el Single Responsibility Principle. Estas métricas nos ayudan a evaluar objetivamente la calidad del diseño.

## Cohesión

### Definición

**Cohesión** mide cuán relacionadas están las responsabilidades dentro de un módulo (clase, paquete).

**Alta cohesión** = Elementos fuertemente relacionados  
**Baja cohesión** = Elementos no relacionados

### Tipos de Cohesión (de peor a mejor)

1. **Cohesión Coincidental**: Elementos sin relación
2. **Cohesión Lógica**: Agrupados por categoría vaga
3. **Cohesión Temporal**: Ejecutados al mismo tiempo
4. **Cohesión Procedimental**: Ejecutados en secuencia
5. **Cohesión Comunicacional**: Operan sobre los mismos datos
6. **Cohesión Secuencial**: Salida de uno es entrada de otro
7. **Cohesión Funcional**: Todos contribuyen a una tarea única ✅ **IDEAL**

### Ejemplo de Cohesión Funcional

```java
// ✅ Alta cohesión funcional
public class PasswordHasher {
    private final MessageDigest digest;
    private final SecureRandom random;
    
    public String hash(String password) {
        byte[] salt = generateSalt();
        byte[] hash = computeHash(password, salt);
        return encodeResult(salt, hash);
    }
    
    private byte[] generateSalt() { /* ... */ }
    private byte[] computeHash(String password, byte[] salt) { /* ... */ }
    private String encodeResult(byte[] salt, byte[] hash) { /* ... */ }
}
```

**Todos los métodos contribuyen a una tarea: hashear passwords**.

### Ejemplo de Baja Cohesión

```java
// ❌ Baja cohesión
public class Utilities {
    public static String formatDate(Date date) { /* ... */ }
    public static double calculateTax(double amount) { /* ... */ }
    public static void sendEmail(String to, String subject) { /* ... */ }
    public static String hashPassword(String password) { /* ... */ }
}
```

**Métodos no relacionados agrupados arbitrariamente**.

## Acoplamiento

### Definición

**Acoplamiento** mide cuán dependiente es un módulo de otros módulos.

**Bajo acoplamiento** = Pocas dependencias  
**Alto acoplamiento** = Muchas dependencias

### Tipos de Acoplamiento (de peor a mejor)

1. **Acoplamiento de Contenido**: Modificación directa de datos internos
2. **Acoplamiento Común**: Variables globales compartidas
3. **Acoplamiento Externo**: Dependencia de formatos externos
4. **Acoplamiento de Control**: Pasar flags que controlan flujo
5. **Acoplamiento de Datos**: Pasar solo datos necesarios
6. **Acoplamiento de Mensaje**: Solo comunicación por interfaces ✅ **IDEAL**

### Ejemplo de Alto Acoplamiento

```java
// ❌ Alto acoplamiento
public class OrderProcessor {
    private MySQLDatabase database;  // Acoplado a implementación específica
    private StripePaymentGateway paymentGateway;  // Acoplado a Stripe
    private SendGridEmailService emailService;  // Acoplado a SendGrid
    
    public void processOrder(Order order) {
        // No se puede cambiar DB, gateway o email sin modificar esta clase
    }
}
```

### Ejemplo de Bajo Acoplamiento

```java
// ✅ Bajo acoplamiento
public class OrderProcessor {
    private final OrderRepository orderRepository;  // Interfaz
    private final PaymentService paymentService;    // Interfaz
    private final NotificationService notificationService;  // Interfaz
    
    public OrderProcessor(
        OrderRepository orderRepository,
        PaymentService paymentService,
        NotificationService notificationService
    ) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
        this.notificationService = notificationService;
    }
    
    public void processOrder(Order order) {
        // Puede cambiar implementaciones sin modificar esta clase
    }
}
```

## Relación con SRP

### Principio Fundamental

> **Alta cohesión** + **Bajo acoplamiento** = **Buen cumplimiento de SRP**

```
┌─────────────────────────────────────────┐
│  SRP CUMPLIDO                           │
│  ┌─────────────────┐                    │
│  │ Alta Cohesión   │ (una responsab.)   │
│  └─────────────────┘                    │
│  ┌─────────────────┐                    │
│  │ Bajo Acoplam.   │ (pocas depend.)    │
│  └─────────────────┘                    │
└─────────────────────────────────────────┘
```

### Caso de Estudio

```java
// ❌ Violación de SRP: Baja cohesión + Alto acoplamiento
public class UserService {
    private Database db;
    private EmailService emailService;
    private SmsService smsService;
    private FileStorage storage;
    private Analytics analytics;
    
    // Grupo 1. Autenticación (usa db, emailService)
    public void register(User user) { /* ... */ }
    public void login(String email, String password) { /* ... */ }
    
    // Grupo 2: Perfil (usa db, storage)
    public void uploadProfilePicture(User user, File image) { /* ... */ }
    
    // Grupo 3: Notificaciones (usa emailService, smsService)
    public void sendWelcome(User user) { /* ... */ }
    
    // Grupo 4: Analytics (usa analytics)
    public void trackActivity(User user, String action) { /* ... */ }
}
```

**Análisis**:
- **Cohesión**: Baja (4 grupos distintos)
- **Acoplamiento**: Alto (5 dependencias)
- **Conclusión**: Múltiples responsabilidades

### Refactorización

```java
// ✅ Alta cohesión + Bajo acoplamiento
public class AuthenticationService {
    private final UserRepository userRepository;
    private final PasswordHasher passwordHasher;
    
    public void register(User user, String password) {
        String hashedPassword = passwordHasher.hash(password);
        user.setPassword(hashedPassword);
        userRepository.save(user);
    }
    
    public boolean authenticate(String email, String password) {
        User user = userRepository.findByEmail(email);
        return passwordHasher.verify(password, user.getPassword());
    }
}

public class ProfileService {
    private final UserRepository userRepository;
    private final ImageStorage imageStorage;
    
    public void updateProfilePicture(User user, File image) {
        String url = imageStorage.upload(image);
        user.setProfilePictureUrl(url);
        userRepository.save(user);
    }
}

public class UserNotificationService {
    private final EmailService emailService;
    private final SmsService smsService;
    
    public void sendWelcome(User user) {
        emailService.send(user.getEmail(), "Welcome!");
        if (user.getPhone() != null) {
            smsService.send(user.getPhone(), "Welcome!");
        }
    }
}

public class UserActivityTracker {
    private final Analytics analytics;
    
    public void track(User user, String action) {
        analytics.track(user.getId(), action);
    }
}
```

**Mejoras**:
- **Cohesión**: Alta en cada clase (métodos relacionados)
- **Acoplamiento**: Bajo (2-3 dependencias por clase)
- **SRP**: Cumplido ✅

## Métricas de Cohesión

### LCOM (Lack of Cohesion of Methods)

**LCOM4**: Número de componentes conectados en el grafo de dependencias.

```
LCOM4 = 1  →  Perfecta cohesión ✅
LCOM4 > 1  →  Múltiples componentes → Separar
```

**Ejemplo**:

```java
public class Example {
    private int a;
    private int b;
    private String x;
    private String y;
    
    public void method1() { a = b + 1; }
    public void method2() { b = a * 2; }
    public void method3() { x = y + "test"; }
    public void method4() { y = x.toUpperCase(); }
}
```

**Grafo de dependencias**:
```
method1 ←→ method2  (comparten a, b)
method3 ←→ method4  (comparten x, y)

LCOM4 = 2  →  Dos componentes separados
```

**Refactorización**:

```java
public class NumericCalculator {
    private int a;
    private int b;
    public void method1() { a = b + 1; }
    public void method2() { b = a * 2; }
}

public class StringProcessor {
    private String x;
    private String y;
    public void method3() { x = y + "test"; }
    public void method4() { y = x.toUpperCase(); }
}
```

### TCC (Tight Class Cohesion)

```
TCC = (Pares de métodos que comparten variables) / (Total de pares)

TCC ≥ 0.5  →  Cohesión aceptable
TCC < 0.5  →  Revisar diseño
```

## Métricas de Acoplamiento

### Afferent Coupling (Ca)

Número de clases externas que dependen de esta clase.

```
Ca alto  →  Clase muy usada (cuidado al cambiar)
Ca bajo  →  Clase poco usada
```

### Efferent Coupling (Ce)

Número de clases de las que esta clase depende.

```
Ce alto  →  Muchas dependencias (riesgo)
Ce bajo  →  Pocas dependencias ✅
```

### Inestabilidad (I)

```
I = Ce / (Ce + Ca)

I = 0  →  Máxima estabilidad (muchas dependen de ella)
I = 1  →  Máxima inestabilidad (depende de muchas)
```

**Regla**: Clases de dominio deben tener I cercano a 0.

## Principios Derivados

### Ley de Demeter (Principle of Least Knowledge)

> "Habla solo con tus amigos directos"

```java
// ❌ Viola Ley de Demeter
public void processOrder(Order order) {
    String city = order.getCustomer().getAddress().getCity();
    // Cadena larga → Alto acoplamiento
}

// ✅ Cumple Ley de Demeter
public void processOrder(Order order) {
    String city = order.getCustomerCity();
    // Delegación → Bajo acoplamiento
}
```

### Common Closure Principle (CCP)

> "Las clases que cambian juntas, permanecen juntas"

Si dos clases siempre se modifican por las mismas razones, deberían estar en el mismo módulo.

### Common Reuse Principle (CRP)

> "Las clases que se usan juntas, permanecen juntas"

Si dos clases siempre se usan simultáneamente, deberían estar en el mismo módulo.

## Herramientas de Medición

### JDepend

```java
import jdepend.framework.JDepend;

JDepend jdepend = new JDepend();
jdepend.addDirectory("/path/to/classes");
Collection packages = jdepend.analyze();

for (JavaPackage pkg : packages) {
    System.out.println("Afferent: " + pkg.afferentCoupling());
    System.out.println("Efferent: " + pkg.efferentCoupling());
    System.out.println("Instability: " + pkg.instability());
}
```

### SonarQube

Dashboard → Measures → Complexity:
- LCOM4
- Coupling Between Objects (CBO)
- Response For Class (RFC)

## Resumen

| Métrica | Objetivo | Indica SRP cuando... |
|---------|----------|----------------------|
| **Cohesión (LCOM4)** | = 1 | Un solo componente conectado |
| **Acoplamiento (Ce)** | < 5 | Pocas dependencias |
| **Inestabilidad (I)** | 0-0.3 (dominio) | Estable, no depende de muchos |
| **TCC** | > 0.5 | Métodos comparten datos |

**Regla de oro**: Alta cohesión interna, bajo acoplamiento externo.
