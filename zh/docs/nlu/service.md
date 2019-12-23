# 封装 NLUService

第三方的自然语言单元可以用任何方式封装, 只要在 Session Pipe 环节加载,
完成对输入消息的解析并赋予 ```Commune\Chatbot\Blueprint\Conversation\NLU```,
就可以实现相关功能.

不过仍然推荐将第三方服务封装成```Commune\Chatbot\OOHost\NLU\Contracts\NLUService``` 对象, 主要是为了实现```NLUService::syncCorpus()``` 方法,
这样就能将 [本地的语料库](/docs/dm/corpus.md) 同步到第三方服务.

## 可被处理的消息

对于 NLU 而言, 并非所有消息都需要调取第三方 API.
首先, 非```Commune\Chatbot\Blueprint\Message\VerbalMsg```(文本类型的消息)的消息,
通常不需要进行处理.

除此之外, 类似 ```1```, ```95.27```这类数字也没必要进行处理. 还有一些 NLU 本身不处理两个字符以内的输入.

这些消息可以通过 ```NLUService::messageCouldHandle()``` 方法排除掉, 避免对第三方服务的无效调用.

## 语义解析

语义解析通常封装在 ```NLUService::match($session)``` 方法中.

它需要把解析结果赋予 ```$session->nlu```, 或者```$conversation->getNLU()``` 提供的 ```Commune\Chatbot\Blueprint\Conversation\NLU``` 对象.
具体实现是 ```Commune\Chatbot\Framework\Conversation\NatureLanguageUnit```.

关于 ```Commune\Chatbot\Blueprint\Conversation\NLU``` 有以下几点值得注意:

__避免重复处理__ : ```NLU::isHandledBy()``` 表示是否已经由别的 NLU 进行过解析.
```NLU::done($nluId)``` 用于告知 NLU 对象已经进行过解析.
两者结合使用, 避免多个 NLU 中间件相互冲突.

__MatchedIntent__ : 表示优先级最高的匹配意图. ```Hearing::runIntent``` 会优先执行它.
当它不存在时, 则会按置信度的高低, 执行 ```NLU::getMostPossibleIntent()```.

__PossibleIntent__ : 表示 NLU 解析到的所有可能意图.
在用 ```NLU::setPossibleIntent()``` 赋值时,
注意 ```$odd``` 参数是整数, 只要能实现排序即可.
而 ```$highlyPossible``` 参数表示置信度高于了阈值.

__Emotions__ : CommuneChatbot 目前的 "情绪" 定义可能和 NLU 并不一致.
当 NLU 提供的功能与系统定义的功能相似时, 这个赋值才有意义.
具体可查看 [Intent 文档中 Emotion 的部分](/docs/dm/intent.md).
注意, 从 NLU 获取到的 Emotion, 应该被转义成继承自 ```Commune\Chatbot\OOHost\Emotion\Emotion``` 接口的类名.

__Words__ : NLU 如果可以提供分词功能, 可以赋值到这个参数.
那么 ```Hearing::hasKeywords()``` 方法将优先使用这里保存的分词进行匹配.

__FocusIntents__ : 这个方法可以获取到多轮对话现在 "关注" 的意图,
具体实现通过 ```Commune\Chatbot\OOHost\Context\Stages\FakeHearing```.
或许有些第三方服务, 可以通过传入这些信息, 达到更高效的意图解析.

## 同步语料库

理想情况下, CommuneChatbot 的本地语料库 ```Corpus```,
可以通过 ```NLUService::syncCorpus()``` 同步到第三方服务.
从而减轻对第三方服务的深度耦合.

可以定义一个专门管理 ```NLUService``` 的多轮对话,
在该多轮对话中依赖注入 ```NLUService`` 的对象, 根据用户意图执行同步方法.

系统自带的管理工具是 ```Commune\Chatbot\OOHost\NLU\Contexts\CorpusManagerTask```,
在 [管理 NLU 服务](/docs/nlu/manager.md) 一篇中介绍如何使用.

## 使用 Component 提供配置

由于 NLUService 需要封装对第三方服务的调用, 显然有大量参数应该允许用户自主配置, 例如超时时间, 重试次数等.

可以参考 [组件化](/docs/components/index.md) 一章的做法, 将某个第三方服务的所有功能,
都封装到这个组件里.
然后通过 ```Commune\Chatbot\Framework\Component\ComponentOption``` 提供配置数据,
使得用户可以在配置文件 ```$chatbotConfig->components``` 中自行定义.

该对象可以依赖注入到 ```NLUService``` 中, 提供各种参数. 可以参考:

- ```Commune\Components\Rasa\RasaComponent```
- ```Commune\Components\Rasa\Services\RasaService```

更多信息请查看 [管理 NLU 服务](/docs/nlu/manager.md).

