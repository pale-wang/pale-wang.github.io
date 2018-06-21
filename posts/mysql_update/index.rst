.. date: 2018-06-14 16:04:00
.. tags: mysql
.. title: MySQL5.1升级5.5
.. type: text
mysql5.1升级5.5
================

背景
-----
在往内网集群部署django应用时 ::

    Traceback (most recent call last):
      File "/home/iknow/Python3.6.2/lib/python3.6/site-packages/django/db/backends/utils.py", line 83, in _execute
        return self.cursor.execute(sql)
      File "/home/iknow/Python3.6.2/lib/python3.6/site-packages/django/db/backends/mysql/base.py", line 71, in execute
        return self.cursor.execute(query, args)
      File "/home/iknow/Python3.6.2/lib/python3.6/site-packages/pymysql/cursors.py", line 170, in execute
        result = self._query(query)
      File "/home/iknow/Python3.6.2/lib/python3.6/site-packages/pymysql/cursors.py", line 328, in _query
        conn.query(q)
      File "/home/iknow/Python3.6.2/lib/python3.6/site-packages/pymysql/connections.py", line 893, in query
        self._affected_rows = self._read_query_result(unbuffered=unbuffered)
      File "/home/iknow/Python3.6.2/lib/python3.6/site-packages/pymysql/connections.py", line 1103, in _read_query_result
        result.read()
      File "/home/iknow/Python3.6.2/lib/python3.6/site-packages/pymysql/connections.py", line 1396, in read
        first_packet = self.connection._read_packet()
      File "/home/iknow/Python3.6.2/lib/python3.6/site-packages/pymysql/connections.py", line 1059, in _read_packet
        packet.check_error()
      File "/home/iknow/Python3.6.2/lib/python3.6/site-packages/pymysql/connections.py", line 384, in check_error
        err.raise_mysql_exception(self._data)
      File "/home/iknow/Python3.6.2/lib/python3.6/site-packages/pymysql/err.py", line 109, in raise_mysql_exception
        raise errorclass(errno, errval)
    pymysql.err.ProgrammingError: (1064, "You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED' at line 1")



MySQL升级路径规划
-----------------
先从MySQL5.1升级到MySQL5.5：

1. 主机LOCK, dump出所有数据，并等待从库同步完成

2. 从库停机

3. 主库升级

4. 启动主库

5. 主库LOCK, dump出所有数据，获取binlog的坐标

6. 从库部署，并建立与主库的同步

5.1老版本启动
__________________
查看机器上5.1版本的进程 ::

    /bin/sh ./mysqld_safe --defaults-file=../my.cnf
    /home/iknow/mysql5/libexec/mysqld --defaults-file=../my.cnf --basedir=/home/iknow/mysql5 --datadir=/home/iknow/mysql5/var --log-error=/home/iknow/mysql5/var/hz01-iknow-udw02.hz01.baidu.com.err --pid-file=/home/iknow/mysql5/var/hz01-iknow-udw02.hz01.baidu.com.pid --socket=/home/iknow/mysql5/tmp/mysql.sock --port=3306

在执行完mysqld_safe后，这个脚本使用mysqld开启真正的服务器进程，同时添加额外的basedir data-dir log-error pid-file socket port参数

导出数据
_________
::

    mysql5/bin/mysqldump -uroot -h 127.0.0.1 --add-drop-table --routines --events --all-databases --force > 2018-06-13-uda-all.sql

服务器关机
___________
::

    mysqladmin -u root -h 127.0.0.1 shutdown

MySQL5.5安装
_______________
安装MySQL5.5

MySQL5.5启动
_______________
首先，需要初始化data目录 ::

    scripts/mysql_install_db --defaults-file=./slave.cnf --user=iknow --datadir=/home/iknow/mysql5.5.60/data

接着启动MySQL5.5 ::

    mysqld_safe --defaults-file=./slave.cnf --user=iknow --datadir=/home/iknow/mysql5.5.60/data  --skip-slave-start

在这个过程中可能会遇到5.5不支持5.1版本option的情况，根据log-error进行修改即可。比如\ ``skip-locking``\ 在在5.5版本中已经被\ ``skip-external-locking``\ 替代 [What-is-New-in-MySQL5.5]_ ，需要进行修改。






当看到如下提示,就意味着MySQL初始化成功了 ::

    Installing MySQL system tables...
    180613 21:43:05 [Note] Ignoring --secure-file-priv value as server is running with --bootstrap.
    180613 21:43:05 [Note] ./bin/mysqld (mysqld 5.5.60) starting as process 30065 ...
    OK
    Filling help tables...
    180613 21:43:05 [Note] Ignoring --secure-file-priv value as server is running with --bootstrap.
    180613 21:43:05 [Note] ./bin/mysqld (mysqld 5.5.60) starting as process 30621 ...
    OK

    To start mysqld at boot time you have to copy
    support-files/mysql.server to the right place for your system

    PLEASE REMEMBER TO SET A PASSWORD FOR THE MySQL root USER !
    To do so, start the server, then issue the following commands:

    ./bin/mysqladmin -u root password 'new-password'
    ./bin/mysqladmin -u root -h hz01-iknow-udw02.hz01.baidu.com password 'new-password'

    Alternatively you can run:
    ./bin/mysql_secure_installation

    which will also give you the option of removing the test
    databases and anonymous user created by default.  This is
    strongly recommended for production servers.

    See the manual for more instructions.

    You can start the MySQL daemon with:
    cd . ; ./bin/mysqld_safe &

    You can test the MySQL daemon with mysql-test-run.pl
    cd ./mysql-test ; perl mysql-test-run.pl

    Please report any problems at http://bugs.mysql.com/

LOCK主库
__________
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000006 |    85746 |              |                  |
+------------------+----------+--------------+------------------+

从库开启同步
______________
使用\ ``CHANGE MASTER TO``\ 来指定开始主库的坐标信息 ::

    CHANGE MASTER TO
        MASTER_HOST='hostname',
        MASTER_USER='username',
        MASTER_PASSWORD='password',
        MASTER_PORT=3306,
        MASTER_LOG_FILE='mysql-bin.000006',
        MASTER_LOG_POS=85746,
        MASTER_CONNECT_RETRY=10;

5.1升级5.5不支持的参数
___________________________
`5.5不支持的master参数`_\ 包括:

- --master-host

- --master-user

- --master-password

- --master-port

- --master-connect-retry

- --master-ssl

- --master-ssl-ca

- --master-ssl-capath

- --master-ssl-cert

- --master-ssl-cipher

- --master-ssl-key

参考资料
-----------
.. [Installing-MySQL-Using-a-Standard-Source-Distribution] https://dev.mysql.com/doc/refman/5.5/en/installing-source-distribution.html
.. [MySQL-Upgrade-Strategies] https://dev.mysql.com/doc/refman/5.5/en/upgrading-strategies.html
.. [Replication-Slave-Options-and-Variables] https://dev.mysql.com/doc/refman/5.5/en/replication-options-slave.html
.. [Change-Master-to-Syntax] https://dev.mysql.com/doc/refman/5.5/en/change-master-to.html
.. [What-is-New-in-MySQL5.5] https://dev.mysql.com/doc/refman/5.5/en/mysql-nutshell.html

.. _5.5不支持的master参数: https://dev.mysql.com/doc/refman/5.5/en/replication-options-slave.html