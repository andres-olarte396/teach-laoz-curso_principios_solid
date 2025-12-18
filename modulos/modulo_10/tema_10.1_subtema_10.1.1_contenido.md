# Proyecto Final: Sistema de Biblioteca Digital

## Requisitos Funcionales

### Gestión de Usuarios
- Registro y autenticación
- Perfiles (estudiante, profesor, administrador)
- Historial de préstamos

### Catálogo de Libros
- Búsqueda y filtrado
- Categorías y etiquetas
- Reseñas y calificaciones

### Préstamos
- Reservar libros
- Préstamos físicos y digitales
- Multas por retraso

### Notificaciones
- Email, SMS, push
- Recordatorios de devolución
- Nuevos libros disponibles

## Requisitos Técnicos

### Arquitectura Hexagonal

```
[REST API]    [CLI]
     ↓          ↓
  [Adapters]
     ↓
   [Ports]
     ↓
   [CORE]
- Domain Models
- Use Cases
- Business Rules
     ↓
   [Ports]
     ↓
  [Adapters]
     ↓
[MySQL] [MongoDB] [Email]
```

### Aplicar SOLID

**SRP**: Módulos por responsabilidad (users, books, loans, notifications)

**OCP**: Estrategias extensibles (notification channels, loan policies)

**LSP**: Jerarquías bien diseñadas (User types, Book formats)

**ISP**: Interfaces segregadas por rol (BookReader, BookWriter, LoanManager)

**DIP**: Inversión completa (DI container, abstracciones)

## Estructura de Módulos

```java
// DOMAIN
class Book {
    private String isbn;
    private String title;
    private BookFormat format; // PHYSICAL, DIGITAL
    private BookStatus status; // AVAILABLE, BORROWED, RESERVED
}

class Loan {
    private Long id;
    private Book book;
    private User user;
    private LocalDate borrowedDate;
    private LocalDate dueDate;
    
    public boolean isOverdue() {
        return LocalDate.now().isAfter(dueDate);
    }
    
    public double calculateFine() {
        if (!isOverdue()) return 0;
        long daysOverdue = ChronoUnit.DAYS.between(dueDate, LocalDate.now());
        return daysOverdue * 0.50; // $0.50 por día
    }
}

// PORTS (abstracciones)
interface BookRepository {
    Optional<Book> findByIsbn(String isbn);
    List<Book> findAvailable();
}

interface LoanRepository {
    void save(Loan loan);
    List<Loan> findByUser(User user);
}

interface NotificationService {
    void send(User user, String message);
}

// USE CASES (SRP)
class BorrowBookUseCase {
    private final BookRepository bookRepo;
    private final LoanRepository loanRepo;
    private final NotificationService notifications;
    
    public Loan execute(String isbn, User user) {
        Book book = bookRepo.findByIsbn(isbn)
            .orElseThrow(() -> new BookNotFoundException());
        
        if (book.getStatus() != BookStatus.AVAILABLE) {
            throw new BookNotAvailableException();
        }
        
        Loan loan = new Loan(book, user, LocalDate.now(), LocalDate.now().plusDays(14));
        book.setStatus(BookStatus.BORROWED);
        
        loanRepo.save(loan);
        bookRepo.save(book);
        
        notifications.send(user, "Book borrowed: " + book.getTitle());
        
        return loan;
    }
}

// ADAPTERS
class JpaBookRepository implements BookRepository {
    @Autowired
    private BookJpaRepository jpaRepo;
    
    public Optional<Book> findByIsbn(String isbn) {
        return jpaRepo.findByIsbn(isbn).map(BookEntity::toDomain);
    }
}

class CompositeNotificationService implements NotificationService {
    private List<NotificationService> services;
    
    public void send(User user, String message) {
        services.forEach(service -> service.send(user, message));
    }
}

// CONTROLLERS
@RestController
@RequestMapping("/api/loans")
class LoanController {
    private final BorrowBookUseCase borrowBookUseCase;
    
    @PostMapping
    public ResponseEntity<LoanDTO> borrowBook(@RequestBody BorrowRequest request) {
        Loan loan = borrowBookUseCase.execute(request.getIsbn(), getCurrentUser());
        return ResponseEntity.ok(LoanDTO.fromDomain(loan));
    }
}
```

## Entregables

1. **Diseño**: Diagrama de arquitectura hexagonal
2. **Código**: Implementación completa con tests
3. **Documentación**: Decisiones de diseño, principios aplicados
4. **Métricas**: Cobertura de tests, complejidad, acoplamiento

## Resumen

**Proyecto integrador** = Aplicación real de todos los principios SOLID + arquitectura hexagonal.
