+++
title =  "Computation Expressions in F#"
description =  "Monadic operations made simpler with Computation Expressions."
date =  2021-03-12T00:05:00+00:00
template = "blog/page.html"
draft =  false

[taxonomies]
authors = ["Rafał Gwoździński"]
#tags =  []
#categories =  ["FSharp"]
+++
<!--more-->

# Computation Expressions in F#
[Computation expression](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions)
is a syntax to express computations with monads (as well as with monoids, monad transformers and applicative functors).
Let's see how we can use it to work with monadic values. We will
try to figure out both easy and more complicated cases (e.g. nested values).
We will use FSharpPlus library that provide set of [general monad expressions](https://fsprojects.github.io/FSharpPlus/computation-expressions.html),
and compare them to "plain" F# approach, to see how it differs.

## Additive monad expression
Let's start with a simple example of concatenation.
We have two lists and a function that takes a list and returns a new list.
```fs
let l1 = [1;2;3]
let l2 = [4;5;6]
let listFun  = bind (fun x -> [x;x;x])
```
A simple way to concatenate them is to use `@` operatior
```fs
let lConcat = l1 @ listFun l2
```

When dealing with more complicated examples, like concatentation of both lists and plain values,
we can use list computation expression from F# core library.
```fs
let lConcat' = 
    [ yield! l1
      yield! listFun l2
      yield 42 ] 
```

Instead of dedicated expression, we can just use generic additive monad expression. 
It works for seq and array types too.
```fs
let lConcat'' = 
    monad.plus {
        return! l1
        return! listFun l2
        return 42 }
```

## Effectful monad expression
Second type of generic monad expression is used to handle monads that have embedded side-effects.

### Async 
First, let's look at `Async` monad handling.
Plain approach uses dedicated `async` expression:
```fs
let async1 = async { return 1 }
let async2 = async { return 2 }

let aResult = 
    async {
        let! x1 = async1
        let! x2 = async2
        return x1 + x2
    } 
```

We can substitute it with generic F#+ `monad` expression:
```fs
let aResult' = 
    monad {
        let! x1 = async1
        let! x2 = async2
        return x1 + x2 
    } 
```

Performing this operations without computation expressions is cumbersome.
We substitute `let!` with `async.Bind` and `return` with `async.Return`, which
gives us:
```fs
let aResult'' = async.Bind(async1, 
                    fun x1 -> async.Bind(async2, 
                                 fun x2 -> async.Return (x1 + x2)
```

We can make it a bit cleaner with F#+ operators:
```fs
let aResult''' = async1 >>= fun x1 -> 
                    async2 >>= fun x2 -> 
                        x1 + x2 |> async.Return
```

But for simple operations we can just use F#+ `map2`
```fs
let aResult'''' = Async.map2 (+) async1 async2
```
or `lift2`
```fs
let aResult''''' = lift2 (+) async1 async2
```

### Option
There is no `Option` computation expression in F# core, so we have to use F#+:
```fs
let opt1 = Some 1
let opt2 = Some 2

let oResult = monad {
    let! x1 = opt1
    let! x2 = opt2
    return x1 + x2 }
```

We can use `bind`/`map`/`lift` as with async:
```fs
let oResult' = 
    opt1 |> Option.bind (fun x1 -> 
        opt2 |> Option.bind (fun x2 -> x1 + x2 |> Some)) 

let oResult''  = lift2       (+) opt1 opt2
let oResult''' = Option.map2 (+) opt1 opt2
```

We can also pattern match:
```fs
let oResult'''' =
    match (opt1, opt2) with
    | (Some opt1, Some opt2) -> Some (opt1 + opt2)
    | (_,_)                  -> None
```

Computation expressions are especially useful for complicated operations:
```fs
let opt3 = Some 3
let add1 = (+) 1
let isEven x = if x % 2 = 0                                 
               then Some x 
               else None
let multiplyBy4 = (*) 4

let harderOptResult = 
    monad {
        let! x1 = opt3
        let! even = x1 |> add1 |> isEven
        return multiplyBy4 even }
```

In this case `bind`/`map` approach doesn't look that bad:
```fs
let harderOptResult' = 
    opt3 
    |> Option.bind (add1 >> isEven) 
    |> Option.map multiplyBy4
```

Pattern matching on Options doesn't scale well. It gets unreadable even with one level of nesting.
```fs
let harderOptResult'' = 
    match opt3 with
    | None -> None
    | Some v -> v |> add1 |> isEven
                |> function 
                    | None ->   None
                    | Some e -> Some <| multiplyBy4 e
```

### Result
Works similarly to Option:
```fs
let res1 : Result<int,string> = Ok 1
let res2 : Result<int,string> = Error "Niepowodzenie"

let rResult = monad {
    let! x1 = res1
    let! x2 = res2
    return x1 + x2 }

let rResult'  = Result.map2 (+) res1 res2
let rResult'' = lift2       (+) res1 res2 
```

Pattern matching on Results is laborous, because we have to handle both errors manually:
```fs
let rResult''' =
    match (res1, res2) with
    | (Ok res1, Ok res2) -> Ok <| res1 + res2
    | (Error e ,_)       -> Error e
    | (_, Error e)       -> Error e
```

### Effectful Result
Let's see what happens when we have effectful function that returns Result of unit. 
Instead of `let!`, we use `do!` keyword.
```fs
let eRes1 () = 
    printfn "computing res1"
    Ok ()

let eRes2 () = Error "Could not compute res2"

let effResult = 
    monad {
        do! eRes1 ()
        do! eRes2 () }
```

Without computation expression, we get code thats pretty awkward
```fs
let effResult' = eRes1 () 
                 |> Result.bind (fun () -> eRes2 () 
                                           |> Result.bind (fun () -> Ok ()))
```

F#+ `bind` operator doesn't help
```fs
let effResult'' = eRes1 () 
                  >>= fun () -> eRes2 () 
                                >>= (fun () -> Ok ())
```

## Code
```fs
#r "nuget: FSharpPlus"

open FSharpPlus

// Additive monad expression

let l1 = [1;2;3]
let l2 = [4;5;6]
let listFun  = bind (fun x -> [x;x;x])

let lConcat = l1 @ listFun l2

let lConcat' = 
    [ yield! l1
      yield! listFun l2
      yield 42 ] 

let lConcat'' = 
    monad.plus {
        return! l1
        return! listFun l2
        return 42 }

// Effectful monad expression

//// Async
let async1 = async { return 1 }
let async2 = async { return 2 }

let aResult = 
    async {
        let! x1 = async1
        let! x2 = async2
        return x1 + x2
    } 
aResult |> Async.RunSynchronously

let aResult' = 
    monad {
        let! x1 = async1
        let! x2 = async2
        return x1 + x2 
    } 
aResult' |> Async.RunSynchronously

let aResult'' = async.Bind(async1, 
                    fun x1 -> async.Bind(async2, 
                                 fun x2 -> async.Return (x1 + x2)))
aResult'' |> Async.RunSynchronously

let aResult''' = async1 >>= fun x1 -> 
                    async2 >>= fun x2 -> 
                        x1 + x2 |> async.Return
aResult''' |> Async.RunSynchronously

let aResult'''' = Async.map2 (+) async1 async2
                  |> Async.RunSynchronously

let aResult''''' = lift2 (+) async1 async2
                   |> Async.RunSynchronously


//// Option
let opt1 = Some 1
let opt2 = Some 2

let oResult = monad {
    let! x1 = opt1
    let! x2 = opt2
    return x1 + x2 }

let oResult' = 
    opt1 |> Option.bind (fun x1 -> 
        opt2 |> Option.bind (fun x2 -> x1 + x2 |> Some)) 

let oResult''  = lift2       (+) opt1 opt2
let oResult''' = Option.map2 (+) opt1 opt2

let oResult'''' =
    match (opt1, opt2) with
    | (Some opt1, Some opt2) -> Some <| opt1 + opt2
    | (_,_)                  -> None


//// Option - complicated
let opt3 = Some 3
let add1 = (+) 1
let isEven x = if x % 2 = 0                                 
               then Some x 
               else None
let multiplyBy4 = (*) 4

let harderOptResult = 
    monad {
        let! x1 = opt3
        let! even = x1 |> add1 |> isEven
        return multiplyBy4 even }

let harderOptResult' = 
    opt3 
    |> Option.bind (add1 >> isEven) 
    |> Option.map multiplyBy4

let harderOptResult'' = 
    match opt3 with
    | None -> None
    | Some v -> v |> add1 |> isEven
                |> function 
                    | None ->   None
                    | Some e -> Some <| multiplyBy4 e

//// Result
let res1 : Result<int,string> = Ok 1
let res2 : Result<int,string> = Error "Niepowodzenie"

let rResult = monad {
    let! x1 = res1
    let! x2 = res2
    return x1 + x2 }

let rResult'  = Result.map2 (+) res1 res2
let rResult'' = lift2       (+) res1 res2 

let rResult''' =
    match (res1, res2) with
    | (Ok res1, Ok res2) -> Ok <| res1 + res2
    | (Error e ,_)       -> Error e
    | (_, Error e)       -> Error e


//// Effectful Result
let eRes1 () = 
    printfn "computing res1"
    Ok ()

let eRes2 () = Error "Could not compute res2"

let effResult = 
    monad {
        do! eRes1 ()
        do! eRes2 () }

let effResult' = eRes1 () 
                 |> Result.bind (fun () -> eRes2 () 
                                           |> Result.bind (fun () -> Ok ()))

let effResult'' = eRes1 () 
                  >>= fun () -> eRes2 () 
                                >>= (fun () -> Ok ())
```
