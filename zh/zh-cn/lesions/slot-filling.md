# 第四节 : 填槽型多轮对话

现在主流的多轮对话机器人,
在实现一阶多轮对话时采取的是 "填槽" (slot-filling) 的模式,
又称之为 "任务型多轮对话".

简单而言, 机器人执行一个任务 (比如查询天气) 时需要输入若干参数,
缺失的参数就称之为 "槽位" (slot).
如果是在浏览器上, 通常会做成一个表单, 一次提交并执行;
而对话则不同, 需要反复引导用户输入.

CommuneChatbot 的多轮对话设计, 是面向状态的, 而不是面向任务的.
所以不存在"槽位" 的设计.
然而也可以用 ```Context``` (上下文语境) 的 ```Stage``` (单轮对话) 的方式模拟 "填槽",
为每一个 "槽位" 提供一个独立的 ```Stage```, 通过可循环的单轮对话来完成参数的引导, 校验与退出.

大多数填槽型对话, 获得所有参数之后一次性执行任务, 就结束了多轮对话.
CommuneChatbot 执行任务可以有若干个环节,
执行过程中仍然可以反复和用户互动.
"填槽" (slot-filling) 的终点, 通常对应 ```Context``` 的起点 (```__onStart```).

## 创建 UserInfo 测试用例

我们先创建一个新的测试用例:

    touch BASE_PATH/demo/src/UserInfo.php

编辑该文件, 写入:

```php
<?php

namespace Commune\Demo;

use Commune\Chatbot\OOHost\Context\Depending;
use Commune\Chatbot\OOHost\Context\Exiting;
use Commune\Chatbot\OOHost\Context\OOContext;
use Commune\Chatbot\OOHost\Context\Stage;
use Commune\Chatbot\OOHost\Directing\Navigator;
use Commune\Chatbot\OOHost\Dialogue\Hearing;

class UserInfo extends OOContext
{

    // 注册这个常量, 可以使语境获得简介. $context->getDef()->getDesc();
    const DESCRIPTION = 'N 阶多轮对话教学用例';

    /**
     * 定义语境的名称, 不是用 命名空间 转义, 而是直接定义字符串.
     * @return string
     */
    public static function getContextName(): string
    {
        // 注意, 仅仅支持 小写字母, 数字, '.' 和 '_' 几种符号.
        // 正则是 /[a-z0-9\-\.]+/
        return 'demo.lesions.user-info';
    }

    public function __onStart(Stage $stage): Navigator
    {
        return $stage->buildTalk()
            ->info('待会再写内容')
            ->fulfill();
    }
}
```

注意, 我们这次定义了 ```Context::DESCRIPTION``` 常量, 作为语境的简介; 同时重写了 ```Context::getContextName()``` 方法, 以定义我们自己想要的语境名称.

## 将 Context 主动注册到系统

[Context](/zh-cn/dm/context.md) (上下文语境) 不会自动注册到系统中(见 [Registrar](/zh-cn/dm/registrar.md)).
所以我们需要先告诉系统, 要主动扫描该目录下的```Context``` 进行注册.

这需要先修改机器人配置文件 ```BASE_PATH/configs/config.php```.

```php

return [
    ...

    'host' => [

        // 在这里用 PSR-4 规范定义出系统启动时要注册Context的路径
        'autoloadPsr4' => [
            "Commune\\Demo\\" => __DIR__ .'/../src/'
        ],

       'rootContextName' => \Commune\Components\Demo\Contexts\DemoHome::class,

        // 新手教程: 添加机器人, 作为一个启动场景.
        'sceneContextNames' => [

            // test 是场景名, 用类名来标记 Context
            'test' => \Commune\Demo\HelloWorld::class,

            // 一阶多轮对话 教程.
            'firstOrder' => \Commune\Demo\FirstOrderConvo::class,

            // 用 depend 定义的一阶对话
            'userInfo' => 'demo.lesions.user-info',
        ],

        ...

    ],
];
```

然后执行 ```php demo/console.php userInfo``` 查看效果, 看看注册是否成功.

## 用 __depend 方法快速定义一阶多轮对话.

接下来, 我们用一个更快捷的方法定义出一个简单的多轮对话.

第一步, 修改原来的 ```Context::__depend``` 方法, 注册依赖的实体

```php

    /**
     * 注册依赖的实体. 只有这些实体有和合法值了, 才能进入 start stage
     * @param Depending $depending
     */
    public static function __depend(Depending $depending): void
    {
        $depending
            ->on('name', '请问我该怎么称呼您?')
            ->on('email', '请问您的邮箱是?');
    }
```

这样, ```UserInfo``` 就多了两个必须回答的值. 为了让 IDE 能够更好地支持, 我们在类名处写上注解:

```php
    /**
     * @property string $name
     * @property string $email
     */
    class UserInfo extends OOContext
```

然后我们修改 ```UserInfo::__onStart``` 方法来查看结果 :

```php
    public function __onStart(Stage $stage): Navigator
    {
        return $stage->buildTalk([
                // Entity "name" 可以通过 $this->name 的方式获取
                'name' => $this->name,
                'email' => $this->email
            ])
            ->info('您的名字是 %name%, 邮箱是 %email%. ')
            ->fulfill();
    }
```

从代码中可以看到, 所有的 ```Entity``` 参数都可以通过 ```$context->{$entityName} ``` 的方式获取到.

```Context::__depend``` 方法用来定义一个语境必须先通过多轮对话获取的参数. CommuneChatbot 称这些参数为 ```Entity``` . 只有这些参数都得到了, 用户才能进入一个多轮对话的 ```start``` stage.

我们再运行 ```php demo/console.php userInfo``` 查看效果.


## 用注解的方式定义 __depend

```public static function __depend(Depending $depending): void``` 方法, 通过```Commune\Chatbot\OOHost\Context\Depending``` 实例有多种定义 ```Entity``` 的办法.

这里我们演示简单的注解方式:

修改```UserInfo```的注解为:

```php
/**
 * @property string $name 请问我该怎么称呼您?
 * @property string $email 请问您的邮箱是?
 */
class UserInfo extends OOContext
```

修改 __depend 方法为:

```php
    /**
     * 注册依赖的实体. 只有这些实体有和合法值了, 才能进入 start stage
     * @param Depending $depending
     */
    public static function __depend(Depending $depending): void
    {
        // 使用注解中的 @property 来定义 entity
        $depending->onAnnotations();
    }
```

运行 ``` php demo/console.php userInfo ``` 查看效果.

## 为 Entity 定义独立的 Stage

我们注意到, ```Context::__depend``` 方法并没有对 ```Entity```进行校验.
如果需要校验的话, 还是需要定义一个独立的 ```Stage``` 最为合理;
可以提供独立的引导, 校验, 容错, 拒答和退出机制.

这里我们拿 Email 参数做例子, 只需要定义与```Entity``` 同名的 ```Stage```, 就会覆盖掉默认的```Stage``` :

```php

    /**
     * 主动定义一个 stage 来覆盖掉 entity 默认的单轮对话
     * @param Stage $stage
     * @return Navigator
     */
    public function __onEmail(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            // 没有suggestions, 允许任何输入
            ->askVerbose('请问您的邮箱是?')
            ->hearing()

            ->todo(function(Dialog $dialog, Message $message){
                // 进行赋值, 直接赋值给 entity
                $this->email = $message->getTrimmedText();

                // 进行到下一步
                return $dialog->next();
            })
                ->pregMatch('/[\w\.]+@[\w\.]+/')

            ->end(function(Dialog $dialog){
                $dialog->say()->warning('邮箱格式似乎不正确');
                // 重新询问
                return $dialog->repeat();
            });

    }
```

运行 ``` php demo/console.php userInfo ``` 查看效果.

值得注意的是, 这个例子中我们首次使用了 ```todo``` api. 这是一种工程上的思路, ```能力的抽象优先于输入的抽象```. 我们先给机器人定义它的能力, 再定义每一个能力的触发条件; 而不是先遍历输入的每种可能性, 再给它们一个一个填写响应逻辑.

这样从代码上看, 可以一目了然的知道机器人能做哪些事情, 也符合设计机器人时的思路, 还有更好的扩展性. 后面我们会越来越多看到这样的例子.

<big>[下一节 : 依赖关系的 N 阶多轮对话](/zh-cn/lesions/n-order-convo.md)</big>

