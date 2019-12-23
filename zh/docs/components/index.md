# 组件化

作为工程化的开发框架, CommuneChatbot 致力于实现 组件化 + 可配置.
无论是工程模块, 还是多轮对话本身, 都可以封装为独立的组件, 通过 composer 的 package 引入项目.

这种组件化的能力基于 ```Commune\Chatbot\Framework\Component\ComponentOption``` 实现.
它既是配置数据, 又是进程级单例 (原理见[配置体系](/docs/engineer/configuration.md)), 同时又参与项目的启动流程. 可以按照自定义的配置, 将需要的各种功能模块引入系统.

CommuneChatbot 目前使用的组件如下:

- 预定义对话功能 : ```Commune\Components\Predefined\PredefinedComponent```
- Demo 多轮对话 : ```Commune\Components\Demo\DemoComponent```
- NLU 组件 : ```Commune\Chatbot\OOHost\NLU\NLUComponent```
- 对话游戏 : ```Commune\Components\Story\StoryComponent```
- 简单闲聊 : ```Commune\Components\SimpleChat\SimpleChatComponent```
- 简单问答 : ```Commune\Components\SimpleWiki\SimpleWikiComponent```
- Rasa 组件 : ```Commune\Components\Rasa\RasaComponent```
- 小度音箱组件 : ```Commune\Platform\DuerOS\DuerOSComponent```
- 微信公众号组件 : ```Commune\Platform\Wechat\WechatComponent```
- 工作站 Demo : ```Commune\Hyperf\Demo\HyperfDemoComponent```
- 网页版组件 : ```Commune\Platform\Web\WebComponent```
- API版组件 : ```Commune\Platform\WebApi\WebApiComponent```

相关的开发与使用文档如下:

- [定义组件](/docs/components/option.md)
- [简单闲聊组件](/docs/components/simple-chat.md)
- [简单问答组件](/docs/components/simple-wiki.md)
- [文字冒险游戏](/docs/components/story.md)