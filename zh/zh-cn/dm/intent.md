# Intent

意图 (intent) 是机器人理解用户输入的最基本抽象. 无论用户输入的是语音, 文字, 图片, 表情, 链接, 手势... 都可以抽象为 "意图".

在 CommuneChatbot 项目中, 用 "intentName" + "entities" ( 意图名 + 实体值 ) 来描述单一意图. 本质上类似于```方法名 + 参数```, 是人类对机器人下达指令的最小完整单元.

与浏览器的表单, shell 里的命令行不同, 作为意图参数的 "entities" 往往需要经过多轮对话来完善. 所以意图被封装成为```Commune\Chatbot\OOHost\Context\Intent\IntentMessage```对象, 既是一个输入消息 ```Commune\Chatbot\Blueprint\Message\Message```对象, 又是一个上下文单元```Commune\Chatbot\OOHost\Context\Context```对象.

系统会对输入的消息进行分析, 获得意图信息, 并在需要的时候将意图信息封装成```Commune\Chatbot\OOHost\Context\Intent\IntentMessage```对象供使用.

## 1. 意图的基本功能

在不同的上下文场景中, 意图可能有三种类型的用法:

- 作为匹配规则
- 作为全局事件
- 作为上下文入口

### 作为匹配规则

我们在语境中需要对用户的意图进行响应, 例如用户说 "能不能逗我开心一下", 机器人就讲一个笑话.
用户的输入消息可能被匹配为```tellJoke``` 这样的意图名,
然后系统通过 Hearing 定义回复规则 :

```php
$hearing->isIntent('tellJoke', TellJokeAction::class)
```

这时意图被视作一系列匹配规则的集合. 而具体的响应逻辑跟上下文有关.

### 作为全局事件

有些意图, 可能在任何一个上下文中都有相同的意思. 例如用户说 "退出", 表示退出机器人的所有对话.

这样的逻辑不必在每一个 Context 中重复定义, 我们将之放在了 ```IntentMessage::navigate($dialog)``` 方法中.
每个意图都需要定义自己的全局响应逻辑.

这时我们可以通过 Hearing 像触发事件一样来响应意图 :

```php

    $hearing
        // 直接运行单个意图的 navigate 方法
        ->runIntent(IntentClassName::class)

        // 用命名空间尝试匹配一类意图的任意一个, 然后执行 navigate 方法
        ->runIntentIn('demo.maze')

        // 如果最高优先级意图, $session->getMatchedIntent() 存在,
        // 则直接运行它的 navigate
        ->runAnyIntent()
        ...
        ->end()

```

### 作为上下文入口

由于意图本身是一个```Commune\Chatbot\OOHost\Context\Context```对象,
它可以像其它 Context 一样在多轮对话中自由调度 :

```php

    public function __onMaze(Stage $stage) : Navigator
    {
        return $stage->dependOn(
            // 迷宫小游戏入口, 既可以作为意图匹配, 也可以多轮对话中前往
            MazeIntent::class,
            ...
        );
    }

```

这种类型的意图, 如果比较网站的话, 有点类似一个网页的 URL, 在上下文允许的时候可以直达功能.

## 2. 意图的名称

每一种意图都有一个全局唯一的字符串ID 用来标记. 它的命名规则与 [Context](/zh-cn/dm/context.md) 保持一致,
由```小写字母```, ```-```, ```.``` 三种符号构成.

> 意图统一用大写或小写字母命名, 是为防止由于大写小不一致导致匹配出错, 又难于排查的问题

意图的命名应该遵循 "命名空间" 的思路, 用 ```.``` 符号隔开命名空间的每一段, 例如```demo.games.maze``` 这种.

由于各种 NLU 对意图命名的规范不一致 (有的只允许大写字母, 有的不能用```.```而必须用下划线), 在实际使用 NLU 过程中往往要对匹配所得的意图名进行转义.

可以通过 ```Commune\Support\Utils\StringUtils::normalizeContextName()``` 方法对非标准字符串进行统一处理, 使之可比较;
或者用 ```Commune\Support\Utils\StringUtils::validateDefName()``` 进行校验.

定义意图名的方式在后文会有介绍. 而获取意图名称和 Context 一致, 可通过 ```$intentMessage->getName()``` 或者 ```$intentMessage->getDef()->getName()``` 来获取.


## 3. 意图信息来源

IntentMessage 通常是对用户输入信息的抽象. 这些抽象信息, 根据工程方案的区别, 有四种不同的来源:

- IncomingMessage
- Request
- NLUService
- IntentMatcher

__IncomingMessage__ : 通过 ```Commune\Chatbot\Blueprint\Conversation\MessageRequest::fetchMessage()``` 方法获得的输入消息, 可以本身就是一个 IntentMessage.
这是由于多个 CommuneChatbot 机器人可以相互连接, 组成一个管道.
上一个机器人已经解析好 IntentMessage, 可以通过接口直接传递给下一个机器人.

__Request__ : 有些客户端, 例如智能音箱(DuerOS), 在发来用户的消息请求时已经携带了意图信息.
这些意图信息可以通过```Commune\Chatbot\Blueprint\Conversation\MessageRequest::fetchNLU()```方法, 封装到 ```Conversation::getNLU()``` 的 ```Commune\Chatbot\Blueprint\Conversation\NLU``` 对象中.
可以在后续的处理逻辑中, 提取为 IntentMessage 对象.

__NLUService__ : 更常见的情况, 是用户一般消息通过 ChatbotPipe 组成的管道时,
调用了一个第三方的自然语言单元 (例如 Rasa, 百度UNIT ) 服务.
该服务被封装为```Commune\Chatbot\OOHost\NLU\Contracts\NLUService``` 对象.
服务返回了意图识别的信息, 保存到```Commune\Chatbot\Blueprint\Conversation\NLU``` 对象中, 供后续处理逻辑调用.

__IntentMatcher__ : 没有任何意图信息时候, 仍然可以通过 CommuneChatbot 项目自带的 PHP 意图匹配工具 ```Commune\Chatbot\OOHost\Context\Intent\IntentMatcher```, 来提取 IntentMessage 对象.

这四种数据的优先级如先后顺序所示.

## 4. 获取意图实例

在 Stage 中想要获取意图实例, 最基本的方法是通过 Session :

```php
    /**
     * @var Commune\Chatbot\OOHost\Session\Session
     * @var string $intentName
     * @var null|Commune\Chatbot\OOHost\Context\Intent\IntentMessage $intentMessage
     */

    // 尝试获取优先级最高的意图, 如没有, 返回 null
    $intentMessage = $session->getMatchedIntent();

    // 尝试获取指定的意图, 如没有返回 null
    $intentMessage = $session->getPossibleIntent($intentName);

```

更常见的方式是通过 [Hearing API](/zh-cn/dm/hearing.md) :

```php

    public function __onSomeStage(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ...
            ->hearing()
            ...
            // 尝试匹配单个意图
            ->isIntent(
                // 允许用类名作为意图名称
                SomeIntentClassName::class,
                // 匹配成功后, 意图的类名可以用于依赖注入
                function(SomeIntentClassName $message, Dialog $dialog) {
                    ...
                }
            )
            ...
            ->end();
    }

```

更多的方法请查看 [Hearing API 相关章节](/zh-cn/dm/hearing.md).


## 5. 意图匹配规则

### NLU 匹配规则

匹配意图最基本的方式是通过第三方的自然语言单元, 例如百度 UNIT 这样的云服务, 或类似 Rasa 这样的开源项目. 匹配到的数据会记录在```Commune\Chatbot\Blueprint\Conversation\NLU``` 对象中.

NLU 通常会返回多种可能的意图, 每个意图都有一个置信度 (通常是小于 1 的浮点数).
我们可以通过设置```阈值```筛选高置信度的意图,
置信度最高的意图, 可以通过```$session->getMatchedIntent()``` 获取.
其它的高可信意图, 可以通过```$session->getPossibleIntent($intentName)``` 获取.

在调用 Hearing API 用规则从多个意图中进行匹配时, 不会参考置信度, 而是按先后顺序.

```php

    return $hearing
        // 以下匹配按代码的先后顺序
        // 然而代码本身可以再通过 api 来生成.
        // 不应该认为匹配顺序只能写死
        ->isIntent('demo.intent.a', $actionA)
        ->isIntent('demo.intent.b', $actionB)
        ...
        ->isIntent('demo.intent.n', $actionN)
        ->end();
```

如果想通过算法来排列优先级, 同时考虑上下文和置信度, 则被看作一种 "决策模块", 系统目前没有内置解决方案. 可以通过对 Hearing 调用进行动态编程来解决.

NLU 匹配到的 Entities 通常被赋予 IntentMessage 对象, 被认为是意图相关的实体. 但某些场景下, 提取的 Entity 可能和 Intent 是相互分离的.

这时可以针对 Entity 定义独立的处理规则, 例如

```php

    return $hearing
        ...
        ->matcheEntity($entityName, $action)
        ...
        ->end();
```

有些意图需要通过多轮对话来补完 Entity 参数, 然后才执行一个上下文相关的任务. 这种情况推荐在上下文中主动进行判断:

```php

    public function __onStart(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ...
            ->hearing()
            ...
            ->isIntent(
                IntentWithEntities::class,
                function(IntentWithEntities $intent, Dialog $dialog) {

                    // 如果 entity 参数已完备
                    if ($intent->isPrepared()) {
                        return $this->handle($intent, $dialog);
                    }

                    // 不完备的话, 进入一个 depend stage

                    // 先赋值到属性中, 方便跨 stage 调用.
                    $this->tempIntentWithEntity = $intent;
                    return $dialog->goStage('fulfillIntent');
                }
            ...
            ->end();
    }


    /**
     * 匹配到意图的处理逻辑
     */
    public function handleIntentWithEntities(IntentWithEntities $intent, Dialog $dialog) : Navigator
    {
        ...
    }

    /**
     * 用一个独立的 Stage 来管理该流程,
     * 以防止 stage:start 需要处理各种可能的回调状态
     */
    public function __onFulfillIntent(Stage $stage) : Navigator
    {
        return $stage->dependOn(
            $this->tempIntentWithEntity,
            [$this, 'handleIntentWithEntities']
        );
    }

```

### IntentMatcher 匹配规则

当系统尝试调用 ```$session->getPossibleIntent($intentName)```, 或通过 Hearing 执行 ``` $hearing->isIntent($intentName, ...)``` 时, 会优先从 NLU 中查找是否匹配到该意图.

如果 NLU 中没有, 系统仍然会尝试通过根据 IntentMessage 定义生成的 ```Commune\Chatbot\OOHost\Context\Intent\IntentMatcher``` 对象, 进行 PHP 规则的匹配.

获取 IntentMatcher 的路径是 ``` $intentMessage->getDef()->getMatcher()```, 具体请查看 ```Commune\Chatbot\OOHost\Context\Intent\IntentDefinition```.

自带的匹配规则目前有三种 :

- 命令格式匹配
- 正则匹配
- 关键字匹配

定义这三类规则的位置在后文有介绍.
这些规则都会存储到一个 ```Commune\Chatbot\OOHost\Context\Intent\IntentMatcherOption``` 对象中.
具体的规则请查看该类的注释.
只要保证可以通过 ```$intentMessage->getDef()->getMatcherOption()```获取到即可.

CommuneChatbot 并不鼓励用代码编写规则, 代替自然语言单元.
但在一些特殊场景下, 可以通过规则达到百分之百置信度, 规则就比自然语言单元更好用, 例如:

- 命令格式匹配: 适用于无 NLU 调试. 避免没有 NLU 就无法开发多轮对话
- 正则匹配: 适用于高度规范, 或简洁的短语场景, 例如单字回复 "嗯", "好" 等.

这些规则在未来可能会增加, 例如根据 Entity 倒过来匹配意图等.

> 仍然再次强调: 意图最好都定义命令规则.
> 因为 NLU 要训练到位需要持续生产语料, 反而让调试多轮对话变得不方便.


## 6. 定义 Intent

CommuneChatbot 默认用类的方式来定义 Intent. 可以使用的基类如下:

- 接口: ```Commune\Chatbot\OOHost\Context\Intent\IntentMessage```
- 基类: ```Commune\Chatbot\OOHost\Context\Intent\AbsIntent```
    - 基础意图类: ```Commune\Chatbot\OOHost\Context\Intent\AbsCmdIntent```
        - 消息意图: ```Commune\Chatbot\App\Intents\MessageIntent```
        - 功能意图: ```Commune\Chatbot\App\Intents\ActionIntent```
        - 导航意图: ```Commune\Chatbot\App\Intents\NavigateIntent```
        - 任务意图: ```Commune\Chatbot\App\Intents\TaskIntent```

从接口类 ```Commune\Chatbot\OOHost\Context\Intent\IntentMessage``` 可以看到,
IntentMessage 比普通的 Context 多了一个 ```IntentMessage::navigate($dialog)``` 方法.

### 基础意图类

定义标准的意图, 可以使用 ```Commune\Chatbot\OOHost\Context\Intent\AbsCmdIntent``` 抽象类:

```php

use Commune\Chatbot\OOHost\Context\Intent\AbsCmdIntent;
use Commune\Chatbot\OOHost\Context\Intent\IntentMatcherOption;

class MyIntent extends AbsCmdIntent
{
    // 定义意图的简介, 通过 $myIntent->getDef()->getDesc() 获取
    const DESCRIPTION = 'description of my intent';

    /**
     * 命令名. 可以用命令的方式来匹配
     * 写法 @see IntentMatcherOption
     */
    const SIGNATURE = '';

    /**
     * 用正则来匹配
     * 写法 @see IntentMatcherOption
     */
    const REGEX = [];

    /**
     * 用关键字来匹配
     * 写法 @see IntentMatcherOption
     */
    const KEYWORDS = [];

    /**
     * 可选方法
     *
     * 定义 PHP 匹配规则的地方.
     * 重写这个方法, 可以真正修改所有自定义匹配规则
     *
     * @return IntentMatcherOption
     */
    public static function getMatcherOption(): IntentMatcherOption
    {
        return new IntentMatcherOption([
            'signature' => static::SIGNATURE,
            'regex' => static::REGEX,
            'keywords' => static::KEYWORDS,
        ]);
    }


    /**
     * 可选方法
     *
     * 重写这个方法, 可以自定义意图名称字符串. 默认用类名来映射.
     */
    public static function getContextName() : string
    {
        return static::class;
    }

    /**
     * 在这个方法里, 定义意图的全局事件
     * 当执行 $hearing->runIntent($intentName) 时, 会执行全局事件.
     * 执行的就是这个方法.
     */
    public function navigate(Dialog $dialog): ? Navigator
    {
        // 常见的做法有以下几种:

        // return $dialog->wait(); // 什么也不做
        // return $dialog->rewind(); // 重复当前提问
        // return null; // 让该匹配规则失效, 继续进行后续的匹配

        // 最常见的做法, 从当前上下文中, 跳转到 Intent 所在上下文
        // 当 Intent 结束时 (执行了 $dialog->fulfill() ) , 则返回当前上下文
        // 类似 web 网页的跳转.
        return $dialog->sleepTo($this);
    }


    /**
     * 在这里定义意图自己上下文的逻辑.
     */
    public function __onStart(Stage $stage) : Navigator
    {
        ...
    }
}
```

开发者可以通过这个类定义各种各样的意图上下文.
考虑到新接触对话开发, 可能不了解意图的用法, 所以项目根据意图的常见用法又封装了四种子类:

- 消息意图
- 功能意图
- 导航意图
- 任务意图

实际开发中不需要拘泥于预定义的类型, 可按自己需要来设计意图.

### 定义意图实体

意图的实体 (例如 ```[长沙](city)[明天](date)天气怎么样```, 意图是```queryWeather```, 实体是 ```city``` 和 ```date```),
通过和 Context 一样的方式来定义.

具体可以查看 [Context 文档中的 "Entity 机制" 一节](/zh-cn/dm/context?id=_6-entity-机制).

值得注意的是, 对于具体的意图而言, "实体" 的值可能是唯一的, 也可能是复数个值.
NLU 作为中间件, 通常不关心这点, 而是提供实体匹配到的 "数组".
这种差别可能导致使用时的逻辑混乱, 拿到的是数组, 用的时候以为是字符串.

为此, 我们可以使用 ```Context::CASTS``` 机制来定义实体值的类型.

```php

class TellWeatherInt extends ActionIntent
{
    const DESCRIPTION = '查询天气测试用例';

    ...

    // 定义 casts 类型, 会自动进行各种转义
    // 通过 $tellWeatherInt->city 拿到的都是转义后的 "string"
    const CASTS = [
        'city' => 'string',
        'date' => 'string',
    ];

    ...
}

```

具体可以查看 [Context 的强制类型转换](/zh-cn/dm/context?id=%e5%bc%ba%e5%88%b6%e7%b1%bb%e5%9e%8b%e8%bd%ac%e6%8d%a2)



### 消息意图

消息类意图的基类是 ```Commune\Chatbot\App\Intents\MessageIntent```.
这种意图没有定义全局事件, 也没有定义上下文, 仅仅作为对用户消息的抽象来使用.

消息类意图特别适合没有 Entity 的情况, 例如情绪性的表达如 打招呼/称赞/反对/确认 等.

```php

use Commune\Chatbot\App\Intents\MessageIntent;

class Confirm extends MessageIntent
{
    // 命令的形式是 '#confirm'
    const SIGNATURE = 'confirm';

    const DESCRIPTION = '确认';

    const REGEX = [
        // 一个数组定义一个正则
        ['/^(是|不错|对|可以|好的)$/'],
    ];

    public static function getContextName(): string
    {
        // nlu 可以通过返回该意图名表示匹配到.
        return 'attitudes.greet';
    }
}

```

### 功能意图

功能类意图的基类是 ```Commune\Chatbot\App\Intents\ActionIntent```.
它类似于填槽型多轮对话, 表示获取所有 Entity 作为参数后, 会执行一个 Action 动作, 同步返回结果, 没有后续的上下文流程.

这种意图往往作为不影响上下文的全局事件来使用. 例如

```php

/**
 * @property string $city
 * @property string $date
 */
class TellWeather extends ActionIntent
{
    // 用命令的方式定义所需实体
    const SIGNATURE = 'tellWeather
    {city : 请问想了解哪个城市的天气}
    {date : 请问想了解哪天的天气}';

    const DESCRIPTION = '查询天气';

    // 与 Context 定义 Entity 的方式相同.
    public static function __depend(Depending $builder) : void
    {
        // 直接用命令名, 映射为实体
        $builder->onSignature(static::SIGNATURE);
    }

    // 定义执行逻辑
    // 执行到这一步时, 已经获取了 city 和 date 参数
    public function action(Dialog $dialog) : Navigator
    {
        // 以下是伪代码

        // 通过 IoC 容器获取 查询天气的服务实例
        $weatherService = $dialog->app->get('weatherServiceName');

        // 调用实体参数,
        $weather = $weatherService->query([
            'city' => $this->city,
            'date' => $this->date
        ]);

        if ($weather['success']) {
            $dialog->say()->info(
                // 用 replyId 作为消息模板
                'tellWeather.success',
                $weather['data']
            );
        } else {
            $dialog->say()->error(
                'tellWeather.fail',
                $weather['error'],
            }
        }

        // 无论查询成功失败, 都返回上一轮对话
        return $dialog->rewind();
    }
}
```

### 导航意图

导航类意图的基类是 ```Commune\Chatbot\App\Intents\NavigateIntent```.
这种意图没有自己的上下文和功能, 只负责多轮对话的语境调度.

例如用户说 "退出", "取消", "返回上一步" 等.

```php
class CancelInt extends NavigateIntent
{

    const SIGNATURE = 'cancel';

    const DESCRIPTION = '退出当前语境';

    public static function getContextName(): string
    {
        return 'navigation.cancel';
    }

    public function navigate(Dialog $dialog): ? Navigator
    {
        return $dialog->cancel();
    }
}

```

### 任务意图

任务类意图的基类是 ```Commune\Chatbot\App\Intents\TaskIntent```,
它本质上就是一个正常的上下文 ```Commune\Chatbot\OOHost\Context\Context``` 对象,
可以定义复杂的多轮对话过程.
只是这种意图可以通过 NLU 直达.

例如用户说 "我要玩迷宫小游戏", 然后上下文直接跳转到 ```MazeInt```. 性质类似于网站的 url.

### 标记情绪

在使用意图的匹配逻辑中, 有时两种意图是分开的, 有时两种意图可以一起使用.

例如 "Affirm" 表示肯定, "Agree" 表示同意, 它们有时效果一样, 有时不一样:

```
// 问题 1
机器人: 请问要不要加冰?

用户: 是的 // affirm 意图, 表示确认
用户: 同意 // agree 意图, 表示确认


// 问题 2
机器人: 请问您是不是男性?

用户: 是的 // affirm 意图, 表示确认
用户: 同意 // agree 意图, 反而在这里莫名其妙了.
```

所以我们需要定义细分的意图, 但给它们贴上类似 "tag" 的标记, 方便在特殊上下文上进行批量的对待. 简单而言, 人类的意图表达其实是树状的, 不一定到叶子节点才能匹配, 有时候在分支节点就可以匹配了 (这也是自然语言单元未来要实现的功能).

CommuneChatbot 通过 "情绪" 的概念实现了这种批量匹配的机制. 简单而言, 需要先定义 "情绪类", 继承自 ```Commune\Chatbot\OOHost\Emotion\Emotion``` 接口.

```php

// 定义一个 "积极" 的情绪
interface Positive extends Emotion
{
}
```

然后让 IntentMessage 类去实现它, 就相当于打上了 Tag :

```php

class AffirmInt extends AbsCmdIntent implements Positive
{
    ...
}
```

然后, 在使用 Hearing API 进行匹配时, 就可以采取情绪策略了:

```php

    return $hearing
        ...
        // 用情绪接口的类名, 作为匹配的逻辑
        // 可以匹配所有实现它的 intent
        ->feels(Positive::class, $action)
        ...
        ->end();
```

系统默认, 最常用的两种 "情绪" 是 :

- 积极: ```Commune\Chatbot\OOHost\Emotion\Emotions\Positive```
- 消极: ```Commune\Chatbot\OOHost\Emotion\Emotions\Negative```

您可以按需定义自己的 Emotion 类.

"情绪" (Emotion) 是 CommuneChatbot 执行比 Intent 更大粒度匹配时所用的策略.
目前还在试用的阶段, 所以没有独立一节. 更具体的内容请查看 [Hearing API 关于情绪匹配的文档](/zh-cn/dm/hearing.md).

### 确保加载

所有继承自 ```Commune\Chatbot\OOHost\Context\Intent\AbsCmdIntent``` 的意图, 都实现了 ```Commune\Chatbot\OOHost\Context\SelfRegister``` 接口.

因此可以在项目启动时接受扫描,  将自己注册到```Commune\Chatbot\OOHost\Context\Contracts\RootIntentRegistrar``` 之中. 关于注册机制, 请查看 [Registrar 一节](/zh-cn/dm/registrar.md).

要保证 Intent 类会在项目启动时被扫描到, 默认的做法是在机器人配置数组的```$chatbotConfig->host->autoloadPsr4``` 数组中按照 [PSR-4 规范](https://www.php-fig.org/psr/psr-4/) 注册好命名空间和目录的映射关系.
具体可以查看配置对象 ```Commune\Chatbot\Config\Children\OOHostConfig```.

## 7. 意图状态

通常意图 (Intent) 都带有实体参数 (Entities), 可通过多轮对话补完.
而用户告知的实体信息不一定正确, 往往要与用户通过多轮对话来核对.

因此 IntentMessage 除了通过 Context 的 Entity 机制定义了实体,
还需要为每一个实体的确认与否定义属性. 允许在多轮对话中设置该状态.

IntentMessage 通过 Context 的 getter 和 setter 机制实现了这个通用功能.
可查看源码 ```Commune\Chatbot\OOHost\Context\Intent\AbsIntent```.

简单来说, 任何一个 AbsIntent 实例都可以用以下方式来 设置/查询 实体的状态:

```
    /**
     * @var Commune\Chatbot\OOHost\Context\Intent\AbsIntent
     */


    // 通过 setter 设置全局的状态
    $intentMessage->isConfirmed = true;
    // 通过 getter 获取全局状态
    assert(true === $intentMessage->isConfirmed);

    // 获取所有实体确认结果的 map
    $confirmedEntities = $intentMessage->confirmedEntities;

    // 判断其中一个 entity 的结果
    assert(true === ($confirmedEntities['entityName'] ?? false));

    // 对单个 entity 的确认结果进行赋值
    $confirmedEntities['entityName'] = true;
    $intentMessage->confirmedEntities = $confirmedEntities;

    // 先判断全局确认的值, 如果为 null, 则判断所有已定义的 Entity 是不是状态都为 true
    assert(true === $intentMessage->isEachEntityConfirmed());
```

如果使用一个优秀的 IDE, 例如 PHPStorm, 这些代码都会有强类型的提示.

> 注意!!
>
> 不能使用 ```$intentMessage->confirmedEntities['name'] = true;``` 的形式给通过魔术方法获得的数组进行赋值. 必须进行两阶段赋值.

## 8. 预定义意图

为了方便基础功能的开发, CommuneChatbot 设置了一批预定义意图, 在命名空间```Commune\Components\Predefined\Intents``` 之下.

简单罗列如下:

- Attitudes : 态度类意图
    - 确认 : ```Commune\Components\Predefined\Intents\Attitudes\AffirmInt```
    - 否认 : ```Commune\Components\Predefined\Intents\Attitudes\DenyInt```
    - 同意 : ```Commune\Components\Predefined\Intents\Attitudes\AgreeInt```
    - 拒绝 : ```Commune\Components\Predefined\Intents\Attitudes\DontInt```
    - 问候 : ```Commune\Components\Predefined\Intents\Attitudes\GreetInt```
    - 感谢 : ```Commune\Components\Predefined\Intents\Attitudes\ThanksInt```
    - 反感 : ```Commune\Components\Predefined\Intents\Attitudes\DissInt```
    - 夸奖 : ```Commune\Components\Predefined\Intents\Attitudes\ComplimentInt```
- Dialogue : 对话类意图
    - 求助 : ```Commune\Components\Predefined\Intents\Dialogue\HelpInt```
    - 序号 : ```Commune\Components\Predefined\Intents\Dialogue\OrdinalInt```
    - 随意 : ```Commune\Components\Predefined\Intents\Dialogue\RandomInt```
- Loop : 循环命令
    - 打断 : ```Commune\Components\Predefined\Intents\Loop\BreakInt```
    - 下个 : ```Commune\Components\Predefined\Intents\Loop\NextInt```
    - 上个 : ```Commune\Components\Predefined\Intents\Loop\PreviousInt```
    - 从头 : ```Commune\Components\Predefined\Intents\Loop\RewindInt```
- Navigation : 导航类意图
    - 回退 : ```Commune\Components\Predefined\Intents\Navigation\BackwardInt```
    - 取消 : ```Commune\Components\Predefined\Intents\Navigation\CancelInt```
    - 退出 : ```Commune\Components\Predefined\Intents\Navigation\QuitInt```
    - 重复 : ```Commune\Components\Predefined\Intents\Navigation\RepeatInt```
    - 起点 : ```Commune\Components\Predefined\Intents\Navigation\HomeInt```

这些意图会在系统启动时通过 ```Commune\Components\Predefined\PredefinedComponent```, 推荐您考虑使用它们, 但也不一定要使用它们.

## 9. 意图占位符

系统默认通过继承自 ```Commune\Chatbot\OOHost\Context\Intent\AbsIntent``` 的类来定义意图, 在启动的时候加载.

但有时候我们不想这么麻烦, 只打算通过 NLU 传入的字符串名称来定义后续的流程. 这种情况有两种做法.

第一种做法, 直接从 NLU 进行匹配 :

```php
    $hearing->hasPossibleIntent(
        // 不需要定义意图的类
        $intentName,
        // 可以将 Commune\Chatbot\Blueprint\Conversation\NLU 对象依赖注入使用
        function(NLU $nlu, Dialog $dialog) {
            ...
        }
    )
```

这种做法的缺点是拿不到 IntentMessage 对象, 也无法使用 PHP 匹配规则.

第二种做法, 则是使用 ```Commune\Chatbot\OOHost\Context\Intent\PlaceHolderIntentDef``` 类在启动时注册意图占位符, 该意图只需要传入意图的字符串名称即可. 具体可以查看 [Registrar 文档](/zh-cn/dm/registrar.md).

[本地语料库](/zh-cn/dm/corpus.md) 就使用这种机制自动注册意图. 在语料库中记录的意图名称, 不需要定义类, 就会自动作为占位符加载.

使用这种方式注册的字符串意图, 没有实体参数, 但可以获取到```Commune\Chatbot\OOHost\Context\Intent\PlaceHolderIntent``` 对象, 作为 IntentMessage 使用.

## 10. 用中间件实现全局意图导航

上文中所述进行意图匹配, 大部分是通过 ```Commune\Chatbot\OOHost\Dialogue\Hearing``` 来操作的.
这样对意图的响应逻辑就完全和上下文相关.

如果需要一部分意图, 拥有全局响应的优先级 (例如用户说 "退出" 退出机器人), 可以通过 Session Pipe 中间件来实现.

机器人配置数组中的 ```$chatbotConfig->host->sessionPipes``` 中默认有 ```Commune\Chatbot\App\SessionPipe\NavigationPipe``` 管道,
可以在该管道中定义全局的导航意图, 作用类似浏览器网页窗口外部的 "收藏"/"打开"/"配置" 等按钮.

这些意图会在每次输入时尝试匹配, 匹配成功的话会中断后续流程, 立刻执行 navigate 方法.

## 11. 模拟输入意图

开发多轮对话机器人, 需要对上下文流程进行各种调试. 依赖自然语言处理单元来调试, 成本非常高昂.

因此我们需要最快的方法模拟匹配意图. 最简单的做法是给每一个 IntentMessage 都定义命令行规则.
但命令行规则也可能重复, 或者未定义.

因此, 可以在机器人配置```$chatbotConfig->host->sessionPipes```中间件中使用测试专用的```Commune\Chatbot\App\SessionPipe\MarkedIntentPipe``` 管道.

只需要输入意图名称, 在两边加上 ```#``` 字符, 例如```#navigation.cancel#```, 就可以模拟匹配到一个意图了. 这类机制对测试流程非常重要, 您可以有自己的设计.
