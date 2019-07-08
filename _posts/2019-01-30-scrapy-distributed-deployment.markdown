---
layout:     post
title:      "Scrapy分布式部署"
subtitle:   "Scrapy Distributed Deployment"
date:       2019-01-30
author:     "David"
header-img: "img/tag-bg.jpg"
tags:
    - 爬虫
---


### 引入scrapy-redis，实现分布式

scrapy框架自身不支持分布式，需要借助scrapy-redis。scrapy-redis是分布式爬虫通用的简单框架。
原理是：用scrapy-redis替代scrapy中的原有队列，存在redis中，多台爬虫机器共享redis里面的队列，从而实现分布式爬虫效果。


### scrapy-redis用法
    - 在master机器上安装redis
    - 在scrapy爬虫机器（slaver）上安装scrapy-redis，命令为：
```python
pip install scrapy-redis
```
    - 在settings.py中进行如下设置，任务调度工作scrapy-redis已经帮我们实现了
```python
# settings.py

SCHEDULER = "scrapy_redis.scheduler.Scheduler"
# 使用scrapy-redis里的调度器对所有爬虫机器统一调度，替换scrapy里的调度器

DUPEFILTER_CLASS = "scrapy_redis.dupefilter.RFPDupeFilter"
# 使用scrapy-redis中的去重组件，代替scrapy中的去重队列

REDIS_HOST = "127.0.0.1"
REDIS_PORT = 6379
REDIS_PARAMS = {
'password': 'longqi'
}
# 设置redis配置，不要密码可注释掉
```

以上部分通常需要设置，下面的可选择性设置
```python
ITEM_PIPELINES = {
'scrapy_redis.pipelines.RedisPipeline': 300
}
# 将scrapy中的item存入redis中

SCHEDULER_PERSIST = True
# 爬虫结束后不会清空redis中的队列（去重队列），方便断点续爬

SCHEDULER_QUEUE_CLASS = "scrapy_redis.queue.PriorityQueue"
# 默认，不需要设置，优先级排序队列。其他方式如下：
# 先进先出队列：
# SCHEDULER_QUEUE_CLASS = "scrapy_redis.queue.FifoQueue"
# 后进先出队列：
# SCHEDULER_QUEUE_CLASS = "scrapy_redis.queue.LifoQueue"
```
    - 在每个爬虫机器（slaver）上启动scrapy即可。


### scrapyd实现分布式部署

scrapyd是部署scrapy分布式爬虫的工具，爬虫机器只需安装scrapyd的web服务，远程客户端就可通过web接口在这台爬虫机器上部署scrapy代码。访问scrapyd对应的url查看scrapy运行状态和日志。
scrapyd安装：
```python
pip install scrapyd
```
安装完成后，需要修改scrapyd配置文件，更改默认绑定地址，开放给外网访问：
```python
# /etc/scrapyd/scrapyd.conf
bind_address = 0.0.0.0
```

启动scrapyd：
```python
scrapyd
```
scrapyd默认端口为：6800

![scrapyd](/img/in-post/scrapy-distributed-deployment/scrapyd.jpg)
<small class="img-hint">scrapyd界面</small>


部署scrapy代码
首先安装scrapyd-client：
```python
pip install scrapyd-client
```

测试是否安装成功：
```python
scrapyd-deploy -h
```
然后在scrapy项目中的scrapy.cfg文件中，设置如下：
![deploy-cfg](/img/in-post/scrapy-distributed-deployment/deploy1.jpg)
<small class="img-hint">scrapy.cfg文件</small>

设置完成后，用命令行切换到scrapy项目所在路径，用如下命令部署：
![deploy-cfg](/img/in-post/scrapy-distributed-deployment/deploy2.jpg)
<small class="img-hint">scrapyd部署</small>

### 设置scrapy版本
```python
# scrapy.cfg

[deploy:Zarten]
url = http://192.168.2.227:6800/
project = scrapy_redis_test
version = r1
```

### 控制scrapy状态
安装scrapydAPI：
```python
pip install python-scrapyd-api
```
用法：
```python
from scrapyd_api import ScrapydAPI
scrapyd_obj = ScrapydAPI('http://localhost:6800')
```
生成一个ScrapydAPI对象，提供了如下几种API方法：
1. schedule(project, spider, settings=None, **kwargs)，远程启动爬虫机器的scrapy
参数说明：
    - project：scrapy工程名称
    - spider：爬虫名称
    - settings：dict，重写scrapy中的settings
    - kwargs：自定义额外参数传递给scrapy的spider的init函数
    - _version：str类型，指定版本号
```python
scrapyd_obj.schedule('freq_word', 'keyword_freq', _version='r1', keyword_id= keyword_id, search_keyword= keyword)
```

__init__函数如下所示：
```python
def __init__(self, keyword_id, search_keyword, **kwargs):
    self.keyword_id = keyword_id
    self.search_keyword = search_keyword
```
返回值：启动爬虫后的job id

2. cancel(project, job, signal=None)，取消（停止）正在或准备运行的爬虫
参数说明：
project：scrapy工程名称
job：爬虫的job_id
signal：终止的信号，一般为None
返回值：爬虫停止前的状态

注：有时运行中的爬虫不管用API还是命令行都无法停止，采用直接杀死进程的解决方案：
    - 查看spider的pid
    - 进入爬虫机器，如果是docker容器运行，执行命令：
```python
docker exec -it 容器id kill -9 pid
```

3. delete_project(project)，删除爬虫项目
参数说明：
project：scrapy工程名称
返回值：成功返回True，否则返回False


### docker内运行scrapyd

不使用docker：需要在每台爬虫机器上安装scrapyd服务以及爬虫运行所需要的库
使用docker：将scrapyd以及其他依赖库制作成docker镜像再上传到docker hub，每台机器pull镜像然后运行容器就可以

![docker-scrapyd](/img/in-post/scrapy-distributed-deployment/requirements.png)
<small class="img-hint">docker镜像所需文件</small>

**scrapyd.conf**
用该配置文件替换原来的配置文件：
```python
[scrapyd]
eggs_dir    = eggs
logs_dir    = logs
items_dir   =
jobs_to_keep = 5
dbs_dir     = dbs
max_proc    = 0
max_proc_per_cpu = 10
finished_to_keep = 100
poll_interval = 5.0
bind_address = 0.0.0.0
http_port   = 6800
debug       = off
runner      = scrapyd.runner
application = scrapyd.app.application
launcher    = scrapyd.launcher.Launcher
webroot     = scrapyd.website.Root

[services]
schedule.json     = scrapyd.webservice.Schedule
cancel.json       = scrapyd.webservice.Cancel
addversion.json   = scrapyd.webservice.AddVersion
listprojects.json = scrapyd.webservice.ListProjects
listversions.json = scrapyd.webservice.ListVersions
listspiders.json  = scrapyd.webservice.ListSpiders
delproject.json   = scrapyd.webservice.DeleteProject
delversion.json   = scrapyd.webservice.DeleteVersion
listjobs.json     = scrapyd.webservice.ListJobs
daemonstatus.json = scrapyd.webservice.DaemonStatus
```

**Dockerfile**
```python
FROM python:3.6
ADD . /code
WORKDIR /code
COPY ./scrapyd.conf /etc/scrapyd/
EXPOSE 6800
RUN pip3 install -r requirements.txt
CMD scrapyd
```

**requirements.txt**
这里放一些项目所需的库，如果需要更通用点，尽可能多放一些常用的库，这样可以部署更多的项目到scrapyd中。若某个项目中使用的库在docker中找不到，那岂不是要再重新制作docker+scrapyd了？
最后就是制作docker镜像

问题：已经制作好的docker镜像中缺少某些依赖库，但又不想重新制作镜像，怎么办？
方案：执行如下docker命令
```python
docker exec -it 容器pid pip install ***
```


### 参考

1. [What is a Python egg?](https://stackoverflow.com/questions/2051192/what-is-a-python-egg)
2. [Install MongoDB Community Edition on Ubuntu](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/#run-mongodb-community-edition)
3. [如何在Ubuntu 18.04 LTS上安装和配置MongoDB](https://www.linuxidc.com/Linux/2018-05/152253.htm)
4. [MongoDB出现错误Error: Failed to execute "listdatabases" command](https://blog.bccn.net/qq1135909556/65409)
5. [Scrapy-redis和Scrapyd用法详解](https://zhuanlan.zhihu.com/p/44564597)

