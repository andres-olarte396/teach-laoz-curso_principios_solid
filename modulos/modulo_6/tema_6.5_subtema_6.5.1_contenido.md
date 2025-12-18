# DIP en Java

## Interfaces y Abstracciones

```java
// ✅ Abstracción estable
interface PaymentProcessor {
    PaymentResult process(Payment payment);
}

// Implementaciones intercambiables
class StripePaymentProcessor implements PaymentProcessor {
    public PaymentResult process(Payment payment) {
        // Stripe API
    }
}

class PayPalPaymentProcessor implements PaymentProcessor {
    public PaymentResult process(Payment payment) {
        // PayPal API
    }
}
```

## Factory Pattern

```java
interface PaymentProcessorFactory {
    PaymentProcessor create(PaymentMethod method);
}

class DefaultPaymentProcessorFactory implements PaymentProcessorFactory {
    public PaymentProcessor create(PaymentMethod method) {
        return switch (method) {
            case CREDIT_CARD -> new StripePaymentProcessor();
            case PAYPAL -> new PayPalPaymentProcessor();
            case BANK_TRANSFER -> new BankTransferProcessor();
        };
    }
}
```

## Spring DI

```java
@Configuration
class PaymentConfig {
    @Bean
    @ConditionalOnProperty(name = "payment.provider", havingValue = "stripe")
    public PaymentProcessor stripeProcessor() {
        return new StripePaymentProcessor();
    }
    
    @Bean
    @ConditionalOnProperty(name = "payment.provider", havingValue = "paypal")
    public PaymentProcessor paypalProcessor() {
        return new PayPalPaymentProcessor();
    }
}

@Service
class OrderService {
    private final PaymentProcessor processor;
    
    @Autowired
    public OrderService(PaymentProcessor processor) {
        this.processor = processor;
    }
}
```

## Resumen

**DIP en Java** = Interfaces + Factory/DI para inversión de dependencias.
