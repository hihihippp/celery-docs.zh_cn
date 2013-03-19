.. _broker-installation:

=====================
 安装 Broker
=====================

.. contents::
    :local:

.. _installing-rabbitmq:

安装 RabbitMQ
===================

请在 RabbitMQ 的网站上查阅 `Installing RabbitMQ`_ 的内容. 如果在 Mac OS X 上安装，请查阅 `Installing RabbitMQ on OS X`_.

.. _`Installing RabbitMQ`: http://www.rabbitmq.com/install.html

.. note::

    如果你在安装后运行 :program:`rabbitmqctl` 程序遇到了 `nodedown` 错误，
    请查阅下面这篇 blog 文章。这篇文章可以帮助你找到问题的根源:

        http://somic.org/2009/02/19/on-rabbitmqctl-and-badrpcnodedown/

.. _rabbitmq-configuration:

配置 RabbitMQ
===================

在使用 celery 前，你需要创建一个 RabbbitMQ 用户、一个虚拟主机（vhost），并且允许用户访问这个虚拟主机::

    $ rabbitmqctl add_user myuser mypassword

    $ rabbitmqctl add_vhost myvhost

    $ rabbitmqctl set_permissions -p myvhost myuser ".*" ".*" ".*"

请查阅 RabbitMQ 的 `Admin Guide`_ 并查阅 `access control`_ 相关的内容.

.. _`Admin Guide`: http://www.rabbitmq.com/admin-guide.html

.. _`access control`: http://www.rabbitmq.com/admin-guide.html#access-control

.. _rabbitmq-osx-installation:

Installing RabbitMQ on OS X
===========================

The easiest way to install RabbitMQ on Snow Leopard is using `Homebrew`_; the new
and shiny package management system for OS X.

In this example we'll install Homebrew into :file:`/lol`, but you can
choose whichever destination, even in your home directory if you want, as one of
the strengths of Homebrew is that it's relocatable.

Homebrew is actually a `git`_ repository, so to install Homebrew, you first need to
install git. Download and install from the disk image at
http://code.google.com/p/git-osx-installer/downloads/list?can=3

When git is installed you can finally clone the repository, storing it at the
:file:`/lol` location::

    $ git clone git://github.com/mxcl/homebrew /lol


Brew comes with a simple utility called :program:`brew`, used to install, remove and
query packages. To use it you first have to add it to :envvar:`PATH`, by
adding the following line to the end of your :file:`~/.profile`::

    export PATH="/lol/bin:/lol/sbin:$PATH"

Save your profile and reload it::

    $ source ~/.profile


Finally, we can install rabbitmq using :program:`brew`::

    $ brew install rabbitmq


.. _`Homebrew`: http://github.com/mxcl/homebrew/
.. _`git`: http://git-scm.org

.. _rabbitmq-osx-system-hostname:

Configuring the system host name
--------------------------------

If you're using a DHCP server that is giving you a random host name, you need
to permanently configure the host name. This is because RabbitMQ uses the host name
to communicate with nodes.

Use the :program:`scutil` command to permanently set your host name::

    sudo scutil --set HostName myhost.local

Then add that host name to :file:`/etc/hosts` so it's possible to resolve it
back into an IP address::

    127.0.0.1       localhost myhost myhost.local

If you start the rabbitmq server, your rabbit node should now be `rabbit@myhost`,
as verified by :program:`rabbitmqctl`::

    $ sudo rabbitmqctl status
    Status of node rabbit@myhost ...
    [{running_applications,[{rabbit,"RabbitMQ","1.7.1"},
                        {mnesia,"MNESIA  CXC 138 12","4.4.12"},
                        {os_mon,"CPO  CXC 138 46","2.2.4"},
                        {sasl,"SASL  CXC 138 11","2.1.8"},
                        {stdlib,"ERTS  CXC 138 10","1.16.4"},
                        {kernel,"ERTS  CXC 138 10","2.13.4"}]},
    {nodes,[rabbit@myhost]},
    {running_nodes,[rabbit@myhost]}]
    ...done.

This is especially important if your DHCP server gives you a host name
starting with an IP address, (e.g. `23.10.112.31.comcast.net`), because
then RabbitMQ will try to use `rabbit@23`, which is an illegal host name.

.. _rabbitmq-osx-start-stop:

Starting/Stopping the RabbitMQ server
-------------------------------------

To start the server::

    $ sudo rabbitmq-server

you can also run it in the background by adding the :option:`-detached` option
(note: only one dash)::

    $ sudo rabbitmq-server -detached

Never use :program:`kill` to stop the RabbitMQ server, but rather use the
:program:`rabbitmqctl` command::

    $ sudo rabbitmqctl stop

When the server is running, you can continue reading `Setting up RabbitMQ`_.

