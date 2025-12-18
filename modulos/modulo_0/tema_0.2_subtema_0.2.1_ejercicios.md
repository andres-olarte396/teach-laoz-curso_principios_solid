# Ejercicios: Unit Testing Básico con JUnit

## BANCO DE EJERCICIOS GRADUADOS

### ⭐ NIVEL 1: Conceptuales (Comprensión)

#### Ejercicio 1.1: Identificar Tests Deficientes

**Enunciado:**
Analiza los siguientes tests e identifica qué tienen de malo:

```java
class MalosTestsTest {
    
    // Test 1
    @Test
    void test1() {
        Calculator calc = new Calculator();
        int result = calc.add(2, 3);
        assertEquals(5, result);
        result = calc.multiply(2, 3);
        assertEquals(6, result);
        result = calc.divide(6, 2);
        assertEquals(3, result);
    }
    
    // Test 2
    private static int counter = 0;
    
    @Test
    void test2() {
        counter++;
        assertEquals(1, counter);
    }
    
    @Test
    void test3() {
        counter++;
        assertEquals(2, counter);
    }
    
    // Test 4
    @Test
    void test() {
        // No hace nada
        assertTrue(true);
    }
    
    // Test 5
    @Test
    void depositoTest() throws Exception {
        CuentaBancaria c = new CuentaBancaria("123", 1000);
        Thread.sleep(1000); // Espera 1 segundo
        c.depositar(500);
        assertEquals(1500, c.getSaldo());
    }
}
```

**Solución Modelo:**

**Test 1 - Múltiples assertions no relacionadas:**
- ❌ Problema: Testea 3 operaciones diferentes en un solo test
- ✅ Solución: Dividir en 3 tests separados con nombres descriptivos

```java
@Test
void sumaDebeRetornarSumaDeOperandos() {
    Calculator calc = new Calculator();
    assertEquals(5, calc.add(2, 3));
}

@Test
void multiplicacionDebeRetornarProductoDeOperandos() {
    Calculator calc = new Calculator();
    assertEquals(6, calc.multiply(2, 3));
}

@Test
void divisionDebeRetornarCocienteDeOperandos() {
    Calculator calc = new Calculator();
    assertEquals(3, calc.divide(6, 2));
}
```

**Tests 2 y 3 - Dependencia entre tests:**
- ❌ Problema: test3 depende de que test2 se ejecute primero
- ✅ Solución: Cada test debe ser independiente

```java
@Test
void primerTest() {
    int counter = 0;
    counter++;
    assertEquals(1, counter);
}

@Test
void segundoTest() {
    int counter = 0;
    counter++;
    assertEquals(1, counter); // Mismo comportamiento, independiente
}
```

**Test 4 - Test inútil:**
- ❌ Problema: No verifica nada útil
- ✅ Solución: Eliminar o escribir un test real

**Test 5 - Test lento:**
- ❌ Problema: `Thread.sleep` hace el test innecesariamente lento
- ✅ Solución: Eliminar delays artificiales

**Rúbrica:**
- Identificación de problemas (60%): ¿Detectó todos los issues?
- Explicación (30%): ¿Explicó por qué son problemáticos?
- Soluciones propuestas (10%): ¿Propuso correcciones válidas?

---

#### Ejercicio 1.2: AAA Pattern

**Enunciado:**
Reorganiza el siguiente test para seguir claramente el patrón AAA (Arrange, Act, Assert):

```java
@Test
void testCarrito() {
    CarritoCompras carrito = new CarritoCompras();
    Producto p1 = new Producto("Laptop", 1000);
    carrito.agregar(p1);
    Producto p2 = new Producto("Mouse", 20);
    assertEquals(1020, carrito.getTotal());
    carrito.agregar(p2);
}
```

**Solución Modelo:**

```java
@Test
void agregarProductosDebeCalcularTotalCorrectamente() {
    // ARRANGE: Preparar objetos y datos
    CarritoCompras carrito = new CarritoCompras();
    Producto laptop = new Producto("Laptop", 1000);
    Producto mouse = new Producto("Mouse", 20);
    
    // ACT: Ejecutar la acción que queremos testear
    carrito.agregar(laptop);
    carrito.agregar(mouse);
    
    // ASSERT: Verificar el resultado
    assertEquals(1020, carrito.getTotal());
}
```

**Explicación:**
- **Arrange**: Creación de carrito y productos
- **Act**: Agregar productos al carrito
- **Assert**: Verificar el total calculado

---

### ⭐⭐ NIVEL 2: Prácticos (Implementación)

#### Ejercicio 2.1: Test Suite Completo para Calculator

**Enunciado:**
Implementa una clase `Calculator` con operaciones básicas y escribe una suite completa de tests que cubra:
1. Operaciones básicas (suma, resta, multiplicación, división)
2. Casos edge: división por cero, overflow, números negativos
3. Validaciones de entrada

**Solución Modelo:**

```java
// Clase de producción
public class Calculator {
    
    public int add(int a, int b) {
        return a + b;
    }
    
    public int subtract(int a, int b) {
        return a - b;
    }
    
    public int multiply(int a, int b) {
        return a * b;
    }
    
    public double divide(double a, double b) {
        if (b == 0) {
            throw new ArithmeticException("División por cero no permitida");
        }
        return a / b;
    }
    
    public double sqrt(double n) {
        if (n < 0) {
            throw new IllegalArgumentException("No se puede calcular raíz de número negativo");
        }
        return Math.sqrt(n);
    }
    
    public int factorial(int n) {
        if (n < 0) {
            throw new IllegalArgumentException("Factorial no definido para negativos");
        }
        if (n > 20) {
            throw new IllegalArgumentException("Factorial causa overflow para n > 20");
        }
        if (n == 0 || n == 1) {
            return 1;
        }
        int result = 1;
        for (int i = 2; i <= n; i++) {
            result *= i;
        }
        return result;
    }
}

// Tests completos
import org.junit.jupiter.api.*;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import static org.junit.jupiter.api.Assertions.*;

@DisplayName("Calculator Tests")
class CalculatorTest {
    
    private Calculator calculator;
    
    @BeforeEach
    void setUp() {
        calculator = new Calculator();
    }
    
    @Nested
    @DisplayName("Suma")
    class SumaTests {
        
        @Test
        @DisplayName("Suma de números positivos")
        void testSumaPositivos() {
            assertEquals(5, calculator.add(2, 3));
        }
        
        @Test
        @DisplayName("Suma de números negativos")
        void testSumaNegativos() {
            assertEquals(-5, calculator.add(-2, -3));
        }
        
        @Test
        @DisplayName("Suma de positivo y negativo")
        void testSumaMixta() {
            assertEquals(1, calculator.add(5, -4));
        }
        
        @Test
        @DisplayName("Suma con cero")
        void testSumaConCero() {
            assertEquals(5, calculator.add(5, 0));
            assertEquals(5, calculator.add(0, 5));
        }
        
        @ParameterizedTest
        @DisplayName("Múltiples casos de suma")
        @CsvSource({
            "1, 1, 2",
            "10, 20, 30",
            "-5, 5, 0",
            "0, 0, 0"
        })
        void testSumaParametrizada(int a, int b, int esperado) {
            assertEquals(esperado, calculator.add(a, b));
        }
    }
    
    @Nested
    @DisplayName("Resta")
    class RestaTests {
        
        @Test
        @DisplayName("Resta básica")
        void testRestaBasica() {
            assertEquals(2, calculator.subtract(5, 3));
        }
        
        @Test
        @DisplayName("Resta que resulta en negativo")
        void testRestaResultadoNegativo() {
            assertEquals(-2, calculator.subtract(3, 5));
        }
        
        @Test
        @DisplayName("Resta con cero")
        void testRestaConCero() {
            assertEquals(5, calculator.subtract(5, 0));
        }
    }
    
    @Nested
    @DisplayName("Multiplicación")
    class MultiplicacionTests {
        
        @Test
        @DisplayName("Multiplicación básica")
        void testMultiplicacionBasica() {
            assertEquals(6, calculator.multiply(2, 3));
        }
        
        @Test
        @DisplayName("Multiplicación por cero")
        void testMultiplicacionPorCero() {
            assertEquals(0, calculator.multiply(5, 0));
            assertEquals(0, calculator.multiply(0, 5));
        }
        
        @Test
        @DisplayName("Multiplicación por uno")
        void testMultiplicacionPorUno() {
            assertEquals(5, calculator.multiply(5, 1));
        }
        
        @Test
        @DisplayName("Multiplicación de negativos")
        void testMultiplicacionNegativos() {
            assertEquals(6, calculator.multiply(-2, -3));
            assertEquals(-6, calculator.multiply(2, -3));
        }
    }
    
    @Nested
    @DisplayName("División")
    class DivisionTests {
        
        @Test
        @DisplayName("División exacta")
        void testDivisionExacta() {
            assertEquals(2.0, calculator.divide(6, 3), 0.001);
        }
        
        @Test
        @DisplayName("División con decimales")
        void testDivisionConDecimales() {
            assertEquals(2.5, calculator.divide(5, 2), 0.001);
        }
        
        @Test
        @DisplayName("División por cero debe lanzar excepción")
        void testDivisionPorCero() {
            ArithmeticException ex = assertThrows(ArithmeticException.class, () -> {
                calculator.divide(5, 0);
            });
            assertTrue(ex.getMessage().contains("División por cero"));
        }
        
        @Test
        @DisplayName("División de cero")
        void testDivisionDeCero() {
            assertEquals(0.0, calculator.divide(0, 5), 0.001);
        }
    }
    
    @Nested
    @DisplayName("Raíz cuadrada")
    class RaizCuadradaTests {
        
        @Test
        @DisplayName("Raíz de número positivo")
        void testRaizPositiva() {
            assertEquals(5.0, calculator.sqrt(25), 0.001);
            assertEquals(3.0, calculator.sqrt(9), 0.001);
        }
        
        @Test
        @DisplayName("Raíz de cero")
        void testRaizDeCero() {
            assertEquals(0.0, calculator.sqrt(0), 0.001);
        }
        
        @Test
        @DisplayName("Raíz de número negativo debe lanzar excepción")
        void testRaizNegativa() {
            IllegalArgumentException ex = assertThrows(IllegalArgumentException.class, () -> {
                calculator.sqrt(-1);
            });
            assertTrue(ex.getMessage().contains("negativo"));
        }
    }
    
    @Nested
    @DisplayName("Factorial")
    class FactorialTests {
        
        @Test
        @DisplayName("Factorial de 0 es 1")
        void testFactorialCero() {
            assertEquals(1, calculator.factorial(0));
        }
        
        @Test
        @DisplayName("Factorial de 1 es 1")
        void testFactorialUno() {
            assertEquals(1, calculator.factorial(1));
        }
        
        @Test
        @DisplayName("Factorial de números pequeños")
        void testFactorialPequeno() {
            assertEquals(2, calculator.factorial(2));
            assertEquals(6, calculator.factorial(3));
            assertEquals(24, calculator.factorial(4));
            assertEquals(120, calculator.factorial(5));
        }
        
        @Test
        @DisplayName("Factorial de número negativo debe lanzar excepción")
        void testFactorialNegativo() {
            IllegalArgumentException ex = assertThrows(IllegalArgumentException.class, () -> {
                calculator.factorial(-1);
            });
            assertTrue(ex.getMessage().contains("negativo"));
        }
        
        @Test
        @DisplayName("Factorial de número grande debe lanzar excepción")
        void testFactorialOverflow() {
            IllegalArgumentException ex = assertThrows(IllegalArgumentException.class, () -> {
                calculator.factorial(25);
            });
            assertTrue(ex.getMessage().contains("overflow"));
        }
    }
    
    @Test
    @DisplayName("Test de integración: operaciones combinadas")
    void testOperacionesCombinadas() {
        // (2 + 3) * 4 / 2 = 10
        int suma = calculator.add(2, 3);
        int multiplicacion = calculator.multiply(suma, 4);
        double division = calculator.divide(multiplicacion, 2);
        
        assertEquals(10.0, division, 0.001);
    }
}
```

**Ejecutar tests:**
```bash
mvn test
```

**Rúbrica:**
- Cobertura completa (30%): Todos los métodos testeados
- Casos edge (30%): División por cero, negativos, cero, overflow
- Organización (20%): Uso de @Nested, nombres descriptivos
- Assertions correctas (20%): Uso apropiado de assertEquals, assertThrows

---

### ⭐⭐⭐ NIVEL 3: Desafíos (TDD y Diseño)

#### Ejercicio 3.1: Shopping Cart con TDD

**Enunciado:**
Desarrolla un carrito de compras siguiendo TDD (Test-Driven Development):

1. **Escribe el test primero** (debe fallar)
2. **Implementa el código mínimo** para que pase
3. **Refactoriza** manteniendo los tests en verde

**Funcionalidades requeridas:**
- Agregar productos con cantidad
- Eliminar productos
- Calcular total
- Aplicar descuentos
- Límite de stock
- Carrito vacío

**Solución Modelo (TDD completo):**

```java
// ITERACIÓN 1: Test para agregar producto
@Test
void carritoVacioTieneTotalCero() {
    CarritoCompras carrito = new CarritoCompras();
    assertEquals(0.0, carrito.getTotal(), 0.01);
}

// Implementación mínima
public class CarritoCompras {
    public double getTotal() {
        return 0.0;
    }
}

// ITERACIÓN 2: Agregar producto
@Test
void agregarProductoDebeIncrementarTotal() {
    CarritoCompras carrito = new CarritoCompras();
    Producto laptop = new Producto("Laptop", 1000.0);
    carrito.agregar(laptop, 1);
    assertEquals(1000.0, carrito.getTotal(), 0.01);
}

// Implementación
public class Producto {
    private String nombre;
    private double precio;
    
    public Producto(String nombre, double precio) {
        this.nombre = nombre;
        this.precio = precio;
    }
    
    public double getPrecio() { return precio; }
    public String getNombre() { return nombre; }
}

public class CarritoCompras {
    private List<ItemCarrito> items = new ArrayList<>();
    
    public void agregar(Producto producto, int cantidad) {
        items.add(new ItemCarrito(producto, cantidad));
    }
    
    public double getTotal() {
        return items.stream()
            .mapToDouble(item -> item.getSubtotal())
            .sum();
    }
}

class ItemCarrito {
    private Producto producto;
    private int cantidad;
    
    public ItemCarrito(Producto producto, int cantidad) {
        this.producto = producto;
        this.cantidad = cantidad;
    }
    
    public double getSubtotal() {
        return producto.getPrecio() * cantidad;
    }
}

// ITERACIÓN 3: Múltiples productos
@Test
void agregarMultiplesProductosDebeCalcularTotalCorrectamente() {
    CarritoCompras carrito = new CarritoCompras();
    carrito.agregar(new Producto("Laptop", 1000), 1);
    carrito.agregar(new Producto("Mouse", 20), 2);
    assertEquals(1040.0, carrito.getTotal(), 0.01); // 1000 + 20*2
}

// La implementación ya funciona gracias al diseño anterior

// ITERACIÓN 4: Eliminar producto
@Test
void eliminarProductoDebeDecrementarTotal() {
    CarritoCompras carrito = new CarritoCompras();
    Producto laptop = new Producto("Laptop", 1000);
    carrito.agregar(laptop, 1);
    carrito.eliminar(laptop);
    assertEquals(0.0, carrito.getTotal(), 0.01);
}

// Nueva implementación
public void eliminar(Producto producto) {
    items.removeIf(item -> item.getProducto().equals(producto));
}

// Actualizar ItemCarrito
public Producto getProducto() { return producto; }

// ITERACIÓN 5: Stock limitado
@Test
void agregarMasQueStockDebeLanzarExcepcion() {
    CarritoCompras carrito = new CarritoCompras();
    Producto laptop = new Producto("Laptop", 1000, 5); // 5 en stock
    
    assertThrows(StockInsuficienteException.class, () -> {
        carrito.agregar(laptop, 10);
    });
}

// Implementación con stock
public class Producto {
    private String nombre;
    private double precio;
    private int stock;
    
    public Producto(String nombre, double precio, int stock) {
        this.nombre = nombre;
        this.precio = precio;
        this.stock = stock;
    }
    
    public void reducirStock(int cantidad) {
        if (cantidad > stock) {
            throw new StockInsuficienteException("Stock insuficiente");
        }
        stock -= cantidad;
    }
    
    public int getStock() { return stock; }
}

public class StockInsuficienteException extends RuntimeException {
    public StockInsuficienteException(String mensaje) {
        super(mensaje);
    }
}

public class CarritoCompras {
    public void agregar(Producto producto, int cantidad) {
        producto.reducirStock(cantidad);
        items.add(new ItemCarrito(producto, cantidad));
    }
}
```

**Suite completa final:**

```java
@DisplayName("CarritoCompras TDD")
class CarritoComprasTest {
    
    private CarritoCompras carrito;
    
    @BeforeEach
    void setUp() {
        carrito = new CarritoCompras();
    }
    
    @Test
    @DisplayName("Carrito vacío tiene total cero")
    void carritoVacioTieneTotalCero() {
        assertEquals(0.0, carrito.getTotal(), 0.01);
    }
    
    @Test
    @DisplayName("Agregar producto incrementa total")
    void agregarProductoIncrementaTotal() {
        Producto laptop = new Producto("Laptop", 1000, 10);
        carrito.agregar(laptop, 1);
        assertEquals(1000.0, carrito.getTotal(), 0.01);
    }
    
    @Test
    @DisplayName("Agregar múltiples productos calcula total correctamente")
    void agregarMultiplesProductos() {
        carrito.agregar(new Producto("Laptop", 1000, 10), 1);
        carrito.agregar(new Producto("Mouse", 20, 50), 2);
        assertEquals(1040.0, carrito.getTotal(), 0.01);
    }
    
    @Test
    @DisplayName("Eliminar producto decrementa total")
    void eliminarProductoDecrementaTotal() {
        Producto laptop = new Producto("Laptop", 1000, 10);
        carrito.agregar(laptop, 1);
        carrito.eliminar(laptop);
        assertEquals(0.0, carrito.getTotal(), 0.01);
    }
    
    @Test
    @DisplayName("Agregar más que stock lanza excepción")
    void agregarMasQueStockLanzaExcepcion() {
        Producto laptop = new Producto("Laptop", 1000, 5);
        assertThrows(StockInsuficienteException.class, () -> {
            carrito.agregar(laptop, 10);
        });
    }
    
    @Test
    @DisplayName("Vaciar carrito deja total en cero")
    void vaciarCarrito() {
        carrito.agregar(new Producto("Laptop", 1000, 10), 1);
        carrito.vaciar();
        assertEquals(0.0, carrito.getTotal(), 0.01);
        assertTrue(carrito.estaVacio());
    }
}
```

---

### ⭐⭐⭐⭐ NIVEL 4: Proyecto Profesional

#### Ejercicio 4.1: Sistema de Reservas de Hotel con Tests Completos

Desarrolla un sistema completo de reservas con:
- Gestión de habitaciones (tipos, precios, disponibilidad)
- Reservas con validación de fechas
- Sistema de pricing dinámico (temporada alta/baja)
- Cancelaciones y reembolsos
- Tests de integración y unitarios
- Cobertura > 90%

Este ejercicio requiere diseño OOP avanzado con TDD riguroso.

---

## CRITERIOS GENERALES DE EVALUACIÓN

| Criterio | Peso | Descripción |
|----------|------|-------------|
| **Corrección** | 35% | Tests pasan, código funciona correctamente |
| **Cobertura** | 20% | % de código cubierto por tests |
| **Calidad de Tests** | 25% | Tests independientes, descriptivos, bien organizados |
| **Naming** | 10% | Nombres claros y descriptivos |
| **Organización** | 10% | Uso de @Nested, @BeforeEach, estructura lógica |
