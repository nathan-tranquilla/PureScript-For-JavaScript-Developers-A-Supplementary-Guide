## Functor

### JavaScript Context: Mapping Over Structures
In JavaScript, mapping over arrays to transform elements is common with `Array.map`:

```javascript
const numbers = [1, 2, 3];
const doubled = numbers.map(x => x * 2); // [2, 4, 6]
console.log(doubled);
```

For other structures, it's less formalized. For optionals (null/undefined):

```javascript
const optionalNumber = 42; // or null
const doubledOptional = optionalNumber !== null ? optionalNumber * 2 : null; // 84 or null
console.log(doubledOptional);
```

For promises:

```javascript
const promise = Promise.resolve(42);
const doubledPromise = promise.then(x => x * 2); // resolves to 84
doubledPromise.then(console.log);
```

For objects (as maps):

```javascript
const obj = { a: 1, b: 2, c: 3 };
const doubledObj = Object.fromEntries(Object.entries(obj).map(([k, v]) => [k, v * 2])); // { a: 2, b: 4, c: 6 }
console.log(doubledObj);
```

This is flexible but ad hoc—you write custom mapping for each type, with no guarantees like identity (mapping x => x returns original) or composition (mapping f then g equals mapping g ◦ f). Chained maps might break if not handled properly, and there's no type safety for mappable structures.

To show how composition can go wrong, consider a custom map with side effects (like logging):

```javascript
let count = 0;
function badMap(f, arr) {
  count++; // Side effect: increment count
  return arr.map(f);
}
const f = x => x * 2;
const g = x => x + 1;
const arr = [1, 2, 3];
const left = badMap(f, badMap(g, arr)); // count incremented twice
console.log(left, count); // [4, 6, 8], 2
count = 0; // Reset for next
const right = badMap(x => f(g(x)), arr); // count incremented once
console.log(right, count); // [4, 6, 8], 1
```

The results are the same, but the side effects (count) differ based on grouping, violating composition predictability. In real code, this could cause bugs with shared state or performance issues.

### PureScript’s `Functor`: A Structured Approach
PureScript’s `Functor` type class formalizes mapping over structures, with laws for predictability. It’s defined as:

```purescript
class Functor f where
  map :: forall a b. (a -> b) -> f a -> f b
```

The `map` function (alias `<$>`) lifts a function over the structure. Laws: identity (map id == id) and composition (map (f ◦ g) == map f ◦ map g).

For arrays:

In PSCi:

```purescript
import Prelude
import Effect.Console
import Data.Array
logShow $ map (\x -> x * 2) [1, 2, 3]
```

Expected output:
```
[2,4,6]
unit
```

For `Maybe`:

In PSCi:

```purescript
import Data.Maybe
logShow $ map (\x -> x * 2) (Just 42)
logShow $ map (\x -> x * 2) Nothing
```

Expected output:
```
(Just 84)
unit
Nothing
unit
```

For `Tuple` (maps second component):

In PSCi:

```purescript
import Data.Tuple
logShow $ map (\x -> x * 2) (Tuple "fixed" 42)
```

Expected output:
```
(Tuple "fixed" 84)
unit
```

The compiler ensures `Functor` instances exist, catching errors at compile time. Laws make mapping reliable—e.g., composition preserves order without side effects.

### Why PureScript’s `Functor` Is Helpful
1. **Compile-Time Safety**: In JS, mapping non-mappable types fails at runtime. Functor requires instances, catching errors early.

2. **Consistent Mapping**: Uniform `map` for arrays, `Maybe`, `Tuple`, maps— no custom hacks.

3. **Laws for Predictability**: Identity and composition ensure mapping behaves as expected, unlike JS where chained maps can surprise (e.g., with side effects).

### Practical Example: Mapping User Data
Map over optional data (Maybe) or pairs (Tuple) uniformly:

In PSCi:

```purescript
import Prelude
import Effect.Console
import Data.Maybe
import Data.Tuple
logShow $ map (\s -> s <> " logged in") (Just "user1")
logShow $ map (\s -> s <> " status:") (Tuple "user2" "active")
```

Expected output:
```
(Just "user1 logged in")
unit
(Tuple "user2" "active status:")
unit
```

This generalizes mapping, reusable across structures.

### REPL Exercises
Try these in a fresh PSCi session (run `:clear` first). Copy and paste each block:

1. **Map an Array**:
   ```purescript
   import Prelude
   import Effect.Console
   import Data.Array
   logShow $ map (\x -> x * 2) [1, 2, 3]
   ```
   Expected output:
   ```
   [2,4,6]
   unit
   ```

2. **Map a Maybe**:
   ```purescript
   import Prelude
   import Effect.Console
   import Data.Maybe
   logShow $ map (\x -> x * 2) (Just 42)
   logShow $ map (\x -> x * 2) Nothing
   ```
   Expected output:
   ```
   (Just 84)
   unit
   Nothing
   unit
   ```

3. **Map a Tuple**:
   ```purescript
   import Prelude
   import Effect.Console
   import Data.Tuple
   logShow $ map (\x -> x * 2) (Tuple "fixed" 42)
   ```
   Expected output:
   ```
   (Tuple "fixed" 84)
   unit
   ```

### Summary
JavaScript’s mapping is structure-specific and requires manual handling for optionals or pairs. PureScript’s `Functor` type class generalizes mapping over containers like arrays, `Maybe`, `Tuple`, with type-safe, law-abiding operations. For JavaScript developers, `Functor` simplifies transforming data in diverse structures, reducing custom code and errors in transformations.


