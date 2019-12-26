# 搭建微信公众号 bot

CommuneChatbot 的工作站  [commune/studio-hyperf](http://packagist.org/packages/commune/studio-hyperf) 开箱自带微信公众号的平台适配器.

这是一个 composer 包 [commune/platform-wechat](http://packagist.org/packages/commune/platform-wechat), 将独立进行迭代.
底层使用了 [Easywechat](https://www.easywechat.com/docs) 项目提供公众号功能.

CommuneChatbot 的微信端 Demo 在微信公众号 "CommuneChatbot" 上, 欢迎查看.

搭建微信公众号需要有公众号的相关知识, 这里会简单介绍几项.

## 创建测试号

要搭建微信公众号机器人, 有几个先决条件:

1. 申请了公众号
1. 服务器部署了公众号机器人应用
1. 服务器有域名解析
1. 将机器人域名配置到公众号回调地址

这样, 当公众号接受到消息时, 会调用机器人的域名, 让机器人去响应对话逻辑.

然而在测试阶段, 并不需要用真正的微信公众号. 使用个人微信帐号绑定微信测试号即可.
相关知识请自行搜索了解.


## 创建公众号对话机器人

接下来介绍在工作站中搭建公众号的对话机器人.

### 配置环境变量

首先公众号如果要对接自己的机器人 API, 需要在公众号 (测试号) 上配置好几个关键参数:

1. APP_ID : 公众号 ID
1. SECRET : 密钥
1. TOKEN : 校验 token
1. AES_KEY : 加密的 key, 通常测试号不需要配.

这些参数不适合放到代码库中间, 推荐使用 [Hyperf 环境变量](https://doc.hyperf.io/#/zh-cn/config?id=%e7%8e%af%e5%a2%83%e5%8f%98%e9%87%8f) 去记录.

工作站默认的公众号环境变量为 :

```
# 公众号服务端监听的端口
CHAT_WECHAT_PORT=9503

# 公众号使用独立的 redis 配置比较好.
WECHAT_REDIS_HOST=localhost
WECHAT_REDIS_AUTH=
WECHAT_REDIS_PORT=6379
WECHAT_REDIS_DB=0

# 公众号的相关参数
WECHAT_APP_ID=
WECHAT_APP_SECRET=
WECHAT_TOKEN=
WECHAT_AES_KEY=
```

这些环境变量, 可以在配置文件中使用 ```env('WECHAT_APP_ID', '')``` 的方式来获取.

### 创建应用配置

公众号机器人也需要在```BASE_PATH/config/autoload/commune.php``` 文件中创建.

```php
return [

    ...

    'apps' => [

        ...

        // 系统自带的 微信公众号服务端. 可作为微信公众号的服务.
        'wechat' => include BASE_PATH . '/config/commune/apps/wechat.php',
    ],
];
```

### 服务端配置

公众号的服务端配置仍然是 ```Commune\Hyperf\Foundations\Options\AppServerOption```.

最重要的是 servers 的 callback, 需要用 ```Commune\Platform\Wechat\Servers\OfficialAccountServer``` 类来响应 ```SwooleEvent::ON_REQUEST```.

### 机器人配置

在机器人配置中需要引入 ```Commune\Platform\Wechat\WechatComponent``` 组件:

```php

return [

    ...

    components => [
        ...

        // 微信公众号的配置
        \Commune\Platform\Wechat\WechatComponent::class => [

            // 公众号的配置
            'wechatConfig' => [
                'app_id'  => env('WECHAT_APP_ID', ''),
                'secret'  => env('WECHAT_APP_SECRET', ''),
                'token'   => env('WECHAT_TOKEN', ''),
                'aes_key' => env('WECHAT_AES_KEY', ''),
            ],

            // 可选, EasyWechat 各种模块的引入.
            'serviceProvider' => WechatAppServiceProvider::class
        ],
    ],

    ...
];

```

底层使用的是 [EasyWechat](https://www.easywechat.com/docs) 项目,
会自动把该项目的 ```Logger```, ```Cache``` 等对象替换为 CommuneChatbot 所用的实例.

### 运行服务端实例

和其它机器人一样的方式启动服务端实例:

```
php bin/hyperf.php commune:start wechat
```

### 提供 Url

公众号机器人的服务端实例也是监听端口, 还需要对外提供域名. 可以用 nginx 反向代理.

## 接受消息

公众号机器人的 MessageRequest 类是 ```Commune\Platform\Wechat\Servers\OfficialAccountRequest``` .
它需要负责把微信公众号的消息, 与 CommuneChatbot 的 [消息体系](/zh-cn/engineer/messages.md), 进行互相转换.

### 默认支持的消息

当前版本支持的微信端输入消息, 可见 ```OfficialAccountRequest::makeInputMessage()``` 方法:

- Text : 文字类型消息
- Event : 事件类消息
- VOICE : 语音类消息, 需要设置微信公众号自动识别语音消息
- IMAGE : 图片类消息

其它的消息暂时作为不支持(```Commune\Chatbot\App\Messages\Unsupported```)的消息.

### 自定义消息转换

如果在 Conversation 容器中绑定了 ```wechat.msgType.$type``` 类型的服务,
则会用 ```$conversation->make('wechat.msgType.$type')``` 的方式获得转义后的消息.

例如 :

```php

class MyWechatServiceProvider extends ServiceProvider
{
    const IS_PROCESS_SERVICE_PROVIDER = false;


    public function register()
    {
        $this->app->singleton(
            'wechat.msgTypes.Text',

            function($app) {
                $conversation = $app[Conversation];
                $request = $conversation->getRequest();

                $input = $request->getWechat()->server->getMessage();

                return ...
            }

    }
}
```

> 微信客户端的语音很好用, 推荐试一试. 围绕语音会有一些适配. 例如语音解析说到 "0" 时, 会解析成 "零。" 系统会尝试自动转义成 "0".

## 回复消息

从 CommuneChatbot 回复消息到微信公众号, 理论上支持任何类型的消息.
这类消息默认的类是 ```Commune\Platform\Wechat\Messages\TemplateMessage```.

系统定义了两个默认的微信类型消息 :

- 图片 : ```Commune\Platform\Wechat\Messages\WechatImage```
- 语音 : ```Commune\Platform\Wechat\Messages\WechatAudio```

可以继承 TemplateMessage 类, 定义更多的回复消息类型.
关键在于 ```TemplateMessage::toMessageData()``` 方法需要返回 [EasyWechat 允许的回复消息](https://www.easywechat.com/docs/4.1/official-account/messages).

> !!!注意 : 为了机器人的适配性, 不应该在 Stage 逻辑里直接返回 TemplateMessage, 而应该在模板渲染的环节, 将 ReplyId 渲染成指定的 TemplateMessage.

CommuneChatbot 默认的文字类消息```Commune\Chatbot\App\Messages\Text``` 则会直接渲染成公众号上的文字回复.

> 微信公众号目前一次用户消息, 只允许同步返回一条服务端消息. 因此多个 Text 也会被合并成一条消息.
> 而 [问答模块](/zh-cn/dm/questions.md) 的选项也无法做互动, 只能靠用户自己输入数字.


查看 ```OfficialAccountRequest::renderSingleMessage()``` 方法可知,
组件默认提供了 ```Commune\Platform\Wechat\Contracts\MessageBabel``` 接口.

如果这个接口的实现注册到 IoC 容器中, 则会调用它来执行 CommuneChatbot 消息向 EasyWechat 消息的转义.


## 持续迭代

由于 CommuneChatbot 项目自身还在草创, 还没针对微信公众号做完善.
所以 [commune/platform-wechat](http://packagist.org/packages/commune/platform-wechat) 库会独立进行迭代, 以提供更多的功能.