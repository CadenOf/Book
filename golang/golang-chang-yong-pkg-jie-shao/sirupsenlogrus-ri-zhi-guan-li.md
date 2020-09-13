# sirupsen/logrus 日志管理

{% hint style="success" %}
####  [https://github.com/Sirupsen/logrus](https://github.com/Sirupsen/logrus) 
{% endhint %}

### logrus features

* 完全兼容 golang 标准库日志模块 logrus 拥有六种日志级别：`debug`、`info`、`warn`、`error`、`fatal`和`panic`,这是golang标准库日志模块的API的超集.如果您的项目使用标准库日志模块,完全可以以最低的代价迁移到logrus上.
  * `logrus.Debug("Useful debugging information.")`
  * `logrus.Info("Something noteworthy happened!")`
  * `logrus.Warn("You should probably take a look at this.")`
  * `logrus.Error("Something failed but I'm not quitting.")`
  * `logrus.Fatal("Bye.")` //log之后会调用os.Exit\(1\)
  * `logrus.Panic("I'm bailing.")` //log之后会panic\(\)
* 可扩展的 Hook 机制 允许使用者通过 hook 的方式将日志分发到任意地方,如本地文件系统、标准输出、`logstash`、`elasticsearch`或者`mq`等,或者通过hook定义日志内容和格式等.
* 可选的日志输出格式 logrus 内置了两种日志格式,`JSONFormatter`和`TextFormatter`,如果这两个格式不满足需求,可以自己动手实现接口Formatter,来定义自己的日志格式.
* `Field`机制 `logrus`鼓励通过Field机制进行精细化的、结构化的日志记录,而不是通过冗长的消息来记录日志.
* `Entry`: `logrus.WithFields`会自动返回一个 `*Entry,Entry`里面的有些变量会被自动加上
  * `time:entry`被创建时的时间戳
  * msg: 在调用`.Info()`等方法时被添加
  * level

