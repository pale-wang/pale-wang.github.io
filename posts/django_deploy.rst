.. date: 2018-06-13 14:37:00
.. tags: django
.. title: Django应用部署
.. type: text

Django应用线上部署路径
======================

使用django-admin的runserver命令很方便就可以在本地进行web开发和测试。部署则需要额外的一些工作，主要步骤包括：

1. 服务器python安装
2. 使用到的python lib安装
3. 前端webserver安装和配置
4. uwsgi配置和安装
5. 服务整体连通性测试


服务器python安装
-----------------

推荐源码包编译，自定义部署路径。原因：

1. 出于内网环境安全的考虑，一般不会给予用户root权限。
2. 方便日后升级等管理工作的进行

使用到的python lib的安装
------------------------

如果内网环境有pip源，则使用pip方式。在pip不可用、内网无法访问外网时，考虑使用wheel方式安装。最后考虑使用lib的源码包进行安装。


webserver安装和配置
-------------------

nginx源码编译。nginx.conf修改，配置static和django用到的location。

uwsgi配置和安装
-----------------
详细步骤参考：`How to use Django with uWSGI`_

uWSGI模型
____________

uWSGI是一个client-server模型。Web Server和django-uwsgi的worker进程交互来提供动态内容。

安装uwsgi
____________
::

    $ pip install uwsgi

uWSGI配置
___________
::

    [uwsgi]
    socket = :8001
    chdir = /path/to/uda_ui
    wsgi-file = uda/wsgi.py
    process = 4
    threads = 2
    max-requests = 5000
    pidfile = /path/to/uda_ui/project-master.pid
    daemonize = /path/to/uda_ui/wsgi.log
    buffer-size = 32678

启动uwsgi
___________
::

    uwsgi --ini uwsgi.ini

服务整体连通性测试
-------------------
在本机进行开发的时候为了便捷，没有考虑使用的组件版本，在利用pip和brew安装的版本跟线上服务器的版本并不一定相同。在进行完上述单机部署的时候，需要对服务器各组件的连通性进行一次整体的排查，保证组件的兼容性。

bad case
_________
nginx的配置已经完成，打开页面发现easyui-ui控件没有数据。在wsgi.log中发现 ::

    File "/home/iknow/Python3.6.2/lib/python3.6/site-packages/pymysql/err.py", line 109, in raise_mysql_exception
        raise errorclass(errno, errval)
    django.db.utils.ProgrammingError: (1064, "You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED' at line 1")
    [pid: 33013|app: 0|req: 2/2] 172.24.191.3 () {50 vars in 2300 bytes} [Tue Jun 12 20:26:07 2018] POST /metadata/metatables => generated 23210 bytes in 519 msecs (HTTP/1.1 500) 4 headers in 145 bytes (1 switches on core 1)

google之，百思不得其解。查看pymysql的版本发现是 ::

    >>> pymysql.__version__
    '0.8.1'

初步怀疑是'SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED'这条命令没办法在线上的MySQL执行，进行验证 ::

    mysql> SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
    Query OK, 0 rows affected (0.00 sec)

是不是pymysql的问题？打开\ `PyMySQL GitHub`_\ 查看安装条件 ::

    Requirements

    Python -- one of the following:
        CPython : 2.7 and >= 3.4
        PyPy : Latest version
    MySQL Server -- one of the following:
        MySQL >= 5.5
        MariaDB >= 5.5


看来需要将MySQL5.1升级MySQL5.5了。

参考资料
--------------
.. [PyMySQL-GITHUB] https://github.com/PyMySQL/PyMySQL
.. [How-to-use-Django-with-wWSGI] https://docs.djangoproject.com/en/2.0/howto/deployment/wsgi/uwsgi/

.. _PyMySQL GitHub: https://github.com/PyMySQL/PyMySQL
.. _How to use Django with uWSGI: https://docs.djangoproject.com/en/2.0/howto/deployment/wsgi/uwsgi/
