.. _guide-tasks:

=======
 任务
=======

.. contents::
    :local:


本文主要介绍如何定义任务的全貌。如果希望了解任务的全部属性和方法，请阅读 :class:`API reference <celery.task.base.BaseTask>`。

.. _task-basics:

基础
======

任务是一个封装了功能和执行选项的类（class）。
函数 `create_user` 有两个参数： `username` 和 `password`，你可以这样定义一个任务：

.. code-block:: python

    from django.contrib.auth import User

    from celery.task import task

    @task
    def create_user(username, password):
        User.objects.create(username=username, password=password)


任务属性也可以作为参数添加到任务 `task`:

.. code-block:: python

    @task(serializer="json")
    def create_user(username, password):
        User.objects.create(username=username, password=password)

.. _task-request-info:

上下文
=======

`task.request` 中包含了当前正在执行的任务的相关信息和状态，其中包含了以下这些属性：

:id: 正任务的唯一 id 。

:taskset: 这个任务所在的（所有）任务组的唯一 id 编号。

:args: 位置参数（Positional arguments）。

:kwargs: 参数的关键字。

:retries: 重试次数，一个大于 `0` 的整数。

:is_eager: 如果被设置成 :const:`True`，则这个任务被 client 执行，而不是 worker。

:logfile: 记录日志的文件。详情请见 `Logging`_。

:loglevel: 日志级别。

:hostname: 执行这个任务的 worker 主机名。

:delivery_info: 额外的消息投递信息。这是用于映射到 exchange 和 routing key，用于投递消息的信息。例如 e.g. :meth:`~celery.task.base.BaseTask.retry` 向同一个目标队列重新发送任务消息。

  **注意** 有一些消息后端并没有高级路由的能力，因此你不能默认这里的映射是有效的。


使用示例
-------------

::

    from celery.task import task

    @task
    def add(x, y):
        print("Executing task id %r, args: %r kwargs: %r" % (
            add.request.id, add.request.args, add.request.kwargs))

.. _task-logging:

日志
=======

可以使用 worker 日志器向 worker 日志中增加添加诊断信息：

.. code-block:: python

    @task
    def add(x, y):
        logger = add.get_logger()
        logger.info("Adding %s + %s" % (x, y))
        return x + y

有多个日志级别可以使用，worker 的 `loglevel` 设置决定了什么信息会被记录到 log 文件中。

当然，你也可以通过使用 `print` 来向标准输出/标准错误输出任何东西，当然这些也都会被记录到日志文件中、

.. _task-retry:

出现错误时重新执行任务
==================================

使用 :meth:`~celery.task.base.BaseTask.retry` 重新发送任务。它会按照 :attr:`~celery.task.base.BaseTask.max_retries` 属性的设置正确地工作：

.. code-block:: python

    @task
    def send_twitter_status(oauth, tweet):
        try:
            twitter = Twitter(oauth)
            twitter.update_status(tweet)
        except (Twitter.FailWhaleError, Twitter.LoginError), exc:
            send_twitter_status.retry(exc=exc)

在上面这个例子里面，我们使用了 `exc` 这个参数把当前的异常传递到 :meth:`~celery.task.base.BaseTask.retry` 中。这些异常都可以作为重试的里程碑使用，直到 :attr:`~celery.task.base.BaseTask.max_retries` 所设置的异常上限被超过。但如果没有传递 `exc` 参数，:exc:`~celery.exceptions.RetryTaskError` 异常将会产生。

.. note::

    :meth:`retry` 将会产生一个异常，因此任何在 retry 之后的代码都不会被执行。这个异常就是 :exc:`celery.exceptions.RetryTaskError`，它不被作为错误进行处理，但可以作为任务被重试的一个标志。

    这是缺省的执行方法，除非重试的 ``throw`` 参数被设置为 :const:`False`。

.. _task-retry-custom-delay:

定制重试延迟
--------------------------

当任务被重试时，任务将会等待一段时间。缺省的任务重试延迟时间是通过 :attr:`~celery.task.base.BaseTask.default_retry_delay` 属性设置的。缺省间隔是 3 分钟。注意，这里所使用的单位是秒（整数或浮点数）。

你可以使用 `countdown` 参数来覆盖 :meth:`~celery.task.base.BaseTask.retry` 这个缺省值。

.. code-block:: python

    @task(default_retry_delay=30 * 60)  # retry in 30 minutes.
    def add(x, y):
        try:
            ...
        except Exception, exc:
            add.retry(exc=exc, countdown=60)  # override the default and
                                              # retry in 1 minute

.. _task-options:

任务参数
============

一般参数
-------

.. _task-general-options:

.. attribute:: Task.name

    任务注册所用的名称。

    你可以手动的设置名称，或者使用由模块自动生成的类名（缺省值）。

    详情参见 :ref:`task-names`。

.. attribute Task.request

    If the task is being executed this will contain information
    about the current request.  Thread local storage is used.

    详情参见 :ref:`task-request-info`。

.. attribute:: Task.abstract

    抽象类没有被注册，可以用来作为新的任务类型的基类。

.. attribute:: Task.max_retries

    任务执行时的最大重试次数。到达最大次数后，将会产生一个 :exc:`~celery.exceptions.MaxRetriesExceeded` 的异常。

    *注意：* 你必须手工设置 :meth:`retry`，这个不是缺省开启的。

.. attribute:: Task.default_retry_delay

    缺省下达重试任务间隔的时间（以秒计）。可以是 :class:`int` 或 :class:`float`，缺省是 3 分钟。

.. attribute:: Task.rate_limit

    设置任务的速率（指定时间内允许执行的次数）。

    设置成 :const:`None` 代表为不做任何速度限制。
    如果设置为整数，将被作为 "tasks per second" 解读。

    可以用每秒（/s），每分钟（/m）或每小时（/h）的方式来设置速度。
    例如: `"100/m"` （每分钟 100 个任务）。缺省被设置为 :setting:`CELERY_DEFAULT_RATE_LIMIT` 中的设置，如果没有指定，则意味着缺省没有任务速率设置。

.. attribute:: Task.time_limit

    任务执行的“硬”超时时间。如果没被设置，则使用 worker 的缺省值。

.. attribute:: Task.soft_time_limit

    任务执行的“软”超时时间。如果没被设置，则使用 worker 的缺省值。

.. attribute:: Task.ignore_result

    不保存任务状态。这也就意味着你不能使用 :class:`~celery.result.AsyncResult` 检查任务是否已经就绪，或是否有返回值。

.. attribute:: Task.store_errors_even_if_ignored

    如果是 :const:`True`，即使 Task.ignore_result 被配置，错误信息也将被保留。

.. attribute:: Task.send_error_emails

    当任务出错时，发送电子邮件。
    缺省将发送到 :setting:`CELERY_SEND_TASK_ERROR_EMAILS` 设置的值。
    请参阅 :ref:`conf-error-mails`。

.. attribute:: Task.error_whitelist

    如果设置了发送错误邮件，但在此清单中的错误异常将不会发送邮件。

.. attribute:: Task.serializer

    设置所使用的序列化方式。缺省会使用 :setting:`CELERY_TASK_SERIALIZER` 的设置。可以被设置为`pickle`、`json`、`yaml` 或任何在 :mod:`kombu.serialization.registry` 已注册的序列化方法。

    请参阅 :ref:`executing-serializers`。

.. attribute:: Task.backend

    保存任务结果的后端。缺省使用 :setting:`CELERY_RESULT_BACKEND` 中的设置。

.. attribute:: Task.acks_late

    如果被设置为 :const:`True` ，任务被执行之 **后** 确认，而不是缺省的执行之 *前* 被确认。

    则意味着如果 worker 在执行过程中出错，任务将被执行两次，这对于某些应用来说是可以接受的。

    全局设置中 :setting:`CELERY_ACKS_LATE` 设置可以被覆盖。

.. attribute:: Task.track_started

    如果被设置为 :const:`True`，任务将在被 worker 执行时报告状态为 "started" 。
    缺省值为 :const:`False`，缺省将不报告这个粒度的状态。任务有 pending、finished 或 waiting to be retried 几种状态。增加 "started" 状态，对于执行时间长或需要汇报正在运行的任务都非常有用。

    主机名和执行这个任务的 worker 的 pid 都将被保存到元数据中（例如：`result.info["pid"]`）

    全局设置中 :setting:`CELERY_TRACK_STARTED` 设置可以被覆盖。


.. seealso::

    API 手册 :class:`~celery.task.base.BaseTask`。

.. _task-message-options:

消息和路由选项
---------------------------

.. attribute:: Task.queue

    使用 :setting:`CELERY_QUEUES` 中所设置队列的路由设置。如果设置了此项，:attr:`exchange` 和 :attr:`routing_key` 选项将被忽略。

.. attribute:: Task.exchange

    覆盖全局设置中 `exchange` 的值。

.. attribute:: Task.routing_key

    覆盖全局设置中缺省 `routing_key` 的值。

.. attribute:: Task.mandatory

    如果设置了此项，任务消息将被强制路由。缺省设置时，如果 broker 无法把消息路由到某个队列，此消息将被地球。然而，一旦任务被设置为强制，系统将会产生一个异常。

    amqplib 不支持此特性。

.. attribute:: Task.immediate

    实时投递。如果一个任务不能及时的被路由到 worker，将会产生一个异常。如果不设置此项，队列将会接收并缓存此任务，但并不承诺此任务是否被执行。

    amqplib 不支持此特性。

.. attribute:: Task.priority

    消息的优先级。0-9 的数字，0 代表最高优先级。

    RabbitMQ 不支持此特性。

.. seealso::

    请参阅 :ref:`executing-routing` 和 :ref:`guide-routing` 获得更多关于消息的选项。

.. _task-names:

任务名称
==========

任务通过 *task name* 来唯一标识。如果没有提供，则将由系统自动使用模块名和类名生成。例如：

.. code-block:: python

    >>> @task(name="sum-of-two-numbers")
    >>> def add(x, y):
    ...     return x + y

    >>> add.name
    'sum-of-two-numbers'

使用模块名作为前缀来分类任务使用的命名空间是推荐的最佳实践。这种方法不会导致和其他的模块名称发生碰撞：

.. code-block:: python

    >>> @task(name="tasks.add")
    >>> def add(x, y):
    ...     return x + y

    >>> add.name
    'tasks.add'


这也是缺省的自动生成任务名称的方法（如果这个模块的名字是 "tasks.py"）：

.. code-block:: python

    >>> @task()
    >>> def add(x, y):
    ...     return x + y

    >>> add.name
    'tasks.add'

.. _task-naming-relative-imports:

自动命名和相对引用
-------------------------------------

相对引用和自动命名不能很好的一起工作，如果你使用了相对引用就需要更加严格的进行命名。

在客户端中引用了 "myapp.tasks" 模块中的 ".tasks"，并且在 worker 中引用了 "myapp.tasks" 模块。此时，自动生成的名称就不会匹配，将会导致 worker 产生一个 :exc:`~celery.exceptions.NotRegistered` 的异常。

这常常在 Django 中发生。当 Django 激活 `project.myapp`::

    INSTALLED_APPS = ("project.myapp", )

Worker 将会自动注册为 "project.myapp.tasks.*"，而在客户端中将被注册为 "myapp.tasks"：

.. code-block:: python

    >>> from myapp.tasks import add
    >>> add.name
    'myapp.tasks.add'

因此，请永远不要使用 "project.app"，而应该把项目目录添加到 Python 目录::

    import os
    import sys
    sys.path.append(os.getcwd())

    INSTALLED_APPS = ("myapp", )

从可重复使用的角度来看，这种方式更透明、也更有意义。

.. _tasks-decorating:

装饰任务（Decorating tasks）
================

当使用其他装饰器时，请注意确保 `task` 是最后一个起作用的装饰器:

.. code-block:: python

    @task
    @decorator2
    @decorator1
    def add(x, y):
        return x + y


也就意味着 `@task` 装饰器必须在声明的最顶端。

.. _task-states:

任务状态
===========

Celery 能够跟踪任务状态，包括成功任务的结果、失败任务的异常和回溯信息。

有多个 *result backends* 可以选择，各自有不同的优缺点（可参考 :ref:`task-result-backends`）。

During its lifetime a task will transition through several possible states,
and each state may have arbitrary metadata attached to it.  When a task
moves into a new state the previous state is
forgotten about, but some transitions can be deducted, (e.g. a task now
in the :state:`FAILED` state, is implied to have been in the
:state:`STARTED` state at some point).

There are also sets of states, like the set of
:state:`failure states <FAILURE_STATES>`, and the set of
:state:`ready states <READY_STATES>`.

The client uses the membership of these sets to decide whether
the exception should be re-raised (:state:`PROPAGATE_STATES`), or whether
the result can be cached (it can if the task is ready).

You can also define :ref:`custom-states`.

.. _task-result-backends:

Result Backends
---------------

Celery needs to store or send the states somewhere.  There are several
built-in backends to choose from: SQLAlchemy/Django ORM, Memcached, Redis,
AMQP, MongoDB, Tokyo Tyrant and Redis -- or you can define your own.

No backend works well for every use case.
You should read about the strengths and weaknesses of each backend, and choose
the most appropriate for your needs.


.. seealso::

    :ref:`conf-result-backend`

AMQP Result Backend
~~~~~~~~~~~~~~~~~~~

The AMQP result backend is special as it does not actually *store* the states,
but rather sends them as messages.  This is an important difference as it
means that a result *can only be retrieved once*; If you have two processes
waiting for the same result, one of the processes will never receive the
result!

Even with that limitation, it is an excellent choice if you need to receive
state changes in real-time.  Using messaging means the client does not have to
poll for new states.

There are several other pitfalls you should be aware of when using the AMQP
backend:

* Every new task creates a new queue on the server, with thousands of tasks
  the broker may be overloaded with queues and this will affect performance in
  negative ways. If you're using RabbitMQ then each queue will be a separate
  Erlang process, so if you're planning to keep many results simultaneously you
  may have to increase the Erlang process limit, and the maximum number of file
  descriptors your OS allows.

* Old results will not be cleaned automatically, so you must make sure to
  consume the results or else the number of queues will eventually go out of
  control.  If you're running RabbitMQ 2.1.1 or higher you can take advantage
  of the ``x-expires`` argument to queues, which will expire queues after a
  certain time limit after they are unused.  The queue expiry can be set (in
  seconds) by the :setting:`CELERY_TASK_RESULT_EXPIRES` setting (not
  enabled by default).

For a list of options supported by the AMQP result backend, please see
:ref:`conf-amqp-result-backend`.


Database Result Backend
~~~~~~~~~~~~~~~~~~~~~~~

Keeping state in the database can be convenient for many, especially for
web applications with a database already in place, but it also comes with
limitations.

* Polling the database for new states is expensive, and so you should
  increase the polling intervals of operations such as `result.wait()`, and
  `tasksetresult.join()`

* Some databases uses a default transaction isolation level that
  is not suitable for polling tables for changes.

  In MySQL the default transaction isolation level is `REPEATABLE-READ`, which
  means the transaction will not see changes by other transactions until the
  transaction is committed.  It is recommended that you change to the
  `READ-COMMITTED` isolation level.


.. _task-builtin-states:

Built-in States
---------------

.. state:: PENDING

PENDING
~~~~~~~

Task is waiting for execution or unknown.
Any task id that is not know is implied to be in the pending state.

.. state:: STARTED

STARTED
~~~~~~~

Task has been started.
Not reported by default, to enable please see :attr:`Task.track_started`.

:metadata: `pid` and `hostname` of the worker process executing
           the task.

.. state:: SUCCESS

SUCCESS
~~~~~~~

Task has been successfully executed.

:metadata: `result` contains the return value of the task.
:propagates: Yes
:ready: Yes

.. state:: FAILURE

FAILURE
~~~~~~~

Task execution resulted in failure.

:metadata: `result` contains the exception occurred, and `traceback`
           contains the backtrace of the stack at the point when the
           exception was raised.
:propagates: Yes

.. state:: RETRY

RETRY
~~~~~

Task is being retried.

:metadata: `result` contains the exception that caused the retry,
           and `traceback` contains the backtrace of the stack at the point
           when the exceptions was raised.
:propagates: No

.. state:: REVOKED

REVOKED
~~~~~~~

Task has been revoked.

:propagates: Yes

.. _custom-states:

Custom states
-------------

You can easily define your own states, all you need is a unique name.
The name of the state is usually an uppercase string.  As an example
you could have a look at :mod:`abortable tasks <~celery.contrib.abortable>`
which defines its own custom :state:`ABORTED` state.

Use :meth:`Task.update_state <celery.task.base.BaseTask.update_state>` to
update a tasks state::

    @task
    def upload_files(filenames):
        for i, file in enumerate(filenames):
            upload_files.update_state(state="PROGRESS",
                meta={"current": i, "total": len(filenames)})


Here we created the state `"PROGRESS"`, which tells any application
aware of this state that the task is currently in progress, and also where
it is in the process by having `current` and `total` counts as part of the
state metadata.  This can then be used to create e.g. progress bars.

.. _pickling_exceptions:

Creating pickleable exceptions
------------------------------

A little known Python fact is that exceptions must behave a certain
way to support being pickled.

Tasks that raises exceptions that are not pickleable will not work
properly when Pickle is used as the serializer.

To make sure that your exceptions are pickleable the exception
*MUST* provide the original arguments it was instantiated
with in its ``.args`` attribute.  The simplest way
to ensure this is to have the exception call ``Exception.__init__``.

Let's look at some examples that work, and one that doesn't:

.. code-block:: python


    # OK:
    class HttpError(Exception):
        pass

    # BAD:
    class HttpError(Exception):

        def __init__(self, status_code):
            self.status_code = status_code

    # OK:
    class HttpError(Exception):

        def __init__(self, status_code):
            self.status_code = status_code
            Exception.__init__(self, status_code)  # <-- REQUIRED


So the rule is:
For any exception that supports custom arguments ``*args``,
``Exception.__init__(self, *args)`` must be used.

There is no special support for *keyword arguments*, so if you
want to preserve keyword arguments when the exception is unpickled
you have to pass them as regular args:

.. code-block:: python

    class HttpError(Exception):

        def __init__(self, status_code, headers=None, body=None):
            self.status_code = status_code
            self.headers = headers
            self.body = body

            super(Exception, self).__init__(status_code, headers, body)

.. _task-custom-classes:

创建定制的任务类
============================

所有的任务类都继承自 :class:`celery.task.Task` 这个类，都具有 :meth:`run` 这个方法。

下面的代码，

.. code-block:: python

    @task
    def add(x, y):
        return x + y


背后大致是这样的场景：

.. code-block:: python

    @task
    def AddTask(Task):

        def run(self, x, y):
            return x + y
    add = registry.tasks[AddTask.name]


实例化
-------------

任务 **不会** 每次请求实例化，但会在全局实例中注册。

这意味着 ``__init__`` 构造函数只可以被一个进程所调用，任务类（task class）从语义上来说，更类似于 actor。

如果你有这样一个任务，

.. code-block:: python

    class NaiveAuthenticateServer(Task):

        def __init__(self):
            self.users = {"george": "password"}

        def run(self, username, password):
            try:
                return self.users[username] == password
            except KeyError:
                return False

如果你把每个请求都路由到同一个进程，系统会在不同请求间保存状态。

这对于保存缓存的资源来说，非常有用::

    class DatabaseTask(Task):
        _db = None

        @property
        def db(self):
            if self._db = None:
                self._db = Database.connect()
            return self._db

抽象类
----------------

抽象类没有注册，但被用作为新任务类型的基类。

.. code-block:: python

    class DebugTask(Task):
        abstract = True

        def after_return(self, \*args, \*\*kwargs):
            print("Task returned: %r" % (self.request, ))


    @task(base=DebugTask)
    def add(x, y):
        return x + y


处理程序
--------

.. method:: execute(self, request, pool, loglevel, logfile, \*\*kw):

    :param request: A :class:`~celery.worker.job.TaskRequest`.
    :param pool: 任务池。
    :param loglevel: 当前的日志级别。
    :param logfile: 当前使用的日志文件。

    :keyword consumer: The :class:`~celery.worker.consumer.Consumer`.

.. method:: after_return(self, status, retval, task_id, args, kwargs, einfo)

    任务返回后将会调用处理程序（Handler）。

    :param status: 当前任务状态。
    :param retval: 任务返回值/异常。
    :param task_id: 此任务的唯一标识。
    :param args: 失败任务的原始参数。
    :param kwargs: 失败任务的原始参数名称。

    :keyword einfo: :class:`~celery.datastructures.ExceptionInfo`
                    instance, containing the traceback (if any).

    处理程序的返回值将被忽略。

.. method:: on_failure(self, exc, task_id, args, kwargs, einfo)

    任务失败时 worker 会运行此处理程序.

    :param exc: The exception raised by the task.
    :param task_id: Unique id of the failed task.
    :param args: Original arguments for the task that failed.
    :param kwargs: Original keyword arguments for the task
                       that failed.

    :keyword einfo: :class:`~celery.datastructures.ExceptionInfo`
                           instance, containing the traceback.

    处理程序的返回值将被忽略。

.. method:: on_retry(self, exc, task_id, args, kwargs, einfo)

    This is run by the worker when the task is to be retried.

    :param exc: The exception sent to :meth:`retry`.
    :param task_id: Unique id of the retried task.
    :param args: Original arguments for the retried task.
    :param kwargs: Original keyword arguments for the retried task.

    :keyword einfo: :class:`~celery.datastructures.ExceptionInfo`
                    instance, containing the traceback.

    处理程序的返回值将被忽略。

.. method:: on_success(self, retval, task_id, args, kwargs)

    Run by the worker if the task executes successfully.

    :param retval: The return value of the task.
    :param task_id: Unique id of the executed task.
    :param args: Original arguments for the executed task.
    :param kwargs: Original keyword arguments for the executed task.

    处理程序的返回值将被忽略。

on_retry
~~~~~~~~

.. _task-how-they-work:

How it works
============

Here comes the technical details, this part isn't something you need to know,
but you may be interested.

All defined tasks are listed in a registry.  The registry contains
a list of task names and their task classes.  You can investigate this registry
yourself:

.. code-block:: python

    >>> from celery import registry
    >>> from celery import task
    >>> registry.tasks
    {'celery.delete_expired_task_meta':
        <PeriodicTask: celery.delete_expired_task_meta (periodic)>,
     'celery.task.http.HttpDispatchTask':
        <Task: celery.task.http.HttpDispatchTask (regular)>,
     'celery.execute_remote':
        <Task: celery.execute_remote (regular)>,
     'celery.map_async':
        <Task: celery.map_async (regular)>,
     'celery.ping':
        <Task: celery.ping (regular)>}

This is the list of tasks built-in to celery.  Note that we had to import
`celery.task` first for these to show up.  This is because the tasks will
only be registered when the module they are defined in is imported.

The default loader imports any modules listed in the
:setting:`CELERY_IMPORTS` setting.

The entity responsible for registering your task in the registry is a
meta class, :class:`~celery.task.base.TaskType`.  This is the default
meta class for :class:`~celery.task.base.BaseTask`.

If you want to register your task manually you can mark the
task as :attr:`~celery.task.base.BaseTask.abstract`:

.. code-block:: python

    class MyTask(Task):
        abstract = True

This way the task won't be registered, but any task inheriting from
it will be.

When tasks are sent, we don't send any actual function code, just the name
of the task to execute.  When the worker then receives the message it can look
up the name in its task registry to find the execution code.

This means that your workers should always be updated with the same software
as the client.  This is a drawback, but the alternative is a technical
challenge that has yet to be solved.

.. _task-best-practices:

Tips and Best Practices
=======================

.. _task-ignore_results:

Ignore results you don't want
-----------------------------

If you don't care about the results of a task, be sure to set the
:attr:`~celery.task.base.BaseTask.ignore_result` option, as storing results
wastes time and resources.

.. code-block:: python

    @task(ignore_result=True)
    def mytask(...)
        something()

Results can even be disabled globally using the :setting:`CELERY_IGNORE_RESULT`
setting.

.. _task-disable-rate-limits:

Disable rate limits if they're not used
---------------------------------------

Disabling rate limits altogether is recommended if you don't have
any tasks using them.  This is because the rate limit subsystem introduces
quite a lot of complexity.

Set the :setting:`CELERY_DISABLE_RATE_LIMITS` setting to globally disable
rate limits:

.. code-block:: python

    CELERY_DISABLE_RATE_LIMITS = True

.. _task-synchronous-subtasks:

Avoid launching synchronous subtasks
------------------------------------

Having a task wait for the result of another task is really inefficient,
and may even cause a deadlock if the worker pool is exhausted.

Make your design asynchronous instead, for example by using *callbacks*.

**Bad**:

.. code-block:: python

    @task
    def update_page_info(url):
        page = fetch_page.delay(url).get()
        info = parse_page.delay(url, page).get()
        store_page_info.delay(url, info)

    @task
    def fetch_page(url):
        return myhttplib.get(url)

    @task
    def parse_page(url, page):
        return myparser.parse_document(page)

    @task
    def store_page_info(url, info):
        return PageInfo.objects.create(url, info)


**Good**:

.. code-block:: python

    @task(ignore_result=True)
    def update_page_info(url):
        # fetch_page -> parse_page -> store_page
        fetch_page.delay(url, callback=subtask(parse_page,
                                    callback=subtask(store_page_info)))

    @task(ignore_result=True)
    def fetch_page(url, callback=None):
        page = myhttplib.get(url)
        if callback:
            # The callback may have been serialized with JSON,
            # so best practice is to convert the subtask dict back
            # into a subtask object.
            subtask(callback).delay(url, page)

    @task(ignore_result=True)
    def parse_page(url, page, callback=None):
        info = myparser.parse_document(page)
        if callback:
            subtask(callback).delay(url, info)

    @task(ignore_result=True)
    def store_page_info(url, info):
        PageInfo.objects.create(url, info)


We use :class:`~celery.task.sets.subtask` here to safely pass
around the callback task.  :class:`~celery.task.sets.subtask` is a
subclass of dict used to wrap the arguments and execution options
for a single task invocation.


.. seealso::

    :ref:`sets-subtasks` for more information about subtasks.

.. _task-performance-and-strategies:

性能和策略
==========================

.. _task-granularity:

粒度
-----------

The task granularity is the amount of computation needed by each subtask.
In general it is better to split the problem up into many small tasks, than
have a few long running tasks.

With smaller tasks you can process more tasks in parallel and the tasks
won't run long enough to block the worker from processing other waiting tasks.

However, executing a task does have overhead. A message needs to be sent, data
may not be local, etc. So if the tasks are too fine-grained the additional
overhead may not be worth it in the end.

.. seealso::

    The book `Art of Concurrency`_ has a whole section dedicated to the topic
    of task granularity.

.. _`Art of Concurrency`: http://oreilly.com/catalog/9780596521547

.. _task-data-locality:

数据局部性
-------------

The worker processing the task should be as close to the data as
possible.  The best would be to have a copy in memory, the worst would be a
full transfer from another continent.

If the data is far away, you could try to run another worker at location, or
if that's not possible - cache often used data, or preload data you know
is going to be used.

The easiest way to share data between workers is to use a distributed cache
system, like `memcached`_.

.. seealso::

    The paper `Distributed Computing Economics`_ by Jim Gray is an excellent
    introduction to the topic of data locality.

.. _`Distributed Computing Economics`:
    http://research.microsoft.com/pubs/70001/tr-2003-24.pdf

.. _`memcached`: http://memcached.org/

.. _task-state:

状态
-----

Since celery is a distributed system, you can't know in which process, or
on what machine the task will be executed.  You can't even know if the task will
run in a timely manner.

The ancient async sayings tells us that “asserting the world is the
responsibility of the task”.  What this means is that the world view may
have changed since the task was requested, so the task is responsible for
making sure the world is how it should be;  If you have a task
that re-indexes a search engine, and the search engine should only be
re-indexed at maximum every 5 minutes, then it must be the tasks
responsibility to assert that, not the callers.

Another gotcha is Django model objects.  They shouldn't be passed on as
arguments to tasks.  It's almost always better to re-fetch the object from
the database when the task is running instead,  as using old data may lead
to race conditions.

Imagine the following scenario where you have an article and a task
that automatically expands some abbreviations in it:

.. code-block:: python

    class Article(models.Model):
        title = models.CharField()
        body = models.TextField()

    @task
    def expand_abbreviations(article):
        article.body.replace("MyCorp", "My Corporation")
        article.save()

First, an author creates an article and saves it, then the author
clicks on a button that initiates the abbreviation task.

    >>> article = Article.objects.get(id=102)
    >>> expand_abbreviations.delay(model_object)

Now, the queue is very busy, so the task won't be run for another 2 minutes.
In the meantime another author makes changes to the article, so
when the task is finally run, the body of the article is reverted to the old
version because the task had the old body in its argument.

Fixing the race condition is easy, just use the article id instead, and
re-fetch the article in the task body:

.. code-block:: python

    @task
    def expand_abbreviations(article_id):
        article = Article.objects.get(id=article_id)
        article.body.replace("MyCorp", "My Corporation")
        article.save()

    >>> expand_abbreviations(article_id)

There might even be performance benefits to this approach, as sending large
messages may be expensive.

.. _task-database-transactions:

Database transactions
---------------------

Let's have a look at another example:

.. code-block:: python

    from django.db import transaction

    @transaction.commit_on_success
    def create_article(request):
        article = Article.objects.create(....)
        expand_abbreviations.delay(article.pk)

This is a Django view creating an article object in the database,
then passing the primary key to a task.  It uses the `commit_on_success`
decorator, which will commit the transaction when the view returns, or
roll back if the view raises an exception.

There is a race condition if the task starts executing
before the transaction has been committed; The database object does not exist
yet!

The solution is to *always commit transactions before sending tasks
depending on state from the current transaction*:

.. code-block:: python

    @transaction.commit_manually
    def create_article(request):
        try:
            article = Article.objects.create(...)
        except:
            transaction.rollback()
            raise
        else:
            transaction.commit()
            expand_abbreviations.delay(article.pk)

.. _task-example:

Example
=======

Let's take a real wold example; A blog where comments posted needs to be
filtered for spam.  When the comment is created, the spam filter runs in the
background, so the user doesn't have to wait for it to finish.

We have a Django blog application allowing comments
on blog posts.  We'll describe parts of the models/views and tasks for this
application.

blog/models.py
--------------

The comment model looks like this:

.. code-block:: python

    from django.db import models
    from django.utils.translation import ugettext_lazy as _


    class Comment(models.Model):
        name = models.CharField(_("name"), max_length=64)
        email_address = models.EmailField(_("email address"))
        homepage = models.URLField(_("home page"),
                                   blank=True, verify_exists=False)
        comment = models.TextField(_("comment"))
        pub_date = models.DateTimeField(_("Published date"),
                                        editable=False, auto_add_now=True)
        is_spam = models.BooleanField(_("spam?"),
                                      default=False, editable=False)

        class Meta:
            verbose_name = _("comment")
            verbose_name_plural = _("comments")


In the view where the comment is posted, we first write the comment
to the database, then we launch the spam filter task in the background.

.. _task-example-blog-views:

blog/views.py
-------------

.. code-block:: python

    from django import forms
    from django.http import HttpResponseRedirect
    from django.template.context import RequestContext
    from django.shortcuts import get_object_or_404, render_to_response

    from blog import tasks
    from blog.models import Comment


    class CommentForm(forms.ModelForm):

        class Meta:
            model = Comment


    def add_comment(request, slug, template_name="comments/create.html"):
        post = get_object_or_404(Entry, slug=slug)
        remote_addr = request.META.get("REMOTE_ADDR")

        if request.method == "post":
            form = CommentForm(request.POST, request.FILES)
            if form.is_valid():
                comment = form.save()
                # Check spam asynchronously.
                tasks.spam_filter.delay(comment_id=comment.id,
                                        remote_addr=remote_addr)
                return HttpResponseRedirect(post.get_absolute_url())
        else:
            form = CommentForm()

        context = RequestContext(request, {"form": form})
        return render_to_response(template_name, context_instance=context)


To filter spam in comments we use `Akismet`_, the service
used to filter spam in comments posted to the free weblog platform
`Wordpress`.  `Akismet`_ is free for personal use, but for commercial use you
need to pay.  You have to sign up to their service to get an API key.

To make API calls to `Akismet`_ we use the `akismet.py`_ library written by
`Michael Foord`_.

.. _task-example-blog-tasks:

blog/tasks.py
-------------

.. code-block:: python

    from akismet import Akismet
    from celery.task import task

    from django.core.exceptions import ImproperlyConfigured
    from django.contrib.sites.models import Site

    from blog.models import Comment


    @task
    def spam_filter(comment_id, remote_addr=None):
        logger = spam_filter.get_logger()
        logger.info("Running spam filter for comment %s" % comment_id)

        comment = Comment.objects.get(pk=comment_id)
        current_domain = Site.objects.get_current().domain
        akismet = Akismet(settings.AKISMET_KEY, "http://%s" % domain)
        if not akismet.verify_key():
            raise ImproperlyConfigured("Invalid AKISMET_KEY")


        is_spam = akismet.comment_check(user_ip=remote_addr,
                            comment_content=comment.comment,
                            comment_author=comment.name,
                            comment_author_email=comment.email_address)
        if is_spam:
            comment.is_spam = True
            comment.save()

        return is_spam

.. _`Akismet`: http://akismet.com/faq/
.. _`akismet.py`: http://www.voidspace.org.uk/downloads/akismet.py
.. _`Michael Foord`: http://www.voidspace.org.uk/
