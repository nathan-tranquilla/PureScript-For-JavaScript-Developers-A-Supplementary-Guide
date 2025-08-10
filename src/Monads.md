## Monads

### JavaScript Context: Chaining Asynchronous Operations
In JavaScript, you often chain asynchronous operations, like fetching user data and then retrieving related information, using Promises to handle the sequence. For example, suppose you want to fetch a user’s profile, then their address, and finally their city:

```javascript
function fetchUser(userId) {
  return Promise.resolve({ id: userId, profileId: `profile-${userId}` });
}
function fetchProfile(profileId) {
  return Promise.resolve({ profileId, addressId: `address-${profileId}` });
}
function fetchCity(addressId) {
  return Promise.resolve({ addressId, city: "Faketown" });
}
fetchUser(123)
  .then(user => fetchProfile(user.profileId))
  .then(profile => fetchCity(profile.addressId))
  .then(address => ({ city: address.city }))
  .then(console.log); // { city: "Faketown" }
fetchUser(null)
  .then(user => fetchProfile(user.profileId))
  .then(profile => fetchCity(profile.addressId))
  .then(address => ({ city: address.city }))
  .catch(error => console.log({ error: error.message })); // { error: "Cannot read properties of null" }
```

This pattern—chaining operations where each step depends on the previous result—is common in JavaScript for handling async tasks like API calls or database queries. You use `.then` to sequence the operations, passing the result of one to the next. However, it’s error-prone: if any step fails (e.g., `user` is `null`), you get runtime errors, and you need `.catch` to handle failures. Extending this to other contexts (like optional values or arrays) requires custom code, and there’s no type safety to ensure each step’s output matches the next step’s input.

### PureScript’s `Monad`: Structured Sequential Computations
PureScript’s `Monad` type class formalizes this pattern of chaining dependent computations, allowing you to sequence operations where each step uses the result of the previous one, with type safety. It builds on `Applicative` and `Apply` (from Chapter 7) and is defined as:

```purescript
class Apply m <= Bind m where
  bind :: forall a b. m a -> (a -> m b) -> m b

class (Applicative m, Bind m) <= Monad m
```

- `bind` (alias `>>=`) takes a computation in a context `m a` (e.g., `Maybe a`, `Aff a`) and a function that produces the next computation `(a -> m b)`, chaining them to produce a final `m b`.
- `pure` (from `Applicative`) wraps a value into the context, and `<*>` (from `Apply`) applies functions within the context.

The `do` notation simplifies writing these chained computations, making them look sequential like JavaScript’s `.then` chains. For example, with `Maybe` (optional values):

In PSCi, paste (using `:paste` mode, then `Ctrl+D`):

```purescript
import Prelude
import Effect.Console
import Data.Map as Map
import Data.Maybe
import Data.Tuple

lookupUser userId users = Map.lookup userId users
lookupProfile profileId profiles = Map.lookup profileId profiles
lookupCity addressId addresses = Map.lookup addressId addresses

userCity userId users profiles addresses = do
  user <- lookupUser userId users {- - Look up user by ID -}
  profile <- lookupProfile user.profileId profiles {- - Look up profile by profileId -}
  address <- lookupCity profile.addressId addresses {- - Look up address by addressId -}
  pure address.city {- - Return the city -}

users = Map.fromFoldable [ Tuple "user123" { userId: "user123", profileId: "profile-123" } ]
profiles = Map.fromFoldable [ Tuple "profile-123" { profileId: "profile-123", addressId: "address-123" } ]
addresses = Map.fromFoldable [ Tuple "address-123" { addressId: "address-123", city: "Faketown" } ]

logShow $ userCity "user123" users profiles addresses 
logShow $ userCity "wrong" users profiles addresses 
```

Expected output:
```
(Just "Faketown")
unit
Nothing
unit
```

This mirrors the JavaScript Promise chain, but `Maybe` handles missing values safely. The `do` notation desugars to `>>=`:

```purescript
userCity userId users profiles addresses =
  lookupUser userId users >>= \user ->
    lookupProfile user.profileId profiles >>= \profile ->
      lookupCity profile.addressId addresses >>= \address ->
        pure address.city
```

For arrays (non-deterministic computations):

In PSCi, paste (using `:paste` mode, then `Ctrl+D`):

```purescript
import Prelude
import Effect.Console
import Data.Array
import Control.Plus
countThrows n = do
  x <- [1, 2, 3, 4, 5, 6]
  y <- [1, 2, 3, 4, 5, 6]
  if x + y == n
    then pure [x, y]
    else empty 
logShow $ countThrows 10 {- - Finds all dice throws summing to 10 -}

```

Expected output:
```
[[4,6],[5,5],[6,4]]
unit
```

The compiler ensures `Monad` instances exist and types match, catching errors at compile time. Monad laws (identity, associativity) ensure predictable behavior, unlike JavaScript’s error-prone Promise chains.

### Why `Monad` Is Helpful
1. **Structured Sequential Computations**: Unlike JS’s `.then` chains, `Monad` sequences dependent computations across contexts (like optionals, arrays, async) uniformly using `do` notation.
2. **Type Safety**: Ensures each step’s output matches the next step’s input, avoiding JS runtime errors from nulls or type mismatches.
3. **Reusable Code**: One `bind` operator (or `do` notation) works across different structures, reducing custom code compared to JS’s context-specific chaining.

### Practical Example: Navigating a deeply-nested map

In PSCi, paste (using `:paste` mode, then `Ctrl+D`):

```purescript
import Prelude
import Effect.Console
import Data.Map as Map
import Data.Maybe
import Data.Tuple

lookupUser userId users = Map.lookup userId users
lookupProfile profileId profiles = Map.lookup profileId profiles
lookupCity addressId addresses = Map.lookup addressId addresses

userCity userId users profiles addresses = do
  user <- lookupUser userId users {- - Look up user by ID -}
  profile <- lookupProfile user.profileId profiles {- - Look up profile by profileId -}
  address <- lookupCity profile.addressId addresses {- - Look up address by addressId -}
  pure address.city {- - Return the city -}

users = Map.fromFoldable [ Tuple "user123" { userId: "user123", profileId: "profile-123" } ]
profiles = Map.fromFoldable [ Tuple "profile-123" { profileId: "profile-123", addressId: "address-123" } ]
addresses = Map.fromFoldable [ Tuple "address-123" { addressId: "address-123", city: "Faketown" } ]

logShow $ userCity "user123" users profiles addresses 
logShow $ userCity "wrong" users profiles addresses 
```

Expected output:
```
(Just "Faketown")
unit
Nothing
unit
```

This mirrors JavaScript’s Promise chaining but is safer and more generalizable.

### REPL Exercises
Try these in a fresh PSCi session (run `:clear` first). Copy and paste each block using `:paste` mode (type `:paste`, paste, then press `Ctrl+D` to execute):

1. **Monad with Maybe**:
   ```purescript
   import Prelude
   import Effect.Console
   import Data.Map as Map
   import Data.Maybe
   import Data.Tuple
 
   lookupUser userId users = Map.lookup userId users
   lookupProfile profileId profiles = Map.lookup profileId profiles
   lookupCity addressId addresses = Map.lookup addressId addresses
 
   userCity userId users profiles addresses = do
     user <- lookupUser userId users {- - Look up user by ID -}
     profile <- lookupProfile user.profileId profiles {- - Look up profile by profileId -}
     address <- lookupCity profile.addressId addresses {- - Look up address by addressId -}
     pure address.city {- - Return the city -}
 
   users = Map.fromFoldable [ Tuple "user123" { userId: "user123", profileId: "profile-123" } ]
   profiles = Map.fromFoldable [ Tuple "profile-123" { profileId: "profile-123", addressId: "address-123" } ]
   addresses = Map.fromFoldable [ Tuple "address-123" { addressId: "address-123", city: "Faketown" } ]
 
   logShow $ userCity "user123" users profiles addresses 
   logShow $ userCity "wrong" users profiles addresses 
   ```
 
   Expected output:
   ```
   (Just "Faketown")
   unit
   Nothing
   unit
   ```
 
2. **Monad with Array**:
   ```purescript
   import Prelude
   import Effect.Console
   import Data.Array
   import Control.Plus
   countThrows n = do
   x <- [1, 2, 3, 4, 5, 6]
   y <- [1, 2, 3, 4, 5, 6]
   if x + y == n
   then pure [x, y]
   else empty 
   logShow $ countThrows 10 {- - Finds all dice throws summing to 10 -}
   ```
 
   Expected output:
   ```
   [[4,6],[5,5],[6,4]]
   unit
   ```
 
3. **Monad with Sequential Array Sum**:
   ```purescript
   import Prelude
   import Effect.Console
   import Data.Maybe
   import Data.Array
   validateNumber n = if n >= 0
     then Just n
     else Nothing {- - Validates non-negative numbers -}
   sequentialSum numbers = do
     first <- validateNumber (head numbers # fromMaybe 0) {- - Get first number -}
     rest <- Just (drop 1 numbers) {- - Get remaining numbers -}
     sumRest <- if null rest
       then Just 0
       else sequentialSum rest {- - Recursively sum rest, depends on first -}
     pure (first + sumRest) {- - Combine first with sum of rest -}
   logShow $ sequentialSum [1, 2, 3] {- - Successful sum -}
   logShow $ sequentialSum [1, -2, 3] {- - Fails if any number invalid -}
   logShow $ sequentialSum [] {- - Handles empty array -}
   ```
   **Explanation**:
   This example demonstrates monadic dependency by computing the sum of an array sequentially, where each step validates and processes elements, depending on prior results. The `sequentialSum` function uses `Maybe` to handle invalid numbers (e.g., negative). It extracts the first number with `validateNumber`, gets the remaining numbers, recursively sums the rest (depending on the first number being valid), and combines the results. If any number is invalid or the array is empty, it handles the case safely. This mirrors JavaScript’s sequential reduction with Promises or callbacks, where each step depends on the previous sum, but with type safety.
   Expected output:
   ```
   (Just 6)
   unit
   Nothing
   unit
   (Just 0)
   unit
   ```

### Summary
JavaScript’s Promise chains, like fetching sequential data, are error-prone and context-specific. PureScript’s `Monad` type class, with `bind` and `do` notation, provides a type-safe, reusable way to chain dependent computations across contexts like `Maybe`, arrays, or `Aff`. It formalizes JavaScript’s chaining patterns (e.g., `.then` or callbacks) without exceeding the complexity of modern web development. For JavaScript developers, `Monad` simplifies sequential tasks, catching errors early and reducing custom code.
