# Text Format

The purpose of this text format is to support:
* View Source on a WebAssembly module, thus fitting into the Web (where every
  source can be viewed) in a natural way.
* Presentation in browser development tools when source maps aren't present
  (which is necessarily the case with [the Minimum Viable Product (MVP)](MVP.md)).
* Writing WebAssembly code directly for reasons including pedagogical,
  experimental, debugging, optimization, and testing of the spec itself.

The text format is equivalent and isomorphic to the [binary format](BinaryEncoding.md).

The text format will be standardized, but only for tooling purposes:
* Compilers will support this format for `.S` and inline assembly.
* Debuggers and profilers will present binary code using this textual format.
* Browsers will not parse the textual format on regular web content in order to
  implement WebAssembly semantics.

Given that the code representation is actually an
[Abstract Syntax Tree](AstSemantics.md), the syntax contains nested
expressions (instead of the linear list of instructions most
assembly languages have).

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

## Philosophy:

 - Reflect the wasm language obviously, rather than being a distinct language.
 - Be concise, but stay readable.
 - Use JS-style sensibilities as a guide, though only a guide.


## High-level ideas:

 - Infix arithmetic operators with simple type inference are concise and
   familiar to most audiences.

 - Large and deep expression trees are difficult to read either as monolithic
   trees or as fully flattened sequences. A middle road is possible: have
   expression trees, but also have a purely syntactic way to split up complex
   trees.

 - Putting block labels and loop-exit labels at the bottoms of blocks and loops
   is more readable than putting them at the top -- that way they indicate
   "where to go" when it goes to that label.

 - Even if wasm adds multiple return values, expressions returning a single
   value are very common and syntactically easy. Optimize for concise
   representation of this case, but still provide a way to support multiple
   return values in the future.

 - `get_local` and `set_local` are *very* common; the syntax is greatly
   decluttered by making `get_local` implicit, and using a compact
   assignment-like syntax for `set_local`.

 - Curly braces, /* */-style, //-style comments, and '=' for assignment are
   familiar to many audiences, including JS programmers, and are reasonably
   concise. However, semicolons are passé.

 - Don't worry too much about parsing speed or minifiability; that's what the
   binary format is for.


## Examples:

### Excerpt from fac.wast:

```
  (func $fac-opt (param $a i64) (result i64)
    (local $x i64)
    (set_local $x (i64.const 1))
    (block $end
      (br_if $end (i64.lt_s (get_local $a) (i64.const 2)))
      (loop
        (set_local $x (i64.mul (get_local $x) (get_local $a)))
        (set_local $a (i64.add (get_local $a) (i64.const -1)))
        (br_if 0 (i64.gt_s (get_local $a) (i64.const 1)))
      )
    )
    (get_local $x)
  )
```

Proposed text syntax:

```
  func $fac-opt ($a:i64) : (i64) {
    var $x:i64
    $x = 1
    {
      br_if $end
      $loop : {
        $x = $x * $a
        $a = $a + -1
        br_if $loop, $a >s 1
      }
    } : $end
    $x
  }
```

The return type has parentheses for symmetry with the parameter types, to
anticipate the possibility of adding multiple return values to wasm in the
future.

The curly braces around the function body are not a separate `block` node; they
are part of the function syntax, reflecting how function bodies in wasm are
block-like.

`>s` means signed greater-than. explicit unsigned or signed operators will be
suffixed with 'u' or 's', respectively.

The last expression of the function body here acts as its return value. This
works in all block-like constructs (`block`, function body, `if`, etc.)

The `$` sigil on user names cleanly ensures that they never collide with wasm
keywords.

### Excerpt from memory_redundancy.wast:

```
  (func $test_redundant_load (result i32)
    (i32.load (i32.const 8))
    (f32.store (i32.const 5) (f32.const -0.0))
    (i32.load (i32.const 8))
  )
```

Corresponding proposed text syntax:

```
  func $test_redundant_load () : (i32) {
    i32.load [8,+0]
    f32.store [5,+0], -0.0
    i32.load [8,+0]
  }
```

Addresses are printed as `[base,+offset]`. It could be shortened to `[base]` when
there is no offset; I made the offset explicit above to illustrate the syntax.
There could also be an optional `/align` or so in there as well for non-natural
alignments.

### A C code example:

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

Corresponding LLVM wasm backend output + binaryen + slight tweaks:

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

Corresponding proposed text syntax:

```
   func $Q_rsqrt ($0:f32) : (f32) {
     var $1:f32
     $1 = f32.reinterpret/i32 (1597463007 - ((i32.reinterpret/f32 $0) >> 1))
     push $0 = $0 * 0.5
     $1 = $1 * (1.5 - $1 * pop * $1)
     $1 * (1.5 - $1 * $0 * $1)
   }
```

This is a good demonstration of the compactness of the infix syntax, and the
awkwardness of deep trees -- these trees are awkward and they aren't even very
deep!

This also introduces the push/pop mechanism for splitting up expression trees.
Push/pop connect two trees, allowing them to be written separately in the text
syntax, but still be part of the same conceptual tree in the wasm semantics, and
in the wasm binary format.

Note that the s-expression version has a `set_local` buried in the middle of a
tree, making it easy for a human to miss. The proposed text format splits out
the `set_local` operators (into assignment-like syntax, for conciseness),
putting them at the top level for the benefit of humans, and uses push/pop to
express the fact that they're all technically still part of the same tree in
wasm.

More broadly, trees are split to put `set_local` at the top level, and to avoid
having multiple nodes with "side effects" in the same tree. This tends to keep
trees to a manageable size, and is also a nice organization for stepping through
in a debugger.

### Excerpt from labels.wast:

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
  func $loop3 () : (i32) {
    var $i:i32
    $i = 0
    $cont : {
      $i = $i + 1
      if $i == 5 {
        push $i
        br $exit
      }
    } : $exit
  }
```

Note that the curly braces are part of the `if`, rather than introducing a
block. This reflects how `if` essentially provides `block`-like capabilities
in the wasm binary format.

This also demonstrates using the "push" syntax for a value live out of a block
in a manner other than the block's own return value. Instead of making `br`'s
value operand a syntactic operand of the `br`, we can use the push/pop
mechanism. This splits up the `br`, making it easier to read, and cleanly
anticipates a future where there are multiple values being returned. Note that
the corresponding pop in this case is implicit in the function return here,
though in other contexts it could be explicit.

## Miscellaneous notes:

`block` and `loop` are only distinguished by whether they have a begin label.
This gently guides readers away from confusion about how "loop doesn't loop",
and also reflects how similar `loop` and `block` are in wasm. Function and
`if` bodies also use similar syntax, reflecting that they have very similar
behavior too.

push/pop can be awkward for people wanting to edit code in place, because it's
important to keep in mind that they're an alternative to deeper trees which
are also hard to edit in place.

Deeply nested structures are an inherent part of wasm, and come from a desire to
reduce code size even at some amount of readability cost. Push/pop in the text
format provide a way to mitigate this and make the code somewhat more
human-readable again.


## Push/pop rules

Expressions containing multiple pops perform their pops right-to-left. This
is surprising at first, but it makes sense when you look at how trees are
evaluated. For example:

```
   push $foo()
   push $bar()
   $qux(pop, pop)
```

Clearly, this syntax should evaluate the call to `$foo` before the call to
`$bar`. And in the wasm semantics, the call to `$qux` evaluates its operands in
the order they appear. Both of these principles are completely intuitive. Put
together as they are here, they imply that the first pop corresponds to the
first push, which effectively means that the pops happen right-to-left.

Another rule governing push/pop is that values must be popped within the
same block as the push, except in a one special circumstance: there can
optionally one value on the stack across the exit of a block-like construct
(`block`, `loop`, `if` arm, function body), provided it doesn't also use a
syntactic return value (this is a restriction in wasm itself, and may be
loosened in the future with support for multiple return values).

Another rule is that sequences of trees tied together with push/pop must be
contiguous. Arbitrary blocks can be placed in the middle of trees, but their
return value has to be consumed by some node in the tree.

## Some of the many bikesheds that will need painting:

 - Should /* */-style comments nest properly?
 - What operator precedence rules to use for the infix expressions?
   (C/JS-style, or should we fix
   [an old mistake](http://www.lysator.liu.se/c/dmr-on-or.html)?)
 - Significant newlines?
 - "func" vs "function" vs "routine"?
 - Remove the sigils from identifiers?
 - Add indices to push/pop, like `push0` and `pop0`, to help pair them up?
 - Invent a better syntax for load/store addresses?
 - End block/loop with "$label : }" instead of "} : $label"?
 - Represent backward `br`/`br_if` as "continue" and/or forward `br`/`br_if` as "break"?
 - Should a plain "continue" be a thing that automatically binds to the nearest loop?"
 - Pick a more concise syntax for conversion operators, like `f32.reinterpret/i32`?
 - Use `&-` for load/store alignment instead of `/`?


# Debug symbol integration

The binary format inherently strips names from functions, locals, globals, etc,
reducing each of these to dense indices. Without help, the text format must
therefore synthesize new names. However, as part of the [tooling](Tooling.md)
story, a lightweight, optional "debug symbol" global section may be defined
which associates names with each indexed entity and, when present, these names
will be used in the text format projected from a binary WebAssembly module.
