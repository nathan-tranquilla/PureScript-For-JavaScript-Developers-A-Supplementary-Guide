## Multi-Parameter Type Classes

Multi-parameter type classes allow you to define relationships between multiple types, enforced at compile time. This section explores the `Action` type class, which defines how a monoid `m` acts on a type `a`, and shows how it provides a structured, type-safe alternative to JavaScript’s ad hoc type combinations.

### JavaScript Context: Ad Hoc Type Combinations
In JavaScript, functions often work with multiple types, relying on runtime checks to ensure compatibility. For example, consider a function that applies an action (like scaling a number or repeating a string) based on the types involved:

```javascript
function applyAction(action, value) {
  if (action.type === 'multiply' && typeof value === 'number') {
    return action.factor * value;
  } else if (action.type === 'repeat' && typeof value === 'string') {
    return value.repeat(action.factor);
  } else {
    throw new Error('Incompatible action and value');
  }
}

const multiplyAction = { type: 'multiply', factor: 2 };
const repeatAction = { type: 'repeat', factor: 3 };
console.log(applyAction(multiplyAction, 5)); // 10
console.log(applyAction(repeatAction, "hi")); // "hihihi"
console.log(applyAction(multiplyAction, "hi")); // Error: Incompatible action and value
```

This approach is flexible but error-prone. You need runtime checks to ensure the action and value types are compatible, and errors only appear when the code runs. Adding new action types or value types requires updating the function with more conditions, increasing complexity and the risk of bugs. There’s no compile-time guarantee that the types align correctly, and no formal way to express that an action’s behavior depends on both the action type and the value type.

### PureScript’s Multi-Parameter Type Classes: Structured Type Relationships
PureScript’s multi-parameter type classes allow you to define relationships between multiple types, enforced at compile time. The `Action` type class defines how a monoid `m` acts on a type `a`:

```purescript
class Monoid m <= Action m a where
  act :: m -> a -> a
```

The `Action` class requires `m` to be a `Monoid` (with an `append` operation and `mempty` identity) and defines how `m` modifies `a`. It has two laws:
- `act mempty a = a` (identity: the monoid’s empty value doesn’t change `a`)
- `act (m1 <> m2) a = act m1 (act m2 a)` (composition: combining monoids before acting is the same as applying actions sequentially)

The book defines a `Multiply` monoid for scaling numbers or repeating strings:

```purescript
newtype Multiply = Multiply Int

instance Semigroup Multiply where
  append (Multiply n) (Multiply m) = Multiply (n * m)

instance Monoid Multiply where
  mempty = Multiply 1
```

An instance for `Action Multiply Int` scales an integer:

```purescript
instance Action Multiply Int where
  act (Multiply n) x = n * x
```

An instance for `Action Multiply String` repeats a string:

```purescript
instance Action Multiply String where
  act (Multiply n) s = Data.Foldable.fold $ replicate n s
```

These instances ensure type-safe operations. The compiler checks that only valid type pairs (e.g., `Multiply` with `Int` or `String`) are used with `act`.

In PSCi, paste (using `:paste` mode, then `Ctrl+D`):

```purescript
import Prelude
import Effect.Console
import Data.Foldable
import Data.Array
class Monoid m <= Action m a where
  act :: m -> a -> a
newtype Multiply = Multiply Int
instance Semigroup Multiply where
  append (Multiply n) (Multiply m) = Multiply (n * m)
instance Monoid Multiply where
  mempty = Multiply 1
instance Action Multiply Int where
  act (Multiply n) x = n * x
instance Action Multiply String where
  act (Multiply n) s = Data.Foldable.fold $ replicate n s
logShow $ act (Multiply 2) 5
logShow $ act (Multiply 3) "hi"
```

Expected output:
```
10
unit
"hihihi"
unit
```

Try an invalid combination:

```purescript
import Prelude
import Effect.Console
class Monoid m <= Action m a where
  act :: m -> a -> a
newtype Multiply = Multiply Int
instance Semigroup Multiply where
  append (Multiply n) (Multiply m) = Multiply (n * m)
instance Monoid Multiply where
  mempty = Multiply 1
instance Action Multiply Int where
  act (Multiply n) x = n * x
logShow $ act (Multiply 2) true
```

Expected error:
```
Error: No type class instance was found for Action Multiply Boolean
```

This demonstrates compile-time enforcement of valid type relationships.

### Why Multi-Parameter Type Classes Are Helpful
1. **Compile-Time Type Safety**: Unlike JavaScript’s runtime checks, multi-parameter type classes ensure valid type pairings at compile time, preventing errors like applying a scaling action to a string.

2. **Modular Extensions**: Add new actions or value types by defining new `Action` instances without modifying existing code, unlike JavaScript’s growing conditionals.

3. **Expressive Relationships**: Encode dependencies (e.g., `Action` requiring `Monoid`) and specific type interactions, making code clearer and safer than JavaScript’s ad hoc checks.

### Practical Example: Action on Custom Types
Apply `Multiply` to a custom `Point` type, scaling its coordinates:

In PSCi, paste (using `:paste` mode, then `Ctrl+D`):

```purescript
import Prelude
import Effect.Console
class Monoid m <= Action m a where
  act :: m -> a -> a
newtype Multiply = Multiply Int
instance Semigroup Multiply where
  append (Multiply n) (Multiply m) = Multiply (n * m)
instance Monoid Multiply where
  mempty = Multiply 1
newtype Point = Point { x :: Int, y :: Int }
instance Action Multiply Point where
  act (Multiply n) (Point p) = Point { x: n * p.x, y: n * p.y }
instance Show Point where
  show (Point p) = "(Point " <> show p.x <> " " <> show p.y <> ")"
logShow $ act (Multiply 2) (Point { x: 3, y: 4 })
```

Expected output:
```
(Point 6 8)
unit
```

This shows how `Action` can extend to custom types, maintaining type safety.

### REPL Exercises
Try these in a fresh PSCi session (run `:clear` first). Copy and paste each block using `:paste` mode (type `:paste`, paste the code, then press `Ctrl+D` to execute):

1. **Action on Int**:
   ```purescript
   import Prelude
   import Effect.Console
   class Monoid m <= Action m a where
     act :: m -> a -> a
   newtype Multiply = Multiply Int
   instance Semigroup Multiply where
     append (Multiply n) (Multiply m) = Multiply (n * m)
   instance Monoid Multiply where
     mempty = Multiply 1
   instance Action Multiply Int where
     act (Multiply n) x = n * x
   logShow $ act (Multiply 2) 5
   ```
   Expected output:
   ```
   10
   unit
   ```

2. **Action on String**:
   ```purescript
   import Prelude
   import Effect.Console
   import Data.Foldable
   import Data.Array
   class Monoid m <= Action m a where
     act :: m -> a -> a
   newtype Multiply = Multiply Int
   instance Semigroup Multiply where
     append (Multiply n) (Multiply m) = Multiply (n * m)
   instance Monoid Multiply where
     mempty = Multiply 1
   instance Action Multiply String where
     act (Multiply n) s = Data.Foldable.fold $ replicate n s
   logShow $ act (Multiply 3) "hi"
   ```
   Expected output:
   ```
   "hihihi"
   unit
   ```

3. **Action on Custom Point**:
   ```purescript
   import Prelude
   import Effect.Console
   class Monoid m <= Action m a where
     act :: m -> a -> a
   newtype Multiply = Multiply Int
   instance Semigroup Multiply where
     append (Multiply n) (Multiply m) = Multiply (n * m)
   instance Monoid Multiply where
     mempty = Multiply 1
   newtype Point = Point { x :: Int, y :: Int }
   instance Action Multiply Point where
     act (Multiply n) (Point p) = Point { x: n * p.x, y: n * p.y }
   instance Show Point where
     show (Point p) = "(Point " <> show p.x <> " " <> show p.y <> ")"
   logShow $ act (Multiply 2) (Point { x: 3, y: 4 })
   ```
   Expected output:
   ```
   (Point 6 8)
   unit
   ```

### Summary
JavaScript’s runtime checks for type combinations are fragile and error-prone. PureScript’s multi-parameter type classes, like `Action`, provide a type-safe way to define relationships between types, such as a monoid acting on a value, enforced at compile time. For JavaScript developers, this reduces bugs by ensuring valid type pairings and enables modular, reusable code.
