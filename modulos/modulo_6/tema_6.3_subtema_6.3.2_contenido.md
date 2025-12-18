# Configuración de Contenedores

## Java Config (Spring)

```java
@Configuration
class AppConfig {
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://localhost/db");
        ds.setUsername("user");
        ds.setPassword("pass");
        return ds;
    }
    
    @Bean
    public OrderRepository orderRepository(DataSource dataSource) {
        return new MySQLOrderRepository(dataSource);
    }
    
    @Bean
    @Primary // Bean por defecto cuando hay múltiples
    public EmailService emailService() {
        return new SMTPEmailService();
    }
    
    @Bean
    @Profile("dev") // Solo en perfil dev
    public EmailService mockEmailService() {
        return new MockEmailService();
    }
}
```

## Profiles

```java
@Configuration
@Profile("production")
class ProductionConfig {
    @Bean
    public DataSource dataSource() {
        // Configuración de producción
    }
}

@Configuration
@Profile("test")
class TestConfig {
    @Bean
    public DataSource dataSource() {
        // H2 in-memory para tests
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}

// Activar perfil
// application.properties: spring.profiles.active=production
```

## Conditional Beans

```java
@Configuration
class CacheConfig {
    @Bean
    @ConditionalOnProperty(name = "cache.enabled", havingValue = "true")
    public CacheManager cacheManager() {
        return new CaffeineCacheManager();
    }
    
    @Bean
    @ConditionalOnMissingBean(CacheManager.class)
    public CacheManager noCacheManager() {
        return new NoOpCacheManager();
    }
}
```

## Resumen

**Configuración IoC** = Java config, profiles, conditional beans para flexibilidad.
