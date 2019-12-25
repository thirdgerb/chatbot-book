# 修改机器人配置

CommuneChatbot 致力于实现高度组件化和可配置.
许多的功能和模块都可以通过配置文件进行修改.

项目使用了```Commune\Support\Option``` 类用于封装配置.
简单来说, 用数组方式定义的配置, 会被封装到一个 Option 子类的实例中.
该实例作为 [进程级单例](/zh-cn/engineer/di.md) 可以在各处依赖注入.
然后用强类型链式调用的方式来获取配置.
这种做法在 可读性/安全性/重构 等方面都带来便利.
具体内容请查看 [配置体系文档](/zh-cn/engineer/configuration.md).

工作站自带的配置主要用于示范.
新建机器人, 推荐创建全新的配置, 也可以了解机器人功能的各个方面.

## 工作站核心配置

工作站 [commune/studio-hyperf](http://packagist.org/packages/commune/studio-hyperf)
关于对话机器人的核心配置文件在 ```BASE_PATH/config/autoload/commune.php``` 文件.

这里定义了工作站默认的各种机器人应用 :

```php
return [

    // tinker 机器人的配置
    'tinker' => include BASE_PATH . '/config/commune/apps/tinker.php',

    // 可以通过 commune:start 命令启动的, 真实客户端.
    // 配置内容请查看 Commune\Hyperf\Foundations\Options\AppServerOption
    'apps' => [
        // 默认的tcp端. 通常供测试用.
        'tcp' => include BASE_PATH . '/config/commune/apps/tcp.php',

        // 系统自带的 web 端, 示范如何开发 web 端的对话机器人.
        'web' => include BASE_PATH . '/config/commune/apps/web.php',

        // 系统自带的 api 端, 可以像 mvc 框架那样通过 http api 访问机器人的数据
        'api' => include BASE_PATH . '/config/commune/apps/api.php',

        // 系统自带的 dueros 端, 可以连接小度音箱设备.
        'dueros' => include BASE_PATH . '/config/commune/apps/dueros.php',

        // 系统自带的 微信公众号服务端. 可作为微信公众号的服务.
        'wechat' => include BASE_PATH . '/config/commune/apps/wechat.php',
    ],
];
```

修改它们, 可以创建或删除机器人应用. CommuneChatbot 的设计思路是一套机器人代码, 可以运用于多个平台.
因此工作站并非只提供一个平台的应用, 而是可以在多个平台启动.

## 应用服务端配置

工作站可以运行的每个机器人应用, 都需要启动一个 Hyperf 的服务端实例, 对应一个独立的配置文件.

为了方便区分和管理,
这些应用配置文件都放在了目录```BASE_PATH/config/commune/apps/``` 目录下.
并通过 ```include BASE_PATH . '/config/commune/apps/web.php'``` 这样的形式,
引进工作站机器人配置文件 ```BASE_PATH/config/autoload/commune.php```.

这些应用配置的结构定义在 ```Commune\Hyperf\Foundations\Options\AppServerOption``` 类.
结构如下 :

```php

use Hyperf\Server\Server;
use Hyperf\Server\SwooleEvent;
use Hyperf\Framework\Bootstrap;

// 从另一个文件中, 获取机器人的配置数组
$chatbot = include BASE_PATH . '/config/commune/chatbots/dueros.php';

/**
 * 返回机器人配置, 结构定义在 AppServerOption 类
 * @see \Commune\Hyperf\Foundations\Options\AppServerOption
 */
return [

    // 机器人配置
    'chatbot' => $chatbot,

    // redis 使用 hyperf 提供的哪一个 redis 连接池
    // 连接池定义在 BASE_PATH/config/autoload/redis.php
    'redisPool' => 'dueros',

    // db 连接使用 hyperf 提供的哪一个连接池
    // 定义在 BASE_PATH/config/autoload/database.php
    'dbPool' => 'default',

    // 是否缓冲消息. 是的话, 消息通常先写入一个 message queue
    'bufferMessage' => true,

    // 从 Hyperf 容器, 引入到机器人进程级容器的单例.
    'shares' => [],

    // hyperf 的 server 配置.
    'server' => [
        'mode' => SWOOLE_PROCESS,
        // 定义监听端口的 swoole 实例
        'servers' => [
            [
                // 实例名称
                'name' => 'dueros_',
                // 端口所用协议
                'type' => Server::SERVER_HTTP,
                // host 地址, ip
                'host' => 'localhost',
                // 监听端口
                'port' => intval(env('CHAT_DUEROS_PORT', 9529)),
                // socket 类型, tcp, udp 等
                'sock_type' => SWOOLE_SOCK_TCP,
                // 该协议下, 各种回调事件调用的方法
                'callbacks' => [
                    SwooleEvent::ON_REQUEST => [\Commune\Platform\DuerOS\Servers\DuerChatServer::class, 'onRequest'],
                ],
            ]
        ],

        // server 的公共配置
        'settings' => [
            // 是否开启协程
            'enable_coroutine' => true,
            // worker 进程数量
            'worker_num' => 1,
            // pid 文件地址
            'pid_file' => BASE_PATH . '/runtime/pid/dueros.pid',
            'open_tcp_nodelay' => true,
            // worker进程最大协程任务数量
            'max_coroutine' => 100000,
            'open_http2_protocol' => true,
            // 最大响应请求数. 达到了之后, 该 worker 进程会关闭重启.
            // 用于防止内存泄漏
            'max_request' => 100000,
            'socket_buffer_size' => 2 * 1024 * 1024,
        ],

        // 定义各种 swoole 事件的响应逻辑
        'callbacks' => [
            SwooleEvent::ON_BEFORE_START => [Bootstrap\ServerStartCallback::class, 'beforeStart'],
            SwooleEvent::ON_WORKER_START => [Bootstrap\WorkerStartCallback::class, 'onWorkerStart'],
            SwooleEvent::ON_PIPE_MESSAGE => [Bootstrap\PipeMessageCallback::class, 'onPipeMessage'],
        ],
    ],
];

```

这些配置主要和 Hyperf 框架有关, 可能需要了解的文档有:

- [swoole server](https://wiki.swoole.com/wiki/page/p-server.html)
- [hyperf 协程](https://doc.hyperf.io/#/zh-cn/coroutine)
- [hyperf 数据库配置](https://doc.hyperf.io/#/zh-cn/db/quick-start)
- [hyperf 连接池](https://doc.hyperf.io/#/zh-cn/pool)

通过这些配置, 可以修改服务端实例的运行参数.

## 机器人配置

服务端实例的配置文件中, 包含机器人的配置(```$appServerOption->chatbot```).
由于机器人的配置比较庞大, 将它们拆分到另一个文件夹```BASE_PATH/config/commune/chatbots/```中方便管理.

其中每一个文件是一个机器人的配置, 结构定义在 ```\Commune\Chatbot\Config\ChatbotConfig``` 类.

以 ```BASE_PATH/config/commune/chatbots/dueros.php``` 为例, 完整的配置应该是:

```php
use Commune\Studio\Providers;
use Commune\Studio\SessionPipes;

/**
 * 机器人的默认配置.
 *
 * @see \Commune\Chatbot\Config\ChatbotConfig
 */
$chatbot = [

    // 系统用 chatbotName 来隔离会话. 必须要填.
    'chatbotName' => 'commune-dueros-demo',

    // 会被 botOption 的 debug 覆盖.
    'debug' => env('CHATBOT_DEBUG', false),

    // ChatServer 使用的类
    'server' => \Commune\Hyperf\Foundations\HyperfChatServer::class,

    // 在这里可以预先绑定一些用 Option 类封装的配置.
    // 会将该配置预绑定到worker容器上, 作为单例.
    // 有三种绑定方式:
    // 1. 只写类名, 默认使用 stub 里的配置.
    // 2. 类名 => 数组,  会用数组内的值覆盖 stub 的相关参数.
    // 3. 类名 => 子类名, 会用子类的实例来绑定父类类名.
    'configBindings' => [
    ],


    // 预加载的组件. 使用方法类似 configBindings
    // 但component 不仅会预加载配置, 而且还能注册各种组件, 进行初始化等.
    'components' => [

        // 百度音箱配置文件
        \Commune\Platform\DuerOS\DuerOSComponent::class => [
            // 校验用的私钥地址
            'privateKey' => env('DUEROS_PRIVATE_KEY', ''),
        ],

        // 官方的基础 demo
        \Commune\Hyperf\Demo\HyperfDemoComponent::class,

        // 系统自带的 NLU 单元 配置
        // 包括本地语料库, 自然语言单元等配置
        \Commune\Chatbot\OOHost\NLU\NLUComponent::class,

        // 对话冒险游戏组件.
        \Commune\Components\Story\StoryComponent::class;

        // 疑案追声模式 demo 组件
        \Commune\Components\UnheardLike\UnheardLikeComponent::class,

        // 闲聊组件
        \Commune\Components\SimpleChat\SimpleChatComponent::class,

        // 简单的文件 wiki 组件
        \Commune\Components\SimpleWiki\SimpleWikiComponent::class,

        // rasa 组件
        \Commune\Components\Rasa\RasaComponent::class => [
            // 服务端地址
            'server' => env('RASA_API', 'localhost:5050'),
            // 语料库输出地址
            'output' => BASE_PATH . '/rasa-demo/data/nlu.md',
            // domain 配置地址.
            'domainOutput' => BASE_PATH . '/rasa-demo/domain.yml',
        ],


    ],

    // 系统默认的服务注册.
    'baseServices' => \Commune\Chatbot\Config\Children\BaseServicesConfig::stub(),

    // 进程级别的服务注册
    'processProviders' => [
        // 注册 feel emotion 模块
        'feeling' => Providers\FeelingServiceProvider::class,
        // register chatbot event
        'event' => Providers\EventServiceProvider::class,
        // 公共的rendering
        'render' =>  Providers\RenderServiceProvider::class,
        // 权限识别
        'ability' => Providers\AbilityServiceProvider::class,
    ],

    // 在worker中注册的服务, 多个请求共享
    'conversationProviders' => [
        // hyperf client driver . redis, db
        // hyperf 的协程客户端
        'client' => \Commune\Hyperf\Foundations\Providers\ClientDriverServiceProvider::class,
        // cache adapter driver
        // 实现 chatbot 需要的 cache adapter
        'cache' => \Commune\Hyperf\Foundations\Providers\CacheServiceProvider::class,
        // oo host session driver
        'session' => \Commune\Hyperf\Foundations\Providers\SessionServiceProvider::class,
        // message request service
        'message' => \Commune\Hyperf\Foundations\Providers\MessageQueueServiceProvider::class,
    ],

    // 系统接收到请求后执行的管道
    'chatbotPipes' => [
        'onUserMessage' => [
            // 发送 conversation, 并且管理所有的异常
            \Commune\Chatbot\App\ChatPipe\UserMessengerPipe::class,
            // 用于锁定 chat, 避免用户输入消息太频繁导致歧义
            \Commune\Chatbot\App\ChatPipe\ChattingPipe::class,
            // 多轮对话管理内核
            \Commune\Chatbot\OOHost\OOHostPipe::class,
        ],
    ],

    // 消息请求加锁的自动过期时间
    'chatLockerExpire' => 3,


    /**
     * 翻译模块配置
     * @see \Commune\Chatbot\Framework\Providers\TranslatorServiceProvider
     */
    'translation' => [
        // 默认配置文件的类型. 也支持 yaml 等.
        'loader' => 'php',
        // 语言包所在的目录, 目录下每一个子目录对应一种语言.
        // 例如 /resources/langs/zh 对应 zh
        // 每个目录里应该都有一个 messages.php 文件, 而且文本变量应该按照 intl 方式定义.
        'resourcesPath' => BASE_PATH . '/resources/langs',
        // 默认使用的语言包.
        'defaultLocale' => 'zh',
        // cache 文件所在路径.
        'cacheDir' => BASE_PATH . '/runtime/cache/langs/',
    ],

    /**
     * 日志文件配置
     * @see Monolog\Handler\StreamHandler
     */
    'logger' => [
        // 日志文件路径
        'path' => BASE_PATH . '/runtime/logs/commune-dueros.log',
        // 日志保存的天数, 默认每天会生成一个日志文件
        'days' => 7,
        // 记录日志的级别,
        'level' => \Psr\Log\LogLevel::INFO,
        'bubble' => true,
        // 指定日志文件的权限.
        'permission' => NULL,
        // 写日志文件时是否加锁.
        'locking' => false,
    ],

    // 系统默认的slots, 所有的reply message 都会使用
    // 多维数组会被抹平
    //  例如 self => [ 'name' => 'a'],  会变成 self_name 这样的形式
    // default reply slots
    // multi-dimension array will be flatten to dot pattern
    // such as 'self.name'
    'defaultSlots' => include __DIR__ . '/../configs/slots.php',

    // 系统在一些特殊场景, 默认回复用户的消息
    'defaultMessages' => \Commune\Chatbot\Config\Children\DefaultMessagesConfig::stub(),

    // 多轮对话管理内核的配置
    'host' => [

        // 默认的根语境名
        'rootContextName' => \Commune\Hyperf\Demo\Contexts\DemoHome::class,

        // 不同场景下的根语境名.
        'sceneContextNames' => [

            // 项目介绍, 使用 SimpleWiki
            'introduce' => 'sw.demo.intro',
            // 项目有什么特点
            'special' => 'sw.demo.intro.special',
            // 游戏的入口
            'game' => \Commune\Components\Demo\Contexts\GameTestCases::class,

            'unheard' => 'unheard-like.episodes.who-is-lizhongwen',
            // 对话冒险游戏的入口
            'story' => 'story.examples.sanguo.changbanpo',
            // nlu 测试工具
            'nlu' => \Commune\Components\Demo\Contexts\NLTestCases::class,
            // 迷宫小游戏
            'maze' => \Commune\Components\Demo\Cases\Maze\MazeInt::getContextName(),
            // 开发工具
            'dev' => \Commune\Hyperf\Demo\Contexts\DevTools::class,
        ],


        // 可 "返回上一步" 的最大次数
        'maxBreakpointHistory' => 2,
        // 单次请求, 多轮对话状态变更的最大次数
        'maxRedirectTimes' => 20,
        // 是否记录多轮对话状态变更路径 到日志
        'logRedirectTracking' => true,
        // Session 的过期时间
        'sessionExpireSeconds' => 3600,
        // 默认扫描, 加载的 Context 类所在路径, 按 psr-4 规范
        'autoloadPsr4' => [
            "Commune\\Studio\\Contexts\\" => BASE_PATH . '/app-studio/Contexts',
        ],

        // 处理每一个消息请求需要经过的中间件
        'sessionPipes' => [
            // event 转 message
            // transfer curtain event messages to other messages
            0 => \Commune\Chatbot\App\SessionPipe\EventMsgPipe::class,

            // 单纯用于测试的管道,#intentName# 模拟命中一个意图.
            // use "#intentName#" pattern to mock intent
            1 => \Commune\Chatbot\App\SessionPipe\MarkedIntentPipe::class,

            // 用户可用的命令.
            2 => SessionPipes\UserCommandsPipe::class,

            // 超级管理员可用的命令. for supervisor only
            3 => SessionPipes\AnalyseCommandsPipe::class,

            // 4. 的位置留给 NLU 中间件

            // 优先级最高的意图, 通常用于导航.
            // 会优先匹配这些意图.
            // highest level intent
            5 => SessionPipes\NavigationIntentsPipe::class,
        ],

        // 通过配置定义的上下文记忆
        'memories' => [
        ],

        // hearing 模块系统默认的 defaultFallback 方法
        // 在 $dialog->hear()->end() 的时候调用.
        'hearingFallback' => \Commune\Components\SimpleChat\Callables\SimpleChatAction::class,
    ],
];


// 根据是否有 rasa 决定是否开启
$hasRasa = env('RASA_API', '');
if (!empty($hasRasa)) {
    $chatbot['host']['sessionPipes'][4] = \Commune\Components\Rasa\RasaSessionPipe::class;
}

return $chatbot;
```

由于整体的配置数组比较大, 仍可以拆分为多个关联文件.

其中, 比较需要注意的配置项是 :

- ```$chatbotConfig->chatbotName``` 机器人的名称, 决定性的参数. 多平台可通过同名机器人共享信息.
- ```$chatbotConfig->host->rootContextName``` 机器人根语境, 需要替换成自己的.
- ```$chatbotConfig->host->sceneContextNames``` 不同场景下的根语境, 需要替换成自己的.


