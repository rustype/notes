# `#[derive(Builder)]`

> This macro generates the boilerplate code involved in implementing the builder pattern in Rust.
> Builders are a mechanism for instantiating structs,
> especially structs with many fields,
> and especially if many of those fields are optional or the set of fields may need to grow backward compatibly over time.

<!-- The macro tests are separated in nine sections and obviously progress in an incremental fashion. -->


## Setting up

Before we start writing our code, it is important to gather the right tools!

First, our libraries, which will ease our implementation process.
Here and throughout the series we'll be using the [`quote`](https://github.com/dtolnay/quote/) and [`syn`](https://github.com/dtolnay/syn/) crates,
these were also built by David Tolnay (he's like a macro wizard), we will also use [`proc-macro2`](https://github.com/alexcrichton/proc-macro2) by Alex Crichton.
`syn` helps us when parsing Rust's `TokenStream`, `quote` serves to easily write our macro's output and `proc-macro2` will aid us when writing auxiliary code.

Another really helpful tool (for macros in general) is [`cargo-expand`](https://github.com/dtolnay/cargo-expand/), written by? You guessed it, David wrote it!
We can just `cargo install cargo-expand` to get it,
if you had any trouble installing it,
please refer to its repository.

## Workflow

As you may have noticed, the repository contains more than one exercise.
We can set up our development workflow with two terminals:

The first one will be open in `proc-macro-workshop/`,
this terminal will mainly be used to check `cargo expand`'s output.
To do so, we can just copy-paste the current test from `tests/` in `main.rs` and run `cargo expand`.

The second terminal will be open in `proc-macro-workshop/builder` and in there we will run `cargo test` as to test our macro against the existing test suite,
we just need to uncomment the tests one by on from `tests/progress.rs`.

## Parse

We are first presented with the following code:

```rust
#[proc_macro_derive(Builder)]
pub fn derive(input: TokenStream) -> TokenStream {
    let _ = input;
    unimplemented!()
}
```

The first test will look for an implementation of the derive macro with the right name,
it does nothing else, so we can simply return an empty result with `quote!().into()`,
however, we should also set up the input parsing for the upcoming tests.

```rust
#[proc_macro_derive(Builder)]
pub fn derive(input: TokenStream) -> TokenStream {
    let _ = parse_macro_input!(input as DeriveInput);
    quote!().into()
}
```

One test down, eight more to go!

## Create Builders

For the builders we are asked to produce the following code:

```rust
impl Command {
    pub fn builder() {}
}
```

So we can simply write that code with `quote!`:

```rust
#[proc_macro_derive(Builder)]
pub fn derive(input: TokenStream) -> TokenStream {
    let _ = parse_macro_input!(input as DeriveInput);
    quote!(
        impl Command {
            pub fn builder() {}
        }
    ).into()
}
```

And if we run `cargo test` we get:

```
test tests/01-parse.rs ... ok
test tests/02-create-builder.rs ... ok
```

Before we go further, we need to improve a small detail!
Attentive readers will have noticed that while `Command` is the name of our structure,
our macro is supposed to serve **any** structure, hence we are required to change the fixed name to a variable.

### Using the tree

From here on out, we will be doing heavy use of the tree,
our framework will be around reading information from the tree and writing new information based on the previous one.

Here, we start with identifiers, in Rust identifiers represent the name for variables, structures, enumerations and so on,
you can read about them in the [Rust Reference](https://doc.rust-lang.org/reference/identifiers.html).

*Example:*
```rust
let number = 5;
//  ^^^^^^
//  `number` is our variable identifier
```

For us to declare the variable which contains the structure name,
we first need to find the original one.

> **Note** - If we print our `input` with `println!("{:#?}", input)` we can see the token tree pretty printed.

At the root of the tree we have a [`DeriveInput`](https://docs.rs/syn/1.0.53/syn/struct.DeriveInput.html),
which in turn has several fields, the one we are interested in is `ident` which is the name of the struct.
With this in mind we can simply extract the field out and write our `quote!` as follows:

```rust
let struct_ident = input.ident;
quote!(
    impl #struct_ident {
        pub fn builder() {}
    }
).into()
```

Notice how `quote!` allows us to do variable interpolation!
We will use this mechanism a lot, in fact, I'd say this is `quote!`'s main feature!

### Adding `struct CommandBuilder`

So far, we have written a `impl` for our structure,
however the exercise description also asks us to write a `CommandBuilder` structure:

```rust
pub struct CommandBuilder {
    executable: Option<String>,
    args: Option<Vec<String>>,
    env: Option<Vec<String>>,
    current_dir: Option<String>,
}
```

#### Writing the basic `struct`

We already know that `Command` will be replaced by the original `struct` identifier,
and we are required to format the string like `format!("{}Builder, ident.to_string)`,
this however, won't work since it will produce a final output like `"Command"Builder` which is not valid Rust!

To solve this problem the `quote` crate provides a handy macro, `format_ident!` which replaces the previous approach with:
`format_ident!("{}Builder", ident)`.

We are now ready to write our first part!

```rust
let builder_struct_ident = format_ident!("{}Builder", struct_ident);
quote!(
    pub struct #builder_struct_ident {}
)
```

Easy enough! Moving on to the rest of the `struct`,
we can divide the work into two parts:

1. Write all `struct` fields
2. Wrap all fields with `Option<T>`

#### Writing the `struct` fields

Writing the `struct` fields is not too complicated.
Just like there was a `ident` field in `DeriveInput`,
there is a field containing the `struct`'s fields and company,
that field is `data` which according to the documentation is: *Data within the struct or enum.*

Not very exciting but upon close inspection `data` as type `Data` which is an `enum`,
in turn it represents three possible types `Data` types, structures, enumerations and unions.
In our case, we're only expecting structures, so let's start by matching against that!

```rust
fn builder_struct(struct_ident: &syn::Ident, data: &syn::Data) -> Option<proc_macro2::TokenStream> {
    if let syn::Data::Struct(struct_data) = data
    {
        let builder_struct_ident = format_ident!("{}Builder", struct_ident);
        Some(quote!(
            pub struct #builder_struct_ident {}
        ))
    } else {
        None
    }
}
```

I've extracted the logic to a separate method,
we are returning a `proc_macro2::TokenStream` to match the `quote!` output,
finally we wrap it with `Option<T>` to deal with the case that `data` is not a `Data::Struct`.

#### Traversing the tree

We can save some work and be more specific about the desired content.
From the `syn::Data::Struct` we just care for the inner `syn::DataStruct`,
furthermore, from the latter we just want to consider the [`fields: syn::Field`](https://docs.rs/syn/1.0.53/syn/enum.Fields.html) field,
finally we want to extract named fields only (as a builder requires the name of the respective variables).

<!-- This is a great candidate for a tree diagram -->

Our previous code should now look like:

```rust
fn builder_struct(struct_ident: &syn::Ident, data: &syn::Data) -> Option<proc_macro2::TokenStream> {
    if let syn::Data::Struct(syn::DataStruct {
        fields:
            syn::Fields::Named(syn::FieldsNamed {
                named: named_fields,
                ..
            }),
        ..
    }) = data
    {
        let builder_struct_ident = format_ident!("{}Builder", struct_ident);
        Some(quote!(
            pub struct #builder_struct_ident {}
        ))
    } else {
        None
    }
}
```

> **Note** - We use `..` to denote other fields in the struct for which we don't care.

While we did a lot of work manually traversing the tree,
we now can simply call the `.iter()` on our `named_fields` to get access to the fields!
The body of the matching part of the `if` will look as follows:

```rust
let builder_struct_fields = named_fields.iter();
let builder_struct_ident = format_ident!("{}Builder", struct_ident);
Some(quote!(
    pub struct #builder_struct_ident {
        #(#builder_struct_fields),*
    }
))
```

Notice that the interpolation for the iterator is different comparing to the normal variable!
`quote!` has a similar syntax to `macro_rules!` when it comes to repetition,
I recommend you read the [Interpolation section](https://docs.rs/quote/1.0.7/quote/macro.quote.html#interpolation) in the `quote!` documentation.

With this, we are currently generating a `CommandBuilder` which looks like:

```rust
pub struct CommandBuilder {
    executable: String,
    args: Vec<String>,
    env: Vec<String>,
    current_dir: String,
}
```

> **Recap** - You can read the code written so far [here](https://github.com/rustype/proc-macro-workshop/blob/7ff13b4fa9bdd8dae4fef47d6236e190d526c8b1/builder/src/lib.rs).

#### Wrapping the fields with `Option<T>`

To finalize exercise two we now need to wrap the type with `Option<T>`.
Since we already have the fields in a `iter` we can just `.map` over them and add the necessary changes.

To ease reading (and because of good practices) let's extract this functionality of "optionizing" a field to a function.

```rust
fn optionize_field(field: &syn::Field) -> proc_macro2::TokenStream {
    let ref field_name = field.ident;
    let ref field_type = field.ty;
    quote!(#field_name: Option<#field_type>)
}
```

So, what is happening you might ask, simply put, we access the `ident` and `ty` of the field and simply write the new field as `#field_name: Option<#field_type>`, effectively wrapping it with `Option<T>`.

An important note is that the code above fails to consider that the user might have redefined `Option<T>`,
to cope with such problem we simply make use of the qualified path:

```rust
fn optionize_field(field: &syn::Field) -> proc_macro2::TokenStream {
    let ref field_name = field.ident;
    let ref field_type = field.ty;
    quote!(#field_name: std::option::Option<#field_type>)
}
```

> **Recap** - You can read the code written so far [here](https://github.com/rustype/proc-macro-workshop/blob/5f2f9ee1276bb96dcb01aca5d0e1d8a9ac0bd833/builder/src/lib.rs).

### Making `builder` useful

The only thing keeping us from finishing this exercise is making `Command::builder` be useful by returning our new `CommandBuilder`,
the pattern is similar to what we have just done, iterate fields and write values.
So let's get it going.

In our `impl_struct` function we add the same `if` as before:

```rust
fn impl_struct(struct_ident: &syn::Ident, data: &syn::Data) -> Option<proc_macro2::TokenStream> {
    if let syn::Data::Struct(syn::DataStruct {
        fields:
            syn::Fields::Named(syn::FieldsNamed {
                named: named_fields,
                ..
            }),
        ..
    }) = data
    {
        Some(quote! (
            impl #struct_ident {
                pub fn builder() {}
            }
        ))
    } else {
        None
    }
}
```

Next we need to `map` again, turning each field into their respective initialization.
Since we're in the business of repeating tasks, we will also create a function to generate the initialization.

```rust
fn initialize_field(field: &syn::Field) -> proc_macro2::TokenStream {
    let ref field_name = field.ident;
    quote!(#field_name: None)
}
```

And we will use it just like the other functions:

```rust
let builder_struct_inits = named_fields.iter().map(initialize_field);
let builder_struct_ident = format_ident!("{}Builder", struct_ident);
Some(quote! (
    impl #struct_ident {
        pub fn builder() -> #builder_struct_ident {
            #builder_struct_ident {
                #(#builder_struct_inits),*
            }
        }
    }
))
```

We have just finished exercise two!

> **Recap** - You can read the code written so far [here](https://github.com/rustype/proc-macro-workshop/blob/21ad770926d4f6d3dda0eba05d2feaca6fd01655/builder/src/lib.rs).

## Call Setters

We are now tasked with writing the setters for our builder,
this implies that we write functions!
Since there is no time like the present, let's dive in!

Conceptually what we need to do is simple,
we just need to create a `impl CommandBuilder` and add setters of the following form:

```
fn #ident(&mut self, #ident: #ty) -> &mut Self {
    self.#ident = Some(#ident)
    self
}
```

`#ident` will be the identifier and `ty` the type of the original variable.
As a quick aside, notice how using `Self` helps us by saving some work!

As usual, let's first break down what we have to do to achieve our goal:

1. Write the `impl`
2. Write the functions

That is it, not that much, is it? Let's get to work!

### Writing the `impl`

This task should be easy enough, we have done this [before](#create-builders).
Again, as wise developers, we create a new method to ease future reading,
the method will be tasked with generating `impl CommandBuilder`.

```rust
fn builder_impl(struct_ident: &syn::Ident) -> proc_macro2::TokenStream {
    let builder_struct_ident = format_ident!("{}Builder", struct_ident);
    quote!(impl #builder_struct_ident {})
}
```

Easy! Let's move on to the functions!

### Writing the functions

Just like we iterated the fields when building `CommandBuilder` we can iterate the fields to get the identifiers and types!
We can steal the function skeleton:

```rust
fn builder_impl(struct_ident: &syn::Ident, data: &syn::Data) -> Option<proc_macro2::TokenStream> {
    if let syn::Data::Struct(syn::DataStruct {
        fields:
            syn::Fields::Named(syn::FieldsNamed {
                named: named_fields,
                ..
            }),
        ..
    }) = data
    {
        let builder_impl_functions = named_fields.iter().map( /* our function goes here */ );
        let builder_struct_ident = format_ident!("{}Builder", struct_ident);
        Some(quote!(
            impl #builder_struct_ident {
                #(#builder_impl_functions)*
            }
        ))
    } else {
        None
    }
}
```

> **Note** - Hmm, that is the same code, and we'll be iterating twice! There should be a better way to do this...
> Here we can weigh our options:
>
> - We write duplicate code and iterate twice, but keep the logic of each generation separate
> - We write a single pass and return a tuple containing both the new fields, and the function implementation
> - We write a generic function which takes a closure
>
> Personally, I dislike **all three**, the first has duplicated code, but the second makes our code hard to read, third just makes it convoluted, I don't think there is a need for a closure in this case.
> In this case I'll opt with the first alternative, if you have ideas how to solve this feel free to reach out to me!

We'll repeat the pattern of extracting a function to generate the desired code, in this case I called it `functionize_field`, amazing I know! You can see it below:

```rust
fn functionize_field(field: &syn::Field) -> proc_macro2::TokenStream {
    let ref field_name = field.ident;
    let ref field_type = field.ty;
    quote!(
        fn #field_name(&mut self, #field_name: #field_type) -> &mut Self {
            self.#field_name = Some(#field_name);
            self
        }
    )
}
```

Currently, you should be able to run `cargo test` and have tests one through three pass.

> **Recap** - You can read the code written so far [here](https://github.com/rustype/proc-macro-workshop/blob/1718a251bcd43962100f5c07a0942ec54ec70214/builder/src/lib.rs).

## Call Build

For the fourth exercise we are tasked with writing `build` which should actually build our structure.
The function looks like:

```rust
impl CommandBuilder {
    pub fn build(&mut self) -> Result<Command, Box<dyn Error>> {
        // ...
    }
}
```

We can start by adding the build function stub to the method we extracted earlier:

```rust
fn builder_impl(struct_ident: &syn::Ident, data: &syn::Data) -> Option<proc_macro2::TokenStream> {
    if let syn::Data::Struct(syn::DataStruct {
        fields:
            syn::Fields::Named(syn::FieldsNamed {
                named: named_fields,
                ..
            }),
        ..
    }) = data
    {
        let builder_impl_functions = named_fields.iter().map(functionize_field);
        let builder_struct_ident = format_ident!("{}Builder", struct_ident);
        Some(quote!(
            impl #builder_struct_ident {
                pub fn build(&mut self) -> std::result::Result<#struct_ident, Box<dyn std::error::Error>> {
                    /* we now have to fill this */
                }
                #(#builder_impl_functions)*
            }
        ))
    } else {
        None
    }
}
```

The filling to our function should contain the original structure name,
as it is the function's output, as well as contain the fields.
The fields however, should throw an error in the case that they are set to `None` (i.e. they were not initialized).

Once again we'll take advantage of the iterators and create a function which only deals with a single field,
writing for one, getting the result for all.

We first extract the identifier,
then we use it to build the assignment,
since we know the original struct field name matches the builder field name we just add the self.

```rust
fn assign_field(field: &syn::Field) -> proc_macro2::TokenStream {
    let ref field_name = field.ident;
    quote!(#field_name: self.#field_name)
}
```

This is not enough however, the type of `self.#field_name` is `Option<T>`,
we also need to return an error in the case it is `None`,
we can use `Option::ok_or` to do so, unwrapping it with `?` to propagate the error.

```rust
fn assign_field(field: &syn::Field) -> proc_macro2::TokenStream {
    let ref field_name = field.ident;
    quote!(#field_name: self.#field_name.ok_or("field not initialized")?)
}
```

Finally, the compiler will complain how the `Option`'s content does not implement `Copy`:

```
error[E0507]: cannot move out of `self.current_dir` which is behind a mutable reference
  --> $DIR/04-call-build.rs:16:10
   |
16 | #[derive(Builder)]
   |          ^^^^^^^
   |          |
   |          move occurs because `self.current_dir` has type `std::option::Option<std::string::String>`, which does not implement the `Copy` trait
   |          help: consider borrowing the `Option`'s content: `Builder.as_ref()`
   |
   = note: this error originates in a derive macro (in Nightly builds, run with -Z macro-backtrace for more info)
```

To solve the error we simply can simply use `.clone`:

```rust
fn assign_field(field: &syn::Field) -> proc_macro2::TokenStream {
    let ref field_name = field.ident;
    quote!(#field_name: self.#field_name.clone().ok_or("field not initialized")?)
}
```

Now we just need to make use of the function:

```rust
fn builder_impl(struct_ident: &syn::Ident, data: &syn::Data) -> Option<proc_macro2::TokenStream> {
    if let syn::Data::Struct(syn::DataStruct {
        fields:
            syn::Fields::Named(syn::FieldsNamed {
                named: named_fields,
                ..
            }),
        ..
    }) = data
    {
        let builder_impl_functions = named_fields.iter().map(functionize_field);
        let builder_fields = named_fields.iter().map(assign_field); // we use the function here
        let builder_struct_ident = format_ident!("{}Builder", struct_ident);
        Some(quote!(
            impl #builder_struct_ident {
                pub fn build(&mut self) -> std::result::Result<#struct_ident, Box<dyn std::error::Error>> {
                    // And the iter inside the function
                    Ok(
                        #struct_ident {
                            #(#builder_fields),*
                        }
                    )
                }

                #(#builder_impl_functions)*
            }
        ))
    } else {
        None
    }
}
```

> **Recap** - You can read the code written so far [here](https://github.com/rustype/proc-macro-workshop/blob/0f65e71219ac39b265f58020338d5e5462e86704/builder/src/lib.rs).


## Method Chaining

As the description says, this should be a freebie of our previous tests are working, and it is!
You can run `cargo test` to check out if everything is running properly.

This comes for free since each function in our builder returns a reference to the `Builder`,
this allows us to chain the methods as follows:

```rust
fn main() {
    let command = Command::builder()
        .executable("cargo".to_owned())
        .args(vec!["build".to_owned(), "--release".to_owned()])
        .env(vec![])
        .current_dir("..".to_owned())
        .build()
        .unwrap();

    assert_eq!(command.executable, "cargo");
}
```

## Optional Field

Arriving at exercise six we have a wall of text to read, I *strongly* recommend you to read it with attention,
as it is interesting, informative and important for the exercise.

> **TL:DR** - We will want to identify `Option<T>` in the original structure and act accordingly.

To make it more concrete, the now original structure is:

```rust
pub struct Command {
    executable: String,
    args: Vec<String>,
    env: Vec<String>,
    current_dir: Option<String>,
}
```

<!-- Another great candidate for a tree -->

Notice the `Option<String>`? That means we will want to make `current_dir` optional when writing our macro's output.
Since we have been extracting the field handling logic to separate functions,
now we can simply check if the field's type is option and act accordingly.
To do so, we first write a function which, when given a type, checks if it is `Option`,
we just need to follow the path suggested in the exercise.

```rust
fn is_type_option(ty: &syn::Type) -> bool {
    if let syn::Type::Path(syn::TypePath {
        path: syn::Path { segments, .. },
        ..
    }) = ty
    {
        if let Some(is_opt) = segments
            .last()
            .map(|path_segment| &path_segment.ident == "Option")
        {
            return is_opt;
        }
    }
    false
}
```

We traverse the tree from `syn::Type` up to the array of `PathSegment`,
from there we just need to get the last element and check if its identifier is `Option`.

> **Note** - We only care for the last element of the path since `Option` and `std::option::Option` are exactly the same, and we want to handle both.

Wielding this powerful function we can just check for `Option` in our functions.

### `optionize_field`

If a field is already `Option<T>` we don't want to make it `Option<Option<T>>`:

```rust
fn optionize_field(field: &syn::Field) -> proc_macro2::TokenStream {
    let ref field_name = field.ident;
    let ref field_type = field.ty;
    if is_type_option(field_type) {
        quote!(#field_name: #field_type)
    } else {
        quote!(#field_name: std::option::Option<#field_type>)
    }
}
```

### `assign_field`

If a field is optional, we don't want to run `ok_or` on it:

```rust
fn assign_field(field: &syn::Field) -> proc_macro2::TokenStream {
    let ref field_name = field.ident;
    let ref field_type = field.ty;
    if is_type_option(field_type) {
        quote!(#field_name: self.#field_name.clone())
    } else {
        quote!(#field_name: self.#field_name.clone().ok_or("field was not set")?)
    }
}
```

### `functionize_field`

In the case the field is `Option<T>`, when writing the setter, we want to use `T` as the type, not `Option<T>`.
For this we are required to extract the inner type of `Option` and not only check,
we can rewrite our `is_type_option` to extract the inner type instead:

```rust
fn extract_inner_type<'t>(ty: &'t syn::Type, expected_ident: &str) -> Option<&'t syn::Type> {
    if let syn::Type::Path(syn::TypePath {
        path: syn::Path { segments, .. },
        ..
    }) = ty
    {
        if let Some(syn::PathSegment {
            ident,
            arguments:
                syn::PathArguments::AngleBracketed(syn::AngleBracketedGenericArguments { args, .. }),
        }) = segments.last()
        {
            if ident == expected_ident {
                if let Some(syn::GenericArgument::Type(ty)) = args.last() {
                    return Some(ty)
                }
            }
        }
    }
    None
}
```

Once again, we abuse the `if let` mechanism to run through the *happy* path,
if at any point we leave the happy path we just reach the end and return `None`.
Now instead of using `is_type_option` we can call `extract_inner_type(ty, "Option")` and check with `is_some`:
In the case of `functionize_field`, instead of `is_some`, we use `if let` again since we want to work with the return value:

```rust
fn functionize_field(field: &syn::Field) -> proc_macro2::TokenStream {
    let ref field_name = field.ident;
    let mut field_type = &field.ty;
    if let Some(inner_ty) = extract_inner_type(field_type, "Option") {
        field_type = inner_ty;
    }
    quote!(
        fn #field_name(&mut self, #field_name: #field_type) -> &mut Self {
            self.#field_name = Some(#field_name);
            self
        }
    )
}
```
