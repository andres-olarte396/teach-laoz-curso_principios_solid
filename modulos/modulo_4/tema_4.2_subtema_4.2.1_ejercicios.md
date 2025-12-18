# Ejercicios: Herencia vs Composición

## ⭐ Nivel 1
¿Cuál cumple LSP?
```java
class Vehicle { void start() {} }
class Car extends Vehicle {}
class Bicycle extends Vehicle { void start() { throw...; } }
```

## ⭐⭐ Nivel 2
Refactoriza jerarquía Bird/Penguin/Eagle usando composición e interfaces.

## ⭐⭐⭐ Nivel 3
Sistema de vehículos (Car, Truck, Bicycle, Boat). Usar composición para características (Engine, Wheels, Sails). Tests LSP.

## ⭐⭐⭐⭐ Nivel 4
Refactoriza jerarquía de herencia profunda (5+ niveles) a composición. Comparar métricas antes/después.
