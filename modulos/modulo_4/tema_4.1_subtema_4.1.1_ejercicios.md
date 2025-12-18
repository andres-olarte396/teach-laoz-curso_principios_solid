# Ejercicios: Definición de LSP

## ⭐ Nivel 1
Identifica violación de LSP:
```java
class Bird { void fly() { /* volar */ } }
class Penguin extends Bird { 
    void fly() { throw new UnsupportedOperationException(); } 
}
```

## ⭐⭐ Nivel 2
Refactoriza Rectángulo-Cuadrado usando composición. Tests que demuestren sustitución.

## ⭐⭐⭐ Nivel 3
Sistema de cuentas bancarias (Account, SavingsAccount, CheckingAccount). Cumplir LSP con precondiciones/postcondiciones.

## ⭐⭐⭐⭐ Nivel 4
Jerarquía de colecciones (List, Set, Map) con invariantes. Tests exhaustivos de sustitución con property-based testing.
