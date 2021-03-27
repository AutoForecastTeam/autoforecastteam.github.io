---
title: "Intro to monads in F#"
description: "Use FSharpPlus library to expand F# functional capabilities."
draft: false
author: "Rafal Gwozdzinski"
date: 2021-03-03T00:08:00+00:00
tags: []
categories: ["FSharp"]
---
<!--more-->

# Intro to monads in F# 
A purpose of this post is to give you an intuition of monads and to show their implementation in
[F#+](https://github.com/fsprojects/FSharpPlus) library.

## What is monad?
It's a concept from a branch of mathematics called Category Theory.
For the purpose of this introduction we can think of it as an abstract, generic type
that implements a specific interface.
On the level of intuition, we can think of monads as "boxes" for values. However,
in some cases this intuition may be a bit of a stretch.

## Why should I bother?
Actually, you don't have to. You can easily write F# code without these kinds of mathematical concepts.
However, using monads can make you see high-level patterns that span across different data types and domains.
It will also make your code shorter and more declarative.
I think it's a worthy pursuit for anyone interested in functional programming.


## Monad "interface"
All monads should obey monad laws. We can think of them loosely as an interface. 
I will try to show the vanilla F# way of handling monads, and compare it to the F#+ way.
I would like to show that monads let us treat different types in a similar way.
We can use same approach to create and combine values in different monadic contexts.

### Unit/Return
**Type signature:** `a -> M a` where `M` is a monad type.

A function that "wraps" a value into a monad. 
We can create a monad in F#+ using `result` function.

Examples of popular monads creation in F# using both FSharp.Core API and F#+ `result`.

```fsharp
let l   = [1]
let l'  = List.singleton 1
let l'' = result 1 : int list 

let s   = seq [1]
let s'  = Seq.singleton 1
let s'' = result 1 : int seq 

let ar   = [|1|]
let ar'  = Array.singleton 1
let ar'' = result 1 : int []

let a   = async { return 1 }
let a'  = async.Return 1
let a'' = result 1 : Async<int>

let o   = Some 1
let o'  = None
let o'' = result 1 : int option

let r   = Ok 1
let r'  = Error "Oops!"
let r'' = result 1 : Result<int,string>
```

We have to add type annotation when assigning `result` to a variable.
The reason is these variables are not used, so the compiler can't infer their types.
In this case, we have to add types manually.

### Join
**Type signature:** `M M a -> M a` where `M` is a monad type.

Join lets us flatten nested monads.
```fsharp
let nestedList = [[1]]
let jl  = nestedList |> List.concat
let jl' = nestedList |> join

let nestedSeq = seq [seq [1]]
let sl  = nestedSeq |> Seq.concat
let sl' = nestedSeq |> join

let nestedArray = [|[|1|]|]
let jar  = nestedArray |> Array.concat
let jar' = nestedArray |> join

let nestedAsync = 1 |> async.Return |> async.Return
let ja  = nestedAsync |> fun x -> async.Bind (x,id)
let ja' = nestedAsync |> join

let nestedOption = Some (Some 1)
let jo  = nestedOption |> Option.flatten
let jo' = nestedOption |> join

let nestedResult = Ok (Ok 1)
let jr : Result<int,int> = 
    nestedResult
    |> function 
        | Ok (Ok v) -> Ok v 
        | Ok (Error e) 
        | Error e -> Error e  
          
let jr' : Result<int,int> = nestedResult |> join
```

We can see, that F#+ becomes really handy. Especially, when dealing with `Option` and`Result` type.
These types are often handled using [Computation Expressions](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions).
Dealing with them using plain F# core functions is pretty awkward.

### Map
**Type signature:** `(a -> b) -> M a -> M b` where `M` is a monad type

Monads are also functors, so we can map over them.

Let's define a simple function
```fsharp
let myFunction x = x + 1 
```
and see how we can apply it to our example types.
```fsharp
let ml   = [1] |> List.map myFunction
let ml'  = [1] |> map myFunction
let ml'' = [1] |>> myFunction

let ms   = seq [1] |> Seq.map myFunction
let ms'  = seq [1] |> map myFunction
let ms'' = seq [1] |>> myFunction

let mar   = [|1|] |> Array.map myFunction
let mar'  = [|1|] |> map myFunction
let mar'' = [|1|] |>> myFunction

let ma   = async { return 1 } |> Async.map myFunction
let ma'  = async { return 1 } |> map myFunction
let ma'' = async { return 1 } |>> myFunction

let mo   = Some 1 |> Option.map myFunction
let mo'  = Some 1 |> map myFunction
let mo'' = Some 1 |>> myFunction

let mr  : Result<int,int> = Ok 1 |> Result.map myFunction
let mr' : Result<int,int> = Ok 1 |> map myFunction
let mr'': Result<int,int> = Ok 1 |>> myFunction
```

We map in F#+ using either generic `map` function, or simply with an operator `|>>`.

### Bind 
**Type signature:** `(a -> M b) -> M a -> M b` where `M` is a monad type

Bind lets us apply monadic value to a function returning monad.
We create a monadic version of `myFunction` for each context by composing it with previously described `result` function.

```fsharp
let blf  = myFunction >> result
let bl   = [1] |> List.collect blf 
let bl'  = [1] |> bind blf 
let bl'' = [1] >>= blf 

let barf  = myFunction >> result 
let bar   = [|1|] |> Array.collect barf
let bar'  = [|1|] |> bind barf
let bar'' = [|1|] >>= barf

let bsf  = myFunction >> result 
let bs   = [|1|] |> Seq.collect bsf
let bs'  = [|1|] |> bind bsf
let bs'' = [|1|] >>= bsf

let baf  = myFunction >> result 
let ba   = async { return 1 } |> fun x -> async.Bind (x, baf)
let ba'  = async { return 1 } |> bind baf
let ba'' = async { return 1 } >>= baf

let bof  = myFunction >> result 
let bo   = Some 1 |> Option.bind bof
let bo'  = Some 1 |> bind bof
let bo'' = Some 1 >>= bof

let brf = myFunction >> result 
let br   : Result<int,int> = Ok 1 |> Result.bind brf
let br'  : Result<int,int> = Ok 1 |> bind brf
let br'' : Result<int,int> = Ok 1 >>= brf
```

Similiarly to `map`, we use bind in F#+ using either generic `bind` function, or with an operator `>>=`.

#### Bind composition
Function `bind` can be defined in terms of `map` and `join`.
We can define our own function
```fsharp
let myBind f = map f >> join 
```
and then use it similarly to previously shown `bind`
```fsharp
let myFun x = [x + 1] 
let bc   = [1] >>= myFun 
let bc' = [1] |> myBind myFun
```

## Example code
```fsharp
#r "nuget: FSharpPlus"
open FSharpPlus

// Unit/Return
let l   = [1]
let l'  = List.singleton 1
let l'' = result 1 : int list 

let s   = seq [1]
let s'  = Seq.singleton 1
let s'' = result 1 : int seq 

let ar   = [|1|]
let ar'  = Array.singleton 1
let ar'' = result 1 : int []

let a   = async { return 1 }
let a'  = async.Return 1
let a'' = result 1 : Async<int>

let o   = Some 1
let o'  = None
let o'' = result 1 : int option

let r   = Ok 1
let r'  = Error "Oops!"
let r'' = result 1 : Result<int,string>

//  Join
let nestedList = [[1]]
let jl  = nestedList |> List.concat
let jl' = nestedList |> join

let nestedSeq = seq [seq [1]]
let sl  = nestedSeq |> Seq.concat
let sl' = nestedSeq |> join

let nestedArray = [|[|1|]|]
let jar  = nestedArray |> Array.concat
let jar' = nestedArray |> join

let nestedAsync = 1 |> async.Return |> async.Return
let ja  = nestedAsync |> fun x -> async.Bind (x,id)
let ja' = nestedAsync |> join

let nestedOption = Some (Some 1)
let jo  = nestedOption |> Option.flatten
let jo' = nestedOption |> join

let nestedResult = Ok (Ok 1)
let jr : Result<int,int> = 
    nestedResult
    |> function 
        | Ok (Ok v) -> Ok v 
        | Ok (Error e) 
        | Error e -> Error e  
          
let jr' : Result<int,int> = nestedResult |> join 

// Map
let myFunction x = x + 1 

let ml   = [1] |> List.map myFunction
let ml'  = [1] |> map myFunction
let ml'' = [1] |>> myFunction

let ms   = seq [1] |> Seq.map myFunction
let ms'  = seq [1] |> map myFunction
let ms'' = seq [1] |>> myFunction

let mar   = [|1|] |> Array.map myFunction
let mar'  = [|1|] |> map myFunction
let mar'' = [|1|] |>> myFunction

let ma   = async { return 1 } |> Async.map myFunction
let ma'  = async { return 1 } |> map myFunction
let ma'' = async { return 1 } |>> myFunction

let mo   = Some 1 |> Option.map myFunction
let mo'  = Some 1 |> map myFunction
let mo'' = Some 1 |>> myFunction

let mr  : Result<int,int> = Ok 1 |> Result.map myFunction
let mr' : Result<int,int> = Ok 1 |> map myFunction
let mr'': Result<int,int> = Ok 1 |>> myFunction

// Bind
let blf = myFunction >> result
let bl   = [1] |> List.collect blf 
let bl'  = [1] |> bind blf 
let bl'' = [1] >>= blf 

let barf  = myFunction >> result 
let bar   = [|1|] |> Array.collect barf
let bar'  = [|1|] |> bind barf
let bar'' = [|1|] >>= barf

let bsf  = myFunction >> result 
let bs   = [|1|] |> Seq.collect bsf
let bs'  = [|1|] |> bind bsf
let bs'' = [|1|] >>= bsf

let baf  = myFunction >> result 
let ba   = async { return 1 } |> fun x -> async.Bind (x, baf)
let ba'  = async { return 1 } |> bind baf
let ba'' = async { return 1 } >>= baf

let bof  = myFunction >> result 
let bo   = Some 1 |> Option.bind bof
let bo'  = Some 1 |> bind bof
let bo'' = Some 1 >>= bof

let brf = myFunction >> result 
let br   : Result<int,int> = Ok 1 |> Result.bind brf
let br'  : Result<int,int> = Ok 1 |> bind brf
let br'' : Result<int,int> = Ok 1 >>= brf

// Bind composition
let myBind f = map f >> join 

let myFun x = [x + 1] 
let bc   = [1] >>= myFun 
let bc' = [1] |> myBind myFun
```
