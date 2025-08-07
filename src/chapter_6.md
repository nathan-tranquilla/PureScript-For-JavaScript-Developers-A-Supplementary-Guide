
## Show 

### The JavaScript Struggle: Debugging Complex Objects

JavaScript developers often debug by logging objects to the console. Suppose you’re working with a user profile object:

```javascript
const userProfile = {
  name: "Alice",
  settings: { theme: "dark", notifications: true },
  preferences: ["email", "sms"]
};
console.log(userProfile); // { name: "Alice", settings: { theme: "dark", notifications: true }, preferences: ["email", "sms"] }
console.log(userProfile.toString()); // "[object Object]"
console.log(JSON.stringify(userProfile)); // "{\"name\":\"Alice\",\"settings\":{\"theme\":\"dark\",\"notifications\":true},\"preferences\":[\"email\",\"sms\"]}"
```

The default `console.log` is verbose, `toString` is useless, and `JSON.stringify` is better but clunky with quotes and braces. If `settings` lacks a custom `toString` or `toJSON`, or if you add a non-serializable property like a function, things break:

```javascript
userProfile.callback = () => console.log("hi");
console.log(JSON.stringify(userProfile)); // Omits `callback`: "{\"name\":\"Alice\",\"settings\":{\"theme\":\"dark\",\"notifications\":true},\"preferences\":[\"email\",\"sms\"]}"
```

You need to manually define `toJSON` or `toString` for every nested object to get useful output, and there’s no guarantee it’ll work consistently. Debugging complex objects in JavaScript often feels like a chore.

### PureScript’s `Show`: Clean, Safe, and Automatic

PureScript’s `Show` type class makes debugging complex types effortless with type-safe, customizable string conversion. It’s defined as:

```purescript
class Show a where
  show :: a -> String
```

Every type used with `show` must have a `Show` instance, enforced by the compiler. Let’s define `userProfile` in In PSCi:

```purescript
import Effect.Console

newtype UserProfile = UserProfile { name :: String, settings :: Settings, preferences :: Array String }
newtype Settings = Settings { theme :: String, notifications :: Boolean }

instance Show Settings where
  show (Settings { theme, notifications }) = "(Settings " <> show theme <> " " <> show notifications <> ")"

instance Show UserProfile where
  show (UserProfile { name, settings, preferences }) = "(UserProfile " <> show name <> " " <> show settings <> " " <> show preferences <> ")"

logShow (UserProfile { name: "Alice", settings: Settings { theme: "dark", notifications: true }, preferences: ["email", "sms"] })
```

This output is clean, readable, and a valid PureScript expression you can paste back into PSCi to recreate the value. Unlike JavaScript, `Show` ensures consistency and catches errors early.

### Why `Show` Excites JavaScript Developers

1. **No More `[object Object]`**: Unlike `toString`, `Show` guarantees meaningful output for every type, enforced at compile time. Forget a `Show` instance, and the compiler stops you:
    
    ```purescript
    > show (\n -> n + 1)
    -- Error: No type class instance was found for Data.Show.Show (Int -> Int)
    ```
    
2. **Automatic Nesting**: `Show` composes effortlessly. The `UserProfile` instance reuses `Show` for `Settings`, `String`, and `Array String`, saving you from defining string conversions for every nested type, unlike `toJSON` or `toString`.
    
3. **Debugging Bliss**: The output is concise and REPL-friendly, perfect for quick debugging. Compare `(UserProfile "Alice" (Settings "dark" true) ["email","sms"])` to JSON’s verbose `{"name":"Alice","settings":{"theme":"dark","notifications":true},"preferences":["email","sms"]}`.
    

### But What About `JSON.stringify`?

`JSON.stringify` is great for serializing data for APIs or storage—its JSON output is standard and interoperable. However, for debugging, it’s verbose, drops non-serializable data (e.g., functions), and requires manual `toJSON` definitions for custom formats. `Show` is tailored for PureScript’s REPL-driven, type-safe workflow, offering concise, reconstructible output and automatic composition for complex types like `userProfile`. If you’re serializing for an API, use a PureScript JSON library like `purescript-argonaut`; for debugging, `Show` is your friend.

### REPL Exercises

Try these in PSCi to see `Show` in action. Copy and paste each block:

1. **Show Basic Types**:
    
    ```purescript
    import Prelude
    import Effect.Console
    logShow true
    logShow 1.0
    logShow "Hello"
    ```
    
    Expected output:
    
    ```
    true
    unit
    1.0
    unit
    "Hello"
    unit
    ```
    
2. **Show a Nested Type**:
    
    ```purescript
    import Prelude
    import Effect.Console
    newtype Settings = Settings { theme :: String, notifications :: Boolean }
    instance Show Settings where
      show (Settings { theme, notifications }) = "(Settings " <> show theme <> " " <> show notifications <> ")"
    logShow (Settings { theme: "dark", notifications: true })
    ```
    
    Expected output:
    
    ```
    (Settings "dark" true)
    unit
    ```
    
3. **Show a Complex Type**:
    
    ```purescript
    import Prelude
    import Effect.Console
    newtype Settings = Settings { theme :: String, notifications :: Boolean }
    instance Show Settings where
      show (Settings { theme, notifications }) = "(Settings " <> show theme <> " " <> show notifications <> ")"
    newtype UserProfile = UserProfile { name :: String, settings :: Settings }
    instance Show UserProfile where
      show (UserProfile { name, settings }) = "(UserProfile " <> show name <> " " <> show settings <> ")"
    logShow (UserProfile { name: "Alice", settings: Settings { theme: "dark", notifications: true } })
    ```
    
    Expected output:
    
    ```
    (UserProfile "Alice" (Settings "dark" true))
    unit
    ```
    
4. **Test Type Safety**:
    
    ```purescript
    import Prelude
    show (\n -> n + 1)
    ```
    
    Expected output: A compile-time error, showing `Show`’s safety:
    
    ```
    Error: No type class instance was found for Data.Show.Show (Int -> Int)
    ```
    

### Summary

JavaScript’s `toString` and `JSON.stringify` struggle with debugging complex objects like `userProfile`, requiring manual hacks and risking inconsistent output. PureScript’s `Show` type class offers a type-safe, composable, and REPL-friendly way to convert values to strings, making debugging a breeze. For JavaScript developers diving into PureScript, `Show` transforms the frustration of `[object Object]` into clean, reliable output, perfect for interactive development.

## Eq 

### The JavaScript Struggle: Comparing Objects
In JavaScript, comparing objects for equality is a common task, but it’s fraught with pitfalls. The `===` operator checks for strict equality, which works well for primitives like numbers and strings but often fails for complex objects:

```javascript
const user1 = { id: 1, name: "Alice" };
const user2 = { id: 1, name: "Alice" };
console.log(user1 === user2); // false
console.log(user1 == user2); // false
```

Why? JavaScript’s `===` compares object references, not values. Even though `user1` and `user2` look identical, they’re different objects in memory. To compare their contents, you’d need a custom function:

```javascript
function areUsersEqual(user1, user2) {
  return user1.id === user2.id && user1.name === user2.name;
}
console.log(areUsersEqual(user1, user2)); // true
```

This works but is tedious. You must write a new comparison function for every object type, and there’s no built-in way to enforce that objects support equality checks. For nested objects, it gets messier:

```javascript
const profile1 = { user: { id: 1, name: "Alice" }, settings: { theme: "dark" } };
const profile2 = { user: { id: 1, name: "Alice" }, settings: { theme: "dark" } };
console.log(areUsersEqual(profile1, profile2)); // Error: settings not checked
```

You’d need to extend `areUsersEqual` to recursively compare `settings`, and if the object structure changes, you must update the function manually. Libraries like Lodash’s `_.isEqual` can help, but they’re still runtime solutions with no guarantee of correctness for all types.

### PureScript’s `Eq`: Type-Safe Equality
PureScript’s `Eq` type class provides a standardized, type-safe way to compare values for equality. It’s defined as:

```purescript
class Eq a where
  eq :: a -> a -> Boolean
```

The `==` operator is an alias for `eq`, and any type used with `==` must have an `Eq` instance, enforced by the compiler. Let’s define the `profile` example in PureScript:

```purescript
newtype User = User { id :: Int, name :: String }
newtype Profile = Profile { user :: User, settings :: Settings }
newtype Settings = Settings { theme :: String }

instance Eq User where
  eq (User u1) (User u2) = u1.id == u2.id && u1.name == u2.name

instance Eq Settings where
  eq (Settings s1) (Settings s2) = s1.theme == s2.theme

instance Eq Profile where
  eq (Profile p1) (Profile p2) = p1.user == p2.user && p1.settings == p2.settings
```

In PSCi:

```purescript
> import Prelude
> let profile1 = Profile { user: User { id: 1, name: "Alice" }, settings: Settings { theme: "dark" } }
> let profile2 = Profile { user: User { id: 1, name: "Alice" }, settings: Settings { theme: "dark" } }
> profile1 == profile2
true
```

This comparison is clean, and the compiler ensures every type used with `==` has an `Eq` instance. Trying to compare unsupported types fails at compile time:

```purescript
> (\n -> n + 1) == (\n -> n + 1)
-- Error: No type class instance was found for Data.Eq.Eq (Int -> Int)
```

### Why `Eq` Excites JavaScript Developers
1. **No More Reference Equality Traps**: Unlike JavaScript’s `===`, which fails for identical objects, `Eq` lets you define value-based equality for custom types like `Profile`, making comparisons intuitive.

2. **Automatic Composition**: The `Profile` instance reuses `Eq` instances for `User` and `Settings`. You don’t need to write recursive comparison logic manually, unlike JavaScript’s custom functions or `_.isEqual`.

3. **Type-Safe Guarantees**: The compiler enforces that every type used with `==` has an `Eq` instance, catching errors early. In JavaScript, you might not notice a missing or incorrect comparison until runtime.

### But What About Lodash’s `_.isEqual`?
JavaScript developers might use `_.isEqual` for deep equality checks:

```javascript
const _ = require('lodash');
console.log(_.isEqual(profile1, profile2)); // true
```

`_.isEqual` is convenient and handles nested objects, but it’s a runtime solution with no type checking. It works for any object but might hide bugs if objects have non-comparable properties (e.g., functions). `Eq` requires explicit instances, ensuring correctness at compile time, and is tailored for PureScript’s type-driven development, where you want precise control over equality.

### REPL Exercises
Try these in PSCi to explore `Eq`. Copy and paste each block:

1. **Compare Basic Types**:
   ```purescript
   import Prelude
   1 == 2
   "Test" == "Test"
   ```
   Expected output:
   ```
   false
   true
   ```

2. **Compare a Custom Type**:
   ```purescript
   import Prelude
   newtype User = User { id :: Int, name :: String }
   instance Eq User where
     eq (User u1) (User u2) = u1.id == u2.id && u1.name == u2.name
   User { id: 1, name: "Alice" } == User { id: 1, name: "Alice" }
   User { id: 1, name: "Alice" } == User { id: 2, name: "Bob" }
   ```
   Expected output:
   ```
   true
   false
   ```

3. **Compare a Nested Type**:
   ```purescript
   import Prelude
   newtype User = User { id :: Int, name :: String }
   newtype Profile = Profile { user :: User }
   instance Eq User where
     eq (User u1) (User u2) = u1.id == u2.id && u1.name == u2.name
   instance Eq Profile where
     eq (Profile p1) (Profile p2) = p1.user == p2.user
   Profile { user: User { id: 1, name: "Alice" } } == Profile { user: User { id: 1, name: "Alice" } }
   ```
   Expected output:
   ```
   true
   ```

4. **Test Type Safety**:
   ```purescript
   import Prelude
   (\n -> n + 1) == (\n -> n + 1)
   ```
   Expected output: A compile-time error:
   ```
   Error: No type class instance was found for Data.Eq.Eq (Int -> Int)
   ```

### Summary
JavaScript’s `===` and `_.isEqual` struggle with object comparisons, requiring manual functions or runtime checks. PureScript’s `Eq` type class offers a type-safe, composable way to define equality for custom types like `Profile`, catching errors at compile time and simplifying nested comparisons. For JavaScript developers, `Eq` eliminates the guesswork of equality checks, making PureScript’s type-driven approach a game-changer for reliable code.

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
    address: p1.address <> p2.address  // Deep merge for address
  }

instance Monoid Profile where
  mempty = Profile { name: Nothing, address: mempty }

instance Show Profile where
  show (Profile p) = "(Profile " <> show p <> ")"
```

In PSCi:

```purescript
> import Prelude
> import Effect.Console
> import Data.Maybe
> import Control.Alt ((<|>))
> let a = Profile { name: Just "Alice", address: Address { street: Just "123 Main St", city: Nothing, zip: Nothing } }
> let b = Profile { name: Just "Bob", address: Address { street: Nothing, city: Just "New York", zip: Nothing } }
> let c = Profile { name: Just "Charlie", address: Address { street: Nothing, city: Nothing, zip: Just "10001" } }
> logShow $ (a <> b) <> c
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

For asynchronous effects (like JS promises, via `Aff` from `purescript-aff`):

In PSCi (assuming `Aff` is installed):

```purescript
import Aff (Aff, launchAff_)
import Effect.Console
import Effect.Class (liftEffect)
launchAff_ do
  doubled <- map (\x -> x * 2) (pure 42 :: Aff Int)  // pure is like Promise.resolve
  liftEffect $ logShow doubled
```

Expected output:
```
84
unit
```

The compiler ensures `Functor` instances exist, catching errors at compile time. Laws make mapping reliable—e.g., composition preserves order without side effects.

### Why PureScript’s `Functor` Is Helpful
1. **Compile-Time Safety**: In JS, mapping non-mappable types fails at runtime. Functor requires instances, catching errors early.

2. **Consistent Mapping**: Uniform `map` for arrays, `Maybe`, `Tuple`, asynchronous effects (`Aff`), maps— no custom hacks.

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

## 
## Show 

### The JavaScript Struggle: Debugging Complex Objects

JavaScript developers often debug by logging objects to the console. Suppose you’re working with a user profile object:

```javascript
const userProfile = {
  name: "Alice",
  settings: { theme: "dark", notifications: true },
  preferences: ["email", "sms"]
};
console.log(userProfile); // { name: "Alice", settings: { theme: "dark", notifications: true }, preferences: ["email", "sms"] }
console.log(userProfile.toString()); // "[object Object]"
console.log(JSON.stringify(userProfile)); // "{\"name\":\"Alice\",\"settings\":{\"theme\":\"dark\",\"notifications\":true},\"preferences\":[\"email\",\"sms\"]}"
```

The default `console.log` is verbose, `toString` is useless, and `JSON.stringify` is better but clunky with quotes and braces. If `settings` lacks a custom `toString` or `toJSON`, or if you add a non-serializable property like a function, things break:

```javascript
userProfile.callback = () => console.log("hi");
console.log(JSON.stringify(userProfile)); // Omits `callback`: "{\"name\":\"Alice\",\"settings\":{\"theme\":\"dark\",\"notifications\":true},\"preferences\":[\"email\",\"sms\"]}"
```

You need to manually define `toJSON` or `toString` for every nested object to get useful output, and there’s no guarantee it’ll work consistently. Debugging complex objects in JavaScript often feels like a chore.

### PureScript’s `Show`: Clean, Safe, and Automatic

PureScript’s `Show` type class makes debugging complex types effortless with type-safe, customizable string conversion. It’s defined as:

```purescript
class Show a where
  show :: a -> String
```

Every type used with `show` must have a `Show` instance, enforced by the compiler. Let’s define `userProfile` in In PSCi:

```purescript
import Effect.Console

newtype UserProfile = UserProfile { name :: String, settings :: Settings, preferences :: Array String }
newtype Settings = Settings { theme :: String, notifications :: Boolean }

instance Show Settings where
  show (Settings { theme, notifications }) = "(Settings " <> show theme <> " " <> show notifications <> ")"

instance Show UserProfile where
  show (UserProfile { name, settings, preferences }) = "(UserProfile " <> show name <> " " <> show settings <> " " <> show preferences <> ")"

logShow (UserProfile { name: "Alice", settings: Settings { theme: "dark", notifications: true }, preferences: ["email", "sms"] })
```

This output is clean, readable, and a valid PureScript expression you can paste back into PSCi to recreate the value. Unlike JavaScript, `Show` ensures consistency and catches errors early.

### Why `Show` Excites JavaScript Developers

1. **No More `[object Object]`**: Unlike `toString`, `Show` guarantees meaningful output for every type, enforced at compile time. Forget a `Show` instance, and the compiler stops you:
    
    ```purescript
    > show (\n -> n + 1)
    -- Error: No type class instance was found for Data.Show.Show (Int -> Int)
    ```
    
2. **Automatic Nesting**: `Show` composes effortlessly. The `UserProfile` instance reuses `Show` for `Settings`, `String`, and `Array String`, saving you from defining string conversions for every nested type, unlike `toJSON` or `toString`.
    
3. **Debugging Bliss**: The output is concise and REPL-friendly, perfect for quick debugging. Compare `(UserProfile "Alice" (Settings "dark" true) ["email","sms"])` to JSON’s verbose `{"name":"Alice","settings":{"theme":"dark","notifications":true},"preferences":["email","sms"]}`.
    

### But What About `JSON.stringify`?

`JSON.stringify` is great for serializing data for APIs or storage—its JSON output is standard and interoperable. However, for debugging, it’s verbose, drops non-serializable data (e.g., functions), and requires manual `toJSON` definitions for custom formats. `Show` is tailored for PureScript’s REPL-driven, type-safe workflow, offering concise, reconstructible output and automatic composition for complex types like `userProfile`. If you’re serializing for an API, use a PureScript JSON library like `purescript-argonaut`; for debugging, `Show` is your friend.

### REPL Exercises

Try these in PSCi to see `Show` in action. Copy and paste each block:

1. **Show Basic Types**:
    
    ```purescript
    import Prelude
    import Effect.Console
    logShow true
    logShow 1.0
    logShow "Hello"
    ```
    
    Expected output:
    
    ```
    true
    unit
    1.0
    unit
    "Hello"
    unit
    ```
    
2. **Show a Nested Type**:
    
    ```purescript
    import Prelude
    import Effect.Console
    newtype Settings = Settings { theme :: String, notifications :: Boolean }
    instance Show Settings where
      show (Settings { theme, notifications }) = "(Settings " <> show theme <> " " <> show notifications <> ")"
    logShow (Settings { theme: "dark", notifications: true })
    ```
    
    Expected output:
    
    ```
    (Settings "dark" true)
    unit
    ```
    
3. **Show a Complex Type**:
    
    ```purescript
    import Prelude
    import Effect.Console
    newtype Settings = Settings { theme :: String, notifications :: Boolean }
    instance Show Settings where
      show (Settings { theme, notifications }) = "(Settings " <> show theme <> " " <> show notifications <> ")"
    newtype UserProfile = UserProfile { name :: String, settings :: Settings }
    instance Show UserProfile where
      show (UserProfile { name, settings }) = "(UserProfile " <> show name <> " " <> show settings <> ")"
    logShow (UserProfile { name: "Alice", settings: Settings { theme: "dark", notifications: true } })
    ```
    
    Expected output:
    
    ```
    (UserProfile "Alice" (Settings "dark" true))
    unit
    ```
    
4. **Test Type Safety**:
    
    ```purescript
    import Prelude
    show (\n -> n + 1)
    ```
    
    Expected output: A compile-time error, showing `Show`’s safety:
    
    ```
    Error: No type class instance was found for Data.Show.Show (Int -> Int)
    ```
    

### Summary

JavaScript’s `toString` and `JSON.stringify` struggle with debugging complex objects like `userProfile`, requiring manual hacks and risking inconsistent output. PureScript’s `Show` type class offers a type-safe, composable, and REPL-friendly way to convert values to strings, making debugging a breeze. For JavaScript developers diving into PureScript, `Show` transforms the frustration of `[object Object]` into clean, reliable output, perfect for interactive development.

## Eq 

### The JavaScript Struggle: Comparing Objects
In JavaScript, comparing objects for equality is a common task, but it’s fraught with pitfalls. The `===` operator checks for strict equality, which works well for primitives like numbers and strings but often fails for complex objects:

```javascript
const user1 = { id: 1, name: "Alice" };
const user2 = { id: 1, name: "Alice" };
console.log(user1 === user2); // false
console.log(user1 == user2); // false
```

Why? JavaScript’s `===` compares object references, not values. Even though `user1` and `user2` look identical, they’re different objects in memory. To compare their contents, you’d need a custom function:

```javascript
function areUsersEqual(user1, user2) {
  return user1.id === user2.id && user1.name === user2.name;
}
console.log(areUsersEqual(user1, user2)); // true
```

This works but is tedious. You must write a new comparison function for every object type, and there’s no built-in way to enforce that objects support equality checks. For nested objects, it gets messier:

```javascript
const profile1 = { user: { id: 1, name: "Alice" }, settings: { theme: "dark" } };
const profile2 = { user: { id: 1, name: "Alice" }, settings: { theme: "dark" } };
console.log(areUsersEqual(profile1, profile2)); // Error: settings not checked
```

You’d need to extend `areUsersEqual` to recursively compare `settings`, and if the object structure changes, you must update the function manually. Libraries like Lodash’s `_.isEqual` can help, but they’re still runtime solutions with no guarantee of correctness for all types.

### PureScript’s `Eq`: Type-Safe Equality
PureScript’s `Eq` type class provides a standardized, type-safe way to compare values for equality. It’s defined as:

```purescript
class Eq a where
  eq :: a -> a -> Boolean
```

The `==` operator is an alias for `eq`, and any type used with `==` must have an `Eq` instance, enforced by the compiler. Let’s define the `profile` example in PureScript:

```purescript
newtype User = User { id :: Int, name :: String }
newtype Profile = Profile { user :: User, settings :: Settings }
newtype Settings = Settings { theme :: String }

instance Eq User where
  eq (User u1) (User u2) = u1.id == u2.id && u1.name == u2.name

instance Eq Settings where
  eq (Settings s1) (Settings s2) = s1.theme == s2.theme

instance Eq Profile where
  eq (Profile p1) (Profile p2) = p1.user == p2.user && p1.settings == p2.settings
```

In PSCi:

```purescript
> import Prelude
> let profile1 = Profile { user: User { id: 1, name: "Alice" }, settings: Settings { theme: "dark" } }
> let profile2 = Profile { user: User { id: 1, name: "Alice" }, settings: Settings { theme: "dark" } }
> profile1 == profile2
true
```

This comparison is clean, and the compiler ensures every type used with `==` has an `Eq` instance. Trying to compare unsupported types fails at compile time:

```purescript
> (\n -> n + 1) == (\n -> n + 1)
-- Error: No type class instance was found for Data.Eq.Eq (Int -> Int)
```

### Why `Eq` Excites JavaScript Developers
1. **No More Reference Equality Traps**: Unlike JavaScript’s `===`, which fails for identical objects, `Eq` lets you define value-based equality for custom types like `Profile`, making comparisons intuitive.

2. **Automatic Composition**: The `Profile` instance reuses `Eq` instances for `User` and `Settings`. You don’t need to write recursive comparison logic manually, unlike JavaScript’s custom functions or `_.isEqual`.

3. **Type-Safe Guarantees**: The compiler enforces that every type used with `==` has an `Eq` instance, catching errors early. In JavaScript, you might not notice a missing or incorrect comparison until runtime.

### But What About Lodash’s `_.isEqual`?
JavaScript developers might use `_.isEqual` for deep equality checks:

```javascript
const _ = require('lodash');
console.log(_.isEqual(profile1, profile2)); // true
```

`_.isEqual` is convenient and handles nested objects, but it’s a runtime solution with no type checking. It works for any object but might hide bugs if objects have non-comparable properties (e.g., functions). `Eq` requires explicit instances, ensuring correctness at compile time, and is tailored for PureScript’s type-driven development, where you want precise control over equality.

### REPL Exercises
Try these in PSCi to explore `Eq`. Copy and paste each block:

1. **Compare Basic Types**:
   ```purescript
   import Prelude
   1 == 2
   "Test" == "Test"
   ```
   Expected output:
   ```
   false
   true
   ```

2. **Compare a Custom Type**:
   ```purescript
   import Prelude
   newtype User = User { id :: Int, name :: String }
   instance Eq User where
     eq (User u1) (User u2) = u1.id == u2.id && u1.name == u2.name
   User { id: 1, name: "Alice" } == User { id: 1, name: "Alice" }
   User { id: 1, name: "Alice" } == User { id: 2, name: "Bob" }
   ```
   Expected output:
   ```
   true
   false
   ```

3. **Compare a Nested Type**:
   ```purescript
   import Prelude
   newtype User = User { id :: Int, name :: String }
   newtype Profile = Profile { user :: User }
   instance Eq User where
     eq (User u1) (User u2) = u1.id == u2.id && u1.name == u2.name
   instance Eq Profile where
     eq (Profile p1) (Profile p2) = p1.user == p2.user
   Profile { user: User { id: 1, name: "Alice" } } == Profile { user: User { id: 1, name: "Alice" } }
   ```
   Expected output:
   ```
   true
   ```

4. **Test Type Safety**:
   ```purescript
   import Prelude
   (\n -> n + 1) == (\n -> n + 1)
   ```
   Expected output: A compile-time error:
   ```
   Error: No type class instance was found for Data.Eq.Eq (Int -> Int)
   ```

### Summary
JavaScript’s `===` and `_.isEqual` struggle with object comparisons, requiring manual functions or runtime checks. PureScript’s `Eq` type class offers a type-safe, composable way to define equality for custom types like `Profile`, catching errors at compile time and simplifying nested comparisons. For JavaScript developers, `Eq` eliminates the guesswork of equality checks, making PureScript’s type-driven approach a game-changer for reliable code.

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
    address: p1.address <> p2.address  // Deep merge for address
  }

instance Monoid Profile where
  mempty = Profile { name: Nothing, address: mempty }

instance Show Profile where
  show (Profile p) = "(Profile " <> show p <> ")"
```

In PSCi:

```purescript
> import Prelude
> import Effect.Console
> import Data.Maybe
> import Control.Alt ((<|>))
> let a = Profile { name: Just "Alice", address: Address { street: Just "123 Main St", city: Nothing, zip: Nothing } }
> let b = Profile { name: Just "Bob", address: Address { street: Nothing, city: Just "New York", zip: Nothing } }
> let c = Profile { name: Just "Charlie", address: Address { street: Nothing, city: Nothing, zip: Just "10001" } }
> logShow $ (a <> b) <> c
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

For asynchronous effects (like JS promises, via `Aff` from `purescript-aff`):

In PSCi (assuming `Aff` is installed):

```purescript
import Aff (Aff, launchAff_)
import Effect.Console
import Effect.Class (liftEffect)
launchAff_ do
  doubled <- map (\x -> x * 2) (pure 42 :: Aff Int)  // pure is like Promise.resolve
  liftEffect $ logShow doubled
```

Expected output:
```
84
unit
```

The compiler ensures `Functor` instances exist, catching errors at compile time. Laws make mapping reliable—e.g., composition preserves order without side effects.

### Why PureScript’s `Functor` Is Helpful
1. **Compile-Time Safety**: In JS, mapping non-mappable types fails at runtime. Functor requires instances, catching errors early.

2. **Consistent Mapping**: Uniform `map` for arrays, `Maybe`, `Tuple`, asynchronous effects (`Aff`), maps— no custom hacks.

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
