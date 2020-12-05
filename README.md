# MiniScript RFC

This RFC proposes an idea for a new open source programming language called MiniScript. Please let me know if you have any feedback!

## What is MiniScript?

MiniScript is a typed functional programming language that is also a _subset_ of JavaScript - **all modern JavaScript environments (like Node or the browser) can run MiniScript without needing a build step**.

JavaScript has proven to be a popular language, but due to its history over the years it's been left with a lot of bloat and complexity. MiniScript aims to simplify "the good parts", whilst also adding a powerful type system.

MiniScript uses a Hindley-Milner type system to infer your types. Writing types out is entirely optional, but your MiniScript code will always be type-checked for runtime errors. To remain valid JavaScript, types are written in comments.

MiniScript has a different mental model to JavaScript. There are fewer concepts to learn and choose from - there should only be one clear way of doing things. For example there is no `var` keyword, no concept of prototypes and classes, and an `Option` type replaces `undefined` and `null`.

Whilst being a different language to JavaScript, MiniScript is designed in a way that all modern JavaScript compilers _just happen_ to produce the correct output. This allows MiniScript to take advantage of the large ecosystem of JavaScript tools and libraries.  Developers familiar with JavaScript can still read your code, even if they're new to MiniScript.

## Example

```js
const add = (a, b) => a + b;
```

Since in MiniScript the `+` operator can _only_ be used with numbers, MiniScript is able to automatically infer the types of `a` and `b` as numbers. The types can also be made explicit through a comment:

```js
// add :: (Number, Number) -> Number
const add = (a, b) => a + b;
```

## Demo

https://miniscript-demo.netlify.app

This is an unfinished prototype of MiniScript in an interactive editor. You can hover over variables to see their inferred type.

90% of the features are missing or not implemented the same as the spec. There are also lots of bugs! This demo only provides some cherry-picked examples to give you a preview of how the language could work.

# Spec

This is an informal description of the MiniScript language. Unless stated otherwise, any features mentioned work the same as JavaScript (ECMAScript 2020). Any JavaScript feature not mentioned can be assumed to _not_ be supported in MiniScript.

## Variables

Variables are declared using `const` or `let`. If declared using `const`, the variable is both unreassignable _and_ immutable.

```js
let y = { name: "MiniScript" };
y.name = "oops";
// --- This is ok

const x = { name: "MiniScript" };
x.name = "oops";
// --- Error: An immutable `const` variable cannot be mutated.
```

If a value is declared with `let`, it is a reference:

```js
let x = 5; // Ref<Number>
const y = x + 2; // Number
```

References act just like non-reference variables and work seamlessly with them. But by being explicit with their type, we can track which variables can be mutated / reassigned and which can't.

## Type Variables

Types can be defined in a comment block at the top of a file. There is a single `type` keyword. Type names must be written in PascalCase.

```js
/*
type Num = Number;
*/
```

## Functions

Functions must be defined using `const` - there is no `function` keyword. Arguments of functions act like `const` variables, therefore you cannot mutate something passed into a function.

The function's type can be _optionally_ declared with a comment on the line above the function:

```js
// add :: (Number, Number) -> Number
const add = (a, b) => a + b;
```

The MiniScript type checker will show an error if the inferred type of the function isn't compatible with the annotated type of the function.

Functions can also be defined with a block and `return` statement.

```js
const add = (a, b) => {
  return a + b;
}
```

Nested functions can also be annotated with a comment:

```js
const add = (a, b) => {
  // innerFunc :: () -> Number
  const innerFunc = () => 5;
  return a + b;
}
```

Function arguments act the same as `const` variables. To mutate the function argument, the `Ref` type can be used for records:

```js
// [impure] mutate :: (Ref<{ x: Number }>) -> Nullish
const mutate = (a) => {
  a.x = 5;
}
```

## Polymorphism (Generics)

MiniScript has a polymorphic type system known as "let-polymorphism". This allows you to write generics like:

```js
/*
type Id<a> = a -> a;
type IdNumber = Id<Number>;
*/

// id<a> :: a -> a;
const id = x => x;
```

The type parameter `a` should be written in camelCase to distinguish it from type variables. Since type parameters are always lowercase, you can omit the `<a>` declaration for function annotations:

```js
// id :: a -> a;
const id = x => x;
```

## Records

Records are a _pure_ representation of data and methods, they do not hold state (i.e. no `this`). Their syntax is identical to JavaScript objects.

```js
const y = { x: true, k: 1 };
const z = y.x;
```

Record types can also be declared as type variables, and can be spread into other types.

```js
/*
type X = { x: Number };
type Y = { ...X, y: String };
*/
```

MiniScript uses row polymorphism for records. For example:

```js
const z = x => x.y + 1;
```

The type of `z` will be inferred as `z<a> :: { ...a, y: Number } -> Number`. The type parameter `a` allows for duck typing. For example in:

```js
const z = x => {
  const unused = x.y + 1;
  return x;
};
const k = z({ y: 1, p: 5 });
```

`k` will have type `{ p: Number, y: Number }`, even though the `z` function doesn't mention the `p` field.

Generics can also be applied to record methods:

```js
/*
type Array<T> = {
  length: T,
  head: () -> T,
  map<U>: ((T, Number) -> U) -> Array<U>
};
*/
```

## Primitives

MiniScript supports the JavaScript primitives `string`, `number`, `boolean`, `bigint` and `symbol`. However since these primitives have object wrappers, MiniScript treats them all as records.

For example, the `String` type is an alias for the type with all of the String object fields:

```js
/*
type String = {
  toLowerCase: () -> String
  toUpperCase: () -> String
  // etc.
}
*/
```

## Numbers

The `Number` type includes both integers and floats.

There is an additional type `Int`, which can be seen as a subset of `Number`. All arithmetic operators (`+` `-` `*` `/` etc) can be used with it, but mixing an `Int` and `Number` will result in a `Number`. An `Int` can always be passed into a function which accepts a `Number`.

```js
const x = 1; // Int
const y = 1.5; // Number

const z = Math.round(y); // Int

const array = [1, 2, 3];
array[x]; // Ok
array[y]; // Error

x + x; // Int
x + y; // Number
x * x; // Int
x / x; // Number
x * y; // Number

// getInt :: Int -> Int
const getInt = (a) => a;

// getNumber :: (Number) -> Number
const getNumber = (b) => b;

getInt(1); // Ok
getInt(1.5); // Error

getNumber(1.5); // Ok
getNumber(1); // Ok
```

The MiniScript type checker does not have a concept of `NaN` or `Infinity`, so care must be taken when handling operations like diving by 0.

## Arrays

Like primitives, arrays are also treated as records.

```js
const x = [1, 2, 3]; // Number[]
let y = [1, 2, 3, "Hello", "World"]; // (Number | String)[]

x[0]; // Number
```

Arrays declared with `const` cannot be mutated (calling `.push` will produce an error). But they can be when declared with `let`.

Looking up an array by index is assumed to be successful (for example accessing `x[100]` above will still provide a type of `Number`). As is the case in most languages, you should do a runtime check of the array length (`x.length`) to avoid runtime errors.

Arrays must also be looked up with an `Int` type.

## Optionals

MiniScript uses an optional type similar to the `Maybe` type of languages like Elm and the `Option` type of Rust.

An optional number can be defined like:

```js
// x :: Option<Number>
const x = 5;
```

Where `Option<a> = Some<a> | Nullish`.

The `Nullish` type covers both `undefined` and `null` values. It can be seen as a `None` in other languages. MiniScript has no way of distinguishing between `undefined` and `null`, so you don't need to worry about doing so either.

To define a `Nullish` value only `null` can be used:

```js
// x :: Option<Number>
const x = null;
```

To operate on the value we can use optional chaining, the nullish coalescing operator or check the runtime type:

```js
// dict :: { a: Option<{ b: Number }> }
const dict = {};

const y = dict.a?.b;
const z = (x ?? 0) + 2;

if (x !== null && x !== undefined) {

}
// or
if (typeof x === "number") {

}
```

Records with an optional field can also omit the field.

Functions which don't return a value have a return type of `Nullish` (since the runtime value returned will be `undefined`).

## `Date`

The `Date` type is represented as a record.

`Date` (as a value) is a record with static fields, and `new Date` is a special name for a function which returns a date. Note the `new` keyword is generally discouraged, and only used to interact with outside code.

## Promise

Similar to `Date`, `Promise` (as a value) is a record with static fields, and `new Promise` is a special name for a function which returns a Promise constructor.

Something of type `Promise<a>` is treated as a Record (with fields such as `then`). The `Promise` type has a type parameter:

```js
/*
type StrPromise = Promise<String>;
*/
```

## Unions

Union types are separated by a `|`.

```js
/*
type A = B | C;
*/
```

## Literal types

Types can be an integer, boolean or string literal.

```js
/*
type MyType = "hello" | "hi" | false | 0;
*/

// x :: MyType
const x = "hello";
```

## Coercion & Narrowing

There are two ways to change the type of a variable:

1. Coercion through a comment annotation when a variable is defined (if consistent with the type checker)
2. Narrowing (`typeof` or checking for values)

```js
/*
type MyType = "hello" | "hi";
*/

// x :: MyType
const x = "hello";

const y = "hi";

// getMyType :: (MyType) -> MyType
const getMyType = (z) => z;

getMyType(x); // this is ok
getMyType(y); // error since y is of type String

if (y === "hello" || y === "hi") {
  getMyType(y); // this is ok as type has been narrowed
}
```

## Unknown type

The unknown type is denoted by `?`. A variable of unknown type can be passed around, but cannot be used directly.

You can use narrowing to determine the actual type of a variable of type unknown.

```js
/*
type Box<a> = { field: a };
*/

// x :: Box<?>[];
const x = [{ field: "hi" }, { field: 0 }];

if (typeof x[0] === "string") {
  x[0]; // you can use this as a String
}
```

## Any type

There is a type called `DANGEROUS_ANY`, which can act as an escape hatch for if you really need to override the type system:

```js
// x :: DANGEROUS_ANY
const x = {};

x.hello().oops(); // no type error
```

## Decoding

Sometimes you receive data from outside of your program you need to work with, for example JSON.

`JSON.parse` has the type signature `(String) -> ?`, so type narrowing must be used to work with the data.

This can be quite cumbersome, so MiniScript will have an optional runtime library which creates decoders from your types through a compile step:

```js
/*
type MyType = { field: "hello" };
*/

import { decode } from "@miniscript/decode";

try {
  // decode<MyType>
  const x = decode(JSON.parse("{ ... }"));

} catch (e) {
  console.log("MyType did not match data");
}
```

When running the MiniScript type checker on your code, it's able to transform and output it with the type information passed into the second argument of `decode` as an object, similar to [ts-validate-type](https://github.com/edbentley/ts-validate-type).

## Switch

`switch` statements must always be exhaustive if there is no `default` field.

Every switch field must be enclosed in a `{}` block.

```js
switch (myField) {
  case "a": {
    break;
  }

  default: {
    break;
  }
}
```

## Modules

MiniScript uses ES modules syntax for all imports and exports.

Types have a similar syntax but must use the `type` keyword, and there are no default imports / exports.

```js
/*
import type { Cat, Dog } from "./my-file";

export type MyType = Number;
*/
```

## Abstract types

Sometimes you have a type that is exported, but you want to hide its internals. MiniScript's type system can enable this with the `abstract` keyword.

`file1.js`:

```js
/*
export type MyType1 = { a: Number };

export abstract type MyType2 = { a: Number };
*/

// x :: MyType1
export const x = { a: 5 };

// y :: MyType2
export const y = { a: 5 };

// add1 :: (MyType2) -> Number
export const add1 = (arg) => arg.a + 1;
```

`file2.js`:

```js
/*
import type { MyType1, MyType2 } from "./file1";
*/

import { x, y, add1 } from "./file1";

x.a + 1; // ok
y.a + 1; // type error since y is abstract type

add1(y); // ok

// x2 :: MyType1
const x2 = { a: 5 }; // this is ok

// y2 :: MyType2
const y2 = { a: 5 }; // type error since MyType2 is abstract
```

Only the file that defines an abstract type can create it, or see its internals (such as record fields).

## Effect types

MiniScript handles effects and errors in a way inspired by [Koka](https://github.com/koka-lang/koka).

When you define a simple function, it is assumed to be _pure_ (no side effects), error-free (no `throw`) and not an async function:

```js
// [] myFunc :: () -> String
const myFunc = () => {
  return "hello world";
}
```

Note that a type comment can explicitly state no effects (`[]`), or omit it to let it be inferred.

i.e. This will result in a type error:

```js
// [] myFunc :: () -> Nullish
const myFunc = () => {
  throw Error("hello world");
}
```

But this will infer the effects and thus not have a type error:

```js
// myFunc :: () -> Nullish
const myFunc = () => {
  throw Error("hello world");
}
```

This avoids you having to change the type of functions when you add something like a `console.log` somewhere nested deep in the function.

### Purity

There are a few ways to make a function impure. One way is referencing mutable variables outside of local scope:

```js
// [impure] myFunc :: () -> Nullish
const myFunc = () => {
  someArray.push(5);
}
```

This adds the `impure` label on the function. If you call an impure function, the outer function is also impure:

```js
// [impure] myFunc :: () -> Number
const myFunc = () => {
  return Math.random();
}
```

The exception to this rule is if the inner function is impure because it accesses a variable in the outer function's scope:

```js
// [] myFunc :: () -> Number
const myFunc = () => {
  let i = 0;
  const innerFunc = () => {
    i++;
  }
  return innerFunc();
}
```

Wrapping an `impure` function will label the impurity within the return type:

```js
// myFunc :: () -> ([impure] () -> Number)
const myFunc = () => {
  return () => Math.random();
}
```

This `impure` label does not affect your program in any way, but it provides information to the programmer about what the function does. Trying to annotate an impure function as pure will also produce a type error.

### Errors

Throwing something will add the `throws` label and the type being thrown to a function:

```js
// [throws Error] myFunc :: () -> Nullish
const myFunc = () => {
  throw Error("hello world");
}
```

You can avoid your outer function also being labelled with `throws` by handling it with `try` / `catch`.

```js
// [] handle :: () -> Number
const handle = () => {
  try {
    myFunc();
  } catch (e) {
    // e has type Error
  }
  return 0;
}
```

A function can have multiple labels (the order doesn't matter):

```js
// [impure, throws Number] myFunc :: () -> Nullish
const myFunc = () => {
  throw Math.random();
}
```

### Async / await

Declaring a function as async also adds an `async` label:

```js
// [async, impure] myFunc :: () -> String
const myFunc = async () => {
  return await someThing();
}
```

Which is equivalent to returning a `Promise`:

```js
// [impure] myFunc :: () -> Promise<String>
const myFunc = async () => {
  return await someThing();
}
```

### if, conditional operator, for, while

`if` statements must take a `Boolean` type - there is no concept of "truthy" values and no coercion happens.

```js
const x = 0;

// Not ok!
if (x) {

}

// Ok
if (x > 0) {

}
```

The same rule applies for the `a ? b : c` conditional operator, `for` loops and `while` loops. Note that `while` loops have no `do` version.

### Declaration files

MiniScript can import JavaScript files it is able to infer the types of. But in most cases (such as if a `class` is defined) it won't be able to, and a declaration file will need to be written. This works in the same way as [TypeScript's declaration files](https://www.typescriptlang.org/docs/handbook/declaration-files/introduction.html) but with a file extension of `.d.js` and as a large comment block.

The declaration file can also be used with npm modules and possibly adapted from TypeScript declaration files.

### Built-in types

Many global types (like `Promise`, `Date`) are built into the language. Other globals like DOM methods are defined in a declaration file. The environment of your code can be defined at the top of a file through `@browser` or `@node` flags, or in a `package.json` file through the [browser](https://docs.npmjs.com/cli/v6/configuring-npm/package-json#browser) and [main](https://docs.npmjs.com/cli/v6/configuring-npm/package-json#main) fields.

```js
/*
@browser
*/
document.getElementById("miniscript"); // ok
```

## The rest

These features all work the same as JavaScript:

- destructuring and spread operators
- regex
- comments
- `===` equality (but no `==` equality available)

## Theory

- MiniScript uses let-polymorphism (rank-1), which means that type inference is always possible without annotations. This also means that type variables cannot be declared in function arguments.
- Like Elm, records are based on the paper [Extensible records with scoped labels](https://www.microsoft.com/en-us/research/publication/extensible-records-with-scoped-labels/).
