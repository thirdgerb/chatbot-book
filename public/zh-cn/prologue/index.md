# 项目介绍

> 开源的工程化对话机器人框架, 目的是用编程的方式实现复杂多轮对话管理, 用于开发跨平台的对话机器人或语音应用.

## 1. 简介

"Commune" 是 __亲切交谈__ 的意思. CommuneChatbot 这个项目是想通过 "对话" 的形式提供一种人与机器的交互方式. 在这个项目的思路中, "对话" 并非目的, 而是 "操作机器" 的手段.

简单来说, CommuneChatbot 是一个 :

- 使用 PHP7 开发的开源项目
- 基于 swoole + hyperf 提供协程化的服务端
- 可对接语音, 即时通讯等平台搭建对话机器人
- 最大特点是 __多轮对话管理引擎__, 用于解决 [复杂多轮对话问题](/zh-cn/core-concepts/complex-conversation.md)
- 提供工程化 (模块化/可配置/组件化) 的开发框架
- 目标是以对话的交互形式, 开发出像网站, 触屏App一样复杂的应用

目前的 Demo 有:

* [项目网站](https://communechatbot.com)
* 微信公众号 Demo: 搜索 "CommuneChatbot"
* 百度智能音箱: 对音箱说 "打开三国群英传"

如有兴趣, 可以加入讨论 QQ 群: 907985715

### 1.1 项目构成

-   [Chatbot](https://github.com/thirdgerb/chatbot) : 核心框架
-   [Studio](https://github.com/thirdgerb/studio-hyperf) : 工作站, 基于 swoole + hyperf 开发, 可创建和运行应用
-   [Components](/zh-cn/components/index.md) : 高度组件化 + 配置化地实现各种多轮对话功能

### 1.2 项目定位与特点

[对话机器人的技术架构](/zh-cn/core-concepts/structure.md) 这篇文章提到了现阶段对话机器人可能涉及到的技术领域.

而 CommuneChatbot 的定位是工程开发框架 + 多轮对话管理 :

* 多轮对话管理
    * 初步实现了复杂多轮对话管理
    * 使用工程化的方式开发多轮对话的逻辑
* 工程框架
    * 提供高性能的服务端
    * 对接各种对话平台 (即时通讯, 语音, 网页等)
    * 对接自然语言中间件
    * 对业务逻辑提供基于 IoC 容器的工程化支持
    * 高度组件化, 配置化

至于本项目为什么用规则编程, 而不是机器学习的方式实现多轮对话管理, 可以通过 [工程化还是机器学习](/zh-cn/core-concepts/engineering-or-machine-learning.md) 了解作者的观点.

### 1.3 同类项目

CommuneChatbot 的多轮对话管理, 是用工程化的方式实现的, 类似传统的应用开发. 与此相似的开源项目有:

* [hubot](https://github.com/hubotio/hubot): js框架, github 开发, >15k stars 
* [bokit](https://botkit.ai/): js框架, 似乎要被微软收购了, >9k stars
* [botman](https://botman.io/): php框架, >4k stars

[CommuneChatbot](https://github.com/thirdgerb/chatbot) 目前是一个新生项目, 和以上项目比较虽不够成熟, 但有两个方向上的主要特点:

* 致力于实现 [复杂的 N 阶多轮对话](/zh-cn/core-concepts/complex-conversation.md)
* 有更多面向生产环境的工程设计

还有一些使用机器学习实现多轮对话管理的开源项目, 例如 [chatterbot](https://github.com/gunthercox/ChatterBot) 和 [rasa x](https://rasa.com/zh-cn/rasa-x/). 供读者参考.

### 1.4 关于开发者

CommuneChatbot 项目由 [ThirdGerb](https://github.com/thirdgerb) 基于个人兴趣设计并开发.

作者是一名服务端工程师, 对于对话交互形式的应用有很强的兴趣.
但想要开发的应用往往卡在复杂多轮对话问题上, 而找到的解决方案还不够理想,
因此自己动手开发了这个项目.

作者关于对话机器人的各种思考和观点, 仅供参考.
如果发现错谬之处, 烦请批评指教, 非常感谢!
若感觉有所共鸣, 也希望不吝褒扬, 给予鼓励.


## 2. 主要依赖项目

- 核心依赖
    -   [swoole](https://www.swoole.com/) : PHP扩展, 用于实现高性能的协程服务端
    -   [composer](http://www.getcomposer.org/) : php项目的标准包管理工具
    -   [hyperf](https://hyperf.io/) : 基于 Swoole 的协程开发框架
    -   [illuminate](https://laravel.com/) : Laravel 提供的 PHP 组件库
    -   [symfony](https://symfony.com/) : Symfony 系列组件库
- 应用依赖
    -   [easywechat](https://www.easywechat.com/docs) : PHP的非官方微信SDK, 用于实现公众号服务端
    -   [duerOS-SDK](https://github.com/dueros/bot-sdk) : 用于对接百度智能音箱

CommuneChatbot 的其它依赖, 详见仓库里的 ```composer.json``` 文件和 ```package.json``` 文件.

## 3. 您可能需要

- [快速教程 ★](/zh-cn/lesions/index.md)
    * [1. hello world](/zh-cn/lesions/helloworld.md)
    * [2. 定义单轮对话](/zh-cn/lesions/single-turn-convo.md)
    * [3. 定义一阶多轮对话](/zh-cn/lesions/first-order-convo.md)
    * [4. 填槽型多轮对话](/zh-cn/lesions/slot-filling.md)
    * [5. 依赖关系的 N 阶多轮对话](/zh-cn/lesions/n-order-convo.md)
    * [6. 不相依赖的 N 阶多轮对话](/zh-cn/lesions/n-thread-convo.md)
    * [7. 使用意图](/zh-cn/lesions/intent.md)
    * [8. 使用上下文记忆](/zh-cn/lesions/memory.md)
- 了解项目
    - [项目核心思路](/zh-cn/core-concepts/index.md)
    - [应用生命周期](/zh-cn/app-lifecircle.md)
    - [多轮对话生命周期](/zh-cn/dm-lifecircle.md)
- [搭建应用 ★](/zh-cn/setup/index.md)
    - [安装工作站](/zh-cn/setup/studio.md)
    - [搭建微信公众号 bot](/zh-cn/setup/wechat.md)
    - [搭建百度音箱 bot](/zh-cn/setup/web.md)
    - [搭建 web bot](/zh-cn/setup/web.md)
- 深入了解功能
    - [工程模块](/zh-cn/engineer/index.md)
        - [双容器与依赖注入](/zh-cn/engineer/di.md)
        - [配置体系](/zh-cn/engineer/configuration.md)
        - [消息体系](/zh-cn/engineer/messages.md)
        - [消息渲染](/zh-cn/engineer/replies.md)
        - [管道](/zh-cn/engineer/pipeline.md)
        - [配置中心抽象层](/zh-cn/engineer/abstract-config.md)
        - [多平台适配](/zh-cn/engineer/platform-adapter.md)
    - [对话模块](/zh-cn/dm/index.md)
        - [Session](/zh-cn/dm/session.md)
        - [Context](/zh-cn/dm/context.md)
        - [Stage](/zh-cn/dm/stage.md)
        - [Intent](/zh-cn/dm/intent.md)
        - [Memory](/zh-cn/dm/memory.md)
        - [Dialog](/zh-cn/dm/dialog.md)
        - [Hearing](/zh-cn/dm/hearing.md)

