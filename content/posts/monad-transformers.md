---
title: "Monad transformers"
description: "A way to work with different monads combined"
draft: false
author: "Rafal Gwozdzinski"
date: 2021-03-15T00:08:00+00:00
tags: []
categories: ["FSharp"]
---
<!--more-->

# Monad transformers

In previous posts we've looked into different Monad types. Today, we will
show methods for working with combinations of monads.

## General pattern

Let's say we have a monadic type `Option<T>` that is nested in `Async<T>`.
The resulting type is `Async<Option<T>>`.

We would like to be able to do some calculation on wrapped value.

Now we have to:
1. Add FSharpPlus to our project.
2. Open `FSharpPlus` and `FSharpPlus.Data` namespaces.
3. Wrap our value with monad transformer of nested monad - `OptionT<T>` in our case.
4. CreateF#+ monad expression with our calculation logic.
5. Materialize result with transformer `run` function - `OptionT.run` in our case.

## Concrete example - Asynchronous Option
Now, let's create an implementation of the pattern shown above.

```fsharp
let v1 : Async<Option<int>> = async { return Some 1 }
let v2 : Async<Option<int>> = async { return Some 2 }

let vResult = 
    monad {
        let! x1 = OptionT v1
        let! x2 = OptionT v2
        return x1 + x2
    } |> OptionT.run
      |> Async.RunSynchronously
```

In a simple case like above we can just use F#+ `lift`/`map` functions:

```fsharp
let vResult' = lift2 (+) (OptionT v1) (OptionT v2)
               |> OptionT.run
               |> Async.RunSynchronously

let vResult'' = OptionT.map2 (+) (OptionT v1) (OptionT v2)
                |> OptionT.run
                |> Async.RunSynchronously
```

## Working with values of different types

When dealing with transformers, we need all values to have the same type,
so that we can wrap them in `OptionT` transformer.

Let's say we have these values:
```fsharp
let ao1 = async { return Some 1 } // Async<Option<int>>
let ao2 = async { return Some 2 } // Async<Option<int>>
let ao3 = Some 3                  // Option<int>, but not Async
let ao4 = async { return 4 }      // Async<int>, but not Option
```

To make them all the same type, we will have to:
- wrap value `ao3` in `Async` using `async.Return`
- lift `ao4` to OptionT context using `OptionT.lift` function
Then we will be able to make calculation using monad expression, as in a previous example.
```fsharp
let aoResult = 
    monad {
        let! x1 = OptionT ao1
        let! x2 = OptionT ao2
        let! x3 = OptionT <| async.Return ao3
        let! x4 = OptionT.lift ao4
        return x1 + x2 + x3 + x4  // Or use `List.reduce (+) [x1;x2;x3;x4]`
    } |> OptionT.run
      |> Async.RunSynchronously
```

## Another example - ResultT
Similar to Async Option:

```fsharp
type AsyncResult = Async<Result<int, string>>
let ar1: AsyncResult = async { return Ok 1 }
let ar2: AsyncResult = async { return Ok 2 }

let ar = 
    monad {
        let! r1 = ResultT ar1
        let! r2 = ResultT ar2
        return r1 + r2 
    } |> ResultT.run
      |> Async.RunSynchronously

let ar'  = lift2 (+) (ResultT ar1) (ResultT ar2)
          |> ResultT.run
          |> Async.RunSynchronously

let ar'' = ResultT.map2 (+) (ResultT ar1) (ResultT ar2)
          |> ResultT.run
          |> Async.RunSynchronously
```

## Async Result with effects and value passing
Now let's see a more interesting case of Result.
Let's define couple of functions:
```fsharp
let r1 () = async { 
    printfn "r1 = %d" 1
    return Ok 1 }

let r2 x = async {
    let r2Val = x + 1
    printfn "r2 = %d" r2Val
    return Ok r2Val }

let r3 x = async {
    let r3Val = x + 1
    printfn "r3 = %d" r3Val
    return Error "Problem in r3" } 

let r4 () = async {
    printfn "r4 is run"
    return Ok () }
```

We wrap them in `ResultT`, so that we can bind returned values to variables,
or just execute effectful actions:
```fsharp
let aer = 
    monad {
        let! v1 = ResultT <| (r1 ())
        let! v2 = ResultT <| (r2 v1)
        do!       ResultT <| r3 v2 
        do!       ResultT <| r4 ()
    } |> ResultT.run 
      |> Async.RunSynchronously
```

## Multiple monadic layers
Let's say we have three layers of monads:
```fsharp
type T1 = Async<Option<Result<int, string>>> 

let v1' = async { return Some (Ok 1) } : T1
let v2' = async { return Some (Ok 2) } : T1
```

We can simplify dealing with nested transformers using function composition:
```fsharp
let combinedT = OptionT >> ResultT
let runCombined : ResultT<OptionT<T1>> -> T1 = 
    ResultT.run >> OptionT.run 
```

Now we can execute in a straightforward way:
```fsharp
let result = 
    monad {
        let! x1 = combinedT v1'
        let! x2 = combinedT v2'
        return x1 + x2 
    } |> runCombined
      |> Async.RunSynchronously
```

## Code
```fsharp
#r "nuget: FSharpPlus"
open FSharpPlus
open FSharpPlus.Data


// Async Option
let v1 : Async<Option<int>> = async { return Some 1 }
let v2 : Async<Option<int>> = async { return Some 2 }

let vResult = 
    monad {
        let! x1 = OptionT v1
        let! x2 = OptionT v2
        return x1 + x2
    } |> OptionT.run
      |> Async.RunSynchronously

let vResult' = lift2 (+) (OptionT v1) (OptionT v2)
               |> OptionT.run
               |> Async.RunSynchronously
let vResult'' = OptionT.map2 (+) (OptionT v1) (OptionT v2)
                |> OptionT.run
                |> Async.RunSynchronously


// Working with values of different types
let ao1 = async { return Some 1 }
let ao2 = async { return Some 2 }
let ao3 = Some 3
let ao4 = async { return 4 }

let aoResult = 
    monad {
        let! x1 = OptionT ao1
        let! x2 = OptionT ao2
        let! x3 = OptionT <| async.Return ao3
        let! x4 = OptionT.lift ao4
        return x1 + x2 + x3 + x4  // Or use `List.reduce (+) [x1;x2;x3;x4]`
    } |> OptionT.run
      |> Async.RunSynchronously


// Async Result 
type AsyncResult = Async<Result<int, string>>
let ar1: AsyncResult = async { return Ok 1 }
let ar2: AsyncResult = async { return Ok 2 }

let ar = 
    monad {
        let! r1 = ResultT ar1
        let! r2 = ResultT ar2
        return r1 + r2 
    } |> ResultT.run
      |> Async.RunSynchronously

let ar'  = lift2 (+) (ResultT ar1) (ResultT ar2)
          |> ResultT.run
          |> Async.RunSynchronously

let ar'' = ResultT.map2 (+) (ResultT ar1) (ResultT ar2)
          |> ResultT.run
          |> Async.RunSynchronously


// Async Result with effects and value passing
let r1 () = async { 
    printfn "r1 = %d" 1
    return Ok 1 }

let r2 x = async {
    let r2Val = x + 1
    printfn "r2 = %d" r2Val
    return Ok r2Val }

let r3 x = async {
    let r3Val = x + 1
    printfn "r3 = %d" r3Val
    return Error "Problems in r3" } 

let r4 () = async {
    printfn "r4 is run"
    return Ok () }

let aer = 
    monad {
        let! v1 = ResultT <| (r1 ())
        let! v2 = ResultT <| (r2 v1)
        do!       ResultT <| r3 v2 
        do!       ResultT <| r4 ()
    } |> ResultT.run 
      |> Async.RunSynchronously


// Multiple monadic layers
type T1 = Async<Option<Result<int, string>>> 

let v1' = async { return Some (Ok 1) } : T1
let v2' = async { return Some (Ok 2) } : T1

let combinedT = OptionT >> ResultT
let runCombined : ResultT<OptionT<T1>> -> T1 = 
    ResultT.run >> OptionT.run 

let result = 
    monad {
        let! x1 = combinedT v1'
        let! x2 = combinedT v2'
        return x1 + x2 
    } |> runCombined
      |> Async.RunSynchronously

```
