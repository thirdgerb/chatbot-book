# Memory

上下文记忆是多轮对话管理非常核心的一环. 在对话中出现的关键信息, 要被携带很多轮之后才被使用.

常见的上下文记忆是存储在 Session 中的, 存在三个关键的问题 :

- 不能持久化 : Session 销毁就会失去记忆. 需要另外开发存储机制.
- 没有作用域隔离 : 同类的信息可能在不同的对话中相互污染.
- 与多轮对话分开 : 获取记忆往往来自多轮对话, 而记忆另外存储, 有/无记忆条件下触发对话的规则较为复杂.

为了解决这些问题, CommuneChatbot 用一种有趣的方式设计了作用域隔离的, 多轮对话相关的上下文记忆机制.


## 从 Context 到 Memory

在 [Context](/docs/dm/context.md) 一节中已经介绍过, Context 可以定义属性, 而这些属性在多轮对话中可以随时读写. 这本身是一种 "语境相关" 的上下文记忆机制.

而 Context 又可以通过 "Entity 机制", 将另一个 Context 的结果作为自己的属性.
这样, 当另一个 Context 对象实体信息不完整时, 会跳转 (dependOn) 到该 Context 尝试补完信息; 如果已经补完, 则不会跳转, 可以直接使用.

因为允许 A Context 持有另一个 B Context, 这本身就是将 B Context 作为 A Context 可的 __上下文记忆对象__ 来使用.

之所以这个 Context 对象在多轮对话中可以一直获取, 因为它是一个 ```Commune\Chatbot\OOHost\Session\SessionData```对象, 可以根据```$context->getId()``` 作为唯一凭据从 Session 存储介质中获取.

Memory (```Commune\Chatbot\OOHost\Context\Memory\Memory```) 就是结合以上机制设计出来的上下文记忆.

简单来说, Memory 对象本身就是 Context 对象, 它可以拥有独立的上下文, 不需要另外定义.
而 Memory 对象的属性, 就是上下文记忆的值.

Memory 对象都是通过 Session 进行持久化存储, 基于唯一的 ```$memory->getId()``` 的值来获取. 从而保证使用同一个 ID, 每次获得的是同一个 Memory.

而```Memory::getId()``` 的值可以用任何规则指定, 默认规则是选择 Session 的若干作用域维度, 将之生成唯一ID, 以此来实现作用域隔离.


## Memory 的作用域

Memory 的作用域, 本质上是任何可以决定```$memory->getId()``` 唯一值的方式. 而默认的作用域来自 Session.

这些作用域定义在 ```Commune\Chatbot\OOHost\Session\Scope``` 对象中. 常用的有:

- SessionId : 当前 Session 唯一ID
- UserId : 当前用户唯一ID
- ChatId : 当前对话唯一ID
- ConversationId : 当次请求的唯一ID
- ChatbotName : 当前机器人的唯一ID
- PlatformId : 所述平台的唯一ID

将所有的这些维度, 挑选一种或若干种, 可以通过 ```$session->scope->makeScopingId($type, $scopes)``` 得到一个唯一ID, 成为这个 Memory 对象的 __座标__.

例如我们用 ```UserId``` 作为唯一的 Scope, 该 Memory 对象只要是和同一个用户对话, 任何时候得到的都是同一个 MemoryId. 所以读写的是用户级别共享的历史记忆.

用这种方法, 理论上可以设计出任何维度的上下文记忆, 并且该记忆的值本身就通过多轮对话来获取, 与上下文高度结合在一起.


## 定义 Memory 类

Memory 默认的基类是 ```Commune\Chatbot\App\Memories\MemoryDef```, 通过它可以快速定义出记忆来:

```php

/**
 * 用户的信息
 * @property string $name 用户姓名
 * @property int $gendar 性别
 * @property int $age 年龄
 */
class UserInfoMemory extends MemoryDef
{
    // 定义 $memory->getDef()->getDesc() 的值
    const DESCRIPTION = '用户信息';

    // 定义作用域, 这儿只需要两个作用域, 用户ID 和所属机器人.
    // 理论上任何时候都要有机器人ID, 避免跨机器人记忆混乱.
    const SCOPE_TYPES = [Scope::USER_ID, Scope::CHABOT_NAME];


    /*------ 以下三个方法均为可选 ------*/

    /**
     * 可选
     *
     * 重写这个方法, 可以修改 Memory 默认的名称
     */
    public static function getContextName() : string
    {
        return StringUtils::normalizeContextName(static::class);
    }

    /**
     * 可选
     *
     * 默认定义Entity 的方法, 从类的 @property 注解直接获取.
     * 可以重写这个方法, 按自己设计的任何方式定义这些 Entity
     */
    public static function __depend(Depending $depending): void
    {
        $depending->onAnnotations();
    }

    /**
     * 可选
     *
     * 可以自己写 Stage, 定义任何一个同名 Entity 的单轮对话.
     */
    public function __onName(Stage $stage) : Navigator
    {
        ...
    }

}
```


## 在 Context 中使用 Memory

默认的 Memory 都是用 ContextName 直接从 Session 中获取的. 所以可以通过以下两种方式在任意场景实例化:

```php
    $userInfoMem = UserInfoMemory::fromSession($session);
    $userInfoMem = UserInfoMemory::from($context);
```

如果要在 Context 中使用, 则变成 :

```php

class SomeContext extends OOContext
{
    ...

    public function someAction(Dialog $dialog) : Navigator
    {
        // 用 Memory::from 从当前 Context 直接获取
        // 对 IDE 支持非常好
        $userInfoMem = UserInfoMemory::from($this);

        // 获取记忆的属性
        $name = $userInfoMem->name;
        ...
        // 给记忆的属性赋值
        $userInfoMem->age = $age;
        ...
    }

    ...
}

```

如果记忆是这个 Context 许多个 Stage 都需要共用的话, 可以定义成一个 getter 对象:

```php

/**
 * @property-read UserInfoMemory $userInfo 用注解标记为只读属性, 方便 IDE 识别
 */
class WelcomeUser extends OOContext
{
    // 用 getter 方法定义这个记忆的调用.
    public function __getUserInfo() : UserInfoMemory
    {
        return UserInfoMemory::from($this);
    }
}

```

如果需要得到该记忆的值, 没有的话通过记忆自己的多轮对话补完, 则可以通过 dependOn :

```php

    public function __onUserInfoMemory(Stage $stage) : Navigator
    {
        return $this->dependOn(

            // 可以直接将记忆当成一个依赖对象
            UserInfoMemory::class,

            // 如果该记忆有值, 会调用这个回调方法
            // 回调时可以把记忆直接依赖注入.
            function(UserInfoMemory $userInfo, Dialog $dialog) {
                ...
            }
        );
    }
```

还可以使用 Entity 机制, 更为简洁地定义一个记忆依赖:

```php

/**
 * @property-read UserInfoMemory $userInfo
 */
class WelcomeUser extends OOContext
{

    public function __depend(Depending $builder) : void
    {
        // 直接在 Depending 中定义整个记忆作为一个 Entity
        // 只有这些数据都拿到了, 才会进入 stage:start
        $builder->onMemory('userInfo', UserInfoMemory::class);

    }
    ....

}
```

我们甚至可以不依赖整个 Memory 对象, 只依赖其中的一个值:

```php

/**
 * @property string $userName
 */
class WelcomeUser extends OOContext
{

    public function __depend(Depending $builder) : void
    {
        // 直接在 Depending 中定义记忆的一个值作为 Entity
        $builder->onMemoryVal(
            // 当前 Context 内 Entity 的名称
            'userName',
            // 记忆的名称
            UserInfoMemory::class,
            // 值在记忆里的名称
            'name'
        );

    }
    ...

}
```

所以上下文记忆的使用方式是非常灵活的. 但要注意, 上下文记忆会持久化保存, 必须考虑数据的量级, 不能滥用.

当然也不一定通过 Memory 对象来管理上下文记忆, 仍然可以自己定义一个服务去管理.

## Memory 锁机制

某些特殊的场景下, 可能会有多个请求同时修改同一个 Memory 对象. 为此系统提供了一个简单的锁机制, ```Commune\Chatbot\OOHost\Context\Memory\Memory::lock()``` 与 ```Commune\Chatbot\OOHost\Context\Memory\Memory::unlock()```. 请慎重使用.

这种场景还是违背了上下文记忆的初衷, 建议用别的服务去实现, 而不是用 Memory 对象.


### 将 Memory 作为上下文使用

和 Intent 可以作为上下文使用一样, Memory 也可以当成有记忆的上下文来使用.

系统自带的 ```Commune\Chatbot\App\Memories\MemorialTask``` 就是一个基类.
Demo 用例中有一个问答对话 ```Commune\Components\Demo\Cases\Questionnaire\ReadPersonality```就是这么实现的.
推荐了解.

### 确保加载

所有继承自 ```Commune\Chatbot\OOHost\Context\Memories\MemoryDef``` 的记忆, 都实现了 ```Commune\Chatbot\OOHost\Context\SelfRegister``` 接口.

因此可以在项目启动时接受扫描,  将自己注册到```Commune\Chatbot\OOHost\Context\Contracts\RootIntentRegistrar``` 之中. 关于注册机制, 请查看 [Registrar 一节](/docs/dm/registrar.md).

要保证 Memory 类会在项目启动时被扫描到, 默认的做法是在机器人配置数组的```$chatbotConfig->host->autoloadPsr4``` 数组中按照 [PSR-4 规范](https://www.php-fig.org/psr/psr-4/) 注册好命名空间和目录的映射关系.
具体可以查看配置对象 ```Commune\Chatbot\Config\Children\OOHostConfig```.


## 在 Session 中直接使用 Memory

有一些场景, 例如基于配置文件生成的上下文, 并不希望手动定义一个 Memory 类, 或是用类名来获取记忆.

因此, CommuneChatbot 也允许在配置中定义 Memory, 并通过 Session 对象获取.

简单而言, 凡是在机器人配置```$chatbot->host->memories``` 位置 (查看```Commune\Chatbot\Config\Children\OOHostConfig```) 上,
用```Commune\Chatbot\Config\Options\MemoryOption``` 格式定义的记忆,
都可以通过 Session 对象直接获取:

```php

// 用数组方式直接获取
$value = $session->memory[$memoryName][$key];

// 用类定义的也可以这么获取
$name = $session->memory[UserInfoMemory::class]['name'];
```

这实际上就是使用```Commune\Chatbot\OOHost\Context\Memory\MemoryBagDefinition``` 在 ```Commune\Chatbot\OOHost\Context\Contracts\RootMemoryRegistrar``` 中快速定义了 Memory. 了解该机制实现, 可以在任何地方用配置定义上下文记忆.

具体请看```Commune\Chatbot\OOHost\HostProcessServiceProvider```.

## Memory 的存取

[CommuneChatbot 的工作站](https://github.com/thirdgerb/studio-hyperf) 自带了一个基于 Mysql 实现的 Memory 存储. 在 ```Commune\Hyperf\Foundations\Drivers\SessionDriver```. 这个方案把 Memory 存在 Mysql 中, 而把其它 Context 的值存在 Redis 中.

您可以根据业务的需要, 自己定义存取的方式, 只需要注册自己的 ```Commune\Chatbot\OOHost\Session\Driver``` 实现到机器人配置中, 覆盖原来的服务就可以了.

