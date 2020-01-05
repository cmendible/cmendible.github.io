---
author: Carlos Mendible
categories:
- dotnet
date: "2009-10-19T15:07:00Z"
description: Issue in non Generic Execute Method in DbLinq.Data.Linq.Implementation.QueryProvider
tags: ["DbLinq", "SQLite"]
title: Issue in non Generic Execute Method in DbLinq.Data.Linq.Implementation.QueryProvider
---
We've been working with DbLinq for a while now, at [HexaSystems](http://hexasystems.com) and recently using it together with Dynamic Linq and SQLite I programed a call to the Count method of a Queryable object as follows:

``` csharp
public static int Count(this IQueryable source)
{
    if (source == null) throw new ArgumentNullException("source");

    return (int)source.Provider.Execute(
        Expression.Call(
            typeof(Queryable), 
            "Count",
            new Type[] { source.ElementType },
            source.Expression));
}
```

I noticed that when doing so against a DbLInq Table object the following method is called:

``` csharp
public object Execute(Expression expression)
{
    .... 
}
```

The problem is that the function calls the generic Execute method inside the same class with the generic argument of type object: "Execute(expression)" which causes an invalid cast exception in the case I'm talking about, and potentially any call to this method could fail when the expression evaluation creates a generic type. I've submitted a patch to DBLinq developers and it was offically accepted: [http://code.google.com/p/dblinq2007/source/detail?r=1224](http://code.google.com/p/dblinq2007/source/detail?r=1224)