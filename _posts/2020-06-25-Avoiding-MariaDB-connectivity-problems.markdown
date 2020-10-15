---
title: "Modifying MariaDB process limits - Avoiding connectivity problems"
layout: post
date: 2020-06-25 00:00
headerImage: false
tag:
- mysql
- mariadb
- linux
- troubleshooting
category: blog
author: t0rrex
description: Solving a problem found connecting to a MariaDB instance
---

# Description:

Few time ago a researcher reports me that their server with a MariaDB instance was refusing their connections. After some tests with unsuccesful results, we found a workaround modifing the process limits to grant again connectivity to the instance.

# Procedure

* Problem found trying to connect to a MariaDB instance:

{% highlight bash %}
root@linux:# telnet sql-node1 3306
Trying 192.168.0.10...
Connected to sql-node1.
Escape character is '^]'.
Connection closed by foreign host.
{% endhighlight %}

* First of all we check the logs and we found that there are some lines tagged as an errors:

{% highlight bash %}
2020-05-10 16:20:23 140123042092800 [ERROR] Invalid (old?) table or database name 'mysql-files'
2020-05-10 16:20:57 140123756480704 [ERROR] Error in accept: Too many open files
2020-05-10 16:25:13 140123756480704 [ERROR] Error in accept: Too many open filesb' user: 'phyReader' host: 'node1' (Got timeout reading communication packets)
{% endhighlight %}

* For diagnose the problem we connect to the mysql server to check one variable:

{% highlight bash %}
sql-node1:# mysql -u root -p

MariaDB [(none)]> SHOW VARIABLES LIKE 'open%';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| open_files_limit | 4186  |
+------------------+-------+
1 row in set (0.03 sec)

MariaDB [(none)]> quit
Bye
{% endhighlight %}

* Reading across the internet we found that the problem derives from the limits of the process.

<http://blogs.reliablepenguin.com/2015/08/28/mariadb-on-centos-7-error-in-accept-too-many-open-files>

* The method to diagnose it, is looking on the limits configuration for the process that runs the mariadb instance.

* First, we obtain the PID of the process:

{% highlight bash %}
sql-node1:# ps -elaf | grep mysql
4 S mysql     6502     1  0  80   0 - 518769 SyS_po 08:50 ?       00:00:00 /usr/sbin/mysqld --defaults-file=/etc/my.cnf --user=mysql
4 S root      6555  5797  0  80   0 -  1858 -      08:52 pts/0    00:00:00 grep --color=auto mysql
{% endhighlight %}

* After, we show the limits state of the mysql process inside the /proc directory:

{% highlight bash %}
sql-node1:# cat /proc/6502/limits
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            8388608              unlimited            bytes
Max core file size        unlimited            unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             14219                14219                processes
Max open files            4186                 4186                 files
Max locked memory         65536                65536                bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       14219                14219                signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
Max realtime timeout      unlimited            unlimited            us
{% endhighlight %}

* The interesting field is the Max Open Files, that in our case is:

{% highlight bash %}
Max open files            4186                 4186                 files
{% endhighlight %}

* We need to change this limit. This change is applied inside the unit file for systemd.

* We open the file to edit it. It's recommendable to make before a copy of the file:

{% highlight bash %}
sql-node1:# vim /usr/lib/systemd/system/mariadb.service
{% endhighlight %}

* In the section [Service] we add this line:

{% highlight bash %}
LimitNOFILE=infinity
{% endhighlight %}

* After, we need to reload the systemd daemon:

{% highlight bash %}
sql-node1:# systemctl daemon-reload
{% endhighlight %}

* And we restart the mysql service:

{% highlight bash %}
sql-node1:# systemctl restart mysql
{% endhighlight %}

* After the change, we need to obtain the new PID of the service:

{% highlight bash %}
sql-node1:# ps -elaf | grep mysql
4 S mysql     6630     1  3  80   0 - 452932 SyS_po 08:53 ?       00:00:00 /usr/sbin/mysqld --defaults-file=/etc/my.cnf --user=mysql
4 S root      6673  5797  0  80   0 -  1858 -      08:53 pts/0    00:00:00 grep --color=auto mysql
{% endhighlight %}

* And, if we read again the limits configuration of the process, we should see the changes:

{% highlight bash %}
sql-node1:# cat /proc/6630/limits
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            8388608              unlimited            bytes
Max core file size        unlimited            unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             14219                14219                processes
Max open files            1048576              1048576              files
Max locked memory         65536                65536                bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       14219                14219                signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
Max realtime timeout      unlimited            unlimited            us
{% endhighlight %}

* Next, we try to connect again to the service, and we obtain a succesful connection:

{% highlight bash %}
root@linux:# telnet sql-node1 3306
Trying 192.168.0.10...
Connected to sql-node1.
Escape character is '^]'.
{% endhighlight %}
