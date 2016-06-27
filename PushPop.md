# Text Format Idea: Explicit Push and Pop

Push and pop are an idea for visually splitting up expression trees. Push
and pop connect subtrees to their parents, allowing them to be written
separately in the text syntax, but still be part of the same conceptual tree
in the wasm semantics, and in the wasm binary format.

Here's the proposed text syntax for the `Q_rsqrt` example from TextFormat.md,
but with `push` and `pop`:

```
   function $Q_rsqrt ($0:f32) : (f32) {
     var $1:f32
     $1 = f32.reinterpret/i32 (1597463007 - ((i32.reinterpret/f32 $0) >> 1))
     push:0 $0 = $0 * 0x1p-1
     $1 = $1 * (0x1.8p0 - $1 * pop:0 * $1)
     $1 * (0x1.8p0 - $1 * $0 * $1)
   }
```

Note that the original version has a `set_local` buried in the middle of a
tree, making it easy for a human to miss. Humans wouldn't write code that
way, but in wasm, compilers are *incentivised* to write it that way, because
it reduces code size. It's going to happen a lot, and the push/pop mechanism
gives us a way to make this more readable in many cases.


## Discussion

In a normal programming language, the preferred way to split up a large
expression tree would be to simply assign some subtrees to their own local
variables. Of course compilers can optimize them away as needed, so there's
no reason not to do this.

However in wasm, introducing locals increases code size, so
compilers producing wasm aren't going to do that. There will be a lot of code
in the wild with very large monolithic trees, because compilers will be writing
code that way to minimize code size. And, binary->text translation can't
introduce local variables, because that would make binary->text->binary lossy.

The solution proposed here: `push` and `pop`. `push` pushes subtrees onto a
conceptual stack, and `pop` pops them and conceptually connects them to the
tree that that point. It's important to realize that this is purely a
text-format device. These constructs just exist to build trees. In the abstract
wasm semantics and in the binary format, the trees just exist in monolithic
form.

Now there's a question: how should a binary->text translator decide where to
split up trees? It turns out, we can let binary->text translators choose what
they think is best in their situation:

 - Split trees at `set_local` operators. This is what the examples here do,
   and it's balance delivering readability while still keeping the code
   fairly concise.
 - Split trees at nodes with "side effects" (call, `store`, etc.). This can
   additionally aid in debugging, as one can clearly see where the side effects
   occur and step through them.
 - Split trees at *all* points. This essentially puts every instruction on its
   own line, which may sometimes be useful for single-step debugging scenarios,
   or for compiler writers.
 - Don't split trees at all. Maximum bushiness.

Each of these strategies map back to the same binary format. A single text
format can support a wide variety of use cases, because binary->text
translators can split up trees to fit the need at hand.


## Details

Expressions containing multiple pops perform their pops right-to-left. This is
surprising at first, but it makes sense when you look at wasm's evaluation order.
For example:

```
   push:0 call $foo()
   push:1 call $bar()
   call $qux(pop:0, pop:1)
```

Clearly, this syntax should evaluate the call to `$foo` before the call to
`$bar`. And in the wasm semantics, the call to `$qux` evaluates its operands in
the order they appear. Both of these principles are completely intuitive. Put
together as they are here, they imply that the first pop corresponds to the
first push, which effectively means that the pops happen right-to-left.

The `:0` and `:1` are stack-depth indicators, which can be useful in pairing
up pushes with their corresponding pops.

Some additional rules governing push and pop are:

 - Pushed expressions must be popped within the same block as the push.
 - Stack-depth indicators start at 0 at the beginning of each block.
 - Sequences of trees tied together with push and pop must be contiguous.
   Arbitrary blocks can be placed in the middle of trees, but their return value
   has to be consumed by some node in the tree.

These rules reflect how the current wasm binary format works. If there are
changes to wasm, these rules would change accordingly.


## Answers to anticipated questions

Q: How about replacing push/pop with something more flexible?

A: Push/pop as described here are meant to be a direct reflection of WebAssembly
   itself. For example, it would be convenient to replace `push` with
   something that would allow a value to be used multiple times. However,
   push/pop are representing expression tree edges in WebAssembly, which
   can only have a single definition and a single use. The way to use a value
   multiple times in WebAssembly is to use `set_local` and `get_local`.


