.. _tut-celery:

========================
 开始 Celery 的第一步
========================

.. contents::
    :local:

.. _celerytut-broker:

选择你的 Broker
====================

在你正式开始使用 Celery 之前，你需要选择、安装并运行一个 broker。
Broker 是一种负责接收、发送任务消息（task messages）的服务

你可以从以下几种 broker 中选择一个：

* `RabbitMQ`_

功能完整、安全、饱经实践检验的 broker。如果对你而言，不丢失消息非常重要，RabbitMQ 将是你最好的选择。

请查看 :ref:`broker-installation` 以便获得更多关于 RabbitMQ 安装和配置相关的信息。

* `Redis`_

也是一个功能完整的 broker，但电源故障或异常将导致数据丢失。

请查看 :ref:`otherqueues-redis` 以便配置相关的信息。

* 数据库

不推荐使用数据库作为消息队列，但在非常小的应用中可以使用。Celery 可以使用 SQLAlchemy 和 Django ORMS。
请查看 :ref:`otherqueues-sqlalchemy` 或 :ref:`otherqueues-django`。

* 更多其他选择。

除了上面列出的以外，还有其他可以选择的传输实现，例如 CouchDB, Beanstalk, MongoDB, and SQS。请查阅 Kombu 的文档以便获得更多信息。

.. _`RabbitMQ`: http://www.rabbitmq.com/
.. _`Redis`: http://redis.io/

.. _celerytut-simple-tasks:

创建一个简单任务（Task）
======================

在这个教程里，我们将创建一个简单的任务（Task） —— 把两个数加起来。通常，我们在 Python 的模块中定义任务。

按照惯例，我们将调用我们的模块 :file:`tasks.py`，看起来会像这个样子：

:file: `tasks.py`

.. code-block:: python

    from celery.task import task

    @task
    def add(x, y):
        return x + y


Behind the scenes the `@task` decorator actually creates a class that
inherits from :class:`~celery.task.base.Task`.  The best practice is to
only create custom task classes when you want to change generic behavior,
and use the decorator to define tasks.

.. seealso::

    The full documentation on how to create tasks and task classes is in the
    :doc:`../userguide/tasks` part of the user guide.

.. _celerytut-conf:

Configuration
=============

Celery is configured by using a configuration module.  By default
this module is called :file:`celeryconfig.py`.

The configuration module must either be in the current directory
or on the Python path, so that it can be imported.

You can also set a custom name for the configuration module by using
the :envvar:`CELERY_CONFIG_MODULE` environment variable.

Let's create our :file:`celeryconfig.py`.

1. Configure how we communicate with the broker (RabbitMQ in this example)::

        BROKER_URL = "amqp://guest:guest@localhost:5672//"

2. Define the backend used to store task metadata and return values::

        CELERY_RESULT_BACKEND = "amqp"

   The AMQP backend is non-persistent by default, and you can only
   fetch the result of a task once (as it's sent as a message).

   For list of backends available and related options see
   :ref:`conf-result-backend`.

3. Finally we list the modules the worker should import.  This includes
   the modules containing your tasks.

   We only have a single task module, :file:`tasks.py`, which we added earlier::

        CELERY_IMPORTS = ("tasks", )

That's it.

There are more options available, like how many processes you want to
use to process work in parallel (the :setting:`CELERY_CONCURRENCY` setting),
and we could use a persistent result store backend, but for now, this should
do.  For all of the options available, see :ref:`configuration`.

.. note::

    You can also specify modules to import using the :option:`-I` option to
    :mod:`~celery.bin.celeryd`::

        $ celeryd -l info -I tasks,handlers

    This can be a single, or a comma separated list of task modules to import
    when :program:`celeryd` starts.


.. _celerytut-running-celeryd:

Running the celery worker server
================================

To test we will run the worker server in the foreground, so we can
see what's going on in the terminal::

    $ celeryd --loglevel=INFO

In production you will probably want to run the worker in the
background as a daemon.  To do this you need to use the tools provided
by your platform, or something like `supervisord`_ (see :ref:`daemonizing`
for more information).

For a complete listing of the command line options available, do::

    $  celeryd --help

.. _`supervisord`: http://supervisord.org

.. _celerytut-executing-task:

Executing the task
==================

Whenever we want to execute our task, we use the
:meth:`~celery.task.base.Task.delay` method of the task class.

This is a handy shortcut to the :meth:`~celery.task.base.Task.apply_async`
method which gives greater control of the task execution (see
:ref:`guide-executing`).

    >>> from tasks import add
    >>> add.delay(4, 4)
    <AsyncResult: 889143a6-39a2-4e52-837b-d80d33efb22d>

At this point, the task has been sent to the message broker. The message
broker will hold on to the task until a worker server has consumed and
executed it.

Right now we have to check the worker log files to know what happened
with the task.  Applying a task returns an
:class:`~celery.result.AsyncResult`, if you have configured a result store
the :class:`~celery.result.AsyncResult` enables you to check the state of
the task, wait for the task to finish, get its return value
or exception/traceback if the task failed, and more.

Keeping Results
---------------

If you want to keep track of the tasks state, Celery needs to store or send
the states somewhere.  There are several
built-in backends to choose from: SQLAlchemy/Django ORM, Memcached, Redis,
AMQP, MongoDB, Tokyo Tyrant and Redis -- or you can define your own.

For this example we will use the `amqp` result backend, which sends states
as messages.  The backend is configured via the ``CELERY_RESULT_BACKEND``
option, in addition individual result backends may have additional settings
you can configure::

    CELERY_RESULT_BACKEND = "amqp"

    #: We want the results to expire in 5 minutes, note that this requires
    #: RabbitMQ version 2.1.1 or higher, so please comment out if you have
    #: an earlier version.
    CELERY_TASK_RESULT_EXPIRES = 300

To read more about result backends please see :ref:`task-result-backends`.

Now with the result backend configured, let's execute the task again.
This time we'll hold on to the :class:`~celery.result.AsyncResult`::

    >>> result = add.delay(4, 4)

Here's some examples of what you can do when you have results::

    >>> result.ready() # returns True if the task has finished processing.
    False

    >>> result.result # task is not ready, so no return value yet.
    None

    >>> result.get()   # Waits until the task is done and returns the retval.
    8

    >>> result.result # direct access to result, doesn't re-raise errors.
    8

    >>> result.successful() # returns True if the task didn't end in failure.
    True

If the task raises an exception, the return value of `result.successful()`
will be :const:`False`, and `result.result` will contain the exception instance
raised by the task.

Where to go from here
=====================

After this you should read the :ref:`guide`. Specifically
:ref:`guide-tasks` and :ref:`guide-executing`.
