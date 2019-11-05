# CommuneChatbot

> 开源的对话机器人框架, 可实现复杂多轮对话, 用于开发跨平台的对话机器人或语音应用.

## 项目简介

"Commune" 是亲切交谈的意思. CommuneChatbot 这个项目是想通过 "对话" 的形式提供一种人与机器的交互方式. 在这个项目的思路中, "对话" 并非目的, 而是 "操作机器" 的手段.

简单来说, CommuneChatbot 是一个 :

- 使用 PHP7 开发的开源项目
- 基于 swoole + hyperf 提供协程化的服务端
- 可对接语音, 即时通讯等平台搭建对话机器人
- 最大特点是可实现复杂多轮对话管理
- 目标是以对话的交互形式, 开发出像网站, 触屏App一样复杂的应用

目前的 Demo 有:

* [项目网站](https://communechatbot.com)
* 微信公众号 Demo: 搜索 "CommuneChatbot"
* 百度智能音箱: 对音箱说 "打开三国群英传"

## 项目构成

-   [Chatbot](https://github.com/thirdgerb/chatbot) : 核心框架
-   [Studio](https://github.com/thirdgerb/studio-hyperf) : 工作站, 基于 swoole + hyperf 开发, 可创建和运行应用
-   Components : 高度组件化 + 配置化地实现各种多轮对话功能

## 项目定位与特点

[对话机器人的技术架构](/docs/core-concepts/structure.md) 这篇文章提到了现阶段对话机器人可能涉及到的技术领域. 

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

多轮对话管理部分, 是用工程化的方式实现的, 类似传统的应用开发. 与此相似的开源项目有:

* [hubot](https://github.com/hubotio/hubot): js框架, github 开发, >15k stars 
* [bokit](https://botkit.ai/): js框架, 似乎要被微软收购了, >9k stars
* [botman](https://botman.io/): php框架, >4k stars

[CommuneChatbot](https://github.com/thirdgerb/chatbot) 目前是一个新生项目, 和以上项目比较虽不够成熟, 但有两个方向上的主要特点:

* 可以实现 [复杂的N阶多轮对话](/docs/core-concepts/complex-conversation.md)
* 有更多面向生产环境的工程设计

还有一些使用机器学习实现多轮对话管理的开源项目, 例如 [chatterbot](https://github.com/gunthercox/ChatterBot) 和 [rasa x](https://rasa.com/docs/rasa-x/). 供读者参考. 

## 主要依赖项目

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

## 您可能还想了解

- [关于对话机器人](docs/articles/about-chatbot.md)
- 对话机器人可以干什么
- 复杂多轮对话问题
- CommuneChatbot 项目特点


## 您可能还需要

- 快速上手
- 新手课程
- [官方Demo](https://communechatbot.com)
- 立刻创建微信公众号机器人
- 立刻创建百度智能音箱机器人
- 立刻创建网页版机器人

