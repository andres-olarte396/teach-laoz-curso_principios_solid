# Ejercicios: API Gateway y OCP

## ⭐ Ejercicio 1: Gateway Básico Extensible

Implementa un API Gateway simple que permita agregar nuevas rutas sin modificar el código core.

**Requisitos**:
```typescript
interface Route {
    pattern: string;
    method: HttpMethod;
    handler: (req: Request) => Promise<Response>;
}

class ApiGateway {
    // Permitir registro dinámico de rutas
    registerRoute(route: Route): void;
    
    // Procesar request sin conocer rutas específicas
    async handle(req: Request): Promise<Response>;
}
```

**Tareas**:
1. Implementar sistema de routing dinámico
2. Agregar middleware pipeline
3. Soportar path parameters (`/users/:id`)
4. Query string parsing

<details>
<summary>💡 Solución</summary>

```typescript
type RouteHandler = (req: Request, params: RouteParams) => Promise<Response>;

interface RouteParams {
    [key: string]: string;
}

class Route {
    private regex: RegExp;
    private paramNames: string[] = [];
    
    constructor(
        public pattern: string,
        public method: HttpMethod,
        public handler: RouteHandler
    ) {
        this.compilePattern();
    }
    
    private compilePattern(): void {
        const paramPattern = /:([a-zA-Z_][a-zA-Z0-9_]*)/g;
        let match;
        
        while ((match = paramPattern.exec(this.pattern)) !== null) {
            this.paramNames.push(match[1]);
        }
        
        const regexPattern = this.pattern.replace(paramPattern, '([^/]+)');
        this.regex = new RegExp(`^${regexPattern}$`);
    }
    
    matches(path: string, method: HttpMethod): RouteParams | null {
        if (this.method !== method) return null;
        
        const match = this.regex.exec(path);
        if (!match) return null;
        
        const params: RouteParams = {};
        this.paramNames.forEach((name, index) => {
            params[name] = match[index + 1];
        });
        
        return params;
    }
}

class ApiGateway {
    private routes: Route[] = [];
    private middlewares: Middleware[] = [];
    
    use(middleware: Middleware): this {
        this.middlewares.push(middleware);
        return this;
    }
    
    registerRoute(pattern: string, method: HttpMethod, handler: RouteHandler): this {
        this.routes.push(new Route(pattern, method, handler));
        return this;
    }
    
    async handle(req: Request): Promise<Response> {
        // Apply middlewares
        for (const middleware of this.middlewares) {
            const result = await middleware.process(req);
            if (result) return result; // Early return
        }
        
        // Find matching route
        for (const route of this.routes) {
            const params = route.matches(req.path, req.method);
            if (params) {
                return await route.handler(req, params);
            }
        }
        
        return new Response(404, 'Not Found');
    }
}

// Uso
const gateway = new ApiGateway()
    .use(new LoggingMiddleware())
    .use(new AuthMiddleware())
    .registerRoute('/users/:id', 'GET', async (req, params) => {
        const user = await userService.getUser(params.id);
        return Response.ok(user);
    })
    .registerRoute('/products/:id/reviews', 'GET', async (req, params) => {
        const reviews = await reviewService.getReviews(params.id);
        return Response.ok(reviews);
    });
```

</details>

---

## ⭐⭐ Ejercicio 2: API Composition con Fallbacks

Implementa un endpoint que agregue datos de múltiples microservicios con estrategia de fallback si alguno falla.

**Endpoint**: `GET /api/products/:id/complete`

**Debe combinar**:
- Product details (obligatorio)
- Reviews (opcional)
- Recommendations (opcional)
- Inventory (opcional)

**Requisitos**:
1. Si Product falla → 404
2. Si servicios opcionales fallan → continuar con datos parciales
3. Timeout de 5 segundos por servicio
4. Cachear respuestas por 60 segundos

<details>
<summary>💡 Solución</summary>

```typescript
interface ProductCompositionResult {
    product: Product;
    reviews?: Review[];
    recommendations?: Product[];
    inventory?: InventoryInfo;
    warnings: string[];
}

class ProductCompositionHandler implements RouteHandler {
    constructor(
        private productService: ProductServiceClient,
        private reviewService: ReviewServiceClient,
        private recommendationService: RecommendationServiceClient,
        private inventoryService: InventoryServiceClient,
        private cache: CacheService
    ) {}
    
    async handle(req: Request, params: RouteParams): Promise<Response> {
        const productId = params.id;
        const cacheKey = `product:composition:${productId}`;
        
        // Check cache
        const cached = await this.cache.get(cacheKey);
        if (cached) {
            return Response.ok(cached);
        }
        
        const warnings: string[] = [];
        
        // Mandatory: Product details
        const product = await this.withTimeout(
            () => this.productService.getProduct(productId),
            5000
        );
        
        if (!product) {
            return Response.notFound('Product not found');
        }
        
        // Optional: Parallel calls with fallbacks
        const [reviews, recommendations, inventory] = await Promise.allSettled([
            this.withTimeout(() => this.reviewService.getReviews(productId), 5000),
            this.withTimeout(() => this.recommendationService.getSimilar(productId), 5000),
            this.withTimeout(() => this.inventoryService.getStock(productId), 5000)
        ]);
        
        const result: ProductCompositionResult = {
            product,
            warnings
        };
        
        if (reviews.status === 'fulfilled') {
            result.reviews = reviews.value;
        } else {
            warnings.push('Reviews unavailable');
        }
        
        if (recommendations.status === 'fulfilled') {
            result.recommendations = recommendations.value;
        } else {
            warnings.push('Recommendations unavailable');
        }
        
        if (inventory.status === 'fulfilled') {
            result.inventory = inventory.value;
        } else {
            warnings.push('Inventory unavailable');
        }
        
        // Cache for 60 seconds
        await this.cache.set(cacheKey, result, 60);
        
        return Response.ok(result);
    }
    
    private async withTimeout<T>(
        fn: () => Promise<T>,
        timeoutMs: number
    ): Promise<T> {
        return Promise.race([
            fn(),
            new Promise<T>((_, reject) => 
                setTimeout(() => reject(new Error('Timeout')), timeoutMs)
            )
        ]);
    }
}
```

</details>

---

## ⭐⭐⭐ Ejercicio 3: Versionado de APIs

Implementa un sistema de versionado que soporte:
- URL versioning (`/api/v1/users`, `/api/v2/users`)
- Header versioning (`Accept: application/vnd.api+json; version=2`)
- Deprecation warnings para versiones antiguas

**Requisitos**:
1. Múltiples versiones activas simultáneamente
2. Respuesta 410 Gone para versiones desactivadas
3. Header `Sunset` indicando fecha de deprecación
4. Transformación automática entre versiones

<details>
<summary>💡 Solución</summary>

```java
// Version metadata
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface ApiVersion {
    String value();
    String sunsetDate() default "";
    boolean deprecated() default false;
}

// Version extractor
public interface VersionExtractor {
    Optional<String> extractVersion(HttpServletRequest request);
}

public class UrlVersionExtractor implements VersionExtractor {
    private static final Pattern VERSION_PATTERN = Pattern.compile("/api/v(\\d+)/.*");
    
    @Override
    public Optional<String> extractVersion(HttpServletRequest request) {
        Matcher matcher = VERSION_PATTERN.matcher(request.getRequestURI());
        return matcher.matches() ? Optional.of(matcher.group(1)) : Optional.empty();
    }
}

public class HeaderVersionExtractor implements VersionExtractor {
    @Override
    public Optional<String> extractVersion(HttpServletRequest request) {
        String acceptHeader = request.getHeader("Accept");
        if (acceptHeader == null) return Optional.empty();
        
        Matcher matcher = Pattern.compile("version=(\\d+)").matcher(acceptHeader);
        return matcher.find() ? Optional.of(matcher.group(1)) : Optional.empty();
    }
}

// Versioned handlers registry
public class VersionedHandlerRegistry {
    private final Map<String, ApiHandler> handlers = new HashMap<>();
    private final Map<String, LocalDate> sunsetDates = new HashMap<>();
    
    public void register(ApiHandler handler) {
        ApiVersion annotation = handler.getClass().getAnnotation(ApiVersion.class);
        if (annotation == null) {
            throw new IllegalArgumentException("Handler must be annotated with @ApiVersion");
        }
        
        handlers.put(annotation.value(), handler);
        
        if (!annotation.sunsetDate().isEmpty()) {
            sunsetDates.put(annotation.value(), LocalDate.parse(annotation.sunsetDate()));
        }
    }
    
    public Optional<ApiHandler> getHandler(String version) {
        return Optional.ofNullable(handlers.get(version));
    }
    
    public boolean isDeprecated(String version) {
        return sunsetDates.containsKey(version);
    }
    
    public Optional<LocalDate> getSunsetDate(String version) {
        return Optional.ofNullable(sunsetDates.get(version));
    }
}

// Gateway with version support
@Component
public class VersionedApiGateway {
    private final List<VersionExtractor> extractors;
    private final VersionedHandlerRegistry registry;
    
    public VersionedApiGateway(VersionedHandlerRegistry registry) {
        this.registry = registry;
        this.extractors = List.of(
            new UrlVersionExtractor(),
            new HeaderVersionExtractor()
        );
    }
    
    public ResponseEntity<?> handle(HttpServletRequest request) {
        // Extract version
        String version = extractors.stream()
            .map(e -> e.extractVersion(request))
            .filter(Optional::isPresent)
            .map(Optional::get)
            .findFirst()
            .orElse("1"); // Default to v1
        
        // Get handler
        Optional<ApiHandler> handler = registry.getHandler(version);
        if (handler.isEmpty()) {
            return ResponseEntity.status(410).body("API version " + version + " is no longer supported");
        }
        
        // Check deprecation
        HttpHeaders headers = new HttpHeaders();
        if (registry.isDeprecated(version)) {
            registry.getSunsetDate(version).ifPresent(date -> {
                headers.add("Sunset", date.format(DateTimeFormatter.RFC_1123_DATE_TIME));
                headers.add("Deprecation", "true");
                headers.add("Link", "</api/v" + (Integer.parseInt(version) + 1) + ">; rel=\"successor-version\"");
            });
        }
        
        // Execute handler
        Object result = handler.get().handle(request);
        return ResponseEntity.ok().headers(headers).body(result);
    }
}

// Example handlers
@ApiVersion("1")
public class UserApiV1Handler implements ApiHandler {
    @Override
    public Object handle(HttpServletRequest request) {
        // V1 response format
        return new UserV1Response(userId, fullName, email);
    }
}

@ApiVersion(value = "2", sunsetDate = "2026-06-01", deprecated = true)
public class UserApiV2Handler implements ApiHandler {
    @Override
    public Object handle(HttpServletRequest request) {
        // V2 response format (will be sunset)
        return new UserV2Response(userId, firstName, lastName, email, avatar);
    }
}

@ApiVersion("3")
public class UserApiV3Handler implements ApiHandler {
    @Override
    public Object handle(HttpServletRequest request) {
        // V3 response format (current)
        return new UserV3Response(
            userId,
            profile: new Profile(firstName, lastName, avatar),
            contacts: new Contacts(email, phone),
            preferences: userPreferences
        );
    }
}
```

</details>

---

## ⭐⭐⭐⭐ Ejercicio 4: BFF Completo con Transformaciones

Implementa un sistema completo de Backend for Frontend con:
- Mobile BFF (datos optimizados)
- Web BFF (datos completos)
- Admin BFF (datos + permisos)
- GraphQL BFF (query flexible)

**Casos de uso**:
1. Mobile solicita home screen → 5 productos + 4 categorías
2. Web solicita home screen → 20 productos + todas categorías + recomendaciones
3. Admin solicita dashboard → estadísticas + usuarios activos + logs
4. GraphQL permite queries personalizados

<details>
<summary>💡 Solución Parcial (Mobile + Web BFF)</summary>

```python
# ============================================
# Mobile BFF
# ============================================
from fastapi import FastAPI, Depends
from typing import List

mobile_app = FastAPI()

class MobileBFFService:
    def __init__(
        self,
        product_service: ProductService,
        category_service: CategoryService,
        user_service: UserService
    ):
        self.product_service = product_service
        self.category_service = category_service
        self.user_service = user_service
    
    async def get_home_screen(self, user_id: str) -> MobileHomeResponse:
        # Optimized for mobile bandwidth
        featured = await self.product_service.get_featured(limit=5)
        categories = await self.category_service.get_top(limit=4)
        user = await self.user_service.get_minimal_profile(user_id)
        
        return MobileHomeResponse(
            user_name=user.first_name,  # Solo nombre
            featured_products=[
                MobileProductCard(
                    id=p.id,
                    name=p.name,
                    price=p.price,
                    thumbnail=p.thumbnail_small  # Imagen pequeña
                ) for p in featured
            ],
            categories=[
                MobileCategoryCard(
                    id=c.id,
                    name=c.name,
                    icon=c.icon_url
                ) for c in categories
            ]
        )

@mobile_app.get("/mobile/home")
async def mobile_home(
    user_id: str = Depends(get_current_user_id),
    bff: MobileBFFService = Depends()
) -> MobileHomeResponse:
    return await bff.get_home_screen(user_id)

# ============================================
# Web BFF
# ============================================
web_app = FastAPI()

class WebBFFService:
    def __init__(
        self,
        product_service: ProductService,
        category_service: CategoryService,
        user_service: UserService,
        recommendation_service: RecommendationService
    ):
        self.product_service = product_service
        self.category_service = category_service
        self.user_service = user_service
        self.recommendation_service = recommendation_service
    
    async def get_home_screen(self, user_id: str) -> WebHomeResponse:
        # Full data for desktop
        featured, categories, user, recommendations = await asyncio.gather(
            self.product_service.get_featured(limit=20),
            self.category_service.get_all(),
            self.user_service.get_full_profile(user_id),
            self.recommendation_service.get_personalized(user_id, limit=10)
        )
        
        return WebHomeResponse(
            user_profile=WebUserProfile(
                id=user.id,
                full_name=user.full_name,
                email=user.email,
                avatar=user.avatar_url,
                member_since=user.created_at,
                loyalty_points=user.loyalty_points
            ),
            featured_products=[
                WebProductCard(
                    id=p.id,
                    name=p.name,
                    description=p.short_description,
                    price=p.price,
                    discount=p.discount,
                    rating=p.average_rating,
                    review_count=p.review_count,
                    images=p.image_urls,  # Múltiples imágenes
                    availability=p.stock_status
                ) for p in featured
            ],
            categories=[
                WebCategoryCard(
                    id=c.id,
                    name=c.name,
                    description=c.description,
                    image=c.banner_url,
                    product_count=c.product_count,
                    subcategories=[s.name for s in c.subcategories]
                ) for c in categories
            ],
            recommendations=[
                WebProductCard.from_product(r) for r in recommendations
            ]
        )

@web_app.get("/web/home")
async def web_home(
    user_id: str = Depends(get_current_user_id),
    bff: WebBFFService = Depends()
) -> WebHomeResponse:
    return await bff.get_home_screen(user_id)
```

</details>

---

## Recursos

- **Spring Cloud Gateway**: https://spring.io/projects/spring-cloud-gateway
- **Kong API Gateway**: https://konghq.com/
- **Pattern**: Backend for Frontend - Sam Newman
- **Book**: "Building Microservices" - O'Reilly
