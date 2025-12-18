# Strategy Pattern para Extensibilidad

## Definición

El **Strategy Pattern** encapsula algoritmos intercambiables en clases separadas, permitiendo que el cliente elija cuál usar en tiempo de ejecución sin modificar el código que los usa.

## Estructura

```java
// Estrategia (interfaz)
public interface CompressionStrategy {
    byte[] compress(byte[] data);
}

// Estrategias concretas (implementaciones)
public class ZipCompression implements CompressionStrategy {
    public byte[] compress(byte[] data) {
        // Algoritmo ZIP
        return compressedData;
    }
}

public class RarCompression implements CompressionStrategy {
    public byte[] compress(byte[] data) {
        // Algoritmo RAR
        return compressedData;
    }
}

public class Lz4Compression implements CompressionStrategy {
    public byte[] compress(byte[] data) {
        // Algoritmo LZ4
        return compressedData;
    }
}

// Contexto (usa la estrategia)
public class FileCompressor {
    private CompressionStrategy strategy;
    
    public FileCompressor(CompressionStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void setStrategy(CompressionStrategy strategy) {
        this.strategy = strategy;
    }
    
    public byte[] compressFile(byte[] fileData) {
        return strategy.compress(fileData);
    }
}

// Uso
FileCompressor compressor = new FileCompressor(new ZipCompression());
byte[] compressed = compressor.compressFile(data);

// Cambiar estrategia en runtime
compressor.setStrategy(new RarCompression());
compressed = compressor.compressFile(data);
```

## Caso de Estudio: Sistema de Descuentos

### Antes (Sin Strategy)

```java
public class Order {
    public double calculateTotal() {
        double subtotal = getSubtotal();
        String customerType = customer.getType();
        
        if (customerType.equals("REGULAR")) {
            return subtotal;
        } else if (customerType.equals("PREMIUM")) {
            return subtotal * 0.9; // 10% discount
        } else if (customerType.equals("VIP")) {
            return subtotal * 0.8; // 20% discount
        } else if (customerType.equals("EMPLOYEE")) {
            return subtotal * 0.7; // 30% discount
        }
        return subtotal;
    }
}
```

### Después (Con Strategy)

```java
public interface DiscountStrategy {
    double applyDiscount(double subtotal);
}

public class NoDiscount implements DiscountStrategy {
    public double applyDiscount(double subtotal) {
        return subtotal;
    }
}

public class PremiumDiscount implements DiscountStrategy {
    public double applyDiscount(double subtotal) {
        return subtotal * 0.9;
    }
}

public class VipDiscount implements DiscountStrategy {
    public double applyDiscount(double subtotal) {
        return subtotal * 0.8;
    }
}

public class EmployeeDiscount implements DiscountStrategy {
    public double applyDiscount(double subtotal) {
        return subtotal * 0.7;
    }
}

public class Order {
    private DiscountStrategy discountStrategy;
    
    public Order(DiscountStrategy discountStrategy) {
        this.discountStrategy = discountStrategy;
    }
    
    public double calculateTotal() {
        double subtotal = getSubtotal();
        return discountStrategy.applyDiscount(subtotal);
    }
}

// Uso
Order regularOrder = new Order(new NoDiscount());
Order premiumOrder = new Order(new PremiumDiscount());
Order vipOrder = new Order(new VipDiscount());
```

## Ventajas del Strategy Pattern

1. **OCP**: Añadir nuevos descuentos (ej: `SeasonalDiscount`) no requiere modificar `Order`
2. **SRP**: Cada estrategia encapsula una lógica de descuento específica
3. **Testing**: Cada estrategia se puede testear aisladamente
4. **Runtime flexibility**: Cambiar estrategia dinámicamente

## Strategy con Dependencias

```java
public interface TaxCalculationStrategy {
    double calculate(double amount);
}

public class USTaxCalculation implements TaxCalculationStrategy {
    private TaxRateService taxRateService;
    
    public USTaxCalculation(TaxRateService taxRateService) {
        this.taxRateService = taxRateService;
    }
    
    public double calculate(double amount) {
        double rate = taxRateService.getRateFor("US");
        return amount * rate;
    }
}

public class EUTaxCalculation implements TaxCalculationStrategy {
    private TaxRateService taxRateService;
    
    public EUTaxCalculation(TaxRateService taxRateService) {
        this.taxRateService = taxRateService;
    }
    
    public double calculate(double amount) {
        double rate = taxRateService.getRateFor("EU");
        return amount * (1 + rate); // VAT incluido
    }
}
```

## Strategy Factory

```java
public class DiscountStrategyFactory {
    public static DiscountStrategy createStrategy(String customerType) {
        switch (customerType) {
            case "REGULAR": return new NoDiscount();
            case "PREMIUM": return new PremiumDiscount();
            case "VIP": return new VipDiscount();
            case "EMPLOYEE": return new EmployeeDiscount();
            default: return new NoDiscount();
        }
    }
}

// Uso
DiscountStrategy strategy = DiscountStrategyFactory.createStrategy(customer.getType());
Order order = new Order(strategy);
```

## Strategy vs Herencia

### Con Herencia (menos flexible)

```java
public abstract class Order {
    public double calculateTotal() {
        return applyDiscount(getSubtotal());
    }
    protected abstract double applyDiscount(double subtotal);
}

public class RegularOrder extends Order {
    protected double applyDiscount(double subtotal) { return subtotal; }
}

public class PremiumOrder extends Order {
    protected double applyDiscount(double subtotal) { return subtotal * 0.9; }
}
```

**Problema**: No puedes cambiar el tipo de orden en runtime. Un `RegularOrder` siempre será regular.

### Con Strategy (más flexible)

```java
Order order = new Order(new NoDiscount());
// Más tarde, el cliente se vuelve premium
order.setDiscountStrategy(new PremiumDiscount());
```

## Resumen

**Strategy Pattern** = Encapsular algoritmos en clases separadas e intercambiables.

**Beneficio OCP**: Nuevos algoritmos = nuevas clases, sin modificar contexto.

**Cuándo usar**: Múltiples variantes de un algoritmo, necesidad de cambiar en runtime.
