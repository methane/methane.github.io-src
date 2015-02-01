---
title: "Welcome to PyMySQL, Mike Bayer (@zzzeek)!"
date: "2015-02-01"
categories:
    - "python"
    - "MySQL"
---

There are three major MySQL Driver in Python: MySQL-python, PyMySQL and MySQL-Connector/Python.

Since MySQL-python's development is slowdown, I've made fork named "mysqlclient-python".  It supports Python 3 and fixed some bugs.

I'm also only active maintainer of PyMySQL. It means I maintains 2/3 of Python-MySQL world.
But I have several interest other than MySQL. I have 9-month old baby. Additionally, I'm not good at writing English.
So I'm not a good maintainer. Python/MySQL world is stopped compared with Python/PostgreSQL.

But last weekend, Mike Bayer (@zzzeek, SQLAlchemy developer) posted one mail to PyMySQL Users ML.
It said OpenStack developers considering about moving from MySQL-python to PyMySQL, and ask a question about can I review and accept pull requests from Mike and OpenStack developers.

The answer is ... No!  I will not have enough time.

But this is a huge chance to me and Python/MySQL world. SQLAlchemy is one of greatest O/R mapper in Python. OpenStack is one of most active project in Python. I invited him to PyMySQL Organization on Github and he agreed readily.

Despite I'm not a good document/test writer, I have some experience of high performance Python and asynchronous I/O programming.
So I want to improve PyMySQL with Mike and OpenStack.

On the other hand, I have some ideas about improving mysqlclient-python: Support new async APIs in MariaDB's connectors, use ctypes or cffi to improve performance on PyPy. But I have enough time. So I decided not doing them in foreseeable future. It doesn't means mysqlclient-python is deprecated. It will be stable MySQL driver as it has so far.
