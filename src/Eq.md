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
import Prelude

newtype User = User { id :: Int, name :: String }
newtype Profile = Profile { user :: User, settings :: Settings }
newtype Settings = Settings { theme :: String }

instance Eq User where
  eq (User u1) (User u2) = u1.id == u2.id && u1.name == u2.name

instance Eq Settings where
  eq (Settings s1) (Settings s2) = s1.theme == s2.theme

instance Eq Profile where
  eq (Profile p1) (Profile p2) = p1.user == p2.user && p1.settings == p2.settings

profile1 = Profile { user: User { id: 1, name: "Alice" }, settings: Settings { theme: "dark" } }
profile2 = Profile { user: User { id: 1, name: "Alice" }, settings: Settings { theme: "dark" } }
profile1 == profile2
```

Expected Output:
```
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

