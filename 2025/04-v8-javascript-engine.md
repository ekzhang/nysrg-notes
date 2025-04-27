- The V8 JavaScript engine (Apr 2025)
    - V8 bytecode is a "register machine with an accumulator register." So this is a bit different from Wasm's stack machine, for instance. There are a fixed set of numbered registers.
    - What is the accumulator register? Looks like an implicit input-output, like `%rax` in x86.
    - "Concise" bytecode, 25-50% the size of machine code. Maybe this was a design goal, but how is it possible for the bytecode to be that much smaller?
        - (Oh, well we're loading named properties from a lookup table instead of GEP's.)
        - Originally Ignition did not exist, and V8 had a faster baseline compiler before its full JIT. This changed with their high-performance interpreter: it generates fast, optimized handlers for each opcode.
        - "Macro-assembly" reuses some of Turbofan's architecture-independent machine code to generate the interpreter handlers. This lets them have a really fast interpreter (faster than C++, avoids trampolines) but also avoids handwriting assembly for each platform.
    - Historically V8 had "full-codegen" and Crankshaft (2010-2016 era). This got replaced with just Ignition and Turbofan around 2017, so Ignition generates bytecode and Turbofan optimizes it after enough passes to trigger the JIT.
        - Ignition bytecode is linear, but Turbofan optimizes it via a control flow graph.
        - Animation of building the **"sea of nodes" graph** by walking through the operations: https://docs.google.com/presentation/d/1chhN90uB8yPaIhx_h2M3lPyxPgdPmkADqSNAoXYQiVE/edit?usp=sharing
    - What V8 bytecode looks like (note: as of 2016, when it was ~100 kLoC):
        - [Example of modern V8 bytecode in for..of loop](https://github.com/v8/v8/blob/ef7991fbda9e4f7d26c3b5b556bea99932e8476d/test/unittests/interpreter/bytecode_expectations/ForOfLoop.golden)
        - Glossary
            - "Ld" = load, "St" = store, "A" = accumulator, "R" = register
            - SMI = Smallint. or a small integer value between -2^31 and 2^31-1, which is efficient to pass around and call functions with.
            - Strict/Sloppy refers to JS "strict mode" (2009) which was introduced to make the language less janky and easier to optimize. Modern websites [use strict mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode).
        - Star/Ldar: accumulator, store into or load from register. (Ld/St : Accumulator : Register)
        - LdaZero, LdaSmi8 (SMI = Smallint), LdaNull, … = load into accumulator
            - LdrUndefined = load into register, micro-optimization for `undefined` since it's common
        - Binary operators: Sub/Add/Mul/…: arithmetic operations on accumulator and a register.
        - New (objects), CreateClosure (functions)
        - LdaGlobal, LdrGlobal, StaGlobal[Strict/Sloppy]
        - Unary operators: Inc, Dec, … DeleteProperty[Strict/Sloppy] act on the accumulator
        - Call, TailCall, CallRuntime, …
        - Conditionals and jumps
            - Test[Equal, NotEqual, LessThan, …]
            - Jump, Jump[Constant, IfTrue, IfFalse, IfUndefined, IfNotHole, …]
        - Context operations
            - PushContext, PopContext, [Lda/Ldr/Sta]ContextSlot
            - What exactly is a context? Is it used for switching between call stacks in generators perhaps, so you can have efficient async code?
            - SuspendGenerator, ResumeGenerator also exist
        - Other control flow
            - Return: return the accumulator.
            - Throw/ReThrow (hm, how are the exceptions caught?)
            - ForIn[Prepare,Next,Done,Step] opcodes baked in to support `for ... in` loops (seems unnecessary in modern JS)
        - Load/store of object properties
            - Separate opcodes for "named" properties or "keyed" accesses by int
    - ["Hidden classes" in V8](https://v8.dev/docs/hidden-classes), "shapes" in SM
        - Important optimization in JITs. Allows the compiler to infer the shape of a repeated object like `{ a: number, b: string }` and then store that more efficiently.
        - Powers the prototype chain, turns dynamic dispatch into more static dispatch. This is what lets you dynamically create objects in JavaScript and be lax about declaring object shapes ahead of time.
        - The JIT is super important here, otherwise you do repetitive work for these kinds of objects. Think of the common pattern of returning arrays of CRUD objects for instance, and filtering or ordering them in a table.
        - Got linked [Hermes](https://github.com/facebook/hermes), a project to AoT-compile React Native apps into bytecode since Apple doesn't allow you to run JIT software on iOS for security reasons.
        - This is tied closely to __deoptimization__, as you track when conditions change and you need to return to bytecode. Optimized code has guardrails or assumptions, and if you notice one of those is breached then you gotta return to the interpreter.
    - Security in V8 (see [sandbox](https://v8.dev/blog/sandbox) blog post)
        - Memory safety is a big issue, in particular, subtle logic issues in memory accesses can turn into vulnerabilities. Can't be fixed by a memory-safe language. Of course that could fix the compiler __itself__, but not the JIT-generated code that's immediately evaluated.
        - Referred to as "2nd-order logic bugs" in the post.
        - __"The basic idea behind the sandbox is to isolate V8's (heap) memory such that any memory corruption there cannot "spread" to other parts of the process' memory."__
        - Ideally you could implement this in hardware, with a mode-switch feature to make it impossible for the CPU to access out-of-sandbox-memory. But no hardware implements this, so they have something software-based instead.
        - How it works: all heap pointers are actually 40-bit offsets (1 TiB) into a reserved range of virtual address space for the sandbox. Not stack, just heap, and you have to compile an extra shift+add on every global memory load. (+1% perf overhead)
        - This is a good security boundary because it contains the impact of memory bugs in JIT-generated code, while also not penalizing performance much.
        - Think about serverless isolates: having strong memory safety is paramount.
    - Looking at a [V8 type confusion RCE](https://github.blog/security/vulnerability-research/the-chromium-super-inline-cache-type-confusion/)
        - Super inline cache (SuperIC) feature, history of being exploitable.
        - "Inline cache" is related to shapes. The Ignition bytecode collects profiling data for speculative optimizations on every run. Feedback is used to generate optimized machine code, so you don't need a dict property lookup.
        - Each V8 JS object has a "map" as its first property. Stores type + property offsets. If two objects have the same map (shape), their memory layouts are identical.
        - Each bytecode is run through simple IGNITION_HANDLER in `interpreter-generator.cc`.
        - Inline cache is IC. LoadIC_BytecodeHandler versus LoadIC_Miss, fallback if there's not enough feedback or the `map` is previously unknown.
        - It gets more complex from here. The tl;dr is that IC is great for optimizing property accesses, but if you mess up the validity of the optimized handler when using it on a LoadIC cache hit, you're in for a bunch of memory vulnerabilities.
        - Vulnerability background: Behavior of `super` and the prototype chain. In V8, there are a few different objects have fundamentally different implementations: compare `JS_OBJECT_TYPE` and `JS_TYPED_ARRAY_TYPE`. You can set the prototype inheritor to be a totally wacky object, which doesn't have the same properties accessible via `super`.
        - So the "super IC" is used for inline caching super property accesses. But it's checking the wrong `lookup_start_object_map`, for the object and not its superclass, boom!
        - To exploit the vulnerability, gets some JS objects, develop arbitrary memory writes in Blink, and then overwrite a compiled WebAssembly.Instance object code.
            - There is a `wasm-memory-protection-keys` flag, but can just overwrite in memory.
        - "Being able to discover and exploit these bugs requires a great wealth and depth of knowledge in both Blink and V8, and these bugs give us a glimpse of both the resources and expertise that bad actors possess and perhaps an area where more research is needed."
    - [CodeStubAssembler](https://v8.dev/blog/csa) (CSA / Torque)
        - Used to generate pre-compiled code implementations for JavaScript builtins, to speed up startup so that TurboFan doesn't have so much work for each operation.
        - For example, `string.length` is a builtin function, written in assembly.
        - CSA is like a cross-platform, universal assembly language that paints over the difference between architectures. So it uses the optimal instruction on each architecture, and it's a pretty lightweight abstraction.
        - CSA involved hand-writing assembly though and was too complicated. Torque is a DSL to make this less buggy / tricky to write.
        - See `src/builtins` directory, with .tq files.
        - For example, here is the implementation of [Array.from()](https://github.com/v8/v8/blob/master/src/builtins/array-from.tq), 170 lines of code. Kind of like a straight port of the JavaScript TC39 specification for its behavior.
    - How does the V8 microtask queue work and how is it different from Node's libuv?
        - Looks like microtasks are just a push-only queue of stuff to run after the JavaScript interpreter finishes what it's currently doing.
        - Microtasks are all run until completion, before the next task is run (e.g., an event handler or callback). Examples of microtasks are the body of a `new Promise()`, the callback of a MutationObserver, or stuff provided to enqueueMicrotask() manually.
        - This way, microtasks are built into V8, rather than being a feature of the surrounding runtime. Like the Blink render engine, or the Node.js event loop based on libuv.
        - Can be used for consistent execution order between Promise/non-Promise branches.
    - How does CSA actually generate machine code from macros — what are the macros?
        - `src/builtins/builtins-util-gen.h` defines TF_BUILTIN macro, used ~300 times
        - `src/codegen/define-code-stub-assembler-macros.inc` has `CSA_` macros
        - It looks like the implementations of TF_BUILTIN() do emit code via functions on the AssemblerBase class like Goto(), GotoIf(), and Branch().
        - I guess the idea is that CSA needs to generate code for builtins, which is generic over different input types (Smi vs Float for instance) that is only available at runtime. So that's why it can't just be a C++ function. But it's not as dynamic as actual JS code, so it's more efficient to write it in this lower-level language.
    - What heuristic does TurboFan use to determine when to optimize code?
        - This is defined in the flags. 8x for Sparkplug (feedback register allocation), 400x for Maglev or 1000x on Android, and 3000x for Turbofan.
        - I guess it means that constant-factor overhead for Turbofan can be something like 1000x the cost of actually running the code. That gives the optimizer a bit of wiggle-room to deduce stuff about the code and run control-flow analysis.
        - ```c++
          // Tiering: Turbofan.
          DEFINE_INT(invocation_count_for_turbofan, 3000,
                     "invocation count required for optimizing with TurboFan")
          DEFINE_INT(invocation_count_for_osr, 500, "invocation count required for OSR")
          DEFINE_INT(osr_to_tierup, 1,
                     "number to decrease the invocation budget by when we follow OSR")
          DEFINE_INT(minimum_invocations_after_ic_update, 500,
                     "How long to minimally wait after IC update before tier up")
          DEFINE_INT(minimum_invocations_before_optimization, 2,
                     "Minimum number of invocations we need before non-OSR optimization")
          ```
    - Compiling V8 and making a small change
        - https://github.com/pranayga/expl0ring_V8/blob/master/experiments/builtins/math_is42/commit_log.md
        - I'll try to add `Math.collatz()`.
        - Nevermind, this guide is 8 years out-of-date, and the MathBuiltinsAssembler doesn't exist anymore. Also all of the Math builtins are in Torque as well.
        - Difference between builtins and builtins-gen, which are generated by CodeAssembler.
        - Seems that the codebase has gotten a good deal more complex in 8 years.
        - We found the implementation of `GlobalIsNaN` though, that one is still in CSA. Good example! It's a bit verbose, but the way they generate the code is by basically having a function to emit each assembly instruction, so the C++ mirrors the assembly.
    - Pointer compression
        - https://v8.dev/blog/oilpan-library
        - https://v8.dev/blog/pointer-compression
