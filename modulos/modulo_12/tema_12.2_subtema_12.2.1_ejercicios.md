# Ejercicios: CQRS Pattern Avanzado

## ⭐ Ejercicio 1. Identificar Violaciones de CQRS

Analiza este código e identifica por qué NO sigue CQRS:

```typescript
class OrderService {
    async createOrder(data: CreateOrderDto): Promise<Order> {
        const order = new Order(data);
        await this.db.save(order);
        return order;  // Retorna entidad completa
    }
    
    async getOrders(customerId: string): Promise<Order[]> {
        // Lee desde la misma tabla de write
        return await this.db.query(
            `SELECT o.*, c.name, c.email 
             FROM orders o 
             JOIN customers c ON o.customer_id = c.id
             WHERE o.customer_id = ?`,
            [customerId]
        );
    }
}
```

**Problemas a identificar**:
1. ¿Qué principio de CQRS se viola?
2. ¿Qué problemas de performance tiene?
3. ¿Cómo refactorizarías esto?

<details>
<summary>💡 Solución</summary>

**Violaciones CQRS**:

❌ **Problema 1**: No separa write/read models
- `createOrder` y `getOrders` usan la misma tabla `orders`
- Queries complejas con JOINs impactan performance de writes

❌ **Problema 2**: Command retorna estado
- `createOrder` retorna `Order` completa
- En CQRS, commands solo retornan ID o void

❌ **Problema 3**: Read no está optimizado
- JOIN en cada query es lento
- Sin denormalización

**Refactorización CQRS**:

```typescript
// ========== WRITE SIDE ==========

// Command (no retorna Order)
interface CreateOrderCommand {
    customerId: string;
    items: OrderItem[];
}

// Command Handler
class CreateOrderCommandHandler {
    constructor(
        private eventStore: EventStore,
        private eventBus: EventBus
    ) {}
    
    async handle(command: CreateOrderCommand): Promise<string> {
        // Validate
        const customer = await this.customerRepo.findById(command.customerId);
        if (!customer) throw new Error('Customer not found');
        
        // Create aggregate
        const order = new Order();
        const event = order.create(command, customer);
        
        // Save event
        await this.eventStore.save(event.orderId, event);
        
        // Publish
        await this.eventBus.publish(event);
        
        return event.orderId;  // Solo retorna ID
    }
}

// ========== READ SIDE ==========

// Read Model (denormalizado)
interface OrderView {
    orderId: string;
    customerId: string;
    customerName: string;      // ✅ Denormalizado
    customerEmail: string;     // ✅ Denormalizado
    items: OrderItemView[];
    total: number;
    status: string;
    createdAt: Date;
}

// Projection (construye read model)
class OrderViewProjection {
    async on(event: OrderCreatedEvent): Promise<void> {
        // Guardar vista denormalizada (SIN joins)
        await this.readDb.insert('order_views', {
            order_id: event.orderId,
            customer_id: event.customerId,
            customer_name: event.customerName,    // De evento
            customer_email: event.customerEmail,  // De evento
            items_json: JSON.stringify(event.items),
            total: event.total,
            status: 'PENDING',
            created_at: event.timestamp
        });
    }
}

// Query Service (lee desde read model)
class OrderQueryService {
    async getCustomerOrders(customerId: string): Promise<OrderView[]> {
        // ✅ Query simple, sin JOINs
        return await this.readDb.query(
            'SELECT * FROM order_views WHERE customer_id = ? ORDER BY created_at DESC',
            [customerId]
        );
    }
}
```

**Beneficios**:
- ✅ Write optimizado (solo inserts en event store)
- ✅ Read optimizado (sin joins, datos denormalizados)
- ✅ Escalabilidad (diferentes DBs para read/write)

</details>

---

## ⭐⭐ Ejercicio 2: Implementar CQRS con Múltiples Read Models

Crea un sistema de blog con CQRS que tenga 3 read models diferentes:

1. **ArticleListView**: Para listar artículos (título, autor, fecha, likes)
2. **ArticleDetailView**: Para mostrar artículo completo (con contenido, comentarios)
3. **AuthorStatsView**: Para dashboard del autor (total artículos, views, likes)

**Requisitos**:
- Command: `PublishArticleCommand`, `LikeArticleCommand`
- Events: `ArticlePublishedEvent`, `ArticleLikedEvent`
- 3 projections independientes

<details>
<summary>💡 Solución</summary>

```typescript
// ========== COMMANDS ==========

interface PublishArticleCommand {
    authorId: string;
    title: string;
    content: string;
    tags: string[];
}

interface LikeArticleCommand {
    articleId: string;
    userId: string;
}

// ========== EVENTS ==========

interface ArticlePublishedEvent {
    articleId: string;
    authorId: string;
    authorName: string;      // Denormalizado
    authorAvatar: string;    // Denormalizado
    title: string;
    content: string;
    tags: string[];
    version: number;
    timestamp: Date;
}

interface ArticleLikedEvent {
    articleId: string;
    userId: string;
    version: number;
    timestamp: Date;
}

// ========== WRITE SIDE ==========

class Article {
    private id: string;
    private authorId: string;
    private title: string;
    private content: string;
    private likes: Set<string> = new Set();
    private version: number = 0;
    
    publish(command: PublishArticleCommand, author: Author): ArticlePublishedEvent {
        if (command.title.length < 10) {
            throw new Error('Title too short');
        }
        
        if (command.content.length < 100) {
            throw new Error('Content too short');
        }
        
        return {
            articleId: generateId(),
            authorId: command.authorId,
            authorName: author.name,
            authorAvatar: author.avatarUrl,
            title: command.title,
            content: command.content,
            tags: command.tags,
            version: this.version + 1,
            timestamp: new Date()
        };
    }
    
    like(command: LikeArticleCommand): ArticleLikedEvent {
        if (this.likes.has(command.userId)) {
            throw new Error('User already liked this article');
        }
        
        return {
            articleId: command.articleId,
            userId: command.userId,
            version: this.version + 1,
            timestamp: new Date()
        };
    }
}

// ========== READ MODELS ==========

// Read Model 1. Lista de artículos
interface ArticleListView {
    articleId: string;
    title: string;
    authorName: string;
    authorAvatar: string;
    excerpt: string;         // Primeros 200 chars
    tags: string[];
    likeCount: number;
    publishedAt: Date;
}

// Read Model 2: Detalle de artículo
interface ArticleDetailView {
    articleId: string;
    title: string;
    content: string;         // Completo
    authorName: string;
    authorAvatar: string;
    tags: string[];
    likeCount: number;
    publishedAt: Date;
}

// Read Model 3: Stats del autor
interface AuthorStatsView {
    authorId: string;
    totalArticles: number;
    totalLikes: number;
    averageLikesPerArticle: number;
    lastPublishedAt: Date;
}

// ========== PROJECTIONS ==========

// Projection 1. Article List
class ArticleListProjection {
    async on(event: ArticlePublishedEvent): Promise<void> {
        await this.db.insert('article_list_views', {
            article_id: event.articleId,
            title: event.title,
            author_name: event.authorName,
            author_avatar: event.authorAvatar,
            excerpt: event.content.substring(0, 200),  // Truncado
            tags: event.tags,
            like_count: 0,
            published_at: event.timestamp
        });
    }
    
    async on(event: ArticleLikedEvent): Promise<void> {
        // Incrementar contador
        await this.db.execute(
            'UPDATE article_list_views SET like_count = like_count + 1 WHERE article_id = ?',
            [event.articleId]
        );
    }
}

// Projection 2: Article Detail
class ArticleDetailProjection {
    async on(event: ArticlePublishedEvent): Promise<void> {
        await this.db.insert('article_detail_views', {
            article_id: event.articleId,
            title: event.title,
            content: event.content,  // Completo
            author_name: event.authorName,
            author_avatar: event.authorAvatar,
            tags: event.tags,
            like_count: 0,
            published_at: event.timestamp
        });
    }
    
    async on(event: ArticleLikedEvent): Promise<void> {
        await this.db.execute(
            'UPDATE article_detail_views SET like_count = like_count + 1 WHERE article_id = ?',
            [event.articleId]
        );
    }
}

// Projection 3: Author Stats
class AuthorStatsProjection {
    async on(event: ArticlePublishedEvent): Promise<void> {
        // Upsert
        await this.db.execute(`
            INSERT INTO author_stats_views (
                author_id, total_articles, total_likes, last_published_at
            ) VALUES (?, 1, 0, ?)
            ON CONFLICT (author_id) DO UPDATE SET
                total_articles = total_articles + 1,
                last_published_at = ?
        `, [event.authorId, event.timestamp, event.timestamp]);
    }
    
    async on(event: ArticleLikedEvent): Promise<void> {
        // Get author from article
        const article = await this.db.queryOne(
            'SELECT author_id FROM article_detail_views WHERE article_id = ?',
            [event.articleId]
        );
        
        // Update stats
        await this.db.execute(`
            UPDATE author_stats_views 
            SET total_likes = total_likes + 1,
                average_likes_per_article = total_likes::float / total_articles
            WHERE author_id = ?
        `, [article.author_id]);
    }
}

// ========== QUERY SERVICES ==========

class ArticleQueryService {
    async listArticles(filters: ArticleFilters): Promise<ArticleListView[]> {
        // Query optimizado (read model 1)
        return await this.db.query(`
            SELECT * FROM article_list_views
            WHERE tags && ?  -- PostgreSQL array contains
            ORDER BY published_at DESC
            LIMIT ? OFFSET ?
        `, [filters.tags, filters.pageSize, filters.offset]);
    }
    
    async getArticle(articleId: string): Promise<ArticleDetailView> {
        // Query optimizado (read model 2)
        return await this.db.queryOne(
            'SELECT * FROM article_detail_views WHERE article_id = ?',
            [articleId]
        );
    }
    
    async getAuthorStats(authorId: string): Promise<AuthorStatsView> {
        // Query optimizado (read model 3)
        return await this.db.queryOne(
            'SELECT * FROM author_stats_views WHERE author_id = ?',
            [authorId]
        );
    }
}
```

**Tests**:

```typescript
describe('Blog CQRS', () => {
    it('should update all 3 read models when article published', async () => {
        const command = {
            authorId: 'author-1',
            title: 'My First Article',
            content: 'Lorem ipsum...'.repeat(50),
            tags: ['tech', 'programming']
        };
        
        const articleId = await commandHandler.handle(command);
        
        // Wait for projections (eventual consistency)
        await waitFor(() => {
            // Read Model 1
            const list = await queryService.listArticles({});
            expect(list).toHaveLength(1);
            expect(list[0].excerpt).toHaveLength(200);
            
            // Read Model 2
            const detail = await queryService.getArticle(articleId);
            expect(detail.content).toContain('Lorem ipsum');
            
            // Read Model 3
            const stats = await queryService.getAuthorStats('author-1');
            expect(stats.totalArticles).toBe(1);
        });
    });
    
    it('should update like counts in list and detail views', async () => {
        // Publish article
        const articleId = await publishArticle();
        
        // Like article
        await commandHandler.handle({
            articleId,
            userId: 'user-1'
        });
        
        await waitFor(() => {
            // Both views updated
            const list = await queryService.listArticles({});
            expect(list[0].likeCount).toBe(1);
            
            const detail = await queryService.getArticle(articleId);
            expect(detail.likeCount).toBe(1);
            
            const stats = await queryService.getAuthorStats('author-1');
            expect(stats.totalLikes).toBe(1);
        });
    });
});
```

</details>

---

## ⭐⭐⭐ Ejercicio 3: CQRS con Diferentes Bases de Datos

Implementa CQRS donde:
- **Write DB**: PostgreSQL (eventos)
- **Read DB 1**: MongoDB (vistas denormalizadas)
- **Read DB 2**: Elasticsearch (búsqueda full-text)

**Escenario**: Sistema de e-commerce con productos

<details>
<summary>💡 Solución</summary>

```python
# ========== WRITE SIDE (PostgreSQL) ==========

from dataclasses import dataclass
from datetime import datetime
import psycopg2

@dataclass
class ProductCreatedEvent:
    product_id: str
    name: str
    description: str
    price: float
    category: str
    version: int
    timestamp: datetime

class PostgresEventStore:
    def __init__(self, connection_string: str):
        self.conn = psycopg2.connect(connection_string)
    
    async def save(self, aggregate_id: str, event: ProductCreatedEvent):
        cursor = self.conn.cursor()
        cursor.execute(
            """
            INSERT INTO events (aggregate_id, event_type, event_data, version, timestamp)
            VALUES (%s, %s, %s, %s, %s)
            """,
            (
                aggregate_id,
                event.__class__.__name__,
                json.dumps(asdict(event)),
                event.version,
                event.timestamp
            )
        )
        self.conn.commit()

# ========== READ SIDE 1 (MongoDB) ==========

from motor.motor_asyncio import AsyncIOMotorClient
from pymongo import IndexModel, ASCENDING, TEXT

class MongoDBProductProjection:
    def __init__(self, mongo_url: str):
        self.client = AsyncIOMotorClient(mongo_url)
        self.db = self.client.ecommerce
        self.products = self.db.product_views
        
        # Create indices
        self.products.create_indexes([
            IndexModel([('category', ASCENDING), ('price', ASCENDING)]),
            IndexModel([('name', TEXT), ('description', TEXT)])
        ])
    
    async def on_product_created(self, event: ProductCreatedEvent):
        # Denormalized document
        await self.products.insert_one({
            '_id': event.product_id,
            'name': event.name,
            'description': event.description,
            'price': event.price,
            'category': event.category,
            'stock_status': 'IN_STOCK',  # Default
            'rating': 0.0,
            'review_count': 0,
            'created_at': event.timestamp,
            'updated_at': event.timestamp
        })
    
    async def on_stock_updated(self, event: StockUpdatedEvent):
        stock_status = 'IN_STOCK' if event.new_stock > 0 else 'OUT_OF_STOCK'
        
        await self.products.update_one(
            {'_id': event.product_id},
            {
                '$set': {
                    'stock_status': stock_status,
                    'updated_at': event.timestamp
                }
            }
        )

class MongoDBProductQueryService:
    def __init__(self, mongo_url: str):
        self.client = AsyncIOMotorClient(mongo_url)
        self.products = self.client.ecommerce.product_views
    
    async def get_product(self, product_id: str):
        return await self.products.find_one({'_id': product_id})
    
    async def get_products_by_category(
        self, 
        category: str,
        min_price: float = 0,
        max_price: float = float('inf'),
        page: int = 1,
        page_size: int = 20
    ):
        # Optimized query with indices
        cursor = self.products.find({
            'category': category,
            'price': {'$gte': min_price, '$lte': max_price}
        }).skip((page - 1) * page_size).limit(page_size)
        
        return await cursor.to_list(length=page_size)

# ========== READ SIDE 2 (Elasticsearch) ==========

from elasticsearch import AsyncElasticsearch

class ElasticsearchProductProjection:
    def __init__(self, es_url: str):
        self.es = AsyncElasticsearch([es_url])
        self._create_index()
    
    def _create_index(self):
        # Create index with mapping
        self.es.indices.create(
            index='products',
            body={
                'mappings': {
                    'properties': {
                        'name': {'type': 'text', 'analyzer': 'standard'},
                        'description': {'type': 'text', 'analyzer': 'standard'},
                        'category': {'type': 'keyword'},
                        'price': {'type': 'float'},
                        'tags': {'type': 'keyword'},
                        'created_at': {'type': 'date'}
                    }
                }
            },
            ignore=400  # Ignore if already exists
        )
    
    async def on_product_created(self, event: ProductCreatedEvent):
        await self.es.index(
            index='products',
            id=event.product_id,
            body={
                'name': event.name,
                'description': event.description,
                'category': event.category,
                'price': event.price,
                'created_at': event.timestamp
            }
        )
    
    async def on_product_updated(self, event: ProductUpdatedEvent):
        await self.es.update(
            index='products',
            id=event.product_id,
            body={
                'doc': {
                    'name': event.name,
                    'description': event.description,
                    'price': event.price
                }
            }
        )

class ElasticsearchProductQueryService:
    def __init__(self, es_url: str):
        self.es = AsyncElasticsearch([es_url])
    
    async def search_products(
        self,
        query: str,
        category: str = None,
        min_price: float = None,
        max_price: float = None,
        page: int = 1,
        page_size: int = 20
    ):
        # Build query
        must = [
            {
                'multi_match': {
                    'query': query,
                    'fields': ['name^2', 'description'],  # Boost name
                    'fuzziness': 'AUTO'
                }
            }
        ]
        
        filters = []
        if category:
            filters.append({'term': {'category': category}})
        
        if min_price is not None or max_price is not None:
            range_filter = {'range': {'price': {}}}
            if min_price is not None:
                range_filter['range']['price']['gte'] = min_price
            if max_price is not None:
                range_filter['range']['price']['lte'] = max_price
            filters.append(range_filter)
        
        # Execute search
        result = await self.es.search(
            index='products',
            body={
                'query': {
                    'bool': {
                        'must': must,
                        'filter': filters
                    }
                },
                'from': (page - 1) * page_size,
                'size': page_size,
                'sort': [
                    {'_score': 'desc'},
                    {'created_at': 'desc'}
                ]
            }
        )
        
        return {
            'hits': [hit['_source'] for hit in result['hits']['hits']],
            'total': result['hits']['total']['value']
        }

# ========== EVENT BUS ==========

class EventBus:
    def __init__(
        self,
        mongo_projection: MongoDBProductProjection,
        es_projection: ElasticsearchProductProjection
    ):
        self.projections = [mongo_projection, es_projection]
    
    async def publish(self, event):
        # Publish to all projections in parallel
        tasks = []
        
        if isinstance(event, ProductCreatedEvent):
            tasks = [p.on_product_created(event) for p in self.projections]
        elif isinstance(event, StockUpdatedEvent):
            tasks = [p.on_stock_updated(event) for p in self.projections]
        
        await asyncio.gather(*tasks)
```

**Usage**:

```python
# Write
command_handler = CreateProductCommandHandler(
    event_store=PostgresEventStore('postgresql://...'),
    event_bus=event_bus
)

product_id = await command_handler.handle(CreateProductCommand(
    name='Laptop Pro',
    description='High-performance laptop with...',
    price=1299.99,
    category='electronics'
))

# Read from MongoDB (structured queries)
mongo_service = MongoDBProductQueryService('mongodb://...')
products = await mongo_service.get_products_by_category(
    category='electronics',
    min_price=1000,
    max_price=2000
)

# Read from Elasticsearch (full-text search)
es_service = ElasticsearchProductQueryService('http://localhost:9200')
results = await es_service.search_products(
    query='laptop high performance',
    category='electronics'
)
```

**Benefits**:
- ✅ PostgreSQL: ACID guarantees for writes
- ✅ MongoDB: Fast structured queries with indices
- ✅ Elasticsearch: Powerful full-text search with relevance scoring
- ✅ Each DB optimized for its purpose

</details>

---

## ⭐⭐⭐⭐ Ejercicio 4: CQRS con Eventual Consistency Handling

Implementa CQRS con manejo explícito de eventual consistency.

**Escenario**: Usuario crea orden, inmediatamente consulta su lista de órdenes. La proyección aún no se completó.

**Requisitos**:
1. Command handler retorna correlation ID
2. Query puede verificar si proyección está completa
3. Frontend puede polling o usar WebSocket para actualizaciones

<details>
<summary>💡 Solución</summary>

```typescript
// ========== CORRELATION TRACKING ==========

interface EventMetadata {
    correlationId: string;
    causationId: string;
    timestamp: Date;
}

class ProjectionTracker {
    private processedEvents: Map<string, Date> = new Map();
    
    markProcessed(correlationId: string): void {
        this.processedEvents.set(correlationId, new Date());
    }
    
    isProcessed(correlationId: string): boolean {
        return this.processedEvents.has(correlationId);
    }
    
    async waitForProjection(
        correlationId: string,
        timeoutMs: number = 5000
    ): Promise<boolean> {
        const startTime = Date.now();
        
        while (Date.now() - startTime < timeoutMs) {
            if (this.isProcessed(correlationId)) {
                return true;
            }
            await sleep(100);  // Poll every 100ms
        }
        
        return false;  // Timeout
    }
}

// ========== COMMAND SIDE ==========

class CreateOrderCommandHandler {
    async handle(command: CreateOrderCommand): Promise<CommandResult> {
        const correlationId = generateId();
        
        const event: OrderCreatedEvent = {
            ...command,
            orderId: generateId(),
            version: 1,
            timestamp: new Date(),
            metadata: {
                correlationId,
                causationId: command.id,
                timestamp: new Date()
            }
        };
        
        await this.eventStore.save(event.orderId, event);
        await this.eventBus.publish(event);
        
        return {
            orderId: event.orderId,
            correlationId  // Return for tracking
        };
    }
}

// ========== PROJECTION SIDE ==========

class OrderViewProjection {
    constructor(
        private readDb: Database,
        private tracker: ProjectionTracker,
        private websocket: WebSocketServer
    ) {}
    
    async on(event: OrderCreatedEvent): Promise<void> {
        // Build view
        await this.readDb.insert('order_views', {
            order_id: event.orderId,
            customer_id: event.customerId,
            // ... other fields
        });
        
        // Mark as processed
        this.tracker.markProcessed(event.metadata.correlationId);
        
        // Notify via WebSocket
        this.websocket.broadcast({
            type: 'OrderViewUpdated',
            correlationId: event.metadata.correlationId,
            orderId: event.orderId
        });
    }
}

// ========== QUERY SIDE WITH CONSISTENCY CHECK ==========

class OrderQueryService {
    async getOrder(
        orderId: string,
        correlationId?: string
    ): Promise<OrderView | null> {
        // If correlation ID provided, check if projection complete
        if (correlationId) {
            const isReady = await this.tracker.isProcessed(correlationId);
            
            if (!isReady) {
                // Projection not ready yet
                return null;
            }
        }
        
        return await this.readDb.queryOne(
            'SELECT * FROM order_views WHERE order_id = ?',
            [orderId]
        );
    }
    
    async getOrderWithWait(
        orderId: string,
        correlationId: string,
        timeoutMs: number = 5000
    ): Promise<OrderView> {
        // Wait for projection
        const ready = await this.tracker.waitForProjection(
            correlationId,
            timeoutMs
        );
        
        if (!ready) {
            throw new Error('Projection timeout - data not yet available');
        }
        
        return await this.getOrder(orderId, correlationId);
    }
}

// ========== API ENDPOINTS ==========

// REST API
app.post('/api/orders', async (req, res) => {
    const result = await commandHandler.handle(req.body);
    
    // Return 202 Accepted (async processing)
    res.status(202).json({
        orderId: result.orderId,
        correlationId: result.correlationId,
        status: 'PROCESSING',
        links: {
            self: `/api/orders/${result.orderId}`,
            status: `/api/orders/${result.orderId}/status?correlation=${result.correlationId}`
        }
    });
});

app.get('/api/orders/:orderId', async (req, res) => {
    const { orderId } = req.params;
    const { correlation, wait } = req.query;
    
    if (wait === 'true' && correlation) {
        // Wait for projection
        try {
            const order = await queryService.getOrderWithWait(
                orderId,
                correlation,
                5000
            );
            res.json(order);
        } catch (error) {
            res.status(408).json({
                error: 'Timeout',
                message: 'Order is being processed, please try again',
                retryAfter: 1  // seconds
            });
        }
    } else {
        // Immediate response
        const order = await queryService.getOrder(orderId, correlation);
        
        if (!order && correlation) {
            // Not ready yet
            res.status(202).json({
                status: 'PROCESSING',
                message: 'Order is being processed',
                retryAfter: 1
            });
        } else if (order) {
            res.json(order);
        } else {
            res.status(404).json({ error: 'Order not found' });
        }
    }
});

// ========== FRONTEND STRATEGIES ==========

// Strategy 1. Polling
async function createOrderWithPolling(orderData: CreateOrderDto) {
    // Create order
    const response = await fetch('/api/orders', {
        method: 'POST',
        body: JSON.stringify(orderData)
    });
    
    const { orderId, correlationId } = await response.json();
    
    // Poll until ready
    for (let attempt = 0; attempt < 10; attempt++) {
        await sleep(500);
        
        const orderResponse = await fetch(
            `/api/orders/${orderId}?correlation=${correlationId}`
        );
        
        if (orderResponse.status === 200) {
            return await orderResponse.json();  // Ready!
        }
    }
    
    throw new Error('Timeout waiting for order');
}

// Strategy 2: WebSocket
class OrderClient {
    private ws: WebSocket;
    private pendingOrders: Map<string, (order: OrderView) => void> = new Map();
    
    constructor() {
        this.ws = new WebSocket('ws://localhost:3000');
        this.ws.onmessage = this.handleMessage.bind(this);
    }
    
    async createOrder(orderData: CreateOrderDto): Promise<OrderView> {
        const response = await fetch('/api/orders', {
            method: 'POST',
            body: JSON.stringify(orderData)
        });
        
        const { orderId, correlationId } = await response.json();
        
        // Wait for WebSocket notification
        return new Promise((resolve, reject) => {
            this.pendingOrders.set(correlationId, resolve);
            
            setTimeout(() => {
                this.pendingOrders.delete(correlationId);
                reject(new Error('Timeout'));
            }, 5000);
        });
    }
    
    private handleMessage(event: MessageEvent): void {
        const message = JSON.parse(event.data);
        
        if (message.type === 'OrderViewUpdated') {
            const resolver = this.pendingOrders.get(message.correlationId);
            if (resolver) {
                // Fetch complete order
                fetch(`/api/orders/${message.orderId}`)
                    .then(r => r.json())
                    .then(resolver);
                
                this.pendingOrders.delete(message.correlationId);
            }
        }
    }
}

// Usage
const client = new OrderClient();
const order = await client.createOrder({
    customerId: 'cust-123',
    items: [...]
});
// Automatically waits for projection via WebSocket

// Strategy 3: Server-Sent Events (SSE)
app.get('/api/orders/:orderId/stream', async (req, res) => {
    const { orderId } = req.params;
    const { correlation } = req.query;
    
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    
    // Check periodically
    const interval = setInterval(async () => {
        const order = await queryService.getOrder(orderId, correlation);
        
        if (order) {
            res.write(`data: ${JSON.stringify(order)}\n\n`);
            clearInterval(interval);
            res.end();
        }
    }, 500);
    
    // Timeout after 10s
    setTimeout(() => {
        clearInterval(interval);
        res.write('event: timeout\ndata: {}\n\n');
        res.end();
    }, 10000);
});
```

**Tests**:

```typescript
describe('Eventual Consistency Handling', () => {
    it('should return 202 when projection not ready', async () => {
        const { orderId, correlationId } = await commandHandler.handle(command);
        
        // Immediate query (projection not ready)
        const response = await request(app)
            .get(`/api/orders/${orderId}?correlation=${correlationId}`);
        
        expect(response.status).toBe(202);
        expect(response.body.status).toBe('PROCESSING');
    });
    
    it('should wait for projection with wait=true', async () => {
        const { orderId, correlationId } = await commandHandler.handle(command);
        
        // Query with wait
        const response = await request(app)
            .get(`/api/orders/${orderId}?correlation=${correlationId}&wait=true`);
        
        expect(response.status).toBe(200);
        expect(response.body.orderId).toBe(orderId);
    });
    
    it('should notify via WebSocket when projection complete', async (done) => {
        const ws = new WebSocket('ws://localhost:3000');
        
        ws.on('message', (data) => {
            const message = JSON.parse(data);
            expect(message.type).toBe('OrderViewUpdated');
            done();
        });
        
        await commandHandler.handle(command);
    });
});
```

</details>

---

## Recursos

- **CQRS**: https://martinfowler.com/bliki/CQRS.html
- **Projection Patterns**: https://www.eventstore.com/blog/projections-1-theory
- **Eventual Consistency**: https://www.allthingsdistributed.com/2008/12/eventually_consistent.html
