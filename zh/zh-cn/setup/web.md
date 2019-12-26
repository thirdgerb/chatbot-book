# 搭建 Web bot

Web 版对话机器人, 通常用 JS 开发的浏览器应用作为前端, 通过 http 与服务端通信.
使用无状态的 Http 协议, 机器人的逻辑会相对简单一些.
更好的做法是用 WebSocket 通信, 这样的机器人可以拥有双工的能力.

CommuneChatbot 开箱自带的 Web 机器人是基于 Http 的.
组件是 ```Commune\Platform\Web\WebComponent```.
它给出了 Web 版机器人的一个示范.
用类似的思路, 可以将对话机器人用于网页上的任何对话界面中.

## 安装前端界面

CommuneChatbot 自带的 Web 机器人, 前端代码在 ```BASE_PATH/app-frontend``` 下.

需要进入该目录, 先安装依赖.

```
npm install
```

可以测试编译代码 :

```
npm run dev
```

也可以监视代码, 改动后自动编译 :

```
npm run watch
```

生成生产级的代码, 请使用 :

```
npm run prod
```

生成的前端资源会进入 ```BASE_PATH/app-frontend/public/``` 目录.
可以通过 Nginx 将该目录对外暴露.

前端工程主要使用了以下项目:

- [vue](https://vuejs.org/) : 大名鼎鼎的 MVVM 框架
- [vuetify](https://vuetifyjs.com/) : 非常优秀的 martial design 风格界面
- [axios](https://github.com/axios/axios) : 处理 http 请求
- [Bubble](https://github.com/dmitrizzle/chat-bubble) : 用来做对话交互效果
- [LaraveMix](https://laravel-mix.com/) : 简化 Webpack 的配置

具体效果请查看 [官方网站](https://communechatbot.com).

这个前端工程是为示范的目的, 作者用几天时间临时开发的.
如有问题烦请反馈.
在生产环境中使用的 Web 对话机器人, 建议参考本项目的实现, 另行专门开发.

## 与服务端通信

前端工程 app-frontend 与服务端通信的地址配置在 ```BASE_PATH/app-frontend/src/js/app.js``` 文件中.

```
// web 机器人地址
const WEB_URI = '/web';
// api 机器人地址
const API_URI = '/api';
```

> 要注意网页, API 机器人, Web 机器人的部署地址, 避免跨域问题的烦恼.

注意它同时需要 Web 和 [API](/zh-cn/setup/api.md) 两个服务端.
因为这个工程尝试了双模态, 当用户与 Web 机器人对话时, 随着多轮对话状态变更,
前端也会同步变更状态.
这时点击界面右上角 "<>" 按钮, 就能显示出当前 Context 的源代码 (通过调用 API 机器人).

因此客户端和 Web 对话机器人的状态是同步的. 后者成为状态中控.
这方面更多思路, 请查看[多模态中控](/zh-cn/core-concepts/multimodal.md)

## 使用 WebComponent

WebComponent 的组件类是 ```Commune\Platform\Web\WebComponent```,
将它引入机器人, 可以启动服务端和 ```app-frontend``` 结合使用.

### 创建应用

在 ```BASE_PATH/config/autoload/commune.php``` 中创建 Web 机器人应用:

```php
return [

    ...

    'apps' => [

        // 系统自带的 web 端, 示范如何开发 web 端的对话机器人.
        'web' => include BASE_PATH . '/config/commune/apps/web.php',

        ...

    ],
];
```

### 服务端配置

服务端配置请参考```BASE_PATH/config/commune/apps/web.php```.
配置结构是```Commune\Hyperf\Foundations\Options\AppServerOption```.

最重要的是 servers 的 callback, 需要用 ```Commune\Platform\Web\Servers\WebServer``` 类来响应 ```SwooleEvent::ON_REQUEST```.

### 机器人配置

在机器人配置中引入 ```Commune\Platform\Web\WebComponent```:

```php

return [
    ...

    'components' => [
        ...

        // 使用默认配置
        Commune\Platform\Web\WebComponent::class,
    ],
    ...
];

```

WebComponent 的默认配置内容如下 :

```

    public static function stub(): array
    {
        return [
            // 渲染接口返回的 Render
            'apiRender' => DemoResponseRender::class,

            // 只能接受文字消息, 允许的最大字符数
            'maxInputLength' => 100,

            // 对回复消息进行渲染的模板. 主要是为了做 Conversational 的交互效果
            'replyRenders' => [

                // base question
                QA\VbQuestion::REPLY_ID => WebQuestionTemp::class,
                QA\Confirm::REPLY_ID => WebConfirmTemp::class,
                QA\Choose::REPLY_ID => WebQuestionTemp::class,
                QA\Selects::REPLY_ID => WebQuestionTemp::class,

                // intent question
                QA\Contextual\AskEntity::REPLY_ID => WebQuestionTemp::class,
                QA\Contextual\ConfirmIntent::REPLY_ID => WebConfirmTemp::class,
                QA\Contextual\ConfirmEntity::REPLY_ID => WebConfirmTemp::class,
                QA\Contextual\ChooseIntent::REPLY_ID => WebQuestionTemp::class,
                QA\Contextual\ChooseEntity::REPLY_ID => WebQuestionTemp::class,
                QA\Contextual\SelectEntity::REPLY_ID => WebQuestionTemp::class,

            ]
        ];
    }
```

## WebRequest

WebComponent 的 MessageRequest 是 ```Commune\Platform\Web\Servers\WebRequest``` 类. 它有以下几点值得注意:

```WebRequest::validateMethod()``` 只接受 Post 请求.

```WebRequest::getScene()``` 从 url 的 query 参数 "scene" 中获取 "场景" 的值. 例如```url/web/?scene=demo.cases.maze```

```WebRequest::parseInput()``` 从请求的 body 中解析 json, 以获取请求数据.

```WebRequest::validateUserId()``` 默认从 cookie 中获取用户 ID, 如果没有会生成一个 uuid 设置到 cookie 中.
完全可以在此处改为从 JWT 中获取用户信息.

```WebRequest::makeInputMessage()``` web 机器人目前只接受纯文本的消息.

显然, 重做 WebRequest, 可以实现各种各样的请求形式. 无论格式, 参数, 身份验证方法都可以改变.

## 渲染消息

Web 机器人的消息渲染分为两部分. 其一是将 ReplyId 渲染成真正的消息, 其二是将消息渲染成 Http 响应.

```$webComponent->replyRenders``` 可以定义各种 ReplyId 的渲染模板.

```$webComponent->apiRender``` 可以定义渲染器 ```Commune\Platform\Web\Contracts\ResponseRender``` 的实现.
默认实现是 ```Commune\Platform\Web\Libraries\DemoResponseRender```,
只支持渲染 文字/图片/链接 三种类型的消息.

## 同步会话状态

WebRequest 实现了接口 ```Commune\Chatbot\OOHost\Dialogue\NeedDialogStatus```.

这使得它在多轮对话内核进入 ```Wait``` 状态时, 会调用 ```WebRequest::logDialogStatus()```  方法.
通过该方法, 可以将 Dialog 的状态信息同步给客户端.

进一步地可以做成广播机制, 从而为 [多模态中控](/zh-cn/core-concepts/multimodal.md) 提供解决方案.

