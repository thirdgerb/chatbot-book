# 第四课 : 定义 N 阶多轮对话

关于复杂多轮对话问题, CommuneChatbot 项目有自己的一套解释, 您可以通过 [这篇文章](/docs/core-concepts/complex-conversation.md) 了解.

简单而言, 当一个多轮对话 A 的某个节点, 嵌套了另一个多轮对话 B , 这就构成了一个 二阶多轮对话. 而 B 可以再嵌套 C, C 可以再嵌套 D ... 从而构成了 N 阶多轮对话.

由于每一个节点都可以有分叉, 走向不同的 1阶多轮对话, 因此整个多轮对话看起来是 ```树状```的.

但实际上, 子级多轮对话的结果, 又可以反过来影响父级多轮对话的走向, 因此整个多轮对话是 ```图结构```的.

接下来我们了解一下 CommuneChatbot 如何实现这种```图结构``` 的多轮对话吧.


## 定义获取用户信息的 UserInfo 对话

### 创建 UserInfo 测试用例

我们先创建一个新的测试用例:

    touch demo/src/UserInfo.php

编辑该文件, 写入:

```php
<?php

namespace Commune\Demo;

use Commune\Chatbot\OOHost\Context\Depending;
use Commune\Chatbot\OOHost\Context\Exiting;
use Commune\Chatbot\OOHost\Context\OOContext;
use Commune\Chatbot\OOHost\Context\Stage;
use Commune\Chatbot\OOHost\Directing\Navigator;

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

    public function __exiting(Exiting $listener): void
    {
    }

    public static function __depend(Depending $depending): void
    {
    }

    public function __onStart(Stage $stage): Navigator
    {
        return $stage->buildTalk()
            ->info('待会再写内容')
            ->fulfill();
    }
}
```

注意, 我们这次定义了 ```Context::DESCRIPTION``` 常量, 作为语境的简介; 同时重写了 ```Context::getContextName()``` 方法, 以定义我们自己想要的语境名称. (默认值是将类的明明空间, 映射为用 '.' 取代 '\\' 的字符串)


不是和命名空间一致的```语境```, 不会自动注册到系统中. 所以我们需要先告诉机器人系统, 要主动加载该目录下的语境:

```php
    // 修改文件 BASE_PATH/configs/config.php 中的代码

    'host' => [

        // 在这里用 PSR-4 规范定义出系统启动时要自动加载的代码路径
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

            // 注册 n阶多轮对话 教程.
            'nOrder' => 'demo.lesions.user-info',
        ],
```


然后执行 ```php demo/console.php nOrder``` 确保注册成功.



### 用 __depend 方法快速定义一阶多轮对话.

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

        // $depending->onAnnotations();
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
                'name' => $this->name,
                'email' => $this->email
            ])
            ->info('您的名字是 %name%, 邮箱是 %email%. ')
            ->fulfill();
    }
```

运行 ``` php demo/console.php nOrder ``` 查看效果. 


```Context::__depend``` 方法用来定义一个语境必须先通过多轮对话获取的参数. CommuneChatbot 称这些参数为 ```Entity``` . 只有这些参数都得到了, 用户才能进入一个多轮对话的 ```start``` stage.


### 用注解的方式定义 __depend

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

运行 ``` php demo/console.php nOrder ``` 查看效果.

### 为 Entity 定义独立的 Stage

我们注意到, ```Context::__depend``` 方法并没有对 ```Entity```进行校验. 如果需要校验的话, 还是需要定义一个stage.



