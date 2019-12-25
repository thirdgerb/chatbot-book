# 注册 NLU 中间件

在 [Intent 文档](/zh-cn/dm/intent.md) 中已经介绍过, NLU 解析信息的多种来源.
而在 [封装 NLUService](/zh-cn/nlu/service.md) 一文中也简单介绍了如何用 ```Commune\Chatbot\Blueprint\Conversation\NLU``` 保存解析信息.

调用第三方服务, 获取自然语言解析信息, 默认的方式还是通过 Session Pipe.

简单而言, 将调用逻辑封装为 ```Commune\Chatbot\OOHost\Session\SessionPipe``` 中间件; 然后注册到机器人配置的 ```$chatbotConfig->host->sessionPipes``` 位置, 具体见 ```Commune\Chatbot\Config\Children\OOHostConfig```.

```php
// 配置数组
return [
    ...

    'host' => [
        ...

        // 机器人会话级管道
        'sessionPipes' => [
            \Commune\Chatbot\App\SessionPipe\EventMsgPipe::class,
            \Commune\Chatbot\App\SessionPipe\MarkedIntentPipe::class,
            \Commune\Chatbot\App\Commands\UserCommandsPipe::class,
            \Commune\Chatbot\App\Commands\AnalyserPipe::class,

            // NLU 中间件, 应该放在 NavigationPipe 之前
            // \Commune\Components\Rasa\RasaSessionPipe::class,

            \Commune\Chatbot\App\SessionPipe\NavigationPipe::class,
        ],

        ...
    ]

```

## 使用 NLUService

尽管 SessionPipe 可以独立提供 NLU 服务, 我们仍然建议先封装 ```Commune\Chatbot\OOHost\NLU\Contracts\NLUService``` 对象.

这样可以通过继承 ```Commune\Chatbot\OOHost\NLU\Pipe\AbsNLUServicePipe``` 类,
快速实现相关中间件.
具体可以参考 ```Commune\Components\Rasa\RasaSessionPipe```.

## NLULogger

在中间件抽象类 ```Commune\Chatbot\OOHost\NLU\Pipe\AbsNLUServicePipe``` 中,
默认使用了 ```Commune\Chatbot\OOHost\NLU\Contracts\NLULogger``` 服务,
用于记录 NLU 中间件的解析结果.

系统默认提供了一个简单的实现 ```Commune\Chatbot\OOHost\NLU\Predefined\SimpleNLULogger```.
它通过组件 ```Commune\Chatbot\OOHost\NLU\NLUComponent``` 注册到系统中.

可以自己另外实现 NLULogger, 然后替换掉系统默认的实现.
具体可以查看 [管理 NLU 服务](/zh-cn/nlu/manager.md).
