## Field

### The JavaScript Struggle: Numeric Operations on Custom Types
JavaScript developers frequently work with numbers, using operators like `+`, `-`, `*`, and `/` for calculations. These work seamlessly for built-in numbers:

```javascript
console.log(1 + 2 * 3); // 7
console.log(10 / 2); // 5
```

But for custom types, like a `Money` object representing currency amounts, JavaScript’s operators fail:

```javascript
const money1 = { amount: 10, currency: "USD" };
const money2 = { amount: 20, currency: "USD" };
console.log(money1 + money2); // "[object Object][object Object]"
```

The `+` operator concatenates objects as strings, which is useless for arithmetic. You’d need custom functions:

```javascript
function addMoney(m1, m2) {
  if (m1.currency !== m2.currency) throw new Error("Currency mismatch");
  return { amount: m1.amount + m2.amount, currency: m1.currency };
}
console.log(addMoney(money1, money2)); // { amount: 30, currency: "USD" }
```

This requires a new function for each operation (`addMoney`, `subtractMoney`, etc.), and you must manually ensure currency compatibility. For multiple operations or complex types, this is repetitive and error-prone, with no guarantee of consistent support.

### PureScript’s `Field`: Type-Safe Numeric Operations
PureScript’s `Field` type class provides a standardized, type-safe way to define numeric operations (`+`, `-`, `*`, `/`) for custom types. It builds on superclasses like `Semiring` (for `+` and `*`), `Ring` (for `-`), `CommutativeRing` (for commutative multiplication), `DivisionRing` (for `/`), and `EuclideanRing` (for integral division and mod). The `Field` class is defined as:

```purescript
class (DivisionRing a, EuclideanRing a) <= Field a
```

Any type with a `Field` instance supports addition, subtraction, multiplication, and division, with compile-time checks. Here’s a `Money` type in PureScript:

```purescript
newtype AppMoney = AppMoney { amount :: Number, currency :: String }

derive instance Eq AppMoney

instance Show AppMoney where
  show (AppMoney { amount, currency }) = "(AppMoney " <> show amount <> " " <> show currency <> ")"

instance Semiring AppMoney where
  add (AppMoney m1) (AppMoney m2) =
    if m1.currency == m2.currency then AppMoney { amount: m1.amount + m2.amount, currency: m1.currency }
    else AppMoney { amount: 0.0, currency: "INVALID" }
  zero = AppMoney { amount: 0.0, currency: "USD" }
  mul (AppMoney m1) (AppMoney m2) =
    if m1.currency == m2.currency then AppMoney { amount: m1.amount * m2.amount, currency: m1.currency }
    else AppMoney { amount: 0.0, currency: "INVALID" }
  one = AppMoney { amount: 1.0, currency: "USD" }

instance Ring AppMoney where
  sub (AppMoney m1) (AppMoney m2) =
    if m1.currency == m2.currency then AppMoney { amount: m1.amount - m2.amount, currency: m1.currency }
    else AppMoney { amount: 0.0, currency: "INVALID" }

instance CommutativeRing AppMoney

instance DivisionRing AppMoney where
  recip (AppMoney m) =
    if m.amount /= 0.0 then AppMoney { amount: 1.0 / m.amount, currency: m.currency }
    else AppMoney { amount: 0.0, currency: "INVALID" }

instance EuclideanRing AppMoney where
  degree _ = 1
  div (AppMoney m1) (AppMoney m2) =
    if m1.currency == m2.currency && m2.amount /= 0.0 then AppMoney { amount: m1.amount / m2.amount, currency: m1.currency }
    else AppMoney { amount: 0.0, currency: "INVALID" }
  mod _ _ = AppMoney { amount: 0.0, currency: "USD" }

instance Field AppMoney
```

In PSCi:

```purescript
> import Prelude
> import Effect.Console
> let money1 = AppMoney { amount: 10.0, currency: "USD" }
> let money2 = AppMoney { amount: 20.0, currency: "USD" }
> logShow $ money1 + money2
(AppMoney 30.0 "USD")
unit
> logShow $ money1 * money2
(AppMoney 200.0 "USD")
unit
> logShow $ money1 / money2
(AppMoney 0.5 "USD")
unit
> logShow $ money1 + AppMoney { amount: 5.0, currency: "EUR" }
(AppMoney 0.0 "INVALID")
unit
```

The compiler ensures `AppMoney` supports all `Field` operations, catching errors like division by zero or currency mismatches at compile time if defined properly. The PureScript compiler optimizes `Field` operations to simple JavaScript for efficiency.

### Why `Field` Excites JavaScript Developers
1. **Unified Numeric Interface**: Unlike JavaScript’s custom functions (`addMoney`, etc.), `Field` provides a single interface for all numeric operations, reusable across types like `AppMoney`.

2. **Type-Safe Arithmetic**: Compile-time checks ensure all `Field` operations are defined, preventing runtime errors from undefined operations or type mismatches.

3. **Composes with Superclasses**: `Field` builds on `Semiring`, `Ring`, `CommutativeRing`, `DivisionRing`, and `EuclideanRing`, allowing partial numeric support (e.g., `Semiring` for types without subtraction) without duplicating code.

### But What About JavaScript’s `+` and Libraries?
JavaScript’s numeric operators are simple for numbers but useless for custom types without custom functions. Libraries like `math.js` can handle custom numeric types:

```javascript
const math = require('mathjs');
const money1 = math.unit('10 USD');
const money2 = math.unit('20 USD');
console.log(math.add(money1, money2).toString()); // "30 USD"
```

`math.js` is powerful but runtime-based, with no compile-time guarantees. Errors (e.g., currency mismatches) are caught at runtime, and you must learn its API. `Field` integrates with PureScript’s type system, ensuring correctness upfront with standard operators.

### REPL Exercises
Try these in a fresh PSCi session (run `:reset` first) to explore `Field`. Copy and paste each block:

1. **Basic Numeric Operations**:
   ```purescript
   import Prelude
   import Effect.Console
   logShow $ 1.0 + 2.0 * 3.0
   logShow $ 10.0 / 2.0
   ```
   Expected output:
   ```
   7.0
   unit
   5.0
   unit
   ```


2. **Custom Money Type**:
   ```purescript
   import Prelude
   import Effect.Console
   newtype AppMoney = AppMoney { amount :: Number, currency :: String }
   derive instance Eq AppMoney
   instance Show AppMoney where
     show (AppMoney { amount, currency }) = "(AppMoney " <> show amount <> " " <> show currency <> ")"
   instance Semiring AppMoney where
     add (AppMoney m1) (AppMoney m2) = if m1.currency == m2.currency
                                       then AppMoney { amount: m1.amount + m2.amount, currency: m1.currency }
                                       else AppMoney { amount: 0.0, currency: "INVALID" }
     zero = AppMoney { amount: 0.0, currency: "USD" }
     mul (AppMoney m1) (AppMoney m2) = if m1.currency == m2.currency
                                       then AppMoney { amount: m1.amount * m2.amount, currency: m1.currency }
                                       else AppMoney { amount: 0.0, currency: "INVALID" }
     one = AppMoney { amount: 1.0, currency: "USD" }
   instance Ring AppMoney where
     sub (AppMoney m1) (AppMoney m2) = if m1.currency == m2.currency
                                       then AppMoney { amount: m1.amount - m2.amount, currency: m1.currency }
                                       else AppMoney { amount: 0.0, currency: "INVALID" }
   instance CommutativeRing AppMoney
   instance DivisionRing AppMoney where
     recip (AppMoney m) = if m.amount /= 0.0
                          then AppMoney { amount: 1.0 / m.amount, currency: m.currency }
                          else AppMoney { amount: 0.0, currency: "INVALID" }
   instance EuclideanRing AppMoney where
     degree _ = 1
     div (AppMoney m1) (AppMoney m2) = if m1.currency == m2.currency && m2.amount /= 0.0
                                       then AppMoney { amount: m1.amount / m2.amount, currency: m1.currency }
                                       else AppMoney { amount: 0.0, currency: "INVALID" }
     mod _ _ = AppMoney { amount: 0.0, currency: "USD" }
   logShow $ (AppMoney { amount: 10.0, currency: "USD" }) + (AppMoney { amount: 5.0, currency: "EUR" })
   logShow $ (AppMoney { amount: 10.0, currency: "USD" }) + (AppMoney { amount: 5.0, currency: "USD" })
   ```
   Expected output:
   ```
   (AppMoney 0.0 "INVALID")
   (AppMoney 15.0 "USD")
   unit
   ```

3. **Test Type Safety**:
   ```purescript
   import Prelude
   import Effect.Console
   newtype NonNumeric = NonNumeric String
   derive instance Eq NonNumeric
   instance Show NonNumeric where
     show (NonNumeric s) = "(NonNumeric " <> show s <> ")"
   logShow $ (NonNumeric "test") + (NonNumeric "test")
   ```
   Expected output:
   ```
   Error: No type class instance was found for Data.Semiring.Semiring NonNumeric
   ```

### Summary
JavaScript’s numeric operators don’t work for custom types like `AppMoney`, requiring repetitive, error-prone custom functions. PureScript’s `Field` type class provides a type-safe, reusable way to define arithmetic operations, with compile-time guarantees and efficient compilation. For JavaScript developers, `Field` simplifies working with custom numeric types, making PureScript’s type-driven approach a powerful tool for robust calculations.

