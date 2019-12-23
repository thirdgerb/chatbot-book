# 自然语言单元

自然语言处理技术有庞大的体系, 覆盖对话机器人的方方面面. 在[对话机器人技术架构](/docs/core-concepts/structure.md)这篇文章的最后一段, 附上了一些国内对话机器人的技术分享文章. 从中我们可以了解到各种自然语言技术是如何应用到对话机器人的.

像 CommuneChatbot 这种以 "平台接入" 和 "对话管理" 为主的开发框架, 通常把自然语言技术作为第三方的功能单元来看待. 会给这些功能设计统一的抽象; 再针对同一功能的不同的服务提供方, 开发不同适配器. 最后通过 API 调用将这些功能单元引入系统.

因此, 我们会更关心对话机器人通用的自然语言处理功能, 其中最重要的是:

- 语义识别
- 实体提取

如果您对这些技术还不太了解, 作者会在本文中分享一些个人的粗糙理解.

## 1. 通用指令抽象层

与软硬件机器人交互的传统方式, 可能通过 按钮/操纵杆/指令行/图形界面 等. 无论输入来自于哪种形式, 最终都会转化成软硬件接口所要求的通用指令.

通用指令也通常被抽象为 "指令名" 和若干个 "参数".

对话和其它输入形式最大的区别在于以下几点:

- 同一个命令, 自然语言有成百上千种不同的表达方式.
- 自然语言产生的指令, 往往不是一次提供了所有必要的参数, 而是通过多轮对话来补完的.
- 自然语言还可能一次产生多个指令.

也许有些机器学习的对话机器人, 是直接将自然语言作为输入来处理的. 例如常见的基于大量样本的闲聊机器人. 它主要关心用一个输出的序列去响应一个输入的序列.

但 CommuneChatbot 这样基于规则的对话机器人, 有一个通用指令的抽象层.
仍要把自然语言转化为 "意图名" + "实体" (等同 "指令名" + "参数") 的通用指令形式, 然后才交给逻辑单元处理.

所以从 "通用指令" 的角度来看, 无论是语音或文字, 和 按钮/操纵杆/指令行/图形界面 没有本质的区别.

当一个机器人完全基于 "通用指令" 的抽象控制层来运作时, 任何一种输入形式都没有本质的差别.
这时它就不再被 "对话机器人" 或 "摇杆机器人" 的形式所束缚, 而变成了多模态机器人.
 任何端的输入信号被统一为通用指令后, 作用于一个公共的状态管理中控 (大脑); 中控再把指令进行广播, 让每一个端作出自己的响应.

## 2. 语义识别

站在通用指令抽象层的立场上看语义识别, 简单来说, 就是识别出各种相似的表达方式其实是同一个 "指令名". 这在自然语言处理单元中往往称之为 "意图" (intent).

例如 "你好", "hello", "hi", "好啊" 等等, 都是 "打招呼" 的意图.
这些表达对于通用指令抽象层而言, 是同一个输入.
这样机器人就可以用相同的逻辑去响应它们.

自然语言处理用于识别语义的方案有许多种, 但对于 "通用指令抽象层" 而言, 具体用哪种技术实现我们并不关心, 只要保证准确率高就足够了.

但实际开发中, 我们还是需要了解现阶段语义理解技术的瓶颈. 目前技术比较擅长解决一些简单明确的语义; 各种复杂表达的语义理解还是有挑战性的. 例如 :

* "(以前)我喜欢一个人, (现在)我喜欢一个人" 这种有歧义的;
* 或是有大量代词, 上下文相关的;
* 又或是一段话包含多个指令的, 例如 "向前走, 到第一个红绿灯右拐, 往前走过三个街区, 看到百货大楼左拐, 再走一百米就到".

对于一句话只有一个意图的简单表达, 通常处理单元也会返回多种可能的意图, 为每一种意图给出一个置信度.

一般情况下, 置信度最高的意图, 最可能是用户的真实意图. 但在复杂多轮对话管理中, 往往因为上下文不同而有不同的权重. 例如 "问: 你叫什么名字? 答: 静静", 和 "问: 你想干什么? 答:静静", 相同的回答明显有不同的意图. 这也是我们需要知道的.

## 3. 实体提取

对于通用指令层的抽象层而言, 自然语言识别的 "意图" 相当于 "指令名", 而 "实体" （Entity）就相当于 "参数" 了.

例如 "北京明天的天气怎么样?" 这句话, 我们认为意图是 "查询天气", 而对应的参数自然是 "地点:北京" 与 "时间:明天". 这句话相当于一个 ```queryWeather(city = "北京", date="明天")``` 的调用.

更多情况下, 一句话不足以提取指令所要求的所有实体参数, 于是会通过多轮对话来填补. 例如:

```
用户: 天气如何?
机器人: 请问您想了解哪个城市的天气?  // 单轮对话提取 city 实体
用户: 北京
机器人: 请问您想了解哪天的天气? // 单轮对话提取 date 实体
用户: 今天
```

需要填补的参数通常被称为 "slot", 于是这种填补参数的多轮对话也被称之为 "slot filling" (填槽式多轮对话). 这是一种简洁而常见的一阶多轮对话.

由于对话常常出现传递信息失误, 因此填槽型多轮对话往往还有机器人要求确认的环节. 一个意图有 n 个参数, 如果全要确认, 那就有 n + 1 次确认.

除此之外还有一些情况是 CommuneChatbot 项目比较在意的 :

- 实体多轮对话
- 半开放域问题

### 实体多轮对话

简单的任务型多轮对话, 往往是 "意图" (intent) 与 "实体" (Entity) 全匹配完之后, 唯一执行一次逻辑. 但自然语言对话的场景, 更多是复杂多轮对话, 对流程的灵活性有更高的要求.

每一个实体的获取, 并不见得一轮对话就可以完成. 我们举一个 "四十五问题" :

当用户回答一个数字, 说 "十五" 时, 语音识别因为各地口音不同, 很容易理解成 "四十五"; 同理, "四十五" 也容易被识别为 "十五". 机器人若不能主动发起澄清, 很可能把错误的结果带到后面的逻辑.

也就是说, 仅仅是提取一个实体, 也可能需要一个小型多轮对话. 这也是为什么自然语言的复杂多轮对话呈现分形式结构的原因.


### 封闭域问题

许多任务型对话机器人, 在匹配意图的环节是完全开放域的, 对所有的意图都放开.然后到 "slot filling" (填槽) 的环节, 又是完全封闭域的, 除了填槽之外不响应任何意图.

而自然场景中的多轮对话, 并非这种线性流程. 用户在 "slot filling" (填槽) 过程中, 可能有各种打破流程的需要:

1. 退出当前任务
1. 上一个回答说错了, 想返回修改
1. 抢答了后面的实体
1. 想请求帮助
1. 突然插入一个别的意图

上述的这些问题, 完全封闭域的填槽是无法响应的. 显然, 对多轮对话的管理需要有比单向 slot filling 更灵活的手段.

## 4. 提供自然语言单元

如上文所述, CommuneChatbot 这一类框架用 "通用指令" 的抽象来看待自然语言单元, 将之解耦为第三方服务, 用到最核心的功能是 "语义识别" 与 "实体提取".

这两类技术在当前都发展到了可用阶段. 对于没有实力自己开发 NLU 的团队, 可以选择云端服务, 例如百度 UNIT 或 DialogFlow 等; 也可以自己搭建开源项目. 例如 Rasa, Spacy 等.

自然语言领域的机器学习技术, 可以提高 "语义识别" 或 "实体抽取" 的泛化能力. 然而传统的语义近似度计算, 关键词和敏感词提取的技术, 在一些场景 (问答, 实体回答) 也能有比较好的效果.

无论哪一种技术实现, 目前都比较依赖高质量 "语料库", 因此我们需要对语料库有基本的了解. 常见的语料库包含三个方面:

* 标注过的对话样本
* 实体词典
* 同义词辞典

__标注过的对话样本__ : 将我们积累的对话语料进行明确标注, 然后喂给自然语言单元.
常见的标注需要指明 "意图", 并标记 "实体" 的位置.
例如 "北京明天天气如何?", 可标注意图为 "queryWeather", 实体位置是 ```[北京](city)[明天](date)天气如何```.

__实体词典__ : 将实体可能的值列为词典, 结合对话样本, 能更好地被提取.
例如全国城市名词典, 汽车品牌和型号名词典等.
虽然关键字匹配算法也能从对话中查找词语, 但难以处理歧义, 例如 "我想去山东" 和 "我想去山东边看看".

__同义词词典__ : 自然语言对话中常常出现语境相关的同义词, 例如 "北京" 和 "首都".
对实体词典的值提供同义词词典, 能更好地提升响应能力.

现在语料库的建设尚没有统一标准, 虽然内容要求大同小异, 但各种自然语言单元都有自己的一套标注规则.

由于 CommuneChatbot 将各种自然语言单元都统一视作第三方服务, 所以也需要有自己的本地语料库, 能够把本地语料通过适配器映射成所用 NLU 的语料来同步, 保证多方的兼容性.


## 5. 对话管理看待自然语言单元的态度

CommuneChatbot 项目核心功能在于多轮对话管理. 站在对话管理的角度, 看自然语言单元 (NLU), 或有三种态度可供参考 :

- 尽可能使用 NLU
- 不局限于现有 NLU
- 不依赖 NLU

__尽可能使用 NLU__ : 虽说 "通用指令抽象" 的观点, 更重视 "意图识别" 与 "实体提取", 但现阶段成熟的自然语言处理技术远不止这些.
"词性标注", "句法分析", "情绪分类" 等技术都能在特定的场景有利于对话的推进.
对话管理的立场是应该尽可能地使用已有的技术, 将这些技术在具体场景中落地.

__不局限于现有 NLU__ : 客观看待, 现在的自然语言技术还有许多不成熟的地方.
对话管理也不应该高度耦合现有技术, 失去了想象力.
例如, "语音对话" 现阶段往往先把语音转化为文字, 然后对文字进行意图识别.
为何不能根据上下文, 直接把语音特征用于意图匹配呢?
对话管理比其它自然语言技术更贴近具体应用场景, 应该对其它技术积极提出需求.

__不依赖 NLU__ : 站在技术角度, 全能的自然语言处理似乎是目标; 但站在应用角度, 对话管理有自己的任务, 不应该受其它技术发展程度局限.
就像早年通过数字来交互的电话机器人一样, 多轮对话即便没有自然语言识别, 也是可以运行的.
类似 "请问是否要加冰?", 回答 "加" 表示肯定这种情况, 目前 NLU 还没有针对性的处理, 对话管理就应该积极用其它技术手段弥补, 而不能依赖 NLU 坐等其成.
