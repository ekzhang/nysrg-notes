- Solid.js — [[August 11th, 2024]]
    - Tutorial notes
        - onCleanup can be called within any scope, and rendering the component is a top-level singular effect. <- interesting, what does this mean for nesting effects and having calls to effects in loops? what is onMount __really__ doing?
        - Event delegation is interesting, what's going on there? Can you emit custom events? Why use delegation versus just native events? (onMouseMove vs on:mousemove)
        - There is a button to see the compiled output in the tutorial. What's going on there? There's an _$effect() function that looks relevant, and _$delegateEvents([]).
        - Syntactic sugar: Style is nice and consistent, going to native CSS variables etc. classList for dynamic or conditional styling, refs, "use:" directives as a compiler transformation.
        - Reactivity breaks with object destructuring, use splitProps. **Hmm… why isn't this an issue when accessing a property of an object?**
        - Something funky is going on in the [props/children tutorial](https://www.solidjs.com/tutorial/props_children), take a closer look.
            - > The reactive variable 'props.children' should be used within JSX, a tracked scope (like createEffect), or inside an event handler function, or else changes will be ignored.
              eslint(solid/reactivity)
        - "Nested signals" and "proxies" seem like the magic sauce. "Stores" combine these all into a unified syntax sugar for updating state and making signals. Compare to Immer.js with the produce() store, local mutability.
        - Context = dependency injection.
    - Source code: about 6000 lines total inside the packages/solid folder, excluding tests
        - h, html: jsx runtime fragments for DOM elements
        - src: core library (3500 lines) — this is also called "solid-js"
            - reactive (1700 lines)
                - Use of the [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) builtin object is evident here.
                - Includes patch of S.js, signals library, along with a timing-tuned scheduler
                - Owner = computations[], cleanups, contexts and name
                - Computation (Init => Next) = effect function, ComputationState (enum 0,1,2 — stale/pending), value, suspense, etc., signal state (sources) each with observers
            - render
                - client rendering core
            - server
                - ssr core
        - store: store library for mutables / deeply nested reactivity (2500 lines)
        - universal: small library to create a custom renderer
            - Actually just a re-export of dom-expressions right now
        - web: connects the reactivity engine to the DOM, plus SSR support (250 lines)
    - dom-expressions -> "rxcore" babel-mapped to solid-js
    - babel-plugin-jsx-dom-expressions
