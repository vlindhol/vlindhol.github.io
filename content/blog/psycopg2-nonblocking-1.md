---
title: "A Useful SQLAlchemy/psycopg2 Trick: Non-blocking mode"
date: 2023-10-13T12:14:57+03:00
tags: ["programming", "python"]
draft: false
---

> This post was written while I was working for [Memfault](https://memfault.com) and re-posted with permission here. The original post is [on Memfault's blog](https://memfault.com/blog/a-useful-sqlalchemy-psycopg2-trick-non-blocking-mode/).

We discovered a handy trick to make SQLAlchemy – or rather the underlying PostgreSQL driver called `psycopg2` – behave nicer in our Python-based backend service, by not tying up the Python process while waiting for a PostgreSQL query to finish. The improvements are especially evident in long-running queries but they also improve general system responsiveness.

## The problem – blocking the Python runtime

We were inconvenienced by one simple fact: firing off a SQLAlchemy query would hand off the query to `psycopg2`, which in turn would use the C-based `libpq` to perform the actual query. Calling C code from Python usually means temporarily stepping out of “Python-land” and entering “C-land”, which means a couple of things:

1. The Python runtime is essentially frozen: it won’t handle any signals sent to it until we leave C-land.
2. Even if we use Python’s threading facilities to run some Python-based logic in parallel with the `psycopg2` query, it’s difficult to cancel the running query programmatically.

What did this mean for our service? We were (and are) heavy users of Celery for running background tasks. One symptom was that long-running PostgreSQL queries wouldn’t react to e.g. Celery’s `soft_time_limit` exceptions, which allows a task a few seconds to gracefully shut down. Instead, we had to use workarounds like always configuring an appropriate PostgreSQL `statement_timeout`, or using Celery’s `time_limit` parameter (which just kills the entire process rather unceremoniously).

## Making the SQLAlchemy PostgreSQL driver non-blocking

The solution would be to have `psycopg2` spin up a new thread, run the PostgreSQL query in C-land and send the results back to Python-land over a socket. In Python-land, we could poll the socket at our leisure, staying responsive to any and all signals and exceptions coming from the application code.

Luckily, this very thing already exists! To make `psycopg2` stay in Python-land, just add the following in your app startup routines:

```python3
def init_psycopg2() -> None:
    import psycopg2.extensions
    import psycopg2.extras

    psycopg2.extensions.set_wait_callback(psycopg2.extras.wait_select)
```

The `psycopg2.extras.wait_select` is just a reference implementation which will replicate the behavior of the default, blocking driver without getting stuck in C-land, and as an added bonus it automatically cancels the SQL query on a `KeyboardInterrupt`. It is of course possible to write your own function to pass into `set_wait_callback()`, which we did when we wrote some more advanced Celery features.

A good starting point to learn about `set_wait_callback` is [https://www.psycopg.org/articles/2014/07/20/cancelling-postgresql-statements-python/](this short article).

The fundamental reason why this feature exists in `psycopg2` is to [https://www.psycopg.org/docs/advanced.html#support-for-coroutine-libraries](enable running it in various coroutine / green thread implementations) in Python. We do not use green threads, we run a vanilla setup of Python which is fully synchronous. It is still extremely useful!

## The result

Celery can now raise a `SoftTimeLimitExceeded` exception even if a PostgreSQL query is running, and it will get canceled and cleaned up appropriately. We can also feel more confident about gracefully shutting down, since we know any cleanup logic in Python-land will run promptly without being blocked. We can also set sensible, global PostgreSQL query timeouts and rely only on per-task time limits as configured through Celery, which is a lot more ergonomic.

We also did not see any performance degradation, even though now we dip into the Python interpreter to shuffle along the query/response bytes!

## Bonus: Understanding `wait_select`

Here’s the current implementation of `psycopg2.extras.wait_select`:

```python
def wait_select(conn):
    """Wait until a connection or cursor has data available.

    The function is an example of a wait callback to be registered with
    `~psycopg2.extensions.set_wait_callback()`. This function uses
    :py:func:`~select.select()` to wait for data to become available, and
    therefore is able to handle/receive SIGINT/KeyboardInterrupt.
    """
    import select
    from psycopg2.extensions import POLL_OK, POLL_READ, POLL_WRITE

    while True:
        try:
            state = conn.poll()
            if state == POLL_OK:
                break
            elif state == POLL_READ:
                select.select([conn.fileno()], [], [])
            elif state == POLL_WRITE:
                select.select([], [conn.fileno()], [])
            else:
                raise conn.OperationalError(f"bad state from poll: {state}")
        except KeyboardInterrupt:
            conn.cancel()
            # the loop will be broken by a server error
            continue
```

So how does this work? `wait_select` is passed a `conn` variable which represents the current DB connection. The connection can be in one of four states:

* `POLL_WRITE`: Writing the query contents (`SELECT * FROM …`) to `libpq`.
* `POLL_READ`: Reading the results of the query from `libpq`.
* `POLL_OK`: The reading of the results is done ⇒ nothing more to do.
* `POLL_ERROR`: Something went wrong.

A connection always starts out in the `POLL_WRITE` state. We then do the follow algorithm in a loop:

1. **Blocking**: Call `conn.poll()`, which will use a socket to read/write some bytes to `libpq`.
2. Perhaps do some other logic (we do nothing in the above implementation).
3. **Blocking**: Depending on what the connection state is (`poll()` returns it), we use `select.select()` to wait until the socket is once again ready for reading/writing.
4. If the state is `POLL_OK`, we can break out of the loop since our work is done.

The thing that is special here is step (2) in the algorithm. In the above default implementation we don’t do anything, but that’s not exactly true: we give the Python interpreter a chance to respond to signals, for example right after `conn.poll()` is where the runtime may raise a `KeyboardInterrupt` in the event you press `ctrl+c` in a REPL session.

This exact thing is taken into account in the default implementation: we wrap the polling algorithm in a `try` block and catch any `KeyboardInterrupt`. At this point we tell `conn` to `cancel()` itself and then advance the loop as usual (`continue`) to allow libpq to continue reading/writing data, emptying its buffers, until internally `psycopg2` can return a `POLL_ERROR` from `conn.poll()`. This may not happen immediately.