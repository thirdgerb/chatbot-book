# 本地语料库 (Corpus)

在 [自然语言单元](/docs/core-concepts/nlu.md) 文档中, 已经介绍过 "语料库" 的概念.
这里再介绍一次.

CommuneChatbot 所需要的自然语言解析功能, 主要是 "语义解析" 和 "实体提取".
目前的主流技术对这两项功能, 都需要依赖高质量/不断迭代的语料库, 才能提高准确性.

语料库目前通常包含三个方面 :

- 意图语料
- 实体词典
- 同义词词典

CommuneChatbot 提供了一个本地语料库的方案, 用来保存这些语料.
然后通过 [NLUService](/docs/nlu/service.md) 同步到第三方服务.

这是因为目前定义语料库缺乏公共标准, 每个开源项目和 NL云服务 都可能有自己一套规范.
而录入语料库是一个非常辛苦的过程. 容易造成对第三方服务毫无必要的重依赖.

理想的情况下, 是有一个 Babel 的项目, 可以将所有框架和云服务的语料库快速转义, 同步.
在此之前, CommuneChatbot 暂时只好自己维护一个简单版的本地语料库.

关于本地语料库的配置, 将在 [管理 NLU 服务](/docs/nlu/manager.md) 一篇中专门介绍.
本文主要介绍语料库的含义和格式.

## 意图语料

现阶段的自然语言理解技术还非常初级, 本质上是先有了机器可以响应的一系列 "意图" 抽象,
它们相当于命令, 用于操作机器; 然后再把人类自然语言五花八门的表达, 映射到这些已定义的意图上. 这就是语义匹配技术.

早期的语义匹配技术, 通常使用 正则/敏感词/关键词 等明确的规则来做.
而现在的自然语言技术, 则可以通过 __已标注__ 的语料, 获得统计学层面的泛化识别能力.

举个例子, 我们提供一部分语料用来表示 "attitudes.affirm" (肯定) 这个意图:

```
- name : attitudes.affirm
  examples :
    - 是的
    - 没错
    - 的确如此
    - 是
    - 就是这样
    - 对的
    - 对
    - 嗯
    - 嗯呢
```

然后交给某个自然语言中间件. 下一次用户可能会说 "是呀", 这句话从未注册到语料库中,
但 NLU 也可能告诉我们它命中 "attitudes.affirm" 的可能性, 通常是某个小于 1 的数, 例如 ```0.9527...```; 用于表示 NLU 给出的置信度.

有一些意图和实体是结合在一起的. 例如 "北京明天怎么样", 它可能命中一个 "queryWeather" 的意图,
同时也提供意图相关的实体 "city:北京" 和 "date:明天".

为意图提供语料的同时, 也应该对实体进行标注, 有利于自然语言技术同时进行意图匹配和实体提取. 例如 :

```
// yaml 格式定义
- name : demo.cases.tellweather
  examples :
    - "[今天](date)天气怎么样"
    - "我想知道[明天](date)的天气如何"
    - "[北京](city)[后天](date)什么天气啊"
    - "气温如何"
    - "[明天](date)多少度"
    - "[后天](date)什么天气啊"
    - "[上海](city)[大后天](date)下雨吗"
    - "您知道[广州](city)的天气吗"
    - "会有暴风雨吗"
    - "请问[明天](date)下雨吗"
    - "[后天](date)多少度啊"
    - "[明天](date)是晴天吗"
    - "[长沙](date)下雨了吗"
    - "[明天](date)[北京](city)什么气温"
    - "[深圳](city)天气"
    - "[上海](city)天气"
    - "[洛阳](city)气温"
```

CommuneChatbot 的意图语料, 可以使用数组来表示, 并封装到 ```Commune\Chatbot\OOHost\NLU\Options\IntentCorpusOption``` 对象.

```php

$data = [
    // 意图的名称
    'name' => 'demo.cases.tellweather',

    // 简介
    'desc' => '查询天气',

    // 意图的语料, 格式参考了 Rasa 的实现
    // 用 [] 包含实体的内容, 用 () 标注实体的名称
    'examples' => [
        "[今天](date)天气怎么样",
        "我想知道[明天](date)的天气如何",
        "[北京](city)[后天](date)什么天气啊",
        "气温如何",
        "[明天](date)多少度",
        "[后天](date)什么天气啊",
        "[上海](city)[大后天](date)下雨吗",
        "您知道[广州](city)的天气吗",
        "会有暴风雨吗",
        "请问[明天](date)下雨吗",
        "[后天](date)多少度啊",
        "[明天](date)是晴天吗",
        "[长沙](date)下雨了吗",
    ],

    // 意图涉及的实体名称, 对应实体词典
    'entityNames' => [
        'date',
        'city',
    ],

    // 意图的关键字, 能够精确匹配到意图的
    'keywords' => [
    ],
]

// 得到 IntentCorpusOption 实例.
$intentCorpus = new Commune\Chatbot\OOHost\NLU\Options\IntentCorpusOption($data);

// 名字
assert('demo.cases.tellweather' === $intentCorpus->name);

// 简介
assert('查询天气' === $intentCorpus->desc);

// 遍历语料的例句
foreach ($intentCorpus->examples as $example) {

    // 可以得到 IntExample 对象
    assert($example instanceof Commune\Chatbot\OOHost\NLU\Corpus\IntExample);

    // 去掉了实体标注的文本
    $example->text;

    // 包含实体标注的文本
    $example->example;

    // 将已标注的 Entity 解析为 Commune\Chatbot\OOHost\NLU\Corpus\ExampleEntity 对象
    $example->entities;

}
```

更多细节请查看以下几个类的源码 :

- ```Commune\Chatbot\OOHost\NLU\Options\IntentCorpusOption```
- ```Commune\Chatbot\OOHost\NLU\Corpus\ExampleEntity```
- ```Commune\Chatbot\OOHost\NLU\Corpus\IntExample```

## 实体词典

在上文关于意图语料的介绍中, 我们提到了例句 ```"明天北京天气如何"```,
其中有一个 "city" 实体 "北京".

而 "city" 这种实体的合法值很多, 并不能单纯从语料中统计出来, 而需要一个专门的 "city" 词典. 可以包含数百个县市名称. 这是自然语言单元所必要的 __知识__.

类似的词典可以有很多, 比如 日期/车型/书名/人名/歌曲名 ... 等等.
这种词典我们可以记录成:

```
// yaml 格式定义
- name : city
  desc : 城市
  values :
    - 上海
    - 重庆
    - 郑州
    - 洛阳
    - 焦作
    - 商丘
    - 信阳
    - 周口
    - 鹤壁
    - 安阳
    - 濮阳
    - 驻马店
    - 南阳
    - 开封
    ...
```

高质量的词典, 对自然语言单元提取实体的质量非常重要.

CommuneChatbot 的词典语料, 可以使用数组来定义, 封装到```Commune\Chatbot\OOHost\NLU\Options\EntityDictOption``` 对象.

## 同义词词典

我们虽然有了 "实体词典", 但在对话场景中仍然不足. 人类的自然语言中各种词汇有大量的同义词. 以城市名为例, "上海" 在某些语境下也可能被称为 "魔都", 而 "重庆" 被称为 "雾都" 等.

这些同义词也一样是自然语言单元重要的 __知识__. 因此本地语料库也可以定义同义词词典, 同样通过数组封装到 ```Commune\Chatbot\OOHost\NLU\Options\SynonymOption``` 对象中.

## Corpus 库

CommuneChatbot 提供一个进程级单例 ```Commune\Chatbot\OOHost\NLU\Contracts\Corpus```, 通过它来持有所有的 意图语料/实体词典/同义词词典. 它会在系统启动的阶段完成初始化.

我们可以使用 Corpus 对象来注册 语料/词典, 也可以从中读取语料和词典并与第三方服务同步.
使用 PHP 实现的实体提取模块, 也是通过实体词典和同义词词典来查找文本中的实体.

### 获取实例

获取 Corpus 最好的方式是依赖注入.
ServiceProvider/中间件/Stage/构造方法 等实现了依赖注入的地方,
都可以通过定义 ```Commune\Chatbot\OOHost\NLU\Contracts\Corpus``` 参数来依赖注入.

### API

Corpus 的 API 请通过相关类的源代码查看 :

- ```Commune\Chatbot\OOHost\NLU\Contracts\Corpus```
- ```Commune\Chatbot\OOHost\NLU\Contracts\Manager```

通过它, 可以关联性地获取 意图语料 + 实体词典 + 同义词词典. 例如:

```php

/**
 * @var Commune\Chatbot\OOHost\NLU\Contracts\Corpus $corpus
 */

// 用意图名获取意图的语料
$intentCorpus = $corpus->intentCorpusManager()->get($intentName);

// intent corpus 通过 corpus 获取实体对应的实体词典
$entityDictions = $intentCorpus->getEntityDictOptions($corpus);

$entityDict = current($entityDictions);

// entity diction 通过 corpus 获取实体词典对应的同义词词典
$entityDict->getSynonymOptions($corpus);

```

### 存储语料

Corpus 默认的实现是```Commune\Chatbot\OOHost\NLU\Corpus\CorpusRepository```, 它持有一个实现了```Commune\Support\OptionRepo\Contracts\OptionRepository``` 的配置仓库,
允许从自定义的存储介质 (文件, mysql, 等等) 中读写```Commune\Support\Option```类型的对象.
IntentCorpusOption/EntityDictOption/SynonymOption 对象都继承自 ```Commune\Support\Option```.

关于 Option 和 OptionRepository, 请查看 [配置体系文档](/docs/engineer/configuration.md) 的相关介绍.

默认使用的存储介质是 Yaml 文件.
OptionRepository 会从指定的 yaml 文件中读取语料信息, 也将这些信息存储到该文件中.
使用它也可以实现懒加载, 系统启动时并不一定加载全部的语料, 只会在用到的时候才加载.

详情请查看 [管理 NLU 服务](/docs/nlu/manager.md).

### 注册语料

Corpus 通过 OptionRepository 加载的语料, 是语料库信息的唯一来源.
通过 Corpus 获得的```Commune\Chatbot\OOHost\NLU\Contracts\Manager```对象,
可以往语料库里注册新的语料.

```php
/**
 * @var Commune\Chatbot\OOHost\NLU\Contracts\Corpus $corpus
 */

// 意图语料库管理工具
$corpus->intentCorpusManager();

// 实体词典的管理工具
$corpus->entityDictManager();

// 同义词词典的管理工具
$corpus->synonymsManager();
```

往 Corpus 中注册语料, 有 ```Manager::register()``` 和 ```Manager::save()``` 两种方法.

```Manager::register``` 只会把语料注册到内存中,
只有执行 ```Manager::sync()``` 的时候才会存储到介质, 仍可决定是否覆盖原有配置.
而执行 ```Manager::save()``` 方法, 则会立刻把语料存储到介质.

这是因为语料库内容可能有多个来源, 例如每一个多轮对话组件可能都有自己预设的语料库;
多个语料库的数据可以在项目启动时动态加载到 Corpus. 而语料的存储只能有一个介质, 优先级也高于其它来源, 避免项目启动时频繁重写语料库数据.

> 注意, 存储介质中的语料信息, 优先级高于从其它组件加载的语料信息.
> 因此可以从其它组件加载语料作为初始化数据, 然后本地再进行修改.

### 语料库生成意图占位符

在语料库中定义过的意图, 即便没有定义专属的 ```Commune\Chatbot\OOHost\Context\Intent\IntentMessage``` 类 (见 [Intent文档](/docs/dm/intent.md)),
也会在启动时自动加载

### 在组件中使用

CommuneChatbot 项目允许把多轮对话的一部分功能封装成独立的组件,
从而允许在应用中按需挑选组件, 并加载.
关于 "组件" 的更多信息, 请查看[组件文档](/docs/components/index.md).

这些组件作为外部模块引入时, 最好能提供基础的语料库, 避免复制粘贴的重复劳动.
因此组件需要允许项目启动时将自己的语料库加载到 Corpus 中.

常见的做法是在 ```Commune\Chatbot\Framework\Component\ComponentOption``` 的 ```bootstrap``` 方法里, 调用```ComponentOption::registerCorpusOptionFromYaml()```方法,
通过 yaml 文件单独加载语料到 Corpus.

具体请查看```Commune\Chatbot\Framework\Component\Providers\RegisterCorpusOptionFromYaml```.


## PHP EntityExtractor

我们通常期待自然语言单元来完成实体提取的任务.
但在拥有了本地实体词典的情况下, 有时也可以用 PHP 的规则去抽取实体, 而不完全依赖自然语言单元.

抽取实体的基本原理就是关键词匹配. 系统定义了 ```Commune\Chatbot\OOHost\NLU\Contracts\EntityExtractor``` 服务,
会将实体词典和同义词词典的数据制作一棵以单字为节点的索引树;
然后逐字遍历用户的输入消息, 尝试在树中找到匹配的实体值.

可以通过依赖注入获取 EntityExtractor 对象. 而 Hearing API 中的 ```Hearing::matchEntity```方法, 如果 nlu 没有返回值时, 也会尝试调用 EntityExtractor.

这种做法显然解决不好断句歧义的问题, 例如 "我要去山东走走" 和 "我要去山东边走走", 两者的 "山东" 完全不是一个意思. 只是从 Corpus 模块中派生出来的一种能力.
请在非常了解利弊的前提下再使用.






