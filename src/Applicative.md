## Applicative Functors 

### JavaScript Context: Validating and Combining Form Inputs
In JavaScript, you often validate form inputs like first name, last name, and email before combining them into a user object. A common approach is using `null` checks:

```javascript
function createUser(first, last, email) {
  if (first && last && email) {
    return { first, last, email };
  }
  return null;
}
console.log(createUser("Phillip", "Freeman", "phil@example.com")); // { first: "Phillip", last: "Freeman", email: "phil@example.com" }
console.log(createUser("Phillip", null, "phil@example.com")); // null
```

This is straightforward but limited. To provide better feedback, you might validate each input and return a result with either the user object or errors, wrapping the result in a context:

```javascript
function validateInput(value, field) {
  if (!value) return { valid: false, errors: [`${field} is required`] };
  if (field === "email" && !value.includes("@")) return { valid: false, errors: ["Valid email is required"] };
  return { valid: true, value };
}
function createUserValidated(first, last, email) {
  const firstResult = validateInput(first, "First name");
  const lastResult = validateInput(last, "Last name");
  const emailResult = validateInput(email, "Email");
  if (firstResult.valid && lastResult.valid && emailResult.valid) {
    return { valid: true, value: { first: firstResult.value, last: lastResult.value, email: emailResult.value } };
  }
  const errors = [];
  if (!firstResult.valid) errors.push(...firstResult.errors);
  if (!lastResult.valid) errors.push(...lastResult.errors);
  if (!emailResult.valid) errors.push(...emailResult.errors);
  return { valid: false, errors };
}
console.log(createUserValidated("Phillip", "Freeman", "phil@example.com")); 
// { valid: true, value: { first: "Phillip", last: "Freeman", email: "phil@example.com" } }
console.log(createUserValidated("Phillip", null, "invalid")); 
// { valid: false, errors: ["Last name is required", "Valid email is required"] }
```

This improved version applies `createUser` to validated inputs within a context (`{ valid, value }` or `{ errors }`), but it’s repetitive. You manually check each result, combine errors, and ensure types match. Extending this to other structures (e.g., arrays or async data) requires more custom code, and there’s no type safety to catch errors early, risking runtime issues if inputs are unexpected types.

### PureScript’s `Applicative`: Structured Function Application
PureScript’s `Applicative` type class builds on `Apply` (which itself builds on `Functor`) to apply functions inside containers (like optionals, arrays, or async computations) to values in other containers, with type safety. It’s defined as:

```purescript
class Apply f <= Applicative f where
  pure :: forall a. a -> f a
```

The `Apply` type class, a prerequisite for `Applicative`, provides:

```purescript
class Functor f <= Apply f where
  apply :: forall a b. f (a -> b) -> f a -> f b
```

- `pure` wraps a value or function into the container’s context. For example, with `Maybe`, `pure x = Just x`; with arrays, `pure x = [x]`. In Applicatives, `pure` lifts a regular function (like `createUser`) into the container (e.g., `Just createUser` or `[createUser]`) so it can be applied to wrapped values.
- `apply` (alias `<*>` ) takes a function in a container and applies it to a value in another container of the same type.

The `pure` function is crucial: it takes a plain function, like `createUser :: String -> String -> String -> { first :: String, last :: String, email :: String }`, and lifts it into the applicative context (e.g., `Just createUser` for `Maybe` or `[createUser]` for arrays). This enables `<*>` to apply the function to wrapped values, combining their effects (e.g., failing if any `Maybe` is `Nothing`, or generating all combinations for arrays).

For example, combining form inputs with `Maybe` (optional values):

In PSCi, paste (using `:paste` mode, then `Ctrl+D`):

```purescript
import Prelude
import Effect.Console
import Data.Maybe
createUser first last email = { first, last, email }
logShow $ pure createUser <*> Just "Phillip" <*> Just "Freeman" <*> Just "phil@example.com" 
logShow $ pure createUser <*> Just "Phillip" <*> (Nothing :: Maybe String) <*> Just "phil@example.com" 
```

Expected output:
```
(Just { first: "Phillip", last: "Freeman", email: "phil@example.com" })
unit
Nothing
unit
```

For arrays (applying to all combinations):

In PSCi, paste (using `:paste` mode, then `Ctrl+D`):

```purescript
import Prelude
import Effect.Console
import Data.Array
createUser first last email = { first, last, email }
logShow $ pure createUser <*> ["Phillip", "Phil"] <*> ["Freeman", "Smith"] <*> ["phil@example.com", "psmith@example.com"] 
```

Expected output:
```
[{ first: "Phillip", last: "Freeman", email: "phil@example.com" },{ first: "Phillip", last: "Freeman", email: "psmith@example.com" },{ first: "Phillip", last: "Smith", email: "phil@example.com" },{ first: "Phillip", last: "Smith", email: "psmith@example.com" },{ first: "Phil", last: "Freeman", email: "phil@example.com" },{ first: "Phil", last: "Freeman", email: "psmith@example.com" },{ first: "Phil", last: "Smith", email: "phil@example.com" },{ first: "Phil", last: "Smith", email: "psmith@example.com" }]
unit
```

The compiler ensures `Applicative` instances exist and types match, catching errors at compile time. Applicative laws (identity, composition, homomorphism) ensure predictable behavior, unlike JavaScript’s manual checks.

### Why `Applicative` Is Helpful
1. **Unified Function Application**: Unlike JS’s repetitive `null` checks, `Applicative` applies functions in containers (like optionals, arrays, async computations) to values in containers uniformly, without manual conditionals.

2. **Type Safety**: Ensures the function and values are compatible before running, avoiding JS runtime errors from missing or mismatched inputs.

3. **Reusable Code**: One `apply` operator works across different structures, reducing custom code compared to JS’s type-specific checks.


The `pure` function lifts `createUser` into `Maybe`, enabling `<*>` to apply it to optional values, similar to JavaScript’s validated checks but safer and more structured.

### REPL Exercises
Try these in a fresh PSCi session (run `:clear` first). Copy and paste each block using `:paste` mode (type `:paste`, paste the code, then press `Ctrl+D` to execute):

1. **Apply with Maybe**:
   ```purescript
   import Prelude
   import Effect.Console
   import Data.Maybe
   createUser first last email = { first, last, email }
   logShow $ pure createUser <*> Just "Phillip" <*> Just "Freeman" <*> Just "phil@example.com"
   logShow $ pure createUser <*> Just "Phillip" <*> (Nothing :: Maybe String) <*> Just "phil@example.com" 
   ```
   Expected output:
   ```
   (Just { first: "Phillip", last: "Freeman", email: "phil@example.com" })
   unit
   Nothing
   unit
   ```

2. **Apply with Array**:
   ```purescript
   import Prelude
   import Effect.Console
   import Data.Array
   createUser first last email = { first, last, email }
   logShow $ pure createUser <*> ["Phillip", "Phil"] <*> ["Freeman", "Smith"] <*> ["phil@example.com", "psmith@example.com"]
   ```
   Expected output:
   ```
   [{ first: "Phillip", last: "Freeman", email: "phil@example.com" },{ first: "Phillip", last: "Freeman", email: "psmith@example.com" },{ first: "Phillip", last: "Smith", email: "phil@example.com" },{ first: "Phillip", last: "Smith", email: "psmith@example.com" },{ first: "Phil", last: "Freeman", email: "phil@example.com" },{ first: "Phil", last: "Freeman", email: "psmith@example.com" },{ first: "Phil", last: "Smith", email: "phil@example.com" },{ first: "Phil", last: "Smith", email: "psmith@example.com" }]
   unit
   ```

3. **Apply with Custom Type**:
   ```purescript
    import Prelude
    import Effect.Console
    newtype Point a = Point { x :: a, y :: a }
    instance Show a => Show (Point a) where
      show (Point p) = "(Point " <> show p.x <> " " <> show p.y <> ")"
    instance Functor Point where
      map f (Point p) = Point { x: f p.x, y: f p.y }
    instance Apply Point where
      apply (Point p) (Point q) = Point { x: p.x q.x, y: p.y q.y }
    instance Applicative Point where
      pure x = Point { x: x, y: x }
    logShow $ pure (\n -> n * 2) <*> Point { x: 3, y: 4 } {- - Double coordinates with pure -}
   ```

  **Explanation**:
  This example works because `Point` is defined with a generic type `a` (`newtype Point a`), allowing it to hold any type, including functions like `Int -> Int`. The `Show a => Show (Point a)` instance depends on the `Show` instance for `a`, ensuring that the `x` and `y` fields (e.g., integers or functions) can be converted to strings for display. This is an instance dependency, similar to how `Show (Array a)` requires `Show a`, as seen in Chapter 6. In the expression `pure (\n -> n * 2) <*> Point { x: 3, y: 4 }`:
  - `pure (\n -> n * 2)` lifts the function `\n -> n * 2 :: Int -> Int` into `Point`, creating `Point { x: \n -> n * 2, y: \n -> n * 2 } :: Point (Int -> Int)`.
  - The `<*>` operator (from `Apply`) applies this `Point` of functions to `Point { x: 3, y: 4 } :: Point Int`, producing `Point { x: (\n -> n * 2) 3, y: (\n -> n * 2) 4 } = Point { x: 6, y: 8 }`.
  - The `Show` instance for `Point Int` relies on `Show Int` to display `(Point 6 8)`. This flexibility makes `Point` a useful example of how Applicatives can work with custom types, mirroring JavaScript’s ability to transform objects but with type safety.

   
  Expected output:
  ```
  (Point 6 8)
  unit
  ```

### Summary
JavaScript’s manual validation and `null` checks for combining form inputs are repetitive and risk runtime errors. PureScript’s `Applicative` type class, with `pure` lifting functions into containers like `Maybe`, arrays, or `Aff`, provides a type-safe, reusable way to apply functions to values in other containers. For JavaScript developers, `Applicative` simplifies combining data across diverse structures, catching errors early and reducing custom code.

