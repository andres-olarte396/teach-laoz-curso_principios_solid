# Ejercicios: Identificación Sistemática de Responsabilidades

## ⭐ Nivel 1. Análisis LCOM

### Ejercicio 1.1. Calcular LCOM Manualmente

Calcula el LCOM de esta clase e identifica grupos de cohesión:

```java
public class ProductCatalog {
    private List<Product> products;
    private Database db;
    private PriceCalculator calculator;
    private EmailService emailService;
    
    // Método 1
    public List<Product> searchByName(String name) {
        return products.stream()
            .filter(p -> p.getName().contains(name))
            .collect(Collectors.toList());
    }
    
    // Método 2
    public List<Product> searchByCategory(String category) {
        return products.stream()
            .filter(p -> p.getCategory().equals(category))
            .collect(Collectors.toList());
    }
    
    // Método 3
    public void saveProduct(Product product) {
        db.execute("INSERT INTO products...");
    }
    
    // Método 4
    public double calculateTotalPrice(List<Product> items) {
        return calculator.calculateWithTax(items);
    }
    
    // Método 5
    public void sendCatalogByEmail(String email) {
        String catalog = products.stream()
            .map(Product::getName)
            .collect(Collectors.joining("\n"));
        emailService.send(email, "Catalog", catalog);
    }
}
```

**Tareas**:
1. Crea una tabla mostrando qué variables usa cada método
2. Identifica grupos de cohesión
3. Calcula LCOM (simplificado)
4. Propón refactorización basada en grupos

**Solución esperada**: Identificar 4 grupos (búsqueda, persistencia, cálculo, notificación).

---

## ⭐⭐ Nivel 2: Análisis Multi-Técnica

### Ejercicio 2.1. Auditoría Completa

Analiza esta clase usando las 7 técnicas del contenido:

```java
public class CustomerService {
    private Database database;
    private EmailService emailService;
    private SMSService smsService;
    private LoyaltyPointsCalculator loyaltyCalculator;
    private InvoiceGenerator invoiceGenerator;
    private CreditScoreProvider creditProvider;
    
    public Customer createCustomer(String name, String email, String phone) {
        // Validación
        if (name == null || name.length() < 3) {
            throw new ValidationException("Invalid name");
        }
        if (!email.contains("@")) {
            throw new ValidationException("Invalid email");
        }
        
        // Verificar duplicados
        Customer existing = database.findByEmail(email);
        if (existing != null) {
            throw new DuplicateException("Customer exists");
        }
        
        // Crear customer
        Customer customer = new Customer(name, email, phone);
        customer.setJoinDate(LocalDate.now());
        
        // Guardar
        database.save(customer);
        
        // Enviar welcome
        emailService.send(email, "Welcome!", "Thanks for joining");
        smsService.send(phone, "Welcome to our service!");
        
        // Log
        System.out.println("Customer created: " + customer.getId());
        
        return customer;
    }
    
    public void makePurchase(Customer customer, double amount) {
        // Verificar crédito
        int creditScore = creditProvider.getScore(customer.getId());
        if (creditScore < 600 && amount > 1000) {
            throw new CreditException("Insufficient credit");
        }
        
        // Calcular puntos
        int points = loyaltyCalculator.calculatePoints(amount);
        customer.addLoyaltyPoints(points);
        
        // Actualizar
        database.update(customer);
        
        // Enviar confirmación
        emailService.send(customer.getEmail(), "Purchase confirmed", 
                         "Amount: $" + amount);
    }
    
    public void generateMonthlyInvoice(Customer customer) {
        List<Purchase> purchases = database.findPurchases(customer.getId());
        String invoice = invoiceGenerator.generate(customer, purchases);
        
        emailService.send(customer.getEmail(), "Monthly Invoice", invoice);
    }
}
```

**Tareas**:
1. **LCOM**: Identifica grupos de variables/métodos
2. **Dependencias**: Lista dominios importados
3. **Actores**: Identifica stakeholders para cada método
4. **Nombres**: Evalúa el nombre de la clase
5. **Testing**: Estima complejidad de tests (número de mocks)
6. **Propuesta**: Diseña clases refactorizadas

---

## ⭐⭐⭐ Nivel 3: Refactorización Guiada por Métricas

### Ejercicio 3.1. Sistema de Reservas

Refactoriza el siguiente sistema aplicando análisis LCOM y de actores:

```java
public class ReservationSystem {
    private Database db;
    private EmailService emailService;
    private PaymentGateway paymentGateway;
    private CalendarService calendarService;
    private SMSService smsService;
    
    public Reservation createReservation(
        User user, 
        Room room, 
        LocalDateTime startTime, 
        LocalDateTime endTime
    ) {
        // Validar disponibilidad
        List<Reservation> existing = db.findReservations(room, startTime, endTime);
        if (!existing.isEmpty()) {
            throw new RoomNotAvailableException();
        }
        
        // Validar horario de negocio
        if (startTime.getHour() < 8 || endTime.getHour() > 22) {
            throw new InvalidTimeException("Business hours: 8 AM - 10 PM");
        }
        
        // Calcular precio
        long hours = Duration.between(startTime, endTime).toHours();
        double price = room.getHourlyRate() * hours;
        
        // Aplicar descuento
        if (user.isMember()) {
            price *= 0.9; // 10% descuento
        }
        
        // Procesar pago
        PaymentResult payment = paymentGateway.charge(user.getPaymentInfo(), price);
        if (!payment.isSuccessful()) {
            throw new PaymentFailedException();
        }
        
        // Crear reserva
        Reservation reservation = new Reservation();
        reservation.setUser(user);
        reservation.setRoom(room);
        reservation.setStartTime(startTime);
        reservation.setEndTime(endTime);
        reservation.setPrice(price);
        reservation.setPaymentId(payment.getTransactionId());
        
        // Guardar
        db.save(reservation);
        
        // Agregar a calendario del usuario
        calendarService.addEvent(user.getCalendarId(), 
                                "Room: " + room.getName(), 
                                startTime, 
                                endTime);
        
        // Notificar
        emailService.send(user.getEmail(), 
                         "Reservation Confirmed", 
                         formatConfirmation(reservation));
        
        smsService.send(user.getPhone(), 
                       "Reservation confirmed for " + startTime);
        
        // Log
        System.out.println("Reservation created: " + reservation.getId());
        
        return reservation;
    }
    
    public void cancelReservation(Reservation reservation) {
        // Verificar política de cancelación
        long hoursUntilStart = Duration.between(
            LocalDateTime.now(), 
            reservation.getStartTime()
        ).toHours();
        
        if (hoursUntilStart < 24) {
            throw new LateCancellationException("Cancel 24h in advance");
        }
        
        // Reembolsar
        paymentGateway.refund(reservation.getPaymentId(), reservation.getPrice());
        
        // Actualizar estado
        reservation.setStatus(ReservationStatus.CANCELLED);
        db.update(reservation);
        
        // Remover de calendario
        calendarService.removeEvent(
            reservation.getUser().getCalendarId(), 
            reservation.getId()
        );
        
        // Notificar
        emailService.send(reservation.getUser().getEmail(), 
                         "Reservation Cancelled", 
                         "Refund processed");
    }
    
    private String formatConfirmation(Reservation reservation) {
        return "Room: " + reservation.getRoom().getName() + "\n" +
               "Time: " + reservation.getStartTime() + " - " + reservation.getEndTime() + "\n" +
               "Price: $" + reservation.getPrice();
    }
}
```

**Entregables**:
1. Tabla LCOM completa
2. Diagrama de actores
3. Mínimo 8 clases extraídas
4. Tests unitarios para 5 clases
5. Comparación métricas antes/después

---

## ⭐⭐⭐⭐ Nivel 4: Herramientas Automatizadas

### Ejercicio 4.1. Análisis con SonarQube

**Requisitos previos**: Docker instalado

1. Ejecuta SonarQube:
```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:latest
```

2. Analiza este proyecto:
```bash
git clone https://github.com/example/legacy-ecommerce
cd legacy-ecommerce
mvn sonar:sonar -Dsonar.host.url=http://localhost:9000
```

3. Tareas:
   - Identifica las 5 clases con mayor LCOM
   - Documenta violaciones de SRP encontradas
   - Propón refactorización para la clase con peor métrica
   - Implementa la refactorización
   - Vuelve a ejecutar análisis y compara resultados

**Métricas esperadas**:
- Reducción LCOM > 50%
- Aumento en cobertura de tests > 20%
- Reducción en complejidad ciclomática > 30%

---

## Resumen de Ejercicios

| Nivel | Enfoque | Tiempo |
|-------|---------|--------|
| ⭐ | Cálculo manual de LCOM | 30 min |
| ⭐⭐ | Análisis con 7 técnicas | 60 min |
| ⭐⭐⭐ | Refactorización completa | 120 min |
| ⭐⭐⭐⭐ | Herramientas automatizadas | 90 min |
