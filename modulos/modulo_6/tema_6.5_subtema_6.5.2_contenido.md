# DIP en TypeScript y Python

## TypeScript: Interfaces + DI

```typescript
// Abstracción
interface EmailService {
    send(to: string, subject: string, body: string): Promise<void>;
}

// Implementaciones
class SMTPEmailService implements EmailService {
    async send(to: string, subject: string, body: string): Promise<void> {
        // SMTP implementation
    }
}

class SendGridEmailService implements EmailService {
    async send(to: string, subject: string, body: string): Promise<void> {
        // SendGrid API
    }
}

// DI con InversifyJS
import { Container, injectable, inject } from 'inversify';

@injectable()
class UserService {
    constructor(@inject('EmailService') private emailService: EmailService) {}
    
    async registerUser(email: string): Promise<void> {
        await this.emailService.send(email, 'Welcome', 'Thanks for signing up');
    }
}

const container = new Container();
container.bind<EmailService>('EmailService').to(SMTPEmailService);
container.bind<UserService>(UserService).toSelf();

const userService = container.get<UserService>(UserService);
```

## Python: Protocols + DI

```python
from typing import Protocol
from abc import ABC, abstractmethod

# Protocol (duck typing)
class EmailService(Protocol):
    def send(self, to: str, subject: str, body: str) -> None: ...

# Implementaciones
class SMTPEmailService:
    def send(self, to: str, subject: str, body: str) -> None:
        # SMTP implementation
        pass

class SendGridEmailService:
    def send(self, to: str, subject: str, body: str) -> None:
        # SendGrid API
        pass

# DI con dependency-injector
from dependency_injector import containers, providers

class Container(containers.DeclarativeContainer):
    email_service = providers.Factory(SMTPEmailService)
    user_service = providers.Factory(
        UserService,
        email_service=email_service
    )

class UserService:
    def __init__(self, email_service: EmailService):
        self.email_service = email_service
    
    def register_user(self, email: str) -> None:
        self.email_service.send(email, 'Welcome', 'Thanks!')

# Uso
container = Container()
user_service = container.user_service()
```

## FastAPI DI

```python
from fastapi import Depends, FastAPI

app = FastAPI()

# Abstracción
class UserRepository(ABC):
    @abstractmethod
    def save(self, user: User) -> None: ...

# Implementación
class SQLUserRepository(UserRepository):
    def save(self, user: User) -> None:
        # SQL implementation
        pass

# Dependency provider
def get_user_repository() -> UserRepository:
    return SQLUserRepository()

# Inyección en endpoint
@app.post("/users")
async def create_user(
    user: User,
    repo: UserRepository = Depends(get_user_repository)
):
    repo.save(user)
    return {"status": "created"}
```

## Resumen

- **TypeScript**: InversifyJS para DI
- **Python**: Protocols + dependency-injector o FastAPI Depends
