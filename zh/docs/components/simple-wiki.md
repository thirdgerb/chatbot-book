# 简单问答组件

问答是对话机器人很常见的功能, 比如智能客服.

高级的问答机器人, 可以用多轮对话来响应用户的问题; 进一步的, 能从一个庞大的结构化知识库中, 自己组装出用户想要的答案; 更进一步的, 能从庞大的普通文字档案中, 自主生成一个结构化知识库.

CommuneChatbot 为测试功能和搭建 Demo, 提供了一个简单的问答组件.
它的原理很简单, 用意图来搭建问题体系, 用文件来记录每一个意图的回答内容.

这个解决方案是目录化的, 一个 yaml 文件对应一个答案. 所以也可以视作一种用问题来查询的简单的 Wiki.

许多要求不高, 数量级有限的 FAQ (常见问题回答), 用这个功能就足以实现.

## 对话形式

定义问答的 yaml 文件会自动映射为 IntentMessage 类, 具体是 ```Commune\Components\SimpleWiki\Libraries\SimpleWikiInt```.

意图名称是文件相对路径的映射, 以```sw.```为前缀. 例如 ```PATH/demo/intro/chatbot.yml``` 文件, 会自动映射成 ```sw.demo.intro.chatbot``` 意图.

在上下文逻辑中只需要调用 ```$hearing->runIntentIn('sw')``` 或 ```$hearing->runIntentIn('sw.demo')```, 就会根据该 yaml 文件自动生成回答.

回答的内容包括两个部分: 正文 + 您可能需要. 只不过这个 "您可能需要" 是假装的,
其实是手动指定,
或者使用目录的相对关系自动生成.

这样生成对话可能是 :

```
CommuneChatbot 项目致力于 "对话管理" 的领域, 希望在解决 "复杂多轮对话" 难题的前提下, 能快速开发出各种基于对话交互的应用, 探索对话交互的各种可能性.

您可能需要: (1)

[1] 使用了哪些轮子
[2] 如何使用本项目
[3] 项目有什么特点
[4] 什么是复杂多轮对话
[5] 什么是复杂多轮对话
[6] 重复内容
[7] 返回上一层
[8] 退出

> 2
本项目目前还在开发 beta 版中, 文档还没有完善. 官方网站是 https://communechatbot.com/

github :

- chatbot : 核心开发框架, 地址 https://github.com/thirdgerb/chatbot
- studio-hyperf :  基于 hyperf 开发的工作站, 地址 https://github.com/thirdgerb/studio-hyperf

您可能需要: (1)

[1] 使用了哪些轮子
[2] 什么是对话机器人
[3] 项目有什么特点
[4] 什么是复杂多轮对话
[5] 重复内容
[6] 返回上一层
[7] 退出

> 1
commune/chatbot 项目基于 php 进行开发, 使用的主要第三方组件如下:

- php >= 7.2
- swoole >= 4.3 : 用于搭建高性能的服务
- hyperf >= 1.0 : swoole 的开发框架, 较为规范, 尤其提供了多种协程客户端
- rasa : 作为系统默认的 NLU 中间
- symfony : 提供大量优秀的组件库.
- illuminate : 提供大量优秀的组件库
- easywechat : 提供微信端和拼音包.
- webpack + vue + vuetify : 用于搭建默认的web端页面

更详细的引用情况请看 composer.json 与 package.json

您可能需要: (1)

[1] 重复内容
[2] 返回上一层
[3] 退出

```

## 引入组件

简单问答组件的类是 ```Commune\Components\SimpleWiki\SimpleWikiComponent```.
它能指定一个文件夹, 文件夹下所有的 yaml 文件会根据相对路径, 自动生成 IntentMessage 类.

在机器人配置中引入这个组件 :

```php

// ChatbotConfig 数组
return [
    ...

    'components' => [
        Commune\Components\SimpleWiki\SimpleWikiComponent::class => [

            // 所有 yaml 文件所处的根目录
            'rootStorage' => [
                'meta' => YamlPathStorageMeta::class,
                'config' => [
                    'path' => __DIR__ .'/resources/wiki',
                    'depth' => '>= 1', // 第一层目录会作为 group 的分组ID.
                    'isDir' => true,
                ]
            ],

            // 在这里定义多个答案库的分组
            // 每一个分组是一个目录, 例如 demo 这个分组对应 __DIR__ ./resources/wiki/demo'
            // 每个分组可以定义一些独立功能
            'groups' => [

                // 系统自带的 demo
                [
                    'id' => 'demo',
                    
                    // 在这里可以给意图定义简写方式
                    // 只适用用分组内部. 
                    'intentAlias' => [
                        // alias => intentName
                    ],
                    
                    // 每个回答之后的 "您可能需要" 通用的菜单功能. 
                    'defaultSuggestions' => [
                        // default suggestions
                        '重复内容' => Redirector::goRestart(),
                        '返回上一层' => Redirector::goFulfill(),
                        '退出' => Redirector::goCancel(),
                    ],

                    // "您可能需要" 的问题, 
                    'question' => 'ask.needs',

                    // 当回答拆成多步时, 告知用户输入空消息继续的提问
                    'askContinue' => 'ask.continue',

                    // 所有回复使用的 ReplyId 前缀
                    'messagePrefix' => 'demo.simpleWiki',
                ],

            ],

            // 所有 Reply 所在的语言包路径
            'langPath' => __DIR__ .'/resources/trans',

        ]
    ],

    ...
];

```

## 定义 Wiki 内容

每个 yaml 文件定义一个回答内容, 该回答内容的结构符合 ```Commune\Components\SimpleWiki\Options\WikiOption```, 查看该类可以看到完整的注释.

这个配置有四个方面内容:

- 问题对应的意图名称
- 问题的语料库
- 回答的内容
- "您可能需要" 的选项

```
# 意图的简介
description: CommuneChatbot 项目介绍

# 意图的语料
examples:
  - 这是一个什么项目
  - 什么是 CommuneChatbot
  - 介绍一下这个项目
  - 说说看这个项目
  - 这是什么项目
  - 说说这个项目

# 回复, 每一行是一个 replyId, 省略了前缀 demo.simpleWiki
# 该前缀定义在 GroupOption 中了.
# 内容定义在语言包内.
# 如果 reply 被拆成多条, 机器人不会一次说完, 而是提示用户输入信息继续.
replies:
  - intro1
  - intro2
  - intro3
  - intro4
  - intro5
# 定义 "您可能需要"
# 会自动根据相对路径生成
suggestions:

  # 以 . 开头, 表示是相同目录之下的 ./intro/chatbot.yml 文件
  - .intro.chatbot

  # 如果是 ./ , 表示同目录下所有其它的文件
  - ./

  # 表示上级目录下的 ../app/name.yml 文件
  - ..app/name

  # 表示上两级目录下的 ../../app/name.yml 文件
  - ...app/name

  # 表示相同 group 下的意图名称, 省略了 sw.groupId 的前缀
  - /intentName
```

