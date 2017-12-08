# Rental - A Macro to generate safe self-referential structs

[Documentation](http://docs.rs/rental)

# Overview
It can sometimes occur in the course of designing an API that it would be convenient, or even necessary, to allow fields within a struct to hold references to other fields within that same struct. Rust's concept of ownership and borrowing is powerful, but can't express such a scenario yet.

Creating such a struct manually would require unsafe code to erase lifetime parameters from the field types. Accessing the fields directly would be completely unsafe as a result. This library addresses that issue by allowing access to the internal fields only under carefully controlled circumstances, through closures that are bounded by generic lifetimes to prevent infiltration or exfiltration of any data with an incorrect lifetime. In short, while the struct internally uses unsafe code to store the fields, the interface exposed to the consumer of the struct is completely safe. The implementation of this interface is subtle and verbose, hence the macro to automate the process.

The API of this crate consists of the `rental` macro, which generates safe self-referential structs, and a few example instantiations to demonstrate the API provided by such structs (see `RentRef`, `RentMut`, `RentRefMap`, `RentMutMap`).

# Use Cases
One instance where this crate is useful is when working with `libloading`. That crate provides a `Library` struct that defines methods to borrow `Symbol`s from it. These symbols are bounded by the lifetime of the library, and are thus considered a borrow. Under normal circumstances, one would be unable to store both the library and the symbols within a single struct, but the macro defined in this crate allows you to define a struct that is capable of storing both simultaneously, like so:

```rust,ignore
rental! {
    pub mod rent_libloading {
        use libloading;

        #[rental(deref_suffix)] // This struct will deref to the target of Symbol.
        pub struct RentSymbol<S: 'static> {
            lib: Box<libloading::Library>, // Library is boxed for stable deref.
            sym: libloading::Symbol<'lib, S>, // The 'lib lifetime borrows lib.
        }
    }
}

fn main() {
    let lib = libloading::Library::new("my_lib.so").unwrap(); // Open our dylib.
    if let Ok(rs) = rent_libloading::RentSymbol::try_new(
        Box::new(lib),
        |lib| unsafe { lib.get::<extern "C" fn()>(b"my_symbol") }) // Loading Symbols is unsafe.
    {
        (*rs)(); // Call our function
    };
}
```

In this way we can store both the `Library` and the `Symbol` that borrows it in a single struct. We can even tell our struct to deref to the function pointer itself so we can easily call it. This is legal because the function pointer does not contain any of the special lifetimes introduced by the rental struct in its type signature, which means reborrowing will not expose them to the outside world. As an aside, the `unsafe` block for loading the symbol is necessary because the act of loading a symbol from a dylib is unsafe, and is unrelated to rental.

# Limitations
There are a few limitations with the current implementation. These limitations aren't fundamental, but rather the result of bugs or pending features in rust itself, and will be lifted once the underlying language allows it.

* Currently, the rental struct itself can only take lifetime parameters under certain conditions. These conditions are difficult to fully describe, but in general, a lifetime param of the rental struct itself must appear "outside" of any special rental lifetimes in the type signatures of the struct fields. To put it another way, replacing the rental lifetimes with `'static` must still produce legal types, otherwise it will not compile. In most situations this is fine, since most of the use cases for this library involve erasing all of the lifetimes anyway, but there's no reason why the head element of a rental struct shouldn't be able to take arbitrary lifetime params. This is currently impossible to fully support due to lack of an `'unsafe` lifetime or equivalent feature.
* Prefix fields must be of the form `Foo<T>` where `Foo` is some `StableDeref` container, or rental will not be able to correctly guess the `Deref::Target` of the type. If you are using a custom type that does not fit this pattern, you can use the `target_ty_hack` attribute on the field to manually specify the target type. This limitation can be lifted once rust gets ATC or possibly after current bugs surrounding HRTB associated item unification are fixed. The requirement for `StableDeref` at all can possibly be lifted if rust gains a notion of immovable types.
* Rental structs can only have a maximum of 32 rental lifetimes, including transitive rental lifetimes from subrentals. This limitation is the result of needing to implement a new trait for each rental arity. More traits can easily be defined though. Lifting this limitation entirely would require some kind of variadic lifetime generics.
* The references received in the constructor closures don't currently have their lifetime relationship expressed in bounds. This is currently not possible until the trait system refactor lands. This is not a soundness hole, but it does prevent some otherwise valid uses from compiling.
