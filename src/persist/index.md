---
title: "Object Persistence"
version: 1
abstract: >
    Some simple kinds of data can be saved as lines of text,
    but more complex data structures require a framework capable of handling aliasing and circularity.
    This chapter shows how to build such a framework,
    and in doing so demonstrates how libraries for handling JSON and other formats work.
syllabus:
-   A persistence framework saves and restores objects.
-   Persistence must handle aliasing and circularity.
-   Users should be able to extend persistence to handle objects of their own types.
-   Software designs should be open for extension but closed for modification.
depends:
-   interp
---

Version control can keep track of our files,
but what should we put in them?
Plain text works well for things like this chapter,
but the data structures used to represent HTML ([%x check %])
or the state of a game
aren't easy to represent in prose.

Another option is to store objects,
i.e.,
to save a list of dictionaries as-is
rather than flattering it into rows and columns.
Python's [`pickle`][py_pickle] module does this in a Python-specific way,
while the [`json`][py_json] module saves some kinds of data as text formatted like JavaScript objects.
As odd as it may seem,
this has become a cross-language standard.

The phrase "some kinds of data" is the most important part of the preceding paragraph.
Since programs can define new [%i "class" "classes" %],
a [%g persistence "persistence framework" %]
has to choose one of the following:

1.  Only handle built-in types,
    or even more strictly,
    only handle types that are common across many languages,
    so that data saved by Python can be read by JavaScript and vice versa.

1.  Provide a way for programs to convert from user-defined types
    to built-in types
    and then save those.
    This option is less restrictive than the first
    but can lead to some information being lost.
    For example,
    if instances of a program's `User` class are saved as dictionaries,
    the program that reads data may wind up with dictionaries instead of users.

1.  Save class definitions as well as objects' values
    so that when a program reads saved data
    it can reconstruct the classes
    and then create fully functional instances of them.
    This choice is the most powerful,
    but it is also the hardest to implement,
    particularly across languages.
    It is also the riskiest:
    if a program is running third-party code in order to restore objects,
    it has to trust that code not to do anything malicious.

This chapter starts by implementing the first option (built-in types only),
then extends it to handle objects that the data structure refers to in several places
(which JSON does not).
To keep parsing and testing simple,
our framework will store everything as text with one value per line;
we will look at non-text options in [%x binary %],
and at how to handle user-defined types in [%x bonus %].

## Built-in Types {: #persist-builtin}

The first thing we need to do is specify our data format.
We will store each [%g atomic_value "atomic value" %] on a line of its own
with a type name and a value separated by a colon:

[%inc format.txt %]

Since we are storing things as text,
we have to handle strings carefully:
for example,
we might need to save the string `"str:something"`
and later be able to tell that it *isn't* the string `"something"`.
We do this by splitting strings on newline characters
and saving the number of lines,
followed by the actual data:

[%inc multiline_input.txt %]
[%inc multiline_output.txt %]

The function `save` handles three of Python's built-in types to start with:

[%inc builtin.py mark=save omit=extras %]

The function that loads data starts by reading a single line,
stripping off the newline at the end
(which is added automatically by the `print` statement in `save`),
and then splitting the line on colons.
After checking that there are two fields,
it uses the type name in the first field
to decide how to handle the second:

[%inc builtin.py mark=load omit=extras %]

Saving a list is almost as easy:
we save the number of items in the list,
and then save each item
with a [%i "recursion" "recursive" %] call to `save`.
For example,
the list `[55, True, 2.71]` is saved as shown in [%f persist-lists %].
The code to do this is:

[%inc builtin.py mark=save_list %]

while to load a list,
we just read the specified number of items:
{: .continue}

[%inc builtin.py mark=load_list %]

[% figure
   slug="persist-lists"
   img="lists.svg"
   alt="Saving lists"
   caption="Saving nested data structures."
   cls="here"
%]

Notice that `save` and `load` don't need to know
what kinds of values are in the list.
Each recursive call advances the input or output stream
by precisely as many lines as it needs to.
As a result,
this approach should handle empty lists and nested lists without any extra work.

Our functions handle sets in exactly the same way as lists;
the only difference is using the keyword `set` instead of the keyword `list`
in the opening line.
To save a [%i "dictionary" %],
we save the number of entries
and then save each key and value in turn:

[%inc builtin.py mark=save_dict %]

The code to load a dictionary is analogous.
With this machinery in place,
we can save our first data structure:

[%inc save_builtin.py mark=save %]
[%inc save_builtin.out %]

We now need to write some unit tests.
We will use two tricks when doing this:

1.  The `StringIO` class from Python's [`io`][py_io] module
    allows us to read from strings and write to them
    using the functions we normally use to read and write files.
    Using this lets us run our tests
    without creating lots of little files as a side effect.

1.  The `dedent` function from Python's [`textwrap`][py_textwrap] module
    removes leading indentation from the body of a string.
    As the example below shows,
    `dedent` allows us to indent a [%i "fixture" %]
    the same way we indent our Python code,
    which makes the test easier to read.

[%inc test_builtin.py mark=test_save_list_flat %]

## Converting to Classes {: #persist-oop}

The `save` and `load` functions we built in the previous section work,
but as we were extending them
we had to modify their internals
every time we wanted to do something new.

The [%g open_closed_principle "Open-Closed Principle" %] states that
software should be open for extension but closed for modification,
i.e., that it should be possible to extend functionality
without having to rewrite existing code.
This allows old code to use new code,
but only if our design permits the kinds of extensions people are going to want to make.
(Even then,
it often leads to deep class hierarchies
that can be hard for the next programmer to understand.)
Since we can't anticipate everything,
it is normal to have to revise a design the first two or three times we try to extend it.
As [%b Brand1995 %] said of buildings,
the things we make learn how to do things better as we use them.

In this case,
we can follow the Open-Closed Principle by rewriting our functions as classes
and by using yet another form of [%i "dynamic dispatch" %] to handle each item
so that we don't have to modify a multi-way `if` statement
each time we add a new capability.
If we have an object `obj`,
then `hasattr(obj, "name")` tells us whether that object has an attribute called `"name"`.
If it does,
`getattr(obj, "name")` returns that attribute's value;
if that attribute happens to be a method,
we can then call it like a function:

[%inc attr.py %]
[%inc attr.out %]

Using this,
the core of our saving class is:

[%inc objects.py mark=save %]

We have called this class `SaveObjects` instead of just `Save`
because we are going to create other variations on it.
`SaveObjects.save` figures out which [%i "method" %] to call
to save a particular thing
by constructing a name based on the thing's type,
checking whether that method exists,
and then calling it.
As in the previous example of dynamic dispatch,
the methods that handle specific items
must all have the same [%i "signature" %]
so that they can be called interchangeably.
For example,
the methods that write integers and strings are:

[%inc objects.py mark=save_examples %]

`LoadObjects.load` combines dynamic dispatch with
the string handling of our original `load` function:

[%inc objects.py mark=load %]

The methods that load individual items are even simpler.
For example,
we load a floating-point number like this:

[%inc objects.py mark=load_float %]

## Aliasing {: #persist-aliasing}

Consider the two lines of code below,
which created the data structure shown in [%f persist-shared %].
If we save this structure and then reload it
using what we have built so far,
we will wind up with two copies of the list containing the string `"content"`
instead of one.
This won't be a problem if we only ever read the reloaded data,
but if we modify the new copy of `fixture[0]`,
we won't see that change reflected in `fixture[1]`,
where we *would* have seen the change in the original data structure:

[%inc ex_aliasing.py %]

[% figure
   slug="persist-shared"
   img="shared.svg"
   alt="Saving aliased data incorrectly"
   caption="Saving aliased data without respecting aliases."
   cls="here"
%]

The problem is that the list `shared` is [%i "alias" "aliased" %],
i.e.,
there are two or more references to it.
To reconstruct the original data correctly,
we need to:

1.  keep track of everything we have saved;

1.  save a marker instead of the object itself
    when we try to save it a second time;
    and

1.  reverse this process when loading data.

We can keep track of the things we have saved
using Python's built-in `id` function,
which returns a unique ID for every object in the program.
For example,
even if two lists contain exactly the same values,
`id` will report different IDs for those lists
because they're stored in different locations in memory.
We can use this to:

1.  store the IDs of all the objects we've already saved
    in a set, and then

1.  write a special entry with the keyword `alias`
    and its unique ID
    when we see an object for the second time.

Here's the start of `SaveAlias`:

[%inc aliasing_wrong.py mark=save %]

Its constructor creates an empty set of IDs seen so far.
If `SaveAlias.save` notices that the object it's about to save
has been saved before,
it writes a line like this:
{: .continue}

[%inc ex_alias_line.txt %]

where `12345678` is the object's ID.
(The exercises will ask why the trailing colon needs to be there.)
If the object hasn't been seen before,
`SaveAlias` saves the object's type,
its ID,
and either its value or its length:
{: .continue}

[%inc aliasing_wrong.py mark=save_list %]

`SaveAlias._list` is a little different from `SaveObjects._list`
because it has to save each object's identifier
along with its type and its value or length.
Our `LoadAlias` class needs a similar change compared to `LoadObjects`.
The first version is shown below;
as we will see,
it contains a subtle bug:

[%inc aliasing_wrong.py mark=load %]

The first test of our new code is:

[%inc test_aliasing_wrong.py mark=no_aliasing %]

which uses this [%i "helper function" %]:
{: .continue}

[%inc test_aliasing_wrong.py mark=roundtrip %]

There isn't any aliasing in the test case,
but that's deliberate:
we want to make sure we haven't broken code that was working
before we move on.
Here's a test that actually includes some aliasing:
{: .continue}

[%inc test_aliasing_wrong.py mark=shared %]

It checks that the aliased sub-list is actually aliased after the data is restored,
then checks that changes to the sub-list through one alias show up through the other.
The second check ought to be redundant,
but it's still comforting.
{: .continue}

There's one more case to check,
and unfortunately it reveals a bug.
The two lines:

[%inc ex_circular.py %]

create the data structure shown in [%f persist-circular %],
in which an object contains a reference to itself.
Our code ought to handle this case but doesn't:
when we try to read in the saved data,
`LoadAlias.load` sees the `alias` line
but then says it can't find the object being referred to.
{: .continue}

[% figure
   slug="persist-circular"
   img="circular.svg"
   alt="A circular data structure"
   caption="A data structure that contains a reference to itself."
%]

The problem is these lines in `LoadAlias.load`
marked as containing a bug,
in combination with these lines inherited from `LoadObjects`:
{: .continue}

[%inc objects.py mark=load_list %]

Let's trace execution for the saved data:
{: .continue}

[%inc ex_list_alias.txt %]

1.  The first line tells us that there's a list whose ID is `4484025600`
    so we `LoadObjects._list` to load a list of one element.

1.  `LoadObjects._list` called `LoadAlias.load` recursively to load that one element.

1.  `LoadAlias.load` reads the second line of saved data,
    which tells it to re-use the data whose ID is `4484025600`.
    But `LoadObjects._list` hasn't created that list yet—it
    is still reading the elements—so
    `LoadAlias.load` hasn't added the list to `seen`.

The solution is to reorder the operations,
which unfortunately means writing new versions
of all the methods defined in `LoadObjects`.
The new implementation of `_list` is:

[%inc aliasing.py mark=load_list %]

This method creates the list it's going to return,
adds that list to the `seen` dictionary immediately,
and *then* loads list items recursively.
We have to pass it the ID of the list
to use as the key in `seen`,
and we have to use a loop rather than
a [%g list_comprehension "list comprehension" %],
but the changes to `save_set` and `save_dict` follow exactly the same pattern.

[%inc save_aliasing.py mark=save %]
[%inc save_aliasing.out %]

## Summary {: #persist-summary}

[%f persist-concept-map %] summarizes the ideas introduced in this chapter,
while [%x bonus %] shows how to extend our framework
to handle user-defined classes.

[% figure
   slug="persist-concept-map"
   img="concept_map.svg"
   alt="Concept map for persistence"
   caption="Concepts for persistence."
   cls="here"
%]

## Exercises {: #persist-exercises}

### Dangling Colon {: .exercise}

Why is there a colon at the end of the line `alias:12345678:`
when we create an alias marker?

### Versioning {: .exercise}

We now have several versions of our data storage format.
Early versions of our code can't read the archives created by later ones,
and later ones can't read the archives created early on
(which used two fields per line rather than three).
This problem comes up all the time in long-lived libraries and applications,
and the usual solution is to include some sort of version marker
at the start of each archive
to indicate what version of the software created it
(and therefore how it should be read).
Modify the code we have written so far to do this.

### Strings {: .exercise}

Modify the framework so that strings are stored using escape characters like `\n`
instead of being split across several lines.

### Who Calculates? {: .exercise}

Why doesn't `LoadAlias.load` calculate object IDs?
Why does it use the IDs saved in the archive instead?

### Using Globals {: .exercise}

The lesson on unit testing introduced the function `globals`,
which can be used to look up everything defined at the top level of a program.

1.  Modify the persistence framework so that
    it looks for `save_` and `load_` functions using `globals`.

1.  Why is this a bad idea?
