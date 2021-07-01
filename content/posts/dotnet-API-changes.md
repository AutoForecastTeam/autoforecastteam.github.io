+++
title =  "Dotnet API changes"
description =  "Dotnet API changes"
date =  2021-02-10T17:34:17+01:00
template = "blog/page.html"
draft =  false

[taxonomies]
authors =  ["Paweł Sierzputowski"]
#tags =  ["dotnet","dotnet standard"]
#categories =  ["dotnet"]
+++

<!--more-->

# Dotnet API changes

## Upgrade to .NET Standard 2.1
In our solution there are a few .NET Core 3.1 applications and a bunch of libraries shared between them targeting .NET Standard in different versions. There is no point in keeping them in such state so I decided to upgrade all the libraries to highest possible version which is .NET Standard 2.1. After doing that compiler showed me some errors like this:

```
  error [CS0121] The call is ambiguous between the following methods or properties:
  'System.Collections.Generic.CollectionExtensions.GetValueOrDefault<TKey, TValue>(
      System.Collections.Generic.IReadOnlyDictionary<TKey, TValue>, TKey)'
  and 'HelperClasses.BasicExtensions.GetValueOrDefault<K, V>(
      System.Collections.Generic.IDictionary<K, V>, K)'
```

It turned out that a new [CollectionExtensions](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.collectionextensions) appeared in the API since dotnet Standard 2.1. It seems that our custom extension method can be removed

```cs
public static class BasicExtensions
{
    public static V GetValueOrDefault<K, V>(this IDictionary<K, V> dictionary, K key)
    {
        return dictionary.TryGetValue(key, out var dictVal) ? dictVal : default(V);
    }
    /// ...
}
```

But in some cases we call `GetValueOrDefault` on `IDictionary`. [IDictionary](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.idictionary-2) and [IReadOnlyDictionary](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.ireadonlydictionary-2) are separate interfaces, none of them derive from the other. However [Dictionary](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.dictionary-2) implements both of them.

## Method dispatch

To eliminate this issue we can cast Dictionary objects to one of interfaces `IDictionary` or `IReadOnlyDictionary`, but it is not convenient solution since there are multiple usages of `Dictionary` where we call `GetValueOrDefault`. To address this problem in a more systematic way we can use the fact, that in C# methods on more derived class is applicable and add another implementation for `Dictionary`:

```cs
public static V GetValueOrDefault<K, V>(this Dictionary<K, V> dictionary, K key)
{
    return ((IReadOnlyDictionary<K, V>) dictionary).GetValueOrDefault(key);
}
```

After that, when we call `Dictionary.GetValueOrDefault` this implementation above is used, because it has higher priority than methods implemented for interfaces implemented by `Dictionary`. And also everything works correctly for cases when we call `GetValueOrDefault` on `IDictionary` and `ReadOnlyDictionary`, because they already have their own extensions.