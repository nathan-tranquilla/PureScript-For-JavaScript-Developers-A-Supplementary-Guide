## Ord

### The JavaScript Struggle: Sorting Objects

JavaScript developers often need to sort or compare objects, like user profiles, by specific fields. The comparison operators (`<`, `>`) work for numbers and strings but fail for objects:

```javascript
const user1 = { id: 1, name: "Alice" };
const user2 = { id: 2, name: "Bob" };
console.log(user1 < user2); // false (coerces to strings: "[object Object]")
```

JavaScript’s `<` operator isn’t helpful here, as it compares object references or stringified versions. To sort an array of users by `id`, you write a custom function for `Array.sort`:

```javascript
const users = [
  { id: 2, name: "Bob" },
  { id: 1, name: "Alice" }
];
users.sort((a, b) => a.id - b.id);
console.log(users); // [{ id: 1, name: "Alice" }, { id: 2, name: "Bob" }]
```

This works but requires a new comparison function for every object type or sorting criterion. For nested objects, like a `profile` with a `user`, it’s even more tedious:

```javascript
const profile1 = { user: { id: 2, name: "Bob" }, settings: { theme: "dark" } };
const profile2 = { user: { id: 1, name: "Alice" }, settings: { theme: "light" } };
profiles.sort((a, b) => a.user.id - b.user.id);
```

You must manually navigate nested fields, and there’s no guarantee the comparison is defined or consistent. Libraries like Lodash’s `_.orderBy` help, but they’re still runtime solutions without type safety.

### PureScript’s `Ord`: Type-Safe Ordering

PureScript’s `Ord` type class provides a standardized, type-safe way to compare values for ordering, building on the `Eq` type class for equality. It’s defined as:

```purescript
data Ordering = LT | EQ | GT

class Eq a <= Ord a where
  compare :: a -> a -> Ordering
```

The `compare` function returns `LT` (less than), `EQ` (equal), or `GT` (greater than), and operators like `<`, `>`, `<=`, and `>=` are built on `compare`. The `Eq` superclass ensures that `compare a b == EQ` aligns with `a == b`. Here’s the `profile` example in PureScript:

```purescript
newtype AppUser = AppUser { id :: Int, name :: String }
newtype Settings = Settings { theme :: String }
newtype Profile = Profile { user :: AppUser, settings :: Settings }

derive instance Eq AppUser
derive instance Eq Settings
derive instance Eq Profile

instance Ord AppUser where
  compare (AppUser u1) (AppUser u2) = compare u1.id u2.id

instance Ord Settings where
  compare (Settings s1) (Settings s2) = compare s1.theme s2.theme

instance Ord Profile where
  compare (Profile p1) (Profile p2) = compare p1.user p2.user
```

In PSCi:

```purescript
> import Prelude
> let profile1 = Profile { user: UniqueAppUser { id: 2, name: "Bob" }, settings: Settings { theme: "dark" } }
> let profile2 = Profile { user: UniqueAppUser { id: 1, name: "Alice" }, settings: Settings { theme: "light" } }
> compare profile1 profile2
GT
> profile1 < profile2
false
> import Data.Array
> import Effect.Console
> logShow $ sort [profile1, profile2]
[(Profile (UniqueAppUser 1 "Alice") (Settings "light")), (Profile (UniqueAppUser 2 "Bob") (Settings "dark"))]
unit
```

The `sort` function uses `Ord` to order by `user.id`. The compiler ensures every type used with `compare` or `<` has an `Ord` instance, preventing errors:

```purescript
> compare (\n -> n + 1) (\n -> n + 1)
-- Error: No type class instance was found for Data.Ord.Ord (Int -> Int)
```

### Why `Ord` Excites JavaScript Developers

1. **No Custom Sort Functions**: Unlike JavaScript’s `sort` with one-off comparison functions, `Ord` defines a reusable `compare` function per type, making ordering consistent.
    
2. **Composes Effortlessly**: The `Profile` instance reuses `UniqueAppUser`’s `Ord` instance, which reuses `Int`’s. No need to manually dig into nested fields like in JavaScript.
    
3. **Type-Safe Sorting**: Compile-time checks ensure `Ord` instances exist, eliminating runtime errors from undefined comparisons.
    

### But What About Lodash’s `_.orderBy`?

Lodash’s `_.orderBy` simplifies sorting:

```javascript
const _ = require('lodash');
console.log(_.orderBy([profile1, profile2], ['user.id'], ['asc']));
// [{ user: { id: 1, name: "Alice" }, settings: { theme: "light" } }, ...]
```

It’s flexible but runtime-based. Misspelling `user.id` or using an invalid property causes errors at runtime. `Ord` ensures correctness at compile time and integrates with PureScript’s `sort`, streamlining type-safe development.

### REPL Exercises

Try these in a fresh PSCi session (run `:reset` first) to explore `Ord`. Copy and paste each block:

1. **Compare Basic Types**:
    
    ```purescript
    import Prelude
    compare 1 2
    compare "A" "Z"
    ```
    
    Expected output:
    
    ```
    LT
    LT
    ```
    
2. **Compare a Custom Type**:
    
    ```purescript
    import Prelude
    newtype UniqueAppUser = UniqueAppUser { id :: Int, name :: String }
    derive instance Eq UniqueAppUser
    instance Show UniqueAppUser where
      show (UniqueAppUser { id, name }) = "(UniqueAppUser " <> show id <> " " <> show name <> ")"
    instance Ord UniqueAppUser where
      compare (UniqueAppUser u1) (UniqueAppUser u2) = compare u1.id u2.id
    compare (UniqueAppUser { id: 1, name: "Alice" }) (UniqueAppUser { id: 2, name: "Bob" })
    (UniqueAppUser { id: 1, name: "Alice" }) < (UniqueAppUser { id: 2, name: "Bob" })
    ```
    
    Expected output:
    
    ```
    LT
    true
    ```
    
3. **Sort an Array**:
    
    ```purescript
    import Prelude
    import Data.Array
    import Effect.Console
    newtype UniqueAppUser = UniqueAppUser { id :: Int, name :: String }
    derive instance Eq UniqueAppUser
    instance Show UniqueAppUser where
      show (UniqueAppUser { id, name }) = "(UniqueAppUser " <> show id <> " " <> show name <> ")"
    instance Ord UniqueAppUser where
      compare (UniqueAppUser u1) (UniqueAppUser u2) = compare u1.id u2.id
    logShow $ sort [UniqueAppUser { id: 2, name: "Bob" }, UniqueAppUser { id: 1, name: "Alice" }]
    ```
    
    Expected output:
    
    ```
    [(UniqueAppUser 1 "Alice"), (UniqueAppUser 2 "Bob")]
    unit
    ```
    
4. **Test Type Safety**:
    
    ```purescript
    import Prelude
    compare (\n -> n + 1) (\n -> n + 1)
    ```
    
    Expected output:
    
    ```
    Error: No type class instance was found for Data.Ord.Ord (Int -> Int)
    ```
    

### Summary

JavaScript’s sorting requires custom, error-prone comparison functions for objects like `profile`. PureScript’s `Ord` type class offers a type-safe, reusable way to define ordering, with automatic composition for nested types and compile-time guarantees. For JavaScript developers, `Ord` eliminates the hassle of ad hoc sorting, making PureScript’s type-driven approach a powerful tool for reliable code.

