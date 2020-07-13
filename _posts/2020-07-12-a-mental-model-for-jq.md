---
title: A Mental Model for jq
description: Describes a way to understand how jq works in order to write queries more effectively
---

[jq](https://github.com/stedolan/jq) is an incredibly useful
command-line tool for parsing and processing json.  I've used it in
various shell scripts to grab data from different APIs.  However, the
syntax has always been a bit elusive to me, and since I don't write `jq`
scripts every day, I end up always having to resort back to the [jq
manual](https://stedolan.github.io/jq/manual/) as well as StackOverflow
to find out how to do slightly more complex tasks.

From my past days as a DBA, I've always approached `jq` as a type of query
language like SQL.  When skimming through the `jq` manual, I would get
confused using this query-like interpretation.  I finally got around to
spending more time reading through the `jq` manual, and I'm posting
these notes here for my future reference.  Note that this is purely a
mental model of how `jq` works - I have not read the source code and
the Python examples in this post are very unlikely how `jq` actually
works.

Instead of thinking of `jq` as a query language to select and identify
bits and pieces of JSON, think of it as a bunch of for-loops chained
together one after another, where each for-loop iterates over each item
in the current data in the chain.  This is similar in principle to
chaining together a series of `map()` and `filter()` in JavaScript.
This idea is well-described as [Sequences as Conventional
Interfaces](https://mitpress.mit.edu/sites/default/files/sicp/full-text/book/book-Z-H-15.html#%_sec_2.2.3)
in Structure and Interpretation of Computer Programs.

Loosely paraphrasing the first few paragraphs of the `jq` manual, a `jq`
script is made of "filters".  A filter is simply a function that takes
an input, and produces an output.  The `jq` script is then applied to
streams of JSON data.  What does this all mean?  Let's start with an
example which uses the [`.` (identity)
filter](https://stedolan.github.io/jq/manual/#Identity:.) - it takes
an input, and returns the input as its output.

`jq` can accept JSON data either through standard input, or from a file.
In all the following examples, we pass data into `jq` through standard
input (through a Unix pipe) to easily show the JSON data, but it is more
common to either pipe in the results from a curl command or specify a
JSON file to read the data in from.

```sh
$ echo '{"foo": "bar", "foo2": "baz"}' | jq '.'
{
  "foo": "bar",
  "foo2": "baz"
}
```

Since filters are functions, what does this `.` (identity) filter look
like?  In Python, this might look like the following:


```python
def identity(s):
  return s

identity({"foo": "bar", "foo2": "baz"})
```

When processing the input JSON data, `jq` does some preliminary parsing - 
it parses the input as a sequence of whitespace-separated JSON values.
Each of these values are then passed to the filter.  To clarify and
illustrate this point, we pass a sequence of three pieces of JSON data
to `jq` - an object, an array, and a number, all separated by
whitespace.  I'm using spaces in the example, but it is more typical to
see newlines instead of spaces.

```sh
$ echo '{"foo": "bar"} [1,2,3] 42' | jq '.'
{
  "foo": "bar"
}
[
  1,
  2,
  3
]
42
```

You can see that `jq` passes each item through the `.` (identity) filter, and
the results are separated by newlines.  Continuing with our Python
example, this would look like:

```python
identity({"foo": "bar"})
identity([1,2,3])
identity(42)
```

We now turn to the [object identifier filter (e.g.
`.foo`)](https://stedolan.github.io/jq/manual/#ObjectIdentifier-Index:.foo,.foo.bar) -
we now can see this as a function that takes the JSON data and a key
name as an input, then returns the value of the key.  For example:

```sh
$ echo '{"name": "Ed"}' | jq '.name'
"Ed"
```

would look like the following in Python:

```python
def object_identifier(input, key):
  return input[key]

object_identifier({"name": "Ed"}, "name")
```

Next, let's try to see how the [`.[]` (array/object value iterator)
filter](https://stedolan.github.io/jq/manual/#Array/ObjectValueIterator:.[])
works.  Suppose we had a JSON array of names, and wanted to list all the
names.  If we just use the `.` (identity) filter, we will get back a
single JSON array.

```sh
$ echo '[{"name": "Ed"}, {"name": "baz"}]' | jq '.'
[
  {
    "name": "Ed"
  },
  {
    "name": "baz"
  }
]
```

We can't do anything with the result of the previous filter, because the
output is still a single JSON data value (i.e. an array).  We need to
split the array up into a sequence of items, so that we can run a filter
on each item.  Recall that `jq` accepts whitespace-separated JSON
values.  This is where the `.[]` filter comes into play:

```sh
$ echo '[{"name": "Ed"}, {"name": "baz"}]' | jq '.[]'
{
  "name": "Ed"
}
{
  "name": "baz"
}
```

Success!  The `.[]` filter took an array and returned each item
separated by newlines.  Again, I've tried to provide a rough Python
translation below - the main point is that the filter returns multiple
values.  The example uses a tuple to return multiple values, but a list
could have been used instead.  Again, this translation is just meant for
illustrative purposes.

```python
def array_iterator(array):
  return tuple(array)

item1, item2 = array_iterator([{"name": "Ed"}, {"name": "baz"}])

print(item1, item2)
# {'name': 'Ed'} {'name': 'baz'}
```

Now that we have each item in the array being returned individually, how
do we use the object_identifier (e.g. `.foo`) filter to get the `name`
values of each item?  We use the [`|` (pipe)
operator](https://stedolan.github.io/jq/manual/#Pipe:|).  This behaves
similarly to Unix pipes - it takes the output from the previous filter,
and passes it as the input to the next filter.  Thus, we have:

```sh
$ echo '[{"name": "Ed"}, {"name": "baz"}]' | jq '.[] | .name'
"Ed"
"baz"
```

In Python:

```python
output = array_iterator([{"name": "Ed"}, {"name": "baz"}])
result = []
for item in output:
  result.append(object_identifier(item, "name"))

print(result)
['Ed', 'baz']
```

`jq` does have a few idiosyncrasies with its syntax - some filters have
short forms.  For example, the previous script could have also been
written as `jq '.[].name'` - the pipe is implied.  Other filters have
different meanings depending on the JSON data type that is being
processed (e.g. in OOP parlance, some filters are overloaded functions).
For example, the `.[]` filter behaves slightly differently when its
input is an object instead of an array - it returns the values of the
object:

```sh
echo '{"name": "Ed", "city": "Toronto"}' | jq '.[]'
"Ed"
"Toronto"
```

With a bit of practice and skimming through the `jq` manual for other
useful built-in filters, `jq` can become a useful tool for
quick-and-dirty queries for exploring datasets or APIs.  You can even
write your own filters, and think of `jq` as a programming language in
itself - see this great talk by Charles Chamberlain: [Serious Programming
with jq?! A Practical and Purely Functional Programming
Language!](https://www.youtube.com/watch?v=PS_9pyIASvQ).


Do you have any comments, questions, and/or suggestions?  If so, feel
free to post a [GitHub issue on my site's
repo](https://github.com/edtan/edtan.github.io).
