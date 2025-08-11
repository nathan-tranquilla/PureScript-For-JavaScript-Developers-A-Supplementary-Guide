## What's Next

Congratulations on working through **PureScript for JavaScript Developers**! You’ve explored key type classes from Chapters 6–8 of *PureScript by Example*, connecting familiar JavaScript patterns—like string conversion, equality checks, array operations, form validation, and Promise chaining—to PureScript’s structured, type-safe abstractions. By now, you should feel confident recognizing how `Show`, `Eq`, `Ord`, `Semigroup`, `Monoid`, `Functor`, `Foldable`, `Applicative`, `Traversable`, and `Monad` formalize JavaScript’s informal practices, making your code more reliable without exceeding the complexity of modern web development.

### Make A Financial Contribution
If this booklet has helped you bridge JavaScript and PureScript, please consider [making a donation](https://github.com/sponsors/nathan-tranquilla).

### Where to Go from Here
This booklet has prepared you to dive deeper into PureScript by bridging JavaScript’s dynamic patterns with PureScript’s type-safe type classes. To build on this foundation, we recommend returning to *PureScript by Example* and picking up with the `Effect` and `Aff` monads in Chapter 8 and beyond:

- **Effect Monad**: The `Effect` monad handles native side effects like console logging, DOM manipulation, or random number generation—tasks you’re familiar with from JavaScript functions like `console.log` or `Math.random`. Chapter 8 of *PureScript by Example* explores how `Effect` encapsulates these side effects safely, offering a structured alternative to JavaScript’s ad hoc approach. Try the exercises to see how `Effect` integrates with real-world applications, like building interactive web interfaces.
- **Aff Monad**: The `Aff` monad manages asynchronous computations, similar to JavaScript’s Promises or `async/await`. It’s a natural extension of the `Monad` concepts you’ve learned, providing type-safe ways to handle API calls, timeouts, or other async tasks. Continue with *PureScript by Example*’s examples and exercises in later chapters to apply `Aff` in practical scenarios, such as fetching data or handling user input.

### How to Proceed
1. **Explore Chapter 8’s Effect and Aff Sections**: Dive into the `Effect` and `Aff` sections of Chapter 8 to see how PureScript handles side effects and async operations. Experiment with the provided examples in `spago repl` or a PureScript project to build familiarity.
2. **Continue with Chapter 9 and Beyond**: *PureScript by Example*’s later chapters introduce advanced topics like monad transformers and real-world applications. These build on the type class foundation you’ve gained, showing how PureScript powers robust web applications.
3. **Apply to Your Projects**: Try using PureScript’s type classes in small projects, like validating forms or processing data, to see how they improve on JavaScript’s patterns. Compare your PureScript solutions to JavaScript equivalents to appreciate the type safety and clarity.

### Why Continue?
The concepts you’ve learned—type classes for polymorphism, data combination, mapping, validation, and chaining—are the backbone of PureScript’s power. By exploring `Effect` and `Aff`, you’ll see how PureScript manages real-world side effects, making it a practical tool for modern web development. These monads are direct analogs to JavaScript’s side-effecting functions and async workflows, so you’ll find them intuitive yet more robust.

Keep experimenting in the REPL, tackle the exercises in *PureScript by Example*, and start building with PureScript to bring type safety to your JavaScript expertise. You’re well on your way to writing reliable, maintainable code with PureScript’s elegant abstractions!