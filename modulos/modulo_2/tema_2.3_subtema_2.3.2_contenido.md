# Separación de Concerns: UI, Lógica, Persistencia

## Arquitectura en Capas

La separación de concerns es fundamental para SRP. Las aplicaciones se dividen en capas:

### Capas Típicas

1. **Presentación (UI)**: Interacción con usuario
2. **Lógica de Negocio**: Reglas del dominio
3. **Acceso a Datos**: Persistencia

## Ejemplo: Violación de Separación

```java
// ❌ Mezcla UI, lógica y persistencia
public class UserRegistrationForm extends JFrame {
    private JTextField nameField;
    private JTextField emailField;
    private Connection dbConnection;
    
    public void onRegisterButtonClick() {
        // UI
        String name = nameField.getText();
        String email = emailField.getText();
        
        // Validación (Lógica)
        if (!email.contains("@")) {
            JOptionPane.showMessageDialog(this, "Invalid email");
            return;
        }
        
        // Persistencia
        try {
            Statement stmt = dbConnection.createStatement();
            stmt.execute("INSERT INTO users VALUES ('" + name + "', '" + email + "')");
            JOptionPane.showMessageDialog(this, "User created!");
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

**Problemas**: Imposible testear lógica sin UI, no reutilizable, alto acoplamiento.

## Solución: Separación en Capas

```java
// CAPA 1: Dominio
public class User {
    private String name;
    private String email;
    // Getters/setters
}

// CAPA 2: Lógica de Negocio
public class UserValidator {
    public ValidationResult validate(User user) {
        if (!user.getEmail().contains("@")) {
            return ValidationResult.invalid("Invalid email");
        }
        return ValidationResult.valid();
    }
}

public class UserRegistrationService {
    private final UserRepository userRepository;
    private final UserValidator validator;
    
    public void register(User user) {
        ValidationResult result = validator.validate(user);
        if (!result.isValid()) {
            throw new ValidationException(result.getErrors());
        }
        userRepository.save(user);
    }
}

// CAPA 3: Persistencia
public class UserRepository {
    private final Connection connection;
    
    public void save(User user) {
        try (PreparedStatement stmt = connection.prepareStatement(
            "INSERT INTO users (name, email) VALUES (?, ?)"
        )) {
            stmt.setString(1, user.getName());
            stmt.setString(2, user.getEmail());
            stmt.executeUpdate();
        }
    }
}

// CAPA 4: UI (ahora delgada)
public class UserRegistrationForm extends JFrame {
    private JTextField nameField;
    private JTextField emailField;
    private UserRegistrationService registrationService;
    
    public void onRegisterButtonClick() {
        User user = new User();
        user.setName(nameField.getText());
        user.setEmail(emailField.getText());
        
        try {
            registrationService.register(user);
            JOptionPane.showMessageDialog(this, "User created!");
        } catch (ValidationException e) {
            JOptionPane.showMessageDialog(this, e.getMessage());
        }
    }
}
```

**Beneficios**: Lógica testeable, UI intercambiable (Swing → Web), persistencia intercambiable (MySQL → MongoDB).

## Arquitecturas Comunes

### MVC (Model-View-Controller)

- **Model**: Lógica de negocio
- **View**: Presentación
- **Controller**: Coordina Model y View

### Hexagonal (Ports & Adapters)

- **Core**: Lógica pura (sin dependencias externas)
- **Ports**: Interfaces
- **Adapters**: Implementaciones (UI, DB, APIs)

### Clean Architecture

- **Entities**: Lógica crítica del negocio
- **Use Cases**: Lógica de aplicación
- **Interface Adapters**: Conversores
- **Frameworks**: UI, DB, Web

## Resumen

**Regla de oro**: Nunca mezcles UI con lógica de negocio o persistencia. Cada capa debe poder cambiar independientemente.
