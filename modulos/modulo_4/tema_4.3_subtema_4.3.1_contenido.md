# Testing para Validar LSP

## Property-Based Testing

Verificar que propiedades se mantienen en toda la jerarquía:

```java
// JUnit + QuickCheck/jqwik
@Property
void testAccountWithdrawContract(@ForAll("accounts") Account account) {
    double initialBalance = account.getBalance();
    double amount = 50;
    
    assumeTrue(initialBalance >= amount); // Precondición
    
    account.withdraw(amount);
    
    // Postcondición: debe cumplirse para TODOS los subtipos
    assertEquals(initialBalance - amount, account.getBalance(), 0.01);
}

@Provide
Arbitrary<Account> accounts() {
    return Arbitraries.of(
        new SavingsAccount(),
        new CheckingAccount(),
        new PremiumAccount()
    );
}
```

## Test de Sustitución

```java
@Test
void testListSubstitution() {
    testListBehavior(new ArrayList<>());
    testListBehavior(new LinkedList<>());
    testListBehavior(new Vector<>());
}

void testListBehavior(List<Integer> list) {
    list.add(1);
    list.add(2);
    assertEquals(2, list.size());
    assertEquals(Integer.valueOf(1), list.get(0));
    list.remove(0);
    assertEquals(1, list.size());
}
```

## Mutation Testing

Verificar que tests detectan violaciones LSP:

```java
// Original (cumple LSP)
class Circle implements Shape {
    double getArea() { return PI * r * r; }
}

// Mutante (viola LSP)
class Circle implements Shape {
    double getArea() { return -1; } // ❌ Viola postcondición (area > 0)
}

// Test debe detectar violación
@Test
void testShapeAreaPositive() {
    Shape shape = new Circle(5);
    assertTrue(shape.getArea() > 0); // ✅ Detecta mutante
}
```

## Resumen

**Testing LSP**:
- Property-based testing para contratos
- Tests parametrizados con toda la jerarquía
- Mutation testing para validar robustez
