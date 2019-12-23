# 简单闲聊组件

"闲聊" 是对话机器人最常见的功能.
优秀的闲聊机器人有技术要求, 尤其是需要庞大的对话语料来支撑
(光整理语料库成本挺高, 试想整理电影里的对白, 做成一个学会用电影对白闲聊的机器人).

而闲聊又是多轮对话中必要的一环, 通常对话的响应逻辑有三个环节 :

- 匹配指定功能
- 尝试匹配闲聊
- 拒答

闲聊环节响应失败了才应该走到拒答. 因此 CommuneChatbot 仍然提供了一个简单的闲聊组件,
用于展示闲聊环节如何作为组件引入.

## 引入闲聊组件

简单闲聊组件的类是 ```Commune\Components\SimpleChat\SimpleChatComponent```.
它的原理很简单, 手写 "意图" 和 "回答" 的关联关系;
当响应逻辑走到闲聊环节, 如果闲聊意图命中, 则从回答中随机拿出一个;
否则拒答.

在机器人配置中引入组件, 并将```Commune\Components\SimpleChat\Callables\SimpleChatAction``` 作为 Hearing API 的默认响应逻辑 :

```php

return [
    ...

    'components' => [
        ...

        Commune\Components\SimpleChat\SimpleChatComponent::class => [
            'rootStorage' => [
                'meta' => YamlStorageMeta::class,
                'config' => [

                    // 闲聊的配置文件所在路径
                    'path' => __DIR__ . '/resources/example.yml',

                    // 表示所有的闲聊配置在同一个文件中.
                    'isDir' => false,
                ],
            ],

        ]
    ],

    ...

    // OOHostConfig 配置
    'host' => [
        ...

        // 通过它定义 Hearing API 的 defaultFallback
        // 允许使用类名来引入一个类, 这个类的实例必须是 callable 对象.
        'hearingFallback' => Commune\Components\SimpleChat\Callables\SimpleChatAction::class
    ],
];

```

## 定义闲聊的内容

闲聊的内容默认存储在 yaml 文件中, 每一个对象都是一个```Commune\Components\SimpleChat\Options\ChatOption```. 一个文件存储所有的闲聊内容.

```
# yaml 文件是一个数组, 每一个对象是一个 ChatOption

# intent 表示意图名
- intent : introduce.whoareyou

  # 用 examples 定义意图的语料
  # 如果 Corpus 还没有该意图的语料库,
  # 会自动加载到 Corpus 中
  # 注意, 意图名千万不要犯大小写错误!!
  examples:
    - 你是谁
    - 你是什么
    - 你是啥玩意
    - 这是什么东西
    - 我在哪里
    - 介绍一下你
  # 定义意图的回复. 是一个数组
  # 每一项是一个回复, 系统单纯随机选择一个
  # 回复的内容可以是 原文/模板/待渲染的 replyId/
  replies:
    - 这里是 %self.name% 全名是%self.fullname%. 是由%self.author% 开发的 %self.desc%
- intent : ask.joke
  examples:
    - 说个笑话吧
    - 会说笑话吗
    - 逗我开心
    - 会开玩笑吗
    - 有什么好笑的
    - 讲笑话
    - 来个笑话听听
  replies:
    - 有个武林高手去藏金阁盗宝 结果他吓死了 因为有密集恐惧症.
    - 我有个邻居，名字叫朱川，他妈妈每次给他买衣服，都会跟人说这是买给我们家朱川的……
    - 病人：“医生，你把剪刀留在我肚子里了。”“没关系，我还有一把。”
```

值得注意的是, 该组件自动加载闲聊意图和语料到 [本地语料库](/docs/nlu/corpus.md).
即便没有定义过该意图类, 也会自动生成 PlaceholderIntent, 可以自由用在多轮对话中.
