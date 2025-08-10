## Show 

### The JavaScript Struggle: Debugging Complex Objects

JavaScript developers often debug by logging objects to the console. Suppose you’re working with a user profile object:

```javascript
const userProfile = {
  name: "Alice",
  settings: { theme: "dark", notifications: true },
  preferences: ["email", "sms"]
};
console.log(userProfile); 
// { name: "Alice", settings: { theme: "dark", notifications: true }, preferences: ["email", "sms"] }
console.log(userProfile.toString()); 
// "[object Object]"
console.log(JSON.stringify(userProfile)); 
// "{\"name\":\"Alice\",\"settings\":{\"theme\":\"dark\",\"notifications\":true},\"preferences\":[\"email\",\"sms\"]}"
```

The default `console.log` is verbose, `toString` is useless, and `JSON.stringify` is better but clunky with quotes and braces. If `settings` lacks a custom `toString` or `toJSON`, or if you add a non-serializable property like a function, things break:

```javascript
userProfile.callback = () => console.log("hi");
console.log(JSON.stringify(userProfile)); 
// Omits `callback`: "{\"name\":\"Alice\",\"settings\":{\"theme\":\"dark\",\"notifications\":true},\"preferences\":[\"email\",\"sms\"]}"
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

