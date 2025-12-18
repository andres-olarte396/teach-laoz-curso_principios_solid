# Ejercicios: Clases, Objetos y Encapsulación

## BANCO DE EJERCICIOS GRADUADOS

### ⭐ NIVEL 1: Conceptuales (Comprensión)

#### Ejercicio 1.1: Identificación de Violaciones de Encapsulación

**Enunciado:**
Analiza el siguiente código y identifica todas las violaciones del principio de encapsulación:

```java
public class Estudiante {
    public String nombre;
    public int edad;
    public double[] calificaciones;
    public String carrera;
    
    public Estudiante(String nombre, int edad) {
        this.nombre = nombre;
        this.edad = edad;
        this.calificaciones = new double[10];
    }
}

// Uso
Estudiante estudiante = new Estudiante("María", 20);
estudiante.edad = -5; // edad negativa
estudiante.calificaciones[0] = 150; // calificación inválida
estudiante.nombre = ""; // nombre vacío
```

**Preguntas:**
1. ¿Qué problemas tiene exponer los atributos como públicos?
2. ¿Qué validaciones deberían existir?
3. ¿Cómo refactorizarías esta clase para aplicar encapsulación correcta?

**Solución Modelo:**

```java
public class Estudiante {
    // Atributos privados
    private String nombre;
    private int edad;
    private double[] calificaciones;
    private String carrera;
    private static final int MAX_CALIFICACIONES = 10;
    
    // Constructor con validación
    public Estudiante(String nombre, int edad, String carrera) {
        setNombre(nombre);
        setEdad(edad);
        setCarrera(carrera);
        this.calificaciones = new double[MAX_CALIFICACIONES];
    }
    
    // Setters con validación
    public void setNombre(String nombre) {
        if (nombre == null || nombre.trim().isEmpty()) {
            throw new IllegalArgumentException("El nombre no puede estar vacío");
        }
        this.nombre = nombre;
    }
    
    public void setEdad(int edad) {
        if (edad < 0 || edad > 120) {
            throw new IllegalArgumentException("Edad inválida");
        }
        this.edad = edad;
    }
    
    public void setCarrera(String carrera) {
        if (carrera == null || carrera.trim().isEmpty()) {
            throw new IllegalArgumentException("La carrera no puede estar vacía");
        }
        this.carrera = carrera;
    }
    
    // Método para agregar calificación con validación
    public void agregarCalificacion(int indice, double calificacion) {
        if (indice < 0 || indice >= MAX_CALIFICACIONES) {
            throw new IndexOutOfBoundsException("Índice fuera de rango");
        }
        if (calificacion < 0 || calificacion > 100) {
            throw new IllegalArgumentException("Calificación debe estar entre 0 y 100");
        }
        this.calificaciones[indice] = calificacion;
    }
    
    // Getters
    public String getNombre() { return nombre; }
    public int getEdad() { return edad; }
    public String getCarrera() { return carrera; }
    
    // Retornar copia defensiva de calificaciones
    public double[] getCalificaciones() {
        return calificaciones.clone();
    }
    
    // Método calculado
    public double getPromedio() {
        double suma = 0;
        int count = 0;
        for (double cal : calificaciones) {
            if (cal > 0) {
                suma += cal;
                count++;
            }
        }
        return count > 0 ? suma / count : 0;
    }
}
```

**Rúbrica de Evaluación:**
- Identificación de problemas (30%): ¿Encontró todas las violaciones?
- Validaciones propuestas (40%): ¿Son completas y correctas?
- Implementación de encapsulación (30%): ¿Aplicó correctamente private/getters/setters?

---

#### Ejercicio 1.2: Tell, Don't Ask

**Enunciado:**
El siguiente código viola el principio "Tell, Don't Ask". Refactorízalo para que la lógica esté dentro del objeto correcto.

```java
public class CarritoCompras {
    public List<Producto> productos;
    public double descuento;
}

public class Producto {
    public String nombre;
    public double precio;
    public int cantidad;
}

// Cliente que viola Tell, Don't Ask
public class Checkout {
    public double calcularTotal(CarritoCompras carrito) {
        double total = 0;
        for (Producto p : carrito.productos) {
            total += p.precio * p.cantidad;
        }
        total = total * (1 - carrito.descuento);
        return total;
    }
}
```

**Solución Modelo:**

```java
public class CarritoCompras {
    private List<Producto> productos;
    private double descuento;
    
    public CarritoCompras() {
        this.productos = new ArrayList<>();
        this.descuento = 0;
    }
    
    public void agregarProducto(Producto producto) {
        productos.add(producto);
    }
    
    public void aplicarDescuento(double descuento) {
        if (descuento < 0 || descuento > 1) {
            throw new IllegalArgumentException("Descuento debe estar entre 0 y 1");
        }
        this.descuento = descuento;
    }
    
    // El carrito ahora calcula su propio total
    public double calcularTotal() {
        double subtotal = productos.stream()
            .mapToDouble(Producto::getSubtotal)
            .sum();
        return subtotal * (1 - descuento);
    }
}

public class Producto {
    private String nombre;
    private double precio;
    private int cantidad;
    
    public Producto(String nombre, double precio, int cantidad) {
        this.nombre = nombre;
        setPrecio(precio);
        setCantidad(cantidad);
    }
    
    private void setPrecio(double precio) {
        if (precio < 0) {
            throw new IllegalArgumentException("Precio no puede ser negativo");
        }
        this.precio = precio;
    }
    
    private void setCantidad(int cantidad) {
        if (cantidad < 0) {
            throw new IllegalArgumentException("Cantidad no puede ser negativa");
        }
        this.cantidad = cantidad;
    }
    
    // El producto calcula su propio subtotal
    public double getSubtotal() {
        return precio * cantidad;
    }
}

// Cliente simplificado
public class Checkout {
    public double procesarPago(CarritoCompras carrito) {
        return carrito.calcularTotal(); // Tell, don't ask
    }
}
```

---

### ⭐⭐ NIVEL 2: Prácticos (Implementación)

#### Ejercicio 2.1: Sistema de Reservas de Hotel

**Enunciado:**
Implementa una clase `Habitacion` para un sistema de reservas de hotel con las siguientes especificaciones:

**Requisitos:**
1. Atributos: número de habitación, tipo (simple/doble/suite), precio por noche, estado (disponible/ocupada/mantenimiento)
2. No se puede cambiar el número de habitación una vez creada
3. El precio debe ser mayor a 0
4. Solo se puede reservar si está disponible
5. Debe mantener un historial de fechas de reserva (lista de pares fecha-inicio/fecha-fin)
6. Implementar método para calcular ingresos totales generados

**Casos de prueba esperados:**
```java
// Crear habitación
Habitacion h = new Habitacion(101, TipoHabitacion.DOBLE, 150.0);

// Reservar
h.reservar(LocalDate.of(2025, 1, 10), LocalDate.of(2025, 1, 15)); // 5 noches = $750

// Intentar reservar cuando está ocupada
h.reservar(LocalDate.of(2025, 1, 12), LocalDate.of(2025, 1, 14)); // Debe lanzar excepción

// Liberar y calcular ingresos
h.liberar();
double ingresos = h.getIngresosTotales(); // $750
```

**Solución Modelo:**

```java
public enum TipoHabitacion {
    SIMPLE, DOBLE, SUITE
}

public enum EstadoHabitacion {
    DISPONIBLE, OCUPADA, MANTENIMIENTO
}

public class Habitacion {
    private final int numero; // Inmutable
    private TipoHabitacion tipo;
    private double precioPorNoche;
    private EstadoHabitacion estado;
    private List<Reserva> historialReservas;
    
    public Habitacion(int numero, TipoHabitacion tipo, double precioPorNoche) {
        if (numero <= 0) {
            throw new IllegalArgumentException("Número de habitación inválido");
        }
        this.numero = numero;
        this.tipo = tipo;
        setPrecioPorNoche(precioPorNoche);
        this.estado = EstadoHabitacion.DISPONIBLE;
        this.historialReservas = new ArrayList<>();
    }
    
    public void setPrecioPorNoche(double precio) {
        if (precio <= 0) {
            throw new IllegalArgumentException("El precio debe ser mayor a 0");
        }
        this.precioPorNoche = precio;
    }
    
    public void reservar(LocalDate fechaInicio, LocalDate fechaFin) {
        if (estado != EstadoHabitacion.DISPONIBLE) {
            throw new IllegalStateException("La habitación no está disponible");
        }
        if (fechaInicio.isAfter(fechaFin) || fechaInicio.isBefore(LocalDate.now())) {
            throw new IllegalArgumentException("Fechas inválidas");
        }
        
        Reserva reserva = new Reserva(fechaInicio, fechaFin, precioPorNoche);
        historialReservas.add(reserva);
        estado = EstadoHabitacion.OCUPADA;
    }
    
    public void liberar() {
        if (estado != EstadoHabitacion.OCUPADA) {
            throw new IllegalStateException("La habitación no está ocupada");
        }
        estado = EstadoHabitacion.DISPONIBLE;
    }
    
    public void ponerEnMantenimiento() {
        if (estado == EstadoHabitacion.OCUPADA) {
            throw new IllegalStateException("No se puede poner en mantenimiento una habitación ocupada");
        }
        estado = EstadoHabitacion.MANTENIMIENTO;
    }
    
    public double getIngresosTotales() {
        return historialReservas.stream()
            .mapToDouble(Reserva::calcularCosto)
            .sum();
    }
    
    public int getNumero() {
        return numero;
    }
    
    public TipoHabitacion getTipo() {
        return tipo;
    }
    
    public EstadoHabitacion getEstado() {
        return estado;
    }
    
    public double getPrecioPorNoche() {
        return precioPorNoche;
    }
    
    // Retornar copia del historial (encapsulación)
    public List<Reserva> getHistorialReservas() {
        return new ArrayList<>(historialReservas);
    }
    
    // Clase interna para representar reservas
    private static class Reserva {
        private final LocalDate fechaInicio;
        private final LocalDate fechaFin;
        private final double precioPorNoche;
        
        public Reserva(LocalDate fechaInicio, LocalDate fechaFin, double precioPorNoche) {
            this.fechaInicio = fechaInicio;
            this.fechaFin = fechaFin;
            this.precioPorNoche = precioPorNoche;
        }
        
        public double calcularCosto() {
            long noches = ChronoUnit.DAYS.between(fechaInicio, fechaFin);
            return noches * precioPorNoche;
        }
    }
}
```

**Tests unitarios:**

```java
class HabitacionTest {
    @Test
    void testReservarHabitacionDisponible() {
        Habitacion h = new Habitacion(101, TipoHabitacion.DOBLE, 150.0);
        LocalDate inicio = LocalDate.of(2025, 1, 10);
        LocalDate fin = LocalDate.of(2025, 1, 15);
        
        h.reservar(inicio, fin);
        
        assertEquals(EstadoHabitacion.OCUPADA, h.getEstado());
        assertEquals(750.0, h.getIngresosTotales(), 0.01); // 5 noches * $150
    }
    
    @Test
    void testReservarHabitacionOcupadaLanzaExcepcion() {
        Habitacion h = new Habitacion(101, TipoHabitacion.DOBLE, 150.0);
        h.reservar(LocalDate.of(2025, 1, 10), LocalDate.of(2025, 1, 15));
        
        assertThrows(IllegalStateException.class, () -> {
            h.reservar(LocalDate.of(2025, 1, 12), LocalDate.of(2025, 1, 14));
        });
    }
    
    @Test
    void testPrecioNegativoLanzaExcepcion() {
        assertThrows(IllegalArgumentException.class, () -> {
            new Habitacion(101, TipoHabitacion.SIMPLE, -50.0);
        });
    }
    
    @Test
    void testLiberar() {
        Habitacion h = new Habitacion(101, TipoHabitacion.DOBLE, 150.0);
        h.reservar(LocalDate.of(2025, 1, 10), LocalDate.of(2025, 1, 15));
        h.liberar();
        
        assertEquals(EstadoHabitacion.DISPONIBLE, h.getEstado());
    }
}
```

**Rúbrica:**
- Encapsulación correcta (25%): Atributos privados, getters/setters apropiados
- Validaciones (25%): Todas las validaciones necesarias implementadas
- Lógica de negocio (30%): Estados y transiciones correctas
- Tests (20%): Cobertura completa de casos

---

### ⭐⭐⭐ NIVEL 3: Desafíos (Optimización)

#### Ejercicio 3.1: Sistema de Versionado de Documentos

**Enunciado:**
Diseña e implementa un sistema de versionado de documentos que cumpla con:

1. Un documento tiene: id, título, contenido, autor, fecha de creación
2. Cada modificación crea una nueva versión inmutable
3. Se puede recuperar cualquier versión anterior
4. Se puede comparar diferencias entre versiones
5. Implementar "undo" para volver a la versión anterior
6. El contenido de versiones antiguas no debe ser modificable
7. Máximo 10 versiones almacenadas (las más antiguas se eliminan)

**Solución Modelo:**

```java
public class Documento {
    private final String id;
    private String titulo;
    private String autor;
    private Deque<Version> versiones; // Stack de versiones
    private static final int MAX_VERSIONES = 10;
    
    public Documento(String id, String titulo, String contenidoInicial, String autor) {
        this.id = id;
        this.titulo = titulo;
        this.autor = autor;
        this.versiones = new LinkedList<>();
        
        // Crear versión inicial
        agregarVersion(contenidoInicial);
    }
    
    public void modificar(String nuevoContenido) {
        agregarVersion(nuevoContenido);
    }
    
    private void agregarVersion(String contenido) {
        Version nuevaVersion = new Version(
            versiones.size() + 1,
            contenido,
            LocalDateTime.now()
        );
        
        versiones.addLast(nuevaVersion);
        
        // Eliminar versiones antiguas si excede el máximo
        if (versiones.size() > MAX_VERSIONES) {
            versiones.removeFirst();
        }
    }
    
    public boolean undo() {
        if (versiones.size() <= 1) {
            return false; // No se puede deshacer la versión inicial
        }
        versiones.removeLast();
        return true;
    }
    
    public String getContenidoActual() {
        return versiones.getLast().getContenido();
    }
    
    public Version getVersion(int numeroVersion) {
        return versiones.stream()
            .filter(v -> v.getNumero() == numeroVersion)
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("Versión no encontrada"));
    }
    
    public List<Version> getHistorialVersiones() {
        return new ArrayList<>(versiones); // Copia defensiva
    }
    
    public String compararVersiones(int version1, int version2) {
        Version v1 = getVersion(version1);
        Version v2 = getVersion(version2);
        
        // Algoritmo simple de diferencias
        String[] lineas1 = v1.getContenido().split("\n");
        String[] lineas2 = v2.getContenido().split("\n");
        
        StringBuilder diff = new StringBuilder();
        int maxLineas = Math.max(lineas1.length, lineas2.length);
        
        for (int i = 0; i < maxLineas; i++) {
            String linea1 = i < lineas1.length ? lineas1[i] : "";
            String linea2 = i < lineas2.length ? lineas2[i] : "";
            
            if (!linea1.equals(linea2)) {
                diff.append("Línea ").append(i + 1).append(":\n");
                diff.append("  V").append(version1).append(": ").append(linea1).append("\n");
                diff.append("  V").append(version2).append(": ").append(linea2).append("\n");
            }
        }
        
        return diff.toString();
    }
    
    public String getId() { return id; }
    public String getTitulo() { return titulo; }
    public String getAutor() { return autor; }
    public int getNumeroVersionActual() { return versiones.getLast().getNumero(); }
    
    // Clase inmutable para representar versiones
    public static class Version {
        private final int numero;
        private final String contenido;
        private final LocalDateTime fecha;
        
        private Version(int numero, String contenido, LocalDateTime fecha) {
            this.numero = numero;
            this.contenido = contenido;
            this.fecha = fecha;
        }
        
        public int getNumero() { return numero; }
        public String getContenido() { return contenido; }
        public LocalDateTime getFecha() { return fecha; }
        
        @Override
        public String toString() {
            return String.format("Versión %d (%s)", numero, fecha);
        }
    }
}
```

**Tests:**

```java
class DocumentoTest {
    @Test
    void testCrearDocumentoConVersionInicial() {
        Documento doc = new Documento("DOC1", "Mi Documento", "Contenido inicial", "Juan");
        
        assertEquals("Contenido inicial", doc.getContenidoActual());
        assertEquals(1, doc.getNumeroVersionActual());
    }
    
    @Test
    void testModificarCreaVersion() {
        Documento doc = new Documento("DOC1", "Mi Documento", "V1", "Juan");
        doc.modificar("V2");
        doc.modificar("V3");
        
        assertEquals("V3", doc.getContenidoActual());
        assertEquals(3, doc.getNumeroVersionActual());
        assertEquals(3, doc.getHistorialVersiones().size());
    }
    
    @Test
    void testUndo() {
        Documento doc = new Documento("DOC1", "Mi Documento", "V1", "Juan");
        doc.modificar("V2");
        doc.modificar("V3");
        
        doc.undo();
        assertEquals("V2", doc.getContenidoActual());
        
        doc.undo();
        assertEquals("V1", doc.getContenidoActual());
        
        // No se puede deshacer más
        assertFalse(doc.undo());
    }
    
    @Test
    void testMaxVersiones() {
        Documento doc = new Documento("DOC1", "Mi Documento", "V1", "Juan");
        
        // Crear 12 versiones (excede el máximo de 10)
        for (int i = 2; i <= 12; i++) {
            doc.modificar("V" + i);
        }
        
        assertEquals(10, doc.getHistorialVersiones().size());
        assertEquals("V12", doc.getContenidoActual());
    }
    
    @Test
    void testCompararVersiones() {
        Documento doc = new Documento("DOC1", "Mi Documento", "Línea 1\nLínea 2", "Juan");
        doc.modificar("Línea 1\nLínea 2 modificada");
        
        String diff = doc.compararVersiones(1, 2);
        assertTrue(diff.contains("Línea 2"));
    }
}
```

**Rúbrica:**
- Diseño e inmutabilidad (30%): Versiones inmutables correctamente implementadas
- Gestión de versiones (30%): Stack de versiones con límite funciona correctamente
- Funcionalidades avanzadas (25%): Undo y comparación implementados
- Tests y documentación (15%): Cobertura completa

---

### ⭐⭐⭐⭐ NIVEL 4: Proyectos Integradores

#### Ejercicio 4.1: Sistema Bancario Completo

**Enunciado:**
Diseña un sistema bancario que incluya:

**Requisitos:**
1. Múltiples tipos de cuenta (Ahorro, Corriente, Inversión)
2. Transacciones con historial completo
3. Límites de retiro diarios
4. Intereses calculados automáticamente
5. Transferencias entre cuentas
6. Estado de cuenta con formato profesional
7. Auditoría completa de todas las operaciones

**Especificaciones técnicas:**
- Cuenta de Ahorro: Interés 2% anual, máximo 5 retiros/mes
- Cuenta Corriente: Sin interés, retiros ilimitados, puede tener sobregiro hasta $500
- Cuenta de Inversión: Interés 5% anual, mínimo $10,000, retiros penalizados 1%

Esta solución requiere aproximadamente 500-700 líneas de código con encapsulación adecuada y tests completos.

**Rúbrica:**
- Arquitectura (20%): Jerarquía de clases bien diseñada
- Encapsulación (25%): Todos los principios aplicados correctamente
- Lógica de negocio (30%): Todas las reglas implementadas
- Testing (15%): >80% cobertura
- Documentación (10%): Javadoc completo

---

## DIAGNÓSTICO DE ERRORES COMUNES

### Error 1: Exposición Innecesaria de Estado

```java
// ❌ INCORRECTO
public class Usuario {
    public List<String> roles = new ArrayList<>();
}

// Cliente puede modificar directamente
usuario.roles.add("ADMIN"); // ¡Peligro!

// ✅ CORRECTO
public class Usuario {
    private List<String> roles = new ArrayList<>();
    
    public void agregarRol(String rol) {
        if (esRolValido(rol)) {
            roles.add(rol);
        }
    }
    
    public List<String> getRoles() {
        return Collections.unmodifiableList(roles);
    }
}
```

**Causa raíz**: No aplicar encapsulación a colecciones
**Impacto**: Estado inconsistente, violación de reglas de negocio
**Solución**: Métodos de negocio + copias defensivas

---

### Error 2: Setters sin Validación

```java
// ❌ INCORRECTO
public void setEdad(int edad) {
    this.edad = edad; // No valida
}

// ✅ CORRECTO
public void setEdad(int edad) {
    if (edad < 0 || edad > 150) {
        throw new IllegalArgumentException("Edad inválida: " + edad);
    }
    this.edad = edad;
}
```

---

## CRITERIOS DE EVALUACIÓN GENERAL

| Criterio | Peso | Descripción |
|----------|------|-------------|
| Atributos privados | 20% | Todos los atributos declarados como private |
| Validación | 25% | Validaciones completas en setters y constructores |
| Inmutabilidad | 15% | Atributos que no deben cambiar son final |
| Copias defensivas | 15% | Retorna copias de objetos mutables |
| Tell, Don't Ask | 15% | Lógica dentro del objeto correcto |
| Tests | 10% | Cobertura >70% con casos edge |

---

## RECURSOS PARA PRÁCTICA ADICIONAL

1. **Refactoring Guru**: Ejemplos interactivos de encapsulación
2. **Exercism.io**: Ejercicios de POO con mentores
3. **LeetCode**: Problemas de diseño de clases
4. **GitHub**: Analizar proyectos open source bien diseñados (Spring, Guava)
