# Text Format

The purpose of this text format is to support:
* View Source on a WebAssembly module, thus fitting into the Web (where every
  source can be viewed) in a natural way.
* Presentation in browser development tools when source maps aren't present
  (which is necessarily the case with [the Minimum Viable Product (MVP)](MVP.md)).
* Working with WebAssembly code directly for reasons including pedagogical,
  experimental, debugging, profiling, optimization, and testing of the spec
  itself.

The text format is equivalent and isomorphic to the [binary format](BinaryEncoding.md).

The text format will be standardized, but only for tooling purposes; browsers
will not parse the textual format on regular web content in order to implement
WebAssembly semantics.

The text format does not use JavaScript syntax; it is not intended to
be evaluated or translated directly into JavaScript. There are also
substantive reasons to use notation that is different than JavaScript (for
example, WebAssembly has a 32-bit integer type, and it is represented
in the text format, since that is the natural thing to do for WebAssembly,
regardless of JavaScript not having such a type). On the other hand,
when there are no substantive reasons and the options are basically
bikeshedding, then it does make sense for the text format to match existing
conventions on the Web (for example, curly braces, as in JavaScript and CSS).

The text format isn't uniquely representable. Multiple textual files can assemble
to the same binary file, for example whitespace isn't relevant and memory initialization
can be broken out into smaller pieces in the text format.

The text format is precise in that values that cannot be accurately represented in the
binary format are considered invalid text. Floating-point numbers are therefore
represented as hexadecimal floating-point as specified by the C99 standard, which
IEEE-754-2008 section 5.12.3 also specifies. The textual format may be improved to also
support more human-readable representations, but never at the cost of accurate representation.

# Official Text Format

## Warning: this is an experiment.

This document has not been submitted to any official WebAssembly forum.
It is not known at this time whether it ever will be, and if it is, it
may be with significant changes.


## Philosophy:

 - Use JS-style sensibilities when there aren't reasons otherwise.
 - It's a compiler target, not a programming language, but readability still counts.


## High-level summary:

 - Curly braces for function bodies, blocks, etc., `/* */`-style and `//`-style
   comments, and whitespace is not significant. Also, no semicolons.
   (TODO: Should `/* */`-style comments nest properly?)

 - `get_local` looks like a simple reference; `set_local` looks like an
   assignment. Constants use a simple literal syntax. This makes wasm's most
   frequent opcodes very concise.

 - Infix syntax for arithmetic, with simple overloading. Explicit grouping via
   parentheses. Concise and familiar with JS and others. (TODO: Use C/JS-style
   operator precedence, or fix
   [an old mistake](http://www.lysator.liu.se/c/dmr-on-or.html)?)

 - Prefix syntax with comma-separated operands for all other operators. For less
   frequent opcodes, prefer just presenting operator names, so that they're easy
   to identify.

 - Typescript-style `name : type` declarations.

 - Parentheses around call arguments, eg. `call $functionname(arg, arg, arg)`,
   and `if` conditions, eg. `if ($condition) { call $then() } else { call $else() }`,
   because they're familiar to many people and not too intrusive.

 - Allow highly complex trees to be syntactically split up into readable parts.

 - Put labels "where they go".


## Examples:

### Basics

```
  function $fac-opt ($a:i64) : (i64) {
    var $x:i64
    $x = 1
    {
      br_if $end ? $a <s 2
      loop $loop {
        $x = $x * $a
        $a = $a + -1
        br_if $loop ? $a >s 1
      }
      $end:
    }
    $x
  }
```

(hand-translated from [fac.wast](https://github.com/WebAssembly/spec/blob/master/ml-proto/test/fac.wast))

The function return type has parentheses for symmetry with the parameter types,
anticipating adding multiple return values to wasm in the future.

The curly braces around the function body are not a `block` node; they are part
of the function syntax, reflecting how function bodies in wasm are block-like.

The last expression of the function body here acts as its return value. This
works in all block-like constructs (`block`, function body, `if`, etc.)

`>s` means *signed* greater-than. explicit unsigned or signed operators will be
suffixed with 'u' or 's', respectively.

The `$` sigil on user names cleanly ensures that they never collide with wasm
keywords, present or future.

`br_if` uses a question mark to announce the condition operand. `select` does
also. (TODO: Is this too cute?)

### Linear memory addresses

```
  function $test_redundant_load () : (i32) {
    i32.load [8,+0]
    f32.store [5,+0], -0x0p0
    i32.load [8,+0]
  }
```

(hand-translated from [memory_redundancy.wast](https://github.com/WebAssembly/spec/blob/master/ml-proto/test/memory_redundancy.wast))

Addresses are printed as `[base,+offset]`. It could be shortened to `[base]` when
there is no offset; I made the offset explicit above just to illustrate the syntax.
There can also be an optional `:align=…` for non-natural alignments.

### A slightly larger example:

Here's some C code:

```
  float Q_rsqrt(float number)
  {
      long i;
      float x2, y;
      const float threehalfs = 1.5F;

      x2 = number * 0.5F;
      y  = number;
      i  = *(long *) &y;
      i  = 0x5f3759df - (i >> 1);
      y  = *(float *) &i;
      y  = y * (threehalfs - (x2 * y * y));
      y  = y * (threehalfs - (x2 * y * y));

      return y;
  }
```

Here's the corresponding LLVM wasm backend output + binaryen + slight tweaks:

```
  (func $Q_rsqrt (param $0 f32) (result f32)
    (local $1 f32)
    (set_local $1
      (f32.reinterpret/i32
        (i32.sub
          (i32.const 1597463007)
          (i32.shr_s
            (i32.reinterpret/f32
              (get_local $0))
            (i32.const 1)))))
    (set_local $1
      (f32.mul
        (get_local $1)
        (f32.sub
          (f32.const 1.5)
          (f32.mul
            (get_local $1)
            (f32.mul
              (get_local $1)
              (set_local $0
                (f32.mul
                  (get_local $0)
                  (f32.const 0.5))))))))
    (f32.mul
      (get_local $1)
      (f32.sub
        (f32.const 1.5)
        (f32.mul
          (get_local $1)
          (f32.mul
            (get_local $0)
            (get_local $1)))))
   )
```

And here's the proposed text syntax:

```
   function $Q_rsqrt ($0:f32) : (f32) {
     var $1:f32
     $1 = f32.reinterpret/i32 (1597463007 - ((i32.reinterpret/f32 $0) >> 1))
     push:0 $0 = $0 * 0x1p-1
     $1 = $1 * (0x1.8p0 - $1 * pop:0 * $1)
     $1 * (0x1.8p0 - $1 * $0 * $1)
   }
```

This shows off the compactness of infix operators with overloading. In the
s-expression syntax, these expressions are quite awkward to read, and this
isn't even a very big example. But the text syntax here is very short.

This also introduces the push and pop mechanism for splitting up expression
trees. Push and pop connect subtrees to their parents, allowing them to be
written separately in the text syntax, but still be part of the same
conceptual tree in the wasm semantics, and in the wasm binary format.

In particular, note that the s-expression version has a `set_local` buried in
the middle of a tree, making it easy for a human to miss. Humans wouldn't write
code that way, but in wasm, compilers are *incentivised* to write it that way,
because it reduces code size. It's going to happen a lot, and the push/pop
mechanism gives us a way to make this more readable in many cases.

See [below](#pushpop) for more information.

### Labels

Excerpt from labels.wast:

```
  (func $loop3 (result i32)
    (local $i i32)
    (set_local $i (i32.const 0))
    (loop $exit $cont
      (set_local $i (i32.add (get_local $i) (i32.const 1)))
      (if (i32.eq (get_local $i) (i32.const 5))
        (br $exit (get_local $i))
      )
      (get_local $i)
    )
  )
```

Corresponding proposed text syntax:

```
  function $loop3 () : (i32) {
    var $i:i32
    $i = 0
    loop $cont {
      $i = $i + 1
      if ($i == 5) {
        br $exit, $i
      }
    $exit:
    }
  }
```

Note that the curly braces are part of the `if`, rather than introducing a
block. This reflects how `if` essentially provides `block`-like capabilities
in the wasm binary format.

### Nested blocks

Label definitions, like the `$exit:` above, introduce additional blocks nested
within the nearest `{`, without requiring their own `{`. This allows the deep
nesting of `br_table` to be printed in a relatively flat manner:

```
  {
    br_table [$red, $orange, $yellow, $green], $default, $index
    $red:
      // ...
    $orange:
      // ...
    $yellow:
      // ...
    $green:
      // ...
    $default:
      // ...
  }
```

representing the following in nested form:

```

  (block $default
    (block $green
      (block $yellow
        (block $orange
          (block $red
            (br_table [$red, $orange, $yellow, $green] $default (get_local $index))
          )
          // ...
        )
        // ...
      )
      // ...
    )
    // ...
  )
```

`br_table`s can have large numbers of labels, so this feature allows us to
avoid very deep nesting in many cases.


## Push and pop

Normally, the preferred way to split up a large expression tree would be to
simply assign some subtrees to their own local variables. Of course compilers
can optimize them away as needed.

However, in wasm, introducing locals like that increases code size, so
compilers producing wasm aren't going to do that. There will be a lot of code
in the wild with very large monolithic trees. Binary->text translation can't
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


## Push and pop details

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


## Operators with special syntax

As mentioned earlier, basic arithmetic operators use an infix notation, some
operators require explicit parentheses, and some operators use `?` to introduce
boolean conditions. The following is a table of special syntax:


## Control flow operators ([described here](https://github.com/WebAssembly/design/blob/master/AstSemantics.md))

| Name | Syntax | Examples
| ---- | ---- | ---- |
| `block` | *label*: | `{ br $a a: }`
| `loop` | `loop` *label* `{` … `}` | `loop $a { br $a }`
| `if` | `if` (*expr*) `{` *expr** `}` | `if (0) { 1 }`
| `if_else` | `if` (*expr*) `{` *expr** `} else {` *expr**`}` | `if (0) { 1 } else { 2 }`
| `select` | `select` *expr*, *expr* ? *expr* | `select 1, 2 ? $x < $y`
| `br` | `br` *label* | `br $a`
| `br_if` | `br` *label* `?` *expr* | `br $a`, `br $a ? $x < $y`
| `br_table` | `br_table [` *case-label* `,` … `] ,` *default-label* `,` *expr* | `br_table [$x, $y], $z, 0`

(TODO: as above, are the `?`s too cute?)

## Basic operators ([described here](https://github.com/WebAssembly/design/blob/master/AstSemantics.md#constants))

| Name | Syntax | Example
| ---- | ---- | ---- |
| `i32.const` | … | `234`, `0xfff7`
| `i64.const` | … | `234`, `0xfff7`
| `f64.const` | … | `0.1p2`, `infinity`, `nan:0x789`
| `f32.const` | … | `0.1p2`, `infinity`, `nan:0x789`
| `get_local` | *name* | `$x + 1`
| `set_local` | *name* `=` *expr* | `$x = 1`
| `call` | `call` *name* `(`*expr* `,` … `)` | `call $min(0, 2)`
| `call_import` | `call_import` *name* `(`*expr* `,` … `)` | `call_import $max(0, 2)`
| `call_indirect` | `call_indirect` *signature-name* `[` *expr* `] (`*expr* `,` … `)` | `call_indirect $foo [1] $min(0, 2)`

## Memory-related operators ([described here](https://github.com/WebAssembly/design/blob/master/AstSemantics.md#linear-memory-accesses))

| Name | Syntax | Example
| ---- | ---- | ---- |
| *memory-immediate* | `[` *base-expression* `,` *offset* `]` | `[$base, 4]`
| `i32.load8_s` | `i32.load8_s [` *base-expression* `, +` *offset-immediate* `]` | `i32.load8_s [$base, +4]`
| `i32.load8_s` | `i32.load8_s [` *base-expression* `, +` *offset-immediate* `]:align=` *align* | `i32.load8_s [$base, +4]:align=2`
| `i32.store8` | `i32.store8 [` *base-expression* `, +` *offset-immediate* `]`, *expr* | `i32.store8 [$base, +4], $value`
| `i32.store8` | `i32.store8 [` *base-expression* `, +` *offset-immediate* `]:align=` *align* `,` *expr* | `i32.store8 [$base, +4]:align=2, $value`

The other forms of `load` and `store` are similar.

## Simple operators ([described here](AstSemantics#32-bit-integer-operators))

| Name | Syntax |
| ---- | ---- |
| `i32.add` | … `+` …
| `i32.sub` | … `-` …
| `i32.mul` | … `*` …
| `i32.div_s` | … `/s` …
| `i32.div_u` | … `/u` …
| `i32.rem_s` | … `%s` …
| `i32.rem_u` | … `%u` …
| `i32.and` | … `&` …
| `i32.or` | … `|` …
| `i32.xor` | … `^` …
| `i32.shl` | … `<<` …
| `i32.shr_u` | … `>>u` …
| `i32.shr_s` | … `>>s` …
| `i32.eq` | … `==` …
| `i32.ne` | … `!=` …
| `i32.lt_s` | … `<s` …
| `i32.le_s` | … `<=s` …
| `i32.lt_u` | … `<u` …
| `i32.le_u` | … `<=u` …
| `i32.gt_s` | … `>s` …
| `i32.ge_s` | … `>=s` …
| `i32.gt_u` | … `>u` …
| `i32.ge_u` | … `>=u` …
| `i32.eqz` | `!` …
| `i64.add` | … `+` …
| `i64.sub` | … `-` …
| `i64.mul` | … `*` …
| `i64.div_s` | … `/s` …
| `i64.div_u` | … `/u` …
| `i64.rem_s` | … `%s` …
| `i64.rem_u` | … `%u` …
| `i64.and` | … `&` …
| `i64.or` | … `\|` …
| `i64.xor` | … `^` …
| `i64.shl` | … `<<` …
| `i64.shr_u` | … `>>u` …
| `i64.shr_s` | … `>>s` …
| `i64.eq` | … `==` …
| `i64.ne` | … `!=` …
| `i64.lt_s` | … `<s` …
| `i64.le_s` | … `<=s` …
| `i64.lt_u` | … `<u` …
| `i64.le_u` | … `<=u` …
| `i64.gt_s` | … `>s` …
| `i64.ge_s` | … `>=s` …
| `i64.gt_u` | … `>u` …
| `i64.ge_u` | … `>=u` …
| `i64.eqz` | `!` …
| `f32.add` | … `+` …
| `f32.sub` | … `-` …
| `f32.mul` | … `*` …
| `f32.div` | … `/` …
| `f32.neg` | `-` …
| `f32.eq` | … `==` …
| `f32.ne` | … `!=` …
| `f32.lt` | … `<` …
| `f32.le` | … `<=` …
| `f32.gt` | … `>` …
| `f32.ge` | … `>=` …
| `f64.add` | … `+` …
| `f64.sub` | … `-` …
| `f64.mul` | … `*` …
| `f64.div` | … `/` …
| `f64.neg` | `-` …
| `f64.eq` | … `==` …
| `f64.ne` | … `!=` …
| `f64.lt` | … `<` …
| `f64.le` | … `<=` …
| `f64.gt` | … `>` …
| `f64.ge` | … `>=` …

All other operators use their actual name in a prefix notation, such as
`f32.sqrt …`.

## Answers to anticipated questions

Q: JS avoids sigils, and uses context-sensitive keywords to avoid trouble.
   Can wasm do this?

A: Sigils are more of a burden when writing code than reading code, and wasm
   will mostly be written by compilers. And it's my subjective opinion that
   it's better to give ourselves maximum flexibility to add new keywords in
   the future without having to be tricky.


Q: Why not let `br` be spelled `break` or `continue` when targeting block and
   loop, respectively?

A: The `br_table` construct has multiple labels, and there may be a mix of
   forward and backward branches, so it isn't always possible to categorize
   branches as forward or backward. Also, `br`, `br_if`, and `br_table` are
   what we have in the spec, so using their actual names avoids needing
   to special-case them.


Q: Why not permit optional semicolons?

A: We don't want people arguing over which way is better. If we don't forbid
   semicolons, the next best option would be to require semicolons. I've
   subjectively chosen to forbid semicolons for now.


# Debug symbol integration

The binary format inherently strips names from functions, locals, globals, etc,
reducing each of these to dense indices. Without help, the text format must
therefore synthesize new names. However, as part of the [tooling](Tooling.md)
story, a lightweight, optional "debug symbol" global section may be defined
which associates names with each indexed entity and, when present, these names
will be used in the text format projected from a binary WebAssembly module.
