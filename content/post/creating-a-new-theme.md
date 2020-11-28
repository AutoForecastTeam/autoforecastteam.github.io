+++
title = "FSharp post"
description = "FSharp description blablabla"
date = 2020-09-11T02:13:50Z
author = "Rafał Gwoździński"
+++

## Introduction

// TODO: Ogarnąć syntax highlight

https://gohugo.io/content-management/syntax-highlighting/
```FSharp
let l  = [1]
let l' = List.singleton 1

let s  = seq [1]
let s' = Seq.singleton 1

let ar  = [|1|]
let ar' = Array.singleton 1

let a  = async { return 1 }
let a' = 1 |> async.Return

let o  = Some 1
let o' = None
```

```
let l  = [1]
let l' = List.singleton 1

let s  = seq [1]
let s' = Seq.singleton 1

let ar  = [|1|]
let ar' = Array.singleton 1

let a  = async { return 1 }
let a' = 1 |> async.Return

let o  = Some 1
let o' = None
```