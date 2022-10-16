---
layout: post
title: AutoFixture Basics
subtitle: <span class='subtitle'>Part 2, Using Simple Anonymous Data</span>
draft: false
publish: true
tags: C# dotNet TDD Test-Driven-Development
date: 2022-10-04 20:41 +0000
excerpt_separator: <!--more-->

---

# Writing Unit Tests With Anonymous Data

In my previous article I setup a fictitious project to demonstrate how brittle unit tests can create a burden on developers as they implement applications and take time to refactor their codebase.
This is the complete opposite of what you want and need from unit tests, it's vital that unit tests are there to help you refactor with confidence, rather than discourage you from refactoring.

In this second article I am going to demonstrate how the tests we wrote can be rewritten to use AutoFixture and anonymous data.

<!--more-->

---
## Anonymous Data
The main feature of AutoFixture is the ability to create anonymous data, but what is anonymous data?
Well, anonymous data is basically data that you don't care to explicitly know the details of. In the same way that c# will allow you to create an anonymous object by using syntax such as
    `var person = new { Name = "John", Surname = "Doe" };`
in this example the anonymous part of the object is the class definition, we haven't defined a class name and we don't care about the implementation details of it, we just want an object instance where the object has a Name and a Surname property.

In a similar vein AutoFixture will allow to ask for a variable, without caring about what the value of that variable is.

## Adding AutoFixture to the Test Projects
Install the main AutoFixture package into each test project using either the command line or a GUI package manager, at the time of writing the latest version is 4.17.0 but I would recommend always using the latest version.
    `NuGet\Install-Package AutoFixture -Version 4.17.0`

## The Fixture Class
The main class that is used in AutoFixture, is the `Fixture` class.
AutoFixture uses this class to implement a generic version of the test builder pattern. 

I actually like to hide the use of AutoFixture behind a test builder pattern of my own, but that's for going into another time, in the examples I'm going to show you we will keep things very simple.

## Applying AutoFixture to the Can_Construct_Blog Test
Lets start off with one of the simpler unit tests in the project, the `Can_Construct_Blog` test.
Currently the test looks like this

```c#    
[Fact]
public void Can_Construct_Blog()
{
    const string expectedBlogName = "My_Test_Blog";

    var sut = new Blog(expectedBlogName);

    sut.Should().NotBeNull();
}
```
We can apply AutoFixture to this test by creating a new instance of the `Fixture` class and asking it to create a string instance, instead of using our own string. 
```c#
[Fact]
public void Can_Construct_Blog()
{
    var fixture = new Fixture();
    var expectedBlogName = fixture.Create<string>();

    var sut = new Blog(expectedBlogName);

    sut.Should().NotBeNull();
}
```
The test doesn't look that much different to what it did before, but the difference is important. Instead of specifying the value of the blog name we now use AutoFixture to create an anonymous string by calling `fixture.Create<string>()`. 

Lets update another test

## Applying AutoFixture to the Constructor_Initialises_BlogPost Test

This test has more setup, before we make any changes it looks like this
```c#
[Fact]
public void Constructor_Initializes_BlogPost()
{
    const string expectedBlogPostTitle = "my test blog post";
    const string expectedBlogBody = "test blog body";
    var expectedBlogPostDateTime = DateTime.Now;
    var expectedBlogAuthor = new BlogAuthor("john.doe");

    var sut = new BlogPost(expectedBlogPostTitle, expectedBlogBody, expectedBlogAuthor, expectedBlogPostDateTime);

    sut.Id.Should().NotBeEmpty();
    sut.Title.Should().Be(expectedBlogPostTitle);
    sut.PostDateTime.Should().Be(expectedBlogPostDateTime);
}
```

If we apply AutoFixture and use the technique shown in the previous test, we can replace the first for lines of expectation setup end up with this..
```c#
    var fixture = new Fixture();

    string expectedBlogPostTitle = fixture.Create<string>();
    string expectedBlogBody = fixture.Create<string>();
    var expectedBlogPostDateTime = fixture.Create<DateTime>();
    var expectedBlogAuthor = new BlogAuthor(fixture.Create<string>());
```
After making these changes the test doesn't look much different but we have removed all explicit variable values.

But we have introduced a problem, sometimes.
I ran this test five times, and twice it failed with this message

`System.ArgumentException : postDateTime date cannot be in the past`

AutoFixture uses various strategies for generating data (which I believe are called Specimen Builders, but I could be mistaken). In the case of creating strings AutoFixture will simply generate a new Guid value.
But it can't do this for dates and numeric types. 

In the case of dates AutoFixture will generate a value between now -2 years, and now + 2 years.

If we provide a date in the past our domain object will throw an exception, so we need to tell AutoFixture that when we ask for this date there is a restriction.

### Discovering unexpected values is a good thing
This problem demonstrates one of the *benefits* of using AutoFixture to create anonymous data. In a real project written using TDD, we could have easily  focused on the happy path through our implementation and completely missed validating that the date cannot be in the past.
By having AutoFixture generating this data for us it improves our chances of finding edge case problems.

But now that AutoFixture is triggering this exception we need to amend the test and tell AutoFixture that we want to override the default creation of a date.
One way to do this is by registering a creation function.

### Register<T>
The `Register<T>` method will allow us to tell AutoFixture what to supply as a value, whenever it needs to create an instance of `T`. If we apply this to our DateTime problem we can make sure that we always get back a DateTime that is valid for our test case.

`fixture.Register<DateTime>(() => DateTime.Now);`

### End Result
The end result of updating the test is shown below.
```c#
[Fact]
public void Constructor_Initializes_BlogPost()
{
    var fixture = new Fixture();
    fixture.Register<DateTime>(() => DateTime.Now);

    string expectedBlogPostTitle = fixture.Create<string>();
    string expectedBlogBody = fixture.Create<string>();
    var expectedBlogPostDateTime = fixture.Create<DateTime>();
    var expectedBlogAuthor = new BlogAuthor(fixture.Create<string>());

    var sut = new BlogPost(expectedBlogPostTitle, expectedBlogBody, expectedBlogAuthor, expectedBlogPostDateTime);

    sut.Id.Should().NotBeEmpty();
    sut.Title.Should().Be(expectedBlogPostTitle);
    sut.PostDateTime.Should().Be(expectedBlogPostDateTime);
}
```
## Part 3 - Simplifying Tests

In this post we applied very basic application of AutoFixture to create anonymous instances of .net built-in types such as strings and dates.

Our final tests are no longer explicitly defining values to use in the tests, but now that we have introduced AutoFixture we can use it to simplify them much more. In part 3 I'll show you how.