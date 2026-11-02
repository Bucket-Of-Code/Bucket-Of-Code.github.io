---
layout: post
permalink: /blog/how-to-log-exceptions-in-python
excerpt: Use logging.exception() inside an except block to log an error message and the full traceback, then decide whether to re-raise the exception or handle it.
---

Logging exceptions in Python can be a little tricky. Especially since you need to know how to capture the exceptions in order to log them. This article gives a short explanation of logging exceptions.

## TL;DR;

```python
import logging

try:
    1/0
except Exception as exc:
    logging.exception("An exception occurred while dividing by zero")
    # If you want to "bubble-up" the exception, re raise the exception
    raise
```

## Use `logging.exception()` Inside an `except` Block

`logging.exception()` logs your message at the `ERROR` level and automatically appends the traceback of the exception that is currently being handled. That traceback is the part you will actually need at 2 AM when you are trying to figure out what went wrong.

```python
import logging

logging.basicConfig(level=logging.INFO)

def divide(a, b):
    return a / b

try:
    divide(1, 0)
except Exception:
    logging.exception("Could not divide")
```

The output looks something like this (the exact formatting of the traceback varies a little between Python versions):

```text
ERROR:root:Could not divide
Traceback (most recent call last):
  File "example.py", line 9, in <module>
    divide(1, 0)
  File "example.py", line 6, in divide
    return a / b
ZeroDivisionError: division by zero
```

Notice that you do not need to pass the exception to `logging.exception()`. It finds the exception on its own, which is why it only works inside an `except` block. If you call it anywhere else, there is no exception to find and you will get `NoneType: None` where the traceback should be.

## Use a Logger Instead of the Root Logger

The examples above use the `logging` module directly, which logs through the root logger. That is fine for a quick script. For anything bigger, create a logger for each module. The logger name shows up in your log output, so you can tell where a message came from.

```python
import logging

logger = logging.getLogger(__name__)

def save_user(user):
    try:
        user.save()
    except Exception:
        logger.exception("Could not save user %s", user.id)
        raise
```

Pass values as arguments (`%s`) instead of building the message with an f-string. The logging module only formats the message if it is actually going to be emitted.

## Log at a Different Level with `exc_info=True`

`logging.exception()` is always at the `ERROR` level. Sometimes an exception is expected and you only want a `WARNING`, or it is serious enough to be `CRITICAL`. Every logging method accepts `exc_info=True`, which attaches the traceback just like `logging.exception()` does.

```python
try:
    value = int(user_input)
except ValueError:
    logger.warning("User entered a bad number: %r", user_input, exc_info=True)
    value = 0
```

`logging.exception("msg")` is the same thing as `logging.error("msg", exc_info=True)`.

## Re-raise or Swallow the Exception?

Logging an exception does not handle it. After the `except` block runs, you have to decide what happens next.

- **Re-raise it** with a bare `raise` when this code cannot recover from the error and something higher up should deal with it. A bare `raise` keeps the original traceback.
- **Swallow it** (do not re-raise) when you can recover, like falling back to a default value. Make sure that is really what you want, because the program will carry on as if nothing happened.
- **Raise a different exception** when you want to add context. Use `raise ... from exc` so the original exception is kept as the cause.

```python
try:
    config = load_config(path)
except FileNotFoundError as exc:
    logger.exception("Config file is missing: %s", path)
    raise RuntimeError("Cannot start without a config file") from exc
```

## Common Mistakes

**Logging only the message of the exception.** This throws away the traceback, which is the most useful part.

```python
# Don't do this: no traceback
except Exception as exc:
    logger.error(str(exc))

# Do this instead
except Exception:
    logger.exception("Something went wrong")
```

**Logging and re-raising at every level.** If each function in a call stack logs the exception and raises it again, the same error shows up in your logs five times. Log the exception where you finally handle it, or at the top of your program.

**Using a bare `except:`.** It also catches `KeyboardInterrupt` and `SystemExit`, so you can no longer stop your program with Ctrl+C. Use `except Exception:` instead.

**Calling `logging.exception()` outside of an `except` block.** There is no exception to log, so you only get `NoneType: None`.

## Get the Traceback as a String

Sometimes you need the traceback as text, for example to put it in an email or a Slack message. Use `traceback.format_exc()` inside the `except` block.

```python
import traceback

try:
    risky_operation()
except Exception:
    details = traceback.format_exc()
    send_alert(details)
    raise
```

## Log Exceptions You Did Not Catch

Any exception that is not caught crashes your program, and by default Python prints the traceback to `stderr` only. If your logs go to a file or a logging service, you will not see the crash there. You can fix that by replacing `sys.excepthook`.

```python
import logging
import sys

logger = logging.getLogger(__name__)

def log_uncaught_exceptions(exc_type, exc_value, exc_traceback):
    if issubclass(exc_type, KeyboardInterrupt):
        # Let Ctrl+C behave the way it normally does
        sys.__excepthook__(exc_type, exc_value, exc_traceback)
        return
    logger.critical("Uncaught exception", exc_info=(exc_type, exc_value, exc_traceback))

sys.excepthook = log_uncaught_exceptions
```

This only covers the main thread. Exceptions in other threads go through `threading.excepthook` (Python 3.8+).

## Log to a File

So far the examples print to the console. To keep the logs, point `basicConfig` at a file.

```python
import logging

logging.basicConfig(
    filename="app.log",
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(name)s: %(message)s",
)
```

The traceback gets written to the file right after the message.

## Summary

- Use `logging.exception()` (or `logger.exception()`) inside an `except` block to log the message and the traceback.
- Use `exc_info=True` when you want the traceback at a level other than `ERROR`.
- Re-raise with a bare `raise` if you are not going to handle the exception.
- Log an exception once, where you handle it, instead of at every level.
- Use `sys.excepthook` to log the exceptions that nothing else caught.
