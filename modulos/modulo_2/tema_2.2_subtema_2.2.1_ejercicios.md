# Ejercicios: God Classes y Feature Envy

## ⭐ Nivel 1. Identificación

### Ejercicio 1.1. Detectar God Class

Analiza esta clase y justifica si es una God Class:

```java
public class BookstoreManager {
    private Database db;
    private EmailService emailService;
    private PaymentGateway paymentGateway;
    
    public void addBook(Book book) {
        if (book.getTitle() == null) throw new ValidationException();
        db.save(book);
    }
    
    public void processPurchase(Customer customer, Book book) {
        PaymentResult result = paymentGateway.charge(customer, book.getPrice());
        if (result.isSuccess()) {
            db.recordPurchase(customer, book);
            emailService.send(customer.getEmail(), "Purchase confirmed");
        }
    }
    
    public List<Book> searchBooks(String query) {
        return db.findBooks(query);
    }
    
    public void sendNewsletter(List<Customer> customers) {
        for (Customer customer : customers) {
            emailService.send(customer.getEmail(), "Newsletter");
        }
    }
}
```

**Tareas**: Cuenta métodos, dependencias, responsabilidades. ¿Es God Class?

### Ejercicio 1.2: Detectar Feature Envy

Identifica Feature Envy en este código:

```java
public class ShoppingCart {
    public double calculateDiscount(Customer customer) {
        double discount = 0;
        if (customer.getMembershipLevel() == MembershipLevel.GOLD) {
            discount = 0.2;
        } else if (customer.getMembershipLevel() == MembershipLevel.SILVER) {
            discount = 0.1;
        }
        return discount;
    }
}
```

**Tarea**: ¿Qué clase "envidia" `ShoppingCart`? Propón solución.

---

## ⭐⭐ Nivel 2: Refactorización

### Ejercicio 2.1. Descomponer God Class

Refactoriza esta God Class en clases especializadas:

```java
public class LibrarySystem {
    private Database db;
    private EmailService emailService;
    private FineCalculator fineCalculator;
    
    public void registerMember(String name, String email) {
        Member member = new Member(name, email);
        db.saveMember(member);
        emailService.send(email, "Welcome!");
    }
    
    public void borrowBook(Member member, Book book) {
        if (book.isAvailable()) {
            Loan loan = new Loan(member, book, LocalDate.now().plusWeeks(2));
            db.saveLoan(loan);
            book.setAvailable(false);
            db.updateBook(book);
        }
    }
    
    public void returnBook(Loan loan) {
        loan.setReturnDate(LocalDate.now());
        db.updateLoan(loan);
        loan.getBook().setAvailable(true);
        db.updateBook(loan.getBook());
        
        if (loan.isOverdue()) {
            double fine = fineCalculator.calculate(loan);
            emailService.send(loan.getMember().getEmail(), "Fine: $" + fine);
        }
    }
    
    public void sendOverdueNotices() {
        List<Loan> overdueLoans = db.findOverdueLoans();
        for (Loan loan : overdueLoans) {
            emailService.send(loan.getMember().getEmail(), "Book overdue!");
        }
    }
}
```

**Entregable**: Mínimo 5 clases con responsabilidades únicas.

### Ejercicio 2.2: Corregir Feature Envy

Refactoriza eliminando Feature Envy:

```java
public class InvoiceFormatter {
    public String format(Invoice invoice) {
        StringBuilder sb = new StringBuilder();
        sb.append("Customer: " + invoice.getCustomer().getName() + "\n");
        sb.append("Address: " + invoice.getCustomer().getAddress().getStreet() + "\n");
        sb.append("City: " + invoice.getCustomer().getAddress().getCity() + "\n");
        
        for (LineItem item : invoice.getLineItems()) {
            sb.append(item.getProduct().getName() + " - $" + item.getPrice() + "\n");
        }
        
        return sb.toString();
    }
}
```

---

## ⭐⭐⭐ Nivel 3: Proyecto Completo

Diseña un sistema de gestión de hospital sin God Classes ni Feature Envy. Implementa: registro de pacientes, citas médicas, historiales clínicos, facturación. Mínimo 12 clases cohesivas.

---

## ⭐⭐⭐⭐ Nivel 4: Análisis de Código Real

Analiza un proyecto open source (Spring Petclinic o similar), identifica God Classes usando métricas, propón refactorización y mide mejora con SonarQube.
