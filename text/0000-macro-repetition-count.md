- Feature Name: macro_repetition_count
- Start Date: 2020-06-07
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Add syntax to declarative macros to allow determination of the number of
metavariable repetitions.

# Motivation
[motivation]: #motivation

Macros with repetitions often expand to code that needs to know or could
benefit from knowing how many repetitions there are.  Consider the standard
sample macro to create a vector, recreating the standard library `vec!` macro:

```
macro_rules! myvec {
    ($($value:expr),* $(,)?) => {
        {
            let mut v = Vec::new();
            $(
                v.push($value);
            )*
            v
        }
    };
}
```

This would be more efficient if it could use `Vec::with_capacity` to
preallocate the vector with the correct length.  However, there is no standard
facility in declarative macros to achieve this.

There are various ways to work around this limitation.  Some common approaches
that users take are listed below, along with some of their drawbacks.

### Use recursion

Use a recursive macro to calculate the length.

```
macro_rules! count_exprs {
    () => {0usize};
    ($head:expr, $($tail:expr,)*) => {1usize + count_exprs!($($tail,)*)};
}

macro_rules! myvec {
    ($(value:expr),* $(,)?) => {
        {
            let size = count_exprs!($($value,)*);
            let mut v = Vec::with_capacity(size);
            $(
                v.push($value);
            )*
            v
        }
    };
}
```

Whilst this is among the first approaches that a novice macro programmer
might take, it is also the worst performing.  It rapidly hits the recursion
limit, and if the recursion limit is raised, it takes more than 25 seconds to
compile a sequence of 2,000 items.  Sequences of 10,000 items can crash
the compiler with a stack overflow.

### Generate a sum of 1s

This example is courtesy of @dtolnay.
Create a macro expansion that results in an expression like `0 + 1 + ... + 1`.
There are various ways to do this, but one example is:

```
macro_rules! myvec {
    ( $( $value:expr ),* $(,)? ) => {
        {
            let size = 0 { $( + { stringify!(value); 1 } ) };
            let mut v = Vec::with_capacity(size);
            $(
                v.push($value);
            )*
            v
        }
    };
}
```

This performs better than recursion, however large numbers of items still
cause problems.  It takes nearly 4 seconds to compile a sequence of 2,000
items.  Sequences of 10,000 items can still crash the compiler with a stack
overflow.

### Generate a slice and take its length

This example is taken from
[https://danielkeep.github.io/tlborm/book/blk-counting.html].  Create a macro
expansion that results in a slice of the form `[(), (), ... ()]` and take its
length.

```
macro_rules! replace_expr {
    ($_t:tt $sub:expr) => {$sub};
}

macro_rules! myvec {
    ( $( $value:expr ),* $(,)? ) => {
        {
            let size = <[()]>::len(&[$(replace_expr!(($value) ())),*]);
            let mut v = Vec::with_capacity(size);
            $(
                v.push($value);
            )*
            v
        }
    };
}
```

This is more efficient, taking less than 2 seconds to compile 2,000 items,
and just over 6 seconds to compile 10,000 items.

## Discoverability

Just considering the performance comparisons misses the point.  While we
can work around these limitations with carefully crafted macros, for a
developer unfamiliar with the subtleties of macro expansions it is hard
to discover which is the most efficient way.

Furthermore, whichever method is used, code readability is harmed by the
convoluted expressions involved.

The compiler already knows how many repetitions there are.  What is
missing is a way to obtain it.

This RFC adds syntax to allow this to be expressed directly:

```
macro_rules! myvec {
    ( $( $value:expr ),* $(,)? ) => {
        {
            let mut v = Vec::with_capacity($#value);
            $(
                v.push($value);
            )*
            v
        }
    };
}
```

The new "**metavariable count**" expansion `$#ident` expands to a literal
number equal to the number of times `ident` would be expanded at the depth
that it appears.

A prototype implementation indicates this compiles a 2,000 item sequence
in less than 1s, and a 10,000 item sequence in just over 2s.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The existing `vec!` based sample in the macros chapter of the Rust book can be
updated to use `Vec::with_capacity($#x)`.  The explanation would be updated
to:

> Now let’s look at the pattern in the body of the code associated with this
> arm: `temp_vec` is intialized with `Vec::with_capacity($#x`).  The `$#x`
> expands to the number of times the pattern matched `$x` at this point, which
> is 3.  Then `temp_vec.push()` with `$()*` is generated ...

> ... the code generated that replaces this macro call will be the following:

```
{
    let mut temp_vec = Vec::with_capacity(3);
    temp_vec.push(1);
    temp_vec.push(2);
    temp_vec.push(3);
    temp_vec
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This RFC adds a new macro syntax similar to a metavariable expansion:
`$#ident`.  This is currently not valid, so the addition of this behaviour is
backwards compatible.  The new expansion is termed the "metavariable count".

It is proposed that this syntax is added to both current `macro_rules!` macros
as well as new macros as defined in https://github.com/rust-lang/rust/issues/39412.

The use of `$#ident` in the RHS of a declarative macro expands to a single
literal number with the value of the number of repetitions of the identifier
at this depth of the macro expansion.

Use of `$#ident` with identifiers that do not repeat, or at a depth where
the specified identifier does not repeat, is an error.

In the simplest case, like the `myvec!` example above, where it is used at the
top level for an identifier in a single nesting of repetitions, it expands to
a literal number equal to the number of repetitions.

In the case of nested repetitions, the value depends on the depth of the
metavariable count expansion, where it expands to the number of repetitions
at that level.

Consider a more complex nested example:

```
macro_rules! nested {
    ( $( { $( { $( $x:expr ),* } ),* } ),* ) => {
        {
            println!("depth 0: {} repetitions", $#x);
            $(
                println!("  depth 1: {} repetitions", $#x);
                $(
                    println!("    depth 2: {} repetitions", $#x);
                    $(
                        println!("      depth 3: x = {}", $x);
                    )*
                )*
            )*
        }
    };
}
```

And given a call of:
```
   nested! { { { 1, 2, 3, 4 }, { 5, 6, 7 }, { 8, 9 } },
             { { 10, 11, 12 }, { 13, 14 }, { 15 } } };
```

This program will print:
```
depth 0: 2 repetitions
  depth 1: 3 repetitions
    depth 2: 4 repetitions
      depth 3: x = 1
      depth 3: x = 2
      depth 3: x = 3
      depth 3: x = 4
    depth 2: 3 repetitions
      depth 3: x = 5
      depth 3: x = 6
      depth 3: x = 7
    depth 2: 2 repetitions
      depth 3: x = 8
      depth 3: x = 9
  depth 1: 3 repetitions
    depth 2: 3 repetitions
      depth 3: x = 10
      depth 3: x = 11
      depth 3: x = 12
    depth 2: 2 repetitions
      depth 3: x = 13
      depth 3: x = 14
    depth 2: 1 repetitions
      depth 3: x = 15
```

# Drawbacks
[drawbacks]: #drawbacks

This adds more syntax to declarative macros.  While this is a small and useful
extension, it may not be desired to complicate declarative macros further.

The identifier syntax is similar to Perl's last-index meta-variable.  In Perl,
for an array `@foo` of length `N`, `$#foo` expands to `N - 1`.  This is
similar to, but not the same as, this proposal.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This adds a small syntax extension to declarative macros that provides
information that otherwise isn't easily attainable.

We could do nothing.  There are workarounds, such as those listed above.
These do work, although they are clumsy to use, hard to discover, and have
their own limitations.

The macro could expand to a `usize` literal (e.g. `3usize`) rather than just
a number literal.  This matches what the number is internally in the compiler,
and may help with type inferencing, but it would prevent users using
`stringify!($#x)` to get the number as a string.

In its simplest form, this only expands to a single level of repetition
counts.  In the example above, if we wanted to know the total count of
repetitions (i.e., 15), we would be unable to do so easily.  There are a
couple of alternatives we could use for this:

* `$#var` could expand to the total count, rather than the count at the
  current level.  This would make it hard to find the count at a particular
  level.

* We could use the number of '#' characters to indicate the number of depths
  to sum over.  In the example above, at the outer-most level, `$#x` expands
  to 2, `$##x` expands to 6, and `$###x` expands to 15.

# Prior art
[prior-art]: #prior-art

Declarative macros with repetition are commonly used in Rust for things that
are implemented using variadic functions in other languages.

* In Java, variadic arguments are passed as an array, so the array's `.length`
  attribute can be used.

* In dynamically-typed languages like Perl and Python, variadic arguments are
  passed as lists, and the usual length operations can be used there, too.

The syntax is similar to obtaining the length of strings and arrays in Bash:

```bash
string=some-string
array=(1 2 3)
echo ${#string}  # 11
echo ${#array[@]}  # 3
```

Similarly, the variable `$#` contains the number of arguments in a function.

# Unresolved questions
[unresolved-questions]: #unresolved-questions


# Future possibilities
[future-possibilities]: #future-possibilities

The syntax being proposed is specific to counting the number of repetitions of
a metavariable, and isn't easily extensible to future ideas without more
special syntax.  A more general form might be:
```
   ${count(ident)}
```

In this syntax extension, `${ ... }` in macro expansions would contain
metafunctions that operate on the macro's definition itself.  The syntax could
then be extended by future RFCs that add new metafunctions.  Metafunctions
could take additional arguments, so the alternative to count repetitions at
multiple depths (`$##x` above) could be represented as `${count(x, 2)}`.

There's nothing to preclude this being a later step, in which case `$#ident`
would become sugar for `${count(ident)}`, so this kind of metalanguage for
metavariables is probably best left to a future RFC that actually needs it.
