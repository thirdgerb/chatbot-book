# 开发对话与服务

工作站的机器人能正常启动之后, 就可以开发自己机器人的逻辑了.
通常自主开发包括两个方面:

- 对话逻辑开发
- 功能服务开发

## 对话逻辑开发

CommuneChatbot 主要通过面向对象的类 (```Commune\Chatbot\OOHost\Context\Context```),
来定义多轮对话逻辑.
相关知识可以通过以下文档来了解 :

- [快速教程](/zh-cn/lesions/index.md)
- __对话模块__
    - [多轮对话生命周期](/zh-cn/dm-lifecircle.md)
    - [定义 Context](/zh-cn/dm/context.md)
    - [定义 Stage](/zh-cn/dm/stage.md)
    - [定义 Intent](/zh-cn/dm/intent.md)
    - [定义 Memory](/zh-cn/dm/memory.md)
    - [Questions 机制](/zh-cn/dm/questions.md)
    - [Hearing API](/zh-cn/dm/hearing.md)
    - [Commands](/zh-cn/dm/commands.md)

自定义的 Context 类, 建议放在工作站的 ```BASE_PATH/app-studio/Contexts/``` 目录下.
这样配合机器人配置 ```$chatbotConfig->host->autoloadPsr4```,
这些 Context 类会自动加载到系统中.

当确定了自定义机器人的各种根语境之后,
应该修改配置 ```$chatbotConfig->host->rootContextName```,
以及```$chatbotConfig->host->sceneContextNames```.
以此来修改对话机器人对用户的第一响应.

## 功能服务开发

作为一个工程化的框架, CommuneChatbot 引入功能服务主要通过 IoC 容器与依赖注入.
相关文档在 [双容器策略与依赖注入](/zh-cn/engineer/di.md).

理想的做法是:

1. 先定义功能模块的类, 最好有 Interface
1. 通过 ServiceProvider 类, 定义注册服务的逻辑
1. 将 ServiceProvider 类通过配置注册到 IoC 容器中
    - ```$chatbotConfig->processProviders``` 进程级服务
    - ```$chatbotConfig->conversationProviders``` 请求级服务
    - ```$chatbotConfig->baseServices``` 系统默认服务
1. 在具体的应用场景, 进行依赖注入.

在 [双容器策略与依赖注入](/zh-cn/engineer/di.md) 文档末尾,
列出了 CommuneChatbot 现有的几十种服务.
可以参考它们的实现, 将自己的服务引入系统来.

## 组件化开发

进一步的, 如果一个复合功能继续要提供多轮对话逻辑, 又可以提供多种功能服务.
这时可以考虑用 "组件" 将之整合起来.

"组件" 是一种集配置和启动流程一体的工程策略. 具体可以查看 [组件化相关文档](/zh-cn/components/index.md).


