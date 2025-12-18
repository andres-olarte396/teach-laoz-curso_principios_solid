# Ejercicios: Pre/Postcondiciones

## ⭐ Nivel 1
¿Cuál viola LSP?
```java
class Base { void method(int x) { /* x >= 0 */ } }
class SubA extends Base { void method(int x) { /* x >= 10 */ } }
class SubB extends Base { void method(int x) { /* x >= 0 */ } }
```

## ⭐⭐ Nivel 2
Refactoriza para cumplir LSP. Tests que demuestren sustitución.

## ⭐⭐⭐ Nivel 3
Sistema de pagos con validaciones. Cumplir LSP en toda la jerarquía.

## ⭐⭐⭐⭐ Nivel 4
Contract testing framework que valida pre/postcondiciones automáticamente con anotaciones.
