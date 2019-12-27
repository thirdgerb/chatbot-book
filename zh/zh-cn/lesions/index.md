# 快速教程

为了方便您快速了解和掌握多轮对话的开发方式, 这里提供了一系列简明的课程.
在开始之前, 您可能需要阅读 [多轮对话生命周期](/zh-cn/dm-lifecircle.md) 这篇文章.

以下是正式的内容:

* [1. hello world](/zh-cn/lesions/helloworld.md)
* [2. 定义单轮对话](/zh-cn/lesions/single-turn-convo.md)
* [3. 定义一阶多轮对话](/zh-cn/lesions/first-order-convo.md)
* [4. 填槽型多轮对话](/zh-cn/lesions/slot-filling.md)
* [5. 依赖关系的 N 阶多轮对话](/zh-cn/lesions/n-order-convo.md)
* [6. 不相依赖的 N 阶多轮对话](/zh-cn/lesions/n-thread-convo.md)
* [7. 使用意图](/zh-cn/lesions/intent.md)
* [8. 使用上下文记忆](/zh-cn/lesions/memory.md)

如果想要进一步深入了解, 您可能需要 :

- [定义 Context](/zh-cn/dm/context.md)
- [定义 Stage](/zh-cn/dm/stage.md)
- [定义 Intent](/zh-cn/dm/intent.md)
- [定义 Memory](/zh-cn/dm/memory.md)
- [Dialog API](/zh-cn/dm/dialog.md)
- [Hearing API](/zh-cn/dm/hearing.md)


此外, 项目也给出了大量 Demo 用例, 通过它们的源码, 您也可以看到更多的示范. 列举部分如下:

- ```Commune\Components\Demo\Contexts\DemoHome``` : Demo 多轮对话入口
- ```Commune\Components\Demo\Contexts\FeatureTest``` : 功能点测试
- ```Commune\Components\Demo\Cases\Weather\TellWeatherInt``` : 查询天气
- ```Commune\Components\Demo\Cases\Maze\MazeInt``` : 迷宫小游戏
- ```Commune\Components\Demo\Cases\Drink\OrderJuiceInt``` : 购买饮料
- ```Commune\Components\Demo\Cases\Questionnaire\ReadPersonality``` : 问卷调查
- ```Commune\Chatbot\OOHost\NLU\Contexts\CorpusManagerTask``` : 语料库管理工具






