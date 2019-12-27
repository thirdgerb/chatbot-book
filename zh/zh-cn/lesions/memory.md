# 使用上下文记忆

多轮对话很关键的一环是上下文记忆.
Context 本身有属性和 Entity, 可以在上下文中还原,
因此自带语境相关的记忆.
这些记忆往往又是可以读写的数据, 又要通过多轮对话来获取.

我们也常常需要有不同作用域的记忆. 比如用户相关的信息, 理论上只需要询问用户一次,
就应该永远记得.

CommuneChatbot 为实现上下文记忆, 定义了 ```Commune\Chatbot\OOHost\Context\Memory\Memory``` 类 :

1. 它与 Intent 类似, 也是一种 Context, 可通过上下文获得信息
1. 它可以定义独立的作用域, 在同一个作用域维度下的记忆是持久化的
1. 记忆可以单纯作为可存取的对象来使用.

更多信息可以查看 [Memory 文档](/zh-cn/dm/memory.md).


## 定义 Memory 类

我们先定义一个 Memory 类 ```demo/src/memories/TestUserInfoMemory.php``` :

```php
<?php

namespace Commune\Demo\Memories;

use Commune\Chatbot\App\Memories\MemoryDef;
use Commune\Chatbot\OOHost\Session\Scope;

/**
 * 定义上下文, 默认直接通过注解定义 Memory 的 Entity
 *
 * @property string $name 请问如何称呼您
 * @property string $email 请问您的邮箱是?
 */
class TestUserInfoMemory extends MemoryDef
{
    // 定义上下文记忆的名称
    const DESCRIPTION = '模拟的用户信息记忆';

    // 定义上下文记忆的作用域维度
    // 记录的维度会决定上下文的 ID, 没有记录的维度都视作 0
    const SCOPE_TYPES = [
        // 表示用户 ID 维度
        Scope::USER_ID,
        // 表示 机器人维度
        Scope::CHATBOT_NAME,
    ];
}
```

## 定义测试场景

测试场景需要使用三个类, 定义三种 Memory 的用法. 我们先创建文件如下:

定义测试用例 A ```demo/src/TestMemoryA.php``` :

```php
<?php


namespace Commune\Demo;


use Commune\Chatbot\OOHost\Context\Depending;
use Commune\Chatbot\OOHost\Context\OOContext;
use Commune\Chatbot\OOHost\Context\Stage;
use Commune\Chatbot\OOHost\Directing\Navigator;
use Commune\Demo\Memories\TestUserInfoMemory;

/**
 * @property string $name
 */
class TestMemoryA extends OOContext
{
    const DESCRIPTION = '测试用例A, 用 Memory 的单个值作为 Entity';


    public static function __depend(Depending $depending) : void
    {
        // 指定 Memory 对象的一个值作为 Entity
        $depending->onMemoryVal('name', TestUserInfoMemory::class, 'name');
    }

    public function __onStart(Stage $stage): Navigator
    {
        return $stage->buildTalk()
            ->info(
                '您的信息是, name:%name%;',
                [
                    'name' => $this->name,
                ]
            )
            ->fulfill();
    }
}
```

定义测试用例 B ```demo/src/TestMemoryB.php``` :

```php
<?php


namespace Commune\Demo;


use Commune\Chatbot\OOHost\Context\Depending;
use Commune\Chatbot\OOHost\Context\OOContext;
use Commune\Chatbot\OOHost\Context\Stage;
use Commune\Chatbot\OOHost\Directing\Navigator;
use Commune\Demo\Memories\TestUserInfoMemory;

/**
 * @property TestUserInfoMemory $user
 */
class TestMemoryB extends OOContext
{
    const DESCRIPTION = '测试用例B, 用 Memory 作为整个 Entity';

    public static function __depend(Depending $depending) : void
    {
        // 定义整个 Memory 对象作为一个 Entity
        $depending->onMemory('user', TestUserInfoMemory::class);
    }

    public function __onStart(Stage $stage): Navigator
    {
        return $stage->buildTalk()
            ->info(
                '您的信息是, name:%name%; email:%email%;',
                [
                    'name' => $this->user->name,
                    'email' => $this->user->email,
                ]
            )
            ->fulfill();
    }

}
```

然后定义测试语境 ```demo/src/TestMemory.php``` :

```php
<?php

namespace Commune\Demo;

use Commune\Chatbot\App\Callables\StageComponents\Menu;
use Commune\Chatbot\OOHost\Context\OOContext;
use Commune\Chatbot\OOHost\Context\Stage;
use Commune\Chatbot\OOHost\Directing\Navigator;
use Commune\Demo\Memories\TestUserInfoMemory;

/**
 * @property-read TestUserInfoMemory $user
 */
class TestMemory extends OOContext
{
    const DESCRIPTION = '测试记忆';

    // 定义一个缓存, 避免频繁获取
    // 这个值在 Context 序列化时不会被存储.
    protected $_user;

    public function __onStart(Stage $stage): Navigator
    {
        $menu = new Menu(
            '请选择测试用例',
            [
                TestMemoryA::class,
                TestMemoryB::class,
                '测试用 getter 查看Memory' => 'getter',
            ]
        );

        return $stage->component($menu);
    }

    public function __onGetter(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ->info(
                '当前用户的信息为, name:%name%; email:%email%',
                [
                    'name' => $this->user->name,
                    'email' => $this->user->email,
                ]
            )
            ->goStage('start');
    }


    public function __getUser() : TestUserInfoMemory
    {
        return $this->_user ?? $this->_user = TestUserInfoMemory::from($this);
    }
}
```

这样, 我们就可以使用 ```php demo/console.php memory``` 打开这个测试了.

## 运行测试用例

阅读源码可以注意到, 测试用例 A 和 B 用了两种不同的方式来定义 Memory 相关的 Entity.
由于记忆在上下文中共享, 因此只要通过对话录入过一次的信息, 就不需要再次录入.

因此测试用例中的三项, 用不同的路径遍历, 会有不一样的结果. 推荐运行 ```php demo/console.php memory``` 将各种顺序都执行依次, 查看效果 :

- 1 => 2 => 3
- 2 => 1 => 3
- 3 => 1 => 2 => 3
- 3 => 2 => 1 => 3

更多定义 Memory 的做法, 请查看 [Memory 文档](/zh-cn/dm/memory.md).

