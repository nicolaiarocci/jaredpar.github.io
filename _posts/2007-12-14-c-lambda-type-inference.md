---
layout: post
---
One of the limitations of C# type inference is that you cannot use it to infer the type of a lambda expression.  For example, the following code will not compile

``` csharp
var f = () => 4;
```

Normally this is not too much of an issue because you can just explicitly declare the type of the lambda expression.

``` csharp
Func<int> f = () => 4;
```

However, this can be annoying at times.  Once you start defining complex lambda expressions the Func/Action declaration can be quite convoluted.  Even worse, if your lambda returns an anonymous type, there is no way to declare a Func<> with the anonymous type parameter because you cannot describe it's shape.

``` csharp
Func<'> f = () => new { Name = "foo" };
```

This is fixable though by using type inference.  The method is very similar to [other anonymous type type tricks]({% post_url 2007-10-01-casting-to-an-anonymous-type %}).  While lambda type inference is not supported for variable declaration it is supported for parameters.  C# supports return type inference so that can be used to type the variable.

``` csharp
static Func<T> Lambda<T>(Func<T> del)
{
    return del;
}

static void Main(string[] args)
{
    var f = Lambda(() => new { Name = "foo" });
}
```

