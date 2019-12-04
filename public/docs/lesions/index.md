# 快速教程

为了方便您快速了解和掌握多轮对话的开发方式, 这里提供了一系列简明的课程. 走完这个流程您能基本了解 CommuneChatbot 多轮对话的开发方式.


* [第一课 : hello world](/docs/lesions/helloworld.md)
* [第二课 : 定义单轮对话](/docs/lesions/single-turn-convo.md)
* [第三课 : 定义一阶多轮对话](/docs/lesions/first-order-convo.md)
* [第四课 : 填槽型多轮对话](/docs/lesions/slot-filling.md)
* [第五课 : 定义依赖关系的 N 阶多轮对话](/docs/lesions/n-order-convo.md)
* [第六课 : 定义不相依赖的 N 阶多轮对话](/docs/lesions/n-thread-convo.md)
* [第七课 : 定义多重会话嵌套]
* [第八课 : 意图与上下文记忆]

如果想要进一步深入了解, 您可能需要 :

- [多轮对话生命周期](/docs/dm/lifecircle.md)
- [定义 Context](/docs/dm/context.md)
- [定义 Stage](/docs/dm/stage.md)
- [定义 Intent](/docs/dm/intent.md)
- [定义 Memory](/docs/dm/memory.md)
- [Dialog API](/docs/dm/dialog.md)
- [Hearing API](/docs/dm/hearing.md)


此外, 项目也给出了大量 Demo 用例, 通过它们的源码, 您也可以看到更多的示范. 列举部分如下:

- ```Commune\Components\Demo\Contexts\DemoHome``` : Demo 多轮对话入口
- ```Commune\Components\Demo\Contexts\FeatureTest``` : 功能点测试
- ```Commune\Components\Demo\Cases\Weather\TellWeatherInt``` : 查询天气
- ```Commune\Components\Demo\Cases\Maze\MazeInt``` : 迷宫小游戏
- ```Commune\Components\Demo\Cases\Drink\OrderJuiceInt``` : 购买饮料
- ```Commune\Components\Demo\Cases\Questionnaire\ReadPersonality``` : 问卷调查
- ```Commune\Chatbot\OOHost\NLU\Contexts\CorpusManagerTask``` : 语料库管理工具






