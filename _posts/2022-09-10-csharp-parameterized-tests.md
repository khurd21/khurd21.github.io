---
title: Parameterized Unit Tests in C#
categories: [ programming ]
tags: [ c#, unit-testing, parameterized-tests ]
last_modified_at: 2022-09-10T12:00:00-05:00
---

During one of my classes at university, we covered how to write
quality unit tests. I noticed early on that some of the tests
were very similar to each other and I often found myself
unnecessarily repeating code or writing tests that were
testing for the same output, but with different inputs. I was not
satisfied with this and eagerly searched for a method to
abstract away the repeated code and make my tests more
readable.

I found a solution to this problem, and the term for it is
"parameterized tests." This form of testing allows the programmer
to write a test that can accept multiple inputs. In this blog, I
will be covering how one can write parameterized tests
in C#. In a future blog I may cover this concept using Python.
I find this tool to be very useful and important
to know about when writing unit tests.

# C# Parameterized Tests

In my example, I will be
creating a simple expression validator that will check if a
string is a valid mathematical expression. The implementation is
not super important, and I am only using it to demonstrate how
to write parameterized tests.

## Set Up the Project

To start, I will create a new C# project using dotnet CLI. I am using
dotnet 6.

```shell
dotnet new sln -n ExpressionValidator
dotnet new classlib -n ExpressionValidator
dotnet new nunit -n ExpressionValidator.Test

dotnet sln add ExpressionValidator
dotnet sln add ExpressionValidator.Test
dotnet add ExpressionValidator.Test reference ExpressionValidator

dotnet build
```

## Create the Expression Validator

Great! Now that we have a project set up, I am going to create two
`ExpressionValidator` classes. One will be for math formula, and the
other will be for confirming an expression is a valid title. The code
here is not important, we are only using it to demonstrate how to
write parameterized tests.

Here is the structure:

```
|- ExpressionValidator
|  |- TitleExpressionValidator.cs
|  |- IExpressionValidator.cs
|- ExpressionValidator.Test
|  |- TitleExpressionValidatorTest.cs
```

Since the validator is going to be testing if an expression
is valid or not, I will create an interface to define the
`IsValid` method.

```csharp
// IExpressionValidator.cs

namespace ExpressionValidator;

public interface IExpressionValidator
{
    bool IsValid(in string expression);
}
```

Below is the implementation of the `IExpressionValidator` interface.

```csharp
// TitleExpressionValidator.cs

namespace ExpressionValidator;

public class TitleExpressionValidator : IExpressionValidator
{
    public bool IsValid(in string expression)
    {
        List<string> words = new(expression.Split(' '));
        return words.All(word => word.Length > 0 && word[0] == char.ToUpper(word[0]));
    }
}
```

## Create the Tests

Now let's write some tests. I will start with with the `TitleExpressionValidator` tests. The current state of the unit tests
are acceptable and they are passing. However, they do not promote extensibility. If I wanted to add more tests for valid scenarios, how
would I do that with the current implementation?

```csharp
// TitleExpressionValidatorTest.cs

namespace ExpressionValidator.Test;

[TestFixture]
public class TitleExpressionValidatorTest
{
    private IExpressionValidator ExpressionValidator { get; set; } = default!;

    [SetUp]
    public void SetUp()
    {
        this.ExpressionValidator = new TitleExpressionValidator();
    }

    [Test]
    public void IsValid_WhenExpressionIsValid_ReturnsTrue()
    {
      Assert.IsTrue(
        this.ExpressionValidator.IsValid("Hello, World!"),
        message: $"Expression: `Hello, World!` should be valid.");
    }

    [Test]
    public void IsValid_WhenExpressionIsNotValid_ReturnsFalse()
    {
        Assert.IsFalse(
          this.ExpressionValidator.IsValid("hello world"),
          message: $"Expression: `hello world` should not be valid.");
    }
}
```

```shell
❯ dotnet test
Starting test execution, please wait...
A total of 1 test files matched the specified pattern.

Passed!  - Failed:     0, Passed:     2, Skipped:     0, Total:     2, Duration: 3 ms
```

The ability to parameterize tests in C# is actually quite simple and very cool.
There are two ways to do this. The first way is to use the `TestCase` attribute.
The second way is to use the `TestCaseSource` attribute. I will be covering both
ways in this blog. `TestCase` is a great way to parameterize tests when there
are not many test cases. If there are a lot of test cases, or if the test cases
can be shared between multiple test classes, then `TestCaseSource` is the way
to go.

In the improvement below, we helped make the test cases more readable and
modular. Now adding an additional test case, or removing a test case, is
as simple as adding or removing a line of code.

```csharp
[Test]
[TestCase("Hello World")]
[TestCase("Hello, World!")]
[TestCase("Greetings, From Mars")]
public void IsValid_WhenExpressionIsValid_ReturnsTrue(in string expression)
{
    Assert.IsTrue(
        this.ExpressionValidator.IsValid(expression),
        message: $"Expression: {expression} should be valid.");
}

[Test]
[TestCase("hello world")]
[TestCase("hello, world!")]
[TestCase("Greetings, From mars")]
public void IsValid_WhenExpressionIsNotValid_ReturnsFalse(in string expression)
{
    Assert.IsFalse(
        this.ExpressionValidator.IsValid(expression),
        message: $"Expression: {expression} should not be valid.");
}
```

```shell
❯ dotnet test
  Determining projects to restore...
  Starting test execution, please wait...
A total of 1 test files matched the specified pattern.

Passed!  - Failed:     0, Passed:     6, Skipped:     0, Total:     6, Duration: 4 ms
```

For this example, stopping here seems perfectly fine. However, let's continue
to try and abstract away at the test cases. Let's say the implementation of
`TitleExpressionValidator` is much more complex and there are a lot of test
scenarios to cover. It would be nice to group the test scenarios together
if the list of test cases grows. This is where `TestCaseSource` comes in.

I found the cleanest way for me to store the test cases is to create a
`TestCaseSources` class containing all of the test cases for the method
under test. Once this is done, we can use the `TestCaseSource` attribute
to reference the test cases.

```csharp
// TestCaseSources/ExpressionValidatorTestCaseSources.cs

namespace ExpressionValidator.Test.TestCaseSources;

public static class ExpressionValidatorTestCaseSources
{
    public static IEnumerable<TestCaseData> IsValid_WhenExpressionIsValid_ReturnsTrue
    {
        get
        {
            yield return new TestCaseData("Hello World").Returns(true);
            yield return new TestCaseData("Hello, World!").Returns(true);
            yield return new TestCaseData("Greetings, From Mars").Returns(true);
        }
    }

    public static IEnumerable<TestCaseData> IsValid_WhenExpressionIsNotValid_ReturnsFalse
    {
        get
        {
            yield return new TestCaseData("hello world").Returns(false);
            yield return new TestCaseData("hello, world!").Returns(false);
            yield return new TestCaseData("Greetings, From mars").Returns(false);
        }
    }
}
```

Modified Test Cases:


```csharp
[Test]
[TestCaseSource(
    typeof(ExpressionValidatorTestCaseSources),
    nameof(ExpressionValidatorTestCaseSources.IsValid_WhenExpressionIsValid_ReturnsTrue))]
public bool IsValid_WhenExpressionIsValid_ReturnsTrue(in string expression)
{
    return this.ExpressionValidator.IsValid(expression);
}

[Test]
[TestCaseSource(
    typeof(ExpressionValidatorTestCaseSources),
    nameof(ExpressionValidatorTestCaseSources.IsValid_WhenExpressionIsNotValid_ReturnsFalse))]
public bool IsValid_WhenExpressionIsNotValid_ReturnsFalse(in string expression)
{
    return this.ExpressionValidator.IsValid(expression);
}
```

```shell
❯ dotnet test
  Determining projects to restore...
  Starting test execution, please wait...
A total of 1 test files matched the specified pattern.

Passed!  - Failed:     0, Passed:     6, Skipped:     0, Total:     6, Duration: 8 ms
```

## Optimal Solution

This new design of utilizing `TestCaseSource` is a very powerful feature
in C#. Looking at the last implementation, we can combine the two test
methods into one. Taking advantage of the `TestCaseData` class, we can
declare the expected result of the test case.

Here is the final product:

```csharp
// TestCaseSources/ExpressionValidatorTestCaseSources.cs

namespace ExpressionValidator.Test.TestCaseSources;

public static class TitleExpressionValidatorTestCaseSources
{
    public static IEnumerable<TestCaseData> IsValid
    {
        get
        {
            yield return new TestCaseData("Hello World").Returns(true);
            yield return new TestCaseData("Hello, World!").Returns(true);
            yield return new TestCaseData("Greetings, From Mars").Returns(true);

            yield return new TestCaseData("hello world").Returns(false);
            yield return new TestCaseData("hello, world!").Returns(false);
            yield return new TestCaseData("greetings, from mars").Returns(false);
        }
    }
}
```

```csharp
// TitleExpressionValidatorTest.cs

using ExpressionValidator.Test.TestCaseSources;

namespace ExpressionValidator.Test;

[TestFixture]
public class TitleExpressionValidatorTest
{
    private IExpressionValidator ExpressionValidator { get; set; } = default!;

    [SetUp]
    public void SetUp()
    {
        this.ExpressionValidator = new TitleExpressionValidator();
    }

    [Test]
    [TestCaseSource(typeof(TitleExpressionValidatorTestCaseSources), nameof(TitleExpressionValidatorTestCaseSources.IsValid))]
    public bool IsValidTest(in string expression)
    {
        return this.ExpressionValidator.IsValid(expression);
    }
}
```

```shell
❯ dotnet test
  Determining projects to restore...
  Starting test execution, please wait...
A total of 1 test files matched the specified pattern.

Passed!  - Failed:     0, Passed:     6, Skipped:     0, Total:     6, Duration: 7 ms
```

And there we go. We have improved how we can write our test cases. Learning
these new features of NUnit and how C# handles tests has been a great
learning opportunity and I hope you enjoyed reading this as much as I
enjoyed writing it.
