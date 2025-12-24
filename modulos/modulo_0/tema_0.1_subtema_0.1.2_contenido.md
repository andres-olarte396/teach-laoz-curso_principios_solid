# Subtema 0.1.2: Herencia y Polimorfismo

## 1. Contexto y Motivación

La herencia y el polimorfismo son dos pilares fundamentales de la POO que permiten crear jerarquías de clases y escribir código flexible y reutilizable. Estos conceptos son esenciales para comprender los principios SOLID, especialmente el Liskov Substitution Principle (LSP) y el Open/Closed Principle (OCP).

Un mal uso de la herencia es una de las causas más comunes de violaciones de los principios SOLID. Por ello, es crucial entender cuándo usar herencia, cuándo preferir composición, y cómo el polimorfismo nos permite escribir código extensible.

## 2. Fundamentos Teóricos

### 2.1 Herencia

La **herencia** es un mecanismo que permite crear nuevas clases (subclases o clases derivadas) basadas en clases existentes (superclases o clases base), reutilizando y extendiendo su funcionalidad.

**Relación IS-A**: La herencia representa una relación "es un/a". Por ejemplo:
- Un `Perro` **es un** `Animal`
- Un `Estudiante` **es una** `Persona`
- Un `CuentaAhorro` **es una** `CuentaBancaria`

**Características:**
- **Reutilización de código**: Subclases heredan atributos y métodos de la superclase
- **Especialización**: Subclases pueden agregar comportamiento específico
- **Sobrescritura (Override)**: Subclases pueden redefinir métodos heredados
- **Palabra clave**: `extends` (Java/TypeScript), `:` (C#/Python)

### 2.2 Polimorfismo

El **polimorfismo** permite que objetos de diferentes clases sean tratados como objetos de una clase común. Existen dos tipos principales:

**Polimorfismo de Subtipo (Runtime/Dinámico):**
- Una referencia de tipo base puede apuntar a objetos de tipos derivados
- El método ejecutado se determina en tiempo de ejecución
- Permite escribir código que trabaja con abstracciones

**Polimorfismo de Sobrecarga (Compile-time/Estático):**
- Múltiples métodos con el mismo nombre pero diferentes parámetros
- Se resuelve en tiempo de compilación

### 2.3 Clases Abstractas vs Interfaces

**Clase Abstracta:**
- Puede tener métodos abstractos (sin implementación) y concretos
- Puede tener atributos
- Solo herencia simple (en la mayoría de lenguajes)
- Usa `abstract` keyword

**Interface:**
- Solo define contratos (métodos sin implementación, en versiones clásicas)
- No tiene atributos (solo constantes)
- Permite implementación múltiple
- Usa `interface` keyword

## 3. Implementación Práctica

### 3.1 Ejemplo Completo en Java

```java
/**
 * Jerarquía de clases para un sistema de gestión de empleados
 */

// Clase base abstracta
public abstract class Empleado {
    private String id;
    private String nombre;
    private String departamento;
    protected double salarioBase; // protected: accesible en subclases
    
    public Empleado(String id, String nombre, String departamento, double salarioBase) {
        this.id = id;
        this.nombre = nombre;
        this.departamento = departamento;
        this.salarioBase = salarioBase;
    }
    
    // Método abstracto: cada tipo de empleado calcula su salario diferente
    public abstract double calcularSalario();
    
    // Método concreto compartido
    public String obtenerInformacion() {
        return String.format("%s - %s (%s)", id, nombre, departamento);
    }
    
    // Método que usa polimorfismo
    public void procesarPago() {
        double salario = calcularSalario(); // Llama a la implementación específica
        System.out.println("Procesando pago de $" + salario + " para " + nombre);
    }
    
    // Getters
    public String getId() { return id; }
    public String getNombre() { return nombre; }
    public String getDepartamento() { return departamento; }
    protected double getSalarioBase() { return salarioBase; }
}

// Subclase: Empleado a tiempo completo
public class EmpleadoTiempoCompleto extends Empleado {
    private double bonoAnual;
    
    public EmpleadoTiempoCompleto(String id, String nombre, String departamento, 
                                   double salarioBase, double bonoAnual) {
        super(id, nombre, departamento, salarioBase); // Llamar al constructor padre
        this.bonoAnual = bonoAnual;
    }
    
    @Override
    public double calcularSalario() {
        return salarioBase + (bonoAnual / 12); // Salario mensual + bono prorrateado
    }
    
    public double getBonoAnual() {
        return bonoAnual;
    }
}

// Subclase: Empleado por horas
public class EmpleadoPorHoras extends Empleado {
    private int horasTrabajadas;
    private double tarifaPorHora;
    
    public EmpleadoPorHoras(String id, String nombre, String departamento, 
                            double tarifaPorHora) {
        super(id, nombre, departamento, 0); // Salario base es 0
        this.tarifaPorHora = tarifaPorHora;
        this.horasTrabajadas = 0;
    }
    
    public void registrarHoras(int horas) {
        if (horas < 0) {
            throw new IllegalArgumentException("Horas no pueden ser negativas");
        }
        this.horasTrabajadas += horas;
    }
    
    @Override
    public double calcularSalario() {
        double salarioRegular = Math.min(horasTrabajadas, 160) * tarifaPorHora;
        double horasExtra = Math.max(0, horasTrabajadas - 160);
        double salarioExtra = horasExtra * tarifaPorHora * 1.5; // 50% más por hora extra
        return salarioRegular + salarioExtra;
    }
    
    public void reiniciarHoras() {
        this.horasTrabajadas = 0;
    }
}

// Subclase: Empleado con comisión (ventas)
public class EmpleadoConComision extends Empleado {
    private double porcentajeComision;
    private double ventasTotales;
    
    public EmpleadoConComision(String id, String nombre, String departamento,
                                double salarioBase, double porcentajeComision) {
        super(id, nombre, departamento, salarioBase);
        this.porcentajeComision = porcentajeComision;
        this.ventasTotales = 0;
    }
    
    public void registrarVenta(double monto) {
        if (monto < 0) {
            throw new IllegalArgumentException("Monto de venta inválido");
        }
        this.ventasTotales += monto;
    }
    
    @Override
    public double calcularSalario() {
        return salarioBase + (ventasTotales * porcentajeComision);
    }
    
    public void reiniciarVentas() {
        this.ventasTotales = 0;
    }
}

// Uso: Polimorfismo en acción
public class SistemaNomina {
    public static void main(String[] args) {
        // Lista polimórfica: todos son Empleado, pero de tipos diferentes
        List<Empleado> empleados = new ArrayList<>();
        
        empleados.add(new EmpleadoTiempoCompleto("E001", "Ana García", "IT", 5000, 6000));
        
        EmpleadoPorHoras emp2 = new EmpleadoPorHoras("E002", "Carlos López", "Soporte", 25);
        emp2.registrarHoras(180); // 20 horas extra
        empleados.add(emp2);
        
        EmpleadoConComision emp3 = new EmpleadoConComision("E003", "María Pérez", "Ventas", 2000, 0.05);
        emp3.registrarVenta(50000);
        empleados.add(emp3);
        
        // Procesamiento polimórfico: el mismo código funciona para todos los tipos
        double nominaTotal = 0;
        for (Empleado emp : empleados) {
            double salario = emp.calcularSalario(); // Polimorfismo: llama al método correcto
            emp.procesarPago();
            nominaTotal += salario;
        }
        
        System.out.println("Nómina total: $" + nominaTotal);
    }
}
```

### 3.2 Interfaces en Java

```java
// Interface para objetos que pueden ser exportados
public interface Exportable {
    String exportarCSV();
    String exportarJSON();
    String exportarXML();
}

// Interface para objetos que pueden ser auditados
public interface Auditable {
    void registrarCambio(String descripcion);
    List<String> obtenerHistorialCambios();
}

// Empleado implementando múltiples interfaces
public class EmpleadoTiempoCompleto extends Empleado 
                                     implements Exportable, Auditable {
    private List<String> historialCambios;
    
    public EmpleadoTiempoCompleto(String id, String nombre, String departamento, 
                                   double salarioBase, double bonoAnual) {
        super(id, nombre, departamento, salarioBase);
        this.bonoAnual = bonoAnual;
        this.historialCambios = new ArrayList<>();
    }
    
    @Override
    public String exportarCSV() {
        return String.format("%s,%s,%s,%.2f,%.2f", 
            getId(), getNombre(), getDepartamento(), getSalarioBase(), bonoAnual);
    }
    
    @Override
    public String exportarJSON() {
        return String.format("{\"id\":\"%s\",\"nombre\":\"%s\",\"departamento\":\"%s\",\"salario\":%.2f,\"bono\":%.2f}",
            getId(), getNombre(), getDepartamento(), getSalarioBase(), bonoAnual);
    }
    
    @Override
    public String exportarXML() {
        return String.format("<empleado><id>%s</id><nombre>%s</nombre><departamento>%s</departamento></empleado>",
            getId(), getNombre(), getDepartamento());
    }
    
    @Override
    public void registrarCambio(String descripcion) {
        String registro = LocalDateTime.now() + ": " + descripcion;
        historialCambios.add(registro);
    }
    
    @Override
    public List<String> obtenerHistorialCambios() {
        return new ArrayList<>(historialCambios);
    }
}
```

### 3.3 Ejemplo en TypeScript

```typescript
// Clase abstracta base
abstract class Forma {
    protected color: string;
    
    constructor(color: string) {
        this.color = color;
    }
    
    abstract calcularArea(): number;
    abstract calcularPerimetro(): number;
    
    // Método concreto
    public describir(): string {
        return `Forma de color ${this.color} con área ${this.calcularArea().toFixed(2)}`;
    }
}

// Subclases concretas
class Circulo extends Forma {
    private radio: number;
    
    constructor(color: string, radio: number) {
        super(color);
        this.radio = radio;
    }
    
    public calcularArea(): number {
        return Math.PI * this.radio ** 2;
    }
    
    public calcularPerimetro(): number {
        return 2 * Math.PI * this.radio;
    }
}

class Rectangulo extends Forma {
    private ancho: number;
    private alto: number;
    
    constructor(color: string, ancho: number, alto: number) {
        super(color);
        this.ancho = ancho;
        this.alto = alto;
    }
    
    public calcularArea(): number {
        return this.ancho * this.alto;
    }
    
    public calcularPerimetro(): number {
        return 2 * (this.ancho + this.alto);
    }
}

class Triangulo extends Forma {
    private base: number;
    private altura: number;
    private lado1. number;
    private lado2: number;
    
    constructor(color: string, base: number, altura: number, lado1. number, lado2: number) {
        super(color);
        this.base = base;
        this.altura = altura;
        this.lado1 = lado1;
        this.lado2 = lado2;
    }
    
    public calcularArea(): number {
        return (this.base * this.altura) / 2;
    }
    
    public calcularPerimetro(): number {
        return this.base + this.lado1 + this.lado2;
    }
}

// Uso polimórfico
const formas: Forma[] = [
    new Circulo("rojo", 5),
    new Rectangulo("azul", 10, 5),
    new Triangulo("verde", 6, 8, 5, 5)
];

let areaTotal = 0;
formas.forEach(forma => {
    console.log(forma.describir());
    areaTotal += forma.calcularArea();
});

console.log(`Área total: ${areaTotal.toFixed(2)}`);
```

### 3.4 Ejemplo en Python

```python
from abc import ABC, abstractmethod
from typing import List

# Clase abstracta base
class Vehiculo(ABC):
    def __init__(self, marca: str, modelo: str, año: int):
        self._marca = marca
        self._modelo = modelo
        self._año = año
    
    @abstractmethod
    def calcular_costo_mantenimiento(self) -> float:
        """Cada tipo de vehículo calcula su costo de mantenimiento"""
        pass
    
    @abstractmethod
    def obtener_tipo(self) -> str:
        """Retorna el tipo de vehículo"""
        pass
    
    def obtener_informacion(self) -> str:
        return f"{self._marca} {self._modelo} ({self._año})"
    
    @property
    def marca(self) -> str:
        return self._marca
    
    @property
    def modelo(self) -> str:
        return self._modelo

# Subclases concretas
class Auto(Vehiculo):
    def __init__(self, marca: str, modelo: str, año: int, num_puertas: int):
        super().__init__(marca, modelo, año)
        self._num_puertas = num_puertas
    
    def calcular_costo_mantenimiento(self) -> float:
        costo_base = 500
        años_uso = 2025 - self._año
        return costo_base + (años_uso * 50)
    
    def obtener_tipo(self) -> str:
        return "Automóvil"

class Motocicleta(Vehiculo):
    def __init__(self, marca: str, modelo: str, año: int, cilindrada: int):
        super().__init__(marca, modelo, año)
        self._cilindrada = cilindrada
    
    def calcular_costo_mantenimiento(self) -> float:
        costo_base = 300
        factor_cilindrada = self._cilindrada / 1000 * 100
        return costo_base + factor_cilindrada
    
    def obtener_tipo(self) -> str:
        return "Motocicleta"

class Camion(Vehiculo):
    def __init__(self, marca: str, modelo: str, año: int, capacidad_carga: float):
        super().__init__(marca, modelo, año)
        self._capacidad_carga = capacidad_carga
    
    def calcular_costo_mantenimiento(self) -> float:
        costo_base = 1500
        factor_carga = self._capacidad_carga * 50
        return costo_base + factor_carga
    
    def obtener_tipo(self) -> str:
        return "Camión"

# Función polimórfica
def generar_reporte_flota(vehiculos: List[Vehiculo]) -> None:
    costo_total = 0
    
    for vehiculo in vehiculos:
        costo = vehiculo.calcular_costo_mantenimiento()
        print(f"{vehiculo.obtener_tipo()}: {vehiculo.obtener_informacion()}")
        print(f"  Costo mantenimiento: ${costo:.2f}")
        costo_total += costo
    
    print(f"\nCosto total de mantenimiento: ${costo_total:.2f}")

# Uso
flota = [
    Auto("Toyota", "Corolla", 2020, 4),
    Motocicleta("Honda", "CBR", 2022, 600),
    Camion("Volvo", "FH16", 2018, 25.0)
]

generar_reporte_flota(flota)
```

## 4. Código Ejecutable con Tests

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class EmpleadoTest {
    @Test
    void testPolimorfismoCalculoSalario() {
        EmpleadoTiempoCompleto emp1 = new EmpleadoTiempoCompleto("E001", "Ana", "IT", 5000, 6000);
        assertEquals(5500.0, emp1.calcularSalario(), 0.01); // 5000 + 6000/12
        
        EmpleadoPorHoras emp2 = new EmpleadoPorHoras("E002", "Carlos", "Soporte", 25);
        emp2.registrarHoras(180);
        assertEquals(4750.0, emp2.calcularSalario(), 0.01); // 160*25 + 20*25*1.5
        
        EmpleadoConComision emp3 = new EmpleadoConComision("E003", "María", "Ventas", 2000, 0.05);
        emp3.registrarVenta(50000);
        assertEquals(4500.0, emp3.calcularSalario(), 0.01); // 2000 + 50000*0.05
    }
    
    @Test
    void testListaPolimorfica() {
        List<Empleado> empleados = Arrays.asList(
            new EmpleadoTiempoCompleto("E001", "Ana", "IT", 5000, 6000),
            new EmpleadoPorHoras("E002", "Carlos", "Soporte", 25),
            new EmpleadoConComision("E003", "María", "Ventas", 2000, 0.05)
        );
        
        assertEquals(3, empleados.size());
        
        // Todos responden al método calcularSalario() polimórficamente
        for (Empleado emp : empleados) {
            assertTrue(emp.calcularSalario() >= 0);
        }
    }
    
    @Test
    void testHerenciaDeMetodosConcretos() {
        EmpleadoTiempoCompleto emp = new EmpleadoTiempoCompleto("E001", "Ana", "IT", 5000, 6000);
        
        String info = emp.obtenerInformacion();
        assertTrue(info.contains("E001"));
        assertTrue(info.contains("Ana"));
        assertTrue(info.contains("IT"));
    }
}
```

## 5. Casos de Uso Reales

### 5.1 Sistema de Notificaciones

```java
public abstract class Notificacion {
    protected String destinatario;
    protected String asunto;
    protected String mensaje;
    
    public abstract void enviar();
    
    protected void registrarEnvio() {
        System.out.println("Notificación enviada a: " + destinatario);
    }
}

public class NotificacionEmail extends Notificacion {
    @Override
    public void enviar() {
        // Lógica específica de email
        System.out.println("Enviando email a: " + destinatario);
        // ... implementación SMTP
        registrarEnvio();
    }
}

public class NotificacionSMS extends Notificacion {
    @Override
    public void enviar() {
        // Lógica específica de SMS
        System.out.println("Enviando SMS a: " + destinatario);
        // ... implementación SMS gateway
        registrarEnvio();
    }
}

public class NotificacionPush extends Notificacion {
    @Override
    public void enviar() {
        // Lógica específica de push
        System.out.println("Enviando notificación push a: " + destinatario);
        // ... implementación Firebase/APNs
        registrarEnvio();
    }
}

// Servicio que usa polimorfismo
public class ServicioNotificaciones {
    public void enviarNotificaciones(List<Notificacion> notificaciones) {
        for (Notificacion n : notificaciones) {
            n.enviar(); // Polimorfismo: cada tipo se envía diferente
        }
    }
}
```

## 6. Comparación: Herencia vs Composición

### 6.1 Cuándo usar Herencia

✅ **Usa herencia cuando:**
- Existe una relación IS-A verdadera
- La subclase es una especialización de la superclase
- Compartes comportamiento y estado común
- La jerarquía es estable y no cambiará frecuentemente

### 6.2 Cuándo preferir Composición

✅ **Usa composición cuando:**
- La relación es HAS-A (tiene un)
- Necesitas flexibilidad para cambiar comportamiento en runtime
- Quieres evitar jerarquías profundas
- No todas las propiedades de la clase base aplican a la subclase

```java
// ❌ MAL: Herencia incorrecta (HAS-A, no IS-A)
class Coche extends Motor {  // Un coche NO ES un motor
    // ...
}

// ✅ BIEN: Composición
class Coche {
    private Motor motor;  // Un coche TIENE un motor
    
    public void arrancar() {
        motor.encender();
    }
}
```

## 7. Mejores Prácticas

### 7.1 Principio de Sustitución de Liskov (Preview)

```java
// ✅ BIEN: La subclase respeta el contrato de la superclase
class Rectangulo {
    protected int ancho;
    protected int alto;
    
    public void setAncho(int ancho) { this.ancho = ancho; }
    public void setAlto(int alto) { this.alto = alto; }
    public int getArea() { return ancho * alto; }
}

class Cuadrado extends Rectangulo {
    @Override
    public void setAncho(int ancho) {
        this.ancho = ancho;
        this.alto = ancho; // Mantiene invariante de cuadrado
    }
    
    @Override
    public void setAlto(int alto) {
        this.ancho = alto;
        this.alto = alto; // Mantiene invariante de cuadrado
    }
}

// ❌ PROBLEMA: Viola LSP si se espera comportamiento de Rectangulo
void probarRectangulo(Rectangulo r) {
    r.setAncho(5);
    r.setAlto(4);
    assert r.getArea() == 20; // Falla si es Cuadrado (área = 16)
}
```

### 7.2 Favorecer Interfaces sobre Clases Abstractas

```java
// ✅ BIEN: Interface define contrato
public interface Volador {
    void volar();
    void aterrizar();
}

public interface Nadador {
    void nadar();
}

// Clase puede implementar múltiples comportamientos
public class Pato implements Volador, Nadador {
    @Override
    public void volar() { /* ... */ }
    
    @Override
    public void aterrizar() { /* ... */ }
    
    @Override
    public void nadar() { /* ... */ }
}
```

## 8. Errores Comunes

### 8.1 Jerarquías demasiado profundas

```java
// ❌ MAL: Jerarquía frágil y difícil de mantener
class SerVivo {}
class Animal extends SerVivo {}
class Mamifero extends Animal {}
class Carnivoro extends Mamifero {}
class Felino extends Carnivoro {}
class Gato extends Felino {}
class GatoPersa extends Gato {}

// ✅ MEJOR: Jerarquía más plana + composición
class Animal {}
class Gato extends Animal {
    private Comportamiento comportamiento;
    private Raza raza;
}
```

### 8.2 Sobrescritura que cambia semántica

```java
// ❌ MAL: Cambia completamente el comportamiento esperado
class CuentaBancaria {
    public void retirar(double monto) {
        saldo -= monto;
    }
}

class CuentaBloqueada extends CuentaBancaria {
    @Override
    public void retirar(double monto) {
        // ¡No hace nada! Viola expectativas
        System.out.println("Cuenta bloqueada");
    }
}

// ✅ MEJOR: Lanzar excepción o usar composición
class CuentaBloqueada extends CuentaBancaria {
    @Override
    public void retirar(double monto) {
        throw new IllegalStateException("Cuenta bloqueada, no se puede retirar");
    }
}
```

## 9. Resumen Ejecutivo

**Herencia:**
- Mecanismo de reutilización y especialización
- Relación IS-A
- Permite polimorfismo de subtipo
- Usar con moderación, preferir composición cuando sea posible

**Polimorfismo:**
- Permite tratar objetos de diferentes tipos de manera uniforme
- Código más flexible y extensible
- Fundamental para los principios Open/Closed y Liskov Substitution

## 10. Puntos Clave

✅ **Herencia** representa relaciones IS-A verdaderas  
✅ **Polimorfismo** permite escribir código genérico que trabaja con abstracciones  
✅ **Clases abstractas** definen comportamiento común parcial  
✅ **Interfaces** definen contratos sin implementación  
✅ **Composición** suele ser preferible a herencia profunda  
✅ **Sobrescritura** debe respetar el contrato de la clase base  
❌ **Evita** jerarquías demasiado profundas (>3 niveles)  
❌ **No uses** herencia solo para reutilizar código (usa composición)
