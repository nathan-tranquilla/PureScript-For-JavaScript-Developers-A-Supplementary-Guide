## Traversable Functors

### JavaScript Context: Validating Arrays of Data
In JavaScript, you often validate collections of data, like an array of phone numbers, to ensure each element is valid before using it. A common approach is mapping over the array with a validation function and collecting results or errors:

```javascript
function validatePhoneNumber(phone) {
  const regex = /^\d{3}-\d{3}-\d{4}$/;
  return regex.test(phone) ? { valid: true, value: phone } : { valid: false, errors: ["Invalid phone format"] };
}
function validatePhones(phones) {
  const results = phones.map(validatePhoneNumber);
  const errors = results.filter(r => !r.valid).map(r => r.errors).flat();
  return errors.length ? { valid: false, errors } : { valid: true, value: results.map(r => r.value) };
}
console.log(validatePhones(["555-555-5555", "555-555-0000"])); 
// { valid: true, value: ["555-555-5555", "555-555-0000"] }
console.log(validatePhones(["555-555-5555", "invalid"])); 
// { valid: false, errors: ["Invalid phone format"] }
```

**How This Differs from the Applicative Example**: Unlike the Applicative example’s `createUserValidated`, which validates and combines independent inputs (first name, last name, email) into a single user object, this example iterates over a single array of phone numbers, applying `validatePhoneNumber` to each element and collecting results or errors while keeping the array structure. The Applicative example lifts a function over multiple validated inputs, whereas this maps a validation function across a collection, preserving its shape. This distinction highlights Traversable’s focus on processing each element in a structure, not combining separate values.

This pattern—mapping a validation function over an array and collecting results or errors—is common in form validation or data processing. However, it’s repetitive and specific to arrays. If you need to validate other structures (like optional values or nested objects), you write custom code for each. There’s no type safety to ensure the validation function and data match, and runtime errors can occur if inputs are unexpected types, like non-strings in the `phones` array.

### PureScript’s `Traversable`: Structured Iteration with Effects
PureScript’s `Traversable` type class builds on `Functor` and `Foldable` to iterate over a data structure (like arrays, optionals, or custom types), applying an effectful function (e.g., validation) to each element and collecting results in a new structure of the same shape, with type safety. It’s defined as:

```purescript
class (Functor t, Foldable t) <= Traversable t where
  traverse :: forall a b m. Applicative m => (a -> m b) -> t a -> m (t b)
  sequence :: forall a m. Applicative m => t (m a) -> m (t a)
```

- `traverse` applies an effectful function (e.g., `a -> m b`, where `m` is an Applicative like `V Errors`) to each element of a structure `t a`, producing a new structure `m (t b)` with combined effects.
- `sequence` flips a structure of effectful computations `t (m a)` into an effectful structure `m (t a)`.

The key idea is that `Traversable` lets you “walk” a structure, apply a function that produces effects (like validation errors), and collect results while preserving the structure’s shape. For example, `traverse` can validate an array of phone numbers, producing either a valid array or a collection of errors.

For arrays with validation (using `V Errors` from `purescript-validation`):

In PSCi, paste (using `:paste` mode, then `Ctrl+D`):

```purescript
import Prelude
import Effect.Console
import Data.Array
import Data.Traversable
import Data.Validation.Semigroup
import Data.List.NonEmpty (singleton)
import Data.String.Regex
import Data.String.Regex.Unsafe
import Data.String.Regex.Flags
newtype PhoneNumber = PhoneNumber String
instance Show PhoneNumber where
  show (PhoneNumber s) = "(PhoneNumber " <> show s <> ")"
phoneNumberRegex = unsafeRegex "^\\d{3}-\\d{3}-\\d{4}$" noFlags
validatePhoneNumber s = if test phoneNumberRegex s
  then pure (PhoneNumber s)
  else invalid (singleton "Invalid phone format")
phones = ["555-555-5555", "555-555-0000"]
logShow $ traverse validatePhoneNumber phones {- - Validates each phone number -}
badPhones = ["555-555-5555", "invalid"]
logShow $ traverse validatePhoneNumber badPhones {- - Collects errors -}
```

Expected output:
```
(valid [(PhoneNumber "555-555-5555"),(PhoneNumber "555-555-0000")])
unit
(invalid (["Invalid phone format"]))
unit
```

For `Maybe` (optional values):

In PSCi, paste (using `:paste` mode, then `Ctrl+D`):

```purescript
import Prelude
import Effect.Console (logShow)
import Data.Maybe (Maybe(..))
import Data.Traversable (traverse)
import Data.List.Types (NonEmptyList(..), List(..))
import Data.Validation.Semigroup (V, invalid)
import Data.NonEmpty (NonEmpty(..), singleton)

nonEmpty field s = 
  if s == "" 
    then invalid (NonEmpty ("Field '" <> field <> "' cannot be empty") Nil)
    else pure s

logShow $ traverse (nonEmpty "Example") Nothing -- No validation for Nothing
logShow $ traverse (nonEmpty "Example") (Just "") -- Validates Just value
logShow $ traverse (nonEmpty "Example") (Just "Testing") -- Successful validation
```

Expected output:
```
(valid Nothing)
unit
(invalid (["Field 'Example' cannot be empty"]))
unit
(valid (Just "Testing"))
unit
```

The compiler ensures `Traversable` instances exist and types match, catching errors at compile time. Traversable laws ensure the structure is preserved and effects are combined predictably, unlike JavaScript’s ad hoc validation.

### Why `Traversable` Is Helpful
1. **Unified Iteration with Effects**: Unlike JS’s array-specific validation loops, `Traversable` works across structures (arrays, `Maybe`, custom types), applying effectful functions uniformly.

2. **Type Safety**: Ensures the validation function and structure are compatible, avoiding JS runtime errors from mismatched types.

3. **Reusable Code**: One `traverse` function handles various structures, reducing custom code compared to JS’s structure-specific loops.

### Practical Example: Validating a List of Names
Validate an array of names using `traverse`:

In PSCi, paste (using `:paste` mode, then `Ctrl+D`):

```purescript
import Prelude
import Effect.Console (logShow)
import Data.Array
import Data.Traversable (traverse)
import Data.List.Types (NonEmptyList(..), List(..))
import Data.Validation.Semigroup (V, invalid)
import Data.NonEmpty (NonEmpty(..), singleton)
nonEmpty field s = if s == ""
  then invalid (NonEmpty ("Field '" <> field <> "' cannot be empty") Nil)
  else pure s
logShow $ traverse (nonEmpty "Name") ["Phillip", "Freeman"] {- - Validates each name -}
logShow $ traverse (nonEmpty "Name") ["Phillip", ""] {- - Collects errors -}
```

Expected output:
```
(valid ["Phillip","Freeman"])
unit
(invalid (["Field 'Name' cannot be empty"]))
unit
```

This generalizes validation across structures, similar to JavaScript’s array validation but safer and more flexible.

### REPL Exercises
Try these in a fresh PSCi session (run `:clear` first). Copy and paste each block using `:paste` mode (type `:paste`, paste the code, then press `Ctrl+D` to execute):

1. **Traverse with Array**:
   ```purescript
   import Prelude
   import Effect.Console (logShow)
   import Data.Array
   import Data.Traversable (traverse)
   import Data.List.Types (NonEmptyList(..), List(..))
   import Data.Validation.Semigroup (V, invalid)
   import Data.NonEmpty (NonEmpty(..), singleton)
   nonEmpty field s = if s == ""
     then invalid (NonEmpty ("Field '" <> field <> "' cannot be empty") Nil)
     else pure s
   logShow $ traverse (nonEmpty "Name") ["Phillip", "Freeman"] {- - Validate names in array -}
   logShow $ traverse (nonEmpty "Name") ["Phillip", ""] {- - Collect errors -}
   ```
   Expected output:
   ```
   (valid ["Phillip","Freeman"])
   unit
   (invalid (["Field 'Name' cannot be empty"]))
   unit
   ```

2. **Traverse with Maybe**:
   ```purescript
   import Prelude
   import Effect.Console (logShow)
   import Data.Maybe (Maybe(..))
   import Data.Traversable (traverse)
   import Data.List.Types (NonEmptyList(..), List(..))
   import Data.Validation.Semigroup (V, invalid)
   import Data.NonEmpty (NonEmpty(..), singleton)
   nonEmpty field s = if s == ""
     then invalid (NonEmpty ("Field '" <> field <> "' cannot be empty") Nil)
     else pure s
   logShow $ traverse (nonEmpty "Example") Nothing {- - No validation for Nothing -}
   logShow $ traverse (nonEmpty "Example") (Just "") {- - Validate Just value -}
   logShow $ traverse (nonEmpty "Example") (Just "Testing") {- - Successful validation -}
   ```
   Expected output:
   ```
   (valid Nothing)
   unit
   (invalid (["Field 'Example' cannot be empty"]))
   unit
   (valid (Just "Testing"))
   unit
   ```

### Summary
JavaScript’s array validation loops, like `validatePhones`, are repetitive, array-specific, and risk runtime errors. PureScript’s `Traversable` type class provides a type-safe, reusable way to apply effectful functions (like validation) across structures like arrays, `Maybe`, or custom types, preserving their shape while combining effects. Unlike Applicative functors, which combine independent effectful values (like form fields in `createUserValidated`), Traversables iterate over a single structure’s elements, making them ideal for validating collections. For JavaScript developers, `Traversable` simplifies validating data in diverse structures, catching errors early and reducing custom code.
