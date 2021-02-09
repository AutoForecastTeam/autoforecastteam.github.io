---
title: "Exception handling in F#"
description: "Functional-first approach to .NET exceptions"
draft: false
# author: "Rafał Gwoździński"
date: 2021-02-09T11:03:32+01:00
tags: []
categories: ["FSharp"]
---
<!--more-->
# Exception Handling in F#
## Try-catch

Simplest way to handle Exception is to use a try-with pattern. It is similar concept to a [try-catch pattern in C#](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/try-catch).
According to an [F# documentation](https://docs.microsoft.com/pl-pl/dotnet/fsharp/language-reference/exception-handling/the-try-with-expression) a structure of try-catch blok is

```fsharp
try
    expression1
with
| pattern1 -> expression2
| pattern2 -> expression3
...
```

The shortest way we can write it, is to give one catch-all pattern for all exception.
Then we can even drop the pipe, and match directly after `with` clause:
```fsharp
let operationThatFails () = failwith "Error"
let x =
  try
    operationThatFails ()
  with e -> sprintf "Catched exception %A" e
```

In above case the exception is handled correctly. Variable x evaluates to `"Catched exception System.Exception: Error"` string.

## Problem with HOFs and partial application
There is one problem, that have caught me unexpectedly. 

I was working on a function that did a lookup on CsvRow from [FSharp.Data](http://fsprojects.github.io/FSharp.Data/) package.
The first version was:
```fsharp
let tryGetValue (row: CsvRow) (columnName: string) =
   try
       row.[columnName]
   with
   | :? KeyNotFoundException -> failwithf "Missing column: %s" columnName
```

I used it to catch exceptions when there is a missing column name in CSV, and then rethrow 
with better message. Default exception message was too cryptic for my use case.

After couple of minutes, I thought that it would be a good idea to refactor this into a
higher-order function that will be partially applied.

```fsharp
let tryGetValue (columnName: string) =
   try
       fun (x: CsvRow) -> x.[columnName]
   with
   | :? KeyNotFoundException -> failwithf "Missing column: %s" columnName
```

At first glance it looked OK. The only problem is, that this solution did not work.

In this case, exceptions are catched only for the anonymous function creation part. 
Application of my newly created function happens in another context, which may or may not have 
it's own exception handling.

Similar thing happens with partial application:
```fsharp
let operationThatFails (_: string) (_: string) : string = failwith "Error"

let x =
  try
    operationThatFails "str1" "str2"
  with 
  | e -> failwithf "Catched exception %A" e

let partiallyApplied : (string -> string) =
  try
    operationThatFails "str1" 
  with 
  | e -> failwithf "Catched exception %A" e

let y = partiallyApplied "str2"
```

Variable x evaluates to string `"Catched exception [...]"`. 

Evaluation of y fails with `System.Exception: Error`

## Alternative approach
An interesting (and more functional) way to handle exceptions is to use an
[FSharpPlus](https://github.com/fsprojects/FSharpPlus) library. 
There is a simple (but very helpful) function `protect` for `Result` type:

```fsharp
let protect (f: 'T->'U) (x: 'T) : Result<'U,exn> =
        try
            Ok (f x)
        with e -> Error e
```

Each time we deal with a function/method that throws exceptions, we can wrap it 
and apply in safe `Result` context:
```fsharp
let iWillThrow () : string = failwith "Error"

Result.protect iWillThrow  // Wrap
|> fun x -> x ()           // Apply
|> function                // Handle Result
   | Ok v    -> sprintf "Value was: %s" v
   | Error e -> sprintf "There was an error: %A" e
```


If we don't care about thrown exception, we can just use `Option.protect`, which will
return `None` for all failed cases.

There is a caveat though, that `protect` accepts only functions that take a single parameter.
However, this constraint may actually be benefitial, because it saves us from making the same mistake
that I did.

Partially applied `tryGetValue` function would then be:
```fsharp
let tryGetValue (columnName: string) =
   fun (x: CsvRow) -> x.[columnName]
   |> Option.protect
   |>> Option.defaultWith (fun () -> failwithf "Missing column: %s" columnName) 
```

where `|>>` is just an FSharpPlus operator for `Option.map` function.

## Disclaimer
In all examples above, `tryGetValue` remains a partial function. It will fail for all `columnName` values
that are not contained in CsvRow. It is written that way, because in my example I treat missing column name
as an unrecoverable error. I apply [Fail-fast](https://en.wikipedia.org/wiki/Fail-fast) principle, 
which means that I want to stop further processing immediately.
