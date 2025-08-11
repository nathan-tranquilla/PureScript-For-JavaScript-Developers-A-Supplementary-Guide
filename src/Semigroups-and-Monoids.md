## Semigroups and Monoids

### The JavaScript Struggle: Combining Values
In JavaScript, combining values like strings or arrays is common, using `+` for strings or `concat` for arrays:

```javascript
console.log("Hello" + " " + "World"); // "Hello World"
console.log([1, 2, 3].concat([4, 5]); // [1, 2, 3, 4, 5]
```

But for custom types, like merging user settings objects, there's no standard way:

```javascript
const settings1 = { theme: "dark", notifications: true };
const settings2 = { theme: "light", alerts: true };
const merged = { ...settings1, ...settings2 }; // { theme: "light", notifications: true, alerts: true }
console.log(merged);
```

This works but is ad hoc. You must define custom merge functions for each type, and there's no guarantee of associativity (e.g., `(a + b) + c == a + (b + c)`). Associativity means the order of grouping combinations doesn't matter, which is crucial for predictable behavior in larger operations like folding arrays. For example, consider merging user profiles with nested `address` objects:

```javascript
const profileA = { name: "Alice", address: { street: "123 Main St", city: undefined, zip: undefined } };
const profileB = { name: "Bob", address: { street: undefined, city: "New York", zip: undefined } };
const profileC = { name: "Charlie", address: { street: undefined, city: undefined, zip: "10001" } };

// Merging A and B first, then with C
const result1 = { ...{ ...profileA, ...profileB }, ...profileC };
// Result: { name: "Charlie", address: { street: undefined, city: undefined, zip: "10001" } } – street and city are lost

// Merging B and C first, then with A
const result2 = { ...{ ...profileB, ...profileC }, ...profileA };
// Result: { name: "Alice", address: { street: "123 Main St", city: undefined, zip: undefined } } – city and zip are lost
```

The grouping affects which nested `address` properties survive, breaking associativity—the results aren't the same. This can lead to subtle bugs in larger merges or folds, where the order of operations unexpectedly changes the result. For empty values (like empty object `{}`), it's often clear what the "identity" is for simple spreads (it combines with any object to return the object unchanged), but for custom merges like this, `{}` might not be an identity if merging adds logic.

For more complex combinations, like folding an array of objects, you need custom reduce functions:

```javascript
const allProfiles = [profileA, profileB, profileC];
const mergedAll = allProfiles.reduce((acc, p) => ({ ...acc, ...p }), {});
console.log(mergedAll); // { name: "Charlie", address: { street: undefined, city: undefined, zip: "10001" } } – depends on order, nested fields overwritten
```

This is flexible but error-prone, with no type safety for the combination operation or empty value.

### PureScript’s `Semigroup` and `Monoid`: Type-Safe Combining
PureScript’s `Semigroup` type class defines an `append` operation (alias `<>`) for combining values, with the law that it's associative. `Monoid` extends it with `mempty`, an identity value. They are defined as:

```purescript
class Semigroup a where
  append :: a -> a -> a

class Semigroup m <= Monoid m where
  mempty :: m
```

Strings and arrays have instances, but you can define for custom types like `Profile` with nested `Address`. To demonstrate associativity, let's define a `Profile` type with deep merging for the nested `address` fields:

```purescript
import Prelude
import Effect.Console
import Data.Maybe
import Control.Alt ((<|>))

newtype Address = Address { street :: Maybe String, city :: Maybe String, zip :: Maybe String }

instance Semigroup Address where
  append (Address a1) (Address a2) = Address {
    street: a1.street <|> a2.street,
    city: a1.city <|> a2.city,
    zip: a1.zip <|> a2.zip
  }

instance Monoid Address where
  mempty = Address { street: Nothing, city: Nothing, zip: Nothing }

instance Show Address where
  show (Address a) = "(Address " <> show a <> ")"

newtype Profile = Profile { name :: Maybe String, address :: Address }

instance Semigroup Profile where
  append (Profile p1) (Profile p2) = Profile {
    name: p1.name <|> p2.name,
    address: p1.address <> p2.address
  }

instance Monoid Profile where
  mempty = Profile { name: Nothing, address: mempty }

instance Show Profile where
  show (Profile p) = "(Profile " <> show p <> ")"

a = Profile { name: Just "Alice", address: Address { street: Just "123 Main St", city: Nothing, zip: Nothing } }
b = Profile { name: Just "Bob", address: Address { street: Nothing, city: Just "New York", zip: Nothing } }
c = Profile { name: Just "Charlie", address: Address { street: Nothing, city: Nothing, zip: Just "10001" } }

logShow $ (a <> b) <> c
logShow $ a <> (b <> c)

```

Expected output:
```
(Profile { name: (Just "Alice"), address: (Address { street: (Just "123 Main St"), city: (Just "New York"), zip: (Just "10001") }) })
unit
> logShow $ a <> (b <> c)
(Profile { name: (Just "Alice"), address: (Address { street: (Just "123 Main St"), city: (Just "New York"), zip: (Just "10001") }) })
unit
```

The results are the same regardless of grouping, adhering to the associativity law. This encourages designs where combinations are predictable and deep merging is explicit, avoiding the pitfalls in JavaScript's custom merges. The compiler ensures all types used with `<>` have a `Semigroup` instance, and the laws (associativity, identity) are enforced by convention.

### Why `Semigroup` and `Monoid` Excite JavaScript Developers
1. **Standardized Combining**: Unlike JavaScript’s ad hoc merges, `Semigroup` provides a consistent `<>` , reusable for custom types like `Profile`.

2. **Identity Value**: `Monoid`’s `mempty` gives a clear empty value, like `{}` in JavaScript for simple merges, but type-safe and guaranteed.

3. **Type-Safe Folds**: Combine with `Foldable` (which we'll cover later) for safe, efficient folding, with compile-time checks— no runtime surprises from undefined merges.

### But What About JavaScript’s `+` or `Object.assign`?
JavaScript’s `+` works for strings/arrays, and `Object.assign` for objects:

```javascript
const merged = Object.assign({}, settings1, settings2);
```

It's flexible but runtime-only, with no associativity guarantees or type safety. `Semigroup` and `Monoid` provide compile-time safety and laws for reliable combinations.

### REPL Exercises
Try these in a fresh PSCi session (run `:clear` first). Copy and paste each block:

1. **Combine Strings and Arrays**:
   ```purescript
   import Prelude
   import Effect.Console
   logShow $ "Hello" <> " " <> "World"
   logShow $ [1, 2, 3] <> [4, 5]
   ```
   Expected output:
   ```
   "Hello World"
   unit
   [1,2,3,4,5]
   unit
   ```

2. **Use Monoid Empty**:
   ```purescript
   import Prelude
   import Effect.Console
   import Data.Monoid
   logShow $ mempty <> "Hello" <> mempty
   logShow $ mempty <> [1, 2] <> mempty
   ```
   Expected output:
   ```
   "Hello"
   unit
   [1,2]
   unit
   ```

3. **Custom Type Combining**:
   ```purescript
   import Prelude
   import Effect.Console
   import Data.Monoid
   import Data.Maybe
   import Control.Alt ((<|>))
   newtype Address = Address { street :: Maybe String, city :: Maybe String, zip :: Maybe String }
   instance Semigroup Address where
     append (Address a1) (Address a2) = Address {
       street: a1.street <|> a2.street,
       city: a1.city <|> a2.city,
       zip: a1.zip <|> a2.zip
     }
   instance Monoid Address where
     mempty = Address { street: Nothing, city: Nothing, zip: Nothing }
   instance Show Address where
     show (Address a) = "(Address " <> show a <> ")"
   newtype Profile = Profile { name :: Maybe String, address :: Address }
   instance Semigroup Profile where
     append (Profile p1) (Profile p2) = Profile {
       name: p1.name <|> p2.name,
       address: p1.address <> p2.address
     }
   instance Monoid Profile where
     mempty = Profile { name: Nothing, address: mempty }
   instance Show Profile where
     show (Profile p) = "(Profile " <> show p <> ")"
   logShow $ Profile { name: Just "Alice", address: Address { street: Just "123 Main St", city: Nothing, zip: Nothing } } <> Profile { name: Nothing, address: Address { street: Nothing, city: Just "New York", zip: Just "10001" } }
   ```
   Expected output:
   ```
   (Profile { name: (Just "Alice"), address: (Address { street: (Just "123 Main St"), city: (Just "New York"), zip: (Just "10001") }) })
   unit
   ```

### Summary
JavaScript’s combining operations like `+` and `Object.assign` are ad hoc and lack safety for custom types like `Profile`. PureScript’s `Semigroup` and `Monoid` type classes offer a type-safe, associative way to combine values, with an identity for folding, making code more reliable and reusable. For JavaScript developers, these classes bring structure to combinations, reducing bugs in data aggregation.


