- Start Date: 2014-05-19
- RFC PR #: (leave this empty)
- Rust Issue #: (leave this empty)

# Summary

Rust currently lacks support for guaranteed tail call optimization. A tail recursive function *may*
be optimized by LLVM's `tailcallelim` pass, but neither Rust or LLVM provides any guarantees. LLVM
passes will often transform the function in a way that prevents tail call optimization, or simply
miss seemingly possible cases due to strict requirements.

# Motivation

Recursion is often a far more natural way of expressing an algorithm. In Rust, recursion can also be
used to express patterns not otherwise possible in safe code. For example, the `find_mut` method for
`collections::TreeMap` uses a recursive algorithm rather than the iterative one used by `find`. It
should be possible to improve the borrow checker to handle these cases, but the recursive algorithm
will remain easier to read and reason about.

# Drawbacks

This proposal adds an extra keyword to the language, although `be` has been reserved with this in
mind for some time. LLVM now provides this feature, but another backend may need to do extra manual
work to provide this as a guarantee.

# Detailed design

The `become` expression is introduced, usable in place of a `return` expression where the result is
another function call.

```rust
fn foo(x: int, y: int) {
    println!("{} {}", x, y);
    return foo(x, y)
}
```

This could also be written with `become`:

```rust
fn foo(x: int, y: int) {
    println!("{} {}", x, y);
    become foo(x, y)
}
```

This will map down to an LLVM `call` instruction marked with `musttail`, followed by a `ret`
instruction. The compiler will perform the necessary type-checking to satisfy the guarantees
required by `musttail`. The required guarantees are as follows:

* The call must immediately precede a ret instruction, or a pointer bitcast followed by a ret
  instruction.
* The ret instruction must return the (possibly bitcasted) value produced by the call or void.
* The caller and callee prototypes must match. Pointer types of parameters or return types may
  differ in pointee type, but not in address space.
* The calling conventions of the caller and callee must match.
* All ABI-impacting function attributes, such as sret, byval, inreg, returned, and inalloca, must
  match.
* The callee does not access allocas or varargs from the caller

The `become` instruction covers part of the requirements itself, by representing a call followed by
a return from the function. The compiler also needs to forbid having any variables with destructors
in scope, as these would need to be called in between the function call and return.

The compiler can force the type signature of the caller and callee to match, resulting in a
guarantee of a matching calling convention / ABI.

The requirement of not accessing the caller's allocas is the hardest to enforce. The compiler would
forbid passing non-immediate types, and a default-deny lint would forbid passing non-primitive types
since the representation may change. The compiler would also forbid passing any lifetime-bounded
types not guaranteed to outlive the caller.

Since unique pointers and references are guaranteed to be pointer-size / immediate (for non-DST
types), these would be the suggested solution for passing around non-primitive types.

# Alternatives

An attribute on `return` could replace the `become` keyword, if and when it becomes possible to
attach attributes to expressions. This feature can remain feature gated until this is resolved one
way or the other, with the promise that it will stay around but without a backwards compatible
syntax.

# Unresolved questions

The current requirements proposed above are very strict. These could be relaxed in the future. For
example, the lint could be informed that a type like `Rc` will have a stable representation and
allow passing it to the callee.
