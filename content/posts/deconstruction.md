+++
title =  "Construction and deconstruction of F# values"
description =  "Different ways to create F# values and retrieve their contents"
date =  2021-03-01T11:00:00+00:00
template = "blog/page.html"
draft =  false

[taxonomies]
authors = ["Rafał Gwoździński"]
#tags = []
#categories = ["fsharp"]
+++

<!--more-->

# Construction and deconstruction of F# values

## Simple type deconstruction

Let's say we have a simple type (aka [Single case discriminated union type](https://fsharpforfunandprofit.com/posts/designing-with-types-single-case-dus/)) that wraps an int.
```fs 
type SimpleInt = SimpleInt of int
let simpleInt = SimpleInt 42
```

We have created a value and bound it to `simpleInt` variable.
How can we retrieve it's content?

#### Anonymous function deconstruction
We can deconstruct `SimpleInt` in lambda expression
```fs
let intValue = simpleInt |> fun (SimpleInt s) -> s
```

#### Let binding deconstruction
We can deconstruct it in `let` assignment
```fs
let (SimpleInt intValue') = simpleInt
```

#### Dedicated value retrieving function 
We can add a module with the type name that contains deconstructing function.
This is a bit more verbose, but can be handy, especially when we use it in many places and are tired of redundant lambdas.
``` fsharp
module SimpleInt =
    let value (SimpleInt s) = s

let intValue'' = simpleInt |> SimpleInt.value
```

#### Direct deconstruction of function parameter
We can also unwrap value directly in a function that uses it.
```fs
module SimpleIntAdder =
    let addSimpleInts (SimpleInt i1) (SimpleInt i2) = 
        SimpleInt (i1 + i2)
```

### Parallel between construction and deconstruction
There is a parallel between notation of construction and deconstruction of simple types.
In both cases they are written as `({Type Name} {Value})`
```fs
let wrapped = (SimpleInt 42)
let (SimpleInt unwrapped) = wrapped
```

## Record types
Now, let's see what happens with records.
Let's define and construct a value for a simple record type.
```fs
type SimpleRecord = 
    { First: SimpleInt
      Second: int }
let simpleRecord = 
    { First = SimpleInt 42
      Second = 1 }
```

### Deconstruction similar to simple types
We can deconstruct record in: 
- lambda expression:
```fs
// Remark: Added type annotations to r1, r2 for clarity
let (r1: SimpleInt), (r2: int) = 
    simpleRecord 
    |> fun {First = x; Second = y} -> (x, y)
```

- let assignment:
```fs
let {First = r1'; Second = r2'} = simpleRecord
```

- function parameter:
```fs
// Remark: We deconstruct both record and it's internal field (which is of type SimpleInt)
module SimpleRecordAdder =
    let addRecordFields {First = (SimpleInt f); Second = s} = f + s
```

There is a parallel between construction and deconstruction of records similar to simple types:
```fs
let wrappedRecord = {First = SimpleInt 42; Second = 1}
let {First = (SimpleInt f); Second = s} = wrappedRecord 
```

#### Partial deconstruction
If we don't need all of record fields, we can use partial deconstruction in all previously defined cases.
```fs
let {Second = s'} = wrappedRecord
let s'' = wrappedRecord |> fun {Second = x} -> x

module SecondFieldAdder =
    let addOneToSecondField {Second = s} = s + 1

let {First = (SimpleInt f')} = wrappedRecord
let f'' = wrappedRecord |> fun {First = (SimpleInt x)} -> x

module FirstFieldAdder =
    let addOneToFirstField {First = (SimpleInt f)} = f + 1
```

## Pattern matching

Pattern matching is one of the most popular usage of deconstruction.

### Simple type
```fs
type MyInt = MyInt of int
let myVal = MyInt 42

let valueIfNotZero = 
    match myVal with 
    | MyInt 0 -> None
    | MyInt x -> Some x
```

### Record type
We can match on all record fields:
``` fsharp
type MyRecord = { First: int; Second: int }
let myRec = {First = 1; Second = 2}

// Full
let addIfNotZeros =
    match myRec with
    | {First = 0; Second = 0} -> None
    | {First = x; Second = y} -> Some (x + y)
```

Or we can use partial deconstruction
```fs
let getSecondIfNotZero =
    match myRec with
    | {Second = 0} -> None
    | {Second = x} -> Some x
```

### List
Lists offer a refined ways of deconstruction. 
Let's define a simple list and see how we can match on it.
```fs
let myList = [1;2;3;4;5]
```

#### Exact match
```fs
let is12345 =
    match myList with
    | [1;2;3;4;5] -> true
    | _ -> false
```

#### Match exact number of items
```fs
let sum5OrZero =
    match myList with
    | [a;b;c;d;e] -> a+b+c+d+e
    | _ -> 0
```

#### Match head/tail
We can match head/tail with `h::t` construct.
Head is the first element of the list, Tail is a list of all further elements.

Example use cases:
- Exact head 
```fs
let is1First =
    match myList with
    | 1::t -> true
    | _ -> false
```

- Exact tail
```fs
let is2345Tail =
    match myList with
    | h::[2;3;4;5] -> true
    | _ -> false
```

- Deconstruct head
```fs
let head =
    match myList with
    | h::t -> Some h
    | _ -> None
```

- Deconstruct tail
```fs
let skipHead =
    match myList with
    | h::t -> t
    | _ -> []
```

- Empty or not?
```fs
let isEmpty =
    match myList with
    | [] -> true 
    | _ -> false
```

- One element
```fs
let isSingle =
    match myList with
    | [x] -> true
    | _ -> false
```

### Arrays
An Array structure offers fewer ways of matching than list.

We construct an array as:
```fs
let myArray = [|1;2;3;4;5|]
```

#### Cases similar to List type 
- Exact match
```fs
let is12345' =
    match myArray with
    | [|1;2;3;4;5|] -> true
    | _ -> false
```

- Deconstruct exact number of items
```fs
let sum5OrZero' =
    match myArray with
    | [|a;b;c;d;e|] -> a+b+c+d+e
    | _ -> 0
```

- Empty?
```fs
let isEmpty' =
    match myArray with
    | [||] -> true
    | _ -> false
```

- One element
```fs
let isSingle' =
    match myArray with
    | [|x|] -> true
    | _ -> false
```
#### Head/tail
Unfortunately, there is no built-in way to match `h::t` on arrays in F#.

## Example Code
To get a better feel for concepts described in this post, I encourage readers to try this code on their own machines.
The easiest way is to put it into `fsx` script and run using REPL.
```fs
////////////////////////////////////
//// Simple type deconstruction ////  
////////////////////////////////////

type SimpleInt = SimpleInt of int
let simpleInt = SimpleInt 42

// We created a value in a usual way and bound it to `simpleInt` variable.
// How can we retrieve a value from it?

// *Anonymous function deconstruction
//  We can deconstruct `SimpleInt` in lambda expression
let intValue = simpleInt |> fun (SimpleInt s) -> s

// *Let binding deconstruction
//  We can deconstruct it in `let` assignment
let (SimpleInt intValue') = simpleInt

// *Dedicated value retrieving function 
//  We can add a module with the type name that contains deconstructing function.
//  This is a bit more verbose, but can be handy, especially when we use it in many places and are tired of redundant lambdas.
module SimpleInt =
    let value (SimpleInt s) = s

let intValue'' = simpleInt |> SimpleInt.value

// *Direct deconstruction of function parameter
//  We can also unwrap value directly in a function that uses it.
module SimpleIntAdder =
    let addSimpleInts (SimpleInt i1) (SimpleInt i2) = 
        SimpleInt (i1 + i2)

//// Parallel between construction and deconstruction
// There is a parallel between notation of construction and deconstruction of simple types.
// In both cases they are written as `({Type Name} {Value})`
let wrapped = (SimpleInt 42)
let (SimpleInt unwrapped) = wrapped

//////////////////////
//// Record types ////
//////////////////////

// Now, let's see what happens with records.
// Let's define and construct a value for a simple record type.
type SimpleRecord = 
    { First: SimpleInt
      Second: int }
let simpleRecord = 
    { First = SimpleInt 42
      Second = 1 }

//// Deconstruction similar to simple types

// We can deconstruct record in: 
// *lambda expression:
//  Remark: Added type annotations to r1, r2 for clarity
let (r1: SimpleInt), (r2: int) = 
    simpleRecord 
    |> fun {First = x; Second = y} -> (x, y)

// *let assignment:
let {First = r1'; Second = r2'} = simpleRecord

// *function parameter:
// Remark: We deconstruct both record and it's internal field (which is of type SimpleInt)
module SimpleRecordAdder =
    let addRecordFields {First = (SimpleInt f); Second = s} = f + s

// There is a similar parallel between construction and deconstruction of records:
let wrappedRecord = {First = SimpleInt 42; Second = 1}
let {First = (SimpleInt f); Second = s} = wrappedRecord 

//// Partial deconstruction
// If we don't need all of record fields, we can use partial deconstruction in all previously defined cases.
let {Second = s'} = wrappedRecord
let s'' = wrappedRecord |> fun {Second = x} -> x

module SecondFieldAdder =
    let addOneToSecondField {Second = s} = s + 1

let {First = (SimpleInt f')} = wrappedRecord
let f'' = wrappedRecord |> fun {First = (SimpleInt x)} -> x

module FirstFieldAdder =
    let addOneToFirstField {First = (SimpleInt f)} = f + 1

//////////////////////////
//// Pattern matching ////
//////////////////////////
   
// Pattern matching is one of the most popular usage of deconstruction.

///// Simple type

type MyInt = MyInt of int
let myVal = MyInt 42

let valueIfNotZero = 
    match myVal with 
    | MyInt 0 -> None
    | MyInt x -> Some x

//// Record type
//   We can match on all record fields:
type MyRecord = { First: int; Second: int }
let myRec = {First = 1; Second = 2}

// Full
let addIfNotZeros =
    match myRec with
    | {First = 0; Second = 0} -> None
    | {First = x; Second = y} -> Some (x + y)

// Or we can use partial deconstruction
let getSecondIfNotZero =
    match myRec with
    | {Second = 0} -> None
    | {Second = x} -> Some x

//// List
//   Lists offer a refined ways of deconstruction. 
//   Let's define a simple list and see how we can match on it.
let myList = [1;2;3;4;5]

// *Exact match
let is12345 =
    match myList with
    | [1;2;3;4;5] -> true
    | _ -> false

// *Match exact number of items
let sum5OrZero =
    match myList with
    | [a;b;c;d;e] -> a+b+c+d+e
    | _ -> 0

// *Match head/tail
//  We can match head/tail with `h::t` construct.
//  Head is the first element of the list, Tail is a list of all further elements.
//  Example use cases:
// Exact head 
let is1First =
    match myList with
    | 1::t -> true
    | _ -> false

// Exact tail
let is2345Tail =
    match myList with
    | h::[2;3;4;5] -> true
    | _ -> false

// Deconstruct head
let head =
    match myList with
    | h::t -> Some h
    | _ -> None

// Deconstruct tail
let skipHead =
    match myList with
    | h::t -> t
    | _ -> []

// Empty or not?
let isEmpty =
    match myList with
    | [] -> true 
    | _ -> false

// One element
let isSingle =
    match myList with
    | [x] -> true
    | _ -> false

//// Arrays
//   An Array structure offers fewer ways of matching than list.
//   We construct an array as:
let myArray = [|1;2;3;4;5|]

// Cases similar to List type 
// *Exact match
let is12345' =
    match myArray with
    | [|1;2;3;4;5|] -> true
    | _ -> false

// *Deconstruct exact number of items
let sum5OrZero' =
    match myArray with
    | [|a;b;c;d;e|] -> a+b+c+d+e
    | _ -> 0

// *Empty?
let isEmpty' =
    match myArray with
    | [||] -> true
    | _ -> false

// *One element
let isSingle' =
    match myArray with
    | [|x|] -> true
    | _ -> false
```
