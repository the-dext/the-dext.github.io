---
layout: post
title: "Eliminating If Statements"
draft: false
publish: true
tags: C# dotNet Functional
---
This is just a quick post to demonstrate a technique for replacing branching code (if-else and switch statements), and replacing it with a look-up. Sometimes by eliminating branching logic this can make your application easier to comprehend (which means more maintainable). 

By the end of the article you should feel quite comfortable in using the technique yourself.

_Note that the examples in this article require C# 7 or later._

## The Problem With If Statements
If statements are one of the fundamental building blogs of an application. I don't think I've ever worked on an app that didn't have at least one if statement and there's nothing wrong with using them per se. But these statements can get out of hand over time, they might run code blocks they run are complex, or have more nested if statements within them. Eventually this code can become quite hard to follow.

Sometimes developers will try to increase readability by replacing the if statements with switch statements. This can improve readability, but only up to a point and really isn't much of an improvement.

I'm going to show you a technique to replace your if and switch statements, but before that we need a small piece of demonstration code.

## Example: Before Refactoring
Lets start off with some code that we can refactor.
Below is an extract from a very simple console application I've written to demonstrate the technique. It uses a few if-then-else-if statements to work out which option a user has chosen, based on the keyboard character the user has pressed. 

If the user selects a valid option then the code for that choice will run and the application will end. 

If the user does not select a valid choice then the menu will be redisplayed and the user will be asked to select an option again.

This isn't supposed to be a complex solution, it's purely a demo. So try to imagine how in a real application there could be a lot of ifs/switches that make it difficult to follow the code.

```c#
static void Main()
{
    // infinite loop to keep the app going until a valid choice is made
    do
    {
        PrintMenu();

        var choice = Console.ReadKey();

        if (choice.KeyChar == 'D' || choice.KeyChar == 'd')
        {
            PrintResult("You voted for Dogs");
        }
        else if (choice.KeyChar == 'C' || choice.KeyChar == 'c')
        {
            PrintResult("You voted for Cats");
        }
        else if (choice.KeyChar == 'R' || choice.KeyChar == 'r')
        {
            PrintResult("You voted for Rabbits");
        }
        else
        {
            // invalid option so try again
            Console.WriteLine("Invalid selection, press return and select an option from the menu");
            Console.ReadLine();
        }
    } while (true);
}
```

We are going to refactor the code and remove the need for the if statements, but before we do this first of all we need to recognise the key elements of what is going on.

* We have an infinite loop that captures the user input and will repeat if the user entered an invalid choice. We will keep this functionality.

* Then for each if statement there is a condition that when met causes the body of the if statement to execute. We will refactor these.

* Finally we have the actual body of each if statement, these will also be refactored.

## Refactoring the Code

We are going to replace the if statements by using some functional C#. Thanks to Linq, tuples and functions being first class citizens in C# this will be quite simple.

*Note: Tuples are supported in C# 7 and later, so you need to be using at least version 7.*

First we will define a list, and this list will hold a number of values (the number of menu options +1).

Each value in the list will be a tuple.

**If you aren't sure what a tuple is then you can read about them [here](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/value-tuples), but for now think about them as a way for us to store related data without defining a class/struct**

Each of the tuples will hold 2 values. 
* The first value will be the predicate from each of the if-else statements (the predicate is the condition that is checked to be truthy before the inner if block is run).
To do this we will define the first tuple value as a `Func<char, bool>`. This is a function that accepts a char and returns true or false.
This corresponds to the predicate of the if statements.

* The second value will be an action to run when the first predicate value returns true. This corresponds to the body of the if statements. (An action is basically a Func that returns void).

### Step 1. Define a List to Hold the Commands
So the first step is to define a list, of tuples with each tuple having a `Func<char,bool>` and an Action. 
We can do this simply by defining the variable as below

```c#
private static List<(Func<char, bool> predicate, Action action)> commandMap;
```
### Step 2. Populate the Command List
Now we will add an entry to the list, one for each of the if statements within a new method that we can call when the console app starts.

```c#
private static void SetupCommandMap ()
{
    commandMap = new List<(Func<char, bool> predicate, Action action)>();

    commandMap.Add(((choice) => choice == 'D' || choice == 'd', () => PrintResult("You voted for Dogs")));
    commandMap.Add(((choice) => choice == 'C' || choice == 'c', () => PrintResult("You voted for Cats")));
    commandMap.Add(((choice) => choice == 'R' || choice == 'r', () => PrintResult("You voted for Rabbits")))

```
Notice that each time the `commandMap.Add` method is called there is a tuple passed. This is the first tuple above, I've spaced out the arguments for readability.
```c#
(  
    (choice) => choice == 'D' || choice == 'd', 
    () => PrintResult("You voted for Dogs")   
)
```
The first part of the tuple, before the ',' character is the predicate we defined. It's an anonymous function that takes a char named 'choice' and then tests that char to see if it is equal to 'D' or 'd'. If it is then it would return true, else it would return false. Just like when this code was part of original if statement.

The second part of the type, after the ',' character is the action. Here you can see that the action takes no argument, and calls the PrintResult method with the text that should be printed out. Hopefully you have already spotted that this is the same code that used to be in the body of the original if statement. 

There's one more thing to do to the list. You may have spotted that earlier I said the list would hold the number of menu items +1.
Well that '+1' is to account for the `else` statement at the end of the original code. Without that we haven't got like for like functionality, so to do this we will add another element to the list. But this time the predicate will always return true.
```c#
    // a default action to take when no command is matched
    commandMap.Add(((choice) => true, () => {
        Console.WriteLine("Invalid selection, press return and select an option from the menu");
        Console.ReadLine();
    }));
```
Notice that the action this time has more than one line of code, so C# requires us to wrap the action in {} brackets.

### Step 3. Replace the If Statements

Now we will use the command list we have built instead of the original if statements.
This will make the code much smaller by removing the branching that if statements cause.

We will do this by using the Linq operator `First` to select the first element of the command list that has a predicate which returns true, for the character that is passed in (the one pressed by the user).
This is really simple, you will probably have written a statement like this many times.

```c#
commandMap.First(x => x.predicate(choice));
```

The `First` operator will invoke the 'predicate' method on each element, starting at the first one and stopping as soon as one of the predicates return true, and returning that tuple from the list.

Remember that if no predicate tuple for the commands that replaced if statements returns true we are covered because the last element in the list has a predicate that is ALWAYS true. So we are guaranteed to have a return value.

Because we know that the First operator is going to return us a value, and that value is an `Action` on a property we named 'action', we can invoke it inline making the line of code above become

```c#
commandMap.First(x => x.predicate(choice)).action();
```

## The End Result
The end result of the refactoring is that the Main method looks much smaller and has no reason to change when new options are added. The branching of if (or switch) statements is gone and replaced with a simple functional programming style call using Linq.

```c#
static void Main()
{
    SetupCommandMap();

    do
    {
        PrintMenu();

        var choice = Console.ReadKey().KeyChar;
        commandMap.First(x => x.predicate(choice)).action();
    } while (true);
}
```

## That's All For Now
Thanks for reading this far. Hopefully you found this article interesting and you can add this technique to your skill set. This technique can help the readability of your code but it's not a magic bullet so I encourage you to experiment and find what works for you.

You can see the full solution (before and after) on my [GitHub repository](https://github.com/the-dext/blog-eliminating-if-statements)
