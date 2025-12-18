# Ejercicios: Extract Class y Extract Method

## ⭐ Nivel 1
Aplica Extract Method en este código:
```java
public void generateReport(List<Sale> sales) {
    double total = 0;
    for (Sale s : sales) total += s.getAmount();
    System.out.println("Total: " + total);
    double avg = total / sales.size();
    System.out.println("Average: " + avg);
}
```

## ⭐⭐ Nivel 2
Aplica Extract Class en esta clase que mezcla usuario y dirección.

## ⭐⭐⭐ Nivel 3
Refactoriza sistema de facturación completo usando todas las técnicas.

## ⭐⭐⭐⭐ Nivel 4
Refactoriza clase legacy de 500+ LOC, documenta cada extracción, mide mejora de métricas.
