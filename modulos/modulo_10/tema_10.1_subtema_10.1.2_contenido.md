# Implementación Multi-Lenguaje

## Java Implementation

```java
// Domain
public class Book {
    private final String isbn;
    private final String title;
    private BookStatus status;
    
    public Book(String isbn, String title) {
        this.isbn = Objects.requireNonNull(isbn);
        this.title = Objects.requireNonNull(title);
        this.status = BookStatus.AVAILABLE;
    }
    
    public boolean canBeBorrowed() {
        return status == BookStatus.AVAILABLE;
    }
}

// Port
public interface BookRepository {
    Optional<Book> findByIsbn(String isbn);
    List<Book> findAvailable();
    void save(Book book);
}

// Use Case
public class BorrowBookUseCase {
    private final BookRepository bookRepository;
    private final LoanRepository loanRepository;
    
    public BorrowBookUseCase(BookRepository bookRepository, 
                             LoanRepository loanRepository) {
        this.bookRepository = Objects.requireNonNull(bookRepository);
        this.loanRepository = Objects.requireNonNull(loanRepository);
    }
    
    public Loan execute(String isbn, User user) {
        Book book = bookRepository.findByIsbn(isbn)
            .orElseThrow(() -> new BookNotFoundException(isbn));
        
        if (!book.canBeBorrowed()) {
            throw new BookNotAvailableException(isbn);
        }
        
        Loan loan = Loan.create(book, user);
        book.markAsBorrowed();
        
        loanRepository.save(loan);
        bookRepository.save(book);
        
        return loan;
    }
}

// Tests
@Test
void shouldBorrowAvailableBook() {
    // Arrange
    BookRepository mockRepo = mock(BookRepository.class);
    LoanRepository mockLoanRepo = mock(LoanRepository.class);
    Book book = new Book("123", "Clean Code");
    when(mockRepo.findByIsbn("123")).thenReturn(Optional.of(book));
    
    BorrowBookUseCase useCase = new BorrowBookUseCase(mockRepo, mockLoanRepo);
    
    // Act
    Loan loan = useCase.execute("123", new User("user@test.com"));
    
    // Assert
    assertNotNull(loan);
    verify(mockLoanRepo).save(any(Loan.class));
}
```

## TypeScript Implementation

```typescript
// Domain
class Book {
    constructor(
        public readonly isbn: string,
        public readonly title: string,
        private status: BookStatus = BookStatus.AVAILABLE
    ) {}
    
    canBeBorrowed(): boolean {
        return this.status === BookStatus.AVAILABLE;
    }
    
    markAsBorrowed(): void {
        if (!this.canBeBorrowed()) {
            throw new Error('Book not available');
        }
        this.status = BookStatus.BORROWED;
    }
}

// Port
interface BookRepository {
    findByIsbn(isbn: string): Promise<Book | null>;
    findAvailable(): Promise<Book[]>;
    save(book: Book): Promise<void>;
}

// Use Case
class BorrowBookUseCase {
    constructor(
        private readonly bookRepository: BookRepository,
        private readonly loanRepository: LoanRepository
    ) {}
    
    async execute(isbn: string, user: User): Promise<Loan> {
        const book = await this.bookRepository.findByIsbn(isbn);
        if (!book) {
            throw new BookNotFoundException(isbn);
        }
        
        if (!book.canBeBorrowed()) {
            throw new BookNotAvailableException(isbn);
        }
        
        const loan = Loan.create(book, user);
        book.markAsBorrowed();
        
        await this.loanRepository.save(loan);
        await this.bookRepository.save(book);
        
        return loan;
    }
}

// Tests
describe('BorrowBookUseCase', () => {
    it('should borrow available book', async () => {
        const mockRepo = {
            findByIsbn: jest.fn().mockResolvedValue(new Book('123', 'Clean Code')),
            save: jest.fn()
        };
        const mockLoanRepo = { save: jest.fn() };
        
        const useCase = new BorrowBookUseCase(mockRepo as any, mockLoanRepo as any);
        
        const loan = await useCase.execute('123', new User('user@test.com'));
        
        expect(loan).toBeDefined();
        expect(mockLoanRepo.save).toHaveBeenCalled();
    });
});
```

## Python Implementation

```python
# Domain
from dataclasses import dataclass
from enum import Enum
from typing import Optional

class BookStatus(Enum):
    AVAILABLE = "available"
    BORROWED = "borrowed"

@dataclass
class Book:
    isbn: str
    title: str
    status: BookStatus = BookStatus.AVAILABLE
    
    def can_be_borrowed(self) -> bool:
        return self.status == BookStatus.AVAILABLE
    
    def mark_as_borrowed(self) -> None:
        if not self.can_be_borrowed():
            raise ValueError("Book not available")
        self.status = BookStatus.BORROWED

# Port (Protocol)
from typing import Protocol, List

class BookRepository(Protocol):
    def find_by_isbn(self, isbn: str) -> Optional[Book]: ...
    def find_available(self) -> List[Book]: ...
    def save(self, book: Book) -> None: ...

# Use Case
class BorrowBookUseCase:
    def __init__(self, book_repository: BookRepository, 
                 loan_repository: 'LoanRepository'):
        self.book_repository = book_repository
        self.loan_repository = loan_repository
    
    def execute(self, isbn: str, user: 'User') -> 'Loan':
        book = self.book_repository.find_by_isbn(isbn)
        if not book:
            raise BookNotFoundException(isbn)
        
        if not book.can_be_borrowed():
            raise BookNotAvailableException(isbn)
        
        loan = Loan.create(book, user)
        book.mark_as_borrowed()
        
        self.loan_repository.save(loan)
        self.book_repository.save(book)
        
        return loan

# Tests
import pytest
from unittest.mock import Mock

def test_should_borrow_available_book():
    # Arrange
    mock_repo = Mock()
    mock_repo.find_by_isbn.return_value = Book("123", "Clean Code")
    mock_loan_repo = Mock()
    
    use_case = BorrowBookUseCase(mock_repo, mock_loan_repo)
    
    # Act
    loan = use_case.execute("123", User("user@test.com"))
    
    # Assert
    assert loan is not None
    mock_loan_repo.save.assert_called_once()
```

## Resumen

**Multi-lenguaje** = Mismos principios SOLID aplicados en Java, TypeScript, Python.
