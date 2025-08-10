## Foldable

### The JavaScript Struggle: Reducing Collections

In JavaScript, reducing a collection to a single value is common, using `Array.reduce`:

```javascript
const numbers = [1, 2, 3, 4, 5];
const sum = numbers.reduce((acc, x) => acc + x, 0); // 15
console.log(sum);
const concatenated = numbers.reduce((acc, x) => acc + x.toString(), ""); // "12345"
console.log(concatenated);
```

This works for arrays, but for other structures like optional values (similar to Maybe) or pairs (similar to Tuple), there's no standard way. For an optional value, you might hack it:

```javascript
const optionalNumber = 42; // or null
const sumOptional = optionalNumber ? 0 + optionalNumber : 0; // 42
console.log(sumOptional);
```

For a pair:

```javascript
const pair = { first: 1, second: 2 };
const sumPair = pair.first + pair.second; // 3 - manual, not generalized
console.log(sumPair);
```

You must define custom reduce functions for each type, and there's no unified way to fold over different structures. For non-array collections like objects (as maps), you need `Object.values` or `Object.keys` with reduce, which is ad hoc and error-prone:

```javascript
const obj = { a: 1, b: 2, c: 3 };
const sumObj = Object.values(obj).reduce((acc, x) => acc + x, 0); // 6
console.log(sumObj);
```

Even more problematic are `Map` objects, which are iterable but lack a built-in reduce. You must convert to an array first:

```javascript
const myMap = new Map([['a', 1], ['b', 2], ['c', 3]]);
const sumMap = Array.from(myMap.values()).reduce((acc, v) => acc + v, 0); // 6
console.log(sumMap);
const concatenatedKeys = Array.from(myMap.keys()).reduce((acc, k) => acc + k, ""); // "abc"
console.log(concatenatedKeys);
```

This conversion is repetitive and inefficient for large Maps. Forget it, and you get runtime errors. No type safety checks if the values are reducible (e.g., numbers for sum).

### PureScript’s `Foldable`: Type-Safe Folding

PureScript’s `Foldable` type class provides a standardized way to fold over container types like arrays, `Maybe`, `Tuple`, and maps, reducing them to a single value. It’s defined as:

```purescript
class Foldable f where
  foldr :: forall a b. (a -> b -> b) -> b -> f a -> b
  foldl :: forall a b. (b -> a -> b) -> b -> f a -> b
  foldMap :: forall a m. Monoid m => (a -> m) -> f a -> m
```

`foldr` and `foldl` are right and left folds, like JS reduceRight and reduce. `foldMap` is powerful: it maps each element to a monoid value and combines them using the monoid's `<>`.

For arrays, `foldMap` specializes to reducing to a monoid:

In PSCi:

```purescript
import Prelude
import Effect.Console
import Data.Foldable
logShow $ foldMap show [1, 2, 3, 4, 5]
```

Expected output:

```
"12345"
unit
```

Here, we map each number to a string (using `show`), and combine with the string monoid (concatenation). This is like JS reduce with string concatenation, but generalized.

`Foldable` works for other types, like `Maybe` (optional values) and `Tuple` (pairs). For example, folding a `Maybe`:

In PSCi:

```purescript
import Data.Maybe
logShow $ foldMap show (Just 42)
logShow $ foldMap show (Nothing :: Maybe Int)
```

Expected output:

```
"42"
unit
""
unit
```

Folding a `Tuple` (note: folds over the second component, treating the first as fixed):

In PSCi:

```purescript
import Data.Tuple
logShow $ foldMap show (Tuple 1 2)
```

Expected output:

```
"2"
unit
```

For a type like `Map` (from `purescript-ordered-collections`), which has a Foldable instance, you can fold directly without conversion:

In PSCi:

```purescript
import Data.Map as Map
import Data.Tuple
myMap = Map.fromFoldable [Tuple "a" 1, Tuple "b" 2, Tuple "c" 3]
logShow $ foldl (+) 0 myMap
logShow $ foldMap show myMap
```

Expected output:

```
6
unit
"123"
unit
```

The compiler ensures all types used with fold functions have a `Foldable` instance, catching errors at compile time. `Foldable` captures the notion of an ordered container that can be reduced.

### Why `Foldable` Is Helpful

1. **Generalized Reducing**: Unlike JS's array-specific `reduce`, `Foldable` works on arrays, `Maybe`, `Tuple`, maps, and custom types, with a uniform interface.
    
2. **Monoid Integration**: `foldMap` combines with `Monoid` (from the previous section) for safe, associative reductions, like reducing to strings or numbers, with compile-time safety.
    
3. **Type-Safe**: No runtime errors from reducing non-reducible structures; the compiler enforces `Foldable` instances.
    

### Practical Example: Reducing User Data

Suppose you have optional user data (Maybe) or paired data (Tuple). In JS, you'd handle each case manually. In PureScript, `foldMap` reduces uniformly:

In PSCi:

```purescript
import Prelude
import Effect.Console
import Data.Foldable
import Data.Maybe
import Data.Tuple
logShow $ foldMap (\s -> s <> " logged in") (Just "user1")
logShow $ foldMap (\s -> "status: " <> s) (Tuple "user2" " active")
```

Expected output:

```
"user1 logged in"
unit
" active status:"
unit
```

This generalizes reduce, making code reusable across structures.

### REPL Exercises

Try these in a fresh PSCi session (run `:clear` first). Copy and paste each block:

1. **Fold an Array**:
    
    ```purescript
    import Prelude
    import Effect.Console
    import Data.Foldable
    logShow $ foldl (+) 0 [1, 2, 3, 4, 5]
    logShow $ foldMap show [1, 2, 3, 4, 5]
    ```
    
    Expected output:
    
    ```
    15
    unit
    "12345"
    unit
    ```
    
2. **Fold a Maybe**:
    
    ```purescript
    import Prelude
    import Effect.Console
    import Data.Foldable
    import Data.Maybe
    logShow $ foldMap show (Just 42)
    logShow $ foldMap show (Nothing :: Maybe Int)
    ```
    
    Expected output:
    
    ```
    "42"
    unit
    ""
    unit
    ```
    
3. **Fold a Tuple**:
    
    ```purescript
    import Prelude
    import Effect.Console
    import Data.Foldable
    import Data.Tuple
    logShow $ foldMap show (Tuple 1 2)
    ```
    
    Expected output:
    
    ```
    "2"
    unit
    ```
    

### Summary

JavaScript’s `reduce` is array-specific and requires manual handling for other structures like optionals or pairs, with clunky conversions for types like Map. PureScript’s `Foldable` type class generalizes folding over containers like arrays, `Maybe`, `Tuple`, and maps, with type-safe, reusable operations. For JavaScript developers, `Foldable` simplifies reducing diverse data shapes, reducing custom code and errors in aggregations.


