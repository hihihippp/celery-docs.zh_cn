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

不推荐使用数据库作为消息队列，但在非常小的应用中可以使用。Celery 可以使用 SQLAlchemy 和 Django ORMS。请查看 :ref:`otherqueues-sqlalchemy` 或 :ref:`otherqueues-django`。

* 更多其他选择。

除了上面列出的以外，还有其他可以选择的传输实现，例如 CouchDB, Beanstalk, MongoDB, and SQS。请查阅 Kombu 的文档以便获得更多信息。

.. _`RabbitMQ`: http://www.rabbitmq.com/
.. _`Redis`: http://redis.io/

.. _celerytut-simple-tasks:

创建一个简单“任务”（Task）
======================

在这个教程里，我们将创建一个简单的“任务”（Task） —— 把两个数加起来。通常，我们在 Python 的模块中定义“任务”。

按照惯例，我们将调用模块 :file:`tasks.py`，看起来会像这个样子：

:file: `tasks.py`

.. code-block:: python

    from celery.task import task

    @task
    def add(x, y):
        return x + y


此时， `@task` 装饰器实际上创建了一个继承自 :class:`~celery.task.base.Task` 的“类”（class）。除非需要修改“任务类”的缺省行为，否则我们推荐只通过装饰器定义“任务”（这是我们推崇的最佳实践）。

.. seealso::

    关于创建任务和任务类的完整文档可以在 :doc:`../userguide/tasks` 中找到。

.. _celerytut-conf:

配置
=============

Celery 使用一个配置模块来进行配置。这个模块缺省北命名为 :file:`celeryconfig.py`。

为了能被 import，这个配置模块要么存在于当前目录，要么包含在 Python 路径中。

同时，你可以通过使用环境变量 :envvar:`CELERY_CONFIG_MODULE` 来随意修改这个配置文件的名字。

现在来让我们创建配置文件 :file:`celeryconfig.py`.

1. 配置如何连接 broker（例子中我们使用 RabbitMQ）::

        BROKER_URL = "amqp://guest:guest@localhost:5672//"

2. 定义用于存储元数据（metadata）和返回值（return values）的后端::

        CELERY_RESULT_BACKEND = "amqp"

   AMQP 后端缺省是非持久化的，你只能取一次结果（一条消息）。

   可以阅读 :ref:`conf-result-backend` 了解可以使用的后端清单和相关参数。

3. 最后，我们列出 worker 需要 import 的模块，包括你的任务。

   我们只有一个刚开始添加的任务模块 :file:`tasks.py`::

        CELERY_IMPORTS = ("tasks", )

这就行了。

你还有更多的选项可以使用，例如：你期望使用多少个进程来并行处理（:setting:`CELERY_CONCURRENCY` 设置），或者使用持久化的结果保存后端。可以阅读 :ref:`configuration` 查看更多的选项。

.. note::

    你可以使用选项 :mod:`~celery.bin.celeryd`:: 的 :option:`-I` 来指定要加载的模块
    

        $ celeryd -l info -I tasks,handlers

    可以在 :program:`celeryd` 启动时，指定加载单个或加载多个模块（通过 , 分隔的模块清单来实现）。


.. _celerytut-running-celeryd:

运行 worker 服务器
================================

为了方便测试，我们将在前台运行 worker 服务器，这样我们就能在终端上看到 celery 上发生的事情::

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
