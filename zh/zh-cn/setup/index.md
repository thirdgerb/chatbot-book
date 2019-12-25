# 搭建应用

CommuneChatbot 的核心项目是 [commune/chatbot](http://packagist.org/packages/commune/chatbot), 作为底层开发框架,
应该可以部署到各种应用框架之中.

如果要搭建对话机器人项目, 推荐使用项目 [commune/studio-hyperf](http://packagist.org/packages/commune/studio-hyperf).
这是基于 [swoole](https://www.swoole.com/) 与 [hyperf](https://hyperf.io/) 开发的工作站. 可以快速启动基于 Swoole 协程的服务端应用.
并且开箱自带 Web版, 微信版与百度音箱等应用的 package, 可以立刻对接这些平台.

----

如果您只是想快速启动, 尝试项目, 推荐查看 [快速上手教程](/zh-cn/lesions/index.md).

如果打算搭建一个真正的应用, 推荐以下路径:

- [安装工作站](/zh-cn/setup/studio.md)
- [工作站目录结构](/zh-cn/setup/directory.md)
- [修改机器人配置](/zh-cn/setup/config.md)
- [运行应用](/zh-cn/setup/run.md)
- [开发对话与服务](/zh-cn/setup/develop.md)

系统自带几个实际可用的 Demo :

- [搭建机器人 API](/zh-cn/setup/api.md)
- [搭建微信公众号 bot](/zh-cn/setup/wechat.md)
- [搭建百度音箱 bot](/zh-cn/setup/web.md)
- [搭建 web bot](/zh-cn/setup/web.md)

