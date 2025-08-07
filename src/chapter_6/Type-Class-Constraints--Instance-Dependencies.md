## Type Class Constraints & Instance Dependencies

### JavaScript Context: Assuming and Documenting Method Dependencies
In JavaScript, functions often depend on objects having certain methods or properties, like `toString` for string conversion or `equals` for custom equality. You assume they're there or check at runtime:

```javascript
function showEquality(x, y) {
  if (typeof x.equals === 'function' && x.equals(y)) {
    return x.toString() + " equals " + y.toString();
  } else {
    return x.toString() + " does not equal " + y.toString();
  }
}
const obj1 = { value: 42, toString: () => "obj1", equals: (other) => this.value === other.value };
const obj2 = { value: 42, toString: () => "obj2", equals: (other) => this.value === other.value };
console.log(showEquality(obj1, obj2)); // "obj1 equals obj2"
const obj3 = { value: 43 }; // No equals or toString
console.log(showEquality(obj1, obj3)); // Runtime fallback or error
```

This works if methods exist, but if missing, you get runtime errors or unexpected behavior. For dependencies (e.g., equals depending on toString for hashing), you must manually check or document—no enforcement. In libraries, this leads to fragile code where assumptions about "interfaces" aren't verified, only documented in comments or JSDoc.

### PureScript’s Type Class Constraints and Instance Dependencies: A Structured Approach
PureScript’s type class constraints restrict functions to types that support specific operations, enforced at compile time. Instance dependencies allow instances to rely on other instances, forming chains the compiler resolves.

Constraints in functions:

```purescript
showEquality :: forall a. (Show a, Eq a) => a -> a -> String
showEquality x y | x == y = show x <> " equals " <> show y
showEquality x y = show x <> " does not equal " <> show y
```

This requires `a` to have `Show` and `Eq` instances. The compiler checks this.

Instance dependencies in declarations (e.g., `Show` for arrays depends on `Show` for elements):

```purescript
instance Show a => Show (Array a) where
  show xs = "[" <> intercalate ", " (map show xs) <> "]"
```

For multiple dependencies:

```purescript
instance (Show a, Show b) => Show (Either a b) where
  show (Left a) = "(Left " <> show a <> ")"
  show (Right b) = "(Right " <> show b <> ")"
```

The compiler resolves instances automatically, inferring from types and hiding complexity. Dependencies form chains (e.g., `Show (Array (Either Int String))` requires `Show Int` and `Show String`).

In PSCi:

```purescript
import Prelude
import Effect.Console
import Data.Either
import Data.Array
logShow [Left 1, Right "hello"]
```

Expected output:
```
["(Left 1)","(Right \"hello\")"]
unit
```

The compiler infers and resolves `Show` instances for `Array`, `Either`, `Int`, and `String`.

### Why PureScript’s Type Class Constraints and Instance Dependencies Are Helpful
1. **Compile-Time Enforcement**: In JS, dependencies are runtime assumptions leading to errors. Constraints and dependencies catch unsupported types or missing implementations early.

2. **Automatic Resolution**: The compiler "infers" code from types, like deriving array showing from element showing, reducing boilerplate.

3. **Safe Hierarchies**: Superclasses (e.g., `Eq` for `Ord`) and dependencies enforce relationships, preventing incomplete implementations without manual checks.

### Practical Example: Dependent Showing
Show an array of Eithers, depending on instances for Array, Either, Int, String:

In PSCi:

```purescript
import Prelude
import Effect.Console
import Data.Either
import Data.Array
logShow [Left 1, Right "hello"]
logShow [Left true] // Compile-time error: No Show for Boolean in Either
```

Expected output:
```
["(Left 1)","(Right \"hello\")"]
unit
```

Error for second line:
```
Error: No type class instance was found for Data.Show.Show Boolean
```

This shows dependency enforcement.

### REPL Exercises
Try these in a fresh PSCi session (run `:clear` first). Copy and paste each block:

1. **Constrained Equality**:
   ```purescript
   import Prelude
   import Effect.Console
   showEquality x y | x == y = show x <> " equals " <> show y
   showEquality x y = show x <> " does not equal " <> show y
   log $ showEquality 1 1
   log $ showEquality 1 2
   ```
   Expected output:
   ```
   "1 equals 1"
   "1 does not equal 2"
   ```

2. **Dependent Array Show**:
   ```purescript
   import Prelude
   import Effect.Console
   import Data.Array
   logShow [1, 2, 3] // Depends on Show Int for Array Int
   ```
   Expected output:
   ```
   [1,2,3]
   unit
   ```

3. **Dependent Either Show**:
   ```purescript
   import Prelude
   import Effect.Console
   import Data.Either
   logShow (Left 42) // Depends on Show Int
   logShow (Right "hello") // Depends on Show String
   ```
   Expected output:
   ```
   (Left 42)
   unit
   (Right "hello")
   unit
   ```

4. **Combined Dependency**:
   ```purescript
   import Prelude
   import Effect.Console
   import Data.Either
   import Data.Array
   logShow [Left 1, Right "hello"] // Depends on Show Array, Show Either, Show Int, Show String
   ```
   Expected output:
   ```
   ["(Left 1)","(Right \"hello\")"]
   unit
   ```

### Summary
JavaScript’s functions assume methods exist without enforcement, leading to runtime errors. PureScript’s type class constraints and instance dependencies provide compile-time safety for functions and implementations, with automatic resolution and hierarchies. For JavaScript developers, they reduce bugs by ensuring types support needed operations upfront.
