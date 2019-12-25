# 第二课 : 定义单轮对话

上一课我们定义了一个只会说 "hello world!" 的机器人, 这一课我们给它定义一些基本的回复规则.


##  用 Stage::talk() 定义单轮对话

单轮对话的节点, 在 CommuneChatbot 中称之为 "stage" (阶段) .

上一课的 ```public function __onStart(Stage $stage): Navigator``` 就定义了一个名为 ```start``` 的 stage.

通常机器人进入一个 ```stage``` 会有主动的表达, 而 "听到" 用户回复时会有一个被动的回复.

我们可以用 ```Stage::talk``` 方式区分主动表达和被动回复.

修改 ```__onStart ``` 方法如下 :

```php

    /**
     * 定义上下文启动
     * @param Stage $stage
     * @return Navigator
     */
    public function __onStart(Stage $stage): Navigator
    {
        return $stage->talk(
            // 主动表表达
            function(Dialog $dialog) : Navigator {
                // 主动说话
                $dialog->say()->info("请问您有什么需求?");

                // 等待用户回复.
                return $dialog->wait();
            },
            // 被动回复
            function(Message $message, Dialog $dialog) : Navigator {

                $dialog->say()->warning(

                    // 回复消息的模板, 用 %text% 表示变量.
                    // 底层使用了 Symfony Translator
                    '您的回复是 %text% ',

                    // 定义回复模板所用的变量
                    [
                        'text' => $message->getText()
                    ]
                );

                // 重复当前 stage
                return $dialog->repeat();
            }
        );
    }
```


接下来执行 ``` php demo/console.php test``` 可以看到效果.


## 活用 callable 让代码更清晰


```Stage::talk()``` 方法接受两个 ```callable``` 对象, 这是一种偏向函数式编程的做法. 查看源代码可以看到更多提示.

PHP 的 ```callable``` 机制可以用各种形式定义一个可传递的函数, 包括:

*   方法名的字符串, 例如 ``` 'is_numeric' ```
*   闭包, 例如 ``` function() {} ```
*   实例的方法, 例如 ``` [$object, 'methodName'] ```
*   静态方法, 例如 ``` [ 'ClassName', 'staticMethodName'] ```
*   拥有 ```__invoke()``` 方法的对象.

所以我们的 ```__onStart``` 可以用以下方式拆分地更清晰:


```php
    /**
     * 用 talk 定义 一个单轮对话, 并且用 callable 来拆分.
     * @param Stage $stage
     * @return Navigator
     */
    public function __onStart(Stage $stage): Navigator
    {
        return $stage->talk(
            [$this, 'talkToUser'],
            [$this, 'hearFromUser']
        );
    }

    public function talkToUser(Dialog $dialog) : Navigator
    {
        // 主动说话
        $dialog->say()->info("请问您有什么需求?");

        // 等待用户回复.
        return $dialog->wait();
    }

    public function hearFromUser(Message $message, Dialog $dialog) : Navigator
    {
        $dialog->say()->warning(

            // 回复消息的模板, 用 %text% 表示变量.
            // 底层使用了 Symfony Translator
            '您的回复是 %text% ',

            // 定义回复模板所用的变量
            [
                'text' => $message->getText()
            ]
        );

        // 重复当前 stage
        return $dialog->repeat();
    }
```


然后执行 ```php demo/console.php test``` 查看效果

## 使用 Hearing API

```Dialog``` 对象相当于 js 开发浏览器应用时用到的 ```window``` 对象, 它管理了定义对话所需的各种模块.

CommuneChatbot 项目提供了一个 ```Hearing``` api, 可以方便我们使用各种规则.

让我们用 ```Hearing``` 重构 ``` hearFromUser()``` 方法, 并给它添加几个额外的功能.

```php
    /**
     * Message 与 Dialog 都通过依赖注入获取
     *
     * @param Message $message
     * @param Dialog $dialog
     * @return Navigator
     */
    public function hearFromUser(Message $message, Dialog $dialog) : Navigator
    {
        return $dialog
            // dialog 听到了输入的消息
            ->hear($message)

            // 精准匹配 hello
            ->is('hello', function(Dialog $dialog) {
                $dialog->say()->info('hello world!');
                // 重复对话
                return $dialog->repeat();
            })

            // php 正则匹配
            ->pregMatch('/test/', [], function(Dialog $dialog) {
                $dialog->say()->info("命中了 /test/ 正则");
                return $dialog->repeat();
            })

            // php 关键字匹配
            // 使用 二维数组作为关键字, 第一维是与的关系, 第二维是或的关系.
           ->hasKeywords(['你', '是', ['谁', '什么']], function(Dialog $dialog){
               $dialog->say()->info('我是 hello world 机器人');
               return $dialog->repeat();
            })

            // 拒答的逻辑, 当上述流程没有任何返回时, 会执行 miss match 事件.
            // 用户会收到一个默认回复.
            ->end();
    }
```

然后运行 ```php demo/console.php test```, 并且测试以下几种输入:

* hello
* test
* test 123
* 你是谁
* 你是什么
* 肯定无法命中的句子

Hearing API 整合了和 NLU (自然语言处理单元) 相关的各种功能. 关于 Hearing api 的更多细节, 请查看 interface 文件 ```Commune\Chatbot\OOHost\Dialogue\Hearing```


## 依赖注入

也许您注意到了, 在 ```Stage::talk``` 和 ```Dialog::hear``` 中传入的各种 ```callable``` 对象, 参数是不一样的. 有的有 ```Message $message```, 有的没有.

这是因为所有传入的```callable``` 对象都进行了依赖注入, 可以按照您的需求, 用变量名的方式, 或者用类型约束 (typehint) 的方式, 注入各种依赖, 包括:

* IoC 容器中已注册的依赖.
* 对话过程中产生的 ```Message $message```, ```Dialog $dialog``` 和 ```Context $self```

而且```$message``` 和 ```$self``` (当前 context 自身的实例) 都允许用多种形式进行依赖注入, 包括:

* 自己的类名
* 父类中的抽象类 (abstract class) 类名
* 父类中的接口类 (interface) 类名


如果定义了参数名```$dependencies```, 您将获得上下文相关的可注入依赖. 我们可以将这个功能做到上文的对话中:

```php


        // 查看上下文中可用的依赖注入对象
        // 不包括 IoC 容器中定义的对象.
        ->is('look di', function (Dialog $dialog, array $dependencies) {
            $dialog->say()
                ->info('当前的依赖有: ')
                ->info(json_encode($dependencies, JSON_PRETTY_PRINT));

            return $dialog->repeat();
        })

        ->end();
```

然后运行 ```php demo/console.php test``` 并且输入 ```look di``` 查看效果.


## [下一课: 定义一阶多轮对话](/zh-cn/lesions/first-order-convo.md)