# 配置体系

## 1. 机器人基础配置

```CommuneChatbot``` 的启动和运行都围绕着根应用 ```Commune\Chatbot\Blueprint\Application```. 最简单的运行方法如下:

```php

// 获取系统的配置.
$chatbotConfig = include  __DIR__ . '/configs/config.php';

// 必须传入配置数组
$app = new \Commune\Chatbot\Framework\ChatApp($chatbotConfig);
$app->getServer()->run();
```

基本的配置数组```ChatbotConfig```定义了对话机器人的所有功能. 详细配置内容见```Commune\Chatbot\Config\ChatbotConfig``` 类.

CommuneChatbot 使用了基于 ```Commune\Support\Option``` 实现的配置体系, 简单而言就是将配置数组, 封装到一个配置对象中; 从而将弱类型的数组式调用, 变成强类型的对象式调用. 具体用法在下文有介绍.


## 2. 服务端实例配置

在```CommuneChatbot```的工作站[studio-hyperf](https://github.com/thirdgerb/studio-hyperf) 中, 机器人的配置基于[Hyperf 的配置体系](https://hyperf.wiki/#/zh/config).

关键文件在 ```BASE_PATH/configs/autoload/commune.php``` :

```php
return [

    // tinker 机器人的配置
    'tinker' => include BASE_PATH . '/config/commune/chatbots/tinker.php',

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

CommuneChatbot 的基本设计思路, 就是同一个机器人可以在多个端使用. 而上述配置则实现了这样的机制:

* ```php bin/hyperf.php commune:tinker```命令, 可以启动基于命令行的 ```tinker``` 机器人
* ```php bin/hyperf.php commune:start [appName]``` 命令, 可以启动```apps``` 数组中定义的机器人服务端

## 3. 工作站配置

[CommuneChatbot 框架](https://github.com/thirdgerb/chatbot) 只需要配置```ChatbotConfig``` 就可以启动. 但在```Studio-hyperf```中启动, 还要定义Hyperf 自己的 server 配置.

这些配置综合起来, 定义在目录 ```BASE_PATH/config/commune/apps/``` 路径下.

每个文件的数组结构, 都对应 option 类 ```Commune\Hyperf\Foundations\Options\AppServerOption```, 查看该类可以具体了解.

运行 ```php bin/hyperf.php commune:start [appName]``` 实际读取的是这部分配置. 详情可查看```Commune\Hyperf\Commands\StartApp```.

可以查看项目自带的```wechat```, ```dueros``` 等 app 了解配置方式.

## 4. 使用环境变量

CommuneChatbot 可以使用 ```.env``` 文件定义环境变量, 从而将配置中的敏感信息解耦到服务器上定义的```.env``` 文件中.

相关功能基于[Hyperf 的环境变量](https://hyperf.wiki/#/zh/config?id=%e7%8e%af%e5%a2%83%e5%8f%98%e9%87%8f) 来实现.


## 5. 基于 Option 的配置体系

CommuneChatbot 参考了 [tharos/schematic](https://packagist.org/packages/tharos/schematic) 项目来设计配置体系.

### 5.1 什么是 Option

简单来说, 作为配置传入的 PHP数组, 将会封装到一个继承自 ```Commune\Support\Option``` 类中, 作为一个实例来调用.

该对象通过注解的方式定义参数, 并且用属性的方式访问配置. 例如 :

```php

/**
 * @property-read string $option1
 * @property-read string $option2
 */
class TestOption extends Commune\Support\Option
{
    // 决定是单例还是枚举值的常量
    const IDENTITY = '';

    // stub 数组将与传入的数组合并
    // 详情见 Option::__construct
    public static function stub() : array
    {
        return [
           'option1' => 'test',
           'option2' => 'test',
        ];
    }
}

// 仍然用数组来描述配置, 但不需要所有的元素
$testConfigArr = [
    'option1'=>'abc'
];

// 将数组配置封装成对象配置.
$test = new TestOption($textConfigArr);

assert($test->option1 === 'abc');
assert($test->option2 === 'test');

```

更多细节请查看```Commune\Support\Option```类, 或是了解[Tharos/Schematic 文档](https://github.com/Tharos/Schematic) (Option比原库增加了许多功能).

这么做的好处有:

* 从容易写错的数组调用, 变成不能写错的对象调用
* 从数组的弱类型, 变成基于注解的强类型
* IDE 支持效果非常好, 而且可以重构所有的调用
* 定义```stub```数组, 为配置提供了默认值
* 自带```validate```方法, 可以校验配置的合法性
* 可以作为单例绑定到```IoC容器```, 于是可以实现```依赖注入```

### 5.2 关联结构

Option 配置是可以相互嵌套的. 定义了嵌套关系后, 传入数组给 Option,
会自动将某个属性变成另一个 Option.
定义时需要用 ```Option::$associations``` 属性 :

```php
class StoryOption extends Option
{

    // 定义关联结构
    protected static $associations = [

        // 定义 1 对 1 的关联结构
        'rootStorage' => MetaHolderOption::class,

        // 定义 1 对 n 的关联结构, 在键名后面加上 []
        'storagePipeline[]' => MetaHolderOption::class,
    ];

    ...
}
```

### 5.3 Option 的单例与枚举值

定义一个```Commune\Support\Option```允许单例和枚举值两种形式.

当子类的 ```IDENTITY``` 常量值为空时, 调用 ```$option->getId()``` 返回的是类名, 这是认为该 Option 是全局单例的配置.

当子类的 ```IDENTITY``` 常量值为配置的某个字段时, 例如:

```php
class ChatbotConfig extends Commune\Support\Option
{
    const IDENTITY = 'chatbotName';

    public static function stub() : array
    {
        return [
           'chatbotName' => 'test',
           ...
        ];
    }
}

$config = new ChatbotConfig();

assert($config->getId() === 'test');
```

这时认为该 Option 是一系列配置的枚举值.


## 6. 系统的核心配置

- ```Commune\Chatbot\Config\ChatbotConfig``` : 对话机器人所有功能模块的配置
- ```Commune\Chatbot\Config\Children\OOHostConfig``` : 多轮对话内核的配置
- ```Commune\Hyperf\Foundations\Options\AppServerOption``` : ```studio-hyperf```启动应用的基本配置.

请查阅相关类文件了解详细配置内容.


## 7. 注册自定义 Option 单例

可以在```Commune\Chatbot\Config\ChatbotConfig::$configBindings``` 中定义需要全局绑定的单例.

在这里绑定的单例, 系统启动时会自动注册成为```进程级单例```, 从而可以在依赖注入中获取. 具体的加载机制请查看 ```Commune\Chatbot\Framework\Bootstrap\LoadConfiguration```.

绑定的几种方式为:

```php

// 返回 ChatbotConfig 配置数组
return [
    ...

    'configBindings' => [

        // 直接用类名来绑定, 则用 stub 提供的样板数组作为数据
        Option1::class,

        // 用类名做 key, 数组做值, 会得到该 new Option2(['a'=>'abc'])
        Option2::class => [
            'a' => 'abc',
        ],

        // 允许用子类来绑定父类, 这样子类和父类的类名都可以获取该实例
        Option3::class => SonOfOption3::class

        // 允许使用闭包作为工厂方法
        Option4::class => function() {
            return new Option4([]);
        }

    ],

];

```

## 8. 注册组件 ComponentOption 单例

ComponentOption 是一种特殊的```单例 Option```, 它既是一个独立组件的配置, 同时又包含了```ComponentOption::bootstrap()``` 方法, 会在项目启动的时候运行. 具体情况请查看[组件化开发文档](/zh-cn/components/index.md) .

这类配置同样需要注册到 ```Commune\Chatbot\Config\ChatbotConfig::$components``` 数组中. 绑定方式与 ```Commune\Chatbot\Config\ChatbotConfig::$configBindings``` 类似. 并且同样作为```进程级单例```可以依赖注入.

这样, 在项目启动的时候, 就会由```Commune\Chatbot\Framework\Bootstrap\LoadComponents``` 加载相关组件, 并进行初始化.



