# Ejercicios: Herencia y Polimorfismo

## BANCO DE EJERCICIOS GRADUADOS

### ⭐ NIVEL 1: Conceptuales (Comprensión)

#### Ejercicio 1.1: Identificar Herencia Incorrecta

**Enunciado:**
Analiza las siguientes jerarquías de herencia e identifica cuáles son correctas y cuáles violan el principio IS-A:

```java
// Caso 1
class Stack extends ArrayList {}

// Caso 2
class Empleado extends Persona {}

// Caso 3
class Circulo extends Punto {}

// Caso 4
class Auto extends Vehiculo {}

// Caso 5
class Cuadrado extends Rectangulo {}
```

**Preguntas:**
1. ¿Cuáles relaciones IS-A son correctas?
2. ¿Qué problemas tienen las incorrectas?
3. ¿Cómo las refactorizarías?

**Solución Modelo:**

1. **Stack extends ArrayList**: ❌ INCORRECTO
   - Problema: Stack expone métodos de ArrayList que no debería (add, remove en posiciones arbitrarias)
   - Solución: Composición - Stack HAS-A ArrayList internamente

2. **Empleado extends Persona**: ✅ CORRECTO
   - Un empleado IS-A persona
   - Relación verdadera de especialización

3. **Circulo extends Punto**: ❌ INCORRECTO
   - Un círculo NO ES un punto, TIENE un centro (que es un punto)
   - Solución: `class Circulo { private Punto centro; private double radio; }`

4. **Auto extends Vehiculo**: ✅ CORRECTO
   - Un auto IS-A vehículo
   - Especialización válida

5. **Cuadrado extends Rectangulo**: ⚠️ PROBLEMÁTICO
   - Matemáticamente correcto, pero viola LSP en programación
   - Un cuadrado no puede ser sustituido por un rectángulo sin romper invariantes
   - Solución: Ambos heredan de Forma, o Cuadrado no hereda de Rectángulo

**Rúbrica:**
- Identificación correcta (40%): ¿Detectó todas las relaciones incorrectas?
- Justificación (40%): ¿Explicó por qué son incorrectas?
- Soluciones propuestas (20%): ¿Propuso alternativas válidas?

---

### ⭐⭐ NIVEL 2: Prácticos (Implementación)

#### Ejercicio 2.1: Sistema de Pagos Polimórfico

**Enunciado:**
Implementa un sistema de procesamiento de pagos que soporte múltiples métodos de pago usando herencia y polimorfismo.

**Requisitos:**
1. Clase base abstracta `MetodoPago` con método `procesarPago(double monto)`
2. Implementar: `TarjetaCredito`, `TarjetaDebito`, `PayPal`, `Transferencia`
3. Cada método tiene reglas específicas:
   - TarjetaCredito: 2% de comisión, verifica límite de crédito
   - TarjetaDebito: Sin comisión, verifica saldo disponible
   - PayPal: 3% de comisión, requiere email verificado
   - Transferencia: 1% de comisión, requiere CBU/IBAN válido
4. Implementar historial de transacciones
5. Método para calcular total de comisiones

**Solución Modelo:**

```java
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

// Clase base abstracta
public abstract class MetodoPago {
    protected String identificador;
    protected List<Transaccion> historial;
    
    public MetodoPago(String identificador) {
        this.identificador = identificador;
        this.historial = new ArrayList<>();
    }
    
    // Método abstracto: cada método de pago lo implementa diferente
    public abstract ResultadoPago procesarPago(double monto);
    
    // Método abstracto: comisión específica
    protected abstract double calcularComision(double monto);
    
    // Método concreto compartido
    protected void registrarTransaccion(double monto, boolean exitosa, String detalles) {
        Transaccion t = new Transaccion(monto, LocalDateTime.now(), exitosa, detalles);
        historial.add(t);
    }
    
    public double calcularTotalComisiones() {
        return historial.stream()
            .filter(Transaccion::isExitosa)
            .mapToDouble(t -> calcularComision(t.getMonto()))
            .sum();
    }
    
    public List<Transaccion> getHistorial() {
        return new ArrayList<>(historial);
    }
    
    public String getIdentificador() {
        return identificador;
    }
}

// Clase para representar resultado de pago
public class ResultadoPago {
    private boolean exitoso;
    private String mensaje;
    private double montoFinal;
    
    public ResultadoPago(boolean exitoso, String mensaje, double montoFinal) {
        this.exitoso = exitoso;
        this.mensaje = mensaje;
        this.montoFinal = montoFinal;
    }
    
    public boolean isExitoso() { return exitoso; }
    public String getMensaje() { return mensaje; }
    public double getMontoFinal() { return montoFinal; }
}

// Clase para transacciones
public class Transaccion {
    private double monto;
    private LocalDateTime fecha;
    private boolean exitosa;
    private String detalles;
    
    public Transaccion(double monto, LocalDateTime fecha, boolean exitosa, String detalles) {
        this.monto = monto;
        this.fecha = fecha;
        this.exitosa = exitosa;
        this.detalles = detalles;
    }
    
    public double getMonto() { return monto; }
    public boolean isExitosa() { return exitosa; }
}

// Implementaciones concretas
public class TarjetaCredito extends MetodoPago {
    private double limiteCredito;
    private double creditoDisponible;
    
    public TarjetaCredito(String numeroTarjeta, double limiteCredito) {
        super(numeroTarjeta);
        this.limiteCredito = limiteCredito;
        this.creditoDisponible = limiteCredito;
    }
    
    @Override
    protected double calcularComision(double monto) {
        return monto * 0.02; // 2%
    }
    
    @Override
    public ResultadoPago procesarPago(double monto) {
        double comision = calcularComision(monto);
        double total = monto + comision;
        
        if (total > creditoDisponible) {
            registrarTransaccion(monto, false, "Límite de crédito excedido");
            return new ResultadoPago(false, "Límite de crédito insuficiente", 0);
        }
        
        creditoDisponible -= total;
        registrarTransaccion(monto, true, "Pago exitoso con tarjeta de crédito");
        return new ResultadoPago(true, "Pago procesado correctamente", total);
    }
    
    public void pagarFactura(double monto) {
        creditoDisponible = Math.min(creditoDisponible + monto, limiteCredito);
    }
}

public class TarjetaDebito extends MetodoPago {
    private double saldo;
    
    public TarjetaDebito(String numeroTarjeta, double saldoInicial) {
        super(numeroTarjeta);
        this.saldo = saldoInicial;
    }
    
    @Override
    protected double calcularComision(double monto) {
        return 0; // Sin comisión
    }
    
    @Override
    public ResultadoPago procesarPago(double monto) {
        if (monto > saldo) {
            registrarTransaccion(monto, false, "Saldo insuficiente");
            return new ResultadoPago(false, "Saldo insuficiente", 0);
        }
        
        saldo -= monto;
        registrarTransaccion(monto, true, "Pago exitoso con tarjeta de débito");
        return new ResultadoPago(true, "Pago procesado correctamente", monto);
    }
    
    public void depositar(double monto) {
        saldo += monto;
    }
}

public class PayPal extends MetodoPago {
    private String email;
    private boolean emailVerificado;
    private double saldo;
    
    public PayPal(String email, double saldoInicial) {
        super(email);
        this.email = email;
        this.saldo = saldoInicial;
        this.emailVerificado = false;
    }
    
    public void verificarEmail() {
        this.emailVerificado = true;
    }
    
    @Override
    protected double calcularComision(double monto) {
        return monto * 0.03; // 3%
    }
    
    @Override
    public ResultadoPago procesarPago(double monto) {
        if (!emailVerificado) {
            registrarTransaccion(monto, false, "Email no verificado");
            return new ResultadoPago(false, "Debe verificar su email primero", 0);
        }
        
        double comision = calcularComision(monto);
        double total = monto + comision;
        
        if (total > saldo) {
            registrarTransaccion(monto, false, "Saldo insuficiente en PayPal");
            return new ResultadoPago(false, "Saldo insuficiente", 0);
        }
        
        saldo -= total;
        registrarTransaccion(monto, true, "Pago exitoso con PayPal");
        return new ResultadoPago(true, "Pago procesado correctamente", total);
    }
}

public class Transferencia extends MetodoPago {
    private String cbu;
    private double saldo;
    
    public Transferencia(String cbu, double saldoInicial) {
        super(cbu);
        if (!validarCBU(cbu)) {
            throw new IllegalArgumentException("CBU inválido");
        }
        this.cbu = cbu;
        this.saldo = saldoInicial;
    }
    
    private boolean validarCBU(String cbu) {
        return cbu != null && cbu.length() == 22 && cbu.matches("\\d+");
    }
    
    @Override
    protected double calcularComision(double monto) {
        return monto * 0.01; // 1%
    }
    
    @Override
    public ResultadoPago procesarPago(double monto) {
        double comision = calcularComision(monto);
        double total = monto + comision;
        
        if (total > saldo) {
            registrarTransaccion(monto, false, "Saldo insuficiente para transferencia");
            return new ResultadoPago(false, "Saldo insuficiente", 0);
        }
        
        saldo -= total;
        registrarTransaccion(monto, true, "Transferencia exitosa");
        return new ResultadoPago(true, "Transferencia procesada correctamente", total);
    }
}

// Procesador de pagos que usa polimorfismo
public class ProcesadorPagos {
    public void procesarCompra(MetodoPago metodoPago, double monto) {
        System.out.println("Procesando pago de $" + monto + " con " + 
                           metodoPago.getClass().getSimpleName());
        
        ResultadoPago resultado = metodoPago.procesarPago(monto);
        
        if (resultado.isExitoso()) {
            System.out.println("✓ " + resultado.getMensaje());
            System.out.println("  Monto final: $" + resultado.getMontoFinal());
        } else {
            System.out.println("✗ " + resultado.getMensaje());
        }
    }
    
    public void generarReporteComisiones(List<MetodoPago> metodosPago) {
        double totalComisiones = 0;
        
        for (MetodoPago mp : metodosPago) {
            double comision = mp.calcularTotalComisiones();
            System.out.println(mp.getIdentificador() + ": $" + comision);
            totalComisiones += comision;
        }
        
        System.out.println("Total comisiones: $" + totalComisiones);
    }
}
```

**Tests:**

```java
class MetodoPagoTest {
    @Test
    void testTarjetaCreditoExitoso() {
        TarjetaCredito tc = new TarjetaCredito("1234-5678", 10000);
        ResultadoPago resultado = tc.procesarPago(1000);
        
        assertTrue(resultado.isExitoso());
        assertEquals(1020, resultado.getMontoFinal(), 0.01); // 1000 + 2%
    }
    
    @Test
    void testTarjetaCreditoLimiteExcedido() {
        TarjetaCredito tc = new TarjetaCredito("1234-5678", 1000);
        ResultadoPago resultado = tc.procesarPago(2000);
        
        assertFalse(resultado.isExitoso());
    }
    
    @Test
    void testPayPalEmailNoVerificado() {
        PayPal pp = new PayPal("user@example.com", 5000);
        ResultadoPago resultado = pp.procesarPago(100);
        
        assertFalse(resultado.isExitoso());
        assertTrue(resultado.getMensaje().contains("verificar"));
    }
    
    @Test
    void testPolimorfismoListaPagos() {
        List<MetodoPago> metodos = Arrays.asList(
            new TarjetaCredito("1234", 5000),
            new TarjetaDebito("5678", 3000),
            new Transferencia("1234567890123456789012", 10000)
        );
        
        for (MetodoPago mp : metodos) {
            ResultadoPago r = mp.procesarPago(100);
            assertNotNull(r);
        }
    }
}
```

**Rúbrica:**
- Jerarquía de herencia (25%): Clase base abstracta correcta
- Polimorfismo (25%): Métodos abstractos implementados correctamente
- Lógica de negocio (30%): Validaciones y cálculos correctos
- Tests (20%): Cobertura completa

---

### ⭐⭐⭐ NIVEL 3: Desafíos (Diseño Avanzado)

#### Ejercicio 3.1: Sistema de Descuentos con Strategy Pattern

**Enunciado:**
Diseña un sistema de descuentos para un e-commerce usando herencia e interfaces. El sistema debe:

1. Soportar múltiples tipos de descuento:
   - Por porcentaje
   - Por monto fijo
   - 2x1, 3x2
   - Por cantidad (compra 5, paga 4)
   - Escalonado (más compras, más descuento)

2. Los descuentos deben ser combinables con reglas:
   - Máximo 3 descuentos por compra
   - Descuento total no puede exceder 50%
   - Algunos descuentos son mutuamente exclusivos

3. Implementar validaciones:
   - Vigencia de descuentos (fecha inicio/fin)
   - Productos aplicables
   - Cantidad mínima de compra

**Solución Modelo (Parcial):**

```java
// Interface para estrategia de descuento
public interface EstrategiaDescuento {
    double aplicarDescuento(double precioOriginal, int cantidad);
    boolean esAplicable(Producto producto, int cantidad);
    String getDescripcion();
}

// Clase abstracta base para descuentos
public abstract class Descuento implements EstrategiaDescuento {
    protected String codigo;
    protected LocalDate fechaInicio;
    protected LocalDate fechaFin;
    protected Set<String> categoriasAplicables;
    
    public Descuento(String codigo, LocalDate inicio, LocalDate fin) {
        this.codigo = codigo;
        this.fechaInicio = inicio;
        this.fechaFin = fin;
        this.categoriasAplicables = new HashSet<>();
    }
    
    public boolean estaVigente() {
        LocalDate hoy = LocalDate.now();
        return !hoy.isBefore(fechaInicio) && !hoy.isAfter(fechaFin);
    }
    
    @Override
    public boolean esAplicable(Producto producto, int cantidad) {
        return estaVigente() && 
               (categoriasAplicables.isEmpty() || 
                categoriasAplicables.contains(producto.getCategoria()));
    }
    
    public String getCodigo() { return codigo; }
}

// Implementaciones concretas
public class DescuentoPorcentaje extends Descuento {
    private double porcentaje;
    
    public DescuentoPorcentaje(String codigo, double porcentaje, LocalDate inicio, LocalDate fin) {
        super(codigo, inicio, fin);
        if (porcentaje < 0 || porcentaje > 100) {
            throw new IllegalArgumentException("Porcentaje inválido");
        }
        this.porcentaje = porcentaje;
    }
    
    @Override
    public double aplicarDescuento(double precioOriginal, int cantidad) {
        double descuento = precioOriginal * cantidad * (porcentaje / 100);
        return precioOriginal * cantidad - descuento;
    }
    
    @Override
    public String getDescripcion() {
        return String.format("%d%% de descuento", (int)porcentaje);
    }
}

public class Descuento2x1 extends Descuento {
    public Descuento2x1(String codigo, LocalDate inicio, LocalDate fin) {
        super(codigo, inicio, fin);
    }
    
    @Override
    public double aplicarDescuento(double precioOriginal, int cantidad) {
        int unidadesPagadas = (cantidad / 2) + (cantidad % 2);
        return precioOriginal * unidadesPagadas;
    }
    
    @Override
    public boolean esAplicable(Producto producto, int cantidad) {
        return super.esAplicable(producto, cantidad) && cantidad >= 2;
    }
    
    @Override
    public String getDescripcion() {
        return "2x1";
    }
}

// Gestor de descuentos con validaciones
public class GestorDescuentos {
    private static final int MAX_DESCUENTOS_POR_COMPRA = 3;
    private static final double MAX_DESCUENTO_PORCENTUAL = 0.5;
    
    public double calcularPrecioFinal(Producto producto, int cantidad, 
                                       List<Descuento> descuentos) {
        // Validar límite de descuentos
        if (descuentos.size() > MAX_DESCUENTOS_POR_COMPRA) {
            throw new IllegalArgumentException("Máximo " + MAX_DESCUENTOS_POR_COMPRA + 
                                                " descuentos por compra");
        }
        
        double precioOriginal = producto.getPrecio() * cantidad;
        double precioFinal = precioOriginal;
        
        // Aplicar descuentos secuencialmente
        for (Descuento descuento : descuentos) {
            if (descuento.esAplicable(producto, cantidad)) {
                precioFinal = descuento.aplicarDescuento(producto.getPrecio(), cantidad);
            }
        }
        
        // Validar descuento máximo
        double descuentoTotal = precioOriginal - precioFinal;
        if (descuentoTotal > precioOriginal * MAX_DESCUENTO_PORCENTUAL) {
            throw new IllegalStateException("Descuento total excede el 50%");
        }
        
        return precioFinal;
    }
}
```

---

### ⭐⭐⭐⭐ NIVEL 4: Proyecto Integrador

#### Ejercicio 4.1: Sistema de Gestión de Biblioteca

Implementa un sistema completo de biblioteca con:
- Jerarquía de items (Libro, Revista, DVD, AudioLibro)
- Tipos de usuarios (Estudiante, Profesor, Público)
- Políticas de préstamo polimórficas
- Sistema de multas diferenciado
- Reservas y listas de espera

Este ejercicio requiere 800+ líneas de código con diseño OOP avanzado.

---

## ERRORES COMUNES Y SOLUCIONES

### Error 1: Romper LSP con sobrescritura

```java
// ❌ MAL
class Rectangulo {
    protected int ancho, alto;
    public void setAncho(int ancho) { this.ancho = ancho; }
    public int getArea() { return ancho * alto; }
}

class Cuadrado extends Rectangulo {
    @Override
    public void setAncho(int ancho) {
        this.ancho = ancho;
        this.alto = ancho; // Rompe invariante de Rectangulo
    }
}

// Test que falla
void test(Rectangulo r) {
    r.setAncho(5);
    r.setAlto(4);
    assert r.getArea() == 20; // Falla si es Cuadrado
}

// ✅ BIEN: Usar composición o jerarquía diferente
interface Forma {
    int calcularArea();
}

class Rectangulo implements Forma {
    private int ancho, alto;
    // ...
}

class Cuadrado implements Forma {
    private int lado;
    // ...
}
```

### Error 2: Exponer implementación interna

```java
// ❌ MAL
class Empleado {
    protected List<String> tareas; // Protegido expone implementación
}

class Gerente extends Empleado {
    public void hackearTareas() {
        tareas.clear(); // Puede romper invariantes
    }
}

// ✅ BIEN: Métodos protegidos, atributos privados
class Empleado {
    private List<String> tareas;
    
    protected void agregarTarea(String tarea) {
        tareas.add(tarea);
    }
}
```
