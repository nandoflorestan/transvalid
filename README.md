Ever dream of a validation library that would work the same in the server as
in the client?

**transvalid** is a Python validation library that works
in CPython and also in the browser via transpilation to JS through
[Transcrypt](https://www.transcrypt.org/).

(Transcrypt requires code to rely on the Python stdlib as little as possible.
I tried converting Cerberus, Colander and schema, but all of these proved
too hard. For instance, the schema library wouldn't work with Transcrypt
because JS can only have strings as dict keys, but schema wants
`Optional("email")` (a Python type) to be a key.)

In the end, [is_valid](https://github.com/daanvdk/is_valid) was
the only library that worked with Transcrypt.

So **transvalid** is a fork of **is_valid**, a simple lightweight
Python library for validation predicates.

Predicates are functions that given a certain input return either `True` or
`False` according to some rules that the input either follows or does not
follow.

## Example
Suppose we have the following data about some books.
```python
>>> books = [
...     {
...         'title': 'A Game of Thrones',
...         'author': 'George R. R. Martin',
...         'pubdate': datetime.date(1996, 8, 1),
...         'ISBN': '0553103547'
...     },
...     {
...         'title': 'Harry Potter and the Philosopher\'s Stone',
...         'author': 'J. K. Rowling',
...         'pubdate': '1997-6-26',
...         'ISBN': '0747532699'
...     },
...     {
...         'title': 'The Fellowship of the Ring',
...         'author': 'J. R. R. Tolkien',
...         'pubdate': datetime.date(1954, 7, 29),
...         'ISBN': '0-618-57494-8'
...     }
... ]
```
We want to validate this data according to a rather strict specification.
To do this we are going to write a predicate that validates our data.
```python
>>> from transvalid import is_list_of, is_dict_where, is_str, is_date, is_match
>>> is_list_of_books = is_list_of(
...     is_dict_where(
...         title=is_str,
...         author=is_str,
...         pubdate=is_date,
...         ISBN=is_match(r'^\d{9}(\d|X)$')
...     )
... )
```
You can see that using only a few simple predicates you can easily create a
predicate that evaluates complex structures of data. Now lets analyze our data!
```python
>>> is_list_of_books(books)
False
```
Okay, so now we know that our data isn't valid according to the specification.
But what if we now want to fix this. In this case just `True` or `False`
doesn't cut it, you might want to know why your data is valid or not. For this
purpose all predicates include an `explain` method. If you call this method the
predicate will return a special `Explanation`-object as result. All
`Explanation` objects have a `valid`, `code`, and `message` attribute.
```python
>>> explanation = is_list_of_books.explain(books)
>>> explanation.valid
False
>>> explanation.message
'not all elements are valid according to the predicate'
```
So in our case the explanation message is still not very useful. Luckily
`Explanation`-objects often have a `details` attribute as well that contains
more specific information. In the case of predicates generated by `is_list_of`
this is a mapping from indexes in the list that weren't valid to an
`Explanation`-object explaining why they weren't valid. So lets use this to
get some more insight into our data.
```python
>>> for i, subexplanation in explanation.details.items():
...     print(i, subexplanation.message)
```
```
1 data is not a dict where all the given predicates hold
2 data is not a dict where all the given predicates hold
```
Still not specific enough, however as you might have guessed already, this
subexplanation also has a details attribute. For predicates generated by
`is_dict_where` this is a mapping of keys to an `Explanation`-object explaining
why the value associated with that key was not valid (if the dictionary has
the right keys). So lets have a look.
```python
>>> for i, subexplanation in explanation.details.items():
...     for key, subsubexplanation in subexplanation.details.items():
...         print(i, key, subsubexplanation.message)
```
```
1 pubdate data is not a date
2 ISBN data does not match /^\d{9}(\d|X)$/
```
So now we've found the issues! Let's try fixing them and evaluating again.
```python
>>> books[1]['pubdate'] = datetime.date(1997, 6, 26)
>>> books[2]['ISBN'] = '0618574948'
>>> is_list_of_books(books)
True
```

So you might think this way of finding all the errors was a bit cumbersome.
This is because the API shown above is optimized to handling the result
programmatically. If we want human readable errors we can just get the summary.
```python
>>> explanation.summary()
```
```
Data is not valid.
[1:'pubdate'] '1997-6-26' is not a date
[2:'ISBN'] '0-618-57494-8' does not match /^\d{9}(\d|X)$/
```
Which would've made it instantly clear to us what to do.

## Installation

**transvalid** shall soon be on
[PyPI](https://pypi.python.org/pypi/transvalid); then you can install it with:
```
pip install transvalid
```

## Building transvalid.py

We have kept the is_valid/ source (slightly modified to appease Transcrypt)
in this project, so we can still merge any upstream improvements.

Our default branch is called `transcrypt`, leaving `master`for upstream.

But for web deployment a single large file is much better than the ~70 small
modules in that directory, so we concatenate these in `concat.py`, generating
`transvalid.py`.

**transvalid** depends on the json module which for the moment exists only in
my fork of Transcrypt:

    pip install -e git://github.com/nandoflorestan/transcrypt.git@json#egg=transcrypt

During transvalid development, build the transvalid.py module with:

    ./concat.py && transcrypt transvalid.py

To use transvalid in your separate application, install it in your virtualenv:

    pip install -e .

...and then just `import transvalid`.
