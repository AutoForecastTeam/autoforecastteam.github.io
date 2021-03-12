---
title: "Parsing 101"
description: "Simple parsing example with FParsec."
draft: false
author: "Rafal Gwozdzinski"
date: 2021-03-12T00:08:00+00:00
tags: []
categories: ["FSharp"]
---
<!--more-->

# Parsing 101
In this post I will try to show a simple example of parsing using [FParsec](https://www.quanttec.com/fparsec/) library.
It uses a Combinatory Parsing approach, which is pervasive across Functional Programming domain.

## Task #1 - Parse simple boolean logic

Let's define a type that represents a boolean expressions.
It is a recursive type for which we can distnguish two kinds of expressions:
- Primitive - `True`, `False`
- Nested - `And`, `Or`, `Not`

```fsharp
type BoolExpr =
    | True
    | False
    | And of BoolExpr * BoolExpr
    | Or of BoolExpr * BoolExpr
    | Not of BoolExpr
```

First, we define primitive parsers.
```fsharp
let pTrue : Parser<BoolExpr,unit> = 
    pstring "TRUE" |>> (fun _ -> True) // See how `|>>` is used to map Parser monad to type. 
                                       // It's the same convention as in FSharpPlus
let pFalse : Parser<BoolExpr,unit> = 
    pstring "FALSE" >>% False // But for map with value dropping, we can just use dedicated operator `>>%`
```

Functional approach: Use composition to build robust abstractions.
We combine two primitive parsers to create a single parser for primitives.
```fsharp
let pBoolLiteral : Parser<BoolExpr,unit> = pTrue <|> pFalse
```

Now, let's create a simple runner:
```fsharp
let parse parser str =
    match run parser str with
    | Success(result, _, _)   -> result
    | Failure(errorMsg, _, _) -> failwith errorMsg
```

We can see if our parsers work correct:

```fsharp
// Parse concrete primitives
let trueVal = parse pTrue "TRUE"
let falseVal = parse pFalse "FALSE"
let trueFail = parse pTrue "NOT_A_TRUE"
let falseFail = parse pFalse "NOT_A_FALSE" 

// Parse primitive expressions
let trueVal' = parse pBoolLiteral "TRUE"
let falseVal' = parse pBoolLiteral "FALSE"
let trueFail' = parse pBoolLiteral "NOT_A_TRUE"
let falseFail' = parse pBoolLiteral "NOT_A_FALSE" 
```

For our implementation to work correctly, we need to be able to parse nested expressions.
Common way to structurize expressions is to use parentheses, which our parser needs to support.

**Example:** `(NOT TRUE) AND (TRUE OR FALSE)` is `And` expression, which is combined of two expressions: `NOT TRUE` and `TRUE OR FALSE` 

We declare dummy parser, that will be filled with proper implementation in future.
**Warning:** Using this parser without binding proper implementation to it, will result in runtime Exception.
```fsharp
let exprParser, exprParserRef = createParserForwardedToRef<BoolExpr,unit>()
```

We use it in our implementation of parenthesized expression parser:
```fsharp
let pParenExpr = between (pchar '(') (pchar ')') exprParser
```

Now, we can create a term parser. 
One thing to consider is potential spaces in our input (e.g. `NOT  (  TRUE )`).
We need to parse them, but they are not meaningful, so we drop them using operators `>>.` and `.>>`.
```fsharp
let pTerm = 
    spaces >>. 
    pBoolLiteral <|> pParenExpr
    .>> spaces
```

Now we want to parse operators.
We could write our own implementation, but it's simpler and more efficient to use a dedicated tool:
```fsharp
let opp = OperatorPrecedenceParser<_,_,_>()
```

Let's define some helpers.
```fsharp
let isSymbolicOperatorChar = notFollowedBy letter >>. spaces
let resultFun op = fun x y -> op (x, y)
```

Now we can add infix operators OR and AND. 
AND has a greater precedence, so we give it 20.
There is a convention of assigning (10, 20, 30) instead of (1, 2, 3) as precedence values.
The reason is, that it's easier to add some operators inbetween existing ones later.
```fsharp
let infixOperator (opp: OperatorPrecedenceParser<_,_,_>) op prec assoc f =
    opp.AddOperator(InfixOperator (op, isSymbolicOperatorChar, prec, assoc, resultFun f))
infixOperator opp "OR" 10 Associativity.Left Or
infixOperator opp "AND" 20 Associativity.Left And
```

Now we add prefix operator NOT. It has greater precedence than infixes, so it gets 30. 
```fsharp
opp.AddOperator(PrefixOperator ("NOT", isSymbolicOperatorChar, 30, true, Not))
```

We assign our term parser.
```fsharp
opp.TermParser <- pTerm
```

And we assign our infix parser as our proper parser implementation instead of previously defined dummy.
```fsharp
exprParserRef := opp.ExpressionParser
```

Let's see if it works.
```fsharp
let pExpr = parse exprParser
pExpr "FALSE AND TRUE"
pExpr "NOT FALSE OR NOT TRUE"
pExpr "NOT ( FALSE AND NOT TRUE )"
```

## Task #2 - Evaluate parsed expressions
Now that we made all the parsing, evaluation is straightforward:
```fsharp
let rec eval = 
    function
    | True -> true
    | False -> false 
    | And (e1, e2) -> (eval e1) && (eval e2) 
    | Or (e1, e2) -> (eval e1) || (eval e2)
    | Not e -> not (eval e)
```

Finally, let's parse and evaluate our expressions. 
```fsharp
let parseAndEval = pExpr >> eval
parseAndEval " FALSE   AND    TRUE   "
parseAndEval "NOT FALSE OR NOT TRUE"
parseAndEval "NOT ( FALSE AND NOT TRUE )"
parseAndEval "NOT TRUE AND NOT FALSE OR TRUE AND NOT FALSE"
```

## Resources
1. Super useful tutorial with explained FParsec internals by Scott Wlaschin.
   https://fsharpforfunandprofit.com/posts/understanding-parser-combinators/
2. FParsec docs. Tutorial and user's guide are especially valuable for beginners.
   https://www.quanttec.com/fparsec/tutorial.html
3. Comparison of different parsing approaches.
   https://www.quanttec.com/fparsec/about/fparsec-vs-alternatives.html

## Code
```fsharp
#r "nuget: FParsec"

open FParsec

// Task #1 - Parse simple boolean logic

type BoolExpr =
    | True   
    | False
    | And of BoolExpr * BoolExpr
    | Or of BoolExpr * BoolExpr
    | Not of BoolExpr

let pTrue : Parser<BoolExpr,unit> = 
    pstring "TRUE" |>> (fun _ -> True) 
let pFalse : Parser<BoolExpr,unit> = 
    pstring "FALSE" >>% False

let pBoolLiteral : Parser<BoolExpr,unit> = pTrue <|> pFalse

let parse parser str =  
    match run parser str with
    | Success(result, _, _)   -> result
    | Failure(errorMsg, _, _) -> failwith errorMsg

let trueVal = parse pTrue "TRUE"
let falseVal = parse pFalse "FALSE"
// let trueFail = parse pTrue "NOT_A_TRUE"
// let falseFail = parse pFalse "NOT_A_FALSE" 

let trueVal' = parse pBoolLiteral "TRUE"
let falseVal' = parse pBoolLiteral "FALSE"
// let trueFail' = parse pBoolLiteral "NOT_A_TRUE"
// let falseFail' = parse pBoolLiteral "NOT_A_FALSE" 

let exprParser, exprParserRef = createParserForwardedToRef<BoolExpr,unit>()

let pParenExpr = between (pchar '(') (pchar ')') exprParser

let pTerm = 
    spaces >>. 
    pBoolLiteral <|> pParenExpr
    .>> spaces

let opp = OperatorPrecedenceParser<_,_,_>()

let isSymbolicOperatorChar = notFollowedBy letter >>. spaces
let resultFun op = fun x y -> op (x, y)

let infixOperator (opp: OperatorPrecedenceParser<_,_,_>) op prec assoc f =
    opp.AddOperator(InfixOperator (op, isSymbolicOperatorChar, prec, assoc, resultFun f))
infixOperator opp "OR" 10 Associativity.Left Or
infixOperator opp "AND" 20 Associativity.Left And

opp.AddOperator(PrefixOperator ("NOT", isSymbolicOperatorChar, 30, true, Not))

opp.TermParser <- pTerm

exprParserRef := opp.ExpressionParser

let pExpr = parse exprParser
pExpr "FALSE AND TRUE"
pExpr "NOT FALSE OR NOT TRUE"
pExpr "NOT ( FALSE AND NOT TRUE )"


// Task #2 - Evaluate parsed expressions

let rec eval = 
    function
    | True -> true
    | False -> false 
    | And (e1, e2) -> (eval e1) && (eval e2) 
    | Or (e1, e2) -> (eval e1) || (eval e2)
    | Not e -> not (eval e)

let parseAndEval = pExpr >> eval
parseAndEval " FALSE   AND    TRUE   "
parseAndEval "NOT FALSE OR NOT TRUE"
parseAndEval "NOT ( FALSE AND NOT TRUE )"
parseAndEval "NOT TRUE AND NOT FALSE OR TRUE AND NOT FALSE"
```
