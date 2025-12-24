# Refactoring Seguro con Tests

## Test-Driven Refactoring

### 1. Escribir Tests de Caracterización

```java
// Test actual comportamiento antes de refactorizar
@Test
void testLegacyBehavior() {
    LegacyService service = new LegacyService();
    Result result = service.complexMethod(input);
    
    assertEquals(expectedOutput, result);
}
```

### 2. Refactorizar Incrementalmente

```java
// Paso 1. Extract Method
void complexMethod(Input input) {
    validate(input); // Extraído
    Result result = process(input); // Extraído
    save(result); // Extraído
    return result;
}

// Ejecutar tests después de cada paso
@Test
void testAfterExtraction() {
    // Mismo test, debe pasar
}
```

### 3. Introducir Abstracciones

```java
// Paso 2: Introducir interface (DIP)
interface Processor {
    Result process(Input input);
}

class LegacyProcessor implements Processor {
    public Result process(Input input) {
        // Lógica movida a implementación
    }
}

// Tests siguen pasando
@Test
void testWithInterface() {
    Processor processor = new LegacyProcessor();
    Result result = processor.process(input);
    assertEquals(expectedOutput, result);
}
```

## Golden Master Testing

```java
// Capturar salida actual como "golden master"
@Test
void testGoldenMaster() {
    String actualOutput = legacySystem.generateReport();
    
    // Primera vez: guardar como master
    // Files.writeString(Path.of("master.txt"), actualOutput);
    
    // Subsecuentes: comparar
    String master = Files.readString(Path.of("master.txt"));
    assertEquals(master, actualOutput);
}
```

## Resumen

**Refactoring seguro** = Tests antes, cambios incrementales, tests después.
