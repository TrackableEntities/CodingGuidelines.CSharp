# C# Style Guide

*These are based on coding guidelines found on the [Microsoft ASNET](https://github.com/aspnet/Home/wiki/Engineering-guidelines#coding-guidelines) repository.*

### Summary:
- [General guidelines](#general-guidelines)
- [Usage of the var keyword](#usage-of-the-var-keyword)
- [Use C# type keywords in favor of .NET type names](#use-c-type-keywords-in-favor-of-net-type-names)
- [When to use internals vs public and when to use InternalsVisibleTo](#when-to-use-internals-vs-public-and-when-to-use-internalsvisibleto)
- [Argument null checking](#argument-null-checking)
- [Async method patterns](#async-method-patterns)
- [Extension method patterns](#extension-method-patterns)
- [Doc comments](#doc-comments)
- [Assertions](#assertions)
- [Unit tests and functional tests](#unit-tests-and-functional-tests)
- [Use only complete words or common-standard abbreviations in public APIs](#use-only-complete-words-or-common-standard-abbreviations-in-public-apis)

#### General guidelines

The most general guideline is that we use all the VS default settings in terms of code formatting, except that we put `System` namespaces before other namespaces (this used to be the default in VS, but it changed in a more recent version of VS).

1. Use four spaces of indentation (no tabs)
2. Use `_camelCase` for private fields
3. Avoid `this.` unless absolutely necessary
4. Always specify member visiblity, even if it's the default (i.e. `private string _foo;` not `string _foo;`)

#### Usage of the var keyword

The `var` keyword can to be used as much as the compiler will allow, but it must be used when the type is not evident.
For example, these are correct:

```c#
var fruit = "Lychee";
var fruits = new List<Fruit>();
var flavor = fruit.GetFlavor();
FruitFlavor flavor = fruit.GetFlavor(); // this is preferred because result of GetFlavor is not evident
string fruit = null; // can't use "var" because the type isn't known (though you could do (string)null, don't!)
const string expectedName = "name"; // can't use "var" with const
```

The following are incorrect:

```c#
string fruit = "Lychee";
List<Fruit> fruits = new List<Fruit>();
```

#### Use C# type keywords in favor of .NET type names

When using a type that has a C# keyword the keyword is used in favor of the .NET type name. For example, these are correct:

```c#
public string TrimString(string s) {
    return string.IsNullOrEmpty(s)
        ? null
        : s.Trim();
}

var intTypeName = nameof(Int32); // can't use C# type keywords with nameof
```

The following are incorrect:

```c#
public String TrimString(String s) {
    return String.IsNullOrEmpty(s)
        ? null
        : s.Trim();
}
```

#### When to use internals vs public and when to use InternalsVisibleTo

As a modern set of frameworks, usage of internal types and members is allowed, but discouraged.

`InternalsVisibleTo` is used only to allow a unit test to test internal types and members of its runtime assembly. We do not use `InternalsVisibleTo` between two runtime assemblies.

If two runtime assemblies need to share common helpers then we will use a "shared source" solution with build-time only packages. Check out the https://github.com/aspnet/Mvc/tree/dev/src/Microsoft.AspNet.Mvc.Common project and how it is referenced from the `project.json` files of sibling projects.

If two runtime assemblies need to call each other's APIs, the APIs must be public. If we need it, it is likely that our customers need it.

#### Argument null checking

**Note:** The `[NotNull]` feature is not being implemented as a compile-time feature. To throw a runtime exception, add an explicit null check and throw an `ArgumentNullException`. Some repos still use the `[NotNull]` attribute, but it is for static analysis only and does not affect runtime behavior.

Null checking is required for parameters that cannot be null (big surprise!). To add null checking to your code, declare this attribute in your assembly in any namespace (use the JetBrains namespace to have ReSharper work):

```c#
using System;

namespace JetBrains.Annotations
{
    [AttributeUsage(
        AttributeTargets.Method | AttributeTargets.Parameter |
        AttributeTargets.Property | AttributeTargets.Delegate |
        AttributeTargets.Field, AllowMultiple = false, Inherited = true)]
    internal sealed class NotNullAttribute : Attribute
    {
    }
}
```

And then annotate parameters of methods or property setters:

```c#
public void GetBanana([NotNull] string variety)
{
    // do not do explicit null check in the method body!
    ...
}

public string Variety
{
    get;
    [param: NotNull]
    set;
}
```

The null checking code will be code-gen'ed at compile time into the method body.

##### Argument null checking in interface member definitions and abstract/virtual methods

If an interface member or abstract/virtual member contractually disallows nulls in its parameters, annotate them with `[NotNull]`.
The implementing method does *not* need the annotations - the code-gen will automatically emit the null checking code on the implementing method.

##### Argument null checking in chained constructors/methods

Null checks should be used on any public entry point where null is not allowed. This ensures the contract of not-nullable is seen by all callers.

```c#
public class Banana
{
    // Even though all this ctor does is chain the next ctor, it still must have NotNull annotations
    public Banana([NotNull] string name, [NotNull] string variety)
        : this(name, variety, string.Empty)
    {
    }

    public Banana([NotNull] string name, [NotNull] string variety, [NotNull] string color)
    {
        ...
    }
}
```

#### Async method patterns

By default all async methods must have the `Async` suffix. There are some exceptional circumstances where a method name from a previous framework will be grandfathered in.

Passing cancellation tokens is done with an optional parameter with a value of `default(CancellationToken)`, which is equivalent to `CancellationToken.None` (one of the few places that we use optional parameters). The main exception to this is in web scenarios where there is already an `HttpContext` being passed around, in which case the context has its own cancellation token that can be used when needed.

Sample async method:

```c#
public Task GetDataAsync(
    QueryParams query,
    int maxData,
    CancellationToken cancellationToken = default(CancellationToken))
{
    ...
}
```

#### Extension method patterns

The general rule is: if a regular static method would suffice, avoid extension methods.

Extension methods are often useful to create chainable method calls, for example, when constructing complex objects, or creating queries.

Internal extension methods are allowed, but bear in mind the previous guideline: ask yourself if an extension method is truly the most appropriate pattern.

The namespace of the extension method class should generally be the namespace that represents the functionality of the extension method, as opposed to the namespace of the target type. One common exception to this is that the namespace for middleware extension methods is normally always the same is the namespace of `IAppBuilder`.

The class name of an extension method container (also known as a "sponsor type") should generally follow the pattern of `<Feature>Extensions`, `<Target><Feature>Extensions`, or `<Feature><Target>Extensions`. For example:

```c#
namespace Food {
    class Fruit { ... }
}
namespace Fruit.Eating {
    class FruitExtensions { public static void Eat(this Fruit fruit); }
  OR
    class FruitEatingExtensions { public static void Eat(this Fruit fruit); }
  OR
    class EatingFruitExtensions { public static void Eat(this Fruit fruit); }
}
```

When writing extension methods for an interface the sponsor type name must not start with an `I`.

#### Doc comments

The person writing the code will write the doc comments. Public APIs only. No need for doc comments on non-public types.

Note: Public means callable by a customer, so it includes protected APIs. However, some public APIs might still be "for internal use only" but need to be public for technical reasons. We will still have doc comments for these APIs but they will be documented as appropriate.

#### Assertions

Use `Debug.Assert()` to assert a condition in the code. Do not use Code Contracts (e.g. `Contract.Assert`).

Please note that assertions are only for our own internal debugging purposes. They do not end up in the released code, so to alert a developer of a condition use an exception.

#### Unit tests and functional tests

##### Assembly naming

The unit tests for the `Microsoft.Fruit` assembly live in the `Microsoft.Fruit.Tests` assembly.

The functional tests for the `Microsoft.Fruit` assembly live in the `Microsoft.Fruit.FunctionalTests` assembly.

In general there should be exactly one unit test assembly for each product runtime assembly. In general there should be one functional test assembly per repo. Exceptions can be made for both.

##### Unit test class naming

Test class names end with `Test` and live in the same namespace as the class being tested. For example, the unit tests for the `Microsoft.Fruit.Banana` class would be in a `Microsoft.Fruit.BananaTest` class in the test assembly.

##### Unit test method naming

Unit test method names must be descriptive about *what is being tested*, *under what conditions*, and *what the expectations are*. Pascal casing and underscores can be used to improve readability. The following test names are correct:

```
PublicApiArgumentsShouldHaveNotNullAnnotation
Public_api_arguments_should_have_not_null_annotation
```

The following test names are incorrect:

```
Test1
Constructor
FormatString
GetData
```

##### Unit test structure

The contents of every unit test should be split into three distinct stages, optionally separated by these comments:

```c#
// Arrange  
// Act  
// Assert 
```

The crucial thing here is that the `Act` stage is exactly one statement. That one statement is nothing more than a call to the one method that you are trying to test. Keeping that one statement as simple as possible is also very important. For example, this is not ideal:

```c#
int result = myObj.CallSomeMethod(GetComplexParam1(), GetComplexParam2(), GetComplexParam3());
```

This style is not recommended because way too many things can go wrong in this one statement. All the `GetComplexParamN()` calls can throw for a variety of reasons unrelated to the test itself. It is thus unclear to someone running into a problem why the failure occurred.

The ideal pattern is to move the complex parameter building into the `Arrange` section:

```c#
// Arrange
P1 p1 = GetComplexParam1();
P2 p2 = GetComplexParam2();
P3 p3 = GetComplexParam3();

// Act
int result = myObj.CallSomeMethod(p1, p2, p3);

// Assert
Assert.AreEqual(1234, result);
```

Now the only reason the line with `CallSomeMethod()` can fail is if the method itself blew up. This is especially important when you're using helpers such as `ExceptionHelper`, where the delegate you pass into it must fail for exactly one reason.

##### Testing exception messages

In general testing the specific exception message in a unit test is important. This ensures that the exact desired exception is what is being tested rather than a different exception of the same type. In order to verify the exact exception it is important to verify the message.

To make writing unit tests easier it is recommended to compare the error message to the RESX resource. However, comparing against a string literal is also permitted.

```c#
var ex = Assert.Throws<InvalidOperationException>(
    () => fruitBasket.GetBananaById(1234));
Assert.Equal(
    Strings.FormatInvalidBananaID(1234),
    ex.Message);
```

##### Use xUnit.net's plethora of built-in assertions

xUnit.net includes many kinds of assertions – please use the most appropriate one for your test. This will make the tests a lot more readable and also allow the test runner report the best possible errors (whether it's local or the CI machine). For example, these are bad:

```c#
Assert.Equal(true, someBool);

Assert.True("abc123" == someString);

Assert.True(list1.Length == list2.Length);

for (int i = 0; i < list1.Length; i++) {
    Assert.True(
        String.Equals
            list1[i],
            list2[i],
            StringComparison.OrdinalIgnoreCase));
}
```

These are good:

```c#
Assert.True(someBool);

Assert.Equal("abc123", someString);

// built-in collection assertions!
Assert.Equal(list1, list2, StringComparer.OrdinalIgnoreCase);
```

##### Parallel tests

By default all unit test assemblies should run in parallel mode, which is the default. Unit tests shouldn't depend on any shared state, and so should generally be runnable in parallel. If the tests fail in parallel, the first thing to do is to figure out *why*; do not just disable parallel tests!

For functional tests it is reasonable to disable parallel tests.

#### Use only complete words or common-standard abbreviations in public APIs

Public namespaces, type names, member names, and parameter names must use complete words or common/standard abbreviations.

These are correct:
```c#
public void AddReference(AssemblyReference reference);
public EcmaScriptObject SomeObject { get; }
```

These are incorrect:
```c#
public void AddRef(AssemblyReference ref);
public EcmaScriptObject SomeObj { get; }
```
