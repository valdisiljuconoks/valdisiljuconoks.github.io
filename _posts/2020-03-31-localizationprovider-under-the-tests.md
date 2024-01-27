---
title: LocalizationProvider Under the Tests
author: valdis
date: 2020-03-31 18:00:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source, DeveloperTools]
tags: [add-on, optimizely, episerver, .net, c#, open source]
---

Recently I got a question from a [very good friend](https://twitter.com/LucGosso) of mine about how to properly unit test localization provider.


Tests and even unit ones have been one of the areas where we probably all have committed sins. And I'm no exception. Only recently I've [added an interface](https://github.com/valdisiljuconoks/LocalizationProvider/issues/165) in front of `LocalizationProvider` class to make it easier to work in unit tests.

Adding an interface is just part of the solution (and the easiest one). Writing unit test for the localization provider turned out not to be so easy.

## Unit Test with FakeItEasy

Challenge to write proper unit tests for the localization provider is due to fact that library heavily relies on [expressions](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/statements-expressions-operators/expressions) (aka lambdas).


### First Attempt

I would write this naive unit tests in hope that it will run green:

```csharp
public class ResourceClass
{
    public static string SomeProperty { get; } = "Some value of the property";
}

public class SomeServiceWithLocalization
{
    private readonly ILocalizationProvider _provider;

    public SomeServiceWithLocalization(ILocalizationProvider provider)
    {
        _provider = provider;
    }

    public string GetTranslation()
    {
        return _provider.GetString(() => ResourceClass.SomeProperty);
    }
}

[Fact]
public void TestInterfaceMock()
{
    var fake = A.Fake<ILocalizationProvider>();

    A.CallTo(() => fake.GetString(() => ResourceClass.SomeProperty)).Returns("Fake");

    var sut = new SomeServiceWithLocalization(fake);

    var result = sut.GetTranslation();

    Assert.Equal("Fake", result);
}
```

But if you run this - it will be <span style="color: red; font-weight: bold;">red</span>.

This is because FakeItEasy (and other mocking frameworks) uses argument comparer to understand whether specified arguments are the same that has been used to configure mock - and if so, specified behavior (in this case return value) should be executed.

Turns out `Expression` is not something that is easy to compare out of the box.
And expression you specified while setting up mock is not the same as one used in `SomeServiceWithLocalization`.

You can even compare them directly in unit test:

```csharp
[Fact]
public void CompareExpressions()
{
    Expression<Func<object>> exp = () => ResourceClass.SomeProperty;
    Expression<Func<object>> exp2 = () => ResourceClass.SomeProperty;

    Assert.Equal(exp, exp2);
}
```

![failed-unittest](/assets/img/2020/03/failed-unittest.png)

This still fails.

## Add Expression Comparer

What you need to do instead is to teach FakeItEasy how to do an expression comparison.
For this task you can use some ready made packages (like [Neleus.LambdaCompare](https://www.nuget.org/packages/Neleus.LambdaCompare/)).

Or you can search around internets and find maybe something suitable for you. I found quite comprehensive expression comparer from [Joseph](https://gist.github.com/jnm2/83b36ad497b4cb1cbcac). This does its job quite well.

Make sure that you copy this somewhere in your unit test project (or shared project that you can use across all your unit tests).

## Second Attempt

With having this comparer in our toolkit we can write second attempt to test localization provider interface.

```csharp
[Fact]
public void TestInterfaceMock()
{
    var fake = A.Fake<ILocalizationProvider>();
    Expression<Func<object>> expression = () => ResourceClass.SomeProperty;
    var comparer = new ExpressionComparer();

    A
        .CallTo(() => fake.GetString(A<Expression<Func<object>>>.That.Matches(e => comparer.Equals(expression, e))))
        .Returns("[SomeProperty] Value from fake");

    var sut = new SomeServiceWithLocalization(fake);

    var result = sut.GetTranslation();

    Assert.Equal("[SomeProperty] Value from fake", result);
}
```

Now this test is <span style="color: green; text-weight: bold;">green</span>.

We can even verify with second expression just to be sure that mock is working as expected.

```csharp
public class ResourceClass
{
    public static string SomeProperty { get; } = "Some value of the property";
    public static string SomeProperty2 { get; } = "Another translation";
}

public class SomeServiceWithLocalization
{
    private readonly ILocalizationProvider _provider;

    public SomeServiceWithLocalization(ILocalizationProvider provider)
    {
        _provider = provider;
    }

    public string GetTranslation()
    {
        return _provider.GetString(() => ResourceClass.SomeProperty);
    }

    public string GetTranslation2()
    {
        return _provider.GetString(() => ResourceClass.SomeProperty2);
    }
}

[Fact]
public void TestInterfaceMock()
{
    var fake = A.Fake<ILocalizationProvider>();
    Expression<Func<object>> expression = () => ResourceClass.SomeProperty;
    Expression<Func<object>> expression2 = () => ResourceClass.SomeProperty2;
    var comparer = new ExpressionComparer();

    A
        .CallTo(() => fake.GetString(A<Expression<Func<object>>>.That.Matches(e => comparer.Equals(expression, e))))
        .Returns("[SomeProperty] Value from fake");

    A
        .CallTo(() => fake.GetString(A<Expression<Func<object>>>.That.Matches(e => comparer.Equals(expression2, e))))
        .Returns("[SomeProperty] Value from fake 2");

    var sut = new SomeServiceWithLocalization(fake);

    var result = sut.GetTranslation();
    var result2 = sut.GetTranslation2();

    Assert.Equal("[SomeProperty] Value from fake", result);
    Assert.Equal("[SomeProperty] Value from fake 2", result2);
}
```

Happy unit testing your translations!

[*eof*]
