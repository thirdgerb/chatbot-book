# 定义组件

CommuneChatbot 的组件兼具配置与启动流程两方面的功能. 通过 ```Commune\Chatbot\Framework\Component\ComponentOption``` 类实现.

ComponentOption 首先是一个 ```Commune\Support\Option``` 对象,
它将 PHP 常用来做配置的数组封装到一个对象中, 从而可以像一个结构体一样,
强类型地调用配置的值.
IDE 的支持也会更好, 不像数组键名那么容易出错, 还可以统一修改.
更多内容请看 [配置体系文档](/docs/engineer/configuration.md).

而 ```ComponentOption::bootstrap()``` 方法会在项目启动的时候,
由 ```Commune\Chatbot\Framework\Bootstrap\LoadComponents``` 执行.
从而可以进行服务注册, 配置注册等一系列初始化工作.

## 创建 ComponentOption 类

以 StoryComponent 为例, 定义如下 :

```php
/**
 * !!! 必须要有注解, 方便 IDE 识别, 还可以统一修改.
 *
 * @property-read string $translationPath 脚本内容文件所在目录.
 * @property-read string $intentsPath 预定义的意图语料
 * @property-read MetaHolder $rootStorage 配置文件根仓库的元配置.
 * @property-read MetaHolder[] $storagePipeline 配置文件缓存仓库的元配置.
 *
 */
class StoryComponent extends ComponentOption
{

    // 定义组件配置的关联 Option
    protected static $associations = [
        'rootStorage' => MetaHolder::class,
        'storagePipeline[]' => MetaHolder::class,
    ];

    // 定义组件配置的默认数组
    public static function stub(): array
    {
        return [
            'translationPath' => __DIR__ . '/resources/langs',
            'intentsPath' => __DIR__ .'/resources/nlu/intents.yml',
            'rootStorage' => [
                'meta' => YamlStorageMeta::class,
                'config' => [
                    'path' => __DIR__ . '/resources/stories/',
                    'isDir' => true,
                ],
            ],
            'storagePipeline' => [
            ],
        ];
    }


    // 定义启动流程的逻辑
    protected function doBootstrap(): void
    {
        // 默认加载脚本几个类.
        $this->loadSelfRegisterByPsr4(
            "Commune\\Components\\Story\\Intents\\",
            __DIR__ . '/Intents/'
        );


        // 注册所有的脚本配置.
        $data = $this->toArray();
        $this->loadOptionRepoCategoryMeta(
            new CategoryMeta([
                'name' => ScriptOption::class,
                'optionClazz' => ScriptOption::class,
                'rootStorage' => $data['rootStorage'] ?? [],
                'storagePipeline' => $data['storagePipeline'] ?? [],
            ])
        );

        // 注册文本文件.
        $this->loadTranslationResource($this->translationPath);


        // 注册意图语料
        $this->registerCorpusOptionFromYaml(
            $this->intentsPath,
            IntentCorpusOption::class
        );

        // 最后注册 Story 容器.
        $this->app->registerProcessService(
            new StoryServiceProvider(
                $this->app->getProcessContainer(),
                $this
            )
        );
    }

}
```

## 注册组件

注册组件的位置是机器人配置的 ```$chatbotConfig->components```, 见```Commune\Chatbot\Config\ChatbotConfig```,
相关介绍在[配置体系](/docs/engineer/configuration.md).

```php

// 机器人配置数组 chatbotConfig
return  [

    ...

    'components' => [
        // 只写类名, 表示用 ComponentOption::stub() 方法提供的 默认值
        Commune\Components\SimpleWiki\SimpleWikiComponent::class,

        // 用类名做 key, 数组做值, 表示用数组的值覆盖默认数据.
        // 不需要所有的键值, 只需要修改的 .
        // 因为最终会执行  $option = $stub + $data 来合并数组.
        StoryComponent::class => [
            'translationPath' => BASE_PATH . '/resources/langs',
            'intentsPath' => BASE_PATH .'/resources/nlu/intents.yml',
        ]
    ]

    ...
];

```

## 定义 bootstrap

在 ```ComponentOption::doBootstrap()``` 方法内定义的流程, 会在系统启动的时候加载.

推荐仍然用 Service Provider 的方式注册服务, 把启动流程定义在 ```ServiceProvider::boot()``` 方法内.
这是因为只有完成所有服务的 register 流程, 才能够进行依赖注入.

```php
class MyComponent extends ComponentOption
{
    ...

    protected function doBootStrap()
    {
        // 与系统配置相关的服务
        // 必须第一时间加载, 优先级最高
        $this->app->registerConfigService(
            new SomeConfigServiceProvider(...),
        );

        // 注册进程级的服务
        $this->app->registerProcessService(
            new SomeProcessServiceProvider(...),
        );

        // 注册会话级的服务
        $this->app->registerConversationService(
            new SomeConversationServiceProvider(...),
        );
    }
}
```

ComponentOption 提供了更多注册服务或配置的方法, 具体请查看 ```Commune\Chatbot\Framework\Component\ComponentOption``` 类的注释.
简单来说, 包括:

- 加载 PSR4 规范的目录下的所有 Context 类
- 加载组件自带的语言包
- 加载组件自带的语料库
- 加载上下文记忆的名称 (允许不用类, 而用 contextName 来定义, 从而起到类似接口的作用)
- 注册消息渲染器 (ReplyTemplate, 见[渲染回复](/docs/engineer/replies.md))
- 加载 OptionRepository 的配置元数据
- 注册事件监听者


除此之外, 还有 ```ComponentOption::dependComponent()``` 方法, 可以定义当前组件依赖的其它组件.
如果机器人配置数组中没有加载这些组件, 则仍然会根据这里的依赖关系将之加载进系统.

## 获取 ComponentOption 实例

所有的组件配置 ComponentOption 对象, 都会作为进程级单例绑定到进程级容器上.
可以通过 IoC 容器直接获取.
在任何可以依赖注入的地方, 都可以定义它从而实现依赖注入.
以 RasaComponent 为例 :

```php
class RasaService implements NLUService
{
    /**
     * @var ClientFactory
     */
    protected $clientFactory;

    /**
     * @var RasaComponent
     */
    protected $config;

    /**
     * 将 RasaComponent 作为参数依赖注入
     *
     * @param ClientFactory $clientFactory
     * @param RasaComponent $config
     */
    public function __construct(ClientFactory $clientFactory, RasaComponent $config)
    {
        $this->clientFactory = $clientFactory;
        $this->config = $config;
    }

    ...

}

```
