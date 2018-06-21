.. date: 2018-06-03
.. tags: python
.. title: 开始使用wheel
.. type: text


Wheels入门
===========

wheels是什么
------------

造轮子这个词应该都不会陌生，wheels可以理解为python提供给用户的轮子。轮子多、易用也成就了是python受欢迎现状。

wheels是新的python分发标准
--------------------------

一个个轮子虽然有了，但是要怎么才能拿过来用是另外一件事情。此前Python包的分发是混乱的，有了wheel之后这种现象正逐步得到改观。从 `pythonwheels`_ 的数据中可以看到，截止[2018/06/03]pypi上下载量前360名的包已经有263个支持wheel工具,也就是说73.06%的包已经wheel。现在是时候使用wheel来武装自己了！

wheel的使用前提
----------------

::

    pip >= 1.4
    setuptools >= 0.8

延伸
-----

从pythonwheels上抓取前360位的包名称。

.. _pythonwheels: https://pythonwheels.com/