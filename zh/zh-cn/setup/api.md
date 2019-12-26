# 搭建机器人 API

CommuneChatbot 有许多内部的功能模块, 如 [配置中心抽象层](/zh-cn/engineer/abstract-config.md) 和 [Session](/zh-cn/dm/session.md) 等,
在项目外部不好直接访问.
又有很多服务是用 ServiceProvider 形式加载的, 配置也都在 CommuneChatbot 内部.

当我们试图用另一个界面, 例如一个后台网站来管理对话机器人时, 就不免需要一些 API, 可以调用到机器人的内部功能.

因此 CommuneChatbot 提供一种简单的方式, 直接将对话机器人架构改造成一个 MVC 框架,
对外暴露 Http API, 可以直接调用机器人的内部服务.

## 原理

工作站  [commune/studio-hyperf](http://packagist.org/packages/commune/studio-hyperf) 基于 Swoole + Hyperf,
可以提供高性能的 Http 服务端.

而 CommuneChatbot 框架本身是基于 [管道](/zh-cn/engineer/pipeline.md) 运行的,
管道的构成决定最终的运行逻辑.

抛开管道提供的对话机器人逻辑的话, 项目本身就是一个 IoC 容器 + 管道的基础框架, 其实可以修改成任何类型的项目.

因此想要得到一个 MVC 框架, 真正需要修改的是以下几个部分 :

- API 专用的 ```Commune\Chatbot\Blueprint\Conversation\MessageRequest```
- 提供类似 Controller 的机制, 用于响应 MessageRequest
- 修改管道
    - 方案1 : 修改 ```$chatbotConfig->chatPipes```, 彻底不走对话管理内核.
    - 方案2 : 修改 ```$chatbotConfig->host->sessionPipes```, 这样能拿到 Session.

系统自带的 WebApiComponent, 使用的是第二种方案, 提供一个没有IO开销的 SessionPipe, 用于 MVC 框架的响应逻辑.
由于 Swoole + Hyperf 的高性能, 并发性能还比较高.

## WebAPIComponent 解决方案

引入组件 ```Commune\Platform\WebApi\WebApiComponent``` 可以得到默认的 API 解决方案.
这是一个示范性的解决方案, 实际可用; 但我们还是认为根据业务场景设计自己的 API 端更合理.

具体的做法有以下几步:

### 创建应用

机器人 API 也需要在```BASE_PATH/config/autoload/commune.php``` 文件中创建.

```php
return [

    ...

    'apps' => [

        // 系统自带的 web 端, 示范如何开发 web 端的对话机器人.
        'web' => include BASE_PATH . '/config/commune/apps/web.php',

        // api 端
        // 可以像 mvc 框架那样通过 http api 访问机器人的数据
        'api' => include BASE_PATH . '/config/commune/apps/api.php',

        ...
    ],
];
```

然后这个机器人可以通过 ```commune:start``` 命令开启端口监听:

```
php bin/hyperf.php commune:start api
```

### 服务端配置

服务端配置请参考```BASE_PATH/config/commune/apps/api.php```.
配置结构是```Commune\Hyperf\Foundations\Options\AppServerOption```.

```php

use Hyperf\Server\Server;
use Hyperf\Server\SwooleEvent;

$chatbot = include BASE_PATH . '/config/commune/chatbots/api.php';

return [

    'chatbot' => $chatbot,

    'redisPool' => 'default',

    'dbPool' => 'default',

    'bufferMessage' => false,

    'shares' => [],

    'server' => [
        'mode' => SWOOLE_PROCESS,
        'servers' => [
            [
                'name' => 'wechat',
                'type' => Server::SERVER_HTTP,
                'host' => 'localhost',

                // 配置监听端口
                'port' => intval(env('CHAT_API_PORT', 9531)),

                'sock_type' => SWOOLE_SOCK_TCP,
                'callbacks' => [

                    // !!!!! 最重要的一行, 需要用 ApiService 来响应请求.
                    SwooleEvent::ON_REQUEST => [\Commune\Platform\WebApi\Servers\ApiServer::class, 'onRequest'],
                ],
            ],
        ],


        'settings' => [
            'enable_coroutine' => true,

            // worker 进程数量
            'worker_num' => 1,

            'pid_file' => BASE_PATH . '/runtime/pid/api.pid',
            'open_tcp_nodelay' => true,
            'max_coroutine' => 100000,
            'open_http2_protocol' => true,
            'max_request' => 100000,
            'socket_buffer_size' => 2 * 1024 * 1024,
        ],

        'callbacks' => [
        ],
    ],
];
```


这个配置最重要的还是 servers 的 callback, 需要用 ```\Commune\Platform\WebApi\Servers\ApiServer``` 类来响应 ```SwooleEvent::ON_REQUEST```,
这样就会将 Http 请求封装到 ```Commune\Platform\WebApi\Servers\ApiRequest``` 对象.
修改这个类, 可以设计自己的 ApiRequest.

### 机器人配置

定义机器人配置时, 首先要考虑这个 API 的定位. 如果它是某个具体机器人的 API, 那么:

* ```ChatbotName``` 应该和目标机器人一致, 这样才能有相同作用域
* AppServerOption 中的数据库应该配置一致, 才能读取相同数据
* 各种服务注册应该保持一致, 这样才能调用到相关功能模块

除此之外, 有以下几个关键环节需要定义:


```php

return [
    ...

    'components' => [
        ...

        // 引入该组件, 并定义路由.
        \Commune\Platform\WebApi\WebApiComponent::class => [

            // get 请求对应的 Action
            'getActions' => [
                    'hello-world' => \Commune\Platform\WebApi\Demo\HelloWorldAction::class,
                    'context-code' => \Commune\Platform\WebApi\Demo\GetContextCode::class,
                ],

            // post 请求对应的 Action
            'postActions' => [

            ]
        ]
    ],

    'chatbotPipes' => [
        'onUserMessage' => [
            \Commune\Chatbot\App\ChatPipe\UserMessengerPipe::class,

            // !!! api 不需要 lock chat. 因此要去掉 chatting pipe
            // \Commune\Chatbot\App\ChatPipe\ChattingPipe::class,

            \Commune\Chatbot\OOHost\OOHostPipe::class,
        ],
    ],

    'host' => [
        ...

        // Session 管道只需要一个就行了, 也就是 ApiActionMatcher
        'sessionPipes' => [

            // 使用 这个中间件, 仍然会用 请求的 MessageId 来创建 Session
            // 等于每一个请求生成一个 Session, 但没有 IO, 所以性能较快
            // 获取 Session 的目的是为了方便获取 Session::$driver 等组件.
            // 而不是真正要一个与用户相关的 Session.
            \Commune\Platform\WebApi\SessionPipes\ApiActionMatcher::class,
        ],

    ],

    ...


];

```

### ApiRequest

WebApiComponent 使用的 MessageRequest 是 ```Commune\Platform\WebApi\Servers\ApiRequest```.

它默认是无状态的请求, 也不区分用户. 有以下几点值得注意:

```MessageRequest::getScene()``` 得到当前指定的路由. 实际上根据 url 的 query 参数 "action" 来判断. 例如一个合法的请求是 ```url/api/?action=hello-world```.

```MessageRequest::getInput()``` 方法得到一个数组表示的请求参数. 如果是```GET```请求, 该参数是 ```$swooleReq->get``` 的值. 如果是 ```POST``` 请求, 该参数是 ```$swooleReq->post``` 的值.

```MessageRequest::validate()``` 方法会判断路由是否存在. 如果不存在, 返回 400 响应.

```MessageRequest::getUserId()``` 用户相关的方法可以自行定义. 默认是用户无关的. 如果要与用户相关, 则应该能从请求的 cookie 或别的元素中, 正确获取用户身份. 建议使用 JWT 来获取用户信息, 方便与其它机器人保持一致.

```MessageRequest::sendResponse()``` 定义了返回响应的格式. 默认是一个 Json 对象 ```{"code":errCode, "msg":errMsg, "data":data}```.

至于如何修改这个类, 实现业务所需的响应逻辑, 这里就不赘述了.

### Action

WebApiComponent 使用 Action 而非 Controller 来响应请求.
Action 的基础类是```Commune\Platform\WebApi\Libraries\AbstractAction```.

一个 Action类响应一个```MessageRequest::getScene()``` 返回的 action. 逻辑可以自行定义.

例如:

```php

class HelloWorldAction extends AbstractAction
{
    public function validateInput(array $input): ? string
    {
        return null;
    }

    public function doHandle(Session $session, array $input): ApiResult
    {
        return new ApiResult(['reply' => 'hello world']);
    }
}

```

Action 类可以通过构造方法进行依赖注入. 所以可以注入已注册的各种功能模块和服务.

如果 ```Action::validateInput()``` 方法返回了字符串, 表示请求不合法, 字符串会作为 ```errMsg``` 响应客户端.

而合法响应需要封装 ```Commune\Platform\WebApi\Libraries\ApiResult``` 对象, 会自动解析成 Json 返回客户端.

### 定义路由

WebApiComponent 可用于定义路由. 这里路由都很简单, 仅仅从 url 的 query 参数```?action=actionName``` 来获取. 每个 action 需要指定一个处理的 Action 类.

```php

// 引入该组件, 并定义路由.
\Commune\Platform\WebApi\WebApiComponent::class => [

    // get 请求对应的 Action
    'getActions' => [
            'hello-world' => \Commune\Platform\WebApi\Demo\HelloWorldAction::class,
            'context-code' => \Commune\Platform\WebApi\Demo\GetContextCode::class,
        ],

    // post 请求对应的 Action
    'postActions' => [

    ]
]
```

### Url 解析

工作站运行的服务端实例, 只负责监听端口.
需要对外暴露 Url, 请考虑使用 Nginx 反向代理等方式.

以测试用例的 nginx 配置为例:

```
...

upstream api {
    server 127.0.0.1:9531;
}

server {
    listen 80;
    server_name communechatbot.test;

    ...

    # api 的访问路径为 http://communechatbot.test/api/?action=actionName
    location /api {
        proxy_pass http://api;
    }

    ...

}

```