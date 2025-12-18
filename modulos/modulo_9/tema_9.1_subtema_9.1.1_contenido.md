# Caso de Estudio: Spring Framework

## Análisis SOLID en Spring

### SRP: Separación Clara

```java
// ApplicationContext: gestión de beans
interface ApplicationContext {
    Object getBean(String name);
}

// BeanFactory: creación de beans
interface BeanFactory {
    Object getBean(String name);
}

// ResourceLoader: carga de recursos
interface ResourceLoader {
    Resource getResource(String location);
}
```

### OCP: Extensibilidad

```java
// ✅ BeanPostProcessor: extender sin modificar
interface BeanPostProcessor {
    Object postProcessBeforeInitialization(Object bean, String name);
    Object postProcessAfterInitialization(Object bean, String name);
}

// Agregar comportamiento custom
@Component
class LoggingBeanPostProcessor implements BeanPostProcessor {
    public Object postProcessAfterInitialization(Object bean, String name) {
        log.info("Bean created: " + name);
        return bean;
    }
}
```

### LSP: Jerarquía de Contexts

```java
// Todos los contexts sustituibles
ApplicationContext context = new AnnotationConfigApplicationContext();
ApplicationContext context2 = new ClassPathXmlApplicationContext();
ApplicationContext context3 = new GenericApplicationContext();

// Mismo contrato
Object bean = context.getBean("myBean");
```

### ISP: Interfaces Segregadas

```java
// Interfaces mínimas por responsabilidad
interface ListableBeanFactory { String[] getBeanDefinitionNames(); }
interface HierarchicalBeanFactory { BeanFactory getParentBeanFactory(); }
interface ConfigurableBeanFactory { void setBeanClassLoader(ClassLoader cl); }

// Implementación completa combina interfaces
class DefaultListableBeanFactory 
    implements ListableBeanFactory, HierarchicalBeanFactory, ConfigurableBeanFactory {
    // Implementa todas las capacidades
}
```

### DIP: Dependency Injection Central

```java
// Core depende de abstracciones
interface ApplicationEventPublisher {
    void publishEvent(ApplicationEvent event);
}

// Implementación en framework
class AbstractApplicationContext implements ApplicationEventPublisher {
    private ApplicationEventMulticaster eventMulticaster;
    
    public void publishEvent(ApplicationEvent event) {
        eventMulticaster.multicastEvent(event);
    }
}

// Usuario depende de abstracción
@Component
class OrderService {
    @Autowired
    private ApplicationEventPublisher publisher;
    
    public void placeOrder(Order order) {
        publisher.publishEvent(new OrderPlacedEvent(order));
    }
}
```

## Lecciones

1. **SRP**: Módulos enfocados (beans, resources, events)
2. **OCP**: Extensión vía interfaces (BeanPostProcessor, ApplicationListener)
3. **LSP**: Jerarquías bien diseñadas (ApplicationContext)
4. **ISP**: Interfaces granulares (ListableBeanFactory, etc.)
5. **DIP**: Todo vía abstracciones (DI container)

## Resumen

**Spring** = Ejemplo real de aplicación completa de SOLID en framework enterprise.
