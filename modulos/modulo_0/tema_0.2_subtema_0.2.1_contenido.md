# Subtema 0.2.1: Unit Testing Básico con JUnit

## 1. Contexto y Motivación

El unit testing (pruebas unitarias) es fundamental para validar que el código cumple con los principios SOLID. Los tests actúan como:
- **Documentación ejecutable**: Muestran cómo se debe usar el código
- **Red de seguridad**: Permiten refactorizar con confianza
- **Diseño guiado por tests**: Código testeable suele tener mejor diseño

Sin tests, es imposible verificar que refactorizaciones para cumplir SOLID no rompen funcionalidad existente.

## 2. Fundamentos Teóricos

### 2.1 ¿Qué es un Unit Test?

Un **unit test** es una prueba automatizada que verifica el comportamiento de una unidad pequeña de código (típicamente un método o clase) de forma aislada.

**Características de buenos unit tests:**
- **Rápidos**: Se ejecutan en milisegundos
- **Aislados**: No dependen de otros tests ni del orden de ejecución
- **Repetibles**: Producen el mismo resultado siempre
- **Auto-verificables**: Pass/fail sin intervención humana
- **Oportunos**: Se escriben junto con el código (TDD) o inmediatamente después

### 2.2 Anatomía de un Test: AAA Pattern

```java
@Test
void testNombreDescriptivo() {
    // ARRANGE: Preparar el escenario
    CuentaBancaria cuenta = new CuentaBancaria("123", 1000);
    
    // ACT: Ejecutar la acción
    cuenta.retirar(500);
    
    // ASSERT: Verificar el resultado
    assertEquals(500, cuenta.getSaldo());
}
```

### 2.3 JUnit 5: Herramienta Estándar

**JUnit 5** (también llamado JUnit Jupiter) es el framework de testing más usado en Java.

**Anotaciones principales:**
- `@Test`: Marca un método como test
- `@BeforeEach`: Se ejecuta antes de cada test
- `@AfterEach`: Se ejecuta después de cada test
- `@BeforeAll`: Se ejecuta una vez antes de todos los tests
- `@AfterAll`: Se ejecuta una vez después de todos los tests
- `@DisplayName`: Nombre legible del test
- `@Disabled`: Desactiva un test temporalmente

**Assertions comunes:**
- `assertEquals(expected, actual)`: Verifica igualdad
- `assertTrue(condition)`: Verifica que sea verdadero
- `assertFalse(condition)`: Verifica que sea falso
- `assertNull(object)`: Verifica que sea null
- `assertNotNull(object)`: Verifica que no sea null
- `assertThrows(Exception.class, () -> {...})`: Verifica que lance excepción
- `assertAll(...)`: Agrupa múltiples assertions

## 3. Implementación Práctica

### 3.1 Configuración de JUnit 5

**Maven (pom.xml):**
```xml
<dependencies>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

**Gradle (build.gradle):**
```groovy
dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.0'
}

test {
    useJUnitPlatform()
}
```

### 3.2 Ejemplo Completo: Clase CuentaBancaria

**Código de producción:**
```java
public class CuentaBancaria {
    private final String numero;
    private double saldo;
    private boolean bloqueada;
    
    public CuentaBancaria(String numero, double saldoInicial) {
        if (numero == null || numero.isBlank()) {
            throw new IllegalArgumentException("Número de cuenta inválido");
        }
        if (saldoInicial < 0) {
            throw new IllegalArgumentException("Saldo inicial no puede ser negativo");
        }
        this.numero = numero;
        this.saldo = saldoInicial;
        this.bloqueada = false;
    }
    
    public void depositar(double monto) {
        if (bloqueada) {
            throw new IllegalStateException("Cuenta bloqueada");
        }
        if (monto <= 0) {
            throw new IllegalArgumentException("Monto debe ser positivo");
        }
        saldo += monto;
    }
    
    public void retirar(double monto) {
        if (bloqueada) {
            throw new IllegalStateException("Cuenta bloqueada");
        }
        if (monto <= 0) {
            throw new IllegalArgumentException("Monto debe ser positivo");
        }
        if (monto > saldo) {
            throw new IllegalArgumentException("Saldo insuficiente");
        }
        saldo -= monto;
    }
    
    public void bloquear() {
        this.bloqueada = true;
    }
    
    public void desbloquear() {
        this.bloqueada = false;
    }
    
    public double getSaldo() {
        return saldo;
    }
    
    public String getNumero() {
        return numero;
    }
    
    public boolean estaBloqueada() {
        return bloqueada;
    }
}
```

**Tests completos:**
```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

@DisplayName("Tests de CuentaBancaria")
class CuentaBancariaTest {
    
    private CuentaBancaria cuenta;
    
    @BeforeEach
    void setUp() {
        // Se ejecuta antes de cada test
        cuenta = new CuentaBancaria("12345", 1000);
    }
    
    @Nested
    @DisplayName("Tests del constructor")
    class ConstructorTests {
        
        @Test
        @DisplayName("Constructor con parámetros válidos debe crear cuenta correctamente")
        void testConstructorValido() {
            CuentaBancaria c = new CuentaBancaria("123", 500);
            assertEquals("123", c.getNumero());
            assertEquals(500, c.getSaldo());
            assertFalse(c.estaBloqueada());
        }
        
        @Test
        @DisplayName("Constructor debe rechazar número de cuenta nulo")
        void testConstructorNumeroNulo() {
            Exception ex = assertThrows(IllegalArgumentException.class, () -> {
                new CuentaBancaria(null, 1000);
            });
            assertTrue(ex.getMessage().contains("inválido"));
        }
        
        @Test
        @DisplayName("Constructor debe rechazar número de cuenta vacío")
        void testConstructorNumeroVacio() {
            assertThrows(IllegalArgumentException.class, () -> {
                new CuentaBancaria("", 1000);
            });
        }
        
        @Test
        @DisplayName("Constructor debe rechazar saldo inicial negativo")
        void testConstructorSaldoNegativo() {
            Exception ex = assertThrows(IllegalArgumentException.class, () -> {
                new CuentaBancaria("123", -100);
            });
            assertTrue(ex.getMessage().contains("negativo"));
        }
        
        @Test
        @DisplayName("Constructor debe aceptar saldo inicial cero")
        void testConstructorSaldoCero() {
            CuentaBancaria c = new CuentaBancaria("123", 0);
            assertEquals(0, c.getSaldo());
        }
    }
    
    @Nested
    @DisplayName("Tests de depósito")
    class DepositoTests {
        
        @Test
        @DisplayName("Depósito válido debe incrementar saldo")
        void testDepositoValido() {
            cuenta.depositar(500);
            assertEquals(1500, cuenta.getSaldo());
        }
        
        @Test
        @DisplayName("Múltiples depósitos deben acumular saldo")
        void testMultiplesDepositos() {
            cuenta.depositar(200);
            cuenta.depositar(300);
            cuenta.depositar(100);
            assertEquals(1600, cuenta.getSaldo());
        }
        
        @Test
        @DisplayName("Depósito debe rechazar monto cero")
        void testDepositoCero() {
            assertThrows(IllegalArgumentException.class, () -> {
                cuenta.depositar(0);
            });
        }
        
        @Test
        @DisplayName("Depósito debe rechazar monto negativo")
        void testDepositoNegativo() {
            assertThrows(IllegalArgumentException.class, () -> {
                cuenta.depositar(-100);
            });
        }
        
        @Test
        @DisplayName("Depósito en cuenta bloqueada debe fallar")
        void testDepositoCuentaBloqueada() {
            cuenta.bloquear();
            Exception ex = assertThrows(IllegalStateException.class, () -> {
                cuenta.depositar(500);
            });
            assertTrue(ex.getMessage().contains("bloqueada"));
        }
    }
    
    @Nested
    @DisplayName("Tests de retiro")
    class RetiroTests {
        
        @Test
        @DisplayName("Retiro válido debe decrementar saldo")
        void testRetiroValido() {
            cuenta.retirar(500);
            assertEquals(500, cuenta.getSaldo());
        }
        
        @Test
        @DisplayName("Retiro de todo el saldo debe dejar saldo en cero")
        void testRetiroCompleto() {
            cuenta.retirar(1000);
            assertEquals(0, cuenta.getSaldo());
        }
        
        @Test
        @DisplayName("Retiro debe rechazar monto superior al saldo")
        void testRetiroSaldoInsuficiente() {
            Exception ex = assertThrows(IllegalArgumentException.class, () -> {
                cuenta.retirar(1500);
            });
            assertTrue(ex.getMessage().contains("insuficiente"));
            assertEquals(1000, cuenta.getSaldo()); // Saldo no cambió
        }
        
        @Test
        @DisplayName("Retiro debe rechazar monto cero")
        void testRetiroCero() {
            assertThrows(IllegalArgumentException.class, () -> {
                cuenta.retirar(0);
            });
        }
        
        @Test
        @DisplayName("Retiro debe rechazar monto negativo")
        void testRetiroNegativo() {
            assertThrows(IllegalArgumentException.class, () -> {
                cuenta.retirar(-100);
            });
        }
        
        @Test
        @DisplayName("Retiro en cuenta bloqueada debe fallar")
        void testRetiroCuentaBloqueada() {
            cuenta.bloquear();
            assertThrows(IllegalStateException.class, () -> {
                cuenta.retirar(500);
            });
        }
    }
    
    @Nested
    @DisplayName("Tests de bloqueo")
    class BloqueoTests {
        
        @Test
        @DisplayName("Bloquear cuenta debe cambiar estado a bloqueada")
        void testBloquear() {
            assertFalse(cuenta.estaBloqueada());
            cuenta.bloquear();
            assertTrue(cuenta.estaBloqueada());
        }
        
        @Test
        @DisplayName("Desbloquear cuenta debe cambiar estado a desbloqueada")
        void testDesbloquear() {
            cuenta.bloquear();
            assertTrue(cuenta.estaBloqueada());
            cuenta.desbloquear();
            assertFalse(cuenta.estaBloqueada());
        }
        
        @Test
        @DisplayName("Operaciones deben funcionar después de desbloquear")
        void testOperacionesDespuesDeDesbloquear() {
            cuenta.bloquear();
            cuenta.desbloquear();
            
            assertAll("Operaciones después de desbloquear",
                () -> assertDoesNotThrow(() -> cuenta.depositar(100)),
                () -> assertDoesNotThrow(() -> cuenta.retirar(50)),
                () -> assertEquals(1050, cuenta.getSaldo())
            );
        }
    }
    
    @Test
    @DisplayName("Test de integración: flujo completo de operaciones")
    void testFlujoCompleto() {
        // Arrange
        CuentaBancaria c = new CuentaBancaria("12345", 1000);
        
        // Act & Assert
        assertAll("Flujo completo de operaciones",
            () -> {
                c.depositar(500);
                assertEquals(1500, c.getSaldo());
            },
            () -> {
                c.retirar(300);
                assertEquals(1200, c.getSaldo());
            },
            () -> {
                c.bloquear();
                assertTrue(c.estaBloqueada());
            },
            () -> {
                assertThrows(IllegalStateException.class, () -> c.depositar(100));
            },
            () -> {
                c.desbloquear();
                assertFalse(c.estaBloqueada());
            },
            () -> {
                c.depositar(800);
                assertEquals(2000, c.getSaldo());
            }
        );
    }
}
```

### 3.3 Parametrized Tests

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;
import org.junit.jupiter.params.provider.CsvSource;

class ParametrizadosTest {
    
    @ParameterizedTest
    @DisplayName("Depósitos válidos con diferentes montos")
    @ValueSource(doubles = {10.0, 100.0, 1000.0, 10000.0})
    void testDepositosVariados(double monto) {
        CuentaBancaria cuenta = new CuentaBancaria("123", 1000);
        cuenta.depositar(monto);
        assertEquals(1000 + monto, cuenta.getSaldo());
    }
    
    @ParameterizedTest
    @DisplayName("Múltiples escenarios de retiro")
    @CsvSource({
        "1000, 500, 500",   // saldoInicial, retiro, saldoFinal
        "1000, 1000, 0",
        "500, 250, 250"
    })
    void testRetirosVariados(double saldoInicial, double retiro, double saldoEsperado) {
        CuentaBancaria cuenta = new CuentaBancaria("123", saldoInicial);
        cuenta.retirar(retiro);
        assertEquals(saldoEsperado, cuenta.getSaldo());
    }
}
```

### 3.4 Test Doubles: Mocks y Stubs

```java
// Usando Mockito para mocks
import org.mockito.Mockito;
import static org.mockito.Mockito.*;

interface ServicioNotificacion {
    void notificarRetiro(String numeroCuenta, double monto);
}

class CuentaBancariaConNotificacion extends CuentaBancaria {
    private ServicioNotificacion servicioNotificacion;
    
    public CuentaBancariaConNotificacion(String numero, double saldo, 
                                          ServicioNotificacion servicio) {
        super(numero, saldo);
        this.servicioNotificacion = servicio;
    }
    
    @Override
    public void retirar(double monto) {
        super.retirar(monto);
        servicioNotificacion.notificarRetiro(getNumero(), monto);
    }
}

class CuentaConNotificacionTest {
    
    @Test
    void testRetiroNotificaServicio() {
        // Mock del servicio
        ServicioNotificacion mockServicio = mock(ServicioNotificacion.class);
        
        CuentaBancariaConNotificacion cuenta = 
            new CuentaBancariaConNotificacion("123", 1000, mockServicio);
        
        cuenta.retirar(500);
        
        // Verificar que se llamó al servicio
        verify(mockServicio).notificarRetiro("123", 500);
    }
}
```

## 4. Mejores Prácticas

### 4.1 Naming Conventions

```java
// ✅ BIEN: Nombres descriptivos que indican lo que testean
@Test
void depositoValidoDebeIncrementarSaldo() { }

@Test
void retiroConSaldoInsuficienteDebeLanzarExcepcion() { }

@Test
void cuentaBloqueadaNoPermiteDepositos() { }

// ❌ MAL: Nombres genéricos
@Test
void test1() { }

@Test
void testDeposito() { }
```

### 4.2 Un Assert por Concepto

```java
// ✅ BIEN: Cada test verifica un concepto
@Test
void depositoDebeIncrementarSaldo() {
    cuenta.depositar(500);
    assertEquals(1500, cuenta.getSaldo());
}

@Test
void depositoNoDebeBloquearCuenta() {
    cuenta.depositar(500);
    assertFalse(cuenta.estaBloqueada());
}

// ❌ MAL: Test que verifica múltiples conceptos no relacionados
@Test
void testTodo() {
    cuenta.depositar(500);
    assertEquals(1500, cuenta.getSaldo());
    cuenta.bloquear();
    assertTrue(cuenta.estaBloqueada());
    // ... 20 assertions más
}
```

### 4.3 Tests Independientes

```java
// ✅ BIEN: Cada test crea su propio estado
@Test
void test1() {
    CuentaBancaria cuenta = new CuentaBancaria("123", 1000);
    cuenta.depositar(500);
    assertEquals(1500, cuenta.getSaldo());
}

@Test
void test2() {
    CuentaBancaria cuenta = new CuentaBancaria("123", 1000);
    cuenta.retirar(500);
    assertEquals(500, cuenta.getSaldo());
}

// ❌ MAL: Tests que dependen de estado compartido
private CuentaBancaria cuentaCompartida = new CuentaBancaria("123", 1000);

@Test
void test1() {
    cuentaCompartida.depositar(500); // Modifica estado global
}

@Test
void test2() {
    // Falla si test1 no se ejecutó primero
    assertEquals(1500, cuentaCompartida.getSaldo());
}
```

## 5. Cobertura de Código

### 5.1 Tipos de Cobertura

- **Cobertura de líneas**: % de líneas ejecutadas por tests
- **Cobertura de ramas**: % de branches (if/else) cubiertos
- **Cobertura de métodos**: % de métodos ejecutados

**Objetivo:** 80-90% de cobertura, pero **calidad > cantidad**

### 5.2 JaCoCo: Herramienta de Cobertura

```xml
<!-- Maven -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.10</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Ejecutar: `mvn test jacoco:report`

## 6. Resumen Ejecutivo

**Unit Testing:**
- Pruebas automatizadas de unidades pequeñas de código
- JUnit 5 es el framework estándar en Java
- Patrón AAA: Arrange, Act, Assert
- Tests deben ser rápidos, aislados, repetibles

**Beneficios:**
- Detectan bugs temprano
- Documentan el comportamiento esperado
- Permiten refactorizar con confianza
- Facilitan el diseño (TDD)

## 7. Puntos Clave

✅ Usa **JUnit 5** para tests en Java  
✅ Sigue el patrón **AAA** (Arrange, Act, Assert)  
✅ Nombres **descriptivos** que explican qué se testea  
✅ **Un concepto** por test  
✅ Tests **independientes** y **repetibles**  
✅ Apunta a **80-90% de cobertura** con tests significativos  
✅ Usa `@BeforeEach` para setup común  
✅ Usa `assertThrows` para verificar excepciones  
❌ Evita tests que dependen de otros tests  
❌ No testees código trivial (getters/setters simples)
