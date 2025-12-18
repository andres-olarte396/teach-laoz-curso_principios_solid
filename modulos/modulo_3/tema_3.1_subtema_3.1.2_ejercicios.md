# Ejercicios: Strategy Pattern

## ⭐ Nivel 1
Identifica qué parte del código es la estrategia, contexto y estrategias concretas:
```java
interface Sorter { void sort(int[] arr); }
class QuickSort implements Sorter { /* ... */ }
class MergeSort implements Sorter { /* ... */ }
class ArraySorter { 
    private Sorter sorter;
    void sortArray(int[] arr) { sorter.sort(arr); }
}
```

## ⭐⭐ Nivel 2
Implementa sistema de envío con múltiples transportistas (Standard, Express, SameDay) usando Strategy.

## ⭐⭐⭐ Nivel 3
Sistema de autenticación con múltiples estrategias (Password, OAuth, JWT, Biometric). Incluye factory y tests.

## ⭐⭐⭐⭐ Nivel 4
Sistema de pricing dinámico para e-commerce (peak hours, bulk discounts, loyalty points, seasonal). Permite combinar estrategias.
