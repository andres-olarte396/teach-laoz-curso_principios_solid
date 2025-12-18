# LSP en Lenguajes Multiparadigma

## Java

### Covarianza de Retorno
```java
class Animal {
    Animal reproduce() { return new Animal(); }
}

class Dog extends Animal {
    @Override
    Dog reproduce() { return new Dog(); } // ✅ Covarianza permitida
}
```

### Generics y LSP
```java
// ❌ Arrays son covariantes (problemático)
Integer[] ints = {1, 2, 3};
Number[] nums = ints; // ✅ Compila
nums[0] = 3.14; // ❌ RuntimeException

// ✅ Generics son invariantes (seguro)
List<Integer> intList = new ArrayList<>();
List<Number> numList = intList; // ❌ No compila - protege LSP
```

## TypeScript

### Structural Typing
```typescript
interface Point {
    x: number;
    y: number;
}

interface Point3D {
    x: number;
    y: number;
    z: number;
}

let point2D: Point = { x: 0, y: 0 };
let point3D: Point3D = { x: 0, y: 0, z: 0 };

point2D = point3D; // ✅ Sustitución estructural
```

## Python

### Duck Typing y LSP
```python
# LSP implícito vía duck typing
def process_file(file_like):
    file_like.read()
    file_like.close()

# ✅ Cualquier objeto con read() y close() funciona
process_file(open('file.txt'))
process_file(io.StringIO('data'))
process_file(gzip.open('file.gz'))
```

## Resumen

Cada lenguaje tiene mecanismos diferentes para LSP:
- **Java**: Herencia nominal, generics invariantes
- **TypeScript**: Structural typing flexible
- **Python**: Duck typing implícito
