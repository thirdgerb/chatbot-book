# 使用自然语言单元

自然语言单元 (以下简称 "NLU" ) 可作为功能模块引入, 通过调用第三方 API, 为 CommuneChatbot 对话管理引入自然语言解析的能力.

CommuneChatbot 主要使用的自然语言解析功能, 是 __语义匹配__ 与 __实体提取__, 还可能用到 __情绪解析__ .

推荐通过以下两篇文章了解 NLU 的功能和使用 :

- [自然语言单元](/zh-cn/core-concepts/nlu.md)
- [意图与实体在对话管理中的封装](/zh-cn/dm/intent.md)

简单来说, 系统需要把第三方的自然语言解析单元封装为```Commune\Chatbot\OOHost\NLU\Contracts\NLUService``` 服务, 注册到应用中.
然后在配置文件里加载该 NLU 的 Session 中间件, 调用服务来解析消息.
然后再维护一个本地语料库, 用于维护各种 NLU 服务的语料库一致性.

这几个功能的整合都在 [管理 NLU 服务](/zh-cn/nlu/manager.md) 一篇中进行介绍.

系统目前自带使用开源项目 [Rasa](https://rasa.com/) 提供的 NLU,
相关文档在 [使用 Rasa 搭建 NLU](/zh-cn/nlu/rasa.md).
未来会陆续添加其它各种解决方案 (百度UNIT, 其它云端 API, 开源项目等) .

相关文档如下:

- [封装 NLUService](/zh-cn/nlu/service.md)
- [注册 NLU 中间件](/zh-cn/nlu/middleware.md)
- [本地语料库](/zh-cn/nlu/corpus.md)
- [管理 NLU 服务](/zh-cn/nlu/manager.md)
- [使用 Rasa 搭建 NLU](/zh-cn/nlu/rasa.md)