---
title: "Protocols"
version: 1
abstract: >
    This chapter starts by showing how we can simplify testing
    by temporarily replacing real functions with ones that return predictable values,
    then uses our need to do that
    to motivate discussion of ways that programmers can hook their own code
    into the Python interpreter.
    Unlike other chapters in this book,
    this one focuses on Python itself rather than the things we can build with it.
syllabus:
-   Temporarily replacing functions with mock objects can simplify testing.
-   Mock objects can record their calls and/or return variable results.
-   Python defines protocols so that code can be triggered by keywords in the language.
-   Use the context manager protocol to ensure cleanup operations always execute.
-   Use decorators to wrap functions after defining them.
-   Use closures to create decorators that take extra parameters.
-   Use the iterator protocol to make objects work with for loops.
depends:
-   func
-   oop
---

This book is supposed to teach software design
by implementing small versions of real-world tools,
but we have reached a point where we need to learn a little more about Python itself
in order to proceed.
Our discussion of closures in [%x func %] was the first step;
in this chapter,
we will look at how Python allows users to tell it to do things at specific moments.

## Mock Objects {: #protocols-mock}

We have already seen that functions are objects referred to by variable names
just like other values.
We can use this fact to change functions at runtime to make testing easier.
For example,
if the function we want to test uses the time of day,
we can temporarily replace the real `time.time` function
with one that returns a specific value
so we know what result to expect in our test:

[%inc mock_time.py %]

Temporary replacements like this are called [%g mock_object "mock objects" %]
because we usually use objects even if the thing we're replacing is a function.
We can do this because Python lets us create objects
that can be "called" just like functions.
If an object `obj` has a `__call__` method,
then `obj(…)` is automatically turned into `obj.__call__(…)`
just as `a == b` is automatically turned into `a.__eq__(b)` ([%x parse %]).
For example,
the code below defines a class `Adder`
whose instances add a constant to their input:

[%inc callable.py %]
[%inc callable.out %]

Let's create a reusable mock object class that:

1.  defines a `__call__` method so that instances can be called like functions;

2.  declares the parameters of that method to be `*args` and `**kwargs`
    so that it can be called with any number of regular or keyword arguments;

3.  stores those [%i "argument" "arguments" %]
    so we can see how the replaced function was called;
    and

4.  returns either a fixed value or a value produced by a user-defined function.

The class itself is only 11 lines long:

[%inc mock_object.py mark=fake %]

For convenience,
let's also define a function that replaces some function we've already defined
with an instance of our `Fake` class:

[%inc mock_object.py mark=fakeit %]

To show how this works,
we define a function that adds two numbers
and write a test for it:

[%inc mock_object.py mark=test_real %]

<div class="pagebreak"></div>

We then use `fakeit` to replace the real `adder` function
with a mock object that always returns 99
([%f protocols-timeline %]):

[%inc mock_object.py mark=test_fixed %]

[% figure
   slug="protocols-timeline"
   img="timeline.svg"
   alt="Timeline of mock operation"
   caption="Timeline of mock operation."
%]

Another test proves that our `Fake` class records
all of the calls:

[%inc mock_object.py mark=test_record %]

And finally,
the user can provide a function to calculate a return value:

[%inc mock_object.py mark=test_calc %]

## Protocols {: #protocols-protocols}

Mock objects are very useful,
but the way we're using them is going to cause strange errors.
The problem is that
each test replaces `adder` with a mock object that does something different.
As a result,
any test that *doesn't* replace `adder` will use
whatever mock object was last put in place
rather than the original `adder` function.

We could tell users it's their job to put everything back after each test,
but people are forgetful.
It would be better if Python did this automatically;
luckily for us,
it provides a [%g protocol "protocol" %] for exactly this purpose.
A protocol is a rule that specifies how programs can tell Python
to do specific things at specific moments.
Giving a class a `__call__` method is an example of this:
when Python sees `thing(…)`,
it automatically checks if `thing` has that method.
Defining an `__init__` method for a class is another example:
if a class has a method with that name,
Python calls it automatically when constructing a new instance of that class.

What we want for managing mock objects is
a [%g context_manager "context manager" %]
that replaces the real function with our mock
at the start of a [%i "block" %] of code
and then puts the original back at the end.
The protocol for this relies on two methods called `__enter__` and `__exit__`.
If the class is called `C`,
then when Python executes a `with` block like this:

[%inc ex_with.py %]

it does the following ([%f protocols-context-manager %]):
{: .continue}

1.  Call `C`'s [%i "constructor" %]
    to create an object that it associates with the code block.
2.  Call that object's `__enter__` method
    and assign the result to the variable `name`.
3.  Run the code inside the `with` block.
4.  Call `name.__exit__()` when the block finishes.

[% figure
   slug="protocols-context-manager"
   img="context_manager.svg"
   alt="A context manager"
   caption="Operations performed by a context manager."
%]

Here's a mock object that inherits all the capabilities of `Fake`
and adds the two methods needed by `with`:

[%inc mock_context.py mark=contextfake %]

Notice that `__enter__` doesn't take any extra parameters:
anything it needs must be provided via the object's constructor.
On the other hand,
`__exit__` will always be called with three values
that tell it whether an [%i "exception" %] occurred,
and if so,
what the exception was.
This test shows that our context manager is doing what it's supposed to:

[%inc mock_context.py mark=test %]

Context managers can't prevent people from making mistakes,
but they make it easier for people to do the right thing.
They are also an example of how programming languages often evolve:
eventually,
if enough people are doing something the same way in enough places,
support for that way of doing things is added to the language.

## Decorators {: #protocols-decorator}

Python programs rely on several other protocols,
each of which gives user-level code a way to interact with
some aspect of the Python [%i "interpreter" %].
One of the most widely used is called a [%g decorator "decorator" %],
which allows us to wrap one function with another.

In order to understand how decorators work,
we must take another look at [%i "closure" "closures" %] ([%x func %]).
Suppose we want to create a function called `logging`
that prints a message before and after
each call to some other arbitrary function.
We could try to do it like this:

[%inc wrap_infinite.py %]

but when we try to call `original` we wind up in an infinite loop.
The wrapped version of our function refers to `original`,
but Python looks up the function associated with that name *at the time of call*,
which means it finds our wrapper function instead of the original function
([%f protocols-recursion %]).
We can prevent this [%g infinite_recursion "infinite recursion" %]
by creating a closure to [%i "variable_capture" "capture" %]
the original function for later use:
{: .continue}

<div class="pagebreak"></div>

[%inc wrap_capture.py %]
[%inc wrap_capture.out %]

[% figure
   slug="protocols-recursion"
   img="recursion.svg"
   alt="Infinite recursion with a wrapped function"
   caption="Infinite recursion caused by careless use of a wrapped function."
%]

Using a closure also gives us a way to pass extra arguments
when we create the wrapped function:

<div class="pagebreak"></div>

[%inc wrap_param.py %]
[%inc wrap_param.out %]

Wrapping functions like this is so useful
that Python has built-in support for doing it.
We define the decorator function that does the wrapping as before,
but then use `@wrap` to apply it
rather than `name = wrap(name)`:

[%inc decorator_simple.py %]
[%inc decorator_simple.out %]

If we want to pass arguments at the time we apply the decorator,
though,
it seems like we're stuck:
a Python decorator must take exactly one argument,
which must be the function we want to decorate.
The solution is to define a function inside a function *inside yet another function*
to create a closure that captures the arguments:

[%inc decorator_param.py %]
[%inc decorator_param.out %]

Decorators didn't need to be this complicated.
In order to define a method that takes \\( N \\) parameters in Python,
we have to write a function of \\( N+1 \\) parameters,
the first of which represents the object for which the method is being called.
Python could have done the same thing with decorators,
i.e.,
allowed people to define a function of \\( N+1 \\) parameters
and have `@` fill in the first automatically:

[%inc alternative_design.py %]

But this isn't the path Python took,
and as a result,
decorators are harder to learn and use than they could have been.
{: .continue}

## Iterators {: #protocols-iterator}

As a last example of how protocols work,
consider the `for` loop.
The statement `for thing in collection`
assigns items from `collection` to the variable `thing` one at a time.
Python implements this using a two-part [%g iterator "iterator" %] protocol,
which is a version of the [%g iterator_pattern "Iterator" %] [%i "design pattern" %]:

1.  If an object has an `__iter__` method,
    that method is called to create an iterator object.

2.  That iterator object must have a `__next__` method,
    which must return a value each time it is called.
    When there are no more values to return,
    it must [%i "raise" %] a `StopIteration` exception.

For example,
suppose we have a class that stores a list of strings
and we want to return the characters from the strings in order.
(We will use a class like this to store lines of text in [%x viewer %].)
In our first attempt,
each object is its own iterator,
i.e.,
each object keeps track of what value to return next when looping:

[%inc naive_iterator.py mark=body %]

If we think of the text in terms of rows and columns,
the `advance` method moves the column marker forward within the current row.
When we reach the end of a row,
we reset the column to 0 and advance the row index by one:

[%inc naive_iterator.py mark=advance %]

Our first test seems to work:

[%inc test_naive_iterator.py mark=success %]

However,
our iterator doesn't work if the buffer contains an empty string:
{: .continue}

[%inc test_naive_iterator.py mark=failure %]

It also fails when we use a nested loop:
{: .continue}

[%inc test_naive_iterator.py mark=nested %]

We can fix the first problem with more careful bookkeeping—we leave that
as an exercise—but
fixing the second problem requires us to re-think our design.
The problem is that we only have one pair of variables
(the `_row` and `_col` attributes of the buffer)
to store the current location,
but two loops trying to use them.
What we need to do instead is
create a separate object for each loop to use:

[%inc better_iterator.py mark=buffer %]

Each [%i "cursor" %] keeps track of the current location
for a single loop
using code identical to what we've already seen
(including the same bug with empty strings):

[%inc better_iterator.py mark=cursor omit=advance %]

With this change in place,
our test of nested loops passes.
{: .continue}

## Summary {: #protocols-summary}

[%f protocols-concept-map %] summarizes
the ideas and tools introduced in this chapter.

[% figure
   slug="protocols-concept-map"
   img="concept_map.svg"
   alt="Concept map of mocks, protocols, and iterators"
   caption="Concept map."
   cls="here"
%]

## Exercises {: #protocols-exercises}

### Testing Exceptions {: .exercise}

Create a context manager that works like `pytest.raises` from the [`pytest`][pytest] module,
i.e.,
that does nothing if an expected exception is raised within its scope
but fails with an assertion error if that kind of exception is *not* raised.

### Timing Blocks {: .exercise}

Create a context manager called `Timer` that reports how long it has been
since a block of code started running:

[%inc ex_timer.py %]

### Handling Empty Strings {: .exercise}

Modify the iterator example so that it handles empty strings correctly,
i.e.,
so that iterating over the list `["a", ""]` produces `["a"]`.

### An Even Better Cursor {: .exercise}

Rewrite the `BetterCursor` class
so that it initializes `self._row` to 0 and `self._col` to \\( -1 \\)
and always calls `self._advance()` as the first action in `self.__next__`.
(You will need to make a few other changes as well.)
Do you think this implementation is simpler than the one presented in this chapter?

### Logging to a File {: .exercise}

Create a decorator that takes the name of a file as an extra parameter
and appends a log message to that file
each time a function is called.
(Hint: open the file in [%g append_mode "append mode" %]
each time it is needed.)
