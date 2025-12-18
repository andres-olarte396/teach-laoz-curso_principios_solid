# ISP en TypeScript y Python

## TypeScript: Interfaces Opcionales

```typescript
// ✅ Propiedades opcionales para ISP
interface Repository<T> {
    save(entity: T): void;
    findById(id: number): T | undefined;
    
    // Métodos opcionales
    findAll?(): T[];
    delete?(id: number): void;
}

// Implementación mínima
class SimpleRepo implements Repository<User> {
    save(user: User): void { /* ... */ }
    findById(id: number): User | undefined { /* ... */ }
    // No implementa findAll ni delete
}

// Implementación completa
class FullRepo implements Repository<User> {
    save(user: User): void { /* ... */ }
    findById(id: number): User | undefined { /* ... */ }
    findAll(): User[] { /* ... */ }
    delete(id: number): void { /* ... */ }
}
```

## Python: Protocols (PEP 544)

```python
from typing import Protocol

# ✅ Protocol para duck typing estructurado
class Readable(Protocol):
    def read(self, size: int = -1) -> bytes: ...

class Writable(Protocol):
    def write(self, data: bytes) -> int: ...

# Clases implícitamente implementan protocols
class File:
    def read(self, size: int = -1) -> bytes:
        return b"data"
    
    def write(self, data: bytes) -> int:
        return len(data)

# Cliente depende solo de lo necesario
def read_data(source: Readable) -> bytes:
    return source.read()

# Funciona con cualquier objeto con read()
read_data(File())
read_data(io.BytesIO(b"test"))
```

## Resumen

- **TypeScript**: Interfaces con propiedades/métodos opcionales
- **Python**: Protocols para ISP con duck typing
