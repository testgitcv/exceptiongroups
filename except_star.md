# Introducing try..except* syntax

## Disclaimer

* We use the `ExceptionGroup` name, even though there
  are other alternatives, e.g. `AggregateException` and `MultiError`.
  Naming of the "exception group" object is out of scope of this proposal.

* We use the term "naked" exception for regular Python exceptions
  **not wrapped** in an `ExceptionGroup`. E.g. a regular `ValueError`
  propagating through the stack is "naked".

* `ExceptionGroup` is an iterable object.
  E.g. `list(ExceptionGroup(ValueError('a'), TypeError('b')))` is
  equal to `[ValueError('a'), TypeError('b')]`

* `ExceptionGroup` is not an indexable object; essentially
  it's similar to Python `set`. The motivation for this is that exceptions
  can occur in random order, and letting users write `group[0]` to access the
  "first" error is error prone. Although the actual implementation of
  `ExceptionGroup` will likely use an ordered list of errors to preserve
  the actual occurrence order for rendering.

* `ExceptionGroup` is a subclass of `BaseException`,
  is assignable to `Exception.__context__`, and can be
  directly handled with `try: ... except ExceptionGroup: ...`.

* The behavior of the regular `try..except` statement will not be modified.

## Syntax

We're considering to introduce a new variant of the `try..except` syntax to
simplify working with exception groups:

```python
try:
  ...
except *SpamError:
  ...
except *BazError as e:
  ...
except *(BarError, FooError) as e:
  ...
```

The new syntax can be viewed as a variant of the tuple unpacking syntax.
The `*` symbol indicates that zero or more exceptions can be "caught" and
processed by one `except *` clause.

## Semantics

### Overview

The `except *SpamError` block will be run if the `try` code raised an
`ExceptionGroup` with one or more instances of `SpamError`. It would also be
triggered if a naked instance of  `SpamError` was raised.

The `except *BazError as e` block would aggregate all instances of `BazError`
into a list, wrap that list into an `ExceptionGroup` instance, and assign
the resultant object to `e`. The type of `e` would be
`ExceptionGroup[BazError]`.  If there was just one naked instance of
`BazError`, it would be wrapped into an `ExceptionGroup` and assigned to `e`.

The `except *(BarError, FooError) as e` would aggregate all instances of
`BarError` or `FooError`  into a list and assign that wrapped list to `e`.
The type of `e` would be `ExceptionGroup[Union[BarError, FooError]]`.

Even though every `except*` star can be called only once, any number of
them can be run during handling of an `ExceptionGroup`. E.g. in the above
example,  both `except *SpamError:` and `except *(BarError, FooError) as e:`
could get executed during handling of one `ExceptionGroup` object, or all
of the `except*` clauses, or just one of them.

It is not allowed to use both regular except blocks and the new `except*`
clauses in the same `try` block. E.g. the following example would raise a
`SyntaxErorr`:

```python
try:
   ...
except ValueError:
   pass
except *CancelledError:
   pass
```

Exceptions are matched using a subclass check. For example:

```python
try:
  low_level_os_operation()
except *OSerror as errors:
  for e in errors:
    print(type(e).__name__)
```

could output:

```
BlockingIOError
ConnectionRefusedError
OSError
InterruptedError
BlockingIOError
```

### Raising ExceptionGroups manually

Exception groups can be raised manually:

```python
try:
  low_level_os_operation()
except *OSerror as errors:
  new_errors = []
  for e in errors:
    if e.errno != errno.EPIPE:
       new_errors.append(e)
  raise ExceptionGroup(*new_errors)
```

The above code ignores all `EPIPE` OS errors, while letting all other
exceptions propagate.

Raising an `ExceptionGroup` introduces nesting:

```python
try:
  raise ExceptionGroup(ValueError('a'), TypeError('b'))
except *ValueError:
  raise ExceptionGroup(KeyError('x'), KeyError('y'))

# would result in:
#
#   ExceptionGroup(
#     ExceptionGroup(
#       KeyError('x'),
#       KeyError('y'),
#     ),
#     TypeError('b'),
#   )
```

Although a regular `raise Exception` would not wrap `Exception` in a group:

```python
try:
  raise ExceptionGroup(ValueError('a'), TypeError('b'))
except *ValueError:
  raise KeyError('x')

# would result in:
#
#   ExceptionGroup(
#     KeyError('x'),
#     TypeError('b')
#   )
```

### Unmatched Exceptions

Example:

```python
try:
  raise ExceptionGroup(
    ValueError('a'), TypeError('b'), TypeError('c'), KeyError('e')
  )
except *ValueError as e:
  print(f'got some ValueErrors: {e}')
except *TypeError as e:
  print(f'got some TypeErrors: {e}')
  raise
```

The above code would print:

```
got some ValueErrors: ExceptionGroup(ValueError('a'))
got some TypeErrors: ExceptionGroup(TypeError('b'), TypeError('c'))
```

and then terminate with an unhandled `ExceptionGroup`:

```
ExceptionGroup(
  KeyError('e'),
  ExceptionGroup(
    TypeError('b'),
    TypeError('c')
  )
)
```

Basically, before interpreting `except *` clauses, the interpreter will
have an "incoming" `ExceptionGroup` object with a list of exceptions in it
to handle, and then:

* A new empty "result" `ExceptionGroup` would be created by the interpreter.

* Every `except *` clause, run from top to bottom, can filter some of the
  exceptions out of the group and process them. If the except block terminates
  with an error, that error is put to the "result" `ExceptionGroup` (with the
  group of unprocessed exceptions referenced via the `__context__` attribute.)

* After there are no more `except*` clauses to evaluate, there are the
  following possibilities:

  * Both "incoming" and "result" `ExceptionGroup` are empty. This means
    that all exceptions were processed and silenced.

  * Both "incoming" and "result" `ExceptionGroup` are not empty.
    This means that not all of the exceptions were matched, and some were
    matched but either triggered new errors, or were re-raised. The interpreter
    would merge both groups into one group and raise it.

  * The "incoming" `ExceptionGroup` is non-empty: not all exceptions were
    processed. The interpreter would raise the "incoming" group.

  * The "result" `ExceptionGroup` is non-empty: all exceptions were processed,
    but some were re-raised or caused new errors. The interpreter would
    raise the "result" group.

The order of `except*` clauses is significant just like with the regular
`try..except`, e.g.:

```python
try:
  raise BlockingIOError
except *OSError as e:
  # Would catch the error
  print(e)
except *BlockingIOError:
  # Would never run
  print('never')

# would print:
#
#   ExceptionGroup(BlockingIOError())
```

### Exception Chaining

If an error occurs during processing a set of exceptions in a `except *` block,
all matched errors would be put in a new `ExceptionGroup` which would be
references from the just occurred exception via its `__context__` attribute:

```python
try:
  raise ExceptionGroup(ValueError('a'), ValueError('b'), TypeError('z'))
except *ValueError:
  1 / 0

# would result in:
#
#   ExceptionGroup(
#     TypeError('z'),
#     ZeroDivisionError()
#   )
#
# where the `ZeroDivisionError()` instance would have
# its __context__ attribute set to
#
#   ExceptionGroup(
#     ValueError('a'), ValueError('b')
#   )
```

It's also possible to explicitly chain exceptions:

```python
try:
  raise ExceptionGroup(ValueError('a'), ValueError('b'), TypeError('z'))
except *ValueError as errors:
  raise RuntimeError('unexpected values') from errors

# would result in:
#
#   ExceptionGroup(
#     TypeError('z'),
#     RuntimeError('unexpected values')
#   )
#
# where the `RuntimeError()` instance would have
# its __cause__ attribute set to
#
#   ExceptionGroup(
#     ValueError('a'),
#     ValueError('b')
#   )
```

### Recursive Matching

The matching of `except *` clauses against an `ExceptionGroup` is performed
recursively. E.g.:

```python
try:
  raise ExceptionGroup(
    ValueError('a'),
    TypeError('b'),
    ExceptionGroup(
      TypeError('c'),
      KeyError('d')
    )
  )
except *TypeError as e:
  print(f'got some TypeErrors: {len(e)}')
except *Exception:
  pass

# would print:
#
#  got some TypeErrors: 2
```

Iteration over an `ExceptionGroup` that has nested `ExceptionGroup` objects
in it effectively flattens the entire tree. E.g.

```python
print(
  list(
    ExceptionGroup(
      ValueError('a'),
      TypeError('b'),
      ExceptionGroup(
        TypeError('c'),
        KeyError('d')
      )
    )
  )
)

# would output:
#
#   [ValueError('a'), TypeError('b'), TypeError('c'), KeyError('d')]
```

### "raise e" vs "raise"

The difference between bare `raise` and a more specific `raise e` is more
significant for exception groups than it is for regular exceptions. Consider
the following two examples that illustrate it.

Bare `raise` preserves the exception group internal structure:

```python
try:
  raise ExceptionGroup(
    ValueError('a'),
    TypeError('b'),
    ExceptionGroup(
      TypeError('c'),
      KeyError('d')
    )
  )
except *TypeError as e:
  raise

# would terminate with:
#
#  ExceptionGroup(
#    ValueError('a'),
#    TypeError('b'),
#    ExceptionGroup(
#      TypeError('c'),
#      KeyError('d')
#    )
#  )
```

Whereas `raise e` would flatten the captured subset:

```python
try:
  raise ExceptionGroup(
    ValueError('a'),
    TypeError('b'),
    ExceptionGroup(
      TypeError('c'),
      KeyError('d')
    )
  )
except *TypeError as e:
  raise e

# would terminate with:
#
#  ExceptionGroup(
#    ValueError('a'),
#    ExceptionGroup(
#      TypeError('b'),
#      TypeError('c'),
#    ),
#    ExceptionGroup(
#      KeyError('d')
#    )
#  )
```

### "continue" and "break" in "except*"

Both `continue` and `break` are disallowed in `except*` clauses, causing
a `SyntaxError`.

Due to the fact that `try..except*` block allows multiple `except*` clauses
to run while handling one `ExceptionGroup` with multiple different exceptions
in it, allowing one innocent `break` or `continue` in one `except*` to
effectively silence the entire group feels very error prone.

### "return" in "except*"

A `return` in a regular `except` or `finally` clause means
"suppress the exception". For example, both of the below functions would
silence their `ZeroDivisionError`s:

```python
def foo():
   try:
      1 / 0
   finally:
      print('the sound of')
      return

def bar():
   try:
      1 / 0
   except ZeroDivisionError:
     return
   finally:
      print('silence')

foo()
bar()

# would print:
#
#   the sound of
#   silence
```

We propose to replicate this behavior in the `except*` syntax as it is useful
as an escape hatch when it's clear that all exceptions can be silenced.

That said, the regular try statement allows to return a value from the except
or the finally clause:

```python
def bar():
   try:
      1 / 0
   except ZeroDivisionError:
     return 42

print(bar())

# would print "42"
```

Allowing non-None returns in `except*` allows to write unpredictable code,
e.g.:

```python
try:
    raise ExceptionGroup(A(), B())
except *A:
    return 1
except *B:
    return 2
```

Therefore non-None returns are disallowed in `except*` clauses.


## Design Considerations

### Why try..except* syntax

Fundamentally there are two kinds of exceptions: *control flow exceptions*
(e.g. `KeyboardInterrupt` or `asyncio.CancelledError`) and
*operation exceptions* (e.g. `TypeError` or `KeyError`).

When writing async/await code that uses a concept of TaskGroups (or Trio's
nurseries) to schedule different code concurrently, the users should
approach these two kinds in a fundamentally different way.

*Operation exceptions* such as `KeyError` should be handled within
the async Task that runs the code.  E.g. this is what users should do:

```python
try:
  dct[key]
except KeyError:
  # handle the exception
```

and this is what they shouldn't do:

```python
try:
  async with asyncio.TaskGroup() as g:
    g.create_task(task1); g.create_task(task2)
except *KeyError:
  # handling KeyError here is meaningless, there's
  # no context to do anything with it but to log it.
```

*Control flow exceptions* are different. If, for example, we want to
cancel an asyncio Task that spawned other multiple concurrent Tasks in it
with a an `asyncio.TaskGroup`, the following will happen:

* CancelledErrors will be propagated to all still running tasks within
  the group;

* CancelledErrors will be propagated to the Task that scheduled the group and
  bubble up from `async with TaskGroup()`;

* CancelledErrors will be propagated to the outer Task until either the entire
  program shuts down with a `CancelledError`, or the cancellation is handled
  and silenced (e.g. by `asyncio.wait_for()`).

*Control flow exceptions* alter the execution flow of a program.
Therefore it is sometimes desirable for the user to react to them and
run code, for example, to free resources.

Suppose we have the `except *ExceptionType` syntax that only matches
`ExceptionGroup[ExceptionType]` exceptions (a naked `ExceptionType` wouldn't
be matched).  This means that we'd see a lot of code duplication:


```python
try:
  async with asyncio.TaskGroup() as g:
    g.create_task(task1); g.create_task(task2)
except *CancelledError:
  log('cancelling server bootstrap')
  await server.stop()
  raise
except CancelledError:
  # Same code, really.
  log('cancelling server bootstrap')
  await server.stop()
  raise
```

Which leads to the conclusion that `except *CancelledError as e` should both:

* catch a naked `CancelledError`, wrap it in an `ExceptionGroup` and bind it
  to `e`. The type of `e` would always be `ExceptionGroup[CancelledError]`.

* if an exception group is propagating through the `try`,
  `except *CancelledError` should split the group and handle all exceptions
  at once with one run of the code in `except *CancelledError` (and not
  run the code for every matched individual exception.)

Why "handle all exceptions at once"? Why not run the code in the except
clause for every matched exception that we have in the group?
Basically because there's no need to. As we mentioned above, catching
*operation exceptions* should be done with the regular `except KeyError`
within the Task boundary, where there's context to handle a `KeyError`.
Catching *control flow exceptions* is needed to **react** to a global
signal, do cleanup or logging, but ultimately to either **stop** the signal
**or propagate** it up the caller chain.

Separating exception kinds to two distinct groups (operation & control flow)
leads to another conclusion: an individual `try..except` block usually handles
either the former or the latter, **but not a mix of both**. Which leads to the
conclusion that `except *CancelledError` should switch the behavior of the
entire `try` block to make it run several of its `except *` clauses if
necessary.  Therefore:

```python
try:
  # code
except KeyError:
  # handle
except ValueError:
  # handle
```

is a regular `try..except` block to be used for reacting to
*operation exceptions*. And:

```python
try:
  # code
except *TimeoutError:
  # handle
except *CancelledError:
  # handle
```

is an entirely different construct meant to make it easier to react to
*control flow* signals.  When specified that way, it is expected from the user
standpoint that both `except` clauses can be potentially run.

Lastly, code that combines handling of both operation and control flow
exceptions is unrealistic and impractical, e.g.:

```python
try:
  async with TaskGroup() as g:
    g.create_task(task1())
    g.create_task(task2())
except ValueError:
  # handle ValueError
except *CancelledError:
  # handle cancellation
  raise
```

In the above snippet it is impossible to attribute which task raised a
`ValueError` -- `task1` or `task2`. So it really should be handled directly
in those tasks. Whereas handling `*CancelledError` makes sense -- it means that
the current task is being canceled and this might be a good opportunity to do
a cleanup.


### Adoption of try..except* syntax

Application code typically can dictates what version of Python it requires.
Which makes introducing TaskGroups and the new `except *` clause somewhat
straightforward. Upon switching to Python 3.10, the application developer
can grep their application code for every *control flow* exception they handle
(search for `except CancelledError`) and mechanically change it to
`except *CancelledError`.

Library developers, on the other hand, will need to maintain backwards
compatibility with older Python versions, and therefore they wouldn't be able
to start using the new `except *` syntax right away.  They will have to use
the new ExceptionGroup low-level APIs along with `try..except ExceptionGroup`
to support running user code that can raise exception groups.


## See Also

* An analysis of how exception groups will likely be used in asyncio
  programs:
  https://github.com/python/exceptiongroups/issues/3#issuecomment-716203284

* A WIP implementation of the `ExceptionGroup` type by @iritkatriel
  tracked [here](https://github.com/iritkatriel/cpython/tree/exceptionGroup).