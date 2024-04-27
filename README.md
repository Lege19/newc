# newc

Despite being over 50 years old, C is still in heavy use today, even though other alternatives like Rust or C++ share many of the advantages of C. So what keeps developers using C all these years later? Simplicity. In C very little goes on between the code you write and what is executed by the CPU, however in Rust the compiler is always there interfering to make sure your code is safe, while many programmers feel that C++ is badly designed, and tries to combine too many different things. That said, modern C still has many quirks, probably because most of it was designed 50 years ago. Thus I propose newc. A language with the transparent workings of C, with a modern syntax and developer experience.

## Design Philosophy

C is a very unsafe language: in all C code, it is the programmer's responsibility to make sure that the code is safe. This can make C code extremely fast and clean.

Rust is (almost always) a safe language: the Rust compiler is constantly taking responsibility for code safety. Although unsafe Rust code *can* be written with the `unsafe` keyword, it is far from convenient to do so, and - if not considered bad practise - should be minimised.

The problem with C is not that it is unsafe, but that it is always unsafe. The fact that C developers have the freedom to aggressively optimise their algorithms to get all the possible performance is great. But the rest of the time, when the programmer is doing something trivial, the compiler *still* assumes that they have carefully considered how their code might break.

The solution is to create a language where the compiler takes as much of the responsibility off the programmer as possible, but where it is extremely easy for the programmer to take responsibility back. This means unsafe code *will* be used frequently, unlike in rust, but where it is used it is used because the programmer has considered whether it is necessary, and (hopefully) what the consequences are.

# Design

## Language Syntax

### Variable Declarations

Many C programmers I have comsulted have expressed a strong preference for C style declarations, i.e.

```c
// c
int x = 5;
```

instead of

```rust
// rust
let x: i32 = 5;
```

This is a pitty because Rust style declarations are objectively better, mainly because Rust declataions are actually pattern matches. They also allow for some extremely powerful patterns, namely `if let` and `let else` (`while let` also exists but is less often used). These can be used as follows:

```rust
// rust
fn foo() -> Option<i32> { ... }// foo will sometimes return None and sometimes return Some
fn bar() {
	if let Some(x) = foo() {
		// this code block only runs if foo returned Some
		println!("{}", x);
	}
	let Some(x) = foo() else {
		// this code block is run if the output of foo() does not match the pattern, i.e. Some(x)
		// this code block must 'diverge', meaning it must break out of the current scope, often by returning from the function
		return;
	};
	println!("{}", x);
}
```
Something not even rust supports however, is type annotation within patterns (and also some other contexts):
<a id="type-annotations-in-patterns"></a>
```rust
let Some(x: i32) = Some(5);// invalid
let Some(x): Option<i32> = Some(5);// valid
```
This can get quite unergonomic, for example:
```rust
struct Foo<A, B, C> {
	a: A,
	b: B,
	c: C
}
fn bar() {
	let Foo {a, b, c} = baz();// (1) valid
	let Foo {a: i32, b: &'static str, c: bool} = baz();// (2) invalid
	let Foo {a, b, c}: Foo<i32, &'static str, bool> = baz();// (3) valid

	// which is the easist to tell the types of a, b, and c? Admitedly the difference between (2) and (3) is not so large with this example, but I hope you can see how with more complex types this could get very confusing
}
fn baz() -> Foo<i32, &'static str, bool> {
	Foo {
		a: 1, 
		b: "hi", 
		c: false
	}
}
```
Sadly, this just wouldn't work very well with C style type "annotations":
```cpp
auto Foo {int a, StrSlice b, bool c} = baz();
// the above implies that
auto Foo {a, b, c} = baz();
// is invalid, and instead you would have to write
auto Foo {auto a, auto b, auto c} = baz();
// which would be valid but verbose and weird
```
The bottom line is that C style declatations allow for type inference, but not true type *annotation*.

The other thing that really puts me off C style declataions is the inflexibility for adding new syntax: by not having a keyword that identifies a statement as a declaration, C syntax becomes less flexible. Changing C declaration syntax in almost any way makes it syntactically ambiguous, even just adding support for generics/templates causes problems as the parser cannot know weather to parse it as a data type or an expression. 

For these reasons I will use `let` for variable declarations. They will be largely the same as with Rust, but I will still make a few changes to make them even more powerful.

1. Pattern syntax will be changed to allow type annotatitions within patterns (like [here](#type-annotations-in-patterns))
2. `let else` will be extended to allow chaining multiple values that could match:
	```rust
	let Some(x) = returns_none() else returns_none() else return;
	```
3. `let` will be allowed as a condition more flexibly:
	```rust
	// invalid in Rust but valid in newc
	if let Some(x) = Some(5) && true {

	}
	// invalid in Rust but valid in newc
	if let Some(x) = Some(5) && let Some(y) = None {

	}
	```
	Note however, that || *cannot* be used with `let` conditions. This is because if only one of multiple patterns was required to match, then the code block would have no way to know which is valid. If instead the intent is to define fallback values to pattern match against, `let else` can be used:
	```rust
	// invalid in Rust but valid in newc
	if let Some(x) = None else Some(5) {

	}
	```
	Note that in this context the `let else` chain does not need to end with a block that diverges, as if no match is made then the inner code simply isn't run.

#### Type Inference & Annotations

Type inference for local variables is convenient, and so long as the variable doesn't live too long isn't very harmful to code readability. However one weird case is type inference for constants. Since these *can* be very far reaching, or might just be local to a module. For this reason, in newc, type inference on private constants will be allowed, but not for public constants.

Rust's syntax for type annotations is good, I will use it. However in newc, type annotations will be allowed in more places, such as within patterns and within iterator based `for` statements. 

#### Function Declarations

I will use the `fn` keyword. As with `let`, this allows more flexibility in declarations. Parameters will use the same type annotation syntax as Rust, however the return type will use TypeScript syntax, as in:
```rust
// newc
fn foo(): i32 {

}
```
instead of
```rust
// rust
fn foo() -> i32 {

}
```
There isn't really anything wrong with `->`, but I suspect the only reason Rust uses it is because Haskell does. It just doesn't make much sense to introduce more syntax when in literally all other conexts type annotation uses `:`.

#### Mutability

Since newc, like Rust, will use pattern matching for assignments, variables will be immutable by default. This is partly for consistency with `match` but mainly because it means programmers will only use mutable variables when they need to (unlike in JavaScript where it `let` is often used for variables that could be `const`).

Mutability in patterns will be the same as in rust, i.e. `let mut`.

### Move Semantics

*see [Rust by Example](https://doc.rust-lang.org/rust-by-example/scope/move.html) for terminology*

In Rust, all data types (except primitives) have move semantics by default. The programmer can switch their data types to use copy semantics instead by placing `#[derive(Copy)]` before their type declaration.
Move semantics are very useful for safer heap allocation, because they prevent many use after free and double free errors. For example, the `Box<T>` smart pointer in Rust manages some data on the heap, if a second `Box<T>` were accidentally created, the both `Box<T>`s would try to free the data in their destructors.
newc *will* have support for move semantics, they can be enabled by adding `move` to a newtype declaration:

```Rust
// newc
move subnewtype Box<T> = &T;
```

### Control Flow

I like Rust's control flow a lot, so I will use it as a base from which to design newc's.

#### `loop`

I considered removing `loop`, because it can be created with `while true`, which is more common and more familiar to C developers. However as was pointed out [here](https://stackoverflow.com/a/28892433) this would make it harder for the compiler to infer that variables initialized inside the loop have definitely been initialized; and making the compiler explicitly recognise `while true` would lead to weird behaviour if the `true` was then moved to a variable or constant for whatever reason. For this reason newc *will* have a `loop` keyword.
Rust has no ternary operator, partly because `?` is used for error handling, and partly because `if condition {expression} else {expression};` is more readable than `condition ? expression : expression` (though this is quite subjective). So newc will not have a ternary operator either.

#### Inline `if`

One thing Rust doesn't have that I would like newc to have is inline `if` statements. This is because I feel that `if (condition) break;` is nicer than `if condition {break};`. But there's a problem: newc, like Rust won't require brackets around conditions, so it would actually be `if condition epression;`, which would be extremely difficult for the compiler to understand, and reduce readability. I considered working around this by instead using `expression if condition`, which *would* partially solve the problem because the compiler could recognise the `if` keyword, and the programmer's syntax highlighting would make it visually clear where the expression ended and the condition began... but I think this isn't enough to justify this weird syntax. Overall I think the only good way to have inline `if` is with brackets around the condition, so that is what newc will use (more formally braces around a single expression are not required if there are parenthesis around the condition) . This means that the following are all valid syntax.

- if condition {expression};
- if (condition) expression;
- if condition {expression} else {expression};
- if (condition) expression else expression;

#### `do {} while`

Rust doesn't have a `do {} while condition;`, and although I have never needed it (and it can easily be recreated with `loop`), I think it is worth including in newc to make it slightly easier for C developers to pick up. The readability of this questionable, because the condition to be checked for is at the end, where you may need to scroll down to see it, but I believe this is ok because all the code above it will get run at least once before the condition is evaluated, so it makes sense that someone reading the code should read the condition only after reading the body of the loop.

#### `for`

In Rust, `for` loops can only be used with an iterator: (`for i in iterator`), this is usually fine, but it does miss out on some flexibility offered by C's `for` loops (`for (initialiser; condition; updater)`, so newc will support both. The compiler will interpret one or the other depending on whether there are parenthesis (conveniently both C and Rust reject missing or unneeded parenthesis, so this is compatible with both).

### Types

Rust has a wonderful type system so most of newc's type system will be the same.

#### Pointers

In C pointers are created using the `&` operator, and dereferenced using the `*` operator. However to declare a pointer variable some C programmers use `T* foo`, and some use `T *foo`, the former preferred by some because it makes it clearer that the pointer is part of the type. Whereas the latter is more widely used because it avoids confusion with having multiple declarations in one statement (in: `T* foo, bar;` , `foo` is a pointer but `bar` is not).
This is a common source of confusion in C so I will instead use `&T foo` to declare a pointer variable in newc. This means that `&` means the same thing in variable declarations as in expressions, which will make newc easier to learn, and avoid weird edge cases.

When applied to a value, `&` is an operator.
When applied to a data type, `&T` is syntactic sugar for `Ptr<T>`, which is in turn defined as `subnewtype Ptr<T> = uintsize`

#### Fat Pointers

Rust supports fat pointers for Unsized types and for trait objects. This would be fine, except it means that the size of `*const T` cannot be known without knowing what `T` is. I really don't like this because I feel it is quite misleading. A pointer *should* just be a memory address, this makes them much easier to work with. Fat pointers would be structs.

#### Slices

Slices in Rust are great to work with, and newc hopes to achieve similar ease of use, but since all pointers in newc are thin pointers, Slices will instead be defined as:

```C
// newc
subnewtype Slice<T> = struct {
	ptr: &T,
	len: uintsize
```

#### Generics

I was suprised that C has no generics (and no clean substitute other than switching to C++) partly because it requires [name mangling](https://en.wikipedia.org/wiki/Name_mangling), which C doesn't have. Generics are required for many of the convenience types Rust (and also newc) provide so newc will have to have them, even if that makes the compiler more complex.

#### Turbofish

Some syntactic ambiguity arises from `<` and `>` being used for both generics and for comparisons (which is made worse by allowing constant expressions as [`const` generics](https://doc.rust-lang.org/reference/items/generics.html#const-generics)). Here is a code snippet from the [bastion of the Turbofish](https://github.com/rust-lang/rust/blob/master/tests/ui/parser/bastion-of-the-turbofish.rs) to illustrate the problem:

```rust
// Rust
let (the, guardian, stands, resolute) = ("the", "Turbofish", "remains", "undefeated");
let _: (bool, bool) = (the<guardian, stands>(resolute));
```

In order to resolve this problem, Rust requires programmers to use `::<T>` (turbofish) instead of just `<T>` when specifying generic parameters in expressions. This solves the problem but isn't very nice, and often confuses beginners. I have thought of five potential solutions to this problem:

1. Use different syntax (e.g. turbofish) in syntax, this is what Rust does
2. Use different syntax for generics in all contexts to remove the ambiguity
3. Use different syntax for comparisons to remove the ambiguity
4. Build a compiler that interprets this syntax based on what the identifiers refer to, this is what C++ does
5. Introduce strict naming rules so that the parser can tell the difference between variables, functions, and data types
6. Put the generics *before* the data type, this is what Java does
7. Require spaces around binary comparison operators

Option 1 is messy, context dependent syntax will always lead to confusion
Option 2 is very promising, using `::<T>` in all contexts to specify generics would probably be fine, especially considering newc's psudo-polymorphism.
Option 3 can be discarded as it makes little sense not to use universal mathematical notation, in order to comply with a non-universal convention for generics syntax.
Option 4 can be discarded as it makes the compiler much more complicated.
Option 5 can also be discarded for being too weird, inflexible, and probably incompatible with identifiers written with other alphabets.
Option 6 might work, but I'm not sure it wouldn't result in some other syntactic ambiguity, in any case I won't write it off just yet.
Option 7 could lead to inconsistency if `1+1` is allowed, but `1>0` is not, although I don't think this would be a huge issue, so long as the error messages helpfully point out what the programmer may have meant.

Overall, the two solutions I am by far the most drawn to is 2 (using `::<T>` everywhere, not just in expressions), and 7 (requiring spaces around binary comparison operators). However I thought of this 'counter example' that really kills option 2:

```rust
(foo<A > (B), C < D>(E))
```

You can't decipher this at a glance, and even if you knew that foo was declared as:

```rust
fn foo<const A: bool, const B: bool>(e: int32) { ... }
```

It would still be pretty hard to work out what the above expression evaluated to. I think that even once programmers were used to this syntax, it would still be confusing: since our brains are wired to match delimiters: `<` and `>` will look like a pair no matter the spacing around them.
But turbofish doesn't totally solve this problem either, as although it is clear when `<` is a comparison, it is not so clear when `>` is a comparison:

```rust
(foo::<A > (B), C < D>(E))
```

However at this point, so long as there aren't any constant expressions as `const` generics, it's totally fine. So I thought of a very simple solution to this problem: require parenthesis around constant expressions in generics. So the above code would become:

```rust
(foo::<(A > (B)), (C < D)>(E))
```

And suddenly, though still not exactly easy to read, this code is no longer ambiguous.

#### Arrays

In newc array types will use C-style syntax, this is just more familiar to more developers.

#### Unions, Enums, Sum Types

C provides the `union` keyword, which allows a developer to declare a data type with several fields that each occupy the same memory, this gives unions two uses:

- unions can be used to interpret the same data as multiple types for example, a much cleaner alternative to [bit casts](#bit-casts)
- unions can be combined with C enums to create a [tagged union](https://en.wikipedia.org/wiki/Tagged_union)

C enums are very simple, they can take one of several predefined values, and that's it. However Rust enums do everything C enums do, but are also a much neater way to create and work with a [tagged union](https://en.wikipedia.org/wiki/Tagged_union). Rust does support `union`s, but they are vary rarely used

Since there are three different things programmers may want here, newc will have three different keywords for them:

- `enum` (a C style enum)
  Syntax identical to a C enum, except the variants are scoped to the enum
- `sum` (a tagged union)
  Syntax the same as a Rust enum
- `union` (a C style union)
  Syntax identical to a C union

I fell these better describe their purpose than the terms used in Rust and C.

#### Structs and Tuples

Structs in newc will be the same as structs in Rust.

Tuples however will be declared with `tuple Name { T1, T2, T3 }`. This will be much easier to parse and means that all types in newc are defined as `keyword Name { ... }` (yay consistency).

#### Anonymous/Inline Types

*There is significant confusion in the terminology here, to clarify*

- *When I refer to 'inline', I mean a data type that is declared without being given a name.*
- *When I refer to 'anonymous' I mean a data type that is used as a field in a struct or union without naming the field.*

Rust supports inline tuples but not inline structs (C has both). Personally I don't think that they are very useful on their own, but when combined with newtypes they are a powerful tool. So newc will have inline declarations of *all* types other than primitives.

C allows programmers to use anonymous `struct`s in `union`s. For example:

```C
// C
union MyUnion {
  struct {
    float x;
    float y;
    float z;
  };
  float data[3];
};
```

This is very useful, so newc will allow programmers to do this. However C also allows programmers to write:

```C
// C
struct MyStruct {
	union {
		float a;
		int b;
	};
	int c;
};
```

Although this is no doubt useful sometimes, I think it causes more confusion than it is worth. When using this `struct`, how is a programmer supposed to know that the fields `a` and `b` occupy the same memory, but `c` gets its own. Forcing the programmer to put the union in its own field would at least group together the fields of the union. Therefore newc will not allow programmers to do *this*.

Finally, anonymous tuples will be allowed in both structs and unions. Note that in:

```c
// newc
union MyUnion {
	struct {
		a: int32,
		b: int32,
	}
	(
		int32,
		int32
	)
}
// &my_union.a == &my_union.0
// &my_union.b == &my_union.1
```

#### Newtypes

Many languages have [newtypes](https://doc.rust-lang.org/rust-by-example/generics/new_types.html), because they provide a good way to separate what data *is*, from what it *represents* (which helps avoid muddling values with the same type but different purposes). But neither Rust nor C implements them as a language feature (they are just design patterns). I think they deserve their own syntax.

#### Type Sets and Newtype Trees

In newc, all types are grouped into sets. Types in the same set are said to be 'parallel'. These sets are organised into newtype trees. The root set of each newtype tree contains one inline type, from which all types in the tree have been derived. Types from sets nearer to root of the tree are said to be 'upstream' of types from sets deeper into the tree (and 'downstream' vice verca).
These types behave according to the [casting rules](#newtype-casts).
To create a newtype parallel (in the same type set) to another type, use the `newtype` keyword:

```c
newtype MyType = T;
```

To create a new type set as a child of another type set, use the `subtype` keyword:

```c
subtype MyType = T;// Where T is any type from the parent type set
```

Note that each time `subtype` is used it creates a new type set

#### Type Declarations

Structs can be declared in the exact same way as Rust structs, but this is syntactic sugar for:

```c
newtype Foo = struct { ... };
```

The same syntax works for `tuple`s, `enum`s, `union`s, and `sum`s.

#### Integers

Rust's integers are great, their names are much clearer and more consistent that integers in C. The only problem I have with them is that there is no 'default' integer (`int` in C), which can be confusing for beginners with no idea what integer they want to use; but this isn't a huge issue and fixing it would require an inconsistent naming system.
These are the integers in newc:

- `int8`
- `int16`
- `int32`
- `int64`
- `intsize`

Each of these and be prefixed with `u` 	for the corresponding unsized integer type.

#### Integer Literals

The exact type of integer literals can be inferred.

#### Floats

Rust's floats are also great, I have nothing to change, except the names, which will closer match newcs integer names:

- `float16`
- `float32`
- `float64`

#### `char`

The `char` type that C programmers use for all 8-bit integers will be replaced with a Rust-style `char`, so it will be four bytes long and guaranteed to hold a valid ['Unicode Scalar Value'](https://www.unicode.org/glossary/#unicode_scalar_value). However as newc is a fundamentally unsafe language, this will be difficult to enforce as such, these restriction will be placed on the `char` type:

- `char`s are immutable and can only be treated as integers by casting them to a 32-bit integer type
- 32-bit integer types can only be cast into `char`s using an [unreliable cast](#unreliable-casts) or an [unsafe cast](#unsafe-casts)

#### Strings

C strings are just an array of bytes, with a 00 byte somewhere denoting the end of the string. The makes strings one of the most unsafe things in C. Additionally, no guarantees are made about the encoding used, which can make working with C strings tricky.

Rust strings are massively better, but have their own problems.
Rust has two* main string types:

- `String` A heap allocated, growable string
- `&str` A string slice

**there is also  `str` which is unsized and therefore cannot be used directly*
These are nice to work with, but their names give no suggestion to what each actually is. Instead strings in newc will be defined as follows:

```c
subnewtype HStr = Box<uint8[]>;
subnewtype StrSlice = Slice<uint8>;
```

### Inline return

Rust allows developers to return from a function or block by dropping the `;` from the end of an expression. I don't like this syntax, so newc won't support it, however it is not quite the same as `return`, as it allows for returning a value from a block, whereas `return` will always return from the function it is in:

```rust
fn foo() {
	println!("{}", {
		do_something();
		0
	});// prints zero
}
fn bar() {
	println!("{}", {
		do_something();
		return 0;
	});// doesn't print anything
}
```

Rust programmers *have* found a work around to this using break:

```rust
fn bar() {
	println!("{}", 'a: {
		do_something();
		break 'a 0;
	});
}
```

But labelling blocks is messy and annoying. Instead newc will change how `break` works, in newc `break` refers to the highest level block it is in, unless the block is specified with a label, or with the block's keyword (`break for;`).

### Operator Modifiers

newc has three 'modifiers' that allow programmers the highest level of control over how their code works.
Take `a[b]` as an example:
b *could* be out of range to index a, how should this be handled?

- this possibility can be ignored, as it is in C, and the compiler should just trust that the programmer ensures b is in range. Resulting in undefined behaviour if the operation *was* invalid
- this can be checked for at runtime, and result in a crash
- this can be checked for at runtime, and return a `Result`

The same three things can apply to many operations that can fail. In newc, programmers can suffix operations with `~` (undefined behaviour), `?` (`Result`) or nothing at all (crash)

### Casts

All casts in newc are 'explicit', but in most cases they can be made very short by the compiler inferring the destination type, so the programmer normally only needs to add a single character before the data.
There are four types of cast:

- **Reliable Casts**<a name="reliable-casts"></a>
  Some types can be cast to other types with zero risk of any data loss/undefined behaviour. For example, an `int8` will always be able to be safely converted to an `int16`. The `#` operator can be used to reliably cast with the type inferred, or with the type explicit: `#<T>`.
- **Unreliable Casts**<a id="unreliable-casts"></a>
  Other types can *sometimes* be safely converted, for example an `int32` will sometimes be valid as a `char`, but other times it will not be. Unreliable casts return a `Result` because they may fail. The `#?` operator can be used to reliably cast with the type inferred, or with the type explicit: `#?<T>`.
- **Unsafe Casts**<a id="unsafe-casts"></a>
  Unsafe casts are a lot like unreliable casts, but do not return `Result`s because it is left up to the programmer to ensure the cast is valid. This results in undefined behaviour or a crash if the cast is invalid. The `#~` operator can be used to reliably cast with the type inferred, or with the type explicit: `#~<T>`.
- **Bit Casts**<a id="bit-casts"></a>
  Bit casts are casts where no actual data conversion is performed, they simply exist to tell the type system to treat one type as another. They don't get their own syntax but can be achieved with `union`s or with `bit_cast(value)`

#### Integer Casts

In C integers are implicitly converted according to 'integer promotion' and 'usual arithmetic conversions'. However in rust, most integer operations require both operands to have the same type, leaving the programmer to cast one or both of them manually.
I want to minimise implicit operations, as one of the attractions of C is that there is very little going on between the code you write and the instructions the CPU runs. So in order to make these conversions explicit without making them too verbose, the programmer should just use the `$` operator to [reliably cast](#reliable-casts) one integer type into the other type.

#### Pointer Casts

In C any pointer can be cast to any other pointer, however the only pointer that should usually be cast to and from is a `void*`, the casting rules in newc reflect this:

- Any pointer can be [reliably cast](#reliable-casts) to a `&void`.
- `&void` can be cast to any other pointer type with an [unsafe cast](#unsafe-casts)*.
- Finally *any* pointer type can be cast to *any* other pointer type using a [bit cast](#bit-casts).

*note that you can't use an [unreliable cast](#unreliable-casts) because there is no way to know if the memory address pointed to is valid.

#### Newtype Casts

*For terminology see [newtypes](#type-sets-and-newtype-trees).*
Casting of newtypes follows these rules:

- Parallel newtypes may be cast between each other with a [reliable cast](#reliable-casts).
- A downstream newtype may be cast to an upstream newtype with a [reliable cast](#reliaible-casts).
- An upstream newtype may be cast to a downstream newtype with an [unsafe cast](#unsafe-casts)

### Boolean vs Bitwise operators

Many languages distinguish between binary and boolean operations (e.g. `!` is boolean not, but `~` is bitwise not).
The main argument for this is that they mean very different things to the computer:
`!true == false`
however true is represented by `1`, and false by `0`, and a *bitwise* not of 1 is:
`~0b00000001 == 0b11111110`
This is especially apparent in C, where `true` isn't even a keyword, and is actually a macro that expands to `1`, so in C:

```C
!true == false
~true == -2
```

However, I would argue that this is only a problem if `true == 1` and `false == 0`. Otherwise, it makes perfect sense `!` to mean different things for `uint8` and `bool`, and this isn't really any different to `+` meaning a different thing for `int32` and `float32`. An extra advantage of this is that it frees up `~` to be used a modifier.
So newc will have the exact same operators a Rust:

- Lazy and: `&&`
- Lazy or: `||`
- Regular and: `&`
- Regular or: `|`
- Xor: `^`
- Not: `!`
- Shift left: `<<`
- Shift right: `>>`

The shifts behave as logical shifts when applied to unsigned integers, but arithmetic shifts when applied to signed integers.

### Attributes

Rust attributes normally use `#[attribute]` before the 'item' (these are almost always declarations of some kind, e.g. structs, functions, function parameters) they refer to. However `#![attribute]` is also supported, and instead refers to the item that the attribute is declared within.
This is fine, but I find this syntax a little weird. It isn't similar to anything else in Rust.
newc will use `.attribute(args)` *after* the item in question. I am aware that this syntax is a little weird too when applied to function parameters: `fn foo(bar: Baz.attribute(args))`, because it looks like the attribute applies to the datatype. So instead, newc will use `fn foo(bar.attribute(args): Baz)` for attributes on function parameters.
This syntax also frees up `#` to be used elsewhere.

### Macros vs Preprocessor

C achieves a level of compile time scripting with it's preprocessor. The preprocessor tends to get used in one of two ways:

- To make header files less bad
- As a poor substitute for various features that C simply doesn't have (e.g. generics)

Since neither of these problems will apply to newc, there will be no preprocessor.

Instead of a preprocessor, Rust has macros, which are a very powerful to way extend the functionality of Rust, they are split into two categories: 'declarative macros' and 'procedural macros'.

Declarative macros are based on pattern matching. Rust's normal pattern matching system isn't designed to handle Rust code, so declarative macros introduce a whole new set of pattern matching syntax (which shares some similarities with regex) that makes declarative macros look intimidating. It's practically a whole different language.

Procedural macros were added later, and use regular rust syntax to manipulate a `TokenStream`. They are very flexible, but have some quirks:

- They are 'unhygenic':
  code emitted by the macro is exposed to the external scope it was called in, which can lead to identifier clashes
- They receive a `TokenStream` (instead of a fragment of AST), which must be parsed (almost always by the [syn](https://docs.rs/syn/latest/syn/) crate) before it can be used.
- They are horrible to write without making use of the [quote](https://docs.rs/quote/latest/quote/) crate, which converts fragments of Rust code back into `TokenStream`s that can be returned from the macro

#### Declarative macros in newc

I initially wanted to do away with declarative macros, and improve procedural macro writing to be equally convenient. However this is impractical: procedural macros *must* be compiled before everything else, and this requires them to be in separate files so the compiler knows where to find them (unless the compiler just searches through everything, which would be slow).
In the end I have decided to keep the distinction between declarative and procedural macros, but to allow developers to use declarative macro syntax inside procedural macros.

There are two ways I could achieve this, either

- Declarative macros use syntax that is a subset of regular newc syntax
- Declarative macros have their own syntax and procedural macros allow programmers to embed declarative macros syntax

I decided to go with the latter, because it would be very confusing if declarative macros *look* too similar to regular newc code, which would lead developers to be confused when they find that they can't do what they expect. That said, I want to keep declarative macro syntax somewhat self explanatory to anyone familiar with newc (which Rust declarative macros completely fail to do):

1. Declarative macros will use `[Range]` to show something that can repeat
2. Declarative macros will use `Option<Pattern>` to show something is optional
3. Metavaraiables will be prefixed with `#` (`$` would be too easily confused with casts)
4. Compile time control flow with `#` before the keyword
5. Macros only have one pattern, this makes it look more like function arguments

Here is a comparison of the example macro given in the [rust book](https://doc.rust-lang.org/book/ch19-06-macros.html#declarative-macros-with-macro_rules-for-general-metaprogramming):

Rust

```rust
#[macro_export]
macro_rules! vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```

newc

```rust
macro vec[#x:expr[0..]] {
	let mut temp_vec = Vec::new();
	#for #a in #x {
		temp_vec.push(#a)
	};
	break temp_vec;
}
```

Macros with multiple patterns can be achieved with:

```rust
macro foo(#x:tt) {
	#match #x {
		(x => #e:expr) => {println!("mode X: {}", #e)},
		(y => #e:expr) => {println!("mode Y: {}", #e)},
	}
}
```

I considered allowing the `#` to be dropped when it was clear that it referred to a metavariable (e.g. `#match #x`), however this could lead to extreme confusion if the same identifier is used as both a normal variable and a meta variable.

#### Procedural macros in newc

Procedural macros don't have to be in a separate crate, but they must be in their own files. The compiler will compile any files with a `.pnc` file extension (instead of `.nc`). This has the potential to slow compile times on huge projects, so if a `proc-macros` key is provided in `package.toml`, then the compiler will restrict its search to files in the directory(s) specified.

In newc, procedural macros are divided into the same three categories as in Rust:

- function-like (called with `!(args)`)
- derive (called with `.derive(Trait)`)
- attribute (called with `.macro_name(args)`)

**Function-like macros** can be declared with:

- `fn foo(x: TokenStream) -> TokenStream { ... }`
- `fn foo(x: TokenStream) -> Result<TokenStream, HStr> { ... }`
- `fn foo(x: TokenTree) -> TokenTree { ... }`
- `fn foo(x: TokenTree) -> Result<TokenTree, HStr> { ... }`

If an `Err` is returned then compilation will fail, and the compiler will display the error message, along with the location and name of the macro, and the where the macro was called.
**Derive macros** can be declared with:

- `fn foo(item: TokenTree) -> TokenTree { ... }.derive_for(name)`
- `fn foo(item: TokenTree) -> Result<TokenTree, Hstr> { ... }.derive_for(name)`

Note that `name` doesn't have to be in scope, or even be a trait. It just means that when `.derive(name)` is called on an item, the `derive` macro will run `foo` and append foo's output to the item. So derive macros don't need to output the original item, and can't make any changes to it.
**Attribute macros** can be declared with:

- `pub fn foo(args: A, item: TokenTree) -> TokenTree { ... }.attribute()`
- `pub fn foo(args: A, item: TokenTree) -> Result<TokenTree, HStr> { ... }.attribute()`

