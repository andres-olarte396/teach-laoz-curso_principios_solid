# Ejercicios: God Classes y Rigidez

## ⭐ Nivel 1
Identifica las responsabilidades en esta God Class:
```java
public class UserManager {
    public void createUser(User user) { /* validate + save + send email */ }
    public void authenticateUser(String email, String password) { /* ... */ }
    public void generateUserReport() { /* ... */ }
    public void exportUsersToCsv() { /* ... */ }
}
```

## ⭐⭐ Nivel 2
Descompón `ProductManager` (validación, pricing, persistencia, notificaciones, reportes) en 5 clases especializadas.

## ⭐⭐⭐ Nivel 3
Refactoriza sistema de facturación rígido (1000 LOC, 8 responsabilidades). Aplica SRP + OCP. Demuestra extensibilidad añadiendo nuevo formato.

## ⭐⭐⭐⭐ Nivel 4
Analiza proyecto legacy con SonarQube. Identifica 3 God Classes (>500 LOC, >15 métodos). Refactoriza completamente, mide mejora de métricas (LOC, complejidad, acoplamiento).
