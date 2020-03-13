# 结论

这里我们构建了一个简单但是功能比较齐全的 Laminas laminas-mvc 应用。

在这个教程中我们简单的接触了框架的几个不同的部分。

建立 laminas-mvc 应用最重要的部分就是
[modules](https://laminas.org.cn/laminas-modulemanager/intro/), 
作为构建 [laminas-mvc application](https://laminas.org.cn/laminas-mvc/quick-start/) 的基础部件。

为了配合我们应用不同部件之间的依赖关系，我们需要只用
[service manager](https://laminas.org.cn/laminas-servicemanager/intro/).

为了能够映射请求的路由到控制器中的操作上，我们需要使用
[routes](https://laminas.org.cn/laminas-router/routing/).

数据持久化使用
[laminas-db](https://laminas.org.cn/laminas-db/adapter/)和数据库进行通讯。
使用[input filters](https://laminas.org.cn/laminas-inputfilter/intro/),
对输入的数据进行验证，
并且配合 [laminas-form](https://laminas.org.cn/laminas-form/intro/),
在视图模型以及域模型之间架起稳健的桥梁。

[laminas-view](https://laminas.org.cn/laminas-view/quick-start/) 
是负责在 MVC 栈中处理视图，并且提供了相当多的
[view helpers](https://laminas.org.cn/laminas-view/helpers/intro/)
可供使用。
