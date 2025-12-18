# Shared Libraries y Dependency Inversion Principle

## Objetivos
- Diseñar shared libraries que respeten DIP
- Evitar acoplamiento entre microservicios vía libraries
- Implementar versioning strategies
- Crear abstractions compartidas sin implementaciones

## 1. El Problema: Shared Libraries Acopladas

```
❌ MAL: Shared Library con Lógica de Negocio
┌─────────────────────────────────────┐
│   common-library.jar                │
│  ┌──────────────────────────────┐   │
│  │ PaymentProcessor            │   │ ← Lógica de negocio
│  │ + process()                 │   │
│  │ + validate()                │   │
│  │                             │   │
│  │ InventoryChecker            │   │ ← Lógica de negocio
│  │ + checkStock()              │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
         ▲              ▲
         │              │
    ┌────┴────┐    ┌───┴──────┐
    │ Order   │    │ Product  │
    │ Service │    │ Service  │
    └─────────┘    └──────────┘

Problemas:
- Servicios acoplados vía library
- Cambio en library requiere redeploy de todos
- Violación de SRP (library hace demasiado)
- Testing difícil
```

```
✅ BIEN: Shared Library solo con Abstractions
┌─────────────────────────────────────┐
│   domain-contracts.jar              │
│  ┌──────────────────────────────┐   │
│  │ interface PaymentGateway    │   │ ← Solo contratos
│  │ interface InventoryService  │   │
│  │ data class Money            │   │ ← Value objects
│  │ data class Address          │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
         ▲              ▲
         │              │
    ┌────┴────┐    ┌───┴──────┐
    │ Payment │    │Inventory │ ← Implementaciones separadas
    │ Service │    │ Service  │
    └─────────┘    └──────────┘
```

## 2. Tipos de Shared Libraries Válidas

### 2.1 Domain Contracts (Interfaces y DTOs)

```java
// shared-contracts/src/main/java/com/ecommerce/contracts/payment/PaymentGateway.java
public interface PaymentGateway {
    /**
     * Process payment for given amount.
     * Implementations should be idempotent.
     */
    PaymentResult charge(PaymentRequest request);
    
    /**
     * Refund previously processed payment.
     */
    RefundResult refund(String paymentId, Money amount);
}

// shared-contracts/src/main/java/com/ecommerce/contracts/payment/PaymentRequest.java
@Value  // Immutable
public class PaymentRequest {
    String orderId;
    Money amount;
    PaymentMethod method;
    String idempotencyKey;  // For retry safety
}

// shared-contracts/src/main/java/com/ecommerce/contracts/payment/PaymentResult.java
public sealed interface PaymentResult {
    record Success(String paymentId, LocalDateTime processedAt) implements PaymentResult {}
    record Declined(String reason) implements PaymentResult {}
    record Error(String message, Exception cause) implements PaymentResult {}
}
```

### 2.2 Value Objects (Sin Lógica de Negocio)

```kotlin
// shared-domain/src/main/kotlin/com/ecommerce/domain/Money.kt
@JvmInline
value class Money private constructor(
    private val cents: Long  // Store as cents to avoid floating point
) : Comparable<Money> {
    
    val amount: BigDecimal
        get() = BigDecimal(cents).divide(BigDecimal(100))
    
    val currency: Currency
        get() = Currency.USD  // Simplified
    
    operator fun plus(other: Money): Money {
        return Money(cents + other.cents)
    }
    
    operator fun minus(other: Money): Money {
        require(cents >= other.cents) { "Cannot subtract to negative" }
        return Money(cents - other.cents)
    }
    
    operator fun times(multiplier: Int): Money {
        return Money(cents * multiplier)
    }
    
    override fun compareTo(other: Money): Int {
        return cents.compareTo(other.cents)
    }
    
    companion object {
        fun zero(): Money = Money(0)
        
        fun of(amount: BigDecimal): Money {
            val cents = amount.multiply(BigDecimal(100)).toLong()
            return Money(cents)
        }
        
        fun of(dollars: Int, cents: Int): Money {
            return Money(dollars * 100L + cents)
        }
    }
}

// shared-domain/src/main/kotlin/com/ecommerce/domain/Address.kt
data class Address(
    val street: String,
    val city: String,
    val state: String,
    val zipCode: String,
    val country: String
) {
    init {
        require(street.isNotBlank()) { "Street cannot be blank" }
        require(zipCode.matches(Regex("\\d{5}"))) { "Invalid ZIP code" }
    }
    
    // No business logic - just validation
}
```

### 2.3 Common Utilities (Stateless Functions)

```typescript
// shared-utils/src/validation.ts
export class ValidationUtils {
    static isValidEmail(email: string): boolean {
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        return emailRegex.test(email);
    }
    
    static isValidPhoneNumber(phone: string): boolean {
        const phoneRegex = /^\+?[1-9]\d{1,14}$/;
        return phoneRegex.test(phone);
    }
    
    // Stateless, pure functions only
}

// shared-utils/src/retry.ts
export class RetryUtils {
    static async withRetry<T>(
        fn: () => Promise<T>,
        maxAttempts: number = 3,
        backoff: (attempt: number) => number = (a) => Math.pow(2, a) * 1000
    ): Promise<T> {
        let lastError: Error;
        
        for (let attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                return await fn();
            } catch (error) {
                lastError = error as Error;
                if (attempt < maxAttempts) {
                    await new Promise(resolve => setTimeout(resolve, backoff(attempt)));
                }
            }
        }
        
        throw lastError!;
    }
}
```

### 2.4 Cross-Cutting Concerns (Logging, Tracing)

```python
# shared_observability/logging.py
import logging
import json
from typing import Any, Dict
from contextvars import ContextVar

# Correlation ID context
correlation_id: ContextVar[str] = ContextVar('correlation_id', default='')

class StructuredLogger:
    """Structured logging for microservices."""
    
    def __init__(self, service_name: str):
        self.service_name = service_name
        self.logger = logging.getLogger(service_name)
    
    def info(self, message: str, **kwargs: Any) -> None:
        self._log(logging.INFO, message, kwargs)
    
    def error(self, message: str, exception: Exception = None, **kwargs: Any) -> None:
        if exception:
            kwargs['exception'] = {
                'type': type(exception).__name__,
                'message': str(exception)
            }
        self._log(logging.ERROR, message, kwargs)
    
    def _log(self, level: int, message: str, extra: Dict[str, Any]) -> None:
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'service': self.service_name,
            'correlation_id': correlation_id.get(),
            'message': message,
            **extra
        }
        self.logger.log(level, json.dumps(log_entry))

# shared_observability/tracing.py
from functools import wraps
from typing import Callable, TypeVar, ParamSpec

P = ParamSpec('P')
T = TypeVar('T')

def traced(operation_name: str) -> Callable[[Callable[P, T]], Callable[P, T]]:
    """Decorator to trace function calls."""
    def decorator(func: Callable[P, T]) -> Callable[P, T]:
        @wraps(func)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> T:
            span_id = generate_span_id()
            
            # Add trace headers
            add_trace_header('x-span-id', span_id)
            add_trace_header('x-operation-name', operation_name)
            
            try:
                result = func(*args, **kwargs)
                record_span_success(span_id, operation_name)
                return result
            except Exception as e:
                record_span_error(span_id, operation_name, e)
                raise
        
        return wrapper
    return decorator
```

## 3. Versioning Strategies

### 3.1 Semantic Versioning

```xml
<!-- pom.xml -->
<groupId>com.ecommerce</groupId>
<artifactId>shared-contracts</artifactId>
<version>2.1.0</version>

<!-- 
Versioning:
- MAJOR: Breaking changes (2.0.0 -> 3.0.0)
- MINOR: New features, backward compatible (2.0.0 -> 2.1.0)
- PATCH: Bug fixes (2.1.0 -> 2.1.1)
-->
```

### 3.2 Compatibility Layers

```java
// Version 1.x - Original interface
public interface PaymentGateway {
    PaymentResult charge(String orderId, BigDecimal amount);
}

// Version 2.x - Enhanced with Money type
public interface PaymentGatewayV2 {
    PaymentResult charge(String orderId, Money amount);
}

// Compatibility adapter
public class PaymentGatewayAdapter implements PaymentGateway {
    private final PaymentGatewayV2 delegate;
    
    @Override
    public PaymentResult charge(String orderId, BigDecimal amount) {
        // Adapt old interface to new
        return delegate.charge(orderId, Money.of(amount));
    }
}
```

### 3.3 Deprecation Strategy

```kotlin
// shared-contracts/src/main/kotlin/PaymentGateway.kt
interface PaymentGateway {
    
    @Deprecated(
        message = "Use charge(PaymentRequest) instead",
        replaceWith = ReplaceWith("charge(PaymentRequest(orderId, amount, PaymentMethod.CREDIT_CARD))"),
        level = DeprecationLevel.WARNING  // Will be ERROR in v3.0
    )
    fun charge(orderId: String, amount: Money): PaymentResult
    
    // New method (available since v2.0)
    fun charge(request: PaymentRequest): PaymentResult
}

// Provide migration period (6 months)
// v2.0.0 (Jan 2024): Both methods available
// v2.5.0 (Apr 2024): Old method WARNING
// v3.0.0 (Jul 2024): Old method removed
```

## 4. Dependency Management

### 4.1 Bill of Materials (BOM)

```xml
<!-- ecommerce-bom/pom.xml -->
<project>
    <groupId>com.ecommerce</groupId>
    <artifactId>ecommerce-bom</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>
    
    <dependencyManagement>
        <dependencies>
            <!-- Shared libraries versions -->
            <dependency>
                <groupId>com.ecommerce</groupId>
                <artifactId>shared-contracts</artifactId>
                <version>2.1.0</version>
            </dependency>
            <dependency>
                <groupId>com.ecommerce</groupId>
                <artifactId>shared-domain</artifactId>
                <version>1.5.0</version>
            </dependency>
            <dependency>
                <groupId>com.ecommerce</groupId>
                <artifactId>shared-observability</artifactId>
                <version>1.2.0</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>

<!-- order-service/pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.ecommerce</groupId>
            <artifactId>ecommerce-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- No version needed - managed by BOM -->
    <dependency>
        <groupId>com.ecommerce</groupId>
        <artifactId>shared-contracts</artifactId>
    </dependency>
</dependencies>
```

### 4.2 Gradle Version Catalog

```toml
# gradle/libs.versions.toml
[versions]
shared-contracts = "2.1.0"
shared-domain = "1.5.0"
shared-observability = "1.2.0"

[libraries]
ecommerce-contracts = { module = "com.ecommerce:shared-contracts", version.ref = "shared-contracts" }
ecommerce-domain = { module = "com.ecommerce:shared-domain", version.ref = "shared-domain" }
ecommerce-observability = { module = "com.ecommerce:shared-observability", version.ref = "shared-observability" }

[bundles]
ecommerce-shared = ["ecommerce-contracts", "ecommerce-domain", "ecommerce-observability"]
```

```kotlin
// order-service/build.gradle.kts
dependencies {
    implementation(libs.bundles.ecommerce.shared)
}
```

## 5. Testing Shared Libraries

```java
// shared-contracts/src/test/java/PaymentGatewayContract.java
public abstract class PaymentGatewayContractTest {
    
    protected abstract PaymentGateway createGateway();
    
    @Test
    public void shouldChargeSuccessfully() {
        PaymentGateway gateway = createGateway();
        
        PaymentRequest request = new PaymentRequest(
            "order-123",
            Money.of(99, 99),
            PaymentMethod.CREDIT_CARD,
            "idempotency-key-1"
        );
        
        PaymentResult result = gateway.charge(request);
        
        assertThat(result).isInstanceOf(PaymentResult.Success.class);
    }
    
    @Test
    public void shouldBeIdempotent() {
        PaymentGateway gateway = createGateway();
        
        PaymentRequest request = new PaymentRequest(
            "order-123",
            Money.of(99, 99),
            PaymentMethod.CREDIT_CARD,
            "same-idempotency-key"
        );
        
        PaymentResult result1 = gateway.charge(request);
        PaymentResult result2 = gateway.charge(request);  // Same key
        
        assertThat(result1).isEqualTo(result2);
    }
}

// payment-service/src/test/java/StripePaymentGatewayTest.java
public class StripePaymentGatewayTest extends PaymentGatewayContractTest {
    
    @Override
    protected PaymentGateway createGateway() {
        return new StripePaymentGateway(stripeMock);
    }
}
```

## Resumen

### Qué Incluir en Shared Libraries

| ✅ Permitido | ❌ Evitar |
|--------------|-----------|
| Interfaces/Contracts | Implementaciones de negocio |
| Value Objects | Entities con estado |
| DTOs inmutables | Repositorios |
| Utilidades stateless | Services con lógica |
| Logging/Tracing helpers | Database schemas |
| Validation rules (declarativas) | Configuration management |

### Best Practices

1. ✅ **Shared library = Contratos, no implementaciones**
2. ✅ **Semantic versioning estricto**
3. ✅ **Deprecation warnings antes de breaking changes**
4. ✅ **BOM para gestión centralizada de versiones**
5. ✅ **Contract tests para validar implementaciones**
6. ❌ **Evitar lógica de negocio en libraries**
7. ❌ **Evitar dependencias pesadas (frameworks)**
8. ❌ **Evitar shared databases via libraries**

### Cuándo NO Usar Shared Library

- Lógica específica de un servicio
- Implementaciones de abstractions
- Configuración específica
- Database migrations
- API clients (mejor usar OpenAPI codegen)
