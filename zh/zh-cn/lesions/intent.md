# 第七节 : 使用意图

目前为止, 我们定义的多轮对话机器人都是非自然语言的, 仍然可以提供基本的对话交互能力.

要让对话机器人具备自然语言的能力, 最简单的方法就是使用自然语言中间件.
这需要您对自然语言解析有基本的认识.
这里再简单介绍一下.

## NLU 的作用

现在的自然语言解析技术有很多种能力, 在 CommuneChatbot 中主要使用 "语义理解" 和 "实体提取" 两种能力.

NLU (自然语言单元) 能够分析用户输入的自然语言, 例如 "长沙明天天气如何",
判断它符合我们预定义的意图 "demo.cases.tellWeatherInt",
从而可以被处理. 这是 "语义理解" 的作用.

同时, NLU 能够从 "长沙明天如何" 中判断出两个参数```city:长沙``` 和 ```date:明天```, 并告知系统. 这是 "实体提取" 的作用.

结合这两种能力, 我们能将用户自然语言的输入,
转变为以```Commune\Chatbot\OOHost\Context\Intent\IntentMessage```
形式定义的 __通用指令抽象__, 来操作对话机器人.

更多讨论, 可以参考文章 [使用自然语言单元](/zh-cn/core-concepts/nlu.md),
以及 NLU (自然语言单元) 模块的 [使用手册](/zh-cn/nlu/index.md).

## 定义 Intent 类

定义 Intent 类, 可以定义一个对话机器人识别的 __通用指令__.
具体的定义逻辑在 [Intent 文档](/zh-cn/dm/intent.md) 中.

而我们现在只需要知道以下几点 :

1. Intent 类的抽象是 ```Commune\Chatbot\OOHost\Context\Intent\IntentMessage```.
1. Intent 类的基类是 ```Commune\Chatbot\OOHost\Context\Intent\AbsCmdIntent```.
1. Intent 类既可用于 [输入消息](/zh-cn/engineer/messages.md) 的抽象, 也可以作为 Context 多轮对话使用.
1. 所有 IntentMessage 都要定义 ```navigate()``` 方法, 作为命中意图的全局事件.
1. Context 允许根据上下文, 定义自己的意图响应逻辑, 而非使用全局事件.

由于意图的使用形式较为多样灵活, 让我们一次定义四种类型的意图, 然后集中查看效果.

> 相信基于前面的教程, 您已经能够读懂 Context 的源码所表示的意涵. 接下来将不再解释代码细节.

### 定义命令意图

命令意图, 指完成所有参数 (实体) 输入后, 机器人自动执行一个命令, 并返回同步结果.

让我们定义一个命令 ```demo/src/Intents/ActionInt.php``` :

```php
<?php

namespace Commune\Demo\Intents;

use Commune\Chatbot\OOHost\Context\Depending;
use Commune\Chatbot\OOHost\Context\Intent\AbsCmdIntent;
use Commune\Chatbot\OOHost\Context\Stage;
use Commune\Chatbot\OOHost\Dialogue\Dialog;
use Commune\Chatbot\OOHost\Directing\Navigator;

/**
 * 模拟命令型意图.
 *
 * @property string $entity1
 * @property string $entity2
 */
class ActionInt extends AbsCmdIntent
{
    // 定义命中意图的命令行输入
    // 可以用 "#testAction [entity1] [entity2]" 的方式模拟意图.
    const SIGNATURE = 'testAction
    {entity1 : 请输入 Entity1 的值}
    {entity2 : 请输入 Entity2 的值}';

    const DESCRIPTION = '模拟命令型意图';

    // 用注解定义 Entities
    public function __depend(Depending $depending) : void
    {
        // 根据定义的命令, 自动生成 Entity
        $depending->onSignature(static::SIGNATURE);
    }

    public static function getContextName(): string
    {
        return 'demo.lesions.action';
    }

    public function navigate(Dialog $dialog): ? Navigator
    {
        return $dialog->redirect->sleepTo($this);
    }

    public function __onStart(Stage $stage): Navigator
    {
        return $stage
            ->onStart([$this, 'action'])
            ->buildTalk()
            ->fulfill();
    }

    public function action(Dialog $dialog) : void
    {
        $dialog->say()->info(
            '输入信息是 entity1:%entity1%, entity2:%entity2%. ',
            [
                'entity1' => $this->entity1,
                'entity2' => $this->entity2,
            ]
        );
    }
}
```

由于没有使用 NLU (自然语言单元), 我们还无法使用自然语言来命中意图.
但是 CommuneChatbot 自带 [PHP 的意图校验机制](/zh-cn/dm/intent?id=intentmatcher-%e5%8c%b9%e9%85%8d%e8%a7%84%e5%88%99), 允许我们用 命令行/正则/关键字 的手段来匹配意图.

当前的类定义了命令格式:

```
const SIGNATURE = 'testAction
    {entity1 : 请输入 Entity1 的值}
    {entity2 : 请输入 Entity2 的值}';
```

这意味着可以使用 shell 风格的命令```testAction entity1 entity2``` 来命中该意图.
进一步说明了意图作为 __通用命令抽象__, 并不局限于自然语言识别.

### 定义消息类意图

消息类意图, 仅仅用于表征用户的输入信息, 而不具备多轮对话或全局事件能力.

让我们定义一个消息意图 ```demo/src/Intents/MessageInt.php``` 如下:

```php
<?php

namespace Commune\Demo\Intents;

use Commune\Chatbot\OOHost\Context\Intent\AbsCmdIntent;
use Commune\Chatbot\OOHost\Context\Stage;
use Commune\Chatbot\OOHost\Dialogue\Dialog;
use Commune\Chatbot\OOHost\Directing\Navigator;

/**
 * 模拟消息类意图
 */
class MessageInt extends AbsCmdIntent
{
    // 定义命令的匹配逻辑
    const SIGNATURE = 'testMessage';

    const DESCRIPTION = '模拟消息类意图';

    // 定义正则的匹配逻辑
    const REGEX = [
        // 定义正则匹配规则
        ['/^hello|你好$/'],
        // 定义更多正则匹配. 故意定义了有瑕疵的正则, hiiiii 也能匹配到
        ['/^hi/'],
    ];

    public static function getContextName(): string
    {
        return 'demo.lesions.message';
    }

    public function navigate(Dialog $dialog): ? Navigator
    {
        return null;
    }

    public function __onStart(Stage $stage): Navigator
    {
        return $stage->dialog->fulfill();
    }
}
```

注意在这个意图里, 我们定义了 ```正则``` 的 PHP 意图匹配规则.
这意味着负责正则的输入消息, 也会被识别为意图.

### 定义导航类意图

导航类意图的全局事件, 是将对话从当前上下文导入另一个上下文.
有点类似于浏览器的 重定向 url.
除此之外它和消息类意图没有差别.

让我们定义一个导航类意图 ```demo/src/Intents/NavigateInt.php``` :

```php
<?php

namespace Commune\Demo\Intents;

use Commune\Chatbot\OOHost\Context\Intent\AbsCmdIntent;
use Commune\Chatbot\OOHost\Context\Stage;
use Commune\Chatbot\OOHost\Dialogue\Dialog;
use Commune\Chatbot\OOHost\Directing\Navigator;
use Commune\Components\Demo\Cases\Maze\MazeInt;

/**
 * 模拟导航类意图
 */
class NavigateInt extends AbsCmdIntent
{
    const SIGNATURE = 'testNav';

    const DESCRIPTION = '模拟导航类意图';

    // 模拟用关键字来命中意图
    const KEYWORDS = [
        '迷宫',
    ];

    public static function getContextName(): string
    {
        return 'demo.lesions.nav';
    }

    // 直接进入迷宫程序.
    public function navigate(Dialog $dialog): ? Navigator
    {
        return $dialog->redirect->sleepTo(MazeInt::class);
    }

    public function __onStart(Stage $stage): Navigator
    {
        return $stage->dialog->fulfill();
    }
}
```

注意我们这次用了```关键字```来定义 PHP 匹配规则, 意味着输入中存在 "迷宫" 一词, 都可以命中意图.

### 定义任务式意图

任务式意图, 指自身就是一个多轮对话任务. 它的全局事件会自动执行该对话流程.
但是推荐用 ```Stage::sleepTo``` 之类的机制更好地管理.

这种意图有点类似于浏览器拥有独立 Url 的网页.

让我们定义一个任务类意图 ```demo/src/Intents/TaskInt.php``` :

```php
<?php

namespace Commune\Demo\Intents;

use Commune\Chatbot\OOHost\Context\Intent\AbsCmdIntent;
use Commune\Chatbot\OOHost\Context\Stage;
use Commune\Chatbot\OOHost\Dialogue\Dialog;
use Commune\Chatbot\OOHost\Directing\Navigator;

class TaskInt extends AbsCmdIntent
{
    // 定义命令, 用 #testTask 方式可以模拟意图命中
    const SIGNATURE = 'testTask';

    const DESCRIPTION = '模拟任务类意图';

    // 定义意图的名称
    public static function getContextName(): string
    {
        return 'demo.lesions.task';
    }

    // 定义默认的全局事件
    public function navigate(Dialog $dialog): ? Navigator
    {
        return $dialog->redirect->sleepTo($this);
    }

    // 定义任务的流程
    public function __onStart(Stage $stage): Navigator
    {
        // 假装执行了任务, 经过了 1, 2, 3 步
        return $stage->buildTalk()
            ->goStagePipes(['step1', 'step2', 'step3', 'final']);
    }

    public function __onStep1(Stage $stage) : Navigator
    {
        return $this->step($stage);
    }

    public function __onStep2(Stage $stage) : Navigator
    {
        return $this->step($stage);
    }

    public function __onStep3(Stage $stage) : Navigator
    {
        return $this->step($stage);
    }

    public function __onFinal(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ->info('模拟任务执行结束, 退出语境')
            ->fulfill();
    }

    public function step(Stage $stage) : Navigator
    {
        $name = $stage->name;
        return $stage->buildTalk()
            ->info("进入 stage : $name")
            ->info("输入任何信息进入下一步")
            ->wait()
            ->next();
    }
}
```

## 定义测试场景

完成了意图类的定义后, 让我们定义一个测试用例, 来将所有的测试串起来.

让我们创建这个类 ```demo/src/TestIntents.php``` :

```php
<?php

namespace Commune\Demo;

use Commune\Chatbot\App\Callables\Actions\Redirector;
use Commune\Chatbot\App\Callables\Actions\Talker;
use Commune\Chatbot\App\Callables\StageComponents\Menu;
use Commune\Chatbot\OOHost\Context\Intent\IntentMessage;
use Commune\Chatbot\OOHost\Context\OOContext;
use Commune\Chatbot\OOHost\Context\Stage;
use Commune\Chatbot\OOHost\Dialogue\Dialog;
use Commune\Chatbot\OOHost\Dialogue\Hearing;
use Commune\Chatbot\OOHost\Directing\Navigator;

class TestIntents extends OOContext
{
    const DESCRIPTION = '模拟命中意图';

    protected $suggestions = [
        '#testMessage',
        '#testNav',
        '#testTask',
        '#testAction'
    ];

    public function __hearing(Hearing $hearing) : void
    {
        $hearing
            // 输入 "m" 返回菜单
            ->is('m', Redirector::goStage('start'))
            // 输入 "q" 退出
            ->is('q', Redirector::goQuit());
    }

    public function __onStart(Stage $stage): Navigator
    {
        $menu = new Menu(
            '请选择您要进行的测试, 随时输入 m 返回菜单, 输入 q 退出.',
            [

                '模拟匹配单个意图' => 'isIntent',

                '模拟匹配一类意图' => 'isIntentIn',

                '模拟匹配任意意图' => 'isAnyIntent',

                '模拟运行单个意图' => 'runIntent',

                '模拟运行一类意图' => 'runIntentIn',

                '模拟运行任意意图' => 'runAnyIntent',
            ]
        );

        return $stage->component($menu);
    }

    public function __onIsIntent(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ->askVerbal(
                '请测试匹配单个意图. 可用命令来模拟意图. ',
                $this->suggestions
            )
            ->hearing()
            ->isIntent('demo.lesions.message', [$this, 'matchIntent'])
            ->isIntent('demo.lesions.nav', [$this, 'matchIntent'])
            ->isIntent('demo.lesions.task', [$this, 'matchIntent'])
            ->isIntent('demo.lesions.action', [$this, 'matchIntent'])
            ->end();
    }

    public function __onIsIntentIn(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ->askVerbal(
                '请测试匹配一类意图. ',
                $this->suggestions
            )
            ->hearing()
            // 使用意图名的前缀, 作为命名空间
            ->isIntent('demo.lesions', [$this, 'matchIntent'])
            ->end();
    }

    public function __onIsAnyIntent(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ->askVerbal(
                '请测试匹配任意意图. PHP 的匹配规则会失效. 可以用 #intentName# 方式模拟',
                [
                    '#demo.lesions.message#',
                    '#demo.lesions.nav#',
                    '#demo.lesions.task#',
                    '#demo.lesions.action#',
                ]
            )
            ->hearing()
            ->isAnyIntent([$this, 'matchIntent'])
            ->end();
    }

    public function __onRunIntent(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ->askVerbal(
                '测试运行单个意图. 请输入命令表示命中意图. 命中后会直接执行',
                $this->suggestions
            )
            ->hearing()
            ->runIntent('demo.lesions.message')
            ->runIntent('demo.lesions.nav')
            ->runIntent('demo.lesions.task')
            ->runIntent('demo.lesions.action')
            ->end(Talker::say()->info('似乎什么事都没发生'));
    }

    public function __onRunIntentIn(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ->askVerbal(
                '测试运行一类意图. 请输入命令表示命中意图. 命中后会直接执行',
                $this->suggestions
            )
            ->hearing()
            // 使用意图的前缀作为命名空间
            ->runIntentIn(['demo.lesions'])
            ->end(Talker::say()->info('似乎什么事都没发生'));
    }

    public function __onRunAnyIntent(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ->askVerbal(
                '测试运行一类意图. 请输入命令表示命中意图. 命中后会直接执行',
                $this->suggestions
            )
            ->hearing()
            ->runAnyIntent()
            ->end(Talker::say()->info('似乎什么事都没发生'));
    }

    public function matchIntent(IntentMessage $intent, Dialog $dialog) : Navigator
    {
        $dialog->say()->info(
            '命中意图名称: %name%; 简介: %desc%',
            [
                'name' => $intent->getName(),
                'desc' => $intent->getDef()->getDesc()
            ]
        );

        // 重复当前状态.
        return $dialog->repeat();
    }
}
```

然后我们将这个类注册成为一个场景, 打开 ```demo/configs/config.php``` 文件, 添加:

```php

return [
    ...

    'host' => [
        ...

        // 新手教程: 添加机器人, 作为一个启动场景.
        'sceneContextNames' => [

            // test 是场景名, 用类名来标记 Context
            'test' => \Commune\Demo\HelloWorld::class,

            // 一阶多轮对话 教程.
            'firstOrder' => \Commune\Demo\FirstOrderConvo::class,

            // 用 depend 定义的一阶对话
            'userInfo' => 'demo.lesions.user-info',

            // n 阶多轮对话的测试用例.
            'nOrder' => \Commune\Demo\WelcomeUser::class,

            // 测试意图相关
            'intent' => \Commune\Demo\TestIntents::class,
        ],
        ...
    ],

];

```

这样, 我们就可以使用 ```php demo/console.php intent``` 打开这个测试了.

## 测试用例

我们创建的测试类```TestIntents``` 使用了 [Hearing API](/zh-cn/dm/hearing.md) 来定义意图匹配后的逻辑.

总得来说有两种做法, ```isIntent``` 和 ```runIntent```. 前者的响应逻辑, 使用上下文内的定义; 而后者则会执行意图自带的全局事件.

在匹配意图时, 又有三种情况 :

- ```Intent(单个意图)``` : 会调用 PHP 规则匹配指定意图
- ```IntentIn(意图命名空间)``` : 会尝试匹配一类意图, 也会调用 PHP 规则
- ```AnyIntent()``` : 不会调用 PHP 规则, 只会匹配 NLU 提供的最优意图.

因此, 我们有以下几种方式来模拟意图输入:

- 使用命令形式 (要执行 PHP 匹配才能命中)
    - ```#testMessage```
    - ```#testNav```
    - ```#testTask```
    - ```#testAction``` 无参数情况
    - ```#testAction entity1``` 单个参数
    - ```#testAction entity1 entity2``` 匹配所有参数
- 符合正则的输入
    - ```hello```
    - ```你好```
    - ```hi```
    - ```hiiiii``` 匹配正则 ```/^hi/```
- 符合关键字的输入
    - ```迷宫```
    - ```进入迷宫```
    - ```迷宫是什么```
- 模拟 NLU 匹配了意图 (会被中间件```Commune\Chatbot\App\SessionPipe\MarkedIntentPipe``` 处理)
    - ```#demo.lesions.message#```
    - ```#demo.lesions.nav#```
    - ```#demo.lesions.task#```
    - ```#demo.lesions.action#```

相信经过测试, 您能够了解意图的基本使用方法.
更多信息请阅读 [Intent 文档](/zh-cn/dm/intent.md).
如果要配合 NLU, 请阅读 [NLU 文档](/zh-cn/nlu/index.md).


<big>[下一节 : 使用上下文记忆](/zh-cn/lesions/memory.md)</big>