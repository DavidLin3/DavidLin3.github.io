---
layout:     post
title:      "Redis-py连接重试机制"
subtitle:   "Redis-py reconnecting mechanism"
date:       2019-04-07 20:40:00
author:     "David"
header-img: "img/post-bg-2015.jpg"
tags:
    - Database
---

> “温故而知新”


## 何为连接池

通常情况下，redis操作时会创建一个连接，并基于这个连接进行redis操作，操作完成后，释放连接。

在高并发的时候，频繁的创建和释放连接对性能有较高影响。

此时宜采用连接池。


## 原理

预先创建多个连接，存储于连接池中，当进行Redis操作时，直接获取已经创建的连接进行操作，操作完成后不会释放，用于后续其它Redis操作。

Redis连接池长时间连接后出现重试时间久的问题。

redis-py连接池使用实例

```python
pool = redis.ConnectionPool(host='127.0.0.1', port=6379, password='xxxxx')
redis_client = redis.Redis(connection_pool=pool)
``` 


## 源码解析

当redis.ConnectionPool实例化时，

```python
def __init__(self, connection_class=Connection, max_connections=None, **connection_kwargs):
    max_connections = max_connections or 2 ** 31
    if not isinstance(max_connections, (int, long)) or max_connections < 0:
        raise ValueError('"max_connections" must be a positive integer')

    self.connection_class = connection_class
    self.connection_kwargs = connection_kwargs
    self.max_connections = max_connections

    self.reset()
```

连接池的实例化未做任何真实的redis连接，仅仅设置了最大连接数、连接参数和连接类及重置操作。

重置操作如下：

```python
def reset(self):
    self.pid = os.getpid()
    self._created_connections = 0
    self._available_connections = []
    self._in_use_connections = set()
    self._check_lock = threading.Lock()
```

设置了pid、已连接数、可用连接队列、在用连接集合和线程锁等参数。
那redis.Redis实例化做了什么？

```python
def __init__(self, ...connection_pool=None...):
    if not connection_pool:
    ...
        connection_pool = ConnectionPool(**kwargs)
    self.connection_pool = connection_pool
```

注：以上仅保留关键部分代码。
可以看出，使用Redis，即使不创建连接池，它也会自己创建。
至此，我们还没有看到redis连接真实发生。

到redis真正执行操作时，如llen、lrange等，执行如下：
```python
def llen(self, name):
    return self.execute_command('LLEN', name)
```

execute_command执行什么呢
```python
def execute_command(self, *args, **options):
    "Execute a command and return a parsed response"
    pool = self.connection_pool
    command_name = args[0]
    connection = pool.get_connection(command_name, **options)
    try:
        connection.send_command(*args)
        return self.parse_response(connection, command_name, **options)
    except (ConnectionError, TimeoutError) as e:
        connection.disconnect()
        if not connection.retry_on_timeout and isinstance(e, TimeoutError):
            raise
        connection.send_command(*args)
        return self.parse_response(connection, command_name, **options)
    finally:
        pool.release(connection)
```

至此，我们看到了连接的获取过程。
```python
connection = pool.get_connection(command_name, **options)
```

这里调用的是ConnectionPool类的get_connection方法
```python
def get_connection(self, command_name, *keys, **options):
    "Get a connection from the pool"
    self._checkpid()
    try:
        connection = self._available_connections.pop()
    except IndexError:
        connection = self.make_connection()
    self._in_use_connections.add(connection)
    return connection
```

如果有可用的连接, 获取可用的连接；如果没有, 创建一个新连接。
```python
def make_connection(self):
    "Create a new connection"
    if self._created_connections >= self.max_connections:
        raise ConnectionError("Too many connections")
    self._created_connections += 1
    return self.connection_class(**self.connection_kwargs)
```

终于，我们看到了连接的创建。
在ConnectionPool的实例中，有两个list，依次是_available_connections和 _in_use_connections，
分别表示可用的连接集合和正在使用的连接集合, 在上面的get_connection中, 我们可以看到获取连接的过程是:

- 从可用的连接集合中尝试获取连接
- 如果获取不到，创建新连接
- 将新连接加到正在使用的连接集合中

上面是连接从可用连接集合中取出，添加到使用中的连接集合的过程，那什么时候将正在使用的连接放回到可用连接集合中的呢

在execute_command里，我们可以看到在执行redis操作时，在finally部分, 会执行pool.release操作
```python
pool.release(connection)
```

在release操作中，将连接从正在使用的连接集合中删除，添加到可用的连接集合中
```python
def release(self, connection):
    "Releases the connection back to the pool"
    self._checkpid()
    if connection.pid != self.pid:
        return
    self._in_use_connections.remove(connection)
    self._available_connections.append(connection)
```
以上即是连接池的实现。

那创建的连接是如何实现的呢
```python
def make_connection(self):
    ...
    return self.connection_class(**self.connection_kwargs)
```
self.connection_class为Connection，Connection类通过TCP协议实现redis连接。
```python
class Connection(object):
    "Manages TCP communication to and from a Redis server"
    def __init__(self, ... , socket_keepalive=False, socket_keepalive_options=None, ...):
        ...
        self.socket_keepalive = socket_keepalive
        self.socket_keepalive_options = socket_keepalive_options or {}
        ...
```
连接的建立方法是通过socket模块实现：
```python
def _connect(self):
    "Create a TCP socket connection"
    # socket.connect()
    err = None
    for res in socket.getaddrinfo(self.host, self.port, 0,
                                  socket.SOCK_STREAM):
        family, socktype, proto, canonname, socket_address = res
        sock = None
        try:
            sock = socket.socket(family, socktype, proto)
            # TCP_NODELAY
            sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
            # TCP_KEEPALIVE
            if self.socket_keepalive:
                sock.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
                for k, v in iteritems(self.socket_keepalive_options):
                    sock.setsockopt(socket.SOL_TCP, k, v)
            # set the socket_connect_timeout before we connect
            sock.settimeout(self.socket_connect_timeout)
            # connect
            sock.connect(socket_address)
            # set the socket_timeout now that we're connected
            sock.settimeout(self.socket_timeout)
            return sock
        except socket.error as _:
            err = _
            if sock is not None:
                sock.close()
    if err is not None:
        raise err
    raise socket.error("socket.getaddrinfo returned an empty list")
```

默认self.socket_keepalive为False，建立的连接为短连接。若希望建立长连接，保证连接的长期有效，则应该设置socket_keepalive和socket_keepalive_options，具体设置可参考[Example 14](https://www.programcreek.com/python/example/101860/socket.TCP_KEEPINTVL)
注：Example 14的设置适用于linux系统的python环境，不适用于windows。


## 参考资料

1. [TCP心跳连接](http://hengyunabc.github.io/why-we-need-heartbeat/)
2. [redis连接池原理](https://www.u3v3.com/ar/1346)
