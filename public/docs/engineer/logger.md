# Logger

CommuneChatbot 默认的日志模块, 是通过  ```Commune\Chatbot\Framework\Providers\LoggerServiceProvider``` 加载到系统中的.
在机器人配置数组中的位置是 ```$chatbotConfig->baseServices->logger```, 详情可查看```Commune\Chatbot\Config\Children\BaseServicesConfig```.

覆盖该配置, 可以注册您的日志服务.

默认的日志服务使用 [Monolog](https://github.com/Seldaek/monolog). 可以通过机器人配置数组中的 ```$chatbotConfig->logger``` 进行配置, 具体定义可以查看```Commune\Chatbot\Config\Children\LoggerConfig```.


## 获取日志实例

系统中可进行依赖注入的场景, 都可以通过 ```Psr\Log\LoggerInterface``` 来获取日志实例.

但使用日志的情景其实有区别 :

- 进程级日志
    - 请求级日志
        - 多轮对话管理日志

这三种日志是包含关系, 后面的日志会记录一些故障排查用到的维度信息. 例如 :

- traceId
- messageId
- conversationId
- sessionId

因此, 最好使用与场景匹配的具体实例来记录日志 :

- 请求级日志 : ```$conversation->getLogger()```
- 会话级日志 : ```$session->logger```

## ConsoleLogger

CommuneChatbot 启动的时候, 会把启动信息打印到 Console 上, 这是通过```Commune\Chatbot\Contracts\ConsoleLogger```.

这个日志也是进程级的单例, 不是通过服务注册, 而是 App 启动时传入 :

```php

    $consoleLogger = new MyConsoleLogger();
    $chatApp = new Commune\Chatbot\Framework\ChatApp($config, null, $consoleLogger);

```

通过该实例, 可以将更多信息输出到 Console 上. 有两种方式获取:

- 通过依赖注入 ```Commune\Chatbot\Contracts\ConsoleLogger``` 实例
- 通过 App 获取, ``` $chatApp->getConsoleLogger() ```