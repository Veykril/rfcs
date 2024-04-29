- Feature Name: `derive_smart_pointer`
- Start Date: 2024-04-29
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#123430](https://github.com/rust-lang/rust/issues/123430)

# Summary
[summary]: #summary

This RFC describes a new derive macro that allows a type to behave more like other standard library
smart pointers.

# Motivation
[motivation]: #motivation

It is currently not possible to implement custom smart pointers that behave the same as the standard
library smart pointers like `Rc` and `Arc` due to some language traits being unstable. TODO Expand this

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

The derive macro `SmartPointer` can be used on Rust structs with at least one type parameter and
exactly one field (and any number of additional one-aligned, zero-sized fields), enabling it
to behave like a "smart pointer" with regards to that type parameter. This means the struct
- can participate in unsizing coercions for that type parameter (`MySmartPtr<String>` ->
  `MySmartPtr<dyn Display>`),
- can be the receiver of a method (`fn f(self: MySmartPtr<Self>)`)
- and be the receiver of a dynamically dispatched method call (`(ptr as MySmartPtr<dyn Display>).fmt()`)

```rs
#[derive(SmartPointer)]
struct MySmartPointer<T: ?Sized>(Box<T>);

trait Trait {
    // `MySmartPointer` is a valid receiver type due to the derive
    fn func(self: MySmartPointer<Self>);
}

impl Trait for i32 {
    fn func(self: MySmartPointer<Self>) {}
}

fn main() {
    // This unsizing coercion would be an error without the derive
    let ptr: MySmartPointer<dyn Trait> = Ptr(Box::new(4));
    // And so would the dynamic dispatch
    ptr.func();
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The derive will expand to three trait implementations, one for `core::ops::CoerceUnsized`, `core::ops::DispatchFromDyn` and
`core::ops::Receiver` respectively.

Given the following code:
```rs
#[derive(SmartPointer)]
struct MySmartPointer<'a, T: ?Sized>(&'a T);
```

the generated `core::marker::Unsize` impl will be:
```rs
#[automatically_derived]
impl<'a, T, U> ::core::ops::CoerceUnsized<MySmartPointer<'a, U>> for MySmartPointer<'a, T>
where
    T: ?Sized + ::core::marker::Unsize<U>,
    U: ?::core::marker::Sized
{}
```

That is, the generated trait impl will carry over all generic parameters and bounds of the struct
definition as usual for derives and in addition, introduce one extra type parameter `U` that is
bounded by `?Sized` and an extra bound `::core::marker::Unsize<U>` for the
target type parameter.

the generated `core::ops::Receiver` impl will be (this assumes the changes of to the trait as
outlined in https://github.com/rust-lang/rfcs/pull/3519):

```rs
#[automatically_derived]
impl<'a, T> ::core::ops::Receiver for MySmartPointer<'a, T>
where
    T: ?Sized
{
    type Target = T;
}
```

That is, the generated trait impl will carry over all generic parameters and bounds of the struct
and contain a `type Target = T` associated type where `T` is the target type parameter.

and the generated `core::ops::DispatchFromDyn` impl will be:

```rs
#[automatically_derived]
impl<'a, T, U> ::core::ops::DispatchFromDyn<MySmartPointer<'a, U>> for MySmartPointer<'a, T>
where
    T: ?Sized + ::core::marker::Unsize<U>,
    U: ?::core::marker::Sized
{}
```

That is, the generated trait impl will carry over all generic parameters and bounds of the struct
definition as usual for derives and in addition, introduce one extra type parameter `U` that is
bounded by `?Sized` and an extra bound `::core::marker::Unsize<U>` for the
target type parameter.

The `DispatchFromDyn` checks will already correctly reject usages of the derive on structs with a
non-rust representation, that have zero or more than one fields that are not
one-aligned and zero-sized, or whose single field type lacks a `DispatchFromDyn<F>` implementation
where `F` is the type of the target type parameter’s field type.

# Drawbacks
[drawbacks]: #drawbacks

- This adds a standard library API that may be deprecated in the future should the implementation
details it abstracts away become stable, though this is unlikely to be the case as the derive
generally has a convenience factor nevertheless.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- The approach outlined here has the benefit that the concrete mechanism behind unsizing and dynamic
  dispatching remains open to changes, as the derive fully hides these details.
- Flesh out the design of and stabilize the `Unsize`, `CoerceUnsized` and `DispatchFromDyn` traits.
  This will take considerable effort and is unlikely to happen in the near future.
- Do nothing. This would mean having proper custom smart pointers that work like the standard
  library ones remains impossible.

# Prior art
[prior-art]: #prior-art

n/a

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should the derive have helper attributes to adjust the behavior? An example would be being able
  to annotate the type parameter that is the target of the unsizing behavior.
- Should the derive reject structs with 0 or more than 1 type parameters? If not, is the first type
  parameter to be the unsizing target a good choice?
- Should the derive implement `core::ops::Deref`?

# Future possibilities
[future-possibilities]: #future-possibilities

n/a
