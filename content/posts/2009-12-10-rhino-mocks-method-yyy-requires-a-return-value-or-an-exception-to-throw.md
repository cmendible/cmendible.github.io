---
author: Carlos Mendible
categories:
- dotNet
date: "2009-12-10T12:15:00Z"
description: Rhino Mocks Method &#8216;YYY' requires a return value or an exception
  to throw.
# tags: ["RhinoMocks", "UnitTests"]
title: Rhino Mocks Method &#8216;YYY' requires a return value or an exception to throw.
---
Yesterday I found myself stucked with a strange exception while programming a unit test using Rhino Mocks.

Supose you have an interface like the following one.

``` csharp
public interface ITest
{
    object Freak();
}
```

You can try a mock like the following:

``` csharp
var test = MockRepository.GenerateMock();

test.Expect(t =&gt; t.Freak()).IgnoreArguments()
    .WhenCalled(x =&gt; x.ReturnValue = "THIS IS A RETURN VALUE")
    .Repeat.Any();
```

Calling **test.Freak();** will result in a return value of **&#8220;THIS IS A RETURN VALUE"**

Now change the interface to be:

``` csharp
public interface ITest
{
    string Freak();
}
```

Calling **test.Freak();** results in the following exception: **_Method &#8216;ITest.Freak();' requires a return value or an exception to throw._**

The solution is easy just change the Expect call to look like this one:

``` csharp
test.Expect(t =>; t.Freak()).IgnoreArguments()
    .WhenCalled(x =>; x.ReturnValue = "THIS IS A RETURN VALUE")
    .Return(string.Empty) // THIS IS NEEDED SO RHINO MOCKS KNOWS THE TYPE RETURNED FROM THE METHOD CALL.
    .Repeat.Any();
```

The fake return value is ignored in favor of the given by the WhenCalled delegate!!!

Hope it helps.