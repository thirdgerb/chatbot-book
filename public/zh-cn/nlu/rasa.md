# 使用 Rasa 搭建 NLU

开源项目 [Rasa](https://rasa.com/) 可以提供基本的 NLU 服务.

它本身是一个管道化的工程, 通过一个个独立的管道, 整合 语义匹配/实体提取/读取语料/训练神经网络 等一系列功能 (通常使用别的开源方案, 也可以自己定义中间件), 并对外提供服务.

Rasa 既可以提供基于机器学习的多轮对话机器人, 也可以独立提供基于 http 的自然语言服务.

CommuneChatbot 使用 Rasa 作为自行搭建 NLU 的样例, 以供新手作为参考.

在项目中引入 package :

```
composer require commune/compnt-rasa
```

## 安装 Rasa

安装 Rasa 可以先查阅 [官方文档](https://rasa.com/zh-cn/rasa/user-guide/installation/), 网上也有许多安装的教程. 这里就不赘述了.

需要注意的是 :

- 使用 Python3 和 pip3 去安装和运行 Rasa, 小心 Python 3.x 与 2.x 导致的兼容问题 (尤其是 yum 和 homebrew)
- 需要安装许多额外的 Python 项目, 例如 [spaCy](https://spacy.io/) 等
- 中文可能要用到 Jieba 分词等项目, 视情况使用.

## 配置 Rasa

使用 Rasa 最重要的就是配置它的管道, 通过管道定义具体的解决方案.

网上有很多种解决方案的分享文章, 值得参考. CommuneChatbot 的 [工作站](https://github.com/thirdgerb/studio-hyperf) 提供了一个示例版本,
代码在 ```BASE_PATH/rasa-demo``` 中. 提供的默认配置如下:

```
# Configuration for Rasa NLU.
# https://rasa.com/zh-cn/rasa/nlu/components/

# 表示使用中文
language: "zh"

- name: "JiebaTokenizer"
- name: "RegexFeaturizer"
- name: "CRFEntityExtractor"
- name: "EntitySynonymMapper"
- name: "CountVectorsFeaturizer"
- name: "CountVectorsFeaturizer"
  analyzer: "char_wb"
  min_ngram: 1
  max_ngram: 4
- name: "EmbeddingIntentClassifier"
```

这仅仅是一个示范的配置, 勉强可以使用.
要使用 Rasa 解决好 __具体业务场景__ 下 __中文__ 的解析效果,
还需要做很多的工作.
建议有相关经验的话, 自己根据应用场景来开发自己的管道, Policy 等.
推荐参考 [rasa_chatbot_cn](https://github.com/GaoQ1/rasa_chatbot_cn).


## 注册 RasaComponent

CommuneChatbot 对 Rasa 服务做了基本的封装, 需要通过 ```Commune\Components\Rasa\RasaComponent``` 引入系统, 具体而言有三步:

1. 配置数组引入 ```Commune\Components\Rasa\RasaComponent```
1. ```NLUComponent::$nluSerivces``` 处, 注册```Commune\Components\Rasa\Services\RasaService```
1. 在配置数组 ```$chatbotConfig->host->sessionPipes``` 中引入中间件```Commune\Components\Rasa\RasaSessionPipe```

```php

// 机器人配置数组 ChatbotConfig

return [
    ...

    'components' => [
        ...

        Commune\Components\Rasa\RasaComponent::class => [
            // rasa http api 地址
            'server' => 'localhost:5005',

            // 如果通过 jwt 验证的话
            'jwt' => '',

            // 意图的置信度, 高于 0.7 才加入 highlyPossibleIntent
            'threshold' => 70,

            // 语料库同步的目标文件位置
            'output' => __DIR__ .'/resources/rasa.md',

            // rasa domain.md 的文件位置
            'domainOutput' => __DIR__ . '/resources/domain.md',

            // 使用 Guzzle 调用 Rasa api 时的配置
            'clientConfig' => [
                'timeout' => 0.3,
            ],
        ],

        NLUComponent::class => [
            'nluServices' => [
                ...
                Commune\Components\Rasa\Services\RasaService::class,
            ]

        ],
    ],

    ...

    'host' => [
        ...

        'sessionPipes' => [
            ...
            Commune\Components\Rasa\RasaSessionPipe::class,
        ]
        ...
    ],

]
```

## 同步语料库

Rasa 所需要的语料库, 格式在 [Rasa training data format](https://rasa.com/zh-cn/rasa/nlu/training-data-format/) 可以查看.
CommuneChatbot 的本地语料库设计参考了 Rasa 定义的格式.

使用 ```Commune\Components\Rasa\Services\RasaService``` 可以直接将本地语料库同步到 rasa 的配置文件中.

在 [本地语料库](/zh-cn/nlu/corpus.md) 和 [管理 NLU](/zh-cn/nlu/manager.md) 中我们介绍了语料库和同步的方法.

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

> 2

Commune\Components\Rasa\Services\RasaService : success
```

您也可以在 [搭建机器人 API](/zh-cn/setup/api.md) 一文中了解如何提供 http 接口来实现相同的功能.

## 训练和启动 Rasa

Rasa 虽然提供了基于机器学习的多轮对话功能, 但我们只需要用到其中的 NLU 功能.

可以参考官方文档 [using nlu only](https://rasa.com/zh-cn/rasa/nlu/using-nlu-only/).

简单来说, 在 Rasa 项目路径下 (例如工作站的 ```BASE_PATH/rasa-demo/```), 运行训练命令:

```
rasa train nlu
```

会得到一个训练完的 model 文件. 然后再启动 Rasa 服务:

```
rasa run --enable-api -m models/nlu-20190515-144445.tar.gz
```

在服务端部署时要考虑持久化与重启, 推荐使用 ```supervisor``` 之类的工具.

## Rasa 多轮对话和 CommuneChatbot 的区别

RasaX 自带的 Rasa Core 使用机器学习来实现多轮对话功能. 有兴趣可以查阅相关文档了解它的实现原理.

CommuneChatbot 是工程化的开发框架, 目标是跨平台的对话交互中控; 而 RasaX 定位是基于文字的对话机器人. 定位有较大的区别.

我们抛开其它部分, 主要讨论多轮对话部分, 和 RasaX 的最主要区别在于:

1. CommuneChatbot 的多轮对话基于规则来定义, 而非标注语料.
1. 定义了 [Message 体系](/zh-cn/engineer/messages.md), 消息不局限于文字
1. CommuneChatbot 的重点是实现复杂多轮对话.

在具体应用场景中使用哪一种方案更合适, 还请开发者自己判断.
