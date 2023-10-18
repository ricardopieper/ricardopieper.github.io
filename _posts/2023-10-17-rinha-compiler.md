---
title: How I improved my interpreter speed by 27x - A Compiler/Interpreter Competition
author:
  name: Ricardo Pieper
  link: https://github.com/ricardopieper
date: 2023-10-17 12:00:00 -0300
categories: [Rust, Interpreters]
tags: [rust, interpreters, tree walker, bytecode, virtual machine]
pin: true
---

In September 2023, one of the most anticipated events in Brazil took place: the "Rinha de Compiladores". This is translated roughly to "Compiler Battle", "Compiler Bout", or "Compiler Fight". If you don't speak Portuguese, "rinha" is pronounced like "rinya" and not "rin haa", the `nh` is pronounced like `ñ` in Spanish. This is part of an ongoing series of Rinhas. The previous one was a backend-focused competition to see which implementation of a simple HTTP API was the fastest. As of the time of writing this text, the Rinha Frontend is taking place, where the goal is to render a huge JSON with many thousands of elements in a tree-viewer-like structure in the least amount of time. The compiler rinha was actually more of an interpreter one, as there were very few compilers (or transpilers).

The masterminds of the competition are [Gabrielle](https://github.com/aripiprazole) and [Sofia](https://github.com/algebraic-sofia). Gabrielle is a 17-year-old girl and Sofia is 21. When I learned about this, I could only remind myself of the time I was 17 years old, playing Counter-Strike all day, playing RF Online, and maybe allegedly probably eating spoonfuls of dirt. This was the general feeling of the Rinha Discord server. Therefore I'd like to say that, although I have some criticism about how the benchmarks were made, what they pulled off is nothing short of incredible. Hundreds of people engaged in the competition, many of those had never written an interpreter in their lives, and now they see it's not some sort of 7-headed beast.

Some very interesting implementations explicitly didn't go for the #1 prize. They were designed to "provide the highest joy to the developer", as did [this person](https://github.com/tiagosh/garbash). This entry deserves the Most Based Implementation prize, but unfortunately, there was no such category.

I ended up not winning the competition. This 27x perf improvement mentioned in the title is compared to my initial implementation, and that 27x only happened after the Rinha finished. My submission only had a 18x improvement over the initial implementation. I admit it's a bit of a clickbait title, but all will be clear in the end. You can check the results by [clicking here](https://github.com/aripiprazole/rinha-de-compiler/blob/main/TESTS.md). I ended up in 18th out of almost 200 submissions (only about 70 working implementations though lol). However, the benchmarking process had some hiccups and was done in a bit of a rush as the organizers admitted. It could very well be that, if the benchmarks were done differently, I could end up even lower in the ranks, but I hope I would finish in a better position.

Some really good implementations, including some of the fastest ones, did not end up in the top 3. I remember the fastest interpreter made in C following the excellent Crafting Interpreters book finished in 7th place. Some even used LLVM and could not hit the top 5. However, I am happy that the winner Raphael Victal actually had a very good implementation, though I'm biased because we had some similar ideas. My interpreter ended up with a score of around 30% less than him, and my interpreter was in fact around 30% slower than his. I am a bit sad with the results, but the reality is that I shouldn't even be on that board, as my submission wouldn't even build 3 minutes before the scores were revealed. Sofia was gracious enough to quickly run my fixed build moments before the reveal livestream, I think I should've been disqualified.

An interpreter usually has 3 phases, as I'm gonna explain in the next paragraphs. Bear in mind that for the Rinha language, we didn't need to write a parser, as the organizers provided a Rust program that would parse the language and spit out the JSON AST. If you don't know what those words mean, don't worry, the next paragraphs explain it.

Interpreter phases
==================

To run code, we don't simply get input text and start running/compiling it. Maybe in the past, some languages actually did it because resources were so scarce, but nowadays this is not the case. Interpreters and compilers usually have multiple steps and each one gives more and more structure to the input program.

## Lexer

The lexer is the simplest part of the interpreter: it takes input strings and returns a stream/list of tokens.


In the Rinha language, function declarations look like this:

```
let add = fn(x) { x + 1 }
```

This is, of course, a function that just returns its parameter. A tokenizer could read this and return a list of the following tokens:

```
[
  Token::LetKeyword,
  Token::Identifier("id"),
  Token::Operator(Operator::Assign),
  Token::FunctionDeclarationKeyword,
  Token::OpenParen,
  Token::Identifier("x"),
  Token::ClosedParen,
  Token::OpenCurlyBraces,
  Token::Identifier("x"),
  Token::Operator(Operator::Plus),
  Token::IntLiteral(1),
  Token::CloseCurlyBraces
]
```

In this case, the only string we have is in identifiers. We could have more strings if I used a string literal somewhere. Identifiers are used for variable names, function names, parameter names, type names, and everything that identifies something in the program. As for the other tokens, I think they are self-explanatory.

That's all a lexer really does. We are now ready for parsing.

## Parser

In 2023, only a madman would be satisfied with just the tokens, and then just interpret them as is. We need more structure. For that, we build an AST, an abstract syntax tree. This process is so mechanical and boring that there are programs you can use to generate parsers for a language. You describe the grammar, and the language makes a parser for you. I'm not a big fan, but I do see why people use them. Some of these tools are ANTLR, Bison, Flex, LALRPOP, and others. You can also write your own Recursive Descent parser, which is described in the Crafting Interpreters book. I really recommend reading this book as it will give you the recipe to build an interpreter.... but I also have to say: It's a lot of fun if you write one in your own way. Then consult the literature to see which mistakes you made. You will never forget how dumb your ideas actually are, and you won't forget about the better idea in the book or even tricks you didn't realize that could be done.

Anyway, at the end of parsing, you'll end up with a structure similar to this:

```
LetExpression {
  name: "id",
  expression:
    FunctionDeclaration {
      parameters: ["x"],
      body: {
        BinaryOp(
          op: Operator::Plus,
          lhs: Variable("x"),
          rhs: IntLiteral(1)
        )
      }
    }
}

```

The astute reader will understand how each piece of this representation relates back to the source code. For Rinha, we also had location information for everything, so what we knew where exactly in the source code each piece was declared.

## Post-parsing

After you parse the input text, you have so many paths you can follow I can't describe every single one in length, so here's a short description. If you know what tree-walking, bytecode VM, JIT, or AOT mean, you can skip this part.

 - **Tree-walking interpreter**: Just recursively walk through the AST and execute the nodes. Along the way, you have to store the currently declared variables and functions. For Rinha, when you find a function, you can evaluate the function as a Function value, just like any other kind of value. For functions, you can store the entire AST data in this value, the parameter names, and a copy of the current environment (like declared variables) that will act as the function closure. This is a bit slow and you can definitely optimize things out, as I'll show in the rest of the article.
 - **Bytecode VM**: Do you know Java? Do you know Javascript? Maybe Python? All of those contain a Virtual Machine (VM) that executes what's called a bytecode, which is a list of instructions similar to Assembly, but for a virtual machine (and not a physical machine like your CPU). The bytecode is designed in tandem with the virtual machine so that the VM can execute the bytecode efficiently. You can think of your physical processor as a physical machine that interprets some ""bytecode"" (machine instructions) and all of that happens in silicon, in hardware. The bytecode VM tries to emulate some of the things the CPU and OS do, like decoding instructions, doing arithmetic, branching (if statements), call stacks, and so on. This tends to be a lot faster than tree-walking interpreters because there's less pointer chasing and better data locality: the AST is a tree structure often with many things allocated all over the heap. Bytecode on the other hand is a more compact representation that fits much better in CPU cache. One of the main overheads of interpreting code is determining "what to do next?", and having a compact representation of the code in the CPU cache helps a lot. It also turns out my approach of HOAS, or what I call Lambda compilation (because it's all boxed lambda functions) also help with that.
 - **JIT (Just In Time) compilers**: In comparison with native code, bytecode still leaves a lot of performance on the table. For a simple instruction in an interpreted language, many dozens of CPU instructions are often needed to fetch values from stack, write on them, manage the VM state, fetching and decoding the next instruction, and so on. But interpreted languages do come with a lot of flexibility and portability. What JIT compilers do is to get the best of the two worlds: at its core, the language is interpreted, but parts of the user's code are compiled down to fast machine code. Sometimes the compiler can figure all of the types of a function even in languages without type annotations (like JS). This is why Javascript is so fast: what V8, JSC, and Spidermonkey do is nothing short of a miracle, and that's due to JIT. When needed and possible, the JIT can generate native code for a function. The same compiled binary or JS file can run in any browser on any computer while retaining really good, sometimes near-native performance. Java and C# are languages that run in a JIT compiler (JVM and RyuJIT respectively, and more recently Java also runs in GraalVM).
 - **AOT (Ahead Of Time) compilers**: This is what Rust, C++, C, Go, and many other languages do. They compile to native code from the beginning. No bytecode VM needed, and you get native performance as a result. You lose the ability to run the same binary in multiple OSs and CPUs, but you gain a lot of performance. Each OS and CPU architecture needs a different binary.


The Rinha Language
------------------

Let's take a look at the Rinha language for a bit to understand how to run it.

```
let fib = fn (n) => {
  if (n < 2) {
    n
  } else {
    fib(n - 1) + fib(n - 2)
  }
};

print("fib: " + fib(10))
```

Rinha is a small dynamically typed functional language. Functions support recursion and you can declare functions inside functions, return functions from functions, take functions by a parameter that, when executed, returns new functions, and all that good stuff from functional programming.

I also wrote a small program that tests some of the language features, except for strings:

```
let iter = fn (from, to, call, prev) => {
  if (from < to) {
    let res = call(from);
    iter(from + 1, to, call, res)
  } else {
    prev
  }
};

let work = fn(x) => {
  let work_closure = fn(y) => {
    let xy = x * y;
    let tuple = (xy, x);
    let f = first(tuple);
    let s = second(tuple);
    f * s
  };

  iter(0, 200, work_closure, 0)
};

let iteration = iter(0, 100, work, 0);

print(iteration)
```

This program does not do anything useful, but it exercises a few things:

 - Tuples (always 2 elements, fetch them using built-in functions `first` and `second`)
 - Closures
 - Function parameters (as in, take functions by parameter and execute them)
 - Tail-calls and Tail-call optimization
 - Some binary operators

This was my benchmarking script for the rest of the competition. Turns out the final tests didn't go that route, but I wanted this thing to run faster and faster.

Just for curiosity, you can build lists in Rinha by using tuples: since tuples can have tuples inside it, you could have something like `(1, (2, (3, (4, "<nil>"))))` or something like that. You can also have maps using a clever closure trick:

```
let empty_map = fn(v) => {
    ("<KeyNotFound>", "undefined key " + v)
};

let set = fn(var, val, m) => {
    fn(v) => {
        if (var == v) {
            val
        } else {
            m(v)
        }
    }
};

let map = set("z", 1, empty_map);
let map = set("x", 2, map);

let v = map("z");
print(v)
```

The inner function `fn(v)` inside `set` takes the current map (represented by the function `m`) by closure. In fact, all `set` parameters are captured by the inner function when `set` is executed, so you can use closures as a mechanism to hold state. To fetch a value, we just recursively call `m` until we find a closure whose `var` is our target. It's a slow `O(n)` map, for sure, but a map nonetheless.

It's interesting to see how powerful functional programming can be: this language does not have built-in types for lists or maps, and yet we can represent those concepts, albeit not in the most performant way. Nonetheless, this idea was used by one of the competitors to make a `meta rinha` program, which is an interpreter built in the Rinha language itself. In fact, the example above comes almost straight from the meta rinha program, I just changed the names. I used meta rinha to find bugs with closures in my interpreter. I think an interpreter that runs another interpreter is a good quick measure for correctness, everything else in the interpreter is simpler than closure handling. This is an example of a meta-rinha program:

```
let fac =
  (ExpLet, ("f", (
    (ExpLambda, ("x",
       (ExpIf, ((ExpVar, "x"),
              ((ExpMul, ((ExpVar, "x"),
                      (ExpApp, ((ExpVar, "f"), (ExpSub, ((ExpVar, "x"), (ExpK, 1))))))),
              (ExpK, 1)))))),
    (ExpApp, ((ExpVar, "f"), (ExpK, 10))))));


let _ = print("factorial 10: " + second(eval(fac, empty_env)));
```

## My entry


My idea for Rinha was to build an initial simple reasonably optimized interpreter and then try to type-check it for AOT, but that didn't work. I did learn a bit about Hindley Milner type inference, but the input language is just too dynamic, there are no type annotations, and it would need some haskell-like type classes to make operator overloading work (i.e. the + does integer sum and string concat). That's too complex for the timeframe I had, so I decided to stick with an interpreter. I wouldn't have much time to dedicate to the Rinha before the competition as I was traveling, but I had been curious about an article I had read about a tree-walker-like interpreter, but on steroids. I did experiment with the approach in the past in a smaller scale, but I thought it was finally time to make it for real

Some time ago, I read a [Cloudflare article](https://blog.cloudflare.com/building-fast-interpreters-in-rust/) in which they built an interpreter for Wireshark filters to add it to their firewall solutions. Their first solution was a treewalker, which was alright... it would be nice if they were a bit faster though. However, the risks and overheads of JITting the filters to all platforms are way too high, outweighing any benefits. They are processing untrusted network input, and doing low-level unsafe programming (like running JIT, generating machine code, marking sections of memory as executable, and so on) also didn't seem like a good idea at the scale they deal with.

Instead, they did a HOAS approach (according to Sofia). HOAS means High-Order Abstract Syntax, and I will not pretend I know what it means. My entry uses Rust, and I actually call it Lambda compilation, because it's all lambdas. It's all dynamic dispatch and closures. HOAS means you use the host language's features to run the language, but it's only true HOAS for the most simple of languages, like lambda calculus. For anything a bit more complicated it's more difficult to pull it off in a pure HOAS fashion. ChatGPT tells me it's actually a mixture of First-Order Abstract Syntax (FOAS) and HOAS, but I will also not pretend I know what exactly it means. Therefore, let's go with lambda compilation, it describes exactly what happens in the code.

Also, bear in mind that I won't worry too much about memory leaks and memory consumption in general, this is a toy interpreter for a competition, not production usage. Still, I think some things I learned would work alright in a production interpreter, while for others I exploited the nature of the competition and just did some hacky stuff to squeeze performance.

For the lambda compilation approach, suppose we need to compile a binary expression that loads a constant value and a variable, I would need this code:

``````

pub type LambdaFunction = Box<dyn Fn(&mut ExecutionContext) -> Value>;

// Enum for runtime values
pub enum Value {
    Int(i32),
    Bool(bool),
    Str(String),
    Tuple(Box<Value>, Box<Value>),
    Closure(...),
    //and so on
}
...

fn compile(&self, ast: &Term) -> LambdaFunction {
  match ast {
    Term::Int { value } => Box::new(move |exec_context| Value::Int(value)),
    Term::Var { name, .. } => Box::new(move |exec_context| exec_context.cur_frame().variables.get(&name).unwrap().clone())
    Term::BinaryExp { op, lhs, rhs } => {
      let lhs_compiled = self.compile(lhs);
      let rhs_compiled = self.compile(lhs);
      match op {
        Operator::Plus =>
          Box::new(move |ec| {
            let lhs = lhs_compiled(ec);
            let rhs = rhs_compiled(ec);
            match (lhs, rhs) => {
              Value::Int(l), Value::Int(r) => Value::Int(l + r),
              //... also handles string and int concat
            }
          }),
        //... also handles other operators
      }
    }
    ...
  }
}

``````

Instead of matching the AST at runtime, we try to pre-compile every decision that the tree walker would do. For instance,
a regular treewalker would have no choice but to pattern-match every kind of operator (plus, multiply, minus, etc) at every execution of a binary operation. In our scenario, we already looked into it and determined we only need to do the dynamic type checking to ensure both sides are `Int`, the operator + handler is returned directly.

How much faster is this compared to a regular tree walker? Cloudflare claims it's 10~15%, and their article also uses Rust, but the language is very different.

The problem is: I didn't measure it in Rust. However, I did measure it in Ocaml using ocamlopt, and I have this [OCaml version of the Rinha interpreter](https://github.com/ricardopieper/rinha-ocaml) that compares the two approaches. In general, I'm getting around 8% improvement using this approach. Doing it in OCaml was much more of a joy compared to Rust, but in Rust it wasn't too difficult either. I'd say this lambda compilation strategy is a bit more complicated to debug though.


So I wrote the initial implementation and tried to run the perf.rinha file, here it is in case you don't want to scroll all the way back:

```
let iter = fn (from, to, call, prev) => {
  if (from < to) {
    let res = call(from);
    iter(from + 1, to, call, res)
  } else {
    prev
  }
};

let work = fn(x) => {
  let work_closure = fn(y) => {
    let xy = x * y;
    let tuple = (xy, x);
    let f = first(tuple);
    let s = second(tuple);
    f * s
  };

  iter(0, 200, work_closure, 0)
};

let iteration = iter(0, 100, work, 0);

print(iteration)
```

This code ran in 43ms. That's our baseline. It's pretty bad. Let's optimize it.


## Optimization 1: BTreeMap instead of HashMap

In the interpreter VM state, I store variables in the call frame data. Each time a function is called, I create a new call frame with an empty `HashMap<&str, Value>` map for the variables, and store all let bindings, closure, and params there.

This first optimization is just changing that map to a `BTreeMap<&str, Value>` instead. Hashing strings is kinda expensive, so let's stop doing that.

After this change, we're down to 28ms, a 49% improvement.

## Optimization 2: Use Vec instead of any kind of map

During compilation, we scan all variables in the program, and I actually have a process in which I intern their strings. I end up with a list of all strings used for variables in the program, like `[iter, from, to, call, prev, res, work, work_closure, y, xy, tuple, f, s, iteration]`, 14 variables. Instead of strings, I have a `usize` number for each one, where `iter=0, from=1, ` and so on. So this is how it worked:

 - At compile time, I tell the call_function procedure to create a call stack with 14 slots
 - Copy parameters and closures into these 14 slots, as available
 - Whenever I need to fetch these values, I just tell which slot index I want. At compile time this becomes a simple `stack_frame.variables[pos]`, no map lookup or hashing needed
 - ????
 - Profit

The result was... underwhelming. Now we're slower than before, 35ms. This is because each new frame eventually needs an allocation of size 14, and the size of `Value` at this point I think was around 24 bytes. This is a bit of foreshadowing so let's keep going.

## Optimization 3: Use a separate Vec for each variable and track their evolution separately

To avoid every new frame needing a new big allocation, we need to make this process trivial. Therefore, I started tracking the variables in my global VM state:

```
struct ExecutionContext {
  ...
  call_stack: Vec<CallFrame>,
  variables: Vec<Vec<Value>>
  ...
}

```

Variables is now a 2D vector in which the first dimension identifies the variable, and the second dimension is just the variable's "stack". So instead of tracking frames of variables, I track variables of frames (???). Reminds me of column databases, where a table is not millions of rows broken apart into many cells for each field: The table has one big array for each of its fields, and each index into that field identifies a record.

To access the variable `from`, one just needs to to `self.variables[0].last().unwrap()`, and new call frames just need to push new values onto the respective indices for each variable. When a stack frame ends, we need to pop the values that were used, therefore, the call frame tracks which variables were pushed:


```
struct CallFrame {
 ...
 pushed_vars: Vec<usize>

}
```

When we pop a frame, we read the pushed_vars and pop each variable:

```
fn pop_frame(&self) {
  let frame = self.call_stack.pop().unwrap();
  for v in frame.pushed_vars {
    self.variables[v].pop()
  }
}

```

That optimization brings us to 14ms, the fastest yet. That's a 204% improvement from baseline.


## Optimization 4: Reuse frame allocations

Let's take a look at our fib function:

```
let fib = fn (n) => {
  if (n < 2) {
    n
  } else {
    fib(n - 1) + fib(n - 2)
  }
};

print("fib: " + fib(10))
```

This code does a lot of unecessary work, allocates a lot of call frames, it's incredibly wasteful. To keep track of pushed vars, we are still doing a lot of allocations and array resizings. What if we could reuse these allocated spaces?

In the line `fib(n - 1) + fib(n - 2)`, we will execute `fib(n-1)` which will allocate some frame data, and then dispose the frame, But we could instead store that frame into a `reusable_frames` array, and at every new frame, we pop this array and clear the data inside the pushed_vars. This does not free the allocation, allowing us to reuse it. This helps on the `perf.rinha` benchmark too, as we execute the functions in a loop, their frames are also reused.


This resulted in a 91% improvement from our last optimization, down to 7.4ms. We're at a 482% improvement from baseline.


## Optimization 5: Non-heap tuples

Let's take a look again at our Value enum:


```
pub enum Value {
  Int(i32),
  Bool(bool),
  Str(String),
  Tuple(Box<Value>, Box<Value>),
  Closure(...),
  //and so on
}
```


Our `perf.rinha` creates a tuple, which results in 2 heap allocations. Heaps can be of any type, so it's necessary for them to be `Box<Value>` so I can have things like `(1, (true, ("foo", fn() {})))`.

**Or do we 🤨?** (cue in Vsauce music)

Turns out `perf.rinha` allocates only (int, int) tuples. So here's the idea:

```
pub enum Value {
  Int(i32),
  Bool(bool),
  Str(String),
  IntTuple(i32, i32),
  BoolIntTuple(bool, i32),
  IntBoolTuple(i32, bool),
  BoolTuple(bool, bool),
  Tuple(Box<Value>, Box<Value>),
  ...
}
```

If we have a primitive type-only tuple without tuples inside it, we can just have specialized tuple types that don't talk with the allocator.

This resulted in a 118% improvement in runtime, down to 3.4ms. We were at 7.4ms before. This is ~1170% improvement from baseline.

However, this is where the low hanging fruit ends. My final result will be 1.55ms, and this remaining 1.9ms will be difficult to eliminate. **But let's take a detour for a moment**, it's time to support recursion in a better way.


## Optimization 6: Trampolines

This item will explain the implementation of Tail-Call optimization, but we won't gain much performance yet. At this point in the competition, I was worried about recursion. Let's take a look at fib again:

```
let fib = fn (n) => {
  if (n < 2) {
    n
  } else {
    fib(n - 1) + fib(n - 2)
  }
};

print("fib: " + fib(10))
```

If you compile similar code with agressive optimization in Rust or C, it transforms this code into a loop. There is an optimization step in LLVM that transforms algorithms such as fib into their iterative form. It only really does this sometimes, not always, there are some constraints that need to hold.
This fibonacci algorithm, as is, is not in tail call form. There's nothing I can do to make it a tail call short of copying LLVM's implementation.

However, in functional programming, you can just write your function in tail call format:

```
let fib_tc = fn (n, a, b) => {
  if (n == 0) {
    a
  } else {
    fib_tc(n - 1, b, a + b)
  }
};
```

Whenever the last thing a function does is a call to itself, we can actually reuse the stack frame by transforming into this format:

```
let fib_tc = fn (n, a, b) => {
  if (n == 0) {
    a
  } else {
    fn () => fib_tc(n - 1, b, a + b)
  }
};
```

In this case, the compiler can apply a pre-processing step on the AST and wrap the function call in another function instead of evaluating it immediately.

Then at runtime the interpreter does something like this, in pseudocode:

```
let result = call(fib_tc, ...);

while (is_callable(result)) {
  result = result();
}

return result
```

Notice that when we call fib_tc, we only create one stack frame. This stack frame is popped when the call ends, and then the `while` loop checks if the result is actually a callable value. If so, we call it (in a new stack frame, but we popped the stack before so it won't create a deep stack), the function returns (stack frame is popped), and so on. Due to this jumping behavior up and down in the stack, this technique is called a `trampoline`.

This makes algorithms able to run indefinitely, or until we run out of memory. There is also CPS (Continuation-Passing Style) that can even handle the original recursive `fib` function, but they are more complex to implement. Trampoline will suffice for now.


This resulted in no performance gain, but it's a nice feature... feat not, the trampoline will come in clutch in a few paragraphs.


## Optimization 7: Reduce the size of the `Value`.

At this point, here's our full `Value` enum:

```
pub struct Closure {
    pub callable_index: usize
}

pub enum Value {
    Int(i32),
    Bool(bool),
    Str(&'static str),
    IntTuple(i32, i32),
    BoolIntTuple(bool, i32),
    IntBoolTuple(i32, bool),
    BoolTuple(bool, bool),
    DynamicTuple(Box<Value>, Box<Value>),
    Closure(Closure),
    Trampoline(Closure),
    Error(&'static str),
}
```

There's a problem: The value is whopping 24 bytes in size. That's bad. On a 64 bit computer, we can handle 8 bytes at a time, and we're 3x larger than that.
This enum is 24 bytes because it uses 1 byte for the tag, which needs to be aligned, so Rust determined 8 bytes are needed. Then we need enough space for the largest variant,which is `DynamicTuple(Box<Value>, Box<Value>)`, used when tuples contain values other than Int or Bool. Each Box needs 8 bytes, so there are 16 bytes for the value, plus 8 bytes for the aligned tag.

At the time I wrote the optimizations we'll see, I did not know about NaN tagging. NaN tagging is a technique where you represent every value as a NaN variant. Turns out there are many many double values that are NaN, and we can use silent NaNs and manipulate them however we want, and if we ever need a double then we can just use the value as is. Rinha, however, has no doubles, only 32 bit signed integers, so the usefulness of NaN tagging is limited. However, NaN tagging also handles pointers: In most machines, pointers only really use 48 bits of information. Maybe in the future I will implement something similar to NaN tagging.

The way I did pointers was to store values in a `Vec` and use a 4-byte index. You can allocate at most ~4 billion tuples, ~4 billion strings and ~4 billion closures in my interpreter... it's a price we pay for performance. This is the new definition:

```
pub struct Closure {
    pub callable_index: u32,
    pub closure_env_index: u32
}
pub struct HeapPointer(u32);
pub struct StringPointer(u32);

#[derive(Clone, Debug, Eq, PartialEq, PartialOrd, Ord)]
pub enum Value {
    Int(i32),
    Bool(bool),
    Str(StringPointer),
    IntTuple(i32, i32),
    BoolIntTuple(bool, i32),
    IntBoolTuple(i32, bool),
    BoolTuple(bool, bool),
    DynamicTuple(HeapPointer, HeapPointer),
    Closure(Closure),
    Trampoline(Closure),
}
```

This is 12 bytes. Now Rust chose a 4-byte alignment due to HeapPointer and values inside Closure being 4 bytes. Notice I also got rid of the Error variant, it was useless. Now we can fit much more Values in CPU cache and they are much faster to copy. Now we are down to 2.9ms from 3.4ms. 17% faster than previous, and 1386% from baseline.



## Optimization 8: Reduce the size of the `Value` even more

This is just an extension of the previous idea. Let's get it down to 8 bytes by creating a `ClosurePointer` of 4 bytes.
I also had to remove the extra tuple values and make all of them point to heap values, otherwise, the size of the enum would still be 12 bytes because i32 and bool align in 4 bytes.

We're at 2.5ms, 16% improvement over last, 1624% over baseline.


## Optimization 9: Frame reuse in TCO

Currently, our TCO implementation just avoids creating a physical call frame on every call inside the recursive function, but it still creates a virtual VM frame. This creates a lot of unnecessary allocations. In the end, I spent a whole afternoon making it work, but the implementation ended up being really simple: a flag on the call frame that tells whether we should reuse the fame. Then calling the function recursively, we read that flag and simply skip popping and pushing that frame. Only when the trampoliner finishes executing we really pop the frame.

This reduced the runtime to 2.2ms, 13.6% faster than the previous, and 1859% over baseline.


## Optimization 10: Empty closures

Some functions don't capture values outside of them. However, the interpreter still allocates a closure value for every function, including ones that are frequently called but capture nothing.

When we compile a function, we store it in a vector and pass a slice of this vector to the execution context:

```
pub struct Closure {
    pub callable_index: usize,
    pub closure_env_index: usize
}

pub struct Callable {
    pub parameters: &'static [usize],
    pub closure_indices: &'static [usize],
    pub body: LambdaFunction,
    pub trampoline_of: Option<usize>
}


pub struct ExecutionContext<'a> {
    ...
    pub functions: &'a [Callable],
    pub closures: Vec<Closure>,
    pub closure_environments: Vec<BTreeMap<usize, Value>>,
    ...
}
```

For closures, at this point we are using `BTreeMaps`. The usize key is the variable ID. Every time we declare a function, we create a new `BTreeMap` and store it in the vector, which makes the `closure_environments` grow rapidly. But there are many cases where this is not necessary: If the compiler analyzes the function and decides it doesn't capture anything, we can just use one canonical "capture-less" function.

Therefore, at the beginning of the execution during instantiation of the `ExecutionContext`, for every callable we store a Closure that points to a non-existent `closure_env_index`, like `u32::MAX`. This points to a value that doesn't exist and would crash if used, but the fact that it tried to load a non-existing closure is a bug in our analysis step.

```
  let empty_closures = {
      let mut closures = vec![];
      for (i, _) in program.functions.iter().enumerate() {
          closures.push(Closure {
              callable_index: i,
              closure_env_index: usize::MAX,
          })
      }
      closures
  };
```

Whenever we find a non-capturing function, we evaluate it as a `Closure { callable_index, closure_env_index: u32::MAX }` and return a `ClosurePointer(callable_index)`. As you can see in the code above, callable index points to the actual callable index in the functions vec.


This entire thing improved the performance by..... 0%. Nice :) Some benchmarks did run without exploding memory though, so that's good.


### Optimization 11: Small Tuples

For tuples with int values that fit into i16, we don't heap allocate them, instead, I created a `SmallTuple(i16, i16)` variant.
Down to 2.1ms.


### The bug fixes

So many optimizations done, everything must be running just fine, right? Well... it turns out I had some massive bugs regarding closures. Sometimes values weren't being loaded from where they should. Basically, something like this would happen:


```
let y = 10;
let f = fn() { y };
let y = 20; //shadowed!
print(y); //should print 10, not 20
```

To my horror it was printing 20 because values weren't being copied into the closure properly. Unfortunately, after fixing it, the performance got much worse, up to 3.4ms. That's a huge setback.

However, the fix was quite easy.

### Optimization 12: Minimal function closures

Our capture analysis in Optimization 10 missed the opportunity to reduce the copies. The way closures were created was:

 - Get all the variables in scope right now
 - Copy them into the closure
 - GG

However, the closures usually need only a subset of these variables. Turns out I had that information in hand, so it was easy to use that.
Down to 2.2ms, almost pre-bugfixing levels.

This was the program I submitted for the competition.

### One more hiccup...

This one got almost everyone by surprise. Sofia tested her benchmark program only to break almost all interpreters, and not for the reasons we expected.

This is the code that they tried to run, which isn't that big, having almost 400 lines of code. You can take a look at it [here](https://gist.github.com/aripiprazole/d46c315d6923c64ad5082b6d221e83e8). Each `let` statement has a `next` node, which could point to another `let`, that has another `next`, and so on. In my case, Serde would just not parse that file. It complained about "maximum recursion depth" or something, so I had to change some configs and add some dependencies, and then I got afraid my compiler would crash due to excessive recursion. I thought they had like thousands and thousands of lines of code.

After changing some stuff (using lists of let expressions instead of a linked list of `let`), I fixed the issue but lost a bit of performance, now the program runs in 2.3ms. I would only understand why a few days later, but at this time, I had to quickly submit a fix.

At this point, my interpreter was running fib(46) in around 220 seconds, while the winning submission from Raphael was running it in 140 seconds without memoization. Ah yes, some people implemented memoization, to varying levels of success. Raphael did too, but this 140s number was acquired when his interpreter didn't have memoization.

The next optimizations were done after the competition.

### Optimization 13: Undo optimization 3

In Optimization 3, we did that weird variable stack tracking that resembled, IMO, a column-based database. I decided to get rid of it, and similar to Optimization 12, compute exactly how much space the stack frame needs. I had the information available. Then I changed the Call frame to store a `Vec<Value>` inside it, like God intended. We got down to 2.0ms from 2.3ms, which is the fastest so far. But this revealed some subtle bugs that I had to fix, so it's also the fastest and most correct implementation at this point.

The next 2 implementations were very easy and had massive effects.

### Optimization 14: Solving the pre-submission slowdown.

What happened in the pre-submission bugfix?

Well, `let` statements became vectors instead of linked lists from the AST format. Because of that, bodies of functions and if statements had lists of expressions that had to be run like this:

```
let all_statements = self.compile_exprs(body);
Box::new(move |ec, frame| {
    let mut last = None;
    for l in leaked {
        last = Some(l(ec, frame));
    }
    last.unwrap()
});
```

It turns out this whole looping machinery causes a lot of overhead when we only have one simple expression that, in the end, just returns a variable or a literal value. If only we knew how many statements we had in the body, we could do something about it...

```
fn join_lambdas(&mut self, mut lambdas: Vec<LambdaFunction>) -> LambdaFunction {
  if lambdas.len() == 1 {
      return lambdas.pop().unwrap();
  }

  Box::new(move |ec| {
    let mut last = None;
    for l in leaked {
        last = Some(l(ec));
    }
    last.unwrap()
  });
}
```

With this change, `perf.rinha` barely changed, from 2.0ms to 1.9ms. However, fib(30) went from 110ms down to 77ms(!). This is a massive gain.

Other loop-heavy code also got significantly faster. For instance, this code that counts to 2 billion:

```
let loop = fn (i, s) => {
   if (i == 0) {
     s
   } else {
      loop(i-1, s+1)
   }
};
print(loop(2000000000, 0))
```

This used to run in 60 seconds and now runs in 47s, rivaling the fastest interpreter among the people in the Discord chat (the 7th place submission from fabiosvm).


### Final optimization: Pass the call frame along

Whenever we build a lambda function, we do this:

```
  Box::new(move |ec| { ec.frame().stack_data.... /*do something with frame data*/ })
```

The `ec.frame()` call returns the top of the call stack, which is stored in a `Vec` inside the execution context. However, why do this when the call stack is quite explicit in our program? Why save in a vector, which needs to do a size check for every push even if enough memory exists?

Turns out we can just do this:

```
Box::new(move |ec, frame| { frame.stack_data.... /*do something with frame data*/ })
```

Just pass the frames along. When we call a function, we create the new frame and pass it to the callee's compiled `LambdaFunction`.

This results in a massive improvement.

`perf.rinha` now runs in 1.55ms, down from 1.95ms. This is where I achieved the 27x speedup. `fib(46)` runs in 120s from 190s of the last optimization and the big looping function that counts to a billion now runs in about 35 seconds instead of a full minute.

I think if we ran the benchmark again I would have some chance of getting a better position ;)


# Conclusion

I had a blast working on Rinha. At some points, I had to fire up callgrind and cachegrind to see what was going on, had to do a bunch of experiments that ended up not working, but I just love writing code in Rust. Even if I didn't win the competition I had a lot of fun programming and talking with people on Discord and exchanging ideas.

Maybe the next challenge is to fully type-check the language and use LLVM, who knows?

Or maybe I should just continue working on my other compiler project, Donkey, which is a statically typed (with type annotations) Python-like programming language that tastes like C with some C++-template-inspired generics. Or maybe rest a little bit :)
