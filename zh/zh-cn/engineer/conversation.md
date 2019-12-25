# Conversation

Conversation 是对用户完整请求的高级封装,
接口定义为```Commune\Chatbot\Blueprint\Conversation\Conversation```,
具体实现为```Commune\Chatbot\Framework\Conversation\ConversationImpl```.

Conversation 本身既是在中间件中传输的对象, 又是一个请求级容器. 绝大多数与请求相关的服务和单例, 都可以通过 Conversation 对象来获取 : ```$conversation->get($abstract)```.

系统启动的时候, 会将所有请求级的服务通过 Service Provider 注册到 Conversation 中, 只注册一次.

每一个请求都会生成一个独立的 Conversation 实例, 并运行已注册的 Service Provider 的```Commune\Chatbot\Blueprint\ServiceProvider::boot()``` 方法. 开发者需要了解该方法每个请求都会运行一次.


## API

Conversation 的所有方法定义, 请查看 ```Commune\Chatbot\Blueprint\Conversation\Conversation```.

## 权限机制

本项目可以用 ```Commune\Chatbot\Blueprint\Conversation\Conversation::isAbleTo(string $abilityName)``` 的形式判断用户是否拥有某种权限.

具体请看[权限机制文档](docs/engineer/abbilities.md).


## ConversationLogger

每一个请求有独立的 ```Commune\Chatbot\Blueprint\Conversation\ConversationLogger```, 在写入日志时会带上和请求相关的信息, 方便查询.

业务逻辑应该尽可能用 ConversationLogger来记录日志.

## Service Provider

Conversation 所有子服务的定义在 ```Commune\Chatbot\Framework\Providers\ConversationalServiceProvider```.

这里注册的服务, 都可以作为单例在依赖注入时获取. 也可以用自己新注册的服务去覆盖默认的.


## onFinish

系统可以通过```Commune\Chatbot\Blueprint\Conversation\Conversation::onFinish()``` 方法注册闭包, 在```Commune\Chatbot\Framework\ChatKernelImpl``` 回收 Conversation 的时候执行.

该功能通常用于系统内部, 用于解决生命周期比较特殊的场景 (例如 Quit), 通常不应该使用.
