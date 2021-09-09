---
layout: post
title: TDD - Mocking vs No Mocking - Here's Why You Should Do Both
draft: (draft - subject to revision)
---
If you're writing unit tests you'll probably have heard the debate about whether or not you should use mock objects. 

Or maybe you're struggling with difficult to write or maintain tests because they're complex?

In this article I'm going to attempt to show that you need both techniques in your solutions and show how understanding when to use each style of test will help you on your TDD and DDD journey.

Bear in mind when reading this article that I am a huge advocate of Domain Driven Design and structure applications following the Onion Architecture. There is going to be some DDD terminology (but not too much).

---

## Classicist vs Mockist
The debate over whether or not to use mock objects is known as 'Classicist vs Mockist' or 'Detroid vs London'. Martin Fowler wrote about this extensively in his blog which you can read more about [right here](https://martinfowler.com/articles/mocksArentStubs.html#ClassicalAndMockistTesting).

In this article I'd like to persuade you not to choose one style over the other, but to use both styles and help you recognise when each is appropriate.

In my opinion this debate isn't helpful, there's another more useful question to consider.

## Orchestration vs Business Logic!
When you are testing a class (from here on I'll call this class the 'SUT' - the Subject Under Test) the class and the layer that it is within should fall into one of two categories.
- Orchestration
- Business Logic

### Orchestrator Classes
The orchestrator classes deal with coordination of work. The sort's of classes that fall into this category are (but not limited to) web API controllers, repositories and most importantly of all the classes in your 'application' layer.

If you were to apply functional programming terminology to these types of classes, many of them would be 'impure', because even given the same initial state (if that's possible) and the same actions, it's still possible that a different result could occur, for example the current Date/Time has changed, or a network request fails.

### Business Logic Classes
The business logic classes should live in your Domain layer. These are the brains of your business, the rules that you enforce, the secret ingredient that makes your app better than everyone elses.

If you were to apply functional programming terminology to these types of classes, many of them would be 'pure', because given the same state and the same actions, the result would always be the same.

### Classes That Do Both
In my opiniion classes that do both orchestration and business logic are not something you should be seeing in good object oriented design. 
They are procedural programming (possibly in an object oriented language) and you should avoid these like the plague.

---

## Testing Orchestration
Testing Orchestration is where mocking can be extremely useful.
Here are some examples
- Returning a 'Not Found' response when the repository does not find data.
- Confirming that 'commit' is called on a unit of work.
- Asserting that a message is placed on a queue.

All of the above are perfect candidates for using mocks, in fact if you are unit testing it would be very difficult to test this scenarios without a mock and you wouldn't be dealing with a single unit in this case.

Notice that all the examples above would need you to verify a call on a mock, and that makes sense because the dependencies don't have state.

If you aren't using a mock here then I would argue that you are actually writing an integration test and that they are different from unit tests.

## Testing Business Logic
Testing business logic should not involve dependecies (such as repositories for example) and should be just about the business rules of your app.

This can be as simple as calling a method that adds two numbers and returns a result, but the real world is often more complex than this and I am willing to bet your application is too. So lets consider how you should go about testing an Aggregate.

#### A Quick Recap of Aggregates
I won't go into all the details about what an aggregate is, but you need to understand some basics.
>* An aggregate is a collection of classes that are closely related and work together. 
>
>* They form a consistency boundary.
>
>* They are fundamental to good Object Oriented Design.
>
>* One class in an aggregate is the 'Aggrgate Root'
>
>* Actions are performed against the aggregate root, not the aggregate members.
>
>* They are a black box (they encapsulate behaviour)

The last two points above are the most important when it comes to unit testing an aggregate.

You should write tests that perform actions against the Aggregate Root. Don't write tests for any other classes in the aggregate, only the aggregate root. This is important to allow you to refactor aggregates easily.

---

## The Benefits of Testing Isolated Business Logic

### Tests Aren't Brittle 

By avoiding use of dependencies that require mocking your unit tests will be much simpler to write but there's another very important benefit to following these principles in that your tests will not be brittle.

Brittle tests are where the unit test is too focussed on how something is done instead of whether the result is correct.

When your tests do not concern themselves with what is done you are free to refactor your aggregate as much as you want and provided that the interface of the aggregate doesn't change then all your unit tests (and indeed the usage of your aggregate in your app) will still be valid and acting as a safety net, telling you if break your logic.

### You Can Achieve 100% Coverage

With a well isolated business logic/domain layer it's perfectly reasonable to aim for **100%** code coverage. There really is no reason or excuse not to because the unit tests become simple to write.

When testing orchestration or procedural code there is a deminishing return as the coverage increases and it's difficult if not impossible to reach 100% coverage. You shouldn't expect to. 
But in my experience that really doesn't happen with an isolated domain model.

### Tests Are Simpler
Tests for business logic are simpler because there's no mocking framework knowledge required. You won't find any advanced mocking black magic such as callbacks for example.

## But What If My Orchestration and Business Logic Are Mixed?
Well, that is your problem right there.

TDD is about using tests to create a feedback loop and discover software *Design* more than it is about having test assertions (though they're a benefit too). 

If you cannot avoid a test having to deal with orchestration and business logic then instead of struggling with the test and trying to arrange all the mocks to go down the correct code path, then stop!
The unit test is telling you that your SUT has a mix of concerns and you need to look at ways to separate them out so that you can follow the two testing styles above.

If you are unable to separate them for some reason - maybe the code is too complex and high risk - then consider whether unit testing is the right thing to be using.
You may be better off having an integration or end-to-end test instead so that you can treat the class as a black box.

---

## Next Time You're Writing A Unit Test

Consider the layers that you're dealing with and aim to put the right kind of code, in the right kind of layer, with the right kind of unit testing.
Notice how difficult to write tests are informing you of design issues.

The TDD process should become pretty smooth when you've practiced this for a while and eventually it becomes second nature.