# Facade Pattern para Simplificación

## Problema

Subsistema complejo con muchas interfaces.

## Solución: Facade

```java
// Subsistema complejo
class CPU {
    void freeze() {}
    void jump(long position) {}
    void execute() {}
}

class Memory {
    void load(long position, byte[] data) {}
}

class HardDrive {
    byte[] read(long lba, int size) {}
}

// ✅ Facade simplifica
class ComputerFacade {
    private CPU cpu;
    private Memory memory;
    private HardDrive hardDrive;
    
    public void start() {
        cpu.freeze();
        memory.load(BOOT_ADDRESS, hardDrive.read(BOOT_SECTOR, SECTOR_SIZE));
        cpu.jump(BOOT_ADDRESS);
        cpu.execute();
    }
}

// Cliente usa interfaz simple
ComputerFacade computer = new ComputerFacade();
computer.start(); // En lugar de coordinar CPU, Memory, HardDrive
```

## Facade e ISP

Facade cumple ISP al exponer solo operaciones de alto nivel:

```java
// Sin Facade (cliente depende de muchas interfaces)
class Client {
    private OrderService orderService;
    private PaymentService paymentService;
    private InventoryService inventoryService;
    private NotificationService notificationService;
    
    public void placeOrder(Order order) {
        orderService.validate(order);
        paymentService.charge(order);
        inventoryService.reserve(order);
        notificationService.send(order);
    }
}

// ✅ Con Facade
interface OrderProcessingFacade {
    void processOrder(Order order);
}

class OrderFacade implements OrderProcessingFacade {
    // Coordina servicios internamente
    public void processOrder(Order order) {
        // Orquestación interna
    }
}

// Cliente depende solo de una interfaz
class Client {
    private OrderProcessingFacade facade;
    
    public void placeOrder(Order order) {
        facade.processOrder(order);
    }
}
```

## Resumen

**Facade** = Interfaz simplificada para subsistema complejo, reduce dependencias del cliente (ISP).
