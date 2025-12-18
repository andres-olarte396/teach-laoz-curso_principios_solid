# Diseño por Contratos (Design by Contract)

## Principios de DbC (Bertrand Meyer)

1. **Precondiciones**: Lo que el cliente debe garantizar
2. **Postcondiciones**: Lo que el método garantiza
3. **Invariantes**: Condiciones siempre verdaderas

## Contratos y LSP

```java
/**
 * @requires balance >= 0
 * @requires amount > 0
 * @ensures balance' = balance - amount
 * @ensures balance' >= 0
 */
class BankAccount {
    private double balance;
    
    public void withdraw(double amount) {
        assert amount > 0 : "Precondition: amount > 0";
        assert balance >= amount : "Precondition: sufficient funds";
        
        balance -= amount;
        
        assert balance >= 0 : "Postcondition: balance >= 0";
    }
}
```

## Reglas de Sustitución

### Regla de Precondiciones
**Subtipos no pueden fortalecer precondiciones**

```java
// Base: amount > 0
class Account {
    void withdraw(double amount) {
        if (amount <= 0) throw new IllegalArgumentException();
    }
}

// ✅ Precondición igual o más débil
class PremiumAccount extends Account {
    void withdraw(double amount) {
        // Acepta amount > 0 (mismo contrato)
        super.withdraw(amount);
    }
}

// ❌ Precondición más fuerte
class RestrictedAccount extends Account {
    void withdraw(double amount) {
        if (amount < 100) throw new IllegalArgumentException(); // ❌ Viola LSP
        super.withdraw(amount);
    }
}
```

### Regla de Postcondiciones
**Subtipos no pueden debilitar postcondiciones**

```java
// Base: retorna resultado ordenado
class Sorter {
    /**
     * @ensures result is sorted ascending
     */
    List<Integer> sort(List<Integer> input) {
        List<Integer> result = new ArrayList<>(input);
        Collections.sort(result);
        return result; // Garantía: ordenado ascendente
    }
}

// ✅ Postcondición igual o más fuerte
class StableSorter extends Sorter {
    /**
     * @ensures result is sorted AND preserves relative order of equal elements
     */
    List<Integer> sort(List<Integer> input) {
        // Más garantías (estable)
        return input.stream().sorted().collect(Collectors.toList());
    }
}

// ❌ Postcondición más débil
class PartialSorter extends Sorter {
    List<Integer> sort(List<Integer> input) {
        // ❌ Solo ordena primeros 10 elementos
        return input.stream().limit(10).sorted().collect(Collectors.toList());
    }
}
```

## Invariantes de Clase

```java
class Stack<T> {
    private List<T> items = new ArrayList<>();
    
    // Invariante: size() >= 0 && size() <= capacity
    private static final int DEFAULT_CAPACITY = 100;
    
    private void checkInvariant() {
        assert items.size() >= 0;
        assert items.size() <= DEFAULT_CAPACITY;
    }
    
    public void push(T item) {
        if (items.size() >= DEFAULT_CAPACITY) {
            throw new IllegalStateException("Stack full");
        }
        items.add(item);
        checkInvariant();
    }
    
    public T pop() {
        if (items.isEmpty()) {
            throw new IllegalStateException("Stack empty");
        }
        T item = items.remove(items.size() - 1);
        checkInvariant();
        return item;
    }
}
```

## Testing de Contratos

```java
@Test
void testWithdrawContract() {
    Account account = new Account();
    account.deposit(100);
    
    // Precondición: amount > 0
    assertThrows(IllegalArgumentException.class, () -> account.withdraw(0));
    assertThrows(IllegalArgumentException.class, () -> account.withdraw(-10));
    
    // Postcondición: balance reducido correctamente
    account.withdraw(30);
    assertEquals(70, account.getBalance());
    
    // Invariante: balance >= 0
    assertThrows(IllegalArgumentException.class, () -> account.withdraw(100));
}

@Test
void testLSPWithContracts() {
    Account base = new Account();
    Account sub = new PremiumAccount();
    
    // Mismo contrato debe funcionar para ambos
    testAccountContract(base);
    testAccountContract(sub);
}

void testAccountContract(Account account) {
    account.deposit(100);
    account.withdraw(50); // amount > 0, debe funcionar
    assertEquals(50, account.getBalance());
}
```

## Resumen

**DbC** = Especificar contratos formales con pre/postcondiciones e invariantes

**LSP + DbC**:
- Precondiciones: pueden debilitarse en subtipos
- Postcondiciones: pueden fortalecerse en subtipos
- Invariantes: deben mantenerse en subtipos
