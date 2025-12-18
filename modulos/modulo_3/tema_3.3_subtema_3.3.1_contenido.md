# Arquitecturas Plugin para Máxima Extensibilidad

## Concepto

Una **arquitectura plugin** permite extender una aplicación sin modificar su código core, cargando módulos dinámicamente.

## Elementos

1. **Core**: Aplicación base inmutable
2. **Plugin Interface**: Contrato que deben cumplir los plugins
3. **Plugin Loader**: Descubre y carga plugins
4. **Plugins**: Módulos independientes

## Ejemplo: Sistema de Procesamiento de Archivos

```java
// 1. Plugin Interface
public interface FileProcessor {
    String getName();
    boolean canProcess(String filename);
    void process(File file);
}

// 2. Core Application
public class FileProcessorApp {
    private List<FileProcessor> processors = new ArrayList<>();
    
    public void registerProcessor(FileProcessor processor) {
        processors.add(processor);
        System.out.println("Registered: " + processor.getName());
    }
    
    public void processFile(File file) {
        for (FileProcessor processor : processors) {
            if (processor.canProcess(file.getName())) {
                processor.process(file);
                return;
            }
        }
        System.out.println("No processor found for: " + file.getName());
    }
}

// 3. Plugins Concretos
public class PdfProcessor implements FileProcessor {
    public String getName() {
        return "PDF Processor";
    }
    
    public boolean canProcess(String filename) {
        return filename.endsWith(".pdf");
    }
    
    public void process(File file) {
        System.out.println("Processing PDF: " + file.getName());
        // Lógica específica de PDF
    }
}

public class ImageProcessor implements FileProcessor {
    public String getName() {
        return "Image Processor";
    }
    
    public boolean canProcess(String filename) {
        return filename.matches(".*\\.(jpg|png|gif)$");
    }
    
    public void process(File file) {
        System.out.println("Processing Image: " + file.getName());
        // Lógica específica de imágenes
    }
}

// 4. Plugin Discovery (manual)
FileProcessorApp app = new FileProcessorApp();
app.registerProcessor(new PdfProcessor());
app.registerProcessor(new ImageProcessor());

app.processFile(new File("document.pdf")); // Output: Processing PDF: document.pdf
app.processFile(new File("photo.jpg"));    // Output: Processing Image: photo.jpg
```

## Plugin Discovery Automático (ServiceLoader)

```java
// META-INF/services/com.example.FileProcessor
com.example.plugins.PdfProcessor
com.example.plugins.ImageProcessor

// Código
public class PluginLoader {
    public static List<FileProcessor> loadPlugins() {
        ServiceLoader<FileProcessor> loader = ServiceLoader.load(FileProcessor.class);
        List<FileProcessor> plugins = new ArrayList<>();
        for (FileProcessor processor : loader) {
            plugins.add(processor);
        }
        return plugins;
    }
}

// Uso
FileProcessorApp app = new FileProcessorApp();
List<FileProcessor> plugins = PluginLoader.loadPlugins();
plugins.forEach(app::registerProcessor);
```

**OCP**: Añadir `VideoProcessor` solo requiere crear la clase plugin, sin modificar `FileProcessorApp`.

## Caso de Estudio: Sistema de Reportes

```java
// Plugin Interface
public interface ReportPlugin {
    String getFormat();
    void generate(Report report, OutputStream output);
}

// Core
public class ReportGenerator {
    private Map<String, ReportPlugin> plugins = new HashMap<>();
    
    public void registerPlugin(ReportPlugin plugin) {
        plugins.put(plugin.getFormat(), plugin);
    }
    
    public void generateReport(Report report, String format, OutputStream output) {
        ReportPlugin plugin = plugins.get(format);
        if (plugin == null) {
            throw new IllegalArgumentException("Unknown format: " + format);
        }
        plugin.generate(report, output);
    }
}

// Plugins
public class PdfReportPlugin implements ReportPlugin {
    public String getFormat() { return "PDF"; }
    
    public void generate(Report report, OutputStream output) {
        // Generar PDF
    }
}

public class ExcelReportPlugin implements ReportPlugin {
    public String getFormat() { return "EXCEL"; }
    
    public void generate(Report report, OutputStream output) {
        // Generar Excel
    }
}

// Uso
ReportGenerator generator = new ReportGenerator();
generator.registerPlugin(new PdfReportPlugin());
generator.registerPlugin(new ExcelReportPlugin());

generator.generateReport(salesReport, "PDF", outputStream);
```

## Plugins con Configuración

```java
public interface ConfigurablePlugin extends FileProcessor {
    void configure(Properties config);
}

public class AdvancedPdfProcessor implements ConfigurablePlugin {
    private boolean enableOcr;
    private int compressionLevel;
    
    public void configure(Properties config) {
        enableOcr = Boolean.parseBoolean(config.getProperty("ocr.enabled", "false"));
        compressionLevel = Integer.parseInt(config.getProperty("compression.level", "5"));
    }
    
    public void process(File file) {
        // Usar enableOcr y compressionLevel
    }
}

// Configuración (config.properties)
ocr.enabled=true
compression.level=9

// Carga
Properties config = new Properties();
config.load(new FileInputStream("config.properties"));

AdvancedPdfProcessor plugin = new AdvancedPdfProcessor();
plugin.configure(config);
```

## Plugins con Dependencias

```java
public interface PluginContext {
    <T> T getService(Class<T> serviceClass);
}

public interface DatabasePlugin {
    void initialize(PluginContext context);
    void execute();
}

public class PostgresPlugin implements DatabasePlugin {
    private DataSource dataSource;
    private Logger logger;
    
    public void initialize(PluginContext context) {
        dataSource = context.getService(DataSource.class);
        logger = context.getService(Logger.class);
    }
    
    public void execute() {
        logger.info("Connecting to PostgreSQL");
        Connection conn = dataSource.getConnection();
        // ...
    }
}
```

## Versionado de Plugins

```java
public interface VersionedPlugin {
    String getName();
    Version getVersion();
    Version getMinCoreVersion();
}

public class ImageProcessorV2 implements FileProcessor, VersionedPlugin {
    public Version getVersion() {
        return new Version(2, 0, 0);
    }
    
    public Version getMinCoreVersion() {
        return new Version(1, 5, 0); // Requiere core >= 1.5.0
    }
}

// Validación al cargar
public void registerPlugin(VersionedPlugin plugin) {
    if (coreVersion.isCompatible(plugin.getMinCoreVersion())) {
        plugins.add(plugin);
    } else {
        System.err.println("Plugin " + plugin.getName() + 
            " requires core version " + plugin.getMinCoreVersion());
    }
}
```

## Ejemplos Reales de Arquitecturas Plugin

### Eclipse IDE
```java
// Extensión de menús
public class MyMenuContribution implements IMenuContribution {
    public void createMenu(IMenuManager manager) {
        manager.add(new Action("My Action") {
            public void run() { /* ... */ }
        });
    }
}
```

### Jenkins CI/CD
```java
public class MyBuilder extends Builder {
    @Override
    public boolean perform(AbstractBuild build, Launcher launcher, BuildListener listener) {
        // Lógica de build personalizada
        return true;
    }
}
```

### IntelliJ IDEA
```xml
<!-- plugin.xml -->
<idea-plugin>
  <extensions defaultExtensionNs="com.intellij">
    <fileType name="MyType" implementationClass="com.example.MyFileType"/>
  </extensions>
</idea-plugin>
```

## Resumen

**Arquitectura Plugin** = Core inmutable + plugins dinámicos que extienden funcionalidad.

**OCP**: Máxima apertura a extensión (plugins ilimitados) con core cerrado a modificación.

**Cuándo usar**: Aplicaciones que requieren extensibilidad ilimitada (IDEs, navegadores, sistemas empresariales).
