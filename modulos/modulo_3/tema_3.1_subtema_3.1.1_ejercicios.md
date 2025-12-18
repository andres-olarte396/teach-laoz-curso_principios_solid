# Ejercicios: Definición de OCP

## ⭐ Nivel 1
Identifica la violación de OCP en este código:
```java
public class ShapeDrawer {
    public void draw(Shape shape) {
        if (shape.getType().equals("CIRCLE")) {
            drawCircle((Circle) shape);
        } else if (shape.getType().equals("RECTANGLE")) {
            drawRectangle((Rectangle) shape);
        }
    }
}
```

## ⭐⭐ Nivel 2
Refactoriza este sistema de notificaciones para cumplir OCP:
```java
public class NotificationSender {
    public void send(String type, String message) {
        if (type.equals("EMAIL")) {
            sendEmail(message);
        } else if (type.equals("SMS")) {
            sendSMS(message);
        } else if (type.equals("PUSH")) {
            sendPush(message);
        }
    }
}
```

## ⭐⭐⭐ Nivel 3
Diseña un sistema de cálculo de impuestos extensible para múltiples países sin modificar el calculador base.

## ⭐⭐⭐⭐ Nivel 4
Analiza un proyecto legacy, identifica 3 violaciones de OCP con métricas (frecuencia de cambios, complejidad ciclomática). Refactoriza y demuestra mejora.
