# OCP en TypeScript y Python

## TypeScript

### Interfaces y Duck Typing

```typescript
interface PaymentProcessor {
    process(payment: Payment): Promise<void>;
}

class CreditCardProcessor implements PaymentProcessor {
    async process(payment: Payment): Promise<void> {
        // Lógica específica
    }
}

class PayPalProcessor implements PaymentProcessor {
    async process(payment: Payment): Promise<void> {
        // Lógica específica
    }
}

// Service usando polimorfismo
class PaymentService {
    constructor(private processors: PaymentProcessor[]) {}
    
    async processPayment(payment: Payment): Promise<void> {
        for (const processor of this.processors) {
            if (await processor.canHandle(payment)) {
                await processor.process(payment);
                return;
            }
        }
        throw new Error("No processor available");
    }
}
```

### Type Guards para Extensibilidad

```typescript
interface Circle {
    kind: 'circle';
    radius: number;
}

interface Rectangle {
    kind: 'rectangle';
    width: number;
    height: number;
}

interface Triangle {
    kind: 'triangle';
    base: number;
    height: number;
}

type Shape = Circle | Rectangle | Triangle;

// ✅ Extensible con type guards
function calculateArea(shape: Shape): number {
    switch (shape.kind) {
        case 'circle':
            return Math.PI * shape.radius ** 2;
        case 'rectangle':
            return shape.width * shape.height;
        case 'triangle':
            return 0.5 * shape.base * shape.height;
        default:
            // TypeScript detecta tipos no manejados
            const _exhaustive: never = shape;
            throw new Error(`Unhandled shape: ${_exhaustive}`);
    }
}
```

### Decorators para Extensión

```typescript
function Log(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function(...args: any[]) {
        console.log(`Calling ${propertyKey} with`, args);
        const result = await originalMethod.apply(this, args);
        console.log(`${propertyKey} returned`, result);
        return result;
    };
}

class UserService {
    @Log
    async createUser(user: User): Promise<void> {
        // Lógica de negocio
    }
}
```

### Generics para Extensibilidad

```typescript
interface Repository<T> {
    save(entity: T): Promise<void>;
    findById(id: string): Promise<T | null>;
    findAll(): Promise<T[]>;
}

class InMemoryRepository<T> implements Repository<T> {
    private items: Map<string, T> = new Map();
    
    async save(entity: T & { id: string }): Promise<void> {
        this.items.set(entity.id, entity);
    }
    
    async findById(id: string): Promise<T | null> {
        return this.items.get(id) || null;
    }
    
    async findAll(): Promise<T[]> {
        return Array.from(this.items.values());
    }
}

// Extensión sin modificación
const userRepo = new InMemoryRepository<User>();
const orderRepo = new InMemoryRepository<Order>();
```

## Python

### Protocolos (PEP 544) para Duck Typing Estructurado

```python
from typing import Protocol

class PaymentProcessor(Protocol):
    def process(self, payment: Payment) -> None:
        ...

class CreditCardProcessor:
    def process(self, payment: Payment) -> None:
        # Lógica específica
        pass

class PayPalProcessor:
    def process(self, payment: Payment) -> None:
        # Lógica específica
        pass

# PaymentService acepta cualquier objeto con método process()
class PaymentService:
    def __init__(self, processors: list[PaymentProcessor]):
        self.processors = processors
    
    def process_payment(self, payment: Payment) -> None:
        for processor in self.processors:
            processor.process(payment)
```

### Abstract Base Classes

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def calculate_area(self) -> float:
        pass

class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius
    
    def calculate_area(self) -> float:
        return 3.14159 * self.radius ** 2

class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        self.width = width
        self.height = height
    
    def calculate_area(self) -> float:
        return self.width * self.height

# Extensión sin modificación
def print_areas(shapes: list[Shape]) -> None:
    for shape in shapes:
        print(f"Area: {shape.calculate_area()}")
```

### Decorators para Extensibilidad

```python
from functools import wraps
import time

def timing_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} took {end - start:.2f}s")
        return result
    return wrapper

def cache_decorator(func):
    cache = {}
    
    @wraps(func)
    def wrapper(*args):
        if args in cache:
            return cache[args]
        result = func(*args)
        cache[args] = result
        return result
    return wrapper

class UserService:
    @timing_decorator
    @cache_decorator
    def get_user(self, user_id: int) -> User:
        # Lógica de negocio
        return database.find_user(user_id)
```

### Context Managers para Extensión

```python
from contextlib import contextmanager

@contextmanager
def database_transaction(connection):
    try:
        connection.begin()
        yield connection
        connection.commit()
    except Exception:
        connection.rollback()
        raise

# Uso
with database_transaction(conn) as tx:
    tx.execute("INSERT INTO users ...")
```

### Plugin System con Entry Points

```python
# setup.py
entry_points={
    'myapp.plugins': [
        'pdf = myapp.plugins.pdf:PdfPlugin',
        'excel = myapp.plugins.excel:ExcelPlugin',
    ]
}

# Plugin loader
import pkg_resources

def load_plugins():
    plugins = {}
    for entry_point in pkg_resources.iter_entry_points('myapp.plugins'):
        plugin = entry_point.load()
        plugins[entry_point.name] = plugin()
    return plugins

# Uso
plugins = load_plugins()
plugins['pdf'].process(data)
```

## Comparación TypeScript vs Python

| Característica | TypeScript | Python |
|----------------|-----------|--------|
| **Tipado** | Estático (opcional) | Dinámico (hints opcionales) |
| **Interfaces** | Nativas | Protocolos / ABC |
| **Generics** | Sí | Sí (typing.Generic) |
| **Decorators** | Experimentales (Stage 3) | Nativos |
| **Duck Typing** | Estructural | Nominal |
| **Plugin Loading** | require() dinámico | Entry points / importlib |

## Frameworks que Promueven OCP

### NestJS (TypeScript)

```typescript
@Injectable()
export class UserService {
    constructor(
        @InjectRepository(User)
        private userRepository: Repository<User>,
        private validator: UserValidator,
    ) {}
    
    async create(user: User): Promise<User> {
        await this.validator.validate(user);
        return this.userRepository.save(user);
    }
}

// Extensión mediante providers
@Module({
    providers: [
        UserService,
        UserValidator,
        EmailValidator, // Nueva validación
    ],
})
export class UserModule {}
```

### FastAPI (Python)

```python
from fastapi import Depends, FastAPI

app = FastAPI()

# Dependency Injection
def get_user_service() -> UserService:
    return UserService(UserRepository(), UserValidator())

@app.post("/users")
async def create_user(
    user: User,
    service: UserService = Depends(get_user_service)
):
    service.create(user)
    return {"status": "created"}
```

## Patrones Idiomáticos

### TypeScript: Mixins

```typescript
type Constructor<T = {}> = new (...args: any[]) => T;

function Timestamped<TBase extends Constructor>(Base: TBase) {
    return class extends Base {
        createdAt = new Date();
        updatedAt = new Date();
        
        touch() {
            this.updatedAt = new Date();
        }
    };
}

class User {
    name: string;
}

// Extensión sin modificar User
const TimestampedUser = Timestamped(User);
const user = new TimestampedUser();
user.touch();
```

### Python: Multiple Inheritance

```python
class Timestamped:
    def __init__(self):
        self.created_at = datetime.now()
        self.updated_at = datetime.now()
    
    def touch(self):
        self.updated_at = datetime.now()

class User:
    def __init__(self, name: str):
        self.name = name

# Extensión sin modificar User
class TimestampedUser(User, Timestamped):
    def __init__(self, name: str):
        User.__init__(self, name)
        Timestamped.__init__(self)

user = TimestampedUser("John")
user.touch()
```

## Resumen

**TypeScript** facilita OCP mediante:
- Interfaces estructurales
- Generics
- Type guards exhaustivos
- Decorators (experimental)

**Python** facilita OCP mediante:
- Protocolos / ABC
- Duck typing natural
- Decorators nativos
- Entry points para plugins
