# Fortalecimiento de Precondiciones

## Definición

**Fortalecimiento de precondición**: Cuando un subtipo exige condiciones MÁS ESTRICTAS que el tipo base para ejecutar un método.

**Violación LSP**: Si un cliente usa el tipo base, no debe fallar al sustituirlo por un subtipo.

## Ejemplo 1. Validación más Estricta

```java
class FileProcessor {
    // Precondición: file != null
    public void process(File file) {
        if (file == null) {
            throw new IllegalArgumentException("File cannot be null");
        }
        // Procesar archivo
    }
}

class ImageProcessor extends FileProcessor {
    @Override
    public void process(File file) {
        // ❌ Precondición más fuerte: file debe ser imagen
        if (!file.getName().endsWith(".jpg") && !file.getName().endsWith(".png")) {
            throw new IllegalArgumentException("File must be JPG or PNG");
        }
        super.process(file);
    }
}

// Cliente
void processFiles(FileProcessor processor, List<File> files) {
    for (File file : files) {
        processor.process(file); // Espera procesar cualquier archivo != null
    }
}

processFiles(new FileProcessor(), allFiles);   // ✅ Procesa todos
processFiles(new ImageProcessor(), allFiles);  // ❌ Falla con .pdf, .txt
```

**Solución**: Usar clases especializadas sin herencia o mover validación al cliente.

```java
interface FileProcessor {
    boolean canProcess(File file);
    void process(File file);
}

class GenericFileProcessor implements FileProcessor {
    public boolean canProcess(File file) {
        return file != null;
    }
    
    public void process(File file) {
        // Procesar cualquier archivo
    }
}

class ImageFileProcessor implements FileProcessor {
    public boolean canProcess(File file) {
        return file != null && (file.getName().endsWith(".jpg") || file.getName().endsWith(".png"));
    }
    
    public void process(File file) {
        if (!canProcess(file)) {
            throw new IllegalArgumentException("Not an image file");
        }
        // Procesar imagen
    }
}

// Cliente verifica antes
void processFiles(FileProcessor processor, List<File> files) {
    for (File file : files) {
        if (processor.canProcess(file)) {
            processor.process(file);
        }
    }
}
```

## Ejemplo 2: Rangos de Valores

```java
class Discount {
    // Precondición: percentage >= 0 && percentage <= 100
    public double apply(double price, double percentage) {
        if (percentage < 0 || percentage > 100) {
            throw new IllegalArgumentException("Invalid percentage");
        }
        return price * (1 - percentage / 100);
    }
}

class PremiumDiscount extends Discount {
    @Override
    public double apply(double price, double percentage) {
        // ❌ Precondición más fuerte: percentage >= 10
        if (percentage < 10) {
            throw new IllegalArgumentException("Premium discount must be at least 10%");
        }
        return super.apply(price, percentage);
    }
}

// Cliente
void applyDiscount(Discount discount, double price) {
    double finalPrice = discount.apply(price, 5); // 5% es válido para Discount base
}

applyDiscount(new Discount(), 100);        // ✅ OK
applyDiscount(new PremiumDiscount(), 100); // ❌ Falla - Viola LSP
```

## Debilitamiento de Postcondiciones

## Definición

**Debilitamiento de postcondición**: Cuando un subtipo NO GARANTIZA lo que prometía el tipo base.

## Ejemplo: Retorno de Valores

```java
class UserRepository {
    // Postcondición: retorna User != null si existe, null si no existe
    public User findById(Long id) {
        User user = database.query("SELECT * FROM users WHERE id = ?", id);
        return user; // null si no encuentra
    }
}

class CachedUserRepository extends UserRepository {
    private Map<Long, User> cache = new HashMap<>();
    
    @Override
    public User findById(Long id) {
        // ❌ Postcondición más débil: puede retornar usuario desactualizado
        User cachedUser = cache.get(id);
        if (cachedUser != null) {
            return cachedUser; // Puede estar desactualizado
        }
        
        User user = super.findById(id);
        if (user != null) {
            cache.put(id, user);
        }
        return user;
    }
}

// Cliente espera datos actualizados
void updateUser(UserRepository repo, Long id) {
    User user = repo.findById(id);
    if (user != null) {
        assert user.getVersion() == latestVersion; // ❌ Falla con CachedUserRepository
    }
}
```

**Solución**: Documentar y validar caché.

```java
class CachedUserRepository extends UserRepository {
    private Map<Long, CachedValue<User>> cache = new HashMap<>();
    private int ttl = 60; // segundos
    
    @Override
    public User findById(Long id) {
        CachedValue<User> cached = cache.get(id);
        if (cached != null && !cached.isExpired(ttl)) {
            return cached.getValue();
        }
        
        User user = super.findById(id);
        if (user != null) {
            cache.put(id, new CachedValue<>(user));
        }
        return user; // ✅ Garantiza datos frescos (< 60s)
    }
}
```

## Resumen

**Fortalecimiento precondiciones** = Exigir MÁS que tipo base → ❌ Viola LSP

**Debilitamiento postcondiciones** = Garantizar MENOS que tipo base → ❌ Viola LSP

**Solución**: Respetar contratos del tipo base o usar composición en lugar de herencia.
