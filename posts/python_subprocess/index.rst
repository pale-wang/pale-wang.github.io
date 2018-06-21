.. title: Python 3 subprocess 新高层API
.. date: 2018-06-09
.. type: text
.. tag: python

subprocess模块
================
subprocess模块用来创建新的进程，连接到它的标准输入/标准输出/错误输出。


使用subprocess
---------------
标准库官方文档的推荐使用run()来处理它能应对的所有功能。当需要更高级的功能的时候，直接使用底层的Popen接口。run()是3.5加入的，如果为了保持兼容性还是可以继续使用老的高层API.

subprocess.run(args, \*, stdin=None, input=None, stdout=None, stdin=None, shell=False, cwd=None, timeout=None,
check=False, encoding=None, errors=None, env=None)

run函数执行args描述的命令，等待子进程完成，返回一个CompletedProcess实例。

完整的参数列表与Popen基本一致，除了timeout, input 和 check, 其他所有的参数都被传递给Popen

默认不捕获stdout和stderr，如果想要捕获，需要传递为stdout/stderr传递PIPE参数。

timeout参数将传递给Popen.communicate().如果超时了，子进程将会被杀死，父进程等在子进程死亡，并且在之后抛出TimeoutExpired

input参数也传递给Popen.communicate(), 也就是子进程的stdin. input必须是byte序列，在制定了encoding或者errors或者universal_newlines 是true的情况下，可以是字符串。在使用的时候，内部的Popen对象已经自动使用stdin=PIPE创建，因此stdin=PIPE可以不需要。

如果check is true, 子进程退出的返回值是非0，那么会抛出CalledProcessError 异常。异常的数值中包含有参数，退出值，stdin stdout的值。

如果声明了encoding或errors或universal_newlines is true,;stdin, stdout, stderr使用特定的encoding和error的text模式打开。其他情况下使用二进制模式打开。

如果env不是None, 那它必须是定义环境变量的一个映射。这些环境变量被传递给子进程，而不是继承父进程的环境变量。

3.5 新引入的
3.6 添加了encoding和errors的支持

*class* subprocess.**CompletedProcess**\

    run()函数的返回值，代表一个已经结束执行的进程。

    **args**
        创建进程传递进来的参数。必须是一个list或者string

    **returncode**
        进程的返回值。通常0代表正常退出。在POSIX下，负数返回值-N,代表进程因signal N结束。

    **stdout**
        捕获子进程的标准输出。是一个byte sequence,如果run()执行时指定了encoding或者error, 返回string. 没有捕获到返回None.

        如果run()的时候stderr=subprocess.STDOUT,那么子进程的标准错误输出会重定向到标准输出。stderr将会是None.

    **stderr**
        捕获子进程的标准错误输出。是一个byte sequence,如果run()执行时指定了encoding或者error, 返回string. 没有捕获到返回None.

    **check_returncode()**
        如果returncode不是0，抛出CalledProcessError

*exception* subprocess.\ **SubprocessError**

  subprocess模块所有异常的基类

  3.3 新引入的

*exception* subprocess.\ **TimeoutExpired**

    SubprocessError的子类。代表子进程超时错误。

*exception* subprocess.\ **CalledProcessError**

    **cmd**

    **timeout**

    **output**

    **stdout**

    **stderr**

    3.5 新增stdout stderr属性

SubprocessError的子类，check_call()或check_output()的返回值非0时抛出该异常。

    **returncode**

    **cmd**

    **output**

    **stdout**

    **stderr**

    3.5 新引入的

替代老的高层API
---------------

Python 3.5前的高层API由三个函数组成。虽然可以使用run()取代它们，但是很多老代码仍然使用这三个函数。

subprocess.\ **call**\(args, \*, stdin=None, stdout=None, stderr=None, shell=False, cwd=None, timeout=None)

    等同于
    run(...).returncode

subprocess.\ **check_call**\(args, \*, stdin=None, stdout=None, stderr=None, shell=False, cwd=None, timeout=None)

    等同于
    run(..., check=True)

subprocess.\ **check_output**\(args, \*, stdin=None, stdout=None, stderr=None, shell=False, cwd=None, encoding=None, errors=None, universal_newlines=False, timeout=None)

    等同于
    run(..., check=True, stdout=PIPE).stdout


我的意见
---------
暂时不会对老代码中的subprocess往新的API迁移，就方便记忆这点来说老API更容易使用。

参考资料
---------
Python3标准库subprocess: https://docs.python.org/3/library/subprocess.html