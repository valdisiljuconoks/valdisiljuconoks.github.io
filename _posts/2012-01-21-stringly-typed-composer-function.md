---
title: “Stringly typed” Composer functions
author: valdis
date: 2012-01-21 23:25:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

When you need to create EPiServer Composer block that requires some properties we usually use Composer functions that basically are defined as page types. Ted has a really good blog post on step-by-step walkthrough how to create Composer functions using `PageTypeBuilder`.

Story today behind Composer functions is about so called “stringly-typed interface” approach :)

Somewhere up in your inheritance path you will have a base class called `Dropit.Extensions.Core.BaseContentFunction` which will give you possibility to access designed properties and its assigned value on the `Composer` page. Code for accessing property value usually looks like this:

```csharp
var count = (int)ContentFunctionData["MyFunctionItemCount"];
```

It would be cool to write something like this:

```csharp
var count = FunctionData(f => f.MyFunctionItemCount);
```

This is pretty simplified sample code. Anyway to follow more defensive programming practices we should after receiving value back we should check for null, if return value is not null, then we can cast it to `PropertyData` and check Value property to finally get access to property value.

```csharp
if (count != null)
{
    var pd = count as PropertyData;
    if (pd != null)
    {
        var finalValue = ((PropertyData)pd).Value;
    }
}
```

The issue here is around indexer accessor that we use to access property data.

It would be great to have approximately the same experience as accessing page properties when using `PageTypeBuilder`.

This can be easily implemented using lambda expressions.

We should implement some kind of base class to be used for providing lambda expression approach when accessing property data from composer block.

Class will be inheriting from `BaseContentFunction`.

```csharp
public class ComposerContentFunctionBase<T> : BaseContentFunction where T : ComposerFunctionBase
```

Class `ComposerFunctionBase` is base class mentioned in Ted’s blog which is required if building Composer functions with page type builder.

```csharp
public abstract class ComposerFunctionBase : TypedPageData
{
    /// Required by Composer - do not remove
    [PageTypeProperty(Type = typeof(ExtensionFunctionProperty), DefaultValueType = DefaultValueType.None, DefaultValue = "",
        SortOrder = 101, DisplayInEditMode = false, Searchable = false, UniqueValuePerLanguage = true, Tab = typeof(ComposerTab))]
    public virtual string ExtensionContentFunctionProperty { get; set; }
    /// Required by Composer - do not remove
    [PageTypeProperty(UniqueValuePerLanguage = false, DefaultValueType = DefaultValueType.None, DefaultValue = "false", SortOrder = 122,
        Searchable = false, DisplayInEditMode = false, Tab = typeof(ComposerTab))]
    public virtual bool NeverUsedProperty { get; set; }
}
```

So after implementing base class we are able to derive some child and make our Composer user control.

```csharp
public partial class MyComposerControl : ComposerContentFunctionBase<MyComposerFunction>
```

Next thing is to write implementation of `FunctionData` method in `ComposerContentFunctionBase` class to accept lambda expression for accessing composer content function data.

```csharp
public R FunctionData<R>(Expression<Func<T, R>> expression)
{
    if (expression == null)
    {
        throw new ArgumentNullException("expression");
    }

    object result = null;
    var expressionBody = expression.Body;
    if (expressionBody.NodeType == ExpressionType.MemberAccess)
    {
        var propertyName = ((MemberExpression)expressionBody).Member.Name;
        result = ContentFunctionData[propertyName];
    }

    // if using PageType builders actual expression type to access PageType property will be substituted by Castle proxy
    if (expressionBody.NodeType == ExpressionType.Convert)
    {
        if (((UnaryExpression)expressionBody).Operand.NodeType == ExpressionType.MemberAccess)
        {
            var propertyName = ((MemberExpression)((UnaryExpression)expressionBody).Operand).Member.Name;
            result = ContentFunctionData[propertyName];
        }
    }

    if (result == null)
    {
        return default(R);
    }

    if ((result as PropertyData) == null)
    {
        return default(R);
    }

    if (((PropertyData)result).Value == null)
    {
        return default(R);
    }

    return (R)((PropertyData)result).Value;
}
```

So what we are doing here? We are waiting for incoming parameter to be `Expression<Func<T, R>>`. Let’s split up into smaller pieces:

1. `T` is the `Composer` function data type we typed when implementing user control. So this will be incoming parameter of lambda expression.

2. `R` is returning type from the lambda expression or data type of which function data property is defined.



So in this expression: `(f => f.MyFunctionItemCount)`

`f –> T` (this is of type `MyComposerFunction`)

`MyFunctionItemCount –> R` (function property is defined as `int`).



3. Then we are investigating given expression and looking for member access node. **NB!** This implementation is pretty simplified and does not handle more complex expressions that result in member access somehow.

4. Expression investigation has 2 if statements. Second (when node type is of type `Convert`) is required because `Castle` proxies are behind the scene when using `PageTypeBuilder` and we need to dig deeper into expression to find member access expression. **NB!** This implementation handles only 1 level within expression tree. Probably not the best solution anyway.

5. Then we are investigating collected result and returning the return value.



Now we can write expressins like this in out `Composer` function user control and get proper type casting also.

```csharp
int count = FunctionData(f => f.MyFunctionItemCount);
```

If you would like to get precise info whether editor supplied value for the function property or not we can always use Nullable types to get new method to work properly when returning `default(R)`.

Pretty simple solution but most probably not 100% bulletproof.

Hope this helps!

