# 运行应用

CommuneChatbot 的工作站基于 Swoole + Hyperf 运行, 会开启进程监听端口,
用于提供机器人服务端的服务. 如果要对外提供 Http 服务, 通常可以结合 Nginx 使用.

工作站是通过 Hyperf 项目的[命令行](https://doc.hyperf.io/#/zh-cn/command) 机制启动机器人应用的.
有两个基础命令类:

- 运行命令行测试 : ```Commune\Hyperf\Commands\Tinker```
- 启动应用 : ```Commune\Hyperf\Commands\StartApp```

运行命令行的测试机器人 :

    php bin/hyperf.php commune:tinker

运行机器人应用的服务端 :

    php bin/hyperf.php commune:start [appName]

参数 ```appName``` 定义在配置文件 ```BASE_PATH/config/autoload/commune.php```.
这样会按配置的定义, 启动服务端实例.

CommuneChatbot 的会话是有状态的, 但可以从无状态的请求中根据 ChatId 还原.
因此同一个机器人可以开启多个服务端实例.

至于这些服务端实例如何治理, 对外提供统一的服务, 则由开发者来决定.

应用部署的相关思路, 可以参考:

- [hyperf 官方文档](https://doc.hyperf.io/)
- [hyperf : nginx 反向代理](https://doc.hyperf.io/#/zh-cn/tutorial/nginx)
- [swoole 官方文档](https://wiki.swoole.com/)
