---
layout:     post
title:      "Python日志配置与使用详解"
subtitle:   "Python Logging Configuration"
date:       2020-06-13 17:30:00
author:     "David"
header-img: "img/post-bg-2015.jpg"
tags:
    - Python
---

> “Life is short, you need Python”


## 前言

软件开发不止要完成需求的功能，还要添加日志记录功能，方便开发过程中的debug和之后项目上线运行过程中记录各种动态数据变化，为运维人员提供维护帮助。当然在开发过程中我们可以简单用print函数打印出信息至控制台来debug，但是上生产线再用print函数就太简陋和不规范了。

所幸Python有logging标准库支持日志，功能非常强大，几乎所有的开源项目都用到它。


## 原理

这里涉及到几个概念：LogRecord、log filter、handler和logger。

LogRecord承载了日志具体内容，由logger生成。log filter起过滤日志的作用，handler对日志内容进行处理（emit），logger则是一个完整的日志功能对象。

官方文档<sup>[1]</sup>提供的日志处理流程：

![logging_flow](/img/in-post/python-logging-configuration/logging_flow.png)
<small class="img-hint">Logging Flow</small>


1. logger输出特定级别的日志，检查logger的level可否输出日志，是的话创建LogRecord
2. logger上的filter是否拒绝record，否的话传递给当前logger的handlers一一处理，然后进入handler flow
3. 在每个handler里同样要经历level和filter，只不过此时是handler的level和filter，通过后最终emit（包括格式化日志）
4. 最后还要判断是否propagate，即是否向parent logger传递该LogRecord

这个流程设计的好处是：对日志的level和filter可以细化到全局和局部的调控。在logger层面上设置level和filter，做到该logger全局控制；在某个handler上设置level和filter，则是局部控制。层级明确，收放自如。

---

## 入门

 从Logging flow可以看出，logger是logging flow的承载体，是必选项。logger可以设置level、filter和handlers，都是可选项，若不设置则采用默认值。每个handler上也可以各自设置level和filter，也是可选项。
 
 从官方的示例<sup>[2]</sup>来入手。

 ```python
import logging

# 生成logger，指定logger name和level
logger = logging.getLogger('simple_example')
logger.setLevel(logging.DEBUG)

# 生成console handler，指定level为debug
ch = logging.StreamHandler()
ch.setLevel(logging.DEBUG)

# 生成formatter
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')

# 给ch添加formatter
ch.setFormatter(formatter)

# 给logger添加ch
logger.addHandler(ch)

# 应用的代码
logger.debug('debug message')
logger.info('info message')
logger.warning('warn message')
logger.error('error message')
logger.critical('critical message')
 ```

 这种方式直接使用Python代码调用配置方法来生成logger、handler和formatter，将配置耦合到代码中，不好维护和管理。

 更好的方式是将配置和使用分开，配置写入配置文件中，然后导入到代码中。


## 配置

熟悉了流程，对配置参数的设置就不陌生。

这里介绍通过yaml文件配置日志的方法，配置简单，方便维护，用于线上项目正合适。

示例如下：

```yml
version: 1  
disable_existing_loggers: yes  
  
formatters:  
  standard: format: '%(asctime)s [%(name)s] %(levelname)s: %(message)s'  
  datefmt: '%Y-%m-%d %H:%M:%S'  
  
handlers:  
  default: class: logging.FileHandler  
    level: INFO  
    formatter: standard  
    filename: msg.log  
  console:  
    class: logging.StreamHandler  
    level: INFO  
    formatter: standard  
    stream: ext://sys.stdout  
  
loggers:  
  message_transfer: level: DEBUG  
    handlers: [default]  
    propagate: no  
  
root:  
  level: DEBUG  
  handlers: [console]
```

version为必选项，作为配置的唯一标识，disable_existing_loggers默认为True，即禁用除配置内的其他存在的loggers。formatters、handlers、loggers和root均为可选项。这样我们可以方便地配置多个formatters、handlers、loggers。使配置参数与代码逻辑解耦合。

---

## 使用

读取yaml文件需要安装PyYAML库：

```python
pip install PyYAML
```

以下是代码示例：

```python
import logging.config
import yaml

with open(os.path.join(BASE_PATH, 'logging_configs.yml'), 'r') as f:
    logging_configs = yaml.load(f, Loader=yaml.FullLoader)
logging.config.dictConfig(logging_configs)

logger = logging.getLogger('message_transfer')
logger.info('Hello World')
```

非常简便！人生苦短，我用Python。


## 补充解释

上述操作的逻辑是先构建一个yaml配置文件，然后加载该文件得到一个**configuring dictionary**（配置字典），这个字典必须遵循**Configuration dictionary schema**（配置字典模式）<sup>[3]</sup>。

该配置字典包括如下keys：

* **version**（必选项）：整数类型，表示模式的版本号。目前的唯一有效值是1，有助于模式的发展，同时保持向后兼容。
* 其它keys是可选项。首先查看有没有特殊的key'**()**'，如果存在，则实例化自定义对象<sup>[4]</sup>。否则，通过上下文决定实例化对象。
* **formatters**：value是一个dict，这个dict的每个key是formatter id，每个value是一个dict，字典的内容是对应的formatter的配置信息（keys为format和datefmt）。
* **filters**：类似的，value是一个dict，这个dict的每个key是filter id，每个value是一个dict，字典的内容是对应的filter的配置信息（keys为name）。
* **handlers**：value是一个dict，这个dict的每个key是handler id，每个value是一个dict，字典的内容是对应的handler的配置信息（keys为class，level，formatter，filters和其他key（用于构造Handler的关键词参数））。
* **loggers**：value是一个dict，这个dict的每个key是logger name，每个value是一个dict，字典的内容是对应的logger的配置信息（keys为level，propagate，filters和handlers）。
* **root**：value是一个dict，字典的内容是root logger的配置信息（keys为level，propagate，filters和handlers）。
* incremental：布尔类型，表示是否增量式配置，默认为False。若为True，则忽略掉所有的filters和formatters，只执行handlers的level、loggers和root logger的level和propagate。
* **disable_existing_loggers**：布尔类型，表示是否禁用配置文件外的其他存在的loggers，默认为True。如果incremental是True，则忽略此项。

配置字典模式使整个配置过程清晰明了，便于拓展和修改。

这个configuring dictionary作为参数传给**dictConfig函数**<sup>[5]</sup>执行。这个函数将configuring dictionary作为构造参数实例化dictConfigClass，然后调用该实例的实例方法configure()使配置生效。

这种logging配置和使用方法将配置和调用过程解耦合，很适合项目的使用。

如果识别**外部的Python对象**<sup>[6]</sup>呢？如：sys.stderr。在YAML这类文本文件中，没办法直接区分sys.stderr和"sys.stderr"。为了帮助区分，在sys.stderr加上前缀"ext://"，后续logging的配置系统会识别该前缀，然后把sys.stderr作为Python对象导入。


## 参考

[1] [Logging Flow](https://docs.python.org/3/howto/logging.html?#logging-flow)
[2] [Configuring Logging](https://docs.python.org/3/howto/logging.html?#configuring-logging)
[3] [Configuration dictionary schema](https://docs.python.org/3/library/logging.config.html#logging-config-dictschema)
[4] [User-defined objects](https://docs.python.org/3/library/logging.config.html#user-defined-objects)
[5] [Configuration functions](https://docs.python.org/3/library/logging.config.html#logging.config.dictConfig)
[6] [Access to external objects](https://docs.python.org/3/library/logging.config.html#access-to-external-objects)
