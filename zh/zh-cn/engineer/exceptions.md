# 异常管理

> CommuneChatbot 目前的异常管理方案, 自觉不够成熟. 希望大家能不吝赐教

## 异常管理

CommuneChatbot 作为对话机器人框架, 设计理念是可以运行在任意应用框架中.
例如项目的工作站[studio-hyperf](https://github.com/thirdgerb/studio-hyperf) 使用了 [Hyeprf 框架](https://hyperf.io) 提供对话以外的其它功能.

因此 CommuneChatbot 不像其它全栈框架那样, 托管 PHP 所有的异常管理 (```set_exception_handler()``` 等), 避免和所在的应用框架冲突.

考虑到服务端主要在 Swoole 中运行, 异常需要在每一个请求的生命周期中进行处理.
否则会导致 Swoole 的 Worker 进程重启.
因此在系统运行的[管道层](/zh-cn/engineer/pipeline.md) 会处理大多数异常.

主要涉及异常管理的有以下几个类, 具体管理逻辑可查看源码:

- ```Commune\Chatbot\App\ChatPipe\UserMessengerPipe``` : 处理消息的启动管道
- ```Commune\Chatbot\Framework\ChatKernelImpl``` : 处理消息的单例
- ```Commune\Chatbot\Framework\Pipeline\ChatbotPipeImpl``` : 运行管道
- ```Commune\Chatbot\OOHost\Directing\Director``` : 多轮对话调度

## 异常上报

系统使用 ```Commune\Chatbot\Contracts\ExceptionReporter``` 进行异常的报告. 默认情况下, 只是把异常打印到 console.

可以注册自己的实现, 修改异常的上报机制, 例如将异常提交给 Sentry. 只需要提供新的 ServiceProvider, 并注册到机器人配置的 ```$chatbotConfig->baseServices->exp``` 处.

系统的 [Logger](/zh-cn/engineer/logger.md) 实例底层使用了 ```Commune\Chatbot\Framework\Impl\MonologWriter```, 如果传入的 ```$message``` 是 ```Throwable```对象时, 会调用 ExceptionReporter 进行上报.

因此拦截到异常时, 将异常作为 ```$message``` 直接传递给 Logger 是通用的上报办法.


## 异常副作用

与 PHP 常见的 Web 多进程应用不同, CommuneChatbot 还要面对两种特殊的异常情况:

- 在 Swoole 长进程中遭遇异常, 要避免 Worker 进程频繁重启
- 和用户的对话陷入逻辑 bug, 导致对话永远卡在系统出错上

所以 CommuneChatbot 的异常管理, 主要根据异常导致的副作用来划分. 这些副作用分为:

- 单次消息请求失效, 告知用户系统异常, 不影响后续对话
- 无法响应客户端
- 需要重置与用户的会话 (Session), 避免死循环
- Worker 进程需要重启
- 逻辑异常, 由代码直接导致, 应该在上线前彻底清除
- 重大异常, 导致系统不应该运行下去

由于未管理好的异常, 不仅可能导致 Swoole 的 Worker 进程频繁重启, 更可能导致与用户的对话卡死; 因此建议开发者严谨地管理异常, 不要轻易忽视.

## 异常分类

CommuneChatbot 的异常体系定义在命名空间 ```Commune\Chatbot\Framework\Exceptions``` 之下, 可以找到相关目录查看已定义的异常.

从处理策略的角度来看, 这些异常分为以下几种:

- 功能型异常
- 逻辑异常
- Conversation 级异常
- Client 级异常
- Session 级异常
- Server 级异常
- App 级异常

开发者可以根据实际的情况, 选择性地抛出这些异常达到目的.

### 功能性异常

功能型异常的目的, 在于跳过各种复杂的逻辑, 直接由上层逻辑进行处理.

```Commune\Chatbot\Framework\Exceptions\ReturnConversationException``` :
可通过它携带一个 Conversation, 直接跳过所有 ChatbotPipe, 返回给用户.

```Commune\Chatbot\Framework\Exceptions\RequestException``` :
可以判定当前请求非法, 系统直接拒绝响应, 调用```Commune\Chatbot\Blueprint\Conversation\MessageRequest::sendRejectResponse()``` 方法拒绝客户端请求.

```Commune\Chatbot\OOHost\Exceptions\NavigatorException``` :
跳过 Stage 的逻辑, 直接指定上下文切换.

```Commune\Chatbot\Framework\Exceptions\ContextFailureException``` :
告知系统一个 Context 内的多轮对话无法进行下去, 会触发退出事件 Failure, 可以在 Exiting 中进行捕获. 功能类似于 ```Dialog::cancel()```. 在 Stage 里定义对话逻辑, 可以通过这个异常来触发退出, 但无法指定给用户的回复.

### 逻辑异常

系统默认的逻辑异常是 ```Commune\Chatbot\Framework\Exceptions\ChatbotLogicException```. 这种异常通常是代码的严重错误, 或者配置文件的严重错误导致的.

这种异常不应该出现, 也不应该在运行中允许抛出. 在测试阶段应该全面地排查修复.

框架启动时的逻辑异常, 会另外封装到 ```Commune\Chatbot\Framework\Exceptions\BootingException``` 中.

### Conversation 级异常

默认的 Conversation 级异常是```Commune\Chatbot\Framework\Exceptions\ConversationalException```. 表示系统运行出现了偶发的问题 (接口超时, 数据库连接失效等), 导致当前这轮响应失败. 但仍然能用消息回复用户, 所以服务端会给客户端发送一个正常的响应, 附带系统错误的消息, 定义在配置```$chatbotConfig->defaultMessages->systemError``` 中.

所有继承自```\RuntimeException``` 的异常, 都认为是对话级异常, 需要在后续请求中自动恢复. 如果其中遭遇了致命错误, 需要重启客户端, 请不要让它抛出到外层, 而是手动捕获并处理.

### Client 级异常

默认的 Client 级异常是```Commune\Chatbot\Framework\Exceptions\CloseClientException```. 表示系统无法正常响应客户端.

这时, 系统会调用 ```Commune\Chatbot\Blueprint\Conversation\MessageRequest::sendFailure()``` 方法, 通知客户端请求失败. 同时调用 ```Commune\Chatbot\Contracts\ChatServer::closeClient()``` 方法尝试关闭连接 (在双工场景中比较重要).

### Session 级异常

默认的 Session 级异常是```Commune\Chatbot\Framework\Exceptions\CloseSessionException```, 它表示多轮对话陷入了严重错误, 无法恢复. 为避免死循环, 系统会重置 Session, 并告知用户服务端出错.

最常见的 Session 级异常是 ```Commune\Chatbot\OOHost\Exceptions\TooManyRedirectException```,
由于对话管理逻辑死循环, 导致服务端超过了状态调度次数;
然后是```Commune\Chatbot\OOHost\Exceptions\SessionDataNotFoundException```, 尝试获取某个不存在的上下文信息, 导致会话永远无法继续下去.
这些异常都应该在上线前彻底排除.

在多轮对话管理阶段, 由于代码逻辑错误抛出```Commune\Chatbot\Framework\Exceptions\ChatbotLogicException```, 也会重置 Session, 而不是关闭 Server 端.

### Server 级异常

默认的 Server 级异常是 ```Commune\Chatbot\Framework\Exceptions\CloseServerException```, 表示服务端遭遇了不可修复的异常, 必须要关闭 Server
实例, 或者重启 Worker 进程.

这时系统会尝试调用 ```Commune\Chatbot\Contracts\ChatServer::fail()``` 方法关闭服务端实例.

一旦进入这种异常, 系统就无稳定性可言了. 也是必须要在上线前清除的问题.

另外注意, Swoole 的 Worker 进程退出不是立刻生效的, 因为同时还有协程任务处理其它的请求. 这方面的管理策略, 请参考 [swoole 文档](https://wiki.swoole.com/wiki/page/1010.html).

### App 级异常

默认的 App 级异常是 ```Commune\Chatbot\Framework\Exceptions\CloseAppException``` . 当这种异常被抛出后, 理论上所有服务端实例都应该停止响应用户, 使用配置中定义的消息```$chatbotConfig->defaultMessages->platformNotAvailable```告知用户系统不可用.

在系统逻辑中不会抛出这个异常, 除非开发者手动抛出.

系统的状态通过 ```Commune\Chatbot\Contracts\ChatServer::setAvailable()``` 方法进行同步.

这个方法理论上要通过分布式配置中心来实现服务端实例的一致性. 目前并没有实装. 您如果有需要, 可以定义自己的```Commune\Chatbot\Contracts\ChatServer``` 实现这一功能.


