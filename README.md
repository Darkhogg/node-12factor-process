12factor-process
================

A small set of utilities thathelps with writing multi-process apps.


Worker Information
------------------

In a multi-process environment, every process is assigned a different name and
number.  The process name identifies the type of the process and therefor what
it should run, while its number distinguishes it from its counterparts.

In order to access process information, use the `worker` entry in the object
returned by `require('12factor-process')` (called `f12proc` here):

  + `f12proc.worker.id`: The full worker name, consisting of worker name and
    number separated by a dot (`.`).
  + `f12proc.worker.name`: Name of the current worker.
  + `f12proc.worker.number`: Number of the current worker.

These values are all obtained from a *worker identifier string*.  This string
is located on one of the following places:

  + The `FOREMAN_WORKER_NAME` environment variable, set by [node-foreman][] for
    every process.
  + The `PS` environment variable, set by [foreman][] for every process.
  + The `DYNO` environment variable, set by [Heroku][] for every process.

If the worker ID is not found, it is set a default value of `web.1`, which is
a reasonable default given the nature of the 12 factor app.

There are a few restrictions on the worker identifier string: Its name must
consist only of alphanumeric characters or the caracters `_` and `-` and its
number must be a positive integer strictly greater than 0 and with no leading
zeroes.  If a restriction is violated, the worker ID is set to `__error__.0`
and other values are set accordingly.

  [foreman]: http://ddollar.github.io/foreman/ "foreman"
  [node-foreman]:
  [heroku]:


Application Crashing
--------------------

12 factor apps should be written to be crash-only or, at least, crash-first.
Crash-only apps are applications that are coded in such a way that shutting
them down is performed just by crashing the app.  Crash-first apps are
crash-only apps that also include a *correct* way of being shut down, which is
generally better and preferred, but still respond well and can work correctly
if they are just crashed every time.

Crash here just means "end abruptly".  For node, this means calling
`process.exit`.  For a Unix process, it means killing a process with SIGKILL.
Either way, the result is the same: The process is stopped with little to no
chance of performing any clean up.

This library thus provides a single function that allows you to creash your application.  It lets you choose the method of crashing and allows you to define *crash-hooks*.  It also allows you to completely bypass the hooks or to set a timeout after which the process is *just crashed*.


### `f12proc.crash([reason[, runHooks[, timeout]]])`

Crashes the current process with the given `reason`.  Let's look into the
arguments in detail:

  + `reason`: The reason of the crash.  This can be one of the following:

      - An integer, which will be passed to `process.exit` directly.
      - A signal name, which will be passed to `process.exit`.
      - An exception object.

    When anything is passed as a `reason` other than 0, the process will exit
    with an exit code different from 0.  If unspecified by the reason, this
    exit code will be 170.

    Its default value is `0`.

  + `runHooks`: Whether to run crash hooks or not.  Note that you should test
    your application by crashing it with and without running crash hooks or you
    risk your application being neither crash-only nor crash-first.

    Its default value is `true`.

  + `timeout`: The time to wait, in milliseconds, for hooks to end before
    definitely crashing the application.  If hooks are not allowed to run, this
    value is completely ignored.  If the application exits due to a timeout, it
    will exit with exit code 171.

    Its default value is `5000`, that is, *five seconds*.


### `f12proc.crash.addHook(hook)`

Adds a new crash hook.  Crash hooks are global, which means that multiple
version of the library loaded by different modules will the same crash hooks.

  + `hook`: A function to run when the application crashes, with the following
    signature: `function (reason[, callback])`.

      - `reason` the crash reason used when the `f12proc.crash` function was
        called.
      - `callback`: An optional callback argument in case the function is
        asynchronous.  The callback follows the common pattern of having an
        error as its first argument.  It doesn't accept any more arguments.

    If the `hook` function does not have a `callback` argument, it is assumed
    that all it had to do was completed when it returns *unless* it returns
    a [Promise][promises-aplus].  If either a promise is returned or a callback
    accepted, the hook is considered asynchronous and will be waited for before
    finishing the crash (unless, of course, it times out).

It is recommended to use promises rather than callbacks, but either way is
fine.  Errors during both synchronous or asynchronous hooks are completely
ignored.

  [promises-aplus]: https://promisesaplus.com/ "Promises/A+"

There's one guarantee about hooks: They will never be run twice.


### `f12proc.crash.handleSignals(signals[, runHooks[, timeout]])`

In order to easily work with signals such as `SIGINT` and `SIGTERM` and handle
them as direct crashes, the `crash.handleSignals` function is provided.

To use it, just pass one or more signal names to it.  When the process receives
said signals, the `crash` method will be automatically called with the passed
arguments and the signal name as the reason.

  + `signals`: A signal name or array of signal names to watch and crash on.
  + `runHooks`: Whether to rook crash hooks before crashing.
  + `timeout`: Time, in milliseconds, to wait for crash hooks.

Note that there might be multiple callers for the same signal.  That is,
however, completely idiotic, as signal handlers should be configured just once
in an application.  For that reason, nothing is done to prevent it or to define
what happens.  Most likely, all handler are called at once and the fastest
wins.  *In any case, crash hooks are never run twice*.