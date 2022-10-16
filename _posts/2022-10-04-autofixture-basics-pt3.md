---
layout: post
title: AutoFixture Basics
subtitle: <span class='subtitle'>Part 3, Simplifying Tests with Complex Types</span>
draft: false
publish: true
tags: C# dotNet TDD Test-Driven-Development
date: 2022-10-16 00:00 +0000
excerpt_separator: <!--more-->

---

# Writing Unit Tests With Anonymous Data

If you've been following this series you'll know that in part 2 we covered using AutoFixture to create anonymous data four our tests. In this article we will take another look at some of the unit tests in our project and see how we can use AutoFixture to create complex types and how this will lead to simpler unit tests.

<!--more-->

---
## Recap

In part 2 of this series I described how we could take a test that explicitly setup test data, and change it to use anonymous data by using AutoFixture.
At the end of part two, the example unit test looked like this.

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

## This Test is Still a Brittle Test
This test is still a brittle test. If the constructor arguments change for the `BlogPost` then the test will fail to compile. *This is desirable* because the purpose of the test is to verify that the BlogPost constructor set the Id, Title and PostDateTime properties , therefore it's reasonable for the test to fail compilation.

If the constructor arguments for the `BlogAuthor` change, then this will also cause this test to fail compilation.
*This is not desirable* because this test does not exist to test the BlogAuthor constructor. It's being forced to change for an unrelated reason.
Therefore it is a brittle test.

## A Less Obvious Brittle Test
The `Can_Construct_BlogPost` is also a brittle test, but for 2 reason. The first reason is if the `BlogAuthor` constructor changes, just like in the previous example. But can you spot the second reason ?
```c#
[Fact]
public void Can_Construct_BlogPost()
{
    var fixture = new Fixture();
    fixture.Register<DateTime>(() => DateTime.Now);

    var sut = new BlogPost(fixture.Create<string>(), fixture.Create<string>(), new BlogAuthor(fixture.Create<string>()), fixture.Create<DateTime>());
    sut.Should().NotBeNull();
}
```

Well done if you worked out that the test is also brittle because if the `BlogPost` constructor changes then the test doesn't compile.
So why is this an issue if it wasn't in the previous example ?
Well the answer is that this test is interesting in knowing if a `BlogPost` can be constructed, but unlike the previous example it is not interested in whether or not the properties of the result have been initialized. Because the test has to be amended for reasons other than what the test is interested it, it is a brittle test.

I'm going to change the `BlogAuthor` constructor to accept a first name and surname instead of just a name. The new constructor signature is now `BlogAuthor(string firstName, string surname)`.
This breaks 4 unit tests and I'll fix them all using the techniques I'm going to show now. But for demonstration purposes we only need to concentrate on fixing the two I've shown above. 

## Simplifying This Test Further With Complex Types
### Fixing Can_Construct_BlogPost
Lets start with the easier of the two tests. To update the `Can_Construct_BlogPost` so that it isn't a brittle test, all we are going to do is ask AutoFixture to create the BlogPost for us, rather than the individual arguments for the constructor.
Remember this is fine for this test because it doesn't test the values. The new version of the test now looks like this.

```c#
[Fact]
public void Can_Construct_BlogPost()
{
    var fixture = new Fixture();
    fixture.Register<DateTime>(() => DateTime.Now);

    // this is the new line, AutoFixture now creates the BlogPost
    var sut = fixture.Create<BlogPost>();
    sut.Should().NotBeNull();
}
```

### Fixing Constructor_Initializes_BlogPost
This test is more complicated, but I've fixed it by asking AutoFixture to once again create the `BlogAuthor`. It now looks like this.

```c#
[Fact]
public void Constructor_Initializes_BlogPost()
{
    var fixture = new Fixture();
    fixture.Register<DateTime>(() => DateTime.Now);

    string expectedBlogPostTitle = fixture.Create<string>();
    string expectedBlogBody = fixture.Create<string>();

    var expectedBlogPostDateTime = fixture.Create<DateTime>();
    var expectedBlogAuthor = fixture.Create<BlogAuthor>();

    var sut = new BlogPost(expectedBlogPostTitle, expectedBlogBody, expectedBlogAuthor, expectedBlogPostDateTime);

    sut.Id.Should().NotBeEmpty();
    sut.Title.Should().Be(expectedBlogPostTitle);
    sut.PostDateTime.Should().Be(expectedBlogPostDateTime);
    sut.Author.Should().Be(expectedBlogAuthor);
}
```

## Compromises

AutoFixture fully supports creating nest types so I could have gone further in this test and used AutoFixture ot create the `BlogPost`. If I do this though then the test won't be able to assert that the values of the properties in the `BlogPost` match the values passed into the constructor, because we won't know what values to expect. 

Instead the best we can do would be to assert that the properties are set to *something*, even if it's not the right thing.
For the sake of completeness, this is an example of what the test would look like *if* I went down this route (you won't find this in the example code repository).

```c#
[Fact]
public void Constructor_Initializes_BlogPost()
{
    var fixture = new Fixture();
    fixture.Register<DateTime>(() => DateTime.Now);

    var sut = fixture.Create<BlogPost>();

    /* We don't know these values anymore so we can't test for them explicitly.
        sut.Title.Should().Be(expectedBlogPostTitle);
        sut.PostDateTime.Should().Be(expectedBlogPostDateTime);
        Instead we will test that the properties have some value assigned to them */
    sut.Id.Should().NotBeEmpty();
    sut.Title.Should().NotBeEmpty();
    sut.PostDateTime.Should().NotBe(DateTime.MinValue);
    sut.Author.Should().NotBeNull();
}
```

###  This Example *Wouldn't be Brittle Enough*
This example would also introduce a new issue, the test *wouldn't be brittle enough* !
If we introduced new constructor arguments to `BlogPost` then there would be no failure in compilation, and nothing to prompt us to come back to this test and add an assertion for the new property.
This may not be an issue if you're strictly doing TDD, but if you aren't then it's important to recognize the potential for an issue and be cautious.

## Part 4 - No Compromises

In a situation like the one demonstrated in the `Constructor_Initializes_BlogPost` test you have a few options, the most obvious is by using AutoFixture as I demonstrated earlier in this article. 

But the flawed example was certainly a lot shorter and easier to read. Isn't there a way to keep this test short but also have accurate assertions ?

Yes, and in part 4 I'll show you how.