# Testing de Código SOLID

## Unit Testing Simplificado

**Código SOLID** = Testing fácil (dependencias inyectables)

```java
// ✅ Fácil de testear (DIP + SRP)
class OrderService {
    private final OrderRepository repository;
    private final PaymentGateway gateway;
    
    public OrderService(OrderRepository repository, PaymentGateway gateway) {
        this.repository = repository;
        this.gateway = gateway;
    }
    
    public void placeOrder(Order order) {
        PaymentResult result = gateway.charge(order.getPayment());
        if (result.isSuccess()) {
            repository.save(order);
        }
    }
}

// Test con mocks
@Test
void testPlaceOrder() {
    OrderRepository mockRepo = mock(OrderRepository.class);
    PaymentGateway mockGateway = mock(PaymentGateway.class);
    when(mockGateway.charge(any())).thenReturn(PaymentResult.success());
    
    OrderService service = new OrderService(mockRepo, mockGateway);
    service.placeOrder(new Order());
    
    verify(mockRepo).save(any(Order.class));
}
```

## Integration Testing

**Interfaces claras** (ISP) = Tests de integración enfocados

```java
@Test
void testOrderFlow() {
    // Usar implementaciones reales de infraestructura
    OrderRepository realRepo = new MySQLOrderRepository(testDataSource);
    PaymentGateway testGateway = new TestPaymentGateway();
    
    OrderService service = new OrderService(realRepo, testGateway);
    service.placeOrder(createTestOrder());
    
    Order saved = realRepo.findById(1L);
    assertEquals(OrderStatus.COMPLETED, saved.getStatus());
}
```

## Contract Testing

**LSP** = Todos los subtipos pasan mismos tests

```java
abstract class PaymentGatewayContractTest {
    protected abstract PaymentGateway createGateway();
    
    @Test
    void shouldChargeSuccessfully() {
        PaymentGateway gateway = createGateway();
        PaymentResult result = gateway.charge(validPayment());
        assertTrue(result.isSuccess());
    }
    
    @Test
    void shouldRejectInvalidCard() {
        PaymentGateway gateway = createGateway();
        PaymentResult result = gateway.charge(invalidPayment());
        assertFalse(result.isSuccess());
    }
}

class StripeGatewayTest extends PaymentGatewayContractTest {
    protected PaymentGateway createGateway() {
        return new StripePaymentGateway();
    }
}

class PayPalGatewayTest extends PaymentGatewayContractTest {
    protected PaymentGateway createGateway() {
        return new PayPalPaymentGateway();
    }
}
```

## Resumen

**SOLID + Testing** = Unit tests simples, integration tests enfocados, contract tests para LSP.
