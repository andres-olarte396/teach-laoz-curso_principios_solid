# Ejercicios: SRP en TypeScript y Python

## ⭐ Nivel 1
Implementa `UserValidator` en TypeScript y Python usando características idiomáticas.

## ⭐⭐ Nivel 2
Refactoriza esta clase que mezcla responsabilidades en ambos lenguajes:
```python
class ProductManager:
    def create_product(self, name, price):
        if price < 0: raise ValueError()
        db.execute("INSERT...")
        send_email("admin@", f"New product: {name}")
```

## ⭐⭐⭐ Nivel 3
Implementa API REST de gestión de tareas usando NestJS (TypeScript) y FastAPI (Python). Demuestra SRP con DI.

## ⭐⭐⭐⭐ Nivel 4
Compara implementaciones del mismo sistema en TypeScript y Python. Analiza pros/contras de cada lenguaje para SRP.
