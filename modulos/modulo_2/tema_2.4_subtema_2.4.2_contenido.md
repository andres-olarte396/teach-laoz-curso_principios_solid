# SRP en TypeScript y Python

## TypeScript

### Interfaces y Tipos

```typescript
interface OrderRepository {
    save(order: Order): Promise<void>;
    findById(id: string): Promise<Order | null>;
}

class MongoOrderRepository implements OrderRepository {
    async save(order: Order): Promise<void> {
        // Implementación MongoDB
    }
    
    async findById(id: string): Promise<Order | null> {
        // Implementación MongoDB
        return null;
    }
}
```

### Decoradores para Separation of Concerns

```typescript
function Log(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    descriptor.value = function(...args: any[]) {
        console.log(`Calling ${propertyKey} with`, args);
        return originalMethod.apply(this, args);
    };
}

class UserService {
    @Log
    createUser(user: User): void {
        // Lógica de negocio pura
    }
}
```

### Dependency Injection con InversifyJS

```typescript
import { injectable, inject } from "inversify";

@injectable()
class UserService {
    constructor(
        @inject("UserRepository") private userRepository: UserRepository,
        @inject("UserValidator") private userValidator: UserValidator
    ) {}
    
    register(user: User): void {
        this.userValidator.validate(user);
        this.userRepository.save(user);
    }
}
```

## Python

### Clases y Métodos

```python
class UserValidator:
    def validate(self, user: User) -> ValidationResult:
        errors = []
        if not user.email or '@' not in user.email:
            errors.append("Invalid email")
        return ValidationResult(is_valid=len(errors) == 0, errors=errors)

class UserRepository:
    def __init__(self, db_connection):
        self.db = db_connection
    
    def save(self, user: User) -> None:
        self.db.execute("INSERT INTO users VALUES (?, ?)", (user.name, user.email))

class UserService:
    def __init__(self, repository: UserRepository, validator: UserValidator):
        self.repository = repository
        self.validator = validator
    
    def register(self, user: User) -> None:
        result = self.validator.validate(user)
        if not result.is_valid:
            raise ValidationException(result.errors)
        self.repository.save(user)
```

### Dataclasses para Value Objects

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class OrderPrice:
    subtotal: float
    tax: float
    shipping: float
    
    @property
    def total(self) -> float:
        return self.subtotal + self.tax + self.shipping
```

### Protocolos (PEP 544) para Duck Typing con SRP

```python
from typing import Protocol

class Notifier(Protocol):
    def send(self, to: str, message: str) -> None:
        ...

class EmailNotifier:
    def send(self, to: str, message: str) -> None:
        # Implementación email
        pass

class SMSNotifier:
    def send(self, to: str, message: str) -> None:
        # Implementación SMS
        pass

class UserService:
    def __init__(self, notifier: Notifier):
        self.notifier = notifier
    
    def welcome_user(self, user: User) -> None:
        self.notifier.send(user.email, "Welcome!")
```

## Comparación de Características

| Característica | TypeScript | Python |
|----------------|-----------|--------|
| **Tipado** | Estático (opcional) | Dinámico (hints opcionales) |
| **Interfaces** | Nativas | Protocolos/ABC |
| **DI** | Frameworks (InversifyJS) | Manual o frameworks (injector) |
| **Decoradores** | Sí | Sí |
| **Inmutabilidad** | readonly, as const | dataclass(frozen=True) |

## Patrones Idiomáticos

### TypeScript: Functional Approach

```typescript
// Composición de funciones para SRP
type Validator<T> = (value: T) => ValidationResult;

const emailValidator: Validator<string> = (email) => {
    return email.includes('@') 
        ? { isValid: true } 
        : { isValid: false, errors: ['Invalid email'] };
};

const lengthValidator = (min: number): Validator<string> => (value) => {
    return value.length >= min
        ? { isValid: true }
        : { isValid: false, errors: [`Min length: ${min}`] };
};

// Combinar validadores
const userEmailValidator = composeValidators(
    emailValidator,
    lengthValidator(5)
);
```

### Python: Context Managers para Separation of Concerns

```python
class DatabaseTransaction:
    def __init__(self, connection):
        self.connection = connection
    
    def __enter__(self):
        self.connection.begin()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            self.connection.rollback()
        else:
            self.connection.commit()

# Uso
class OrderRepository:
    def save(self, order: Order) -> None:
        with DatabaseTransaction(self.connection):
            self.connection.execute("INSERT INTO orders...")
```

## Frameworks que Promueven SRP

### TypeScript: NestJS

```typescript
@Injectable()
export class UserService {
    constructor(
        private readonly userRepository: UserRepository,
        private readonly userValidator: UserValidator
    ) {}
    
    async register(user: User): Promise<void> {
        await this.userValidator.validate(user);
        await this.userRepository.save(user);
    }
}
```

### Python: FastAPI

```python
from fastapi import Depends

def get_user_service(
    repository: UserRepository = Depends(get_user_repository),
    validator: UserValidator = Depends(get_user_validator)
) -> UserService:
    return UserService(repository, validator)

@app.post("/users")
async def create_user(
    user: User,
    service: UserService = Depends(get_user_service)
):
    service.register(user)
    return {"status": "created"}
```

## Resumen

TypeScript y Python, aunque dinámicos, soportan SRP mediante interfaces/protocolos, DI, y patterns funcionales. La clave es aprovechar las fortalezas de cada lenguaje: tipado gradual en TS, flexibilidad en Python.
