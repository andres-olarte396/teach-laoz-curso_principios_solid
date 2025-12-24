# Subtema 0.1.1. Repaso de Clases, Objetos y Encapsulación

## 1. Contexto y Motivación

Antes de adentrarnos en los principios SOLID, es fundamental que refresquemos los conceptos fundamentales de la Programación Orientada a Objetos (POO). Estos conceptos son la base sobre la cual se construyen los principios SOLID y son esenciales para comprender cómo diseñar software mantenible y escalable.

La encapsulación, junto con las clases y objetos, representa el pilar fundamental de la POO. Sin un entendimiento sólido de estos conceptos, será difícil apreciar por qué los principios SOLID son tan importantes y cómo aplicarlos efectivamente en nuestro código.

## 2. Fundamentos Teóricos

### 2.1 Clases

Una **clase** es un modelo o plantilla que define las características (atributos) y comportamientos (métodos) que tendrán los objetos creados a partir de ella.

**Características principales:**
- Define el estado mediante **atributos** (variables de instancia)
- Define el comportamiento mediante **métodos** (funciones)
- Actúa como un tipo de dato personalizado
- Puede tener constructores para inicializar objetos

### 2.2 Objetos

Un **objeto** es una instancia concreta de una clase. Es una entidad que existe en memoria y que posee:
- **Estado**: Valores específicos de sus atributos
- **Comportamiento**: Puede ejecutar los métodos definidos en su clase
- **Identidad**: Es único, incluso si tiene el mismo estado que otro objeto

### 2.3 Encapsulación

La **encapsulación** es el principio de ocultar los detalles internos de implementación de un objeto y exponer solo lo necesario a través de una interfaz pública.

**Niveles de acceso típicos:**
- **Público (public)**: Accesible desde cualquier parte
- **Privado (private)**: Solo accesible dentro de la propia clase
- **Protegido (protected)**: Accesible en la clase y sus subclases

**Beneficios de la encapsulación:**
1. **Ocultamiento de información**: Los detalles internos están protegidos
2. **Mantenibilidad**: Podemos cambiar la implementación interna sin afectar a los clientes
3. **Validación**: Controlamos cómo se accede y modifica el estado
4. **Acoplamiento reducido**: Los clientes dependen solo de la interfaz pública

## 3. Implementación Práctica

### 3.1 Ejemplo en Java

```java
/**
 * Clase que representa una cuenta bancaria con encapsulación adecuada
 */
public class CuentaBancaria {
    // Atributos privados (encapsulados)
    private String numeroCuenta;
    private String titular;
    private double saldo;
    private boolean activa;
    
    // Constructor
    public CuentaBancaria(String numeroCuenta, String titular, double saldoInicial) {
        if (saldoInicial < 0) {
            throw new IllegalArgumentException("El saldo inicial no puede ser negativo");
        }
        this.numeroCuenta = numeroCuenta;
        this.titular = titular;
        this.saldo = saldoInicial;
        this.activa = true;
    }
    
    // Métodos públicos (interfaz)
    public void depositar(double monto) {
        if (!activa) {
            throw new IllegalStateException("La cuenta está inactiva");
        }
        if (monto <= 0) {
            throw new IllegalArgumentException("El monto debe ser positivo");
        }
        saldo += monto;
    }
    
    public void retirar(double monto) {
        if (!activa) {
            throw new IllegalStateException("La cuenta está inactiva");
        }
        if (monto <= 0) {
            throw new IllegalArgumentException("El monto debe ser positivo");
        }
        if (monto > saldo) {
            throw new IllegalArgumentException("Saldo insuficiente");
        }
        saldo -= monto;
    }
    
    // Getters (acceso controlado de lectura)
    public double getSaldo() {
        return saldo;
    }
    
    public String getNumeroCuenta() {
        return numeroCuenta;
    }
    
    public String getTitular() {
        return titular;
    }
    
    public boolean isActiva() {
        return activa;
    }
    
    // Método para desactivar cuenta
    public void desactivar() {
        if (saldo > 0) {
            throw new IllegalStateException("No se puede desactivar una cuenta con saldo positivo");
        }
        activa = false;
    }
}
```

**Uso de la clase:**

```java
public class Main {
    public static void main(String[] args) {
        // Crear objeto (instancia de la clase)
        CuentaBancaria cuenta = new CuentaBancaria("001-12345", "Juan Pérez", 1000.0);
        
        // Usar métodos públicos
        cuenta.depositar(500.0);
        cuenta.retirar(200.0);
        
        // Acceder al estado mediante getters
        System.out.println("Saldo actual: $" + cuenta.getSaldo()); // 1300.0
        
        // Los atributos privados NO son accesibles directamente
        // cuenta.saldo = 999999; // ❌ ERROR DE COMPILACIÓN
    }
}
```

### 3.2 Ejemplo en TypeScript

```typescript
/**
 * Clase con encapsulación usando modificadores de acceso de TypeScript
 */
class CuentaBancaria {
    // Atributos privados
    private numeroCuenta: string;
    private titular: string;
    private saldo: number;
    private activa: boolean;
    
    constructor(numeroCuenta: string, titular: string, saldoInicial: number) {
        if (saldoInicial < 0) {
            throw new Error("El saldo inicial no puede ser negativo");
        }
        this.numeroCuenta = numeroCuenta;
        this.titular = titular;
        this.saldo = saldoInicial;
        this.activa = true;
    }
    
    public depositar(monto: number): void {
        if (!this.activa) {
            throw new Error("La cuenta está inactiva");
        }
        if (monto <= 0) {
            throw new Error("El monto debe ser positivo");
        }
        this.saldo += monto;
    }
    
    public retirar(monto: number): void {
        if (!this.activa) {
            throw new Error("La cuenta está inactiva");
        }
        if (monto <= 0) {
            throw new Error("El monto debe ser positivo");
        }
        if (monto > this.saldo) {
            throw new Error("Saldo insuficiente");
        }
        this.saldo -= monto;
    }
    
    // Getters
    public getSaldo(): number {
        return this.saldo;
    }
    
    public getNumeroCuenta(): string {
        return this.numeroCuenta;
    }
    
    public getTitular(): string {
        return this.titular;
    }
    
    public isActiva(): boolean {
        return this.activa;
    }
    
    public desactivar(): void {
        if (this.saldo > 0) {
            throw new Error("No se puede desactivar una cuenta con saldo positivo");
        }
        this.activa = false;
    }
}

// Uso
const cuenta = new CuentaBancaria("001-12345", "Juan Pérez", 1000);
cuenta.depositar(500);
cuenta.retirar(200);
console.log(`Saldo: $${cuenta.getSaldo()}`); // 1300
```

### 3.3 Ejemplo en Python

```python
class CuentaBancaria:
    """
    Clase que representa una cuenta bancaria con encapsulación.
    En Python usamos convención de _ y __ para indicar privacidad.
    """
    
    def __init__(self, numero_cuenta: str, titular: str, saldo_inicial: float):
        if saldo_inicial < 0:
            raise ValueError("El saldo inicial no puede ser negativo")
        
        self.__numero_cuenta = numero_cuenta  # Privado (name mangling)
        self.__titular = titular
        self.__saldo = saldo_inicial
        self.__activa = True
    
    def depositar(self, monto: float) -> None:
        if not self.__activa:
            raise ValueError("La cuenta está inactiva")
        if monto <= 0:
            raise ValueError("El monto debe ser positivo")
        self.__saldo += monto
    
    def retirar(self, monto: float) -> None:
        if not self.__activa:
            raise ValueError("La cuenta está inactiva")
        if monto <= 0:
            raise ValueError("El monto debe ser positivo")
        if monto > self.__saldo:
            raise ValueError("Saldo insuficiente")
        self.__saldo -= monto
    
    # Properties (getters pythónicos)
    @property
    def saldo(self) -> float:
        return self.__saldo
    
    @property
    def numero_cuenta(self) -> str:
        return self.__numero_cuenta
    
    @property
    def titular(self) -> str:
        return self.__titular
    
    @property
    def activa(self) -> bool:
        return self.__activa
    
    def desactivar(self) -> None:
        if self.__saldo > 0:
            raise ValueError("No se puede desactivar una cuenta con saldo positivo")
        self.__activa = False


# Uso
cuenta = CuentaBancaria("001-12345", "Juan Pérez", 1000.0)
cuenta.depositar(500.0)
cuenta.retirar(200.0)
print(f"Saldo: ${cuenta.saldo}")  # 1300.0

# Intento de acceso directo (name mangling lo dificulta)
# print(cuenta.__saldo)  # ❌ AttributeError
```

## 4. Código Ejecutable con Tests

### 4.1 Tests Unitarios en Java (JUnit 5)

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import static org.junit.jupiter.api.Assertions.*;

class CuentaBancariaTest {
    private CuentaBancaria cuenta;
    
    @BeforeEach
    void setUp() {
        cuenta = new CuentaBancaria("001-12345", "Juan Pérez", 1000.0);
    }
    
    @Test
    void testConstructorValido() {
        assertEquals(1000.0, cuenta.getSaldo());
        assertEquals("001-12345", cuenta.getNumeroCuenta());
        assertEquals("Juan Pérez", cuenta.getTitular());
        assertTrue(cuenta.isActiva());
    }
    
    @Test
    void testConstructorConSaldoNegativoLanzaExcepcion() {
        assertThrows(IllegalArgumentException.class, () -> {
            new CuentaBancaria("002", "Ana", -100.0);
        });
    }
    
    @Test
    void testDepositarIncrementaSaldo() {
        cuenta.depositar(500.0);
        assertEquals(1500.0, cuenta.getSaldo());
    }
    
    @Test
    void testDepositarMontoNegativoLanzaExcepcion() {
        assertThrows(IllegalArgumentException.class, () -> {
            cuenta.depositar(-100.0);
        });
    }
    
    @Test
    void testRetirarDecrementaSaldo() {
        cuenta.retirar(300.0);
        assertEquals(700.0, cuenta.getSaldo());
    }
    
    @Test
    void testRetirarMasDelSaldoLanzaExcepcion() {
        assertThrows(IllegalArgumentException.class, () -> {
            cuenta.retirar(1500.0);
        });
    }
    
    @Test
    void testDesactivarCuentaSinSaldo() {
        cuenta.retirar(1000.0);
        assertDoesNotThrow(() -> cuenta.desactivar());
        assertFalse(cuenta.isActiva());
    }
    
    @Test
    void testDesactivarCuentaConSaldoLanzaExcepcion() {
        assertThrows(IllegalStateException.class, () -> {
            cuenta.desactivar();
        });
    }
    
    @Test
    void testOperacionesEnCuentaInactivaLanzanExcepcion() {
        cuenta.retirar(1000.0);
        cuenta.desactivar();
        
        assertThrows(IllegalStateException.class, () -> {
            cuenta.depositar(100.0);
        });
        
        assertThrows(IllegalStateException.class, () -> {
            cuenta.retirar(100.0);
        });
    }
}
```

## 5. Casos de Uso Reales

### 5.1 Sistema de Gestión de Empleados

```java
public class Empleado {
    private String id;
    private String nombre;
    private String departamento;
    private double salario;
    private LocalDate fechaContratacion;
    
    public Empleado(String id, String nombre, String departamento, double salario) {
        this.id = id;
        this.nombre = nombre;
        this.departamento = departamento;
        setSalario(salario); // Usar setter para validación
        this.fechaContratacion = LocalDate.now();
    }
    
    // Encapsulación: validación al modificar salario
    public void setSalario(double nuevoSalario) {
        if (nuevoSalario < 0) {
            throw new IllegalArgumentException("El salario no puede ser negativo");
        }
        if (nuevoSalario < this.salario * 0.8) {
            throw new IllegalArgumentException("No se puede reducir el salario más del 20%");
        }
        this.salario = nuevoSalario;
    }
    
    public double getSalario() {
        return salario;
    }
    
    // Método calculado (no expone atributo directo)
    public int getAntiguedad() {
        return Period.between(fechaContratacion, LocalDate.now()).getYears();
    }
    
    public double calcularSalarioAnual() {
        return salario * 12;
    }
}
```

## 6. Comparación con Alternativas

### 6.1 Sin Encapsulación (Mal Diseño)

```java
// ❌ ANTI-PATRÓN: Clase con atributos públicos
public class CuentaBancariaMala {
    public String numeroCuenta;
    public String titular;
    public double saldo; // ¡PELIGRO! Acceso directo
    public boolean activa;
    
    public CuentaBancariaMala(String numero, String titular, double saldo) {
        this.numeroCuenta = numero;
        this.titular = titular;
        this.saldo = saldo;
        this.activa = true;
    }
}

// Uso problemático
CuentaBancariaMala cuenta = new CuentaBancariaMala("001", "Juan", 1000);
cuenta.saldo = -999999; // ❌ ¡Saldo negativo! No hay validación
cuenta.activa = false;   // ❌ Desactivada sin verificar saldo
```

**Problemas:**
- No hay validación de datos
- Estado inconsistente posible
- Imposible cambiar implementación interna sin romper código cliente
- Difícil de mantener y evolucionar

### 6.2 Con Encapsulación (Buen Diseño)

```java
// ✅ PATRÓN CORRECTO: Encapsulación adecuada
public class CuentaBancariaBuena {
    private double saldo;
    
    public void setSaldo(double nuevoSaldo) {
        if (nuevoSaldo < 0) {
            throw new IllegalArgumentException("Saldo no puede ser negativo");
        }
        this.saldo = nuevoSaldo;
    }
    
    public double getSaldo() {
        return saldo;
    }
}
```

## 7. Mejores Prácticas

### 7.1 Principio de Tell, Don't Ask

```java
// ❌ MAL: Preguntar y luego actuar (viola encapsulación)
if (cuenta.getSaldo() >= monto) {
    cuenta.setSaldo(cuenta.getSaldo() - monto);
}

// ✅ BIEN: Delegar la lógica al objeto
cuenta.retirar(monto); // El objeto decide si puede retirar
```

### 7.2 Inmutabilidad cuando sea posible

```java
// Clase inmutable (final + sin setters)
public final class Punto {
    private final int x;
    private final int y;
    
    public Punto(int x, int y) {
        this.x = x;
        this.y = y;
    }
    
    public int getX() { return x; }
    public int getY() { return y; }
    
    // Métodos que retornan nuevos objetos en lugar de mutar
    public Punto mover(int dx, int dy) {
        return new Punto(x + dx, y + dy);
    }
}
```

### 7.3 Validación consistente

```java
public class Persona {
    private String email;
    
    public void setEmail(String email) {
        if (!esEmailValido(email)) {
            throw new IllegalArgumentException("Email inválido");
        }
        this.email = email;
    }
    
    private boolean esEmailValido(String email) {
        return email != null && email.matches("^[A-Za-z0-9+_.-]+@(.+)$");
    }
}
```

## 8. Errores Comunes

### 8.1 Getters que exponen mutabilidad

```java
// ❌ MAL: Retorna referencia mutable
public class Pedido {
    private List<Producto> productos = new ArrayList<>();
    
    public List<Producto> getProductos() {
        return productos; // ❌ Cliente puede modificar la lista
    }
}

// ✅ BIEN: Retorna copia defensiva
public List<Producto> getProductos() {
    return new ArrayList<>(productos);
}

// ✅ MEJOR: Retorna vista inmutable
public List<Producto> getProductos() {
    return Collections.unmodifiableList(productos);
}
```

### 8.2 Setters innecesarios

```java
// ❌ MAL: Setter para ID (debería ser inmutable)
public class Usuario {
    private String id;
    
    public void setId(String id) { // ❌ El ID no debería cambiar
        this.id = id;
    }
}

// ✅ BIEN: ID inmutable
public class Usuario {
    private final String id;
    
    public Usuario(String id) {
        this.id = id;
    }
    
    public String getId() {
        return id;
    }
}
```

## 9. Recursos Adicionales

- **Effective Java** (Joshua Bloch) - Item 15: "Minimize mutability"
- **Clean Code** (Robert C. Martin) - Capítulo sobre objetos y estructuras de datos
- **Design Patterns** (Gang of Four) - Patrón State para encapsulación de comportamiento

## 10. Resumen Ejecutivo

La encapsulación es el principio de ocultar los detalles de implementación y exponer solo lo necesario a través de una interfaz pública. Se logra mediante:

1. **Atributos privados**: Proteger el estado interno
2. **Métodos públicos**: Proporcionar interfaz controlada
3. **Validación**: Garantizar invariantes de clase
4. **Ocultamiento de implementación**: Permitir cambios internos sin afectar clientes

## 11. Puntos Clave

✅ **Siempre** declara atributos como privados  
✅ **Proporciona** getters/setters solo cuando sea necesario  
✅ **Valida** datos en setters y constructores  
✅ **Retorna** copias defensivas de objetos mutables  
✅ **Prefiere** inmutabilidad cuando sea posible  
✅ **Aplica** el principio "Tell, Don't Ask"  
❌ **Nunca** expongas atributos públicamente sin razón justificada  
❌ **Evita** setters para atributos que no deberían cambiar
