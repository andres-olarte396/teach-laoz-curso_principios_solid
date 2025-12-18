# Ejercicios: Separación de Concerns

## ⭐ Nivel 1
Identifica qué capa viola cada fragmento de código (UI, Lógica, Datos).

## ⭐⭐ Nivel 2
Refactoriza esta clase que mezcla todo en capas separadas:
```java
public class ProductManager extends JPanel {
    public void saveProduct() {
        String name = nameField.getText();
        if (name.length() < 3) {
            JOptionPane.showMessageDialog(this, "Name too short");
            return;
        }
        Connection conn = DriverManager.getConnection("jdbc:mysql://...");
        conn.executeUpdate("INSERT INTO products...");
    }
}
```

## ⭐⭐⭐ Nivel 3
Diseña aplicación de tareas (TODO) con arquitectura en 3 capas. Implementa UI console y web intercambiables.

## ⭐⭐⭐⭐ Nivel 4
Implementa Clean Architecture para sistema de blog. Diagrama de dependencias, tests de cada capa.
