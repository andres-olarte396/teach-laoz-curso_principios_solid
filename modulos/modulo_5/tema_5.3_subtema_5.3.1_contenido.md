# Detección de Fat Interfaces

## Métricas

### 1. Número de Métodos
**Heurística**: Interfaz con >10 métodos probablemente viola ISP.

### 2. Cohesión de Interfaz
Métodos que no se usan juntos → baja cohesión.

### 3. Implementaciones con UnsupportedOperationException
Señal clara de violación ISP.

## Herramientas

```bash
# SonarQube: regla S1214
# PMD: ExcessivePublicCount
# IntelliJ IDEA: Inspecciones de interfaz
```

## Refactoring

1. Identificar grupos de métodos relacionados
2. Extraer interfaces cohesivas
3. Actualizar implementaciones
4. Actualizar clientes

## Resumen

**Detección ISP**: Contar métodos, analizar cohesión, buscar métodos no implementados.
