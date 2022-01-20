Homework 1: Simple Syntactic Analysis With CodeQL
=================================================

### Assignment Objectives:
- Get set up and run CodeQL queries on a public GitHub project.
- Implement simple analyses with CodeQL queries.

### Handin Instructions:
- Submit via [Canvas](https://canvas.cmu.edu/courses/27636/quizzes/77722).

### Grading:
This homework is worth 100 points (with a possible 30 bonus points). The point
distribution is broken down inline with this document.

### Setup:

1. Log in to https://lgtm.com
2. Once logged in, navigate to "My Projects". Add the repositories
   https://github.com/CMU-program-analysis/s2-hw1 and
   https://github.com/CMU-program-analysis/mypy-hw1 as projects. **It is
   important that you use the CMU Program Analysis versions of these
   repositories. Your answers may be wrong if you run queries against different
   versions.**
3. Click the project, then click the "Query this project" button to open the
   Query Console.

---

Problem 1: Java Shift analysis (25 points):
-------------------------------------------

In Java, the shift operators (`<<`, `>>`, and `>>>`) shift the bits in a value
a certain distance. [According to the Java
specification](https://docs.oracle.com/javase/specs/jls/se7/html/jls-15.html#jls-15.19),
if the type of the left-hand argument is a primitive `int`, only the lower five
bits of the right-hand argument are used as the shift distance. In other words,
the shift distance actually used is always in the range `0` to `31`,
inclusive. Thus, a programmer trying to use a shift distance outside of this
range suggests that they misunderstand of the behavior of the shift operators.

Copy the code below into the Query Console. Complete the code by replacing `...`
with the proper code to identify all bit shift operations (`<<`, `>>`, and
`>>>`) that shift by a constant that is greater than 31 or less than 0.

In addition to the Console autocomplete, the following will be helpful
references:\
[Basic Java query
example](https://lgtm.com/help/lgtm/console/ql-java-basic-example)\
[Java AST
Reference](https://codeql.github.com/docs/codeql-language-guides/abstract-syntax-tree-classes-for-working-with-java-programs)\
[CodeQL Library for Java](https://codeql.github.com/codeql-standard-libraries/java)

For a more complete reference, see the [QL Language
Reference](https://codeql.github.com/docs/ql-language-reference)

---

After you run your final query, use the "Share" button to get a link to the
query and put it here **(5 points)**:

**You should update the code in this document and commit it to your
repository.** Although the "Share" link should contain your code we have no
guarantees that the link will persist. Saving your code here allows us to run it
again if needed.

```
import java

// Looks up all binary expressions and compile time constant expressions
from BinaryExpr expr, CompileTimeConstantExpr const

// Your code here
// --------------------
where
    // Constrain "expr" to shift operands (5 points)
    (expr instanceof ...) and

    // Constrain "const" to be the right operand of "expr" (5 points)
    // Note that "=" is the comparison operator
    ... and

    // Constrain the value of "const" to be negative or greater than 31 (5 points)
    (... or ...)

    // Running this query after implementing the three above constraints should
    // give you 19 results, however not all of these are errors because longs
    // can be shifted by more than 31 bits. Add one more predicate constraining
    // the left hand operand to an "int". This should leave you with one result.
    // (5 points)
// --------------------

// Shows a link to the expression and the value of "const"
// You can click the result in the "expr" column to navigate to the matched line
// of code.
select expr, const.getIntValue()
```

---

Problem 2: Python `v = v or default_value` (75 points)
------------------------------------------------------

Default arguments to Python are evaluated once when the function is
defined. This leads to a common error when the default arguments are mutable
(credit for this example goes to [The Hitchhiker's Guide to
Python](https://docs.python-guide.org/writing/gotchas/#mutable-default-arguments))

```
def append_to(element, to=[]):
    to.append(element)
    return to

my_list = append_to(12)
print(my_list)
# Prints "[12]"

my_other_list = append_to(42)
print(my_other_list)
# Prints "[12, 42]" instead of "[42]"
```

The standard solution to this is to instead use `None` as a default value and
initialize to a new copy of a mutable default value inside of the function:

```
def append_to(element, to=None):
    if to is None:
        to = []
    to.append(element)
    return to
```

A clever alternative to this pattern takes advantage of the combination of
short-circuit evaluation and the fact that `None` is "falsy" to make the code
more concise:

```
def append_to(element, to=None):
    to = to or []
    to.append(element)
    return to
```

In this example, if `to` is a "falsy" value like `None` it is initialized to an
empty list, otherwise its value is kept. However, this code is subtly different
from the explicit comparison to `None`. If `to` is passed a "falsy" value that
does not implement `append()` (e.g., `0`, `False`), the former code will fail
with a `AttributeError`, while the latter will happily assign a default value
and continue, which can lead to difficult-to-find bugs.

In this problem, you will implement an analysis to find instances of this
pattern in the [MyPy](https://github.com/python/mypy) project. The check you
will implement has two parts:
1. Identify parameters of functions that have a default value of `None`.
2. For those parameters, identify statements inside of the containing function
   of the form `x = x or ...`.

---

The [QL Language Reference](https://codeql.github.com/docs/ql-language-reference/)
will be helpful for this problem, particularly the sections on
[predicates](https://codeql.github.com/docs/ql-language-reference/predicates/) and
[`exists`](https://codeql.github.com/docs/ql-language-reference/formulas/#exists).

The starter code is below. You should run this query on the
[mypy-hw1](https://github.com/CMU-program-analysis/mypy-hw1) repository; if your
implementation is correct you should have 27 results.

---

After you run your final query, use the "Share" button to get a link to the
query and results and put it here **(5 points)**:

**You should update the code in this document and commit it to your
repository.** Although the "Share" link should contain your code we have no
guarantees that the link will persist. Saving your code here allows us to run it
again if needed.

```
import python

// This predicate checks that a local variable is a parameter of a function.
predicate function_has_parameter(Function f, LocalVariable v) {
  v.isParameter() and v.getScope() = f
}

// Your code here
// --------------------

// Write a predicate that checks that a variable is a parameter of a function
// and has a default value of None. (15 points)
predicate parameter_default_none(Function f, LocalVariable v) {
  function_has_parameter(f, v) and
  exists(Parameter p |
    ...
  )
}

// Write a predicate that checks if a function contains a statement of the form
// `v = v or ...` and return it.
// (see the "Predicates with result" section of the reference)
//
// This predicate must:
// - Check that v is a parameter of f (5 points).
// - Identify a statement inside the function that assigns v a value (10 points).
// - Check that the value is an Or expression (15 points).
// - Check that the first operand of the Or expression is the variable itself
//   (15 points).
// - Return the statement (10 points).
Stmt assigned_self_or_default(Function f, LocalVariable v) {
  ...
}

// --------------------

from Function f, LocalVariable v, Stmt s
where
  parameter_default_none(f, v) and
  s = assigned_self_or_default(f, v)
select s, v
```

---

Bonus: Find and report a bug in open-source software (up to 30 points):
-----------------------------------------------------------------------

For bonus credit, find a bug in an open-source project using CodeQL and submit
an issue to the project's issue tracker on GitHub. For credit you must:

1. Use a CodeQL query to identify a bug (either one of the above or another *you
   write yourself*), and include the query and a link to the results here.

2. Submit an issue to the project's issue tracker with a description of the bug,
   and include a link to the issue here. **Make sure to check the project's
   guidelines and follow them when submitting a bug report.**

Complete the above two tasks *by the homework deadline* for an extra **20 bonus
points** (no extensions will be given for the bonus). If the developers of a
project acknowledge your report as a bug or patch it any time before the end of
the semester we will add an extra **10 bonus points** (just let us know so we
can adjust your grade).

### Rules:

Our goal is for you to make high-quality contributions to open-source projects
using program analysis, so we ask that you follow some rules when submitting bug
reports. We will not award bonus points to submissions that do not follow these
rules:

- Be sure to understand the bug you are submitting to the issue tracker. Not all
  matching instances of the above analyses are bugs, and blindly submitting
  analysis results as bugs without understanding the context is often
  counterproductive.

- Please only submit one bug report for this bonus. Do not spam issue trackers
  with bugs in an attempt to get extra points.

- Be nice.
