# Session

多轮对话上下文的所有数据都通过 ```Commune\Chatbot\OOHost\Session\Session``` 对象来管理. 具体实现是```Commune\Chatbot\OOHost\Session\SessionImpl```. 通过```Commune\Chatbot\OOHost\HostConversationalServiceProvider``` 注册.

一个 Chat 可能拥有若干次多轮对话, 场景不同, 多轮对话的起点也不同, 一直进行到对话主动结束(调用了```Dialog::quit()```)或者超时, 才结束一次多轮对话.

每一次多轮对话都会有一个独立的 Session, 通过```Session::$sessionId``` 来唯一标记.


## 获取 Session

Session 的主要方法都由系统来调度. 获取 Session 实例的主要目的, 是获取 Session 管理的各种数据. 常用的有 :

- ```Session::$conversation``` : Conversation 实例
- ```Session::$incomingMessage``` : 输入的消息
- ```Session::$nlu``` : 存储自然语言处理结果的 NLU 模块
- ```Session::$logger``` : 日志模块, 会记录当前 SessionId 作为参数之一
- ```Session::$chatbotConfig``` : 机器人的配置
- ```Session::$hostConfig``` : 多轮对话模块的配置
- ```Session::$contextRepo``` : 注册了所有 Context 的仓库
- ```Session::$intentRepo``` : 注册了所有意图 ( Intent ) 的仓库, 从属 $contextRepo
- ```Session::$memoryRepo``` : 注册了所有记忆 ( Memory ) 的仓库, 从属 $contextRepo

Session 在 Conversation 中并非单例, 所以不能通过依赖注入来获取. 正常开发中有两种获取 Session 的方式.

第一种方法, 通过定义在配置文件```$chatbotConfig->host->sessionPipes``` 中的 SessionPipe 中间件. Session 会作为传输对象传给这些中间件.

```php

class MyPipe extends Commune\Chatbot\OOHost\Session\SessionPipe
{

    public function handle(Session $session, \Closure $next) : Session
    {
        ...
    }
}

```

常用的 Session 中间件如下:

```EventMsgPipe``` : 将某些事件类消息 (例如通道连接) 转化为别的消息, 例如指令性质的消息.

```MarkedIntentPipe``` : 调试工具, 用户消息若将意图名称用 # 包含, 例如 "#dialog.help#", 会将该意图设置为匹配到的意图.

```NavigationPipe``` : 导航中间件. 可以将一部分意图 (通常是导航类意图, 例如```Commune\Components\Predefined\Intents\Navigation\BackwardInt```) 列为最高优先级, 主动进行匹配. 如果匹配到了这些意图, 会调用它们的 ```navigate()``` 方法进行重定向. 这些意图优先于 Context 的对话逻辑.


第二种方法是通过 Dialog 对象获取 :

```php

    public function __onStart(Stage $stage) : Navigator
    {
        return $stage->talk(
            function(Dialog $dialog){

                // 通过 Dialog 的属性来获取.
                $session = $dialog->session;
            },
            ...
        );
    }

```

## Scope

使用 ```$session->scope``` 可以获得当前 Session 的各种维度参数, 可用于定义一个作用域. 上下文记忆就是根据这个作用域来存取的. 具体实现在```Commune\Chatbot\OOHost\Session\Scope```.

常用的作用域维度有 :

```php

class Scope implements ArrayAndJsonAble
{
    const SESSION_ID = 'sessionId';
    const USER_ID = 'userId';
    const PLATFORM_ID = 'platformId';
    const CHATBOT_USER_ID = 'chatbotName';
    const CHAT_ID = 'chatId';
    const CONVERSATION_ID = 'conversationId';
    const INCOMING_MESSAGE_ID = 'incomingMessageId';
    const DATE = 'date';
    const YEAR = 'year';
    const MONTH = 'month';
    const DAY = 'day';
    const WEEK = 'week';
    const WEEK_DAY = 'weekDay';
    const HOUR = 'hour';
    const MINUTE = 'minute';
```



## 分布式一致性

CommuneChatbot 实现了多轮对话的分布式一致性. 每次单轮对话结束时, Session 相关的数据会被存储到数据库 ( 通常用 Redis 或 Mysql ), 然后收到下一个用户消息时, 从存储介质里将会话数据还原出来.

系统配置中默认的中间件 ```Commune\Chatbot\App\ChatPipe\ChattingPipe``` 会对 Chat 加锁, 避免同一时间唤醒两个相同的 Session, 造成歧义.

Session 的过期时间定义在机器人配置的```$chatbotConfig->host->sessionExpireSeconds```参数, 以秒为单位.

### Driver

Session 的数据存储都通过 ```Commune\Chatbot\OOHost\Session\Driver``` 来进行操作.
 [studio-hyperf](https://github.com/thirdgerb/studio-hyperf) 中的实现是 ```Commune\Hyperf\Foundations\Drivers\SessionDriver```,
上下文信息除了 Memory 存到 Mysql 数据库之外, 其它的数据都存在 Redis 的 Hash中.

另外也提供了```Commune\Chatbot\App\Drivers\Demo\ArraySessionDriver```, 用 PHP 数组来存储相关数据. 这种做法无法实现分布式一致性, 但可以用于本地化的单点机器人 (例如树莓派机器人, 或者命令行对话机器人).

您也可以定义自己的 Session Driver, 将之注册到配置文件的 ```$chatbotConfig->conversationProviders``` 并保证唯一就可以了.

### 不保存上下文

Session 在系统调用 ```Session::finish()``` 时, 会将上下文相关数据进行存储. 但有时我们并不想记录这一轮对话 (例如调用了一个调试命令), 这时需要确定已调用了 ```$session->beSneak()``` 方法, 这样 Session 结束时就会跳过保存状态环节.

### SessionData

通过 Session 进行存取的数据, 大部分都实现了 ```Commune\Chatbot\OOHost\Session\SessionData```.

这样的对象在存储前会进行序列化, 并且可以通过```Commune\Chatbot\OOHost\Session\SessionDataIdentity```作为 ID 查找到, 反序列化还原为对象.

所有的 SessionData 在 Session 中根据 ID 只会存储一份. Session 相当于所有多轮对话上下文数据的公共内存.

因此, 在相互持有的情况下, 例如 ContextA 将 ContextB 作为属性, 实际上只需要持有 SessionDataIdentity 就可以了.

### SessionInstance

如上所述, Session 相当于多轮对话上下文数据的公共内存. 因此有一个新的上下文数据时, 必须将它加入到 Session 中. 这种类型的数据实现了 ```Commune\Chatbot\OOHost\Session\SessionInstance```,
必须通过```toInstance($session)```方法注册到 Session 中, 才是真实可用的. 例如 :


```php

    public function __onDependTo(Stage $stage) : Navigator
    {
        // Context 允许将另一个 Context 作为自己的参数
        $target = new TargetContext($this);

        // 通过 to Instance 方法注册到 Session
        // 这只是示范, 其实没有必要. 因为 dependOn 方法内置了 toInstance
        $target = $target->toInstance($this->getSession());

        return $stage->dependOn(
            $target,
            ...
        );
    }
```

### 不能序列化

Session 只和当前单轮对话有关, 将 Session 序列化没有意义, 因此序列化时会抛出异常. 这也是为了防止某些操作不当, 导致序列化 Session 将海量的数据变为字符串试图存储.