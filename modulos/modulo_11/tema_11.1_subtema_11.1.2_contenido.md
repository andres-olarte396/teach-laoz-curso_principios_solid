# API Gateway y Open/Closed Principle

## Objetivos
- Diseñar API Gateways extensibles usando OCP
- Implementar composición de APIs sin modificar código existente
- Gestionar versioning de APIs
- Aplicar Backend for Frontend (BFF) pattern

## 1. API Gateway Pattern

### Responsabilidades del API Gateway

```java
// ❌ MAL: Gateway monolítico con lógica acoplada
public class ApiGateway {
    public Response handleRequest(Request request) {
        if (request.getPath().startsWith("/users")) {
            return userServiceClient.call(request);
        } else if (request.getPath().startsWith("/products")) {
            return productServiceClient.call(request);
        } else if (request.getPath().startsWith("/orders")) {
            // Composición hardcodeada
            User user = userServiceClient.getUser(request.getUserId());
            List<Product> products = productServiceClient.getProducts(request.getProductIds());
            Order order = orderServiceClient.createOrder(user, products);
            return new Response(order);
        }
        // Agregar nueva ruta requiere modificar esta clase ❌
    }
}
```

```java
// ✅ BIEN: Gateway extensible con OCP
public interface RouteHandler {
    boolean canHandle(HttpRequest request);
    CompletableFuture<HttpResponse> handle(HttpRequest request);
}

public class ApiGateway {
    private final List<RouteHandler> handlers;
    private final FallbackHandler fallbackHandler;
    
    public ApiGateway(List<RouteHandler> handlers) {
        this.handlers = new ArrayList<>(handlers);
        this.fallbackHandler = new NotFoundHandler();
    }
    
    public void registerHandler(RouteHandler handler) {
        this.handlers.add(handler);  // Extensión sin modificación
    }
    
    public CompletableFuture<HttpResponse> route(HttpRequest request) {
        return handlers.stream()
            .filter(h -> h.canHandle(request))
            .findFirst()
            .orElse(fallbackHandler)
            .handle(request);
    }
}

// Handlers específicos (abiertos para extensión)
public class UserServiceHandler implements RouteHandler {
    private final UserServiceClient client;
    
    @Override
    public boolean canHandle(HttpRequest request) {
        return request.getPath().matches("/api/users(/.*)?");
    }
    
    @Override
    public CompletableFuture<HttpResponse> handle(HttpRequest request) {
        return client.call(request)
            .thenApply(this::transform);
    }
}

public class OrderCompositionHandler implements RouteHandler {
    private final OrderOrchestrator orchestrator;
    
    @Override
    public boolean canHandle(HttpRequest request) {
        return request.getMethod() == HttpMethod.POST &&
               request.getPath().equals("/api/orders");
    }
    
    @Override
    public CompletableFuture<HttpResponse> handle(HttpRequest request) {
        return orchestrator.createOrderWithDependencies(request)
            .thenApply(order -> HttpResponse.ok(order));
    }
}
```

## 2. API Composition Extensible

### Orchestration Pattern

```typescript
// Strategy pattern para composición
interface CompositionStrategy {
    execute(request: CompositionRequest): Promise<CompositionResult>;
}

class OrderCompositionStrategy implements CompositionStrategy {
    constructor(
        private userService: UserServiceClient,
        private productService: ProductServiceClient,
        private inventoryService: InventoryServiceClient,
        private pricingService: PricingServiceClient
    ) {}
    
    async execute(request: CompositionRequest): Promise<CompositionResult> {
        // Parallel calls
        const [user, products, inventory, pricing] = await Promise.all([
            this.userService.getUser(request.userId),
            this.productService.getProducts(request.productIds),
            this.inventoryService.checkAvailability(request.productIds),
            this.pricingService.calculatePrice(request.productIds, request.userId)
        ]);
        
        return {
            user,
            products: products.map((p, i) => ({
                ...p,
                available: inventory[i].quantity,
                price: pricing[i].finalPrice
            }))
        };
    }
}

class ProductDetailCompositionStrategy implements CompositionStrategy {
    constructor(
        private productService: ProductServiceClient,
        private reviewService: ReviewServiceClient,
        private recommendationService: RecommendationServiceClient
    ) {}
    
    async execute(request: CompositionRequest): Promise<CompositionResult> {
        const productId = request.params.productId;
        
        const product = await this.productService.getProduct(productId);
        
        // Non-blocking parallel calls for enrichment
        const [reviews, recommendations] = await Promise.allSettled([
            this.reviewService.getReviews(productId),
            this.recommendationService.getSimilar(productId)
        ]);
        
        return {
            product,
            reviews: reviews.status === 'fulfilled' ? reviews.value : [],
            recommendations: recommendations.status === 'fulfilled' ? recommendations.value : []
        };
    }
}

// Registry de strategies (OCP)
class CompositionRegistry {
    private strategies = new Map<string, CompositionStrategy>();
    
    register(route: string, strategy: CompositionStrategy): void {
        this.strategies.set(route, strategy);
    }
    
    getStrategy(route: string): CompositionStrategy | undefined {
        return this.strategies.get(route);
    }
}
```

## 3. API Versioning

### URL-Based Versioning con OCP

```csharp
// Abstract handler para versiones
public abstract class VersionedApiHandler
{
    public abstract string SupportedVersion { get; }
    public abstract bool CanHandle(HttpContext context);
    public abstract Task<IActionResult> HandleAsync(HttpContext context);
}

// V1 implementation
public class UserApiV1Handler : VersionedApiHandler
{
    public override string SupportedVersion => "v1";
    
    public override bool CanHandle(HttpContext context)
    {
        return context.Request.Path.StartsWithSegments("/api/v1/users");
    }
    
    public override async Task<IActionResult> HandleAsync(HttpContext context)
    {
        // V1 logic: Simple user DTO
        var user = await _userService.GetUserAsync(userId);
        return new OkObjectResult(new UserV1Dto
        {
            Id = user.Id,
            Name = user.FullName,
            Email = user.Email
        });
    }
}

// V2 implementation (extendido sin modificar V1)
public class UserApiV2Handler : VersionedApiHandler
{
    public override string SupportedVersion => "v2";
    
    public override bool CanHandle(HttpContext context)
    {
        return context.Request.Path.StartsWithSegments("/api/v2/users");
    }
    
    public override async Task<IActionResult> HandleAsync(HttpContext context)
    {
        // V2 logic: Enriched user DTO
        var user = await _userService.GetUserAsync(userId);
        var preferences = await _preferenceService.GetPreferencesAsync(userId);
        
        return new OkObjectResult(new UserV2Dto
        {
            Id = user.Id,
            Profile = new ProfileDto
            {
                FirstName = user.FirstName,
                LastName = user.LastName,
                Avatar = user.AvatarUrl
            },
            Email = user.Email,
            Preferences = preferences
        });
    }
}

// Gateway dispatcher
public class VersionedApiGateway
{
    private readonly List<VersionedApiHandler> _handlers;
    
    public VersionedApiGateway(IEnumerable<VersionedApiHandler> handlers)
    {
        _handlers = handlers.ToList();
    }
    
    public async Task<IActionResult> RouteAsync(HttpContext context)
    {
        var handler = _handlers.FirstOrDefault(h => h.CanHandle(context));
        
        if (handler == null)
        {
            return new NotFoundResult();
        }
        
        return await handler.HandleAsync(context);
    }
}
```

### Header-Based Versioning

```python
from abc import ABC, abstractmethod
from typing import Optional

class ApiVersion(ABC):
    @abstractmethod
    def supports_version(self, version: str) -> bool:
        pass
    
    @abstractmethod
    async def handle(self, request):
        pass

class ProductApiV1(ApiVersion):
    def supports_version(self, version: str) -> bool:
        return version == "1.0" or version == "1"
    
    async def handle(self, request):
        # V1. Basic product info
        product = await self.product_service.get_product(request.product_id)
        return {
            "id": product.id,
            "name": product.name,
            "price": product.price
        }

class ProductApiV2(ApiVersion):
    def supports_version(self, version: str) -> bool:
        return version == "2.0" or version == "2"
    
    async def handle(self, request):
        # V2: Includes inventory and images
        product = await self.product_service.get_product(request.product_id)
        inventory = await self.inventory_service.get_stock(request.product_id)
        
        return {
            "id": product.id,
            "name": product.name,
            "description": product.description,
            "pricing": {
                "currency": "USD",
                "amount": product.price,
                "discount": product.discount
            },
            "inventory": {
                "available": inventory.available,
                "warehouse": inventory.warehouse_id
            },
            "images": product.image_urls
        }

class VersionRouter:
    def __init__(self):
        self.versions: list[ApiVersion] = []
    
    def register(self, version: ApiVersion):
        self.versions.append(version)
    
    async def route(self, request):
        # Extract version from header
        version_header = request.headers.get("Api-Version", "1.0")
        
        for v in self.versions:
            if v.supports_version(version_header):
                return await v.handle(request)
        
        raise UnsupportedVersionError(version_header)
```

## 4. Backend for Frontend (BFF) Pattern

```java
// Mobile BFF (datos optimizados para móvil)
@RestController
@RequestMapping("/mobile-bff")
public class MobileBFFController {
    
    @GetMapping("/home")
    public MobileHomeResponse getHomeScreen(@AuthenticationPrincipal User user) {
        // Composición específica para mobile
        CompletableFuture<List<Product>> featuredProducts = 
            productService.getFeatured(5); // Solo 5 para mobile
        
        CompletableFuture<List<Category>> categories = 
            categoryService.getTop(8); // 8 categorías
        
        CompletableFuture<UserProfile> profile = 
            userService.getProfile(user.getId());
        
        return CompletableFuture.allOf(featuredProducts, categories, profile)
            .thenApply(v -> new MobileHomeResponse(
                featuredProducts.join(),
                categories.join(),
                profile.join().getFirstName() // Solo nombre, no todos los datos
            )).join();
    }
}

// Web BFF (datos más ricos para desktop)
@RestController
@RequestMapping("/web-bff")
public class WebBFFController {
    
    @GetMapping("/home")
    public WebHomeResponse getHomeScreen(@AuthenticationPrincipal User user) {
        // Composición específica para web
        CompletableFuture<List<Product>> featuredProducts = 
            productService.getFeatured(20); // 20 productos para grid
        
        CompletableFuture<List<Category>> categories = 
            categoryService.getAll(); // Todas las categorías
        
        CompletableFuture<List<Recommendation>> recommendations = 
            recommendationService.getPersonalized(user.getId(), 10);
        
        CompletableFuture<UserProfile> profile = 
            userService.getFullProfile(user.getId());
        
        return CompletableFuture.allOf(
            featuredProducts, categories, recommendations, profile
        ).thenApply(v -> new WebHomeResponse(
            featuredProducts.join(),
            categories.join(),
            recommendations.join(),
            profile.join()
        )).join();
    }
}
```

## 5. Middleware Pipeline (Chain of Responsibility + OCP)

```typescript
interface GatewayMiddleware {
    handle(context: GatewayContext, next: () => Promise<void>): Promise<void>;
}

class AuthenticationMiddleware implements GatewayMiddleware {
    async handle(context: GatewayContext, next: () => Promise<void>): Promise<void> {
        const token = context.request.headers['authorization'];
        
        if (!token) {
            context.response.status(401).send('Unauthorized');
            return;
        }
        
        context.user = await this.validateToken(token);
        await next();
    }
}

class RateLimitMiddleware implements GatewayMiddleware {
    constructor(private limiter: RateLimiter) {}
    
    async handle(context: GatewayContext, next: () => Promise<void>): Promise<void> {
        const allowed = await this.limiter.tryConsume(context.clientId);
        
        if (!allowed) {
            context.response.status(429).send('Too many requests');
            return;
        }
        
        await next();
    }
}

class LoggingMiddleware implements GatewayMiddleware {
    async handle(context: GatewayContext, next: () => Promise<void>): Promise<void> {
        const startTime = Date.now();
        
        await next();
        
        const duration = Date.now() - startTime;
        this.logger.info(`${context.request.method} ${context.request.path} - ${duration}ms`);
    }
}

class MiddlewarePipeline {
    private middlewares: GatewayMiddleware[] = [];
    
    use(middleware: GatewayMiddleware): this {
        this.middlewares.push(middleware);
        return this;
    }
    
    async execute(context: GatewayContext): Promise<void> {
        let index = 0;
        
        const next = async (): Promise<void> => {
            if (index < this.middlewares.length) {
                const middleware = this.middlewares[index++];
                await middleware.handle(context, next);
            }
        };
        
        await next();
    }
}

// Configuración (extensible sin modificar pipeline)
const pipeline = new MiddlewarePipeline()
    .use(new CorsMiddleware())
    .use(new LoggingMiddleware())
    .use(new AuthenticationMiddleware())
    .use(new RateLimitMiddleware(rateLimiter))
    .use(new CircuitBreakerMiddleware())
    .use(new CachingMiddleware(cache));
```

## Ejemplo Completo: API Gateway con Spring Cloud Gateway

```java
@Configuration
public class GatewayConfiguration {
    
    @Bean
    public RouteLocator customRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            // User Service Route
            .route("user-service", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .rewritePath("/api/users/(?<segment>.*)", "/users/${segment}")
                    .addRequestHeader("X-Gateway-Version", "1.0")
                    .circuitBreaker(c -> c.setName("userServiceCB"))
                )
                .uri("lb://user-service")
            )
            // Product Service Route with rate limiting
            .route("product-service", r -> r
                .path("/api/products/**")
                .filters(f -> f
                    .requestRateLimiter(c -> c
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver())
                    )
                    .retry(c -> c
                        .setRetries(3)
                        .setBackoff(Duration.ofMillis(100), Duration.ofSeconds(1), 2, true)
                    )
                )
                .uri("lb://product-service")
            )
            // Aggregation endpoint
            .route("order-aggregation", r -> r
                .path("/api/orders/checkout")
                .filters(f -> f
                    .filter(orderAggregationFilter())
                )
                .uri("no://op")  // No direct routing, handled by filter
            )
            .build();
    }
    
    @Bean
    public GatewayFilter orderAggregationFilter() {
        return (exchange, chain) -> {
            // Aggregate multiple service calls
            OrderCheckoutRequest request = exchange.getAttribute("request");
            
            return Mono.zip(
                userClient.getUser(request.getUserId()),
                productClient.getProducts(request.getProductIds()),
                pricingClient.calculateTotal(request.getItems())
            ).flatMap(tuple -> {
                User user = tuple.getT1();
                List<Product> products = tuple.getT2();
                PricingResult pricing = tuple.getT3();
                
                OrderCheckoutResponse response = new OrderCheckoutResponse(
                    user, products, pricing
                );
                
                return chain.filter(exchange.mutate()
                    .response(new ServerHttpResponseDecorator(exchange.getResponse()) {
                        @Override
                        public Mono<Void> writeWith(Publisher<? extends DataBuffer> body) {
                            return super.writeWith(Mono.just(serialize(response)));
                        }
                    })
                    .build());
            });
        };
    }
}
```

## Resumen

### Principios Aplicados

1. **OCP en Routing**: Agregar rutas sin modificar gateway core
2. **Strategy Pattern**: Composición personalizada por endpoint
3. **Chain of Responsibility**: Middleware pipeline extensible
4. **BFF Pattern**: Backends específicos por client type
5. **Versioning**: Evolución de APIs sin breaking changes

### Mejores Prácticas

- ✅ Usar middleware pipeline para cross-cutting concerns
- ✅ Implementar circuit breakers y rate limiting
- ✅ Cachear respuestas de servicios downstream
- ✅ Versioning desde el inicio
- ✅ BFF para diferentes client types
- ❌ No agregar lógica de negocio en gateway
- ❌ No hacer transformaciones complejas
