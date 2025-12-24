# Bounded Contexts y Single Responsibility Principle

## Objetivos de Aprendizaje
- Comprender la relación entre Domain-Driven Design y SRP
- Identificar bounded contexts en sistemas complejos
- Aplicar SRP en la descomposición de monolitos a microservicios
- Evitar el anti-patrón Distributed Monolith

## 1. Domain-Driven Design y SOLID

### Bounded Contexts como Responsabilidades

En DDD, un **Bounded Context** define límites claros donde ciertos modelos y términos tienen significado específico. Esto se alinea directamente con SRP:

```java
// ❌ MAL: Modelo único para todo el sistema (viola SRP)
public class Order {
    private Long id;
    private String customerName;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
    
    // Responsabilidad de ventas
    public void calculateDiscounts() { }
    
    // Responsabilidad de inventario
    public void reserveStock() { }
    
    // Responsabilidad de envíos
    public void scheduleDelivery() { }
    
    // Responsabilidad de contabilidad
    public void generateInvoice() { }
}
```

```java
// ✅ BIEN: Modelos separados por bounded context

// Sales Context
public class SalesOrder {
    private OrderId id;
    private CustomerId customerId;
    private Money totalAmount;
    private List<OrderLine> lines;
    
    public DiscountResult applyPromotions(List<Promotion> promotions) {
        // Lógica de descuentos
    }
}

// Inventory Context
public class StockReservation {
    private ReservationId id;
    private OrderId orderId;
    private List<ReservedItem> items;
    
    public ReservationResult reserve() {
        // Lógica de reserva
    }
}

// Shipping Context
public class Shipment {
    private ShipmentId id;
    private OrderId orderId;
    private Address destination;
    private ShippingMethod method;
    
    public void schedule(LocalDateTime preferredDate) {
        // Lógica de programación
    }
}

// Billing Context
public class Invoice {
    private InvoiceId id;
    private OrderId orderId;
    private Money amount;
    private List<InvoiceLineItem> lineItems;
    
    public void generate() {
        // Lógica de facturación
    }
}
```

### Context Mapping

```
┌─────────────┐         ┌──────────────┐
│   Sales     │◄────────│  Inventory   │
│   Context   │ Customer│   Context    │
└─────────────┘ Supplier└──────────────┘
      │                         │
      │ Shared Kernel           │ Conformist
      │                         │
      ▼                         ▼
┌─────────────┐         ┌──────────────┐
│   Billing   │         │   Shipping   │
│   Context   │         │   Context    │
└─────────────┘         └──────────────┘
```

## 2. Microservicios vs Monolitos Modulares

### Cuándo Usar Microservicios

```java
// Criteria para separar en microservicio
public interface MicroserviceDecisionCriteria {
    
    // ✅ Indicadores positivos
    boolean hasDifferentScalingNeeds();      // Inventory alta carga
    boolean hasDifferentDataConsistency();   // Billing strong, Shipping eventual
    boolean managedByDifferentTeams();       // Autonomía de equipos
    boolean hasDifferentReleaseSchedules();  // Billing mensual, Sales diario
    boolean hasDifferentTechnologyNeeds();   // Payment PCI-compliant stack
    
    // ❌ Indicadores negativos (mantener monolito modular)
    boolean hasHighCommunicationOverhead();  // Muchas llamadas sincrónicas
    boolean requiresDistributedTransactions(); // ACID necesario
    boolean hasSimpleDomain();               // CRUD simple
}
```

### Ejemplo: E-commerce Decomposition

```typescript
// Monolito modular (BIEN para empezar)
class EcommerceMonolith {
    // Módulos con boundaries claros
    private catalogModule: CatalogModule;
    private orderModule: OrderModule;
    private paymentModule: PaymentModule;
    private shippingModule: ShippingModule;
    
    // Cada módulo expone interfaces claras
    interface CatalogModule {
        searchProducts(query: SearchQuery): Product[];
        getProduct(id: ProductId): Product;
    }
    
    interface OrderModule {
        createOrder(cart: Cart): OrderResult;
        getOrder(id: OrderId): Order;
    }
}

// Evolución a microservicios (cuando sea necesario)
class CatalogService {
    private repository: ProductRepository;
    private searchEngine: SearchEngine;
    
    async searchProducts(query: SearchQuery): Promise<Product[]> {
        return this.searchEngine.search(query);
    }
}

class OrderService {
    private orderRepository: OrderRepository;
    private catalogClient: CatalogServiceClient;  // HTTP/gRPC
    private paymentClient: PaymentServiceClient;
    
    async createOrder(cart: Cart): Promise<OrderResult> {
        // Comunicación distribuida
        const products = await this.catalogClient.getProducts(cart.itemIds);
        const payment = await this.paymentClient.processPayment(cart.total);
        
        if (payment.success) {
            return this.orderRepository.save(new Order(cart, payment));
        }
    }
}
```

## 3. Anti-patrón: Distributed Monolith

### Características del Distributed Monolith

```python
# ❌ MAL: Microservicios acoplados (Distributed Monolith)

class OrderService:
    def create_order(self, cart: Cart) -> Order:
        # Dependencia síncrona de 5+ servicios
        customer = self.customer_service.get_customer(cart.customer_id)  # HTTP
        products = self.catalog_service.get_products(cart.item_ids)      # HTTP
        inventory = self.inventory_service.reserve(cart.items)           # HTTP
        pricing = self.pricing_service.calculate(cart.items)             # HTTP
        shipping = self.shipping_service.estimate(cart, customer.address) # HTTP
        
        # Si cualquier servicio falla, todo falla
        # No hay autonomía real
        # Latencia acumulativa
        # Debugging distribuido complejo
        
        return Order(customer, products, inventory, pricing, shipping)
```

```python
# ✅ BIEN: Microservicios autónomos

class OrderService:
    def __init__(
        self,
        order_repository: OrderRepository,
        event_bus: EventBus,
        customer_cache: Cache,  # Cache local de datos de Customer
        catalog_cache: Cache    # Cache local de datos de Catalog
    ):
        self.order_repository = order_repository
        self.event_bus = event_bus
        self.customer_cache = customer_cache
        self.catalog_cache = catalog_cache
    
    async def create_order(self, cart: Cart) -> Order:
        # Datos esenciales en cache/replicated read model
        customer = self.customer_cache.get(cart.customer_id)
        products = self.catalog_cache.get_many(cart.item_ids)
        
        # Crear orden con eventual consistency
        order = Order.create(cart, customer, products)
        await self.order_repository.save(order)
        
        # Comunicación asíncrona vía eventos
        await self.event_bus.publish(OrderCreated(
            order_id=order.id,
            items=order.items,
            customer_id=customer.id,
            shipping_address=customer.default_address
        ))
        
        # Otros servicios reaccionan independientemente:
        # - Inventory escucha OrderCreated → reserva stock
        # - Shipping escucha OrderCreated → programa envío
        # - Billing escucha OrderCreated → genera factura
        
        return order
```

### Comparación de Arquitecturas

| Aspecto | Monolito Modular | Distributed Monolith ❌ | Microservicios ✅ |
|---------|------------------|------------------------|------------------|
| **Deployment** | Single unit | Múltiples servicios acoplados | Servicios independientes |
| **Comunicación** | In-process | Síncrona HTTP/gRPC | Async eventos + cache |
| **Consistencia** | ACID transacciones | Distributed transactions | Eventual consistency |
| **Fallo en cascada** | No aplica | Alto riesgo | Circuit breakers |
| **Autonomía de equipos** | Baja | Baja | Alta |
| **Complejidad operacional** | Baja | Muy alta | Alta (pero justificada) |

## 4. Identificación de Bounded Contexts

### Event Storming

```
1. Identificar eventos de dominio:
   - OrderPlaced
   - PaymentProcessed
   - InventoryReserved
   - ShipmentScheduled
   - InvoiceGenerated

2. Agrupar eventos relacionados:
   ┌─────────────────────────────┐
   │ Sales Context               │
   │ - OrderPlaced               │
   │ - OrderCancelled            │
   │ - PromotionApplied          │
   └─────────────────────────────┘
   
   ┌─────────────────────────────┐
   │ Inventory Context           │
   │ - InventoryReserved         │
   │ - StockReplenished          │
   │ - ReservationExpired        │
   └─────────────────────────────┘

3. Identificar agregados:
   - Sales: Order (aggregate root)
   - Inventory: StockItem (aggregate root)
   - Shipping: Shipment (aggregate root)
```

### Domain Language

```csharp
// Sales Context
public class SalesOrder
{
    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public OrderStatus Status { get; private set; }  // Draft, Confirmed, Cancelled
    public Money Total { get; }
    
    // Sales domain language
    public void Confirm() { /* ... */ }
    public void ApplyDiscount(DiscountCode code) { /* ... */ }
}

// Fulfillment Context (different language!)
public class FulfillmentOrder
{
    public FulfillmentOrderId Id { get; }
    public OrderId SalesOrderId { get; }  // Reference to Sales context
    public FulfillmentStatus Status { get; private set; }  // Pending, Packed, Shipped
    
    // Fulfillment domain language
    public void AllocateWarehouse(WarehouseId warehouse) { /* ... */ }
    public void Pack() { /* ... */ }
    public void Ship(Carrier carrier) { /* ... */ }
}
```

## 5. Migración Gradual

### Strangler Fig Pattern

```java
// Fase 1. Monolito original
public class OrderController {
    @PostMapping("/orders")
    public OrderResponse createOrder(@RequestBody OrderRequest request) {
        return monolithService.createOrder(request);
    }
}

// Fase 2: Router introduce abstracción
public class OrderController {
    private final OrderRouter router;
    
    @PostMapping("/orders")
    public OrderResponse createOrder(@RequestBody OrderRequest request) {
        if (featureFlags.isEnabled("new-order-service")) {
            return router.routeToMicroservice(request);
        } else {
            return router.routeToMonolith(request);
        }
    }
}

// Fase 3: Microservicio completo
public class OrderController {
    private final OrderService orderService;  // Nuevo microservicio
    
    @PostMapping("/orders")
    public OrderResponse createOrder(@RequestBody OrderRequest request) {
        return orderService.createOrder(request);
    }
}
```

## Ejemplo Completo: Sistema de Pedidos

```typescript
// ============================================
// SALES BOUNDED CONTEXT
// ============================================
export class SalesOrder {
    private constructor(
        public readonly id: OrderId,
        public readonly customerId: CustomerId,
        private lines: OrderLine[],
        private status: OrderStatus
    ) {}
    
    static create(customerId: CustomerId, lines: OrderLine[]): SalesOrder {
        const order = new SalesOrder(
            OrderId.generate(),
            customerId,
            lines,
            OrderStatus.DRAFT
        );
        
        // Domain event
        DomainEvents.publish(new OrderCreated(order.id, customerId, lines));
        
        return order;
    }
    
    confirm(): void {
        if (this.status !== OrderStatus.DRAFT) {
            throw new Error('Only draft orders can be confirmed');
        }
        
        this.status = OrderStatus.CONFIRMED;
        DomainEvents.publish(new OrderConfirmed(this.id));
    }
}

// ============================================
// INVENTORY BOUNDED CONTEXT
// ============================================
export class StockReservation {
    private constructor(
        public readonly id: ReservationId,
        public readonly orderId: OrderId,
        private items: Map<ProductId, Quantity>,
        private expiresAt: Date
    ) {}
    
    static async reserve(orderId: OrderId, items: Map<ProductId, Quantity>): Promise<StockReservation> {
        const reservation = new StockReservation(
            ReservationId.generate(),
            orderId,
            items,
            DateTime.now().plus({ hours: 24 }).toJSDate()
        );
        
        // Check availability
        for (const [productId, quantity] of items.entries()) {
            const available = await InventoryRepository.getAvailable(productId);
            if (available < quantity) {
                throw new InsufficientStockError(productId, available, quantity);
            }
        }
        
        DomainEvents.publish(new StockReserved(reservation.id, orderId, items));
        
        return reservation;
    }
}

// ============================================
// EVENT HANDLERS (Anti-Corruption Layer)
// ============================================
export class OrderEventHandlers {
    constructor(
        private inventoryClient: InventoryServiceClient,
        private shippingClient: ShippingServiceClient
    ) {}
    
    @EventHandler(OrderConfirmed)
    async onOrderConfirmed(event: OrderConfirmed): Promise<void> {
        // Traducir del modelo de Sales al modelo de Inventory
        const reservationRequest = {
            order_id: event.orderId.value,
            items: event.lines.map(line => ({
                product_id: line.productId.value,
                quantity: line.quantity
            }))
        };
        
        await this.inventoryClient.reserveStock(reservationRequest);
    }
    
    @EventHandler(StockReserved)
    async onStockReserved(event: StockReserved): Promise<void> {
        // Traducir del modelo de Inventory al modelo de Shipping
        const shipmentRequest = {
            order_id: event.orderId.value,
            items: Array.from(event.items.entries()).map(([productId, quantity]) => ({
                product_id: productId.value,
                quantity
            }))
        };
        
        await this.shippingClient.scheduleShipment(shipmentRequest);
    }
}
```

## Resumen

### Principios Clave

1. **Bounded Context = Responsabilidad**: Cada contexto tiene una sola razón para cambiar
2. **Autonomía**: Los microservicios deben poder desplegarse independientemente
3. **Eventual Consistency**: Usar eventos asíncronos reduce acoplamiento
4. **Anti-Corruption Layer**: Traducir entre modelos de diferentes contextos
5. **Migración Gradual**: Strangler Fig para evolucionar de monolito a microservicios

### Cuándo NO Usar Microservicios

- **Startups tempranas**: Domain todavía está evolucionando
- **Equipos pequeños**: < 10 desarrolladores
- **Dominios simples**: CRUD básico sin lógica compleja
- **Bajo tráfico**: < 1000 requests/día

### Cuándo SÍ Usar Microservicios

- **Escalado diferenciado**: Catalog tiene 1000x más lectura que Payment
- **Equipos múltiples**: > 3 equipos trabajando en el mismo codebase
- **Tecnologías especializadas**: Payment requiere PCI-compliant stack
- **Release independiente**: Marketing quiere deploys diarios, Finance semanales
