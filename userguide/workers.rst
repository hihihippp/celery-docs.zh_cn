.. _guide-worker:

===============
 Workers 手册
===============

.. contents::
    :local:

.. _worker-starting:

启动 worker
===================

你可以通过下列命令在前台启动 celeryd::

    $ celeryd --loglevel=INFO

你也许会想用守护进程类工具使 `celeryd` 在后台工作。请查看 :ref:`daemonizing` 如何使用流行的守护进程工具调用 `celeryd`。

查看 :mod:`~celery.bin.celeryd` 了解完整的命令行参数清单，或通过执行::

    $ celeryd --help

你可以在同一台服务器上启动多个 worker，但一定要为每个 worker 指定唯一的名称（可以通过 :option:`--hostname|-n` 参数实现）::

    $ celeryd --loglevel=INFO --concurrency=10 -n worker1.example.com
    $ celeryd --loglevel=INFO --concurrency=10 -n worker2.example.com
    $ celeryd --loglevel=INFO --concurrency=10 -n worker3.example.com

.. _worker-stopping:

关闭 worker
===================

发送 :sig:`TERM` 信号即可关闭 worker。

收到关闭信号以后，worker 还会继续将当前任务完成才真正关闭。如果这个任务对你而言真的很重要，建议不要在任务完成前进行其他激烈的行为（例如，发送 :sig:`KILL` 信号）。

当 worker 迟迟不能关闭, 例如陷入了无限循环, 可以考虑向其发送 :sig:`KILL` 信号来强制终止 worker。但这样会导致当前任务丢失（除非任务设置了 :attr:`~celery.task.base.Task.acks_late` 选项）。

Worker 进程并不能确保所有子进程都能收到 :sig:`KILL` 信号，在此情况下可以通过手动执行来确保子进程被杀死。可以使用以下的命令::

    $ ps auxww | grep celeryd | awk '{print $2}' | xargs kill -9

.. _worker-restarting:

Restarting the worker
=====================

Other than stopping then starting the worker to restart, you can also
restart the worker using the :sig:`HUP` signal::

    $ kill -HUP $pid

The worker will then replace itself with a new instance using the same
arguments as it was started with.

.. _worker-concurrency:

兵法
===========

缺省情况下 celery 使用多进程模型来并发执行任务，你也可以使用 :ref:`Eventlet <concurrency-eventlet>`。可以通过 :option:`--concurrency` 参数来设置进程/线程的数量（缺省等于计算机的 CPU 数）。

.. admonition:: Number of processes (multiprocessing)

    More worker processes are usually better, but there's a cut-off point where
    adding more processes affects performance in negative ways.
    There is even some evidence to support that having multiple celeryd's running,
    may perform better than having a single worker.  For example 3 celeryd's with
    10 worker processes each.  You need to experiment to find the numbers that
    works best for you, as this varies based on application, work load, task
    run times and other factors.

.. _worker-persistent-revokes:

Persistent revokes
==================

Revoking tasks works by sending a broadcast message to all the workers,
the workers then keep a list of revoked tasks in memory.

If you want tasks to remain revoked after worker restart you need to
specify a file for these to be stored in, either by using the `--statedb`
argument to :mod:`~celery.bin.celeryd` or the :setting:`CELERYD_STATE_DB`
setting.  See :setting:`CELERYD_STATE_DB` for more information.

.. _worker-time-limits:

Time limits
===========

.. versionadded:: 2.0

A single task can potentially run forever, if you have lots of tasks
waiting for some event that will never happen you will block the worker
from processing new tasks indefinitely.  The best way to defend against
this scenario happening is enabling time limits.

The time limit (`--time-limit`) is the maximum number of seconds a task
may run before the process executing it is terminated and replaced by a
new process.  You can also enable a soft time limit (`--soft-time-limit`),
this raises an exception the task can catch to clean up before the hard
time limit kills it:

.. code-block:: python

    from celery.task import task
    from celery.exceptions import SoftTimeLimitExceeded

    @task()
    def mytask():
        try:
            do_work()
        except SoftTimeLimitExceeded:
            clean_up_in_a_hurry()

Time limits can also be set using the :setting:`CELERYD_TASK_TIME_LIMIT` /
:setting:`CELERYD_SOFT_TASK_TIME_LIMIT` settings.

.. note::

    Time limits do not currently work on Windows and other
    platforms that do not support the ``SIGUSR1`` signal.


Changing time limits at runtime
-------------------------------
.. versionadded:: 2.3

You can change the soft and hard time limits for a task by using the
``time_limit`` remote control command.

Example changing the time limit for the ``tasks.crawl_the_web`` task
to have a soft time limit of one minute, and a hard time limit of
two minutes::

    >>> from celery.task import control
    >>> control.time_limit("tasks.crawl_the_web",
                           soft=60, hard=120, reply=True)
    [{'worker1.example.com': {'ok': 'time limits set successfully'}}]

Only tasks that starts executing after the time limit change will be affected.

.. _worker-maxtasksperchild:

Max tasks per child setting
===========================

.. versionadded: 2.0

With this option you can configure the maximum number of tasks
a worker can execute before it's replaced by a new process.

This is useful if you have memory leaks you have no control over
for example from closed source C extensions.

The option can be set using the `--maxtasksperchild` argument
to `celeryd` or using the :setting:`CELERYD_MAX_TASKS_PER_CHILD` setting.

.. _worker-remote-control:

远程控制
==============

.. versionadded:: 2.0

可以通过高优先级的广播进行远程控制。命令将被发送到全部或者部分 worker。

Commands can also have replies.  The client can then wait for and collect
those replies.  Since there's no central authority to know how many
workers are available in the cluster, there is also no way to estimate
how many workers may send a reply, so the client has a configurable
timeout — the deadline in seconds for replies to arrive in.  This timeout
defaults to one second.  If the worker doesn't reply within the deadline
it doesn't necessarily mean the worker didn't reply, or worse is dead, but
may simply be caused by network latency or the worker being slow at processing
commands, so adjust the timeout accordingly.

In addition to timeouts, the client can specify the maximum number
of replies to wait for.  If a destination is specified, this limit is set
to the number of destination hosts.

.. seealso::

    The :program:`celeryctl` program is used to execute remote control
    commands from the command line.  It supports all of the commands
    listed below.  See :ref:`monitoring-celeryctl` for more information.

.. _worker-broadcast-fun:

The :func:`~celery.task.control.broadcast` function.
----------------------------------------------------

This is the client function used to send commands to the workers.
Some remote control commands also have higher-level interfaces using
:func:`~celery.task.control.broadcast` in the background, like
:func:`~celery.task.control.rate_limit` and :func:`~celery.task.control.ping`.

Sending the :control:`rate_limit` command and keyword arguments::

    >>> from celery.task.control import broadcast
    >>> broadcast("rate_limit", arguments={"task_name": "myapp.mytask",
    ...                                    "rate_limit": "200/m"})

This will send the command asynchronously, without waiting for a reply.
To request a reply you have to use the `reply` argument::

    >>> broadcast("rate_limit", {"task_name": "myapp.mytask",
    ...                          "rate_limit": "200/m"}, reply=True)
    [{'worker1.example.com': 'New rate limit set successfully'},
     {'worker2.example.com': 'New rate limit set successfully'},
     {'worker3.example.com': 'New rate limit set successfully'}]

Using the `destination` argument you can specify a list of workers
to receive the command::

    >>> broadcast
    >>> broadcast("rate_limit", {"task_name": "myapp.mytask",
    ...                          "rate_limit": "200/m"}, reply=True,
    ...           destination=["worker1.example.com"])
    [{'worker1.example.com': 'New rate limit set successfully'}]


Of course, using the higher-level interface to set rate limits is much
more convenient, but there are commands that can only be requested
using :func:`~celery.task.control.broadcast`.

.. _worker-rate-limits:

.. control:: rate_limit

Rate limits
-----------

Example changing the rate limit for the `myapp.mytask` task to accept
200 tasks a minute on all servers::

    >>> from celery.task.control import rate_limit
    >>> rate_limit("myapp.mytask", "200/m")

Example changing the rate limit on a single host by specifying the
destination host name::

    >>> rate_limit("myapp.mytask", "200/m",
    ...            destination=["worker1.example.com"])

.. warning::

    This won't affect workers with the
    :setting:`CELERY_DISABLE_RATE_LIMITS` setting on. To re-enable rate limits
    then you have to restart the worker.

.. control:: revoke

Revoking tasks
--------------

All worker nodes keeps a memory of revoked task ids, either in-memory or
persistent on disk (see :ref:`worker-persistent-revokes`).

When a worker receives a revoke request it will skip executing
the task, but it won't terminate an already executing task unless
the `terminate` option is set.

If `terminate` is set the worker child process processing the task
will be terminated.  The default signal sent is `TERM`, but you can
specify this using the `signal` argument.  Signal can be the uppercase name
of any signal defined in the :mod:`signal` module in the Python Standard
Library.

Terminating a task also revokes it.

**Example**

::

    >>> from celery.task.control import revoke
    >>> revoke("d9078da5-9915-40a0-bfa1-392c7bde42ed")

    >>> revoke("d9078da5-9915-40a0-bfa1-392c7bde42ed",
    ...        terminate=True)

    >>> revoke("d9078da5-9915-40a0-bfa1-392c7bde42ed",
    ...        terminate=True, signal="SIGKILL")

.. control:: shutdown

Remote shutdown
---------------

This command will gracefully shut down the worker remotely::

    >>> broadcast("shutdown") # shutdown all workers
    >>> broadcast("shutdown, destination="worker1.example.com")

.. control:: ping

Ping
----

This command requests a ping from alive workers.
The workers reply with the string 'pong', and that's just about it.
It will use the default one second timeout for replies unless you specify
a custom timeout::

    >>> from celery.task.control import ping
    >>> ping(timeout=0.5)
    [{'worker1.example.com': 'pong'},
     {'worker2.example.com': 'pong'},
     {'worker3.example.com': 'pong'}]

:func:`~celery.task.control.ping` also supports the `destination` argument,
so you can specify which workers to ping::

    >>> ping(['worker2.example.com', 'worker3.example.com'])
    [{'worker2.example.com': 'pong'},
     {'worker3.example.com': 'pong'}]

.. _worker-enable-events:

.. control:: enable_events
.. control:: disable_events

Enable/disable events
---------------------

You can enable/disable events by using the `enable_events`,
`disable_events` commands.  This is useful to temporarily monitor
a worker using :program:`celeryev`/:program:`celerymon`.

.. code-block:: python

    >>> broadcast("enable_events")
    >>> broadcast("disable_events")

.. _worker-custom-control-commands:

Writing your own remote control commands
----------------------------------------

Remote control commands are registered in the control panel and
they take a single argument: the current
:class:`~celery.worker.control.ControlDispatch` instance.
From there you have access to the active
:class:`~celery.worker.consumer.Consumer` if needed.

Here's an example control command that restarts the broker connection:

.. code-block:: python

    from celery.worker.control import Panel

    @Panel.register
    def reset_connection(panel):
        panel.logger.critical("Connection reset by remote control.")
        panel.consumer.reset_connection()
        return {"ok": "connection reset"}


These can be added to task modules, or you can keep them in their own module
then import them using the :setting:`CELERY_IMPORTS` setting::

    CELERY_IMPORTS = ("myapp.worker.control", )

.. _worker-inspect:

Inspecting workers
==================

:class:`celery.task.control.inspect` lets you inspect running workers.  It
uses remote control commands under the hood.

.. code-block:: python

    >>> from celery.task.control import inspect

    # Inspect all nodes.
    >>> i = inspect()

    # Specify multiple nodes to inspect.
    >>> i = inspect(["worker1.example.com", "worker2.example.com"])

    # Specify a single node to inspect.
    >>> i = inspect("worker1.example.com")


.. _worker-inspect-registered-tasks:

Dump of registered tasks
------------------------

You can get a list of tasks registered in the worker using the
:meth:`~celery.task.control.inspect.registered`::

    >>> i.registered()
    [{'worker1.example.com': ['celery.delete_expired_task_meta',
                              'celery.execute_remote',
                              'celery.map_async',
                              'celery.ping',
                              'celery.task.http.HttpDispatchTask',
                              'tasks.add',
                              'tasks.sleeptask']}]

.. _worker-inspect-active-tasks:

Dump of currently executing tasks
---------------------------------

You can get a list of active tasks using
:meth:`~celery.task.control.inspect.active`::

    >>> i.active()
    [{'worker1.example.com':
        [{"name": "tasks.sleeptask",
          "id": "32666e9b-809c-41fa-8e93-5ae0c80afbbf",
          "args": "(8,)",
          "kwargs": "{}"}]}]

.. _worker-inspect-eta-schedule:

Dump of scheduled (ETA) tasks
-----------------------------

You can get a list of tasks waiting to be scheduled by using
:meth:`~celery.task.control.inspect.scheduled`::

    >>> i.scheduled()
    [{'worker1.example.com':
        [{"eta": "2010-06-07 09:07:52", "priority": 0,
          "request": {
            "name": "tasks.sleeptask",
            "id": "1a7980ea-8b19-413e-91d2-0b74f3844c4d",
            "args": "[1]",
            "kwargs": "{}"}},
         {"eta": "2010-06-07 09:07:53", "priority": 0,
          "request": {
            "name": "tasks.sleeptask",
            "id": "49661b9a-aa22-4120-94b7-9ee8031d219d",
            "args": "[2]",
            "kwargs": "{}"}}]}]

Note that these are tasks with an eta/countdown argument, not periodic tasks.

.. _worker-inspect-reserved:

Dump of reserved tasks
----------------------

Reserved tasks are tasks that has been received, but is still waiting to be
executed.

You can get a list of these using
:meth:`~celery.task.control.inspect.reserved`::

    >>> i.reserved()
    [{'worker1.example.com':
        [{"name": "tasks.sleeptask",
          "id": "32666e9b-809c-41fa-8e93-5ae0c80afbbf",
          "args": "(8,)",
          "kwargs": "{}"}]}]
