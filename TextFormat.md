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

# ~~Official~~*Experimental* Text Format

## This is an experiment!

This document is a sketch of a possible Text Format proposal for WebAssembly to
use for the "View Source" functionality in browsers. WebAssembly looks enough
like a programming language that it tends to activate our programmer intuitions
about syntax, but it differs from normal programming languages in numerous
respects, so we don't fully trust our intuitions.

So, we're sketching something up, and building a trial implementation of it in
Firefox. This way, we can try it out on real code in a real browser setting, and
see if it actually "works" in practice. Maybe we'll like it and propose it to
the official WebAssembly project. Maybe it'll need changes. Or maybe it'll
totally flop and we'll drop it and pursue something completely different!

Comments, questions, suggestions, and reactions are welcome on
[this repo's issue tracker](https://github.com/sunfishcode/design/issues) for
the moment. As the experiment progresses, we may shift to other discussion
forums, but for now we're keeping it simple.


## Philosophy:

 - Use JS-style sensibilities when there aren't reasons otherwise.
 - It's a compiler target, not a programming language, but readability still counts.

## High-level summary:

 - Curly braces for function bodies, blocks, etc., `/* */`-style and `//`-style
   comments, and whitespace is not significant.
   (TODO: Should `/* */`-style comments nest properly?)

 - `get_local` looks like a simple reference; `set_local` looks like an
   assignment. Constants use a simple literal syntax. This makes wasm's most
   frequent opcodes very concise.

 - Infix syntax for arithmetic, with simple overloading. Explicit grouping via
   parentheses. Concise and familiar with JS and others. (TODO: Use C/JS-style
   operator precedence, or fix 
   [an old mistake](http://www.lysator.liu.se/c/dmr-on-or.html)?)

 - Prefix syntax with operands in parentheses for most other operators (e.g. 
   `i32.rotl($0, 8)`). For less frequent opcodes, prefer just presenting operator
   names, so that they're easy to identify.

 - Typescript-style `name : type` declarations.

 - Parentheses around call arguments, eg. `$functionname(arg, arg, arg)`,
   and `if` conditions, eg. `if ($condition) { $then() } else { $else() }`,
   because they're familiar to many people and not too intrusive.

 - Allow highly complex trees to be syntactically split up into readable parts.

 - Put labels "where they go".

 - The text format will be compatible with the [LES](http://loyc.net/les) text 
   format. It _is not_ compatible with the current LES specification, but LES
   is in beta and can still be tweaked to wasm's needs. Based on the wasm text 
   format, a third version of LES (LESv3) will be drafted before the end of 2016.
   Meanwhile, the wasm text format will be syntactically constrained in such a 
   way that it will be an appropriate basis for LESv3. For the MVP, parsers of 
   the wasm text format will be able to choose whether to use a custom parser 
   dedicated to wasm or a generic LES parser.

 - TODO: should semicolons should be required at the end of each expression 
   in a block? If newlines are the primary separator, then LES will cease to 
   be a superset of JSON (since JSON ignores newlines), but there are benefits
   on the flip side (such as eliminating the need for semicolons!). In this 
   document it is assumed that a newline **does** mark the end of an 
   expression if the newline does not appear directly inside parentheses (as 
   inside parentheses, expressions are always terminated by commas or by a 
   closing parenthesis). In any case it would be useful to _allow_ semicolons,
   so that one can write multiple expressions on a single line.

## Examples:

### Basics

```
  function $@fac-opt($a:i64) : i64 {
    $x:i64
    $x = 1
    br_if end ? $a < 2
    loop $loop {
      $x = $x * $a
      $a = $a + -1
      br_if loop ? $a > 1
    }
    :end
    $x
  }
```

(hand-translated from [fac.wast](https://github.com/WebAssembly/spec/blob/master/ml-proto/test/fac.wast))

The `$` sigil on function and variable names cleanly ensures that they never 
collide with wasm keywords, present or future. The `@` sign on `fac-opt` allows 
certain special characters to appear in identifiers, such as `-` which would 
otherwise be treated as a subtraction operator.

The function return type can have parentheses (`: (i64)`) for symmetry with the 
parameter types, since we anticipate adding multiple return values to wasm in the
future, but they are not required.

The curly braces around the function body are not a `block` node; they are part
of the function syntax, reflecting how function bodies in wasm are block-like.

The last expression of the function body here acts as its return value. This
works in all block-like constructs (`block`, function body, `if`, etc.)

`>` means *signed* greater-than. Unsigned operators will have a `|` before the last character of the operator, so `|>` is *unsigned* greater-than.

`br_if` uses a question mark to announce the condition operand. `select` does
also. (TODO: Is this too cute? Also, should the order be reversed as in 
`br_if $a < 2 ? end`?)

### Linear memory addresses

```
  function $test_redundant_load() : (i32) {
    i32.load [8,+0]
    f32.store [5,+0] = -0x0p0
    i32.load [8,+0]
  }
```

(hand-translated from [memory_redundancy.wast](https://github.com/WebAssembly/spec/blob/master/ml-proto/test/memory_redundancy.wast))

Addresses are printed as `[base,+offset]`. It could be shortened to `[base]` when
there is no offset; I made the offset explicit above just to illustrate the syntax.
There can also be an optional `align …` for non-natural alignments, e.g. 
`i32.load [8,+0, align 2]`.

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
   function $Q_rsqrt($0:f32) : (f32) {
     $1:f32
     $1 = f32.reinterpret'i32(1597463007 - (i32.reinterpret'f32($0) >> 1))
     $1 = $1 * (0x1.8p0 - $1 * ($0 = $0 * 0x1p-1) * $1)
     $1 * (0x1.8p0 - $1 * $0 * $1)
   }
```

This shows off the compactness of infix operators with overloading. In the
s-expression syntax, these expressions are quite awkward to read, and this
isn't even a very big example. But the text syntax here is very short.

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
    $i:i32
    $i = 0
    loop $cont {
      $i = $i + 1
      if ($i == 5) {
        br exit => $i
      }
      :exit
    }
  }
```

Note that the curly braces are part of the `if`, rather than introducing a
block. This reflects how `if` essentially provides `block`-like capabilities
in the wasm binary format.

Due to syntactic requirements of LES, the colon `:` appears before the label 
name (`:exit`) rather than afterward.

### Nested blocks

Label definitions that do not appear at the end of the enclosing block, such as
the `:exit` above, introduce additional blocks nested within the nearest `{`, 
without requiring their own `{`. This allows the deep nesting of `br_table` to 
be printed in a relatively flat manner:

```
  {
    br_table [red, orange, yellow, green, default] : $index
    :red
      // ...
    :orange
      // ...
    :yellow
      // ...
    :green
      // ...
    :default
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

## Pushing and popping

## Operators with special syntax

As mentioned earlier, basic arithmetic operators use an infix notation, some
operators require explicit parentheses, and some operators with boolean 
conditions use `?`. The following is a table of special syntax:

## Control flow operators ([described here](https://github.com/WebAssembly/design/blob/master/AstSemantics.md))

| Name       | Syntax                     | Examples 
| ---------- | -------------------------- | -------- 
| `block`    | :*label*                   | `{ br a; :a }`
| `loop`     | `loop` *label* `{` … `}`   | `loop a { br a }`
| `if`       | `if (`*expr*`)` `{` *expr** `}` | `if ($x) { $f($x) }`
| `if_else`  | `if (`*expr*`)` `{` *expr** `} else {` *expr** `}` | `if (0) { 1 } else { 2 }`
| `select`   | `select` *expr* `:` *expr* `?` *expr*`)` | `select 1 : 2 ? $x < $y`
| `br`       | `br` *label* [=> $result]    | `br a`, `br a => $x`
| `br_if`    | `br_if` *label* `(if` *expr*`)` [`=>` *expr*] | `br a (if $x < $y) => 0`
| `br_table` | `br_table {` *case-label* `,` … `,` *default-label*] `} from` *expr* | `br_table [a, b, c] : $x`

(TODO: as above, are the `?`s too cute?)

## Basic operators ([described here](https://github.com/WebAssembly/design/blob/master/AstSemantics.md#constants))

| Name        | Syntax      | Example
| ----------- | ----------- | ---- |
| `i32.const` | see example | `234`, `0xfff7`
| `i64.const` | see example | `234L`, `0xfff7L`
| `f64.const` | see example | `0.1p2`, `@inf`, `@nan'0x789`
| `f32.const` | see example | `0.1p2f`, `@inf_f`, `@nan'0x789`
| `get_local` | *name* (including the `$`) | `$x`
| `set_local` | *name* `=` *expr* | `$x = 1`
| `call`      | *name* `(`*expr* `,` … `)` | `$min(0, 2)`
| `call_import` | `$` *name* `(`*expr* `,` … `)` | `$$max(0, 2)`
| `call_indirect` | *expr* `::` *signature-name* [`[` *expr* `]`] `(`*expr* `,` … `)` | `$func::$signature(0, 2)`

## Memory-related operators ([described here](https://github.com/WebAssembly/design/blob/master/AstSemantics.md#linear-memory-accesses))

| Name | Syntax | Example
| ---- | ---- | ---- |
| *memory-immediate* | `[` *base-expression* `,` *offset* `]` | `[$base, 4]`
| `i32.load8_s` | `i32.load8_s [` *base-expression* `, +` *offset-immediate* `]` | `i32.load8_s [$base, +4]`
| `i32.load8_s` | `i32.load8_s [` *base-expression* `, +` *offset-immediate* `, align ` *align* `]` | `i32.load8_s [$base, +4, align 2]`
| `i32.store8` | `i32.store8 [` *base-expression* `, +` *offset-immediate* `]`, *expr* | `i32.store8 [$base, +4], $value`
| `i32.store8` | `i32.store8 [` *base-expression* `, +` *offset-immediate* `, align ` *align* `]` `=` *expr* | `i32.store8 [$base, +4, align 2] = $value`

The other forms of `load` and `store` are similar.

## Simple operators ([described here](AstSemantics#32-bit-integer-operators))

| Name | Syntax |
| ---- | ---- |
| `i32.add` | … `+` …
| `i32.sub` | … `-` …
| `i32.mul` | … `*` …
| `i32.div_s` | … `/` …
| `i32.div_u` | … `|/` …
| `i32.rem_s` | … `%` …
| `i32.rem_u` | … `|%` …
| `i32.and` | … `&` …
| `i32.or` | … `|` …
| `i32.xor` | … `^` …
| `i32.shl` | … `<<` …
| `i32.shr_s` | … `>>` …
| `i32.shr_u` | … `>|>` …
| `i32.eq` | … `==` …
| `i32.ne` | … `!=` …
| `i32.lt_s` | … `<` …
| `i32.le_s` | … `<=` …
| `i32.lt_u` | … `|<` …
| `i32.le_u` | … `<|=` …
| `i32.gt_s` | … `>` …
| `i32.ge_s` | … `>=` …
| `i32.gt_u` | … `|>` …
| `i32.ge_u` | … `>|=` …
| `i32.eqz` | `!` …
| `i64.add` | … `+` …
| `i64.sub` | … `-` …
| `i64.mul` | … `*` …
| `i64.div_s` | … `/` …
| `i64.div_u` | … `|/` …
| `i64.rem_s` | … `%` …
| `i64.rem_u` | … `|%` …
| `i64.and` | … `&` …
| `i64.or` | … `\|` …
| `i64.xor` | … `^` …
| `i64.shl` | … `<<` …
| `i64.shr_s` | … `>>` …
| `i64.shr_u` | … `>|>` …
| `i64.eq` | … `==` …
| `i64.ne` | … `!=` …
| `i64.lt_s` | … `<` …
| `i64.le_s` | … `<=` …
| `i64.lt_u` | … `|<` …
| `i64.le_u` | … `<|=`  …
| `i64.gt_s` | … `>` …
| `i64.ge_s` | … `>=` …
| `i64.gt_u` | … `|>` …
| `i64.ge_u` | … `>|=` …
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


Q: Why is, for example, the unsigned shift operator called `>|>` rather than 
   the more logical `|>>`, or even `|>>|`?

A: None of the "unsigned" operators are built into LES. The precedence of 
   non-built-in operators is derived in a predictable way from the built-in
   operators, so that for example `>|>` has the same precedence as `>>`,
   whereas `|>>` has the same precedence as `>`, and `|>>|` has the same 
   precedence as `||`. Placing the vertical bar in the middle allows the 
   operator to keep the same precedence as the built-in operator.
   

# Debug symbol integration

The binary format inherently strips names from functions, locals, globals, etc,
reducing each of these to dense indices. Without help, the text format must
therefore synthesize new names. However, as part of the [tooling](Tooling.md)
story, a lightweight, optional "debug symbol" global section may be defined
which associates names with each indexed entity and, when present, these names
will be used in the text format projected from a binary WebAssembly module.

Since LES allows "attribute" expressions to be attached to any expression, 
these could be used someday to represent additional debug information, 
comments, or other "side-channel" information that may be stored in the 
binary format in the future.
