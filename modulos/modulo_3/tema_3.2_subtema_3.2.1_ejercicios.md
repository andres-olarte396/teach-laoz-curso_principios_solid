# Ejercicios: Template Method Pattern

## ⭐ Nivel 1
Identifica el template method, métodos abstractos y hooks:
```java
abstract class Game {
    final void play() {
        initialize();
        startPlay();
        if (shouldSaveScore()) saveScore();
        endPlay();
    }
    abstract void startPlay();
    void initialize() { }
    boolean shouldSaveScore() { return false; }
}
```

## ⭐⭐ Nivel 2
Implementa sistema de importación de datos (CSV, JSON, XML) usando Template Method. Pasos: leer → parsear → validar → guardar.

## ⭐⭐⭐ Nivel 3
Sistema de compilación multi-lenguaje (Java, Python, C++). Template: validate → compile → test → package. Hooks opcionales.

## ⭐⭐⭐⭐ Nivel 4
Framework de testing personalizado usando Template Method. Compara con JUnit. Implementa setup, execute, teardown, reporting.
