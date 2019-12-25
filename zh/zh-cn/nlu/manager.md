# 管理 NLU 服务

在前几篇文档:

- [封装 NLUService](/zh-cn/nlu/service.md)
- [注册 NLU 中间件](/zh-cn/nlu/middleware.md)
- [本地语料库](/zh-cn/nlu/corpus.md)

我们介绍了与 NLU 相关的功能与实现. CommuneChatbot 也对它们提供了统一的管理方案.

## NLUComponent

CommuneChatbot 启动的时候, 默认会加载组件 ```Commune\Chatbot\OOHost\NLU\NLUComponent```.
该组件主要用来管理 NLUService, NLULogger 和 Corpus.

### 自定义配置

关于组件, 更详细的介绍请查看 [组件化文档](/zh-cn/components/index.md).
组件都通过一个 ```Commune\Chatbot\Framework\Component\ComponentOption```
对象加载到系统中.

可以通过修改机器人配置的```$chatbotConfig->components``` 参数, 来自定义该配置.
详情可查看 ```Commune\Chatbot\Config\ChatbotConfig```类, 和[配置体系文档](/zh-cn/engineer/configuration.md).

```php

// 机器人的基础配置
return [
    ...

    'components' => [

        // 自定义 NLUComponent 的配置. 不定义, 则会加载默认配置.
        Commune\Chatbot\OOHost\NLU\NLUComponent::class => [

            // 系统可用的 NLUService 封装
            'nluServices' => [
                RasaService::class,
            ],

            // corpus service provider
            'corpusService' => CorpusServiceProvider::class,

            // NLULogger 的实现
            'nluLogger' => NLULoggerServiceProvider::class,

            // 意图语料库存储介质, 根介质
            'intentRootStorage' => [
                // 表示使用 yaml 文件来存储
                'meta' => YamlStorageMeta::class,
                // 配置细节
                'config' => [
                    // 文件所在路径
                    'path' => __DIR__ . '/resources/nlu/intents/',
                    // 表示是文件夹, 每一个 yaml 文件是一个 intent
                    'isDir' => true,
                ],
            ],

            // 实体词典的根存储介质
            'entityRootStorage' => [
                'meta' => YamlStorageMeta::class,
                'config' => [
                    'path' => __DIR__ . '/resources/nlu/entities/',
                    'isDir' => true,
                ],
            ],

            // 同义词的根存储介质
            'synonymRootStorage' => [
                'meta' => YamlStorageMeta::class,
                'config' => [
                    'path' => __DIR__ . '/resources/nlu/synonyms.yml',
                    // 表示是单个文件, 包含了所有的同义词词典.
                    'isDir' => false,
                ],
            ],

            // 意图语料的存储管道
            'intentStoragePipeline' => [
            ],

            'entityStoragePipeline' => [
            ],

            'synonymStoragePipeline' => [
            ],
        ]
    ]

    ...
];
```

数组中每一项配置都是可选的, 如果没有填写, 则使用 ```NLUComponent::stub()``` 提供的默认值.

### NLUComponent 做了什么

和其它的[组件](/zh-cn/components/index.md)一样,
NLUComponent 最重要的是 ```NLUComponent::doBootstrap()``` 方法,
里面可以看到该组件在系统启动时做的事情.

包括注册 Service Provider, 定义语料库等.
修改 NLUComponent 的配置, 就能改变默认的服务内容.

### 注册 NLUService

配置的 ```NLUComponent::$nluServices``` 属性应当定义所有可用的 ```Commune\Chatbot\OOHost\NLU\Contracts\NLUService``` 对象.

这样就可以在系统运行逻辑中感知到自己拥有哪些 NLUService, 可以遍历操作.
例如, 同步语料库到所有已注册的 NLUService.

### Corpus 数据源

在 [本地语料库文档](/zh-cn/nlu/corpus.md) 中介绍了,
默认的本地语料库 ```Commune\Chatbot\OOHost\NLU\Contracts\Corpus```
通过配置仓库 ```Commune\Support\OptionRepo\Contracts\OptionRepository``` 持有语料数据.

而在 NLUComponent 中, 可以定义语料的存储介质. 默认是使用 yaml 文件.
这是因为文件存储可以加入 git 进行版本控制, 当语料库数据量不大时,
方便在项目发布时就带上相关的语料用例做示范.

语料库很大的情况下, 用 yaml 可能不一定合适. 可以按需定义自己的存储介质, 具体请参考 [配置体系文档](/zh-cn/engineer/configuration.md).

## 在多轮对话中管理 NLU

像 CommuneChatbot 这类工程化的多轮对话机器人框架,
最大的优点之一是可以不依赖 NLU 和语料, 快速开发出多轮对话功能.

这也使得 __用多轮对话管理多轮对话机器人__ 变得可能.
可以在对话逻辑中加入各种超级管理员功能, 用于操作系统自身.

NLUComponent 就自带了几个对话管理模块:

- 语料库管理 : ```Commune\Chatbot\OOHost\NLU\Contexts\CorpusManagerTask```
- 管理意图语料 : ```Commune\Chatbot\OOHost\NLU\Contexts\IntCorpusEditor```
- 编辑意图语料 : ```Commune\Chatbot\OOHost\NLU\Contexts\IntExampleEditor```
- 测试 NLU : ```Commune\Chatbot\OOHost\NLU\Contexts\NLUMatcherTask```

系统还自带了一个开发工具的多轮对话入口 :```Commune\Components\Demo\Contexts\DevTools```. 用起来的效果是 :

```
CommuneChatbot 开发了一批用多轮对话管理机器人自己的工具.

请选择您要使用的开发工具:  (1)

[1] nlu 语料库管理

> 1

选择功能 (1)

[1] 查看NLU命中的意图
[2] 同步语料库到NLU
[3] 同步语料库到本地存储
[4] 管理意图的语料样本

> 1

进入NLU意图管理.
请输入任意语句, 会给出命中的意图.
输入'b'退出工具

> 你好

当前匹配结果如下:

{
    "matchedIntentName": null,
    "possibleIntents": [],
    "entities": null,
    "intentEntities": [],
    "replies": null,
    "emotions": null,
    "sorted": true,
    "focusIntents": null,
    "words": null,
    "handled": null
}

进入NLU意图管理.
请输入任意语句, 会给出命中的意图.
输入'b'退出工具

> b

完成测试
```

如示例所示, 可以在该多轮对话中选择 ```同步语料库到本地存储```, 将语料库数据存储到介质中.
或是 ```同步语料库到NLU```, 自动将语料库同步到已注册的 NLUService.
