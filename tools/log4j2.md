# log4j2 学习笔记


## log4j2 体系结构

![](../res/Log4j2-Classes.jpg)

> Applications using the Log4j 2 API will request a Logger with a specific name from the LogManager. The LogManager will locate the appropriate LoggerContext and then obtain the Logger from it. If the Logger must be created it will be associated with the LoggerConfig that contains either a) the same name as the Logger, b) the name of a parent package, or c) the root LoggerConfig.LoggerConfig objects are created from Logger declarations in the configuration. -- [log4j2 体系结构（官方版）](http://logging.apache.org/log4j/2.x/manual/architecture.html)

- 配置文件中, 每定义一个 `logger` 就定义了一个 `LoggerConfig`。
- 后期通过 `LogManager.getLogger()` 来获取 `logger` 时， 新的 `logger` 会直接关联到相应的 `LoggerConfig` 上。

## 参考

1. [log4j2 体系结构（官方版）](http://logging.apache.org/log4j/2.x/manual/architecture.html)
1. [Log4j2 架构及概念简介](http://www.cnblogs.com/elaron/archive/2013/01/14/2860259.html)
1. [Log4j 2使用教程](http://www.cnblogs.com/leo-lsw/p/log4j2tutorial.html), 写的很不错。推荐一下。
1. [PatternLayout](http://logging.apache.org/log4j/2.x/manual/layouts.html#PatternLayout)。用来规定输出格式的详细文档。