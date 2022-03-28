# Sheertex Technology Coding Style Rules and Guidelines

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

## Edge cases should be handled with ``return`` or ``continue``, not ``else``

In general, if a function or loop contains one or more "edge case" checks
before the "main body", the edge case checks should be handled first, and
a ``return`` or ``continue`` should be used instead of an ``else``
statement.

For example:

```python
# Good:
def get_user_settings_by_email(email):
    user = User.get(email=email)
    if not user:
        # Edge case: the user does not exist
        return None

    settings = UserSettings.get_for_user(user)
    if not settings:
        # Edge case: the user does not have any settings
        return UserSettings.get_default()

    return settings

# Good
def send_login_reminder_emails(users):
    for user in users:
        if user.did_login_recently():
            continue
        if not user.account_is_active():
            continue
        email.send("login_reminder", user=user)

# Bad:
def get_user_settings_by_email(email):
    user = User.get(email=email)
    if user:
        settings = UserSettings.get_for_user(user)
        if settings:
            return settings
        else:
            return UserSettings.get_default()
    else:
        return None
        
# Bad
def send_login_reminder_emails(users):
    for user in users:
        if not user.did_login_recently():
            if user.account_is_active():
                email.send("login_reminder", user=user)
```

<details>
  <summary><strong>Explanation</strong></summary>

Handling edge cases with ``return`` or ``continue`` instead of ``else`` has
a few benefits:

1. The reader knows that the return statement at the bottom of the function
   is what the author "expects" to happen, so it's easier for them to
   understand the *intent* of the function, and figure out what it's
   *supposed* to do.

2. It reduces indentation, and therefor the amount of state the reader needs to
   keep in their head as they are reading the function. Each time a reader
   encounters a level of indent, to understand the code in that indent block, they
   must go back and see what caused the indent (ex, if you see code inside an "if"
   statement, to understand that code, you must also read an understand the "if"
   statement); this is "state" the reader must keep track of. This style of
   function generally reduces the amount of code that's indented, thus the amount
   of "state" they need to keep in their head as they read.

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


## React Component Props in Typescript

[Context for this decision](https://github.com/Sheertex/sheertex-ops.com/pull/2071#discussion_r830137097)

Within a React component, when using Typescript, the attributes of the props object should be declared in the type signature.

```typescript
// Good
const Component = (p: {first: string, second: number}) {
  ...
}

// Bad
const Component = (p) {
  const { first, second } = p
}
```

Once declared, the attributes should be accessed as members of the `props` (or `p`) parameter without destructuring to additional variables.


## Python String Formatting

In general, strings should be formatted using either `f"..."` string
formatting, or `"..." %(...)` formatting. `"...".format(...)` should be
avoided.

As much as possible - and especially in log messages - values should be
`repr`'d using `f"{...!r}"` or `"%r" %(...)`. This can help debugging when
unexpected values arise (see explanation, below).

**The `%d` formatter should be avoided** as it can raise an exception
if the input value isn't an integer (eg, `log.info("bag tag id: %d",
bag_tag.id) # BAD` will raise an exception if `bag_tag.id == None`).

```python
# GOOD:
log.info("bag tag id: %r", bag_tag.id) # GOOD
message = "sku added: %r" %(new_sku, ) # GOOD
print(f"knitting machine: {knitting_machine.number!r}") # GOOD

# BAD:
log.info("bag tag id: %d", bag_tag.id) # BAD
message = "sku added: {}".format(new_sku) # BAD
print("knitting machine: {knitting_machine_number}".format(knitting_machine_number=knitting_machine.number) # BAD
```

<details>
<summary><strong>Explanation</strong></summary>

This guideline exists to promote consistent, robust, debuggable code.

The `.format(...)` method is avoided for consistency (it's not frequently
used in the codebase), and because it has no advantages over `f`-string
formatting (`f"..."`).

The `%r` formatter is generally preferred over the `%s` formatter -
especially for log messages - because it can simplify debugging: if a variable
has an unexpected type or value, that might be obscured by the `%s` formatter,
where the `%r` formatter will often make the variable's type obvious.

For example, consider the log message:

```python
def add_tubes_to_flow(flow_id, tube_count):
  log.info("adding %s tubes to flow %s", tube_count, flow_id) # BAD
  # ...
```

If the function is called with unexpected or invalid inputs, the log message
will make it difficult to notice those errors:

```
>>> add_tubes_to_flow("", "42")
INFO: adding 42 tubes to flow
```

Contrast with the same log message, but `%r` is used instead of `%s`, which
makes it immediately obvious that the `tube_count` is a `str` instead of an
`int`, and the `flow_id` is an empty string:

```python
>>> def add_tubes_to_flow(flow_id, tube_count):
...   log.info("adding %r tubes to flow %r", tube_count, flow_id) # GOOD
...   # ...
>>> add_tubes_to_flow("", "42")
INFO: adding '42' tubes to flow ''
```

The `%d` formatter should also generally be avoided, as it can cause unexpected
exceptions if the input is not an integer. For example, consider the log
message:

```python
def close_bag_tag(bag_tag):
  log.info("closing bag tag: %d", bag_tag.id) # BAD
  # ...
```

If the `bag_tag.id` is `None` (for example, because it was freshly created and
not saved to the database yet), the log message will raise an exception:

```
>>> close_bag_tag(BagTag())
...
TypeError: %d format: a number is required, not NoneType
```

(this could also happen if, ex, the input is a `str` instead of an `int`)

In the relatively rare situations where `%d` formatting is specifically
necessary - for example, because the input is a `float` and only the integral
portion should be displayed - explicitly casting the value to an `int` can
make the intention of the code more obvious:

```python
log.info("duration: %s seconds", int(duration))
```

(and has the added benefit that, if the input value cannot be converted to
an `int`, the original value will be included in the exception's message:

```
>>> int("foo")
...
ValueError: invalid literal for int() with base 10: 'foo'
>>> "%d" %("foo", )
...
TypeError: %d format: a number is required, not str
```

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

#### Python: errors in HTTP handlers and `500` errors

HTTP handlers (`@route`s) should never need to catch `Exception` or return a
`500` error. Exceptions raised in HTTP routes are indicitive of a bug in the
underlying code, and should be handled by the Flask exception handling
middleware, which will log them and return a `500` error to the caller:

```python
@route("/api/some/route")
def api_some_route():
  try:
    return JsonResponse(do_some_thing())
  except Exception as e:
    # Bad: this exception could be the result of an underlying bug, but
    # the error will not be logged, no alert will be sent to
    # tech-software-alerts, and there will be no stack trace to help debug
    # the error.
    return ApiError({ "msg": "Error with some_thing: %r" %(e, ) }) # Bad
```

Instead, don't worry about catching unknown exceptions; they will be handled
by Flask's middleware:

```python
@route("/api/some/route")
def api_some_route():
  # Good: exceptions will be caught and handled by the exception handling
  # middleware.
  return JsonResponse(do_some_thing()) # Good
```

#### Python: using `flask.abort()`

The `flask.abort()` helper should not be used. It leads to less clear code (it
raises an exception interally, but raising an exception explicitly is more
obvious), and does not return a JSON response:

```python
@route("/api/some/route")
def api_some_route():
  if some_condition:
    # Bad: abort() should not be used
    abort() # Bad
  return JsonResponse(do_some_thing())
```

Instead, an `ApiError(...)` should be raised:

```python
@route("/api/some/route")
def api_some_route():
  if some_condition:
    raise ApiError("some_condition was not met") # Good
  return JsonResponse(do_some_thing())
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
        raise ApiError(404, f"User not found: {user_id!r}") # Good
    ...
```


# Guidelines-in-progress

The following are guidelines-in-progress which should be turned into more
complete examples:

* Never have a function with non-obvious unnamed arguments, *especially* when they are the same type.

```typescript
   // Bad:
   function saveUser(username: string, name: string, favoriteColor: string) {}
   
   // Good:
   function saveUser(u: { username: string, name: string, favoriteColor: string }) {}
```
   
```python
   # Good:
   def save_user(*, username, name, favorite_color)
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
