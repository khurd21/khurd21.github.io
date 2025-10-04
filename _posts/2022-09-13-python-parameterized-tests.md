---
title: Parameterized Unit Tests in Python
last_modified_at: 2022-09-13T12:00:00-05:00
---

In a previous blog post, I covered how to write parameterized tests
in C#. To me, C# test cases are easier to write than Python test cases.
I think this is due to the fact that C# has a lot of test features built
into the language. In the case of parameterized tests, Python needs to
use a third-party library to achieve the same result. The two popular
libraries to achieve this right now are
<a href="https://pypi.org/project/pytest/"
  target="_blank">pytest</a>
and
<a href="https://pypi.org/project/parameterized"
  target="_blank">parameterized</a>.
For this post, I will be using the parameterized library in conjunction
with the unittest library already built into Python.

I searched for a while to find an equivalent to the C#
<a href="https://docs.nunit.org/articles/nunit/writing-tests/TestCaseData.html"
  target="_blank">TestCaseData</a>
class for Python, but could not find one. This would honestly be a
great addition to the parameterized library, as it gave me a different
perspective on how I can approach writing tests. It also seems like a great
way to separate data from the test cases themselves.

# Python Parameterized Tests

For this example, I made a quick Python class that contains functions to determine
if a string is title and to make a string a title if it is not already. The
implementation is not important for this post, and I will provide a few
examples of what some expected results should be.

```python
# validator.py

class TitleExpressionValidator:

    _excluded_words_to_capitalize = ['of', 'the', 'a', 'an']

    def is_valid(self, title_expression: str):
        if title_expression is None or len(title_expression) == 0:
            return False

        if title_expression[0].islower():
            return False

        all_words = title_expression.split()[1:]
        potential_uppercase_words = list(filter(lambda word: len(word) > 3, all_words))
        potential_lowercase_words = list(filter(lambda word: len(word) <= 3 and word in self._excluded_words_to_capitalize, all_words))
        return all(map(lambda word: word[0].isupper(), potential_uppercase_words)) and \
            all(map(lambda word: word[0].islower(), potential_lowercase_words))

    def make_title(self, title_expression: str):
        if title_expression is None or len(title_expression) == 0:
            raise ValueError('title_expression cannot be None or empty')

        words = title_expression.split()
        first_word = words[0].capitalize()
        rest_words = words[1:]
        return f'{first_word} {" ".join(map(lambda word: word.capitalize() if word not in self._excluded_words_to_capitalize else word.lower(), rest_words))}'
```

## Expected Results

| Input | is_valid output | make_title output |
| --- | --- | --- |
| `The Lord of the Rings` | `True` | `The Lord of the Rings` |
| `The Little Drummer Boy` | `True` | `The Little Drummer Boy` |
| `Hello, World!` | `True` | `Hello, World!` |
| `The lord of the rings` | `False` | `The Lord of the Rings` |
| `the Little Drummer Boy` | `False` | `The Little Drummer Boy` |
| `The Little drummer Boy` | `False` | `The Little Drummer Boy` |

## Set Up the Project

In a directory for this project, all you'll need is two python script files.
The `validator.py` script is the same script that was displayed in the
[introduction](#python-parameterized-tests) section of this blog post.

```
|- validator.py
|- validator_test.py
|- venv/
|   |- **
```

Additionally, since we are going to install a 3rd party library, let's set up
a virtual environment.

```shell
python3 -m venv venv
source venv/bin/activate
pip3 install parameterized
pip3 freeze > requirements.txt
```

## Writing the Test Cases

For the first section, I am going to write tests that focus on the `is_valid`
function. The trivial approach would be to write a test for each of the expected
results. Since there are two outputs possible for any given input, this would
mean we could write two types of tests, one for `True` and the other for `False`.
The impact of this approach is that we would have to write 12 tests, which is
not scalable when the size of tests grows. Here is an example:

### Appraoch 1: Write a Test for Each Expected Result

```python
# validator_test.py

import validator
import unittest
import parameterized

class ValidatorTest(unittest.TestCase):

    def setUp(self):
        self.validator = validator.TitleExpressionValidator()
        return

    def test_is_valid_returns_true(self):
        self.assertTrue(self.validator.is_valid("The Lord of the Rings"))
        return

    def test_is_valid_returns_false(self):
        self.assertFalse(self.validator.is_valid("The lord of the rings"))
        return

    # ...
```

### Approach 2: Write a Parameterized Test

The first major improvement we can make to this implementation is to introduce the
`parameterized` library and use it reduce the number of test functions we need to
implement down to two. We can parameterize the test cases by using the
`parameterized.expand` decorator.

```python
class ValidatorTest(unittest.TestCase):

    def setUp(self):
        self.validator = validator.TitleExpressionValidator()
        return

    @parameterized.parameterized.expand([
        'The Lord of the Rings',
        'The Little Drummer Boy',
        'Hello, World!'
    ])
    def test_is_valid_returns_true(self, expression):
        self.assertTrue(self.validator.is_valid(expression))

    @parameterized.parameterized.expand([
        'The lord of the rings',
        'the Little Drummer Boy',
        'hello, World!'
    ])
    def test_is_valid_returns_false(self, expression):
        self.assertFalse(self.validator.is_valid(expression))
        return
```

### Approach 3: Combining Tests per Function

This approach is a lot better. We can now add or remove test cases by simply
adding or removing them from the list. However, I am still not satisfied with
this approach. The problem with this approach is that we are separating the
test case sources based on output. For boolean functions, this is not a big
deal, but for functions that return other types, say an integer or string,
how would we separate the test cases based on the output? Additionally,
since the test case sources are separated, it could be easy to overlook or mistake
a test case for the wrong test function. My solution to this problem is to
combine the two test functions into one and add a second parameter to the
`parameterized.expand` decorator. This additional parameter can represent
the expected output of the test case.

```python
class ValidatorTest(unittest.TestCase):

    def setUp(self):
        self.validator = validator.TitleExpressionValidator()
        return

    @parameterized.parameterized.expand([
        ('The Lord of the Rings', True),
        ('The Little Drummer Boy', True),
        ('Hello, World!', True),
        ('The lord of the rings', False),
        ('the Little Drummer Boy', False),
        ('The Little drummer Boy', False),
    ])
    def test_is_valid(self, expression, expected):
        self.assertEqual(self.validator.is_valid(expression), expected)
        return
```

### Approach 4: Separating Test Sources from Test Cases

[Approach 3](#approach-3-combining-tests-per-function) to me is an ideal solution.
However, if the test case sources become too large, it will be difficult to
read and maintain the test suite. In situations like this, it would be nice
to remove the test case sources from the test function and store them in a
separate file. This approach looks very similar to the approach I took in my
previous blog post on parameterized testing in C#. In this implementation,
we can simply take the list of test sources and store them in a generator function.

```python
def _validator_responses():
    '''yield (<input>, <is_valid expected>, <make_title expected>)'''

    yield ('The Lord of the Rings', True, 'The Lord of the Rings')
    yield ('The Little Drummer Boy', True, 'The Little Drummer Boy')
    yield ('Hello, World!', True, 'Hello, World!')
    yield ('The lord of the rings', False, 'The Lord of the Rings')
    yield ('the Little Drummer Boy', False, 'The Little Drummer Boy')
    yield ('The Little dummer Boy', False, 'The Little Dummer Boy')
```

We can then use this generator to test two functions within `TitleExpressionValidator`.
Separating the test data from the test implementation to me makes it much more
readable.

```python
class ValidatorTest(unittest.TestCase):

    def setUp(self):
        self.validator = validator.TitleExpressionValidator()
        return

    @parameterized.parameterized.expand(_validator_responses)
    def test_is_valid(self, expression, expected, _):
        self.assertEqual(self.validator.is_valid(expression), expected)
        return

    @parameterized.parameterized.expand(_validator_responses)
    def test_make_title(self, expression, _, make_title):
        self.assertEqual(self.validator.make_title(expression), make_title)
        return
```

### Approach 5: Parameterizing at the Class Level

I added this approach as it was unique. I have not seen this available
in any other testing framework or library. The decorator `parameterized_class`
can be used to parameterize the test cases at the class level. In this decorator,
you can specify the names of the field names that will be used to represent
the test case sources. The generator function will then assign the values to
the specified field names to be used in the class. This is a very powerful feature
that we can use to take advantage of the fact that an `expression` can be passed into
both `is_valid` and `make_title` and produce a valid result.

```python
@parameterized.parameterized_class(('expression', 'is_valid', 'make_title'), _validator_responses())
class ValidatorTest(unittest.TestCase):

    def setUp(self):
        self.validator = validator.TitleExpressionValidator()
        return

    def test_is_valid(self):
        self.assertEqual(self.validator.is_valid(self.expression), self.is_valid)
        return

    def test_make_title(self):
        self.assertEqual(self.validator.make_title(self.expression), self.make_title)
```
