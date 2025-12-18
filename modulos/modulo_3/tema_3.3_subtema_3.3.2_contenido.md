# Event-Driven Design para Extensibilidad

## Concepto

**Event-Driven Design** desacopla componentes mediante eventos: el emisor no conoce a los receptores, permitiendo añadir nuevos oyentes sin modificar el emisor.

## Patrón Observer

```java
// Evento
public class OrderPlacedEvent {
    private final Order order;
    private final LocalDateTime timestamp;
    
    public OrderPlacedEvent(Order order) {
        this.order = order;
        this.timestamp = LocalDateTime.now();
    }
    
    public Order getOrder() { return order; }
    public LocalDateTime getTimestamp() { return timestamp; }
}

// Oyente (Observer)
public interface OrderEventListener {
    void onOrderPlaced(OrderPlacedEvent event);
}

// Publicador (Subject)
public class OrderService {
    private List<OrderEventListener> listeners = new ArrayList<>();
    
    public void addListener(OrderEventListener listener) {
        listeners.add(listener);
    }
    
    public void removeListener(OrderEventListener listener) {
        listeners.remove(listener);
    }
    
    public void placeOrder(Order order) {
        // Procesar orden
        order.setStatus("CONFIRMED");
        saveOrder(order);
        
        // Notificar evento
        OrderPlacedEvent event = new OrderPlacedEvent(order);
        notifyListeners(event);
    }
    
    private void notifyListeners(OrderPlacedEvent event) {
        for (OrderEventListener listener : listeners) {
            listener.onOrderPlaced(event);
        }
    }
}

// Oyentes Concretos
public class EmailNotificationListener implements OrderEventListener {
    public void onOrderPlaced(OrderPlacedEvent event) {
        System.out.println("Sending email for order: " + event.getOrder().getId());
        // Enviar email
    }
}

public class InventoryListener implements OrderEventListener {
    public void onOrderPlaced(OrderPlacedEvent event) {
        System.out.println("Updating inventory for order: " + event.getOrder().getId());
        // Actualizar inventario
    }
}

public class AnalyticsListener implements OrderEventListener {
    public void onOrderPlaced(OrderPlacedEvent event) {
        System.out.println("Recording analytics for order: " + event.getOrder().getId());
        // Registrar métricas
    }
}

// Uso
OrderService orderService = new OrderService();
orderService.addListener(new EmailNotificationListener());
orderService.addListener(new InventoryListener());
orderService.addListener(new AnalyticsListener());

orderService.placeOrder(new Order());
// Output:
// Sending email for order: 123
// Updating inventory for order: 123
// Recording analytics for order: 123
```

**OCP**: Añadir `LoyaltyPointsListener` no requiere modificar `OrderService`.

## Event Bus

```java
public class EventBus {
    private Map<Class<?>, List<Consumer<?>>> subscribers = new HashMap<>();
    
    public <T> void subscribe(Class<T> eventType, Consumer<T> handler) {
        subscribers.computeIfAbsent(eventType, k -> new ArrayList<>()).add(handler);
    }
    
    public <T> void publish(T event) {
        Class<?> eventType = event.getClass();
        List<Consumer<?>> handlers = subscribers.get(eventType);
        if (handlers != null) {
            for (Consumer<?> handler : handlers) {
                ((Consumer<T>) handler).accept(event);
            }
        }
    }
}

// Uso
EventBus eventBus = new EventBus();

// Suscripciones
eventBus.subscribe(OrderPlacedEvent.class, event -> {
    System.out.println("Email: " + event.getOrder().getId());
});

eventBus.subscribe(OrderPlacedEvent.class, event -> {
    System.out.println("Inventory: " + event.getOrder().getId());
});

eventBus.subscribe(OrderPlacedEvent.class, event -> {
    System.out.println("Analytics: " + event.getOrder().getId());
});

// Publicación
eventBus.publish(new OrderPlacedEvent(order));
```

## Eventos Asíncronos

```java
public class AsyncEventBus {
    private ExecutorService executor = Executors.newFixedThreadPool(10);
    private Map<Class<?>, List<Consumer<?>>> subscribers = new HashMap<>();
    
    public <T> void subscribe(Class<T> eventType, Consumer<T> handler) {
        subscribers.computeIfAbsent(eventType, k -> new ArrayList<>()).add(handler);
    }
    
    public <T> void publish(T event) {
        Class<?> eventType = event.getClass();
        List<Consumer<?>> handlers = subscribers.get(eventType);
        if (handlers != null) {
            for (Consumer<?> handler : handlers) {
                executor.submit(() -> {
                    try {
                        ((Consumer<T>) handler).accept(event);
                    } catch (Exception e) {
                        System.err.println("Error processing event: " + e.getMessage());
                    }
                });
            }
        }
    }
}
```

## Eventos con Prioridad

```java
public class PriorityEventBus {
    private static class Subscription implements Comparable<Subscription> {
        final Consumer<?> handler;
        final int priority;
        
        Subscription(Consumer<?> handler, int priority) {
            this.handler = handler;
            this.priority = priority;
        }
        
        public int compareTo(Subscription other) {
            return Integer.compare(other.priority, this.priority); // Mayor prioridad primero
        }
    }
    
    private Map<Class<?>, List<Subscription>> subscribers = new HashMap<>();
    
    public <T> void subscribe(Class<T> eventType, Consumer<T> handler, int priority) {
        List<Subscription> subs = subscribers.computeIfAbsent(eventType, k -> new ArrayList<>());
        subs.add(new Subscription(handler, priority));
        Collections.sort(subs); // Ordenar por prioridad
    }
    
    public <T> void publish(T event) {
        Class<?> eventType = event.getClass();
        List<Subscription> subs = subscribers.get(eventType);
        if (subs != null) {
            for (Subscription sub : subs) {
                ((Consumer<T>) sub.handler).accept(event);
            }
        }
    }
}

// Uso
PriorityEventBus eventBus = new PriorityEventBus();

// Alta prioridad: validación
eventBus.subscribe(OrderPlacedEvent.class, event -> {
    System.out.println("PRIORITY 10: Validation");
}, 10);

// Media prioridad: email
eventBus.subscribe(OrderPlacedEvent.class, event -> {
    System.out.println("PRIORITY 5: Email");
}, 5);

// Baja prioridad: analytics
eventBus.subscribe(OrderPlacedEvent.class, event -> {
    System.out.println("PRIORITY 1: Analytics");
}, 1);

eventBus.publish(new OrderPlacedEvent(order));
// Output (en orden de prioridad):
// PRIORITY 10: Validation
// PRIORITY 5: Email
// PRIORITY 1: Analytics
```

## Eventos con Filtrado

```java
public interface EventFilter<T> {
    boolean accept(T event);
}

public class FilteredEventBus {
    private static class Subscription<T> {
        final Consumer<T> handler;
        final EventFilter<T> filter;
        
        Subscription(Consumer<T> handler, EventFilter<T> filter) {
            this.handler = handler;
            this.filter = filter;
        }
    }
    
    private Map<Class<?>, List<Subscription<?>>> subscribers = new HashMap<>();
    
    public <T> void subscribe(Class<T> eventType, Consumer<T> handler, EventFilter<T> filter) {
        subscribers.computeIfAbsent(eventType, k -> new ArrayList<>())
                   .add(new Subscription<>(handler, filter));
    }
    
    public <T> void publish(T event) {
        Class<?> eventType = event.getClass();
        List<Subscription<?>> subs = subscribers.get(eventType);
        if (subs != null) {
            for (Subscription<?> sub : subs) {
                Subscription<T> typedSub = (Subscription<T>) sub;
                if (typedSub.filter.accept(event)) {
                    typedSub.handler.accept(event);
                }
            }
        }
    }
}

// Uso
FilteredEventBus eventBus = new FilteredEventBus();

// Solo órdenes > $100
eventBus.subscribe(OrderPlacedEvent.class, 
    event -> System.out.println("High-value order: " + event.getOrder().getId()),
    event -> event.getOrder().getTotal() > 100
);

// Solo órdenes de clientes premium
eventBus.subscribe(OrderPlacedEvent.class,
    event -> System.out.println("Premium customer: " + event.getOrder().getId()),
    event -> event.getOrder().getCustomer().isPremium()
);
```

## Event Sourcing

```java
public abstract class Event {
    private final UUID id = UUID.randomUUID();
    private final LocalDateTime timestamp = LocalDateTime.now();
    
    public UUID getId() { return id; }
    public LocalDateTime getTimestamp() { return timestamp; }
}

public class OrderCreatedEvent extends Event {
    private final Order order;
    public OrderCreatedEvent(Order order) { this.order = order; }
    public Order getOrder() { return order; }
}

public class OrderShippedEvent extends Event {
    private final UUID orderId;
    private final String trackingNumber;
    public OrderShippedEvent(UUID orderId, String trackingNumber) {
        this.orderId = orderId;
        this.trackingNumber = trackingNumber;
    }
}

public class EventStore {
    private List<Event> events = new ArrayList<>();
    
    public void append(Event event) {
        events.add(event);
    }
    
    public List<Event> getEvents() {
        return new ArrayList<>(events);
    }
    
    public Order rebuildOrder(UUID orderId) {
        Order order = null;
        for (Event event : events) {
            if (event instanceof OrderCreatedEvent) {
                OrderCreatedEvent e = (OrderCreatedEvent) event;
                if (e.getOrder().getId().equals(orderId)) {
                    order = e.getOrder();
                }
            } else if (event instanceof OrderShippedEvent) {
                OrderShippedEvent e = (OrderShippedEvent) event;
                if (e.getOrderId().equals(orderId)) {
                    order.setStatus("SHIPPED");
                    order.setTrackingNumber(e.getTrackingNumber());
                }
            }
        }
        return order;
    }
}
```

## Frameworks Event-Driven

### Spring Events

```java
@Component
public class OrderEventPublisher {
    @Autowired
    private ApplicationEventPublisher publisher;
    
    public void placeOrder(Order order) {
        // Procesar orden
        publisher.publishEvent(new OrderPlacedEvent(this, order));
    }
}

@Component
public class EmailListener {
    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // Enviar email
    }
}
```

### Guava EventBus

```java
EventBus eventBus = new EventBus();

public class EmailService {
    @Subscribe
    public void handleOrder(OrderPlacedEvent event) {
        // Enviar email
    }
}

eventBus.register(new EmailService());
eventBus.post(new OrderPlacedEvent(order));
```

## Resumen

**Event-Driven Design** = Desacoplamiento mediante eventos y oyentes.

**OCP**: Nuevos oyentes = nuevas clases, sin modificar publicadores.

**Cuándo usar**: Múltiples reacciones a un evento, componentes desacoplados.
