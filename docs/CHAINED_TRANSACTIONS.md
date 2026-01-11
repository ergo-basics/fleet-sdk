# Chained Transactions en Fleet SDK

Fleet SDK ofrece soporte nativo para **transacciones encadenadas** (Chained Transactions). Esta funcionalidad permite construir una secuencia de transacciones donde los outputs de una transacción sirven como inputs para la siguiente, todo ello antes de que cualquiera de ellas sea firmada o enviada a la red.

Esto es especialmente útil para flujos complejos que requieren múltiples pasos atómicos o cuando se desea utilizar inmediatamente los fondos resultantes de una operación (el "cambio") en una operación subsiguiente.

## Cómo funciona

El encadenamiento se gestiona a través del método `.chain()` disponible en una instancia de `ErgoUnsignedTransaction` (el resultado de `.build()`).

Cuando invocas `.chain(callback)`, Fleet SDK realiza lo siguiente automáticamente:

1.  **Pre-selección de Inputs**: Crea un nuevo `TransactionBuilder` cuyos inputs son **exclusivamente los outputs de cambio** (change boxes) de la transacción padre.
2.  **Herencia de Configuración**: El nuevo builder hereda la `changeAddress` y la configuración de `fee` de la transacción padre, simplificando la configuración.
3.  **Callback de Construcción**: Ejecuta tu función callback proporcionando este nuevo `builder` y la transacción `parent` completa.

### Firma del Método

```typescript
transaction.chain((builder, parent) => {
  // builder: TransactionBuilder pre-configurado con los change outputs del padre
  // parent: La ErgoUnsignedTransaction padre completa
  
  return builder.to(...).build();
});
```

---

## Ejemplos

### 1. Encadenamiento Básico (Gastando el Cambio)

Este es el caso más común: realizas un pago y quieres usar inmediatamente el cambio restante para otra operación. No es necesario seleccionar inputs manualmente; Fleet ya ha seleccionado el cambio por ti.

```typescript
import { TransactionBuilder, OutputBuilder } from "@fleet-sdk/core";

// 1. Primera Transacción (TX1)
const tx1 = new TransactionBuilder(height)
  .from(inputs)
  .to(new OutputBuilder(1_000_000_000n, recipientA)) // Enviar 1 ERG a A
  .sendChangeTo(myAddress)
  .payMinFee()
  .build();

// 2. Segunda Transacción (TX2)
// El 'builder' ya tiene como inputs los boxes de cambio de TX1
const chain = tx1.chain((builder) => {
  return builder
    .to(new OutputBuilder(500_000_000n, recipientB)) // Enviar 0.5 ERG a B (desde el cambio de TX1)
    .build();
});

// Obtener todas las transacciones para firmar
const transactions = chain.toEIP12Object();
// transactions[0] -> TX1
// transactions[1] -> TX2
```

### 2. Encadenamiento Avanzado (Gastando Outputs Específicos)

A veces, la segunda transacción no debe gastar el cambio, sino un output específico generado por la primera (por ejemplo, un token recién acuñado o una caja de contrato específica).

En este caso, debes limpiar los inputs automáticos y seleccionar explícitamente los que necesitas del objeto `parent`.

```typescript
const tx1 = new TransactionBuilder(height)
  .from(inputs)
  .to(
    new OutputBuilder(1_000_000n, contractAddress) // Output 0: Caja de contrato
      .setAdditionalRegisters({ R4: "..." })
  )
  .sendChangeTo(myAddress)
  .payMinFee()
  .build();

const chain = tx1.chain((builder, parent) => {
  // 1. Limpiamos los inputs automáticos (que son el cambio)
  // 2. Seleccionamos explícitamente el primer output de la TX padre
  return builder
    .from(parent.outputs[0]) 
    .to(new OutputBuilder(1_000_000n, anotherAddress))
    .build();
});
```

### 3. Encadenamiento Múltiple (Más de 2 Transacciones)

Puedes encadenar tantas transacciones como necesites. El método `.chain()` devuelve un objeto que mantiene la referencia a la cadena.

```typescript
const chain = tx1
  .chain((builder) => {
    // TX2: Usa el cambio de TX1
    return builder.to(outputB).build();
  })
  .chain((builder) => {
    // TX3: Usa el cambio de TX2
    return builder.to(outputC).build();
  });

// El resultado es un array con [TX1, TX2, TX3]
const allTxs = chain.toEIP12Object();
```

## Detalles Técnicos Importantes

*   **Validación de Inputs**: Al construir la cadena, Fleet SDK asume que la transacción padre es válida. Si la transacción padre falla al minarse, toda la cadena fallará.
*   **IDs de Transacción**: Las transacciones hijas hacen referencia a los `boxId` de los outputs de la transacción padre. Estos IDs son deterministas y se calculan antes de firmar, lo que hace posible este encadenamiento *offline*.
*   **Change Address**: Si no especificas un `sendChangeTo` en las transacciones hijas, se usará la misma dirección que en la transacción padre.
