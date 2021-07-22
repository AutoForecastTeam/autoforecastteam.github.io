+++
title =  "C# expressions that try to be monadic"
description =  ""
date =  2021-07-01
template = "blog/page.html"
draft =  false

[taxonomies]
authors = ["Robert Marciniak"]
#tags =  ["monad", "linq", "async"]
#categories =  []
+++

# Why we can't have nice things, c# query expression edition

The closest thing to f#'s computational expression is either combination of `async` `await` keywords in contexts that
do not always translate to asynchronous context or query expression which does not always translate to querying something.

That's because C# is a corporate enterprise language which tries to smuggle Nice Things through, but in a different,
more corporate-enterprise-like form - not too abstract, sprinkled with "query SQL from code!"-, "Improve responsiveness without big refactors!"-, or similar seasoning.

## About query expression

[Query expression](https://docs.microsoft.com/en-us/dotnet/csharp/linq/query-expression-basics) is a syntactic sugar which allows compiler to translate

this
```cs
private Gen<TimeSeriesValueColumn> TimeSeriesValueColumnGen() =>
    from dto in TimeSeriesDtoGen()
    from time in Arb.Generate<DateTime>()
    select TimeSeriesValueColumn.CreateColumns(dto).Single();
```

into this

```cs
private Gen<TimeSeriesValueColumn> TimeSeriesValueColumnGen() =>
    TimeSeriesDtoGen()
        .SelectMany(dto => Arb.Generate<DateTime>(),
            (dto, time) => TimeSeriesValueColumn.CreateColumns(dto).Single());
```

and can be added to any monad-like type simply by adding few functions
with particular names and signatures, e.g. this short snippet allowed me to create valid code for 'case 2'

```cs
public static class TaskExtensions
{
    public static async Task<B> Bind<A, B>(this Task<A> @this, Func<A, Task<B>> fn)
        => await fn(await @this);

    public static async Task<B> Map<A, B>(this Task<A> @this, Func<A, B> fn)
        => await Task.FromResult(fn(await @this));

    public static Task<C> SelectMany<A, B, C>(this Task<A> @this, Func<A, Task<B>> fn, Func<A, B, C> select)
        => @this.Bind(a => fn(a).Map(b => select(a, b)));
}
```

## 'what's the underlying type?'

### Case 1

```cs
from index in Enumerable.Range(1, 5)
from date in Enumerable.Repeat(DateTime.Today, 5)
select (index, date) into pair
select pair.date.AddDays(pair.index);
```

```fs
seq {
    let! index = Enumerable.Range(1, 5)
    let! date = Enumerable.Repeat(DateTime.Today, 5)
    let pair = (index, date)
    yield pair |> (fun (i, d) -> d.AddDays(i |> float))
}
```

### Case 2

```cs
from index in GetIndexAsync()
from date in Task.FromResult(DateTime.Today)
select (index, date) into pair
select pair.date.AddDays(pair.index);
```

```fs
monad {
    let! index = GetIndexAsync()
    let! date = Task.FromResult(DateTime.Today)
    let pair = (index, date)
    yield pair |> (fun (i, d) -> d.AddDays(i |> float))
}
```

### Case 3
```cs
from index in Option.Some(5)
from date in Option.None<DateTime>()
select (index, date) into pair
select pair.date.AddDays(pair.index);
```

```cs
async Option<DateTime> Fun() {
    var index = await Option.Some(5);
    var date = await Option.None<DateTime>();
    var pair = (index, date);
    return pair.date.AddDays(pair.index)
}
```

```fs
monad {
    let! index = Some 5
    let! date = None
    let pair = (index, date)
    yield pair |> (fun (i, (d:DateTime)) -> d.AddDays(i |> float))
}
```
`monad` is from [F#+](http://fsprojects.github.io/FSharpPlus/), though implementing `option` Computation Expression manually is pretty straight-forward

## ...converging into the same topic, never reaching it

These c# constructs do exactly the same thing, without any major or unsafe hacks. The main barrier from using these in a monad-like manner is their naming. Despite being perfectly viable in multiple scenarios their naming suggests using them in LINQ-to-* and asynchronous code respectively.

- in c# there's no way to create a generic `Monad<T<E>>` because there's no higher-kinded polymorphism
- poor type inference, no partial inference, partly due to complicated inheritance laws
- query expression has no ability to switch from one monad type to another using concise language syntax, while functional languages have transformers and one can nest computational expressions.
