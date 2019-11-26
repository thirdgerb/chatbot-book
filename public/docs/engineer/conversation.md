# Conversation

Conversation 是对用户完整请求的高级封装,
接口定义为```Commune\Chatbot\Blueprint\Conversation\Conversation```,
具体实现为```Commune\Chatbot\Framework\Conversation\ConversationImpl```.

Conversation 本身既是在中间件中传输的对象, 又是一个请求级容器. 绝大多数与请求相关的服务和单例, 都可以通过 Conversation 对象来获取.


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

