# Sheertex Automation Coding Style Rules and Guidelines

This rules and guidelines in this document exist so that code written at
Sheertex is, in order of importance:

1. **Correct**: the code we're writing manages production processes and reports
   on data used to make critical business decisions. Bugs, inconsistencies,
   unclear interfaces, poorly defined data, and other sources of incorrectness
   can lead to tens, hundreds, or thousands of dollars of wasted time and
   resources. Correctness is of the *utmost* importance.
2. **Debuggable**: no programmer is perfect; no code is perfect. We will all
   introduce bugs, and there's a good chance someone else will need to debug
   them (even if that "someone else" is "you, in six weeks"). Because we
   support daily production processes, it's important that extra care is taken
   to make sure code is fast and easy to debug through helpful log and error
   messages, descriptive variable and function names, idempotent methods, and
   well-considered abstractions.
3. **Consistent**: you are a programmer, you don't need to hear again about the
   importance of consistent coding style 🙂

# Rules

These must be followed everywhere; they should be in the forefront of your mind
as you're writing code, and PR reviewers should consider them acceptable
blockers.

That said, these are not rules in the sense of "be at work on time"; these are
rules in the sense of "don't touch a hot stove": easy to follow, have very few
exceptions, and will save lots of pain and frustration in the future.

## Rules In All Languages

### Rule: Include Context in Error Messages

All error messages (log messages, exceptions, etc) should include enough
context that someone reading the message can understand the scope and nature of
the problem.

For example:

```jsx
// Bad:
throw Error("Invalid value")
// What was the value? How was it being used? Why is it invalid?

// Good:
throw Error(`Invalid machine assigning flow: ${machineNumber} (does not exist)`)
throw Error(`Invalid completed tube count: ${count} (must be between 0 and 100)`)
// Provides context on why the error happened, includes the invalid value, and
// a short description of why it's invalid.
```


## Rules in JavaScript / TypeScript

### Rule: All possible states must be handled for every HTTP request

For every HTTP request made by the front-end:

 1. All possible states (pending, error, and success) of every HTTP request
    must be handled.
 2. The UI must reflect the state of every request (ex, buttons are disabled
    while a request is pending, a loading spinner is shown, an error message
    is displayed on failure).

This is most simply done with ``useAsyncPromise()``:

```jsx
// Bad:
const onSubmit = () => {
  serverApi.saveSomeThing()
}

// Good:
const req = useAsyncPromise()
const onSubmit = () => {
  req.bind(serverApi.saveSomeThing())
}

return <div>
  <Button disabled={req.isPending} onClick={onSubmit} />
  {req.isError && <LoadingError err={req.loadError} />}
</div>
```

Other useful tools:

* `<LoadingSpinner />`: `{req.isPending && <LoadingSpinner />}`
* `renderLoadingPromise(...)`: `if (!req.isDone) return renderLoadingPromise(req)`


## Rules in Python

### Rule: Never use `except:`

In Python, all `except` blocks should specify an exception type, even if it's
`Exception`:

```python
# Bad
try:
  do_stuff()
except: # Bad
  handle_error()


# Good
try:
  do_stuff()
except Exception:
  handle_error()
```

There are two reasons for this:

1. Due to a quirk in Python's implementation, it will capture `KeyboardError`
   (the exception raised when `ctrl-c` is pressed) and `SystemExit` (the
   exception raised when `sys.exit()` is called), which can lead to code that
   will never terminate, or otherwise behave unpredictably.

   (In the *extremely* unlikely case that you intend to catch these exceptions,
   use `except BaseException:`. But you almost certainly do not want to do
   this.)

2. It often leads to exception handling that is hard to debug. See also:
   [Exception Handling Guidelines](#exception-handling-guidelines))


## Rules in SQL

### Rule: *always* wrap production queries in `BEGIN` and `ROLLBACK`

This is, without qualification, the most important rule in this document.

Any time queries are being run against a production database (with the possible
exception of `SELECT` queries), they should be wrapped in `BEGIN` and
`ROLLBACK` and the output examined before changing the `ROLLBACK` to `COMMIT`
once you are completely satisfied the query does what you expect.

It can also be helpful to add `RETUNING *` to see exactly which rows will be
updated

```sql
-- BAD
update foo set bar = 1 where baz = 2;

-- GOOD
begin;
update foo set bar = 1 where baz = 2 returning *;
rollback;
-- visually confirm that the rows being updated match expectations
-- then change the 'rollback' to a 'commit' and re-run the query.
```

I truly cannot understate the importance of following this rule, even (and
*especially*) for the queries so quick and simple there's *no way* they could
be wrong.

<details>
  <summary><strong>Example</strong></summary>

For example, consider a simple `UPDATE` which cancels all knitting flows
making either of two particular styles:

```sql
  -- BAD! DO NOT DO THIS!
  update knitting_flow
  set status = 'CANCELED'
  where
    status in ('ORDERED', 'ASSIGNED') and
    greige_sku like 'PH-FOO%' or greige_sku like 'PH-BAR%'
```

On running this query, you would see thousands of rows updated, because the
query evaluated:

```sql
  where
    (status in … and greige_sku like 'PH-FOO%') or
    greige_sku like 'PH-BAR%'
```

Instead of the query you expected.

If, instead, `BEGIN`, `RETURNING *`, and `ROLLBACK` were used, this error
would have become quickly obvious:

```sql
  -- Good! Do this!
  begin;
  update knitting_flow
  set status = 'CANCELED'
  where
    status in ('ORDERED', 'ASSIGNED') and
    greige_sku like 'PH-FOO%' or greige_sku like 'PH-BAR%'
  returning *;
  rollback;
```
</details>

# Strong Guidelines

As opposed to the rules above, these strong guidelines should *almost always*
be followed, and PR reviewers are encourage - at their discretion - to block
PRs for violating these guidelines.

## Code style should follow that of the surrounding code

New code should follow the style of the surrounding code, even if that code
has minor deviations from the following guidelines.

Specifically, when writing new code, scroll your editor through the code you've
just written and consider:
* Does it look similar to the code around it?
* What is the same, what is different?
* Could this new code be changes so that it looks more similar to the
  surrounding code?

Common differences include:

* Error handling practices
* String quote styles (`"` VS `'`)
* Indentation styles
* Variable name conventions


## Indenting should be done in consistent levels

All indents should be done in consistent levels, either multiple of 4 spaces
(in Python), or 2 spaces (in JavaScript and SQL).

In Python:

```python
# Good:
foo = some_function(
    bar,
    baz=42,
)
bar = {
  "some_key": "some_value",
  "another_key": "another_value",
}

# Bad:
foo = some_function(bar,
                    baz=42)
bar = {
       "some_key": "some_value",
       "another_key": "another_value",
}
```

In JavaScript:
```javascript
// Good:
const foo1 = someFunction(
  bar,
  { baz: 42 },
)
const foo2 = someFunction(bar, {
  baz: 42,
})
const bar = {
  someKey: 'someValue',
  anotherKey: 'anotherValue',
}

// Bad:
const foo = someFunction(bar,
                         { baz: 42 })
const bar = { someKey: 'someValue',
              anotherKey: 'anotherValue' }
```

In SQL:

```sql
-- Good:
SELECT
  st.foo,
  dt.bar
FROM some_table AS st
LEFT JOIN different_table AS dt ON dt.some_table_id = st.id
WHERE
  st.location = 'SOME_LOCATION' AND
  dt.state = 'SOME_STATE'

-- Bad:
SELECT st.foo,
       dt.bar
FROM some_table AS st
LEFT JOIN different_table AS dt ON dt.some_table_id = st.id
WHERE st.location = 'SOME_LOCATION'
      AND dt.state = 'SOME_STATE'
```

<details>
  <summary><strong>Explanation</strong></summary>

Using consistent indent levels is important because it leads to more uniform
code, which groups together "like things".

For example, consider the following code:

```python
a = some_function(foo,
                  bar,
                  baz=42)
another_thing = some_other_function(another_fiz,
                                    another_biz,
                                    baz=42)
```

Even though the function arguments (`foo`, `bar`, and `baz`) are all the
"same things" - ie, arguments to a function call - they are at dramatically
different indent levels, which forces the reader's brain to pause and
understand that context of each indent block (ie, because it is not clear
from the indent alone).

Contrast that with:

```python
a = some_function(
    foo,
    bar,
    baz=42,
)
another_thing = some_other_function(
    another_fiz
    another_biz,
    baz=42,
)
```

Which is more uniform, and makes it easier for the reader to understand the
context of each indent block without searching for as much context.
</details>

## Functions should have one "general return" and many "edge case returns"

Functions should - in general - have one "general case return" statement at the
end, with many "edge case" / "exceptional case" return statements in the body.

This is best understood by way of example. Consider a function which finds the
last SKU knit by a particular knitting machine. In this example, the "general
case" is "the knitting machine exists and has knit a SKU", and the "edge cases"
are "the knitting machine doesn't exist" and "it has not knit anything".

For example:

```python
# Good:
def knitting_machine_get_last_sku(machine_number):
    machine = KnittingMachine.get(number=machine_number)
    if not machine:
        # Edge case: the machine doesn't exist
        return None

    latest_flow = machine.get_latest_flow()
    if not latest_flow:
        # Edge case: the machine has never knit anything before
        return None

    # General case: the machine has knit something
    return latest_flow.sku

# Bad:
def knitting_machine_get_last_sku(machine_number):
    machine = KnittingMachine.get(number=machine_number)
    if machine:
        latest_flow = machine.get_latest_flow()
        if latest_flow:
            return latest_flow.sku
    return None
```

<details>
  <summary><strong>Explanation</strong></summary>

Having one "general case" return and many "edge case" returns makes it easier
to read and understand a function for two reasons:

1. The reader knows that the return statement at the bottom of the function
   is what the author "expects" to happen, so it's easier for them to
   understand the *intent* of the function, and figure out what it's
   *supposed* to do.

2. It reduces the amount of state the reader needs to keep in their head as
   they are reading the function. Each time a reader encounters a level of
   indent, to understand the code in that indent block, they must go back and
   see what caused the indent (ex, if you see code inside an "if" statement,
   to understand that code, you must also read an understand the "if"
   statement); this is "state" the reader must keep track of. This style of
   function generally reduces the amount of code that's indented, thus the
   amount of "state" they need to keep in their head as they read.

3. It can make it easier to verify the function's correctness. Because the
   reader can assume that all code which isn't indented is "general case",
   they can assume that any necessary error checks have already been
   performed. And if they want to verify that any particular check has been
   performed, they only need to look at the top-level (ie, non-indented)
   "if" statement so see whether the check in question has been performed.
</details>

## Line length

In general, lines shouldn't be longer than about 80-90 characters.

This [may seem arbitrary and antiquated](https://softwareengineering.stackexchange.com/a/148678),
but there is good reason to impose some line length limit: modern monitors are
larger than an IBM punch card, and that horizontal space can be used to show
multiple columns of code at once.

By limiting lines to 80-90 characters, developers can reliably have two or
three columns of code open at the same time.

## Multi-line function calls + list/dictionary/object literals

If a function call, object literal, or HTML tag would exceed the 80-90
character line length limit, it should be split across multiple lines.

The following are guidelines for how lines should be split:

1. There should be "one thing" per line:

```javascript
    // Good
    const foo = [
      some_long_thing,
      another_long_thing,
      third_long_thing,
      fourth_long_thing,
    ]

    // Bad:
    const foo = [
      some_long_thing, another_long_thing,
      third_long_thing, fourth_long_thing,
    ]
```

2. Lines should be indented with "one level" of indenting (see "consistent
   indent levels", above)
3. Line joiners (commas, logical operators, etc) should be at the end of the
   line.

   In JavaScript:

```javascript
   // Good
   const foo = [
     some_long_thing,
     another_long_thing,
   ]

   // Bad:
   Const foo = [
     some_long_thing
     , another_long_thing
   ]
```

   In SQL:

```sql
   -- Good:
   WHERE
     foo = 'some_foo' AND
     bar = 'some_bar'

   -- Bad:
   WHERE
     foo = 'some_foo'
     AND some_bar = 'some_bar'
```

<details>
  <summary><strong>Explanation</strong></summary>

This guidelines is another example of encouraging consistency: by putting
"one thing" on each line, indenting each line with one level of indenting,
and putting line joiners at the end of each line, the reader can easily scan
the first column of each line to understand the "point" of that line, instead
of needing to read each line to understand what it does.
</details>

### Exception Handling Guidelines

#### All languages: be careful about hiding bugs in exception handlers

Consider the following code:

```python
try:
  foo = get_some_value()
except Exception: # Bad
  foo = "default value"
```

Or the equivalent JavaScript:

```javascript
let foo
try {
  foo = getSomeValue()
} catch (e) { // Bad
  foo = "default value"
}
```


In both examples, the code's author likely expected the `get_some_value()` /
`getSomeValue()` function to raise some specific exception, and in the event of
that exception, use a default value.

But if the `get_some_value()` / `getSomeValue()` function raises an exception
the code's author did not expect (ex, due to a bug, or some other error), then
the default value will still be used, and it will be impossible to either
notice or fix the underlying bug.

Instead, exception handlers should be as specific as possible, and re-raise any
exceptions they do not expect to catch. For example:

```python
try:
  foo = get_some_value()
except ValueError as e: # Good
  if "default value not found" not in str(e):
    raise
  foo = "default value"
```

Or the equivalent JavaScript:

```javascript
let foo
try {
  foo = getSomeValue()
} catch (e) { // Good
  const isDefaultNotFound = (
    e instanceof ValueError &&
    e.toString().indexOf("default value not found") >= 0
  )
  if (!isDefaultNotFound)
    throw e
  foo = "default value"
}
```


#### Python: `ApiError`

In Python, use the `ApiError` exception to raise and error along with
message/data to be returned to an API client.

By convention, this can be even from HTTP-agnostic service methods, since -
practically speaking - service methods are almost always called from HTTP,
and non-HTTP callers can treat ApiErrors like regular exceptions.

For example:

```python
def do_some_thing(user_id):
    user = User.get(id=user_id)
    if not user:
        raise ApiError(404, f"User not found: {user_id!r}")
    ...
```


# Guidelines-in-progress

The following are guidelines-in-progress which should be turned into more
complete examples:

* Never have a function with non-obvious unnamed arguments, *especially* when they are the same type.

```python
   Bad:  function saveUser(username: string, name: string, favoriteColor: string)
   Good: function saveUser(u: { username: string, name: string, favoriteColor: string })
   Good: def save_user(*, username, name, favorite_color)
```

   **EXERCISE:** why not?

* API request and response bodies should never be a scalar, and should rarely be an array.

```python
   Bad:  $.post("/foo", { data: "42" })
   Bad:  def post_foo(): return JsonResponse("42")
   Good: $.post("/foo", { data: { count: "42" }})
   Good: def post_foo(): return JsonResponse({ "count": "42" })
```

   **EXERCISE:** why not?

* Understand, deeply, the definition of idempotency:

   Which of the following functions are likely idempotent?

```jsx
   showConfirmationModal()
   setConfirmationModalVisible(true)
   printBagTag()
   closeLastFlow()
   closeFlow(currentFlow.id)
   setTubeCount(currentFlow.id, currentTubeCount)
   incrementTubeCount(curentFlow.id, 1)
```

* React's `useEffect` should *only* be used to:
   1. Perform actions on component mount and unmount (ex, starting an interval with `setInterval`)
   2. Update non-React components in response to state changes (ex, changing the browser window's title)
   3. Update aggregate, cached, or asynchronously fetched values used by the view (ex, using `userReq.bind(serverApi.getUser(props.userId))` to fetch user data each time the `userId` changes)

```jsx
   Bad:

   Good:
```
