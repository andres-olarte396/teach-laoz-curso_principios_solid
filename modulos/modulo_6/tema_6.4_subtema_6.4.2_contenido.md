# Clean Architecture

## Capas

```
External (UI, DB, APIs)
    ↓
Interface Adapters (Controllers, Presenters)
    ↓
Application Business Rules (Use Cases)
    ↓
Enterprise Business Rules (Entities)
```

## Regla de Dependencia

**Dependencias apuntan hacia adentro**: Externo → Interno, nunca al revés.

```java
// ENTITIES (Core)
class User {
    private String id;
    private String email;
    private String passwordHash;
    
    public boolean authenticate(String password) {
        return BCrypt.checkpw(password, passwordHash);
    }
}

// USE CASES
class LoginUseCase {
    private final UserRepository userRepo;
    private final TokenGenerator tokenGen;
    
    public LoginResult execute(LoginRequest request) {
        User user = userRepo.findByEmail(request.getEmail());
        if (user == null || !user.authenticate(request.getPassword())) {
            return LoginResult.failed();
        }
        String token = tokenGen.generate(user);
        return LoginResult.success(token);
    }
}

// INTERFACE ADAPTERS
@RestController
class AuthController {
    private final LoginUseCase loginUseCase;
    
    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(@RequestBody LoginDTO dto) {
        LoginRequest request = new LoginRequest(dto.getEmail(), dto.getPassword());
        LoginResult result = loginUseCase.execute(request);
        
        if (result.isSuccess()) {
            return ResponseEntity.ok(new LoginResponse(result.getToken()));
        }
        return ResponseEntity.status(401).build();
    }
}

// FRAMEWORKS & DRIVERS
class JpaUserRepository implements UserRepository {
    @Autowired
    private UserJpaRepository jpaRepo;
    
    public User findByEmail(String email) {
        UserEntity entity = jpaRepo.findByEmail(email);
        return entity != null ? entity.toDomain() : null;
    }
}
```

## Crossing Boundaries

Use **DTOs** para cruzar capas:

```java
// Use Case Input
class LoginRequest {
    private final String email;
    private final String password;
}

// Use Case Output
class LoginResult {
    private final boolean success;
    private final String token;
}

// Controller DTO
class LoginDTO {
    @JsonProperty String email;
    @JsonProperty String password;
}
```

## Resumen

**Clean Architecture** = Capas concéntricas, dependencias hacia el core, DTOs entre capas.
