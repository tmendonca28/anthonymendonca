---
layout: default
title: "Secure coding practices in Java"
permalink: /tech-learning/sec-coding-java/
description: "A few security best practices when writing Java code."
---
<h1>{{ page.title }}</h1>
<p class="subtitle">July 2022</p>

{% include reading-time.html %}

## Intro
Defensive or secure coding is mainly done so as to improve source code in terms of:
* General Quality - Reducing the number of software bugs and inherent problems. SOLID, Design Patterns, tested.
* Readability - Making it more readable for approval in a code audit situation. Naming, DRY, well-formatted and well-documented.
* Predictability - Making sure it behaves in a predictable manner without any unexpected actions

The main point of defensive coding is guarding against errors you don't expect.
> **Expecting the unexpected**

We should therefore shift 'Security Left' as bugs become more costly and hard to find later on i.e. in Production. It is very important to know, the sooner we find a bug, the better.

## Validating User Input
Errors should be detected as soon as possible. [Joshua Bloch, Effective Java](https://www.amazon.co.uk/Effective-Java-Joshua-Bloch/dp/0134685997)  
Guard clauses should be used and can do 1 of 3 things:
* Return early
* Fail fast i.e. throw an exception
* Alternative execution i.e. displaying a user-friendly message of what went wrong

### Validating Null
It is very important not to return null. Try introducing null checks and throw exceptions at the top and then the handling of proper logic after this. This assists tremendously with readability. Also, try aggregating exceptions into one generic one and throw this to the user. This helps in reducing code size and the number of times the user needs to correct their input.  
Another important point is to always place guard clauses at the very beginning. The usual logic then comes after this and can prove to be a huge time-saver.  
In terms of code, this should be our aim and at the very top of our functions:
```java
    if (not ok) { throw Exception }  // rest of code
```

### Validating Number Ranges
For number ranges, we need to be careful around the logic operators used for the border values. Aim for:
```java
    if ( check1 || check2) {// rest of code}
```

### Validating Strings
A technique called Decompose Conditional should be used. This means factoring out the conditional to a function like isValidString() or is ValidEmail().  
Use regular expressions as well for fields like Email Address but use them sparingly as they can get super complex.

### Validating Dates
It is not advisable to use the java.util.Data API anymore.  
As of Java 8 we can and should use the java.time API with LocalData, LocalTime, Instant etc.  
It is also advised not to store dates as Strings but instead as DateTime objects.  
Don't use regex to validate String dates but instead use native.parse() method.

### General best practice for what exception to throw
It is not a good idea to throw generic top-level errors, exceptions or throwables; instead throw specific exceptions e.g. IllegalArgumentException, IllegalStateException etc.
```java
    double calc(double input) {
	    return 13 / input;  // Arithmetic Exception if input is 0
    }
```

## Using Frameworks for Validation
We can leverage a few Frameworks to assist with validation.

### Native Java API
Objects class can be used for validation. A few of the methods are equals(), deepEquals(), requireNonNullElseGet(), checkIndex() etc.

### Google Guava
First, add Guava as a dependency.  
The class Preconditions has a number of static validation methods. checkNotNull() is synonymous to the requireNonNull() in the Native Java API. Other examples include checkArgument() and checkState().  
Note: It is advisable not to use both Objects and Preconditions API from Guava. Stick to one.

### Apache Commons
The Apache Commons Library also has the Validate class that has the following methods similar to Guava above : notNull(), isTrue(), and validState(). An advantage of the Commons class is that it has additional validations available such as notEmpty(), noNullElements(), exclusiveBetween() and inclusiveBetween().

## Improving Return Values
Magic Numbers such as -1 or 0 should generally be avoided as return values. These tend to force engineers to learn their meaning first (by reading through the application's documentation) and can further result in poor client-side code.  
A few valid return values in place of magic numbers could be:
* True/False : for success and failure respectively
* Void/Throw : nothing happens when successful, throw an exception if any failure occurs.
Try not to mix boolean returns with exceptions and null values.

### Don't return null
Instead we should throw an exception, return a sensible default value, empty collection or Optional<T>.  
An Optional can be described as a container that can hold at most one value. It is a container object that may or may not contain a non-null value. This can be used to avoid NullPointerExceptions and forces the user to confront the fact that there may be no value.
```java
    Optional<Journey> getRoute(String start)
```

## Other Defensive Practices
Encapsulation refers to information and implementation hiding. Strive to make each member as inaccessible as possible. It isn't advisable to provide getters and setters for the fun of it.  
Be careful with method side effects; this happens if a method changes some state outside of its local scope. Strive to not make methods both return values and produce side effects.  
In terms of exception handling we should:
* Use Java-7 try-with-resources
* Pass useful and specific information to your exceptions

Use Static Analysis Tools as they help in preventing bugs within code e.g. SonarLint.