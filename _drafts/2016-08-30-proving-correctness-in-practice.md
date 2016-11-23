---
layout: post
title: Proving correctness in practice
date: '2016-08-30T00:00:00.000-00:00'
tags:
- "software engineering"
- "formal verification"
- "type theory"
- "type systems"
- "static typing"
modified_time: '2016-08-30T00:00:00.000-00:00'
thumbnail:
blogger_id:
blogger_orig_url:
---

Software engineers deliver programs. Most of us write programs that provide some kind of service to a business or to the end user. Our biggest risks are service outage and data loss. Some of us write programs that drive cars, fly airplanes, and manage nuclear power plants. The stakes are much higher in the latter cases, but we still have the same goal: correct, reliable programs.

There are 4 steps to create a program.
1. We observe an opportunity for a program.
2. We create a mental model of the program that will aid us.
3. We write down a specification of the mental model.
4. We write a program that adheres to the specification.

The first 3 steps are out of the scope of this article, although they are very important and hugely discussed topics. We concern ourselves with step 4, where we take a specification and write a program that adheres to it.

## Background

Historically, computer scientists focused on the reliability of programs. After all, real world computer programs were extremely safety-critical for a long time: the Apollo program that landed us on the moon, nuclear reactors, missile systems, fighter jets and other military equipment. Since we didn’t want to accidentally start a nuclear war, we started testing our software before using it.

Software testing is expensive and error prone. Manual testing requires  human labor and time. Automated testing requires writing good tests, which is more difficult than most people assume. Although software testing gives us some guarantees, the need for better tools that allow us to write safer programs grew. Therefore we started borrowing concepts from mathematicians.

## Static typing and typechecking

Static typing is one such concept that we are all familiar with today. In statically typed languages, we annotate types in our program and have the compiler prove that the types of variables and expressions are consistent. We use this tool to prove certain properties of our programs. The more powerful the type system is, the more properties we can prove.
Type safety isn’t a dichotomy: it doesn’t make sense to call our program type safe. Instead, we should say *"these properties of my program are proven by type checking"*.

Compare the JavaScript function `function parseCsvLine(line) { ... }` to the Java function `StringList parseCsvLine(String line) { ... }`. The latter guarantees that no matter what, we always get a list of strings. By simply adding the type information and having the compiler check them, we have eliminated a whole class of bugs where the function could return something that is not a list of strings. We don’t even need to unit tests those cases anymore.

Could we do better? Java’s type system makes it possible for our function to return a null, or receive null as its argument. Surely one could say “I’m not that dumb, I will simply check whether the returned value is a null or not”. However, having to do null checks for all references is error prone, and any software company will have a mountain of segmentation faults or Null Pointer Exceptions in their bug tracker to support this point. There’s a reason Sir Tony Hoare called it his “billion dollar mistake”.

# Non-nullable references

So we need to guarantee that our program can’t cause an NPE. Let’s create a [variant of Java](https://kotlinlang.org/docs/reference/basic-syntax.html) that uses non-nullable references by default, and allows explicit nullable types denoted by “?”. This language should also reject dereferencing like “obj.method()” where obj has a nullable type.

{% highlight java %}
String str = maybeNull();  // compiler error, maybeNull can return
                           // null, but str has a non-nullable type.
String? str = maybeNull(); // it compiles.
{% endhighlight %}

Our `parseCsvLine()` function is now guaranteed to return a StringList and never a null. We are also required to pass it a String, not a null.

# Error handling

Let’s continue improving our program. It’s very possible that our CSV parser function will encounter a bad line that cannot be parsed. An example would be `field1,field2,"field3`. Notice the unfinished string literal at the end. We’d like our function to tell us about the bad string. There are a few ways of implementing this, depending on the language’s features and the school of error handling:

1. **Returning null**. We could make our return type nullable, and return null whenever the string can’t be parsed. The downside of this approach is that we can’t provide more information about the error that happened.
2. **Returning an error code**. This is the classical error handling method in C APIs. We change our signature to the following: int parseCsvLine(String inStr, StringList outStrs). The returned value is 0 if the function was successful, otherwise a non-zero error code is returned. The downside of this approach is that we can forget to check the error code and continue execution with an invalid outStrs.
3. **Unchecked exceptions**. Exceptions stop the execution, go up the stack trace and try to find an exception handler. If none is found, the program is terminated. Unchecked exceptions make it easy to lazily write prototype programs that assume nothing will go wrong. But since they are not required to be handled, it is easy to forget handling error cases and cause an abrupt, unwanted termination of the program.
4. **Checked exceptions**. Exceptions that are required to be handled, or the compiler will reject the code. The function that calls a function that might throw an exception must either catch the exception, or explicitly allow it to bubble up. While this is a very good mechanism for ensuring that all error cases are handled, it is infamous for slowing down development.

Functional programming languages usually take a different approach. They allow the construction of tagged union types. Compare this C union type:

{% highlight c %}
union StringOrInt {
  char* s;
  int i;
}
{% endhighlight %}

to this Haskell union type:

{% highlight haskell %}
data StringOrInt = S String | I Int
{% endhighlight %}

The C union requires you to be careful and only read from the union element that you have last written to, meaning you have to keep track of which element of the union is valid at a given time. Haskell’s tagged unions, on the other hand, keep the type information at runtime and force you to do a check when you want to read from the union. Let’s demonstrate this.

This C code gives no compiler error:

{% highlight c %}
StringOrInt u;
u.s = "test string";
printf("Here's your number: %d\n", u.i); // OOPS, invalid read!
{% endhighlight %}

while Haskell will prevent us from doing this mistake:

{% highlight haskell %}
getInteger :: StringOrInt -> Int
getInteger (I i) = i
let u :: StringOrInt = S "test string" in getInteger u
-- Compile error: Non-exhaustive patterns in function getInteger
{% endhighlight %}

 5\. **Union return types**. We can use a tagged union as our return type. This forces our caller to handle both error and success cases. This is actually a simpler and less powerful version of checked exceptions, but it is also very easy to understand and use.

# Unreachable states

The benefit of exceptions and union types is that they make certain states unreachable. When `parseCsvLine()` returns a union type or throws an exception, we can’t access a bad list of strings because the code will be either in an exception handler or in a different branch of a switch-case statement.

It’s possible to use the tagged union mechanism to hide more states that we think should be unreachable. Imagine a media player application where the user opens a video file or an audio file. There are 3 possible states: no open file, an open video file, or an open audio file. The C structure that holds our application state can be the following:

{% highlight c %}
struct AppState {
  VideoFile* video_;
  AudioFile* audio_;
};
{% endhighlight %}

video_ and audio_ each have 2 states: they are either null or non-null. When a file is opened we will set the corresponding pointer and clear the other one. But since 2 x 2 = 4, AppState has 4 possible states. The extra state that the program should never get into is when both pointers are non-null. It is impossible to open two files at the same time.
This example could be taken further by adding more variables to AppState. 10 variables with 2 states each will result in 2^{10} = 1024 possible states. It is very likely that most of these states are invalid and untested.

Wouldn’t it be nice if our types protected us from invalid states? Let’s give it a try, using an imaginary C-like language with tagged unions:

{% highlight c %}
union AppState {
  struct {
    VideoFile* file_;
    Volume* volume_;
    Subtitles* subtitles_;
  } video_;

  struct {
    AudioFile* file_;
    Volume* volume_;
    PlaybackSpeed* speed_;
  } audio_;

  void nothing_;
};
{% endhighlight %}

One could argue that I compared product types (struct) to sum types (union). Another argument against my case would be that we could get union-like behavior by using interfaces and virtual methods. The point I’m trying to make here is a practical one. Surely we could use a C union and manually keep track of its state, but C unions are avoided in most codebases I have seen for this exact reason. My language can represent the entire application state without resorting to overly engineered class hierarchies or unsafe unions. Can yours do that?

A perfect example to make this point is Redux-style state management. Since JavaScript is a dynamically typed language, we could have infinitely many unreachable states for any model. Elm, which is a language that inspired Redux, does have static types and tagged unions. Elm forces the programmer to think about almost all possible states the application can get into.

// elm example here

# Linear and affine types




TODO:
- how we ended up here
--non-critical software grew at an enormous rate after the 70's. it turns out even banking isn’t that critical.
--we preferred “informal verification” to mathematically proven formal verification. a trade-off: correctness vs practicality
- practical solution: software testing... proof by induction
-- throw test cases and hope for the best
-- relies on the number and quality of tests (critical testing: we must write tests that are least likely to pass)
- static typing:
-- linear/affine types
--- state machines in my types! makes it impossible for us to do bad operations (use-after-free, no file.write() after file.close())
-- dependent types (practical DT languages are in their infancy, but they’re starting to draw attention)
-- types depending on values
-- `List append(List list, int el)` vs `List<n+1> append(List<n> list, int el)`
-- CoC
-- very advanced, Coq is an example
-- write theorems and prove them!
-- okay, but what can we prove with these tools irl?
-- exceptions
-- state management: unreachable state
-- react/redux state management
-- Go return tuples
-- safety critical properties
-- guaranteed termination (at the cost of turing completeness)
-- memory allocation guarantees
-- bad cases in other fields:
-- sql databases, migration scripts
-- we should properly type schemas, version them, and typecheck version transitions
-- ad-hoc protocols (idls)
