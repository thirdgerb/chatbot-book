# 更多工具类

CommuneChatbot 除了注册各种工程模块的服务之外, 还使用了一些工具类以实现必要的功能.

## Id Generator

作为一个对话系统, 有大量情况下要生成全局唯一ID. 但它又不适合注册为公共服务, 因为有些 UUID 的生成逻辑可能并不一致.

为了让 UUID 生成变为可自定义的, CommuneChatbot 约定了接口 ```Commune\Support\Uuid\HasIdGenerator```.

任何实现该接口的类, 既能够生成 UUId, 又能够通过 ```HasIdGenerator::setGenerator($idGenerator)``` 的方式替换掉 UUId 的生成类.

CommuneChatbot 默认使用了 UUId 的场景很多, 通过 IDE 能够快速查询到. 它们都是通过自带的 Trait ```Commune\Support\Uuid\IdGeneratorHelper```来获得 ID 生成能力的.

系统默认使用的 ID 生成库是[ramsey/uuid](https://github.com/ramsey/uuid). 如果希望替换, 您可以自己注册一个进程级的 ServiceProvider :

```php

class UUIdServiceProvider extends ServiceProvider
{
    const IS_PROCESS_SERVICE_PROVIDER = true;

    public function boot($app)
    {
        // 通过 IoC 容器获取 id 生成器实例
        $idGenerator = $app[Commune\Support\Uuid\IdGenerator::class];

        // 替换掉该类默认的 id 生成器
        Commune\Chatbot\Framework\Conversation\IncomingMessageImpl::setIdGnerator($idGenerator);

        ...

    }

    public function register()
    {
        $this->app->singleton(
            Commune\Support\Uuid\IdGenerator::class
            function(){...}
        );
    }
}


```


## RunningSpy

由于 CommuneChatbot 默认基于 Swoole 启动, 作为长进程提供服务. 因此需要排查内存泄漏的问题. 除了 Swoole 官方提供的各种排查方法之外, 这里也使用一个基于 PHP 实现的策略.

简单而言, 就是目标类需要实现 ```Commune\Chatbot\Blueprint\Conversation\RunningSpy``` 接口, 通过使用 Trait ```Commune\Chatbot\Framework\Conversation\RunningSpyTrait```.

然后在运行中调用方法:

```php

class MyClass implements Commune\Chatbot\Blueprint\Conversation\RunningSpy
{
    use Commune\Chatbot\Framework\Conversation\RunningSpyTrait;


    protected $id;

    public function __construct(...)
    {
        $this->id = $id;
        ...
        // 记录当前实例的唯一 ID
        static::addRunningTrace($this->id, $this->id);
    }

    ...

    public function __destruct()
    {
        // 销毁前, 删除当前实例的注册
        static::removeRunningTrace($this->id);
    }

}
```

这样, 我们在```Commune\Chatbot\Framework\Conversation\RunningSpies``` 中可以查到有多少个相关实例在运行.

进一步的, 可以在多轮对话中使用系统自带的命令```Commune\Chatbot\App\Commands\Analysis\RunningSpyCmd``` 来查看当前进程的单例持有情况.

```
用户 : /runningSpy
机器人 :

Commune\Chatbot\Framework\Predefined\SymfonyEventDispatcher 运行中实例共 1 个
Commune\Chatbot\Framework\Conversation\ConversationLoggerImpl 运行中实例共 1 个
Commune\Chatbot\Framework\Conversation\ConversationImpl 运行中实例共 1 个
Commune\Chatbot\App\Drivers\Demo\ArrayCache 运行中实例共 1 个
Commune\Chatbot\App\Drivers\Demo\ArraySessionDriver 运行中实例共 1 个
Commune\Chatbot\OOHost\Session\RepositoryImpl 运行中实例共 1 个
Commune\Chatbot\OOHost\Session\SessionImpl 运行中实例共 1 个
Commune\Chatbot\OOHost\Session\SessionLogger 运行中实例共 1 个
Commune\Chatbot\OOHost\History\History 运行中实例共 1 个
Commune\Chatbot\OOHost\Dialogue\DialogImpl 运行中实例共 1 个
Commune\Chatbot\OOHost\Directing\Director 运行中实例共 0 个

```

这种方法简单有效, 可以在开发过程中快速定位内存泄漏问题. 更高级的内存泄漏排查方法请阅读 Swoole 的相关文档与扩展工具.


> 由于 RunningSpy 机制使用 PHP 数组来记录, 高并发的情况下对性能有干扰. 所以默认只在 Debug 状态运行.
> 您可以通过 ```Commune\Chatbot\Framework\Conversation\RunningSpies::$run``` 静态属性来控制它的开关.
