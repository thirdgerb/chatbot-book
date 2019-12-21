# Dialog

Dialog 对象是当前多轮对话的管理者, 持有各种数据和 api, 相当于浏览器中的 ```window``` 对象.

关于 Dialog 对象的详细 api 定义, 请查看```Commune\Chatbot\OOHost\Dialogue\Dialog```, 具体的实现类则是```Commune\Chatbot\OOHost\Dialogue\DialogImpl```

## 获取 Dialog 对象

Dialog 对象可以在定义 Stage 的时候, 通过依赖注入传递给 callable 对象:

```php

public function __onStart(Stage $stage) : Navigator
{
    return $stage->talk(
        function(Dialog $dialog) {
            ...
        },
        function(Dialog $dialog) {
            ...
        }
    );
}

```

## 通过 Dialog 发送回复

虽然可以通过 Conversation 对象直接发送回复, 但在具体的语境中, Dialog 才是首选项.

### 发送一个封装好的消息

```php
    $message = new Text('hello world');
    $dialog->reply($message);
```

### 使用 DialogSpeech 说话

使用 ```Dialog::say()``` 方法可以得到一个```Commune\Chatbot\OOHost\Dialogue\DialogSpeech```对象, 通过它能使用链式调用发送各种预定义的 ```Commune\Chatbot\Blueprint\Message\ReplyMsg```.

例如:

```php
    $dialog
        ->say(['name'=>'张三')           // 定义公共的slots
        ->info('demo.welcomeUser')      // 欢迎用户
        ->info('demo.selfDescription')  // 自我介绍
        ->withReply(new Link('https://communechatbot.com')); // 发送一个链接类型的回复

```

也可以将若干条消息合并成一段话:


```php
    $dialog
        ->say()
        ->info(...)
        ->beginParagraph()  // 在 begin 和 end 之间的信息会被合并成一段.
            ->info(...)
            ->info(...)
            ->info(...)
        ->endParagraph()
        ->info(...)

```

更重要的是可以与用户进行问答式对话:

```php
    $dialog
        ->say(['name'=>'张三')
        ->info('你好, %name%')
        ->askChoose(
            '请问我有什么可以帮您?',
            [
                1 => '功能1',
                2 => '功能2',
                ...
            ]
        )

```

更多问答相关的内容请查看 [问答文档](/docs/dm/questions.md).

## 使用 Hearing 响应用户

我们可以使用```Dialog::hear()```方法, 开启```Commune\Chatbot\OOHost\Dialogue\Hearing``` API 用于定义规则, 理解并回复用户的消息.

```php
    return $dialog
        ->hear()
        ->todo(...)
            ...
        ->todo(...)
            ...
        ->end();

```

关于 Hearing API 的使用方法请查看 [Hearing文档](/docs/dm/hearing.md)

## 使用 Dialog 切换上下文

Dialog 最重要的功能就是用来管理上下文的状态和切换. 多轮对话的每一步推进都依赖 Dialog 的操作.

例如:

```php

public function __onStart(Stage $stage) : Navigator
{
    return $stage->talk(
        function(Dialog $dialog) {
            ...

            // 等待用户回复
            return $dialog->wait();
        },
        function(Dialog $dialog) {
            ...

            // 重复当前对话
            return $dialog->repeat();
        }
    );
}

```

这类方法都会返回一个```Commune\Chatbot\OOHost\Directing\Navigator```对象, 以便```Commune\Chatbot\OOHost\Directing\Director```进行下一轮调度. 所以必须将方法的结果返回给 Stage.

具体的方法请查看源代码. 这里着重介绍几个 :

__Dialog::wait()__ : 得到```Commune\Chatbot\OOHost\Directing\End\Wait``` 对象, 让机器人挂起等待用户的下一轮消息.

__Dialog::missMatch()__ : 得到```Commune\Chatbot\OOHost\Directing\End\MissMatch``` 对象, 明确表示用户的消息没有匹配到任何功能, 是无法理解的消息.

__Dialog::rewind()__ : 对话历史还原到本轮对话, 用户回复之前的状态. 无法消除所有副作用.

__Dialog::backward()__ : 对话历史退回到上一轮对话. 无法消除所有副作用.


## Dialog::$app

使用 ``` $dialog->app``` 可以得到一个```Commune\Chatbot\OOHost\Dialogue\App``` 对象, 这是在上下文中使用的 IoC 容器.

```php
    $instance = $dialog->app->make($abstractName);
```

它本质上仍然是调用请求级容器 ```Commune\Chatbot\Blueprint\Conversation\Conversation```, 但调用 ```$dialog->app->call()``` 方法时, 除了给 callable 对象依赖注入已注册的服务之外, 还可以依赖注入上下文相关的一些实例 (Dialog, Context 等)

## Dialog::$redirect

使用 ```$dialog->redirect``` 可以得到一个```Commune\Chatbot\OOHost\Dialogue\Redirect```对象, 通过它可以直接控制多个 Context 之间的跳转.

建议最好不要用这种方式来控制 Context 跳转, 这样会导致一个 Stage 有非常复杂的状态变化, 不利于开发和维护代码.

仍然推荐用一个独立的 Stage 来管理 Context 的跳转. 例如:

```php

    // 不优雅的做法
    public function __onStart(Stage $stage) : Navigator
    {
        return $stage
            // 要定义所有可能的回调状态和处理逻辑
            ->onStart(...)
            ->onFallback(...)
            ->onIntended(...)
            ->talk(
                ...,
                function(Dialog $dialog){

                    // 将各种状态变更和回调定义在同一个 Stage 内部
                    if (...) {
                        return $dialog->redirect->dependOn(...);
                    } elseif (...) {
                        return $dialog->redirect->sleepTo(...);
                    }
                    ...

                }
            );


    }


    // 好一点的做法
    public function __onHear(Stage $stage) : Navigator
    {
        ...
        // 定义一个独立的 stage
        return $dialog->goStage('task1');
        ...
    }

    // 定义了独立的 Stage
    public function __onTask1(Stage $stage) : Navigator
    {
        return $stage->dependOn(
            'task1',
            function(Dialog $dialog) {
                // 定义了独立的回调逻辑, 更加清晰.
            }
        );
    }


```


## 子对话

Dialog 负责管理一个完整的多轮对话, 对应一个多轮对话上下文进程(```Commune\Chatbot\OOHost\History\Process```).

所有 Dialog 实例都有一个字符串的 ```Dialog::$belongsTo``` 属性. 每一个 Dialog 都实际上从属于这个字符串, 这个字符串可以是 ChatId, SessionId, 甚至从属一个 ContextId. 任何唯一字符串都可以用于创建一个独立的多轮对话.

通过这种方式, 我们可以在一个 Session 中实现多个互相之间不干涉的多轮对话. 每个多轮对话都管理一套独立的上下文 Process. 这样就可以实现多任务嵌套和调度.

举个例子, 如果把用户和机器人对话比喻成浏览器, 浏览器可以打开多个窗口, 就相当于对话中可以创建多个子对话. 浏览器每个窗口里的内容是独立的, 相当于对话中每个子对话是独立的.

在浏览器中, 用户不仅可以在窗口内交互, 还可以在浏览器上点击收藏, 编辑, 查找等功能. 这在一个多轮对话机器人中, 就是两层对话的嵌套, 第一层对话可以拦截一小部分上级意图, 只有它不拦截的意图才会交给子对话去进行.

我们可以用```Commune\Chatbot\OOHost\Dialogue\Dialog::getSubDialog()``` 方法在一个多轮对话上下文中创建另一个独立的多轮对话上下文. 不过更好的做法是使用```Commune\Chatbot\OOHost\Context\Stage::onSubDialog()``` 方法. 在[stage](/docs/dm/stage.md)一节中我们再来讨论.
