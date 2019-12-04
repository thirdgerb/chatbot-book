# Director

当对话机器人接受到一个用户消息后, 会通过它还原出当前的会话对象```Commune\Chatbot\OOHost\Session\Session```, 然后调用```Session::handle()``` 方法来处理所有响应逻辑.

在一个复杂的多轮对话逻辑中, 尽管只响应了用户的一条消息, 但实际上整个多轮对话状态机已经经历了若干轮状态变更.
这些状态变更逻辑比较复杂, 如果放任的话可能会造成几十甚至上百层的函数调用, 一旦出现问题排查起来很麻烦.

所以 CommuneChatbot 项目的 OOHost 模块使用了 ```Commune\Chatbot\OOHost\Directing\Director``` 类来负责做多轮对话状态机的调度.
将上下文状态切换的逻辑保存在 ```Commune\Chatbot\OOHost\Directing\Navigator```对象中, 从 Context 层层返回到 Director 之后, 才执行逻辑.
这样函数调用的深度会更小, 而且也便于追踪轨迹.

用户只需要专注于定义 "单轮对话" [Stage](/docs/dm/stage.md), 在上下文逻辑中通过 Dialog 正确返回了```Commune\Chatbot\OOHost\Directing\Navigator```对象就足够. Director 会自行处理调度逻辑.

```php

    public function __onStart(Stage $stage) : Navigator
    {
        return $stage->talk(
            // 机器人说话给用户
            function(Dialog $dialog) {
                $dialog->say()->info('hello');

                // 返回一个 Wait Navigator
                return $dialog->wait();
            },

            // 用户回话给机器人
            function(Dialog $dialog) {
                ...

                // 返回一个 Repeat Navigator, 重复当前对话.
                return $dialog->repeat();
            }
        );
    }
```

系统定义了许多种 ```Commune\Chatbot\OOHost\Directing\Navigator```, 在命名空间```Commune\Chatbot\OOHost\Directing``` 之下, 可以查看源代码了解.

开发者通常只需要通过```Commune\Chatbot\OOHost\Dialogue\Dialog```提供的 API 去返回调度逻辑即可. 不用手动去生成它们.

## 最大重定向次数

多轮对话状态机经过 Director 进行状态变更, 被称作重定向. 如果上下文逻辑出现错误的死循环, 理论上重定向会一直循环下去.

因此需要设定重定向的最大次数, 超过次数就抛出异常, 终止会话. 这个重定向次数定义在机器人配置文件中的 ```$chatbotConfig->host->maxRedirectTimes``` 中. 具体可以查看```Commune\Chatbot\Config\Children\OOHostConfig```.

如果您的单轮对话逻辑, 在特殊情况下状态变更次数超过默认值, 您可以调高这个参数.


## Session::$tracker

Director 进行状态调度时, 会把运行流程记录到 ```Commune\Chatbot\OOHost\History\Tracker``` 中. 可以通过 ```Commune\Chatbot\OOHost\Session\Session::$tracker``` 来获取它.

```php

    public function __onStart(Stage $stage) : Navigator
    {
        dd($stage->dialog->session->tracker);
    }

```

系统默认每一轮对话都会记录 tracker 的信息到日志. 一个日志片段大概是这样的 :

```
demo.INFO: sessionTracking 3 times :

{
    "run":"CallbackStage",
    "name":"commune.components.demo.contexts.featuretest",
    "id":"328f1147-c1ae-423a-ba0e-74c200f31463",
    "stage":"testMemory",
    "belongsTo":"4b905076-a5a0-4ced-8cc0-29f8b46d3083"
}
|
{
    "run":"Repeat",
    "name":"commune.components.demo.contexts.featuretest",
    "id":"328f1147-c1ae-423a-ba0e-74c200f31463",
    "stage":"testMemory",
    "belongsTo":"4b905076-a5a0-4ced-8cc0-29f8b46d3083"
}
|
{
    "run":"Wait",
    "name":"commune.components.demo.contexts.featuretest",
    "id":"328f1147-c1ae-423a-ba0e-74c200f31463",
    "stage":"testMemory",
    "belongsTo":"4b905076-a5a0-4ced-8cc0-29f8b46d3083"
}
```

您可以通过机器人配置中的参数```$chatbotConfig->host->logRedirectTracking``` 决定是否记录这些日志. 您也可以关闭默认的日志, 通过一个自定义的 SessionPipe 中间件去手动记录 tracking. 详见 [管道文档](/docs/engineer/pipeline.md).