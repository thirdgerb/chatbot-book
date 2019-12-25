# 管道

现代工程在设计生命周期时, 为了实现高度的解耦与可拓展, 往往会使用管道的机制.

简单而言, 就是把请求生命周期的每一个环节, 都做成一个管道, 首位相连构成一个长管. 需要在生命周期中增加功能模块, 就只要增加一个管道; 相反则削减值.

管道在实际调用中构成了这样的 ```穿透洋葱式``` 流程.

    // A, B, C 为管道的名称

    请求 => (A => ( B  => ( C ) => B ) => A ) => 响应

任何一个管道, 都有权利将请求直接返回给上层管道, 而不传输给后续管道. 因此管道就起到了```中间件```的功能.

## 基于闭包的管道实现

CommuneChatbot 的管道机制, 修改自 [Laravel 的管道模块](https://packagist.org/packages/illuminate/pipeline) . 它是基于```依赖注入```和```闭包```实现的.

有兴趣了解原理, 可以查看```Commune\Chatbot\Framework\Utils\OnionPipeline```.

## 注册管道

CommuneChatbot 项目中有两种管道:

* chatPipe : 机器人整体的管道
    - 传输对象 : ```Commune\Chatbot\Blueprint\Conversation\Conversation```
    - 实现 : ```Commune\Chatbot\Blueprint\Conversation\Conversation\ChatbotPipe```
    - 配置位置 : ```Commune\Chatbot\Config\ChatbotConfig::$chatbotPipes```
* sessionPipe : 对话管理模块的管道
    - 传输对象 : ```Commune\Chatbot\OOHost\Session\Session```
    - 实现 : ```Commune\Chatbot\OOHost\Session\SessionPipe```
    - 配置位置 : ```Commune\Chatbot\Config\Children\OOHostConfig::$sessionPipes```

可以在配置数组中进行修改.

系统默认的管道功能如下:

```php

    'chatbotPipes' =>
        [
            'onUserMessage' => [
                // 发送 conversation, 并且管理所有的异常
                \Commune\Chatbot\App\ChatPipe\MessengerPipe::class,

                // 用于锁定 chat, 避免用户输入消息太频繁导致歧义
                \Commune\Chatbot\App\ChatPipe\ChattingPipe::class,

                // 多轮对话管理内核
                \Commune\Chatbot\OOHost\OOHostPipe::class,
            ],
        ],

    ...

    'host' => [
        ...

        'sessionPipes' => [
            // event 转 message
            \Commune\Chatbot\App\SessionPipe\EventMsgPipe::class,

            // 单纯用于测试的管道,#intentName# 模拟命中一个意图.
            \Commune\Chatbot\App\SessionPipe\MarkedIntentPipe::class,

            // 用户可用的命令.
            SessionPipes\UserCommandsPipe::class,

            // 超级管理员可用的命令.
            SessionPipes\AnalyseCommandsPipe::class,

            // 优先级最高的意图, 通常用于导航.
            // 会优先匹配这些意图.
            SessionPipes\NavigationIntentsPipe::class,
        ],

```


## 在 Hearing 中使用 SessionPipe

针对```Session```定义的管道, ```Commune\Chatbot\OOHost\Session\SessionPipe```, 同样可以用于 ```Dialog::hear```.

例如 :

```php

    public function __onStart(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ->ask(...)
            ->hearing()

            // 运行中会直接调用管道作为中间件
            // 如果 session 被中间件拦截了, 则不会继续执行后续逻辑.
            ->middleware( AppCommandPipe::class )
            ...
            ->end();
    }
```

## 自定义管道

定义一个管道, 可以参考 [laravel 中间件文档](https://laravel.com/zh-cn/6.x/middleware).

简单而言:

```php

class PipeExample {

    // 构造方法可以实现依赖注入
    public function __construct(...) {}

    public function handle($passable, Closure $next)
    {
        // 对入参进行处理的逻辑
        ...

        // 调用后续的管道, 得到出参
        $passable = $next($passable);

        // 对出参进行处理的逻辑
        ...

        return $passable;
    }

}


```


如果需要调用管道, 可以查阅 ```Commune\Chatbot\Framework\Utils\OnionPipeline```, 基本的用法是:

```php

    $pipes = [
        // 用类名做中间件
        MiddlewareClass1::class,

        // 允许用这种方式给中间件传入更多参数
        MiddlewareClass2::class .':param1,param2,param3'
    ];

    $pipeline = new OnionPipeline($contianer, array $pipes);

    // 执行管道.
    $passable = $pipeline
        // 定义穿过的方法
        ->via('handle')

        // 传入传输的 passable 对象, 并定义终点方法
        ->send(
            // 可传输对象, 例如 session
            $passable,
            // 自定义的终点方法
            function($passable) {
                return $passable;
            }
        );
```





