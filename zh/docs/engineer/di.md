# 双容器与依赖注入

控制反转容器 (Inversion of Control Container) 与依赖注入 (dependency injection) 已经成为现代工程的标配, 通过它们可实现面向接口 (interface) 编程, 极大地方便松耦合的组件式开发与测试.

简单而言, 应用所有的工厂方法都要注册到一个 IoC 容器中, 然后通过 IoC 容器获取实例 (见 [psr-11 container](https://packagist.org/packages/psr/container)). 而类的构造方法, 或者方法的参数, 自动从 IoC 容器中获取, 从而实现依赖注入.

关于 IoC 容器的更多了解, 可以参考[laravel 的文档](https://laravel.com/docs/6.x/container).

CommuneChatbot 项目全面支持依赖注入, 并且考虑 Swoole 协程的特点, 设计了双容器策略. 本节介绍相关概念以及使用方法.


## 1. 什么是双容器策略

PHP 通常基于多进程模型运行, 每一个进程同一时间只处理一个请求, 而且完成请求后通常会释放资源. 用这种方式处理请求, 不太需要考虑公共资源 (比如静态变量) 的争抢, 污染, 内存泄漏等问题, 所以逻辑写起来清晰简洁.

Swoole 4 实现了[基于协程 + 通道 的CSP编程模型](https://wiki.swoole.com/wiki/page/p-coroutine.html), 可以用同步的编程方式写逻辑, 底层用异步的 IO. 这样一个请求由一个协程任务负责处理, 而 Worker 进程可能同时调度多个协程.


这样就衍生出一个问题. PHP 主流的 IoC 容器 (例如[laravel](https://laravel.com) 的 [illuminate/container](https://packagist.org/packages/illuminate/container))都是基于多进程模型来设计的, 如果一个容器同时应用于多个请求, 那么它持有的单例 (session, request) 很势必相互污染.


CommuneChatbot 设计了双容器策略来解决这一问题. 简单而言, 进程中运行了两种容器:

* 进程级容器
    - 一个进程只有一个实例
    - 所有工厂方法和单例在进程中共享
    - 进程启动时注册服务 (register)
    - 进程启动时初始化 (boot)
* 请求级容器
    - N 个请求有 N 个实例, 请求结束时销毁
    - 所有工厂方法在进程中共享 (通过静态变量 )
    - 所有单例只在请求内共享
    - 持有进程级容器作为父容器, 可以无感地访问所有进程级容器的数据
    - 进程启动时注册服务 (register)
    - 请求启动时初始化 (boot)

这要求服务注册者 (Service Provider) 有意识地区分进程级服务和请求级服务, 但服务使用方 (依赖注入时) 是完全无感的.

通过这种方式, CommuneChatbot 既实现了请求级单例的隔离, 又保证了依赖注入的简洁.


### 1.1 Commune/Container 库

CommuneChatbot 的双容器策略通过 [Commune/Container 库](https://packagist.org/packages/commune/container) 来实现. 主要借鉴了[illuminate/container](https://packagist.org/packages/illuminate/container) 库, 为实现递归容器, 将之修改为```Trait``` .

Container 的 api 请查阅 [Commune\Container\ContainerContract](https://github.com/thirdgerb/container/blob/master/src/ContainerContract.php).


## 2. CommuneChatbot 的双容器

CommuneChatbot 的根应用是 [Commune\Chatbot\Blueprint\Application](https://github.com/thirdgerb/chatbot/blob/master/src/Chatbot/Blueprint/Application.php).
它的默认实现是```Commune\Chatbot\Framework\ChatApp```. 系统所有的运行都基于它:

```php

// 初始化根应用
$chatApp = new Commune\Chatbot\Framework\ChatApp($config);

// 启动服务, 自动监听用户消息
$chatApp->getServer()->run();
```

想要获取根应用, 可以通过依赖注入, 或是调用 :

```php
$app = Commune\Chatbot\Framework\ChatApp::getInstance();
```

根应用持有进程级容器与请求级容器.

* 进程级容器 : ``` $app->getProcessContainer() ```
* 请求级容器 : ``` $app->getConversationContainer() ```

分别对应:

* 进程级容器 : [Commune\Container\ContainerContract](https://github.com/thirdgerb/container/blob/master/src/ContainerContract.php)
* 请求级容器 : [Commune\Chatbot\Blueprint\Conversation\ConversationContainer](https://github.com/thirdgerb/chatbot/blob/master/src/Chatbot/Blueprint/Conversation/ConversationContainer.php)

## 3. 服务注册

Laravel 为代表的 ```ServiceProvider``` 机制是目前注册服务的最佳实践, 将注册服务的逻辑定义在独立的类文件中, 方便管理. 除此之外也有类似```java spring```的基于文件扫描与注解的方式来注册.

### 3.1 Service Provider

关于 Service Provider 的用法, 可以参考 [Laravel 的文档](https://laravel.com/docs/6.x/providers), CommuneChatbot 试图和 Laravel 的做法保持一致性.

CommuneChatbot 的 ```ServiceProvider``` 基类是 [Commune\Chatbot\Blueprint\ServiceProvider](https://github.com/thirdgerb/chatbot/blob/master/src/Chatbot/Blueprint/ServiceProvider.php)

这是一个具体的示例 :

```php
/**
 * Sound like 的服务注册
 * 可以自己写一个 process service provider, 注册自己想要的 parser
 */
class SoundLikeServiceProvider extends BaseServiceProvider
{
    const IS_PROCESS_SERVICE_PROVIDER = true;

    public function boot($app)
    {
    }

    public function register()
    {
        // $this->app 获得当前容器的实例, 并不需要关心是进程级还是请求级.
        if ($this->app->bound(SoundLikeInterface::class)) {
            return;
        }

        $this->app->singleton(SoundLikeInterface::class, function(){
            $pinyin = new Pinyin(MemoryFileDictLoader::class);
            $zhParser = new PinyinParser($pinyin);
            $manager = new SoundLikeManager();
            $manager->registerParser(SoundLikeInterface::ZH, $zhParser);
            return $manager;
        });
    }
}
```

> 注意常量 ```IS_PROCESS_SERVICE_PROVIDER```, 用来标记一个 ```ServiceProvider``` 是进程级的, 还是请求级的. 如果注册错误, 启动的时候会在 stdio 提示 warning 级别的错误.

容器会遍历所有已注册的 ```ServiceProvider```, 执行 ```ServiceProvider::register```方法以注册所有的工厂方法. 之后才会在合适的时机执行```ServiceProvider::boot```方法, 以保证每个组件完成初始化.

Service Provider 可以通过 ```$this->app``` 获得 IoC 容器的实例, 通过它来注册服务. 可用的 API 请查看```Commune\Container\ContainerContract```. 基本上和 Laravel 风格保持一致, 但在```Container::instance()``` 有所不同. 常用方法如下:

```php
    // Container 是这个 $app 的类.
    $app = new Container();

    // 用闭包作为工厂, 定义一个单例
    $app->singleton(ClassName1::class, function(){...});

    // 用类名来描述单例的实现, ImplementsName 也会在构造时进行依赖注入
    $app->singleton(InterfaceName::class, ImplementsName::class);

    // 注册工厂方法, 但不是单例
    $app->bind(ClassName::class, function(){...});

    // 用字符串作为抽象, 标记工厂方法. 无法用于依赖注入
    $app->bind('cache', function(){...});


    $object = new SomeClass();

    // 绑定一个实例作为单例, Container 自身所有的容器实例都持有这个实例
    $app->instance('instance1', $object);


    // 绑定一个实例作为单例, 只有 $app 才持有这个实例, Container 其它实例则不持有.
    $app->share('instance2', $object);

```


### 3.2 在配置中注册服务

CommuneChatbot 可以在配置文件中定义需要注册的服务. 配置文件的详细说明在[配置体系](/docs/engineer/configuration.md) 中. 打开 [ChatbotConfig](https://github.com/thirdgerb/chatbot/blob/master/src/Chatbot/Config/ChatbotConfig.php) 可以查看具体的配置.

影响服务注册的是三个关键的数组:

    // 系统预注册的服务, 必须存在
    'baseServices' => BaseServicesConfig::stub(),

    // 自定义的进程级组件
    'processProviders' => [
    ],

    // 自定义的请求级组件
    'conversationProviders' => [
    ],

数组接受的值是 ```ServiceProvider``` 的类名. 以系统自带的预注册服务([BaseServicesConfig](https://github.com/thirdgerb/chatbot/blob/master/src/Chatbot/Config/Children/BaseServicesConfig.php))为例:

```php

use Commune\Chatbot\Framework\Providers;

return [
    // 异常处理
    'exp' => Providers\ExpHandlerServiceProvider::class,
    // 配置中心抽象层
    'optionRepo' => Providers\OptionRepoServiceProvider::class,
    // 翻译组件
    'translation' => Providers\TranslatorServiceProvider::class,
    // 回复渲染
    'render' => Providers\ReplyRendererServiceProvider::class,
    // 默认日志
    'logger' => Providers\LoggerServiceProvider::class,
    // 事件模块
    'event' => Providers\EventServiceProvider::class,
    // 发音匹配
    'soundLike' => Providers\SoundLikeServiceProvider::class,
    // 请求级服务
    'conversational' => Providers\ConversationalServiceProvider::class,
    // 多轮对话内核 进程级服务
    'hostProcess' => HostProcessServiceProvider::class,
    // 多轮对话内核 请求级服务
    'hostConversation' => HostConversationalServiceProvider::class,
];
```

查看这些注册器, 可以了解 CommuneChatbot 的基础功能模块都是如何定义的.

### 3.3 直接通过容器注册服务

在一些特殊的情况下, 比如在 [组件](/docs/components/index.md) 中, 可以获取到系统的根应用 ([Commune\Chatbot\Blueprint\Application](https://github.com/thirdgerb/chatbot/blob/master/src/Chatbot/Blueprint/Application.php)), 也可以通过它来直接注册服务 :

```php

    // provider 可以是类名, 或者 ServiceProvider 实例
    // 优先级从上往下.

    // 注册配置级服务, 优先级最高
    $app->registerConfigService($provider);

    // 注册进程级服务
    $app->registerProcessService($provider);

    // 注册请求级服务
    $app->registerConversationService($provider);

```


## 4. 获取服务的方式

### 4.1 通过依赖注入

CommuneChatbot 有以下环节实现了依赖注入:

__ServiceProvider__ : 以类名方式注册的```ServiceProvider```, 构造方法都实现了依赖注入.

__管道__ : 关于管道, 详情见[管道的文档](/docs/engineer/pipeline.md). 系统级的```ChatPipe```, 和对话管理层的```SessionPipe``` 都对构造器 ```__construct``` 方法实现了依赖注入.

例如:
```php
class MessengerPipe implements InitialPipe
{
    /**
     * @var Application
     */
    public $app;

    public function __construct(Application $app)
    {
        $this->app = $app;
    }
```

__事件__ : 关于事件, 详情见[事件的文档](/docs/engineer/dispatcher.md). 所有事件的```listener```的构造方法都实现了依赖注入.

__命令__ : 关于命令, 详情见[命令的文档](/docs/dm/command.md). 所有命令的构造器```__construct```方法都实现了依赖注入.

__Stage中的callable对象__ : 定义```Stage```过程中的```callable```对象也都实现了依赖注入. 具体而言, 如果```callable```是 :

* function 名称 : 对入参进行依赖注入
* 可执行的object : 对```__invoke```方法进行依赖注入
* 闭包 : 对参数进行依赖注入
* array(类名, 静态方法名) : 对方法进行依赖注入
* array(类名, 动态方法名) : 对类的构造器进行依赖注入, 对执行方法也进行依赖注入
* array($object, 动态方法名) : 对执行方法进行依赖注入

例如 :

```php
    public function __onStart(Stage $stage) : Navigator
    {
        return $stage->talk(
            // 依赖注入
            function(Dialog $dialog, Corpus $corpus) {...},
            function(Dialog $dialog, NLU $nlu) {...}
        );
    }
```

### 4.2 通过容器获取

系统的进程级容器 (```ProcessContainer```) 和请求级容器 (```ConversationContainer```) 都实现了 [psr-11](https://www.php-fig.org/psr/psr-11/), 所以能拿到容器时, 可以运行:

```php
    // 通过请求级容器获取实例
    $conversation->get($abstractName);

    // 通过进程级容器获取实例
    $processContainer->get($abstractName);
```

请求级容器```Conversation``` 通常只能从依赖注入中获取. 但进程级容器可以通过根应用获取:

```php
    $ioc = Commune\Chatbot\Framework\ChatApp::getInstance()->getProcessContainer();
```

## 5. 系统默认的服务

### 5.1 进程级服务

以下是项目启动时注册的服务, 都是```进程级单例```

|interface                                  | 简介 |
|-                                          |- |
|Commune\Chatbot\Blueprint\Application      |系统的根应用 |
|Commune\Chatbot\Blueprint\ChatKernel           |负责响应消息的内核 |
|Commune\Chatbot\Contracts\ChatServer       |负责运行循环响应的服务端 |
|Commune\Chatbot\Contracts\ConsoleLogger    |系统输出到 console 的日志模块 |
|Commune\Chatbot\Contracts\ExceptionReporter |上报异常, 比如提交给 Sentry |
|Commune\Chatbot\Contracts\EventDispatcher  |系统默认的事件机制 |
|Psr\Log\LoggerInterface                    |系统默认的日志模块 |
|Commune\Support\OptionRepo\Contracts\OptionRepository  |配置中心抽象层仓库 |
|Commune\Support\SoundLike\SoundLikeInterface           |判断两个字符串读音是否相似 |
|Commune\Chatbot\Blueprint\Conversation\Renderer        |用于渲染回复消息的模块 |
|Commune\Chatbot\Contracts\Translator       |默认的翻译模块, 实现i18n |

### 5.2 系统配置

以下是系统默认加载的配置, 都是```进程级的单例```

|interface                                      | 简介 |
|-                                              |- |
|Commune\Chatbot\Config\ChatbotConfig           |chat系统的默认配置 |
|Commune\Chatbot\Config\Children\OOHostConfig   |多轮对话内核配置 |
|Commune\Components\Predefined\PredefinedComponent |预定义组件, 加载系统默认的意图和默认语料 |
|Commune\Chatbot\OOHost\NLU\NLUComponent        |NLU组件, 加载系统默认的语料库 |

### 5.3 请求级服务

以下是在请求生命周期内, 可以通过```Conversation::get``` 获取的```请求级单例```.

|interface                                              | 简介 |
|-                                                      |- |
|Commune\Chatbot\Blueprint\Conversation\Conversation    |管理请求所有数据的容器|
|Commune\Chatbot\Blueprint\Conversation\MessageRequest  |对请求的封装, 负责转义|
|Commune\Chatbot\Blueprint\Conversation\ConversationLogger |请求级的日志模块|
|Commune\Chatbot\Contracts\CacheAdapter                 |系统默认的缓存模块 |
|Commune\Chatbot\Contracts\ClientFactory                |用于创建guzzle客户端 |
|Commune\Chatbot\Blueprint\Conversation\IncomingMessage |通过Request转义后的输入消息|
|Commune\Chatbot\Blueprint\Conversation\User            |从请求中获取的用户信息 |
|Commune\Chatbot\Blueprint\Conversation\Chat            |请求所属的会话 |
|Commune\Chatbot\Blueprint\Conversation\Speech          |用于生成文本回复的模块 |
|Commune\Chatbot\Blueprint\Conversation\NLU             |封装输入消息的自然语言解析数据|

### 5.4 对话管理服务 (请求级)

以下是对话管理内核定义的```请求级服务```

|interface                                      | 简介 | 是否单例 |
|-                                              |- |- |
|Commune\Chatbot\OOHost\Session\Session         |对会话上下文数据的封装 |false|
|Commune\Chatbot\OOHost\Dialogue\Hearing        |用于理解输入消息的模块 |false|
|Commune\Chatbot\OOHost\Session\Driver          |Session用于读写的模块 |true|
|Commune\Chatbot\OOHost\NLU\Contracts\NLULogger |记录NLU解析情况的模块 |true|

### 5.5 对话管理服务 (进程级)

以下是对话管理内核定义的```进程级单例```

|interface                                      | 简介 |
|-                                              |- |
|Commune\Chatbot\OOHost\Emotion\Feeling         |判断用户输入消息是否符合某种情绪|
|Commune\Chatbot\OOHost\Context\Contracts\RootContextRegistrar |ContextDefinition的根仓库 |
|Commune\Chatbot\OOHost\Context\Contracts\RootMemoryRegistrar |MemoryDefinition的根仓库 |
|Commune\Chatbot\OOHost\Context\Contracts\RootIntentRegistrar |IntentDefinition的根仓库 |
|Commune\Chatbot\OOHost\NLU\Contracts\Corpus    |本地的语料库 |
|Commune\Chatbot\OOHost\NLU\Contracts\EntityExtractor |基于本地语料库的简单实体匹配模块|


### 5.6 从 Hyperf 框架继承服务

CommuneChatbot 的工作站 [studio-hyperf](https://github.com/thirdgerb/studio-hyperf) 底层使用 [Hyperf 框架](https://www.hyperf.io/). ```Hyperf框架``` 有自己的容器和服务管理机制, 详见 [Hyperf 的依赖注入文档](https://hyperf.wiki/#/zh/di)

为避免 Hyperf 注册的服务和 CommuneChatbot 产生绑定冲突, 不能从 CommuneChatbot 的容器中直接获取 Hyperf 中注册的服务, 但可以获取到 Hyperf 自己的 ```Container```, 通过它再获取其它服务.

```Hyperf``` 框架在```CommuneChatbot``` 的默认```进程级单例```为:

|interface                          | 简介 |
|-                                  |- |
|Psr\Container\ContainerInterface   |```Hyperf```的依赖注入容器|
|Commune\Hyperf\Foundations\ProcessContainer::HYPERF_CONTAINER_ID | 容器绑定的别名 |
|"hyperf.container"                 |与上相同|
|Commune\Hyperf\Foundations\Options\AppServerOption|```CommuneChatbot```在```Hyperf```中的配置|
|Hyperf\Guzzle\ClientFactory        |```Hyperf```生成```guzzle```协程客户端的工厂|
|Hyperf\Redis\RedisFactory          |```Hyperf```获取```redis```协程客户端的工厂|
|Hyperf\Database\ConnectionResolverInterface |```Hyperf```数据库连接的协程客户端|


如果需要从```Hyperf```框架中继承更多服务 (只能继承单例) 到```CommuneChatbot```, 则可以在配置```Commune\Hyperf\Foundations\Options\AppServerOption``` 中定义```shares``` 数组.

这个数组里的单例, 会被```CommuneChatbot``` 的进程级容器所继承.


## 6. 内存泄漏排查

双容器策略将请求封装到独立的 IoC 容器实例 ```Commune\Chatbot\Blueprint\Conversation\Conversation``` 中, 带来了一个额外的好处, 就是主要服务的内存泄漏更好排查了, 因为相互持有很可能导致容器自身也无法释放.

系统提供了```Commune\Chatbot\Blueprint\Conversation\RunningSpy``` 接口和 ```Commune\Chatbot\Framework\Conversation\RunningSpyTrait``` 来记录进程内单例的数量.

在机器人运行后, 给它输入 ```/runningSpy``` 命令, 会告诉我们主要类的实例数量. 如果哪个类的实例数随着请求线性上升了, 那就一定发生了内存泄漏.