# Hearing

工程化的对话机器人框架中, 机器人的对话逻辑是用规则来实现的, CommuneChatbot 中定义规则的 API 工具是 ```Commune\Chatbot\OOHost\Dialogue\Hearing```.
通过它可以快速定义出各种规则, 是项目的核心工具类之一.

## 1. 链式调用

```php

    public function __onStart(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            // 要求用户选择
            ->askChoose(
                '请问您想要吃点什么',
                [
                    1 => '饮料',
                    2 => '主食',
                    3 => '小食',
                ]
            )

            // 获得 Hearing 对象
            ->hearing()

            // 精确匹配
            ->is('hello', ...)

            // 匹配意图, 并执行意图自带的事件
            ->runAnyIntent()

            // 命中了选项 1
            ->isChoice(1, Redirector::goStage('orderDrink'))

            // 命中了选项 2
            ->isChoice(2, Redirector::goStage('orderFood'))

            // 命中了选项 3
            ->isChoice(3, Redirector::goStage('orderSimpleFood'))

            // 拒答逻辑
            ->end(Talker::say()->info('对不起, 没有明白您的意思'));

    }

```

这是一个 Hearing 调用的基本例子. Hearing API 使用了 注解 + 强类型约束, 来定义它允许的链式调用, 专门针对 IDE 的使用做了优化. 如果使用一个成熟的 IDE, 例如 ```PHPStorm```, 写 Hearing 的相关代码会很容易.

> Hearing API 的代码看上去是写死的. 但它其实是完全可编程的, 您可以用配置文件生成, 或用 API 生成. 以实现逻辑更加复杂, 实现更加简洁, 而且可以随着状态一起变更的逻辑.

## 2. 获取 Hearing 对象

Interface ```Commune\Chatbot\OOHost\Dialogue\Hearing``` 的默认实现是```Commune\Chatbot\OOHost\Dialogue\Hearing\HearingHandler```. 具体的功能请查看源码.

通过```Commune\Chatbot\OOHost\HostConversationalServiceProvider``` 注册到 Conversation 容器中, 可以通过服务注册重写实现.

在项目中获取 Hearing 有两种方式, 第一种是通过```Commune\Chatbot\OOHost\Dialogue\Dialog::hear()```方法.


```php
    public function __onStart(Stage $stage) : Navigator
    {
        return $stage->talk(
            ...
            function(Dialog $dialog) : Navigator {

                // 获取 Hearing 对象
                $hearing = $dialog->hear();

                // 定义匹配规则
                $hearing->
                    ...

                // 通过 end 方法返回 Navigator 对象
                return $hearing->end();
            }
        );
    }
```

由于写闭包的方式常常有代码重复, 所以 Stage 也提供了特殊封装的 Hearing API, 如下:

```php
    public function __onStart(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            // 定义 start 状态下的逻辑
            ->info(...)

            // 定义 callback 状态下的逻辑
            ->hearing()

            // 定义各种匹配规则
            ->...

            // 返回 Navigator 对象
            ->end();
    }
```

这些链式调用 API 都用注解和强类型做过约束, 如果您使用一个成熟的IDE, 例如 ```PHPStorm```, 用起来会更加清晰.


## 3. 条件匹配

Hearing 默认用```匹配条件 + 执行逻辑``` 的方式来定义规则.
相关链式调用方法, 在```Commune\Chatbot\OOHost\Dialogue\Hearing\Matcher```中定义.

所以 Hearing 正常写起来是这样 :

```php
    return $dialog->hear($message)
        // 第一组条件
        ->expect($condition1, $action1)
        // 第二组
        ->expect($condition2, $action2)
        // 第三组
        ->expect($condition3, $action3)
        // 拒答逻辑
        ->end();
```

但我们有时候会遇到多种匹配规则, 对应的是同一个执行逻辑. 这时可以用 ```Hearing::then()``` 方法来简化代码, 变成

```php
    return $dialog->hear($message)
        // 第一组条件
        ->expect($condition1)
        ->is($exactly)
        ->pregMatch($pattern)
        ->isCommand($command)
            ->then($action1)
        // 第二组
        ->expect($condition2, $action2)
        // 第三组
        ->expect($condition3, $action3)
        // 拒答逻辑
        ->end();
```

## 4. Todo API

建议您组织复杂匹配逻辑时, 优先考虑用```Hearing::todo()``` 方法. 它是 ```Hearing::then()``` 方法的倒装. 写成代码是这样的:


```php
    return $dialog->hear($message)
        ->todo($action1)
            // 第一组条件
            ->expect($condition1)
            ->is($exactly)
            ->pregMatch($pattern)
            ->isCommand($command)
        ->otherwise() // otherwise 方法过度回 Hearing 的链式调用.
        // 第二组
        ->todo($action2)
            ->expect($condition2)
        ->otherwise()
        // 第三组
        ->todo($action3)
            ->expect($condition3)
        ->otherwise()
        // 拒答逻辑
        ->end();
```

当使用了 ```Hearing::todo()``` 后得到的是一个模仿 Hearing 的 ```Commune\Chatbot\OOHost\Dialogue\Hearing\ToDoWhileHearing``` 对象. 可以使用所有的匹配规则.

使用 ```otherwise()``` 方法可以返回到 Hearing 对象上, 但如果一个 todo 的下一步仍然是 todo, 或者直接到了 end, ```otherwise()``` 方法也可以省略掉. 但要注意, 错误的省略了 otherwise() 会造成难以预期的后果.

> 强烈建议用 ```Hearing::todo``` 这种方式组织代码.
> 因为即便规则很复杂, 您对机器人能够干什么可以一目了然.
> 本质上遵循了 __对能力的抽象高于对输入的抽象__ 的思路.
> 这样更符合开发者的思维方式, 先定义能力, 再定义各种输入, 将两者连起来.

## 5. 使用 callable 定义执行逻辑

Hearing API 中许多方法定义了 callable 对象作为参数. 例如

```php
    return $dialog->hear($message = null)
        ->is('hello', $callable1)
        ->is('world', $callable2)
        ->fallback($callable3)
        ->end($callable4);
```

这些 callable 对象通常是匹配规则的执行逻辑. 例如

```php
    return $dialog->hear()
        // 匹配到了 "你好", 会执行传入的闭包
        ->is(
            '你好',
            // 使用闭包 (Closure) 定义执行逻辑. 返回一个 Navigator
            function(Dialog $dialog) : Navigator {

                // 向用户回礼
                $dialog->say()->info('你也好啊!');

                // 等待用户下一次输入
                return $dialog->wait();
            }
        )

```

> Hearing 中传入的 Callable 对象就是执行逻辑, 理想情况下每一个执行逻辑都通过 Dialog 返回一个 Navigator, 除非充分了解为什么不返回.

这些 Callable 对象都实现了依赖注入. 除了系统默认已注册的服务都可以直接依赖注入之外, 还提供了一些上下文相关的依赖. 最常用的有:

- $message : ```Commune\Chatbot\Blueprint\Message\Message```
- $dialog : ```Commune\Chatbot\OOHost\Dialogue\Dialog```
- $self : ```Commune\Chatbot\OOHost\Context\Context```

您可以给 callable 对象定义一个```$dependencies``` 参数, 然后查看它,  这个参数记录了所有可以注入的上下文相关依赖.

这里再次介绍一下 PHP 常用的 callable 类型:

- 闭包
- function 的名称字符串, 例如 "is_numeric"
- 拥有```__invoke```方法的对象
- 类的静态方法, 通过数组 ```[类名, 公共静态方法名]```来表示
- 类的动态方法, 通过数组 ```[类实例, 公共动态方法名]``` 来表示


```php
class CallableClass {

    public function __invoke() {...}

    public static function call() {...}
}

$ins = new CallableClass();

// 以下三种 callable 对象都是合法的.
call_user_func($ins);
call_user_func([$ins, '__invoke']);
call_user_func([CallableClass::class, 'call']);
```

CommuneChatbot 额外允许一种类型, ```[类名, 公共动态方法名]```.
如果写成这种情况, 项目会先用依赖注入, 实现该类名的实例;
然后再调用动态方法, 并对动态方法也进行依赖注入.

## 6. Hearing 的生命周期

基于 Hearing API 可以写出很长的链式调用, 一直到最后调用```$hearing->end()``` 方法返回一个```Commune\Chatbot\OOHost\Directing\Navigator``` 对象为止.

如果查看 ```Commune\Chatbot\OOHost\Dialogue\Hearing\HearingHandler```源代码可以发现, 链式调用中间任何一步得到了一个 Navigator 的话, 后面的步骤都不会再真实执行. 所以 Hearing 的优先级是按代码的先后顺序来执行的.

可以尝试 :

```php
    return $dialog->hear()
        ->is('123', function(Dialog $dialog) {

            $dialog->say()->info('match 1');

            return null;  // 没有返回 Navigator
        ))
        ->is('123', function(Dialog $dialog) {

            $dialog->say()->info('match 2');

            return $dialog->wait(); // 返回了 Wait Navigator
        ))
        ->is('123', function(Dialog $dialog) {

            $dialog->say()->info('match 3');

            return $dialog->wait();
        ))
        ->end();
```

用这个例子去测试, 对机器人输入 "123", 得到的应该是 "match 1" 和 "match 2", 由于第二个返回了 Navigator, 所以第三个虽然匹配到了, 也没有真正执行.

### 6.1 Hearing::end()

所有调用 Hearing 的地方, 最后应该调用 ```Hearing::end($defaultFallback)```, 以返回链式调用的匹配逻辑中生成的 ```Commune\Chatbot\OOHost\Directing\Navigator```对象.

查看 ```Commune\Chatbot\OOHost\Dialogue\Hearing\HearingHandler::end()```  方法可以更详细地了解这个流程, 它包括 :

1. 优先运行 Hearing::component(), 加载组件逻辑
1. 如果输入是 EventMsg, 说明是一个可以不响应的事件信号, 则不触发任何响应.
1. 检查是否命中了 "帮助" 意图 (HelpInt), 并主动响应.
1. 执行上文通过 ```Hearing::fallback($fallback)``` 注册的逻辑
1. 执行默认的 defaultFallback 逻辑
1. 如果仍然没有任何匹配逻辑返回的 Navigator, 执行 MissMatch 逻辑, 告知用户无法理解输入.


了解 end() 方法的生命周期, 就可以了解在 Hearing 链式调用的前文中, 什么需求下可以使用 ```Hearing::component```, ```Hearing::fallback```等方法.


### 6.2 Context::__hearing()

由于 Hearing 通常都是在一个 Context 语境里定义, 有可能该 Context 的每一个 Stage 都有一些相同的 Hearing 逻辑.

这时可以把公共的逻辑定义到```Context::__hearing()``` 方法中, 默认在该语境下的任何一个 Stage 调用 Hearing 时都会先通过该方法做初始化. 具体可以查看 ```Commune\Chatbot\OOHost\Dialogue\DialogImpl::hear()``` 方法.

```php

class Test extends Context
{
    public function __onStage(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ...
            ->hearing()
            ...
            ->end();
    }

    /**
     * 所有的 Hearing 都先经过这个方法
     */
    public function __hearing(Hearing $hearing) : void
    {
        $hearing->is('abc', function(Dialog $dialog){...});
    }

}
```

> 注意用 __hearing 方法定义的逻辑优先于 Stage 自己定义的 Hearing 逻辑. 如果想要定义低优先级的逻辑, 可以与 ```Hearing::component``` 组合使用.

### 6.3 Hearing::component()

传入 ```Hearing::component($component)``` 中的 callable 对象, 将得到```Commune\Chatbot\OOHost\Dialogue\Hearing``` 作为唯一参数. 用这样的方法, 可以把一些公共的 Hearing 构建逻辑拆分成组件. 例如:

```php
// 定义公共的组件
class CommonMenu {
    public static function component(Hearing $hearing) : void {...}
}
```

```php
// 在 Hearing 中用到
return $dialog->hear()
    ->component([CommonMenu::class, 'component'])
    ...
    ->end();

```

要注意 component 会先注册, 直到```Hearing::end()```方法的时候才会执行.


### 6.4 Hearing::defaultFallback()

对于无法理解的用户输入, 默认的拒答逻辑会在 ```Hearing::end()``` 方法中执行.

如果没有设置默认的拒答逻辑, 系统会自动调用机器人配置数组里的```$chatbotConfig->defaultMessages->messageMissMatched``` 来回复给用户. 具体配置可以查看```Commune\Chatbot\Config\Children\DefaultMessagesConfig```.

有时候我们希望给用户一个更灵活的通用回复模块, 例如闲聊模块. 当用户的意图无法理解时, 就调用闲聊模块试图用闲聊数据库来响应用户的意图.

类似这样的模块可以定义在配置文件里的```$chatbotConfig->host->hearingFallback``` 参数. 具体可以查看```Commune\Chatbot\Config\Children\OOHostConfig```.

SimpleChatComponent 就提供了这样一个可以用于拒答逻辑的闲聊模块, 具体查看```\Commune\Components\SimpleChat\Callables\SimpleChatAction```.

### 6.5 Hearing::onHelp()

CommuneChatbot 设计了默认的 "帮助" 机制. 当用户在多轮对话某个环节陷入迷惑, 想要得到帮助时, 可以试着提出 "dialogue.help" 意图 (哪些语句命中这个意图, 需要 NLU 去配置), 或者直接输入 "?" 号.

而开发者可以预先通过 Hearing 来定义帮助信息的内容.

```php

    return $dialog->hear()
        ...
        // 在 Hearing 中定义帮助内容.
        ->onHelp(function(Dialog $dialog) {...})
        ...
        ->end();

```

此外, 也可以给整个 Context 定义帮助内容, 通过 ```Context::__help()``` 方法. 这个方法将作为 callable 对象, 在 ```Hearing::end()``` 环节去调用.

如果用户求助, 却没有任何帮助方法可用时, 会用机器人配置数组中定义的```$chatbotConfig->defaultMessages->noHelpInfoExists``` 回复给用户. 具体请看```Commune\Chatbot\Config\Children\DefaultMessagesConfig```.

### 6.6 Hearing::heardOrMiss()

Hearing 虽然有默认的执行顺序, 但可以通过手动调用 ```Hearing::runFallback()```, ```Hearing::runDefaultFallback()``` 等方法调整.

此外, 如果想跳过整个 ```Hearing::end()``` 的流程, 也可以使用更简单的 ```Hearing::heardOrMiss()``` 来返回结果.


## 7. Hearing 的匹配规则

Hearing 的匹配规则定义在```Commune\Chatbot\OOHost\Dialogue\Hearing\Matcher```中. 具体实现定义在```Commune\Chatbot\OOHost\Dialogue\Hearing\HearingHandler```.
具体请查看相关注解和源代码, 这里仅介绍部分功能和方法.

匹配规则大致有以下几类:

- 匹配意图 : 基于 NLU 单元, 根据意图识别来匹配
- 匹配问答 : 基于[问答模块](/zh-cn/dm/questions.md), 来实现回答的匹配
- 匹配情绪 : 用 "情绪" 方式进行范围的匹配.
- PHP匹配规则 : 基于 PHP 实现的匹配规则

### 7.1 匹配意图

CommuneChatbot 项目中, 意图被抽象为```Commune\Chatbot\OOHost\Context\Intent```的形式. 更多关于 "意图" 的介绍, 请查看 [Intent文档](/zh-cn/dm/intent.md). 这里重点介绍如何在 Hearing API 中匹配意图.

#### 意图信息的来源

意图的信息都来源于用户的输入. 但判断用户的输入信息是否符合一个定义好的 "Intent", 有以下方式:

1. 从 MessageRequest 解析得到的 Message, 本身就是 IntentMessage
1. ```Conversation::getNlu()``` 模块在被自然语言单元赋值了意图数据, 通常有多个可能值
1. 使用项目自带的```IntentMatcher```, 用关键字或正则等逻辑匹配出了意图

无论哪种方式获得的意图数据, 都通过统一的 API 获取 :

```php

    /**
     * @var Commune\Chatbot\OOHost\Session\Session $session
     */

    // 获取优先级最高的意图
    $intent = $session->getMatchedIntent();

    // 在所有置信度高于阈值的意图中, 挑选一个指定的意图. 不存在返回 null
    $intent = $session->getPossibleIntent($intentName);
```

#### 匹配单个意图

在 Hearing 中, 匹配单个意图可以写成 :

```php
    return $dialog->hear()

        ...
        ->isIntent(
            // 意图的名称, 可以是真实名称, 也可以是 IntentMessage 的类名.
            $intentName,

            // 匹配到意图后的执行逻辑.
            // 会传入匹配到的 IntentMessage 作为可依赖注入的对象.
            // IntentMessage 的依赖注入约束, 可以用 Message, IntentMessage 或自己的类名.
            function(Dialog $dialog, IntentMessage $message) {...}
        )
        ...

        ->end();

```

获得的 IntentMessage, 是否所有必填的 Entity 都已经具备了,
可以通过 ```IntentMessage::depends()``` 方法来检查. 一个不完整的 IntentMessage 是否可以在逻辑中使用, 由开发者自己来判断.

Hearing 提供了另一个方法简化这种情形的代码:

```php

    return $dialog->hear()
        ...
        ->isPreparedIntent(
            $intentName,
            $whenIntentIsPrepared,
            $whenIntentIsNotPrepared
        )
        ...
        ->end();
```

#### 匹配多个意图

系统所有已注册的意图, 都可以在 ```Commune\Chatbot\OOHost\Context\Contracts\RootIntentRegistrar``` (```$session->intentRepo```)中找到.

意图的名称和其它 Context 的名称一致, 都是用 "." 隔开的小写字符串, 可以将之视作意图的命名空间. 因此, 可以用意图的前缀批量获取意图的名称:

```php

    // 通过共同的前缀, 获取多个意图的名称.
    $names = $session
        ->intentRepo
        ->getDefNamesByDomain($intentPrefix);
```

在 Hearing 中也允许用相同的原理, 一次匹配多种意图并执行逻辑.

```php
    return $dialog->hear()
        ...
        ->isIntentIn(
            'demo.story', // 意图名称的前缀
            $intentAction
        )
        ...
        ->end();
```

#### 匹配任意意图

使用 ```Hearing::isAnyIntent($intentAction)``` 可以在 ```$session->getMatchedIntent()``` 方法有值的时候, 执行响应的逻辑.

> 注意, 这种方法不会使用项目自带的 PHP 匹配规则 ```Commune\Chatbot\OOHost\Context\Intent\IntentMatcher```


#### 使用意图自带的事件机制

项目对意图的封装是 ```Commune\Chatbot\OOHost\Context\Intent\IntentMessage``` 对象, 默认有```IntentMessage::navigate()``` 方法, 作为意图自带的事件机制.

如果一种意图有通用的执行逻辑, 就可以将之封装到```IntentMessage::navigate()``` 方法中.

要在 Hearing 里触发 IntentMessage 自带的逻辑, 可以用以下 API :

```php
    return $dialog->hear()
        ...
        ->runIntent($intentName) // 如果匹配到, 直接运行 navigate
        ->runIntentIn($intentNamePrefix) // 匹配到前缀中的任意意图, 运行 navigate
        ->runAnyIntent() // 如果 $session->getMatchedIntent() 存在, 运行它的 navigate 方法
        ...
        ->end();
```

### 7.2 匹配问答

问答是机器人引导用户对话的最主要形式. 具体的用法在 [问答文档](/zh-cn/dm/questions.md) 中有介绍.

Hearing API 为问答封装了部分方法以简化使用. 最基本的方法是 :

```php
    return $dialog->hear($message)
        ...
        // 可以用 Commune\Chatbot\Blueprint\Message\QA\Answer 来依赖注入
        ->isAnswer(function(Answer $message, Dialog $dialog) {
            // 选项
            $choice = $answer->getChoice();
            // 原消息
            $origin = $answer->getOriginMessage();
            // 对问题的回答, 根据问题类型不同, 回答也有区别.
            // 例如 Confirmation 返回的是 bool
            // 选择类问题, 用户只要局部匹配了, 返回也是完整的答案
            $result = $answer->toResult();
            ...
        })
        ...
        ->end();
```

如果匹配到了答案, 则答案对象```Commune\Chatbot\Blueprint\Message\QA\Answer```会取代原来的 Message, 对处理逻辑进行依赖注入.

#### 使用 Choice

由于所有的问题抽象 ```Commune\Chatbot\Blueprint\Message\QA\Question``` 都允许给用户建议选项, 所有的回答抽象 ```Commune\Chatbot\Blueprint\Message\QA\Answer``` 都有```getChoice()``` 方法, 所以 Hearing 也可以用 choice 来进行匹配:

```php
    return $stage->buildTalk()
        ->askChoose(
            $question,
            [
                $answer0,  // 没有指定 index
                1 => $answer1, // 整数作为 index
                'a' => $answer2, // 字符串作为 index
                'B' => $answer3, // 大写字符作为 index
                ...
            ]
        )

        ->hearing()

        // 用序号匹配到 $answer0
        // 使用 Commune\Chatbot\App\Messages\QA\Choice 作为依赖注入对象
        ->isChocie(0, function(Choice $message, Dialog $dialog) {
            ...
        })

        // 匹配到 $answer1
        ->isChoice(1, ...)

        // 匹配到 $answer2
        ->isChoice('a', ...)

        // 字符串无关大小写
        ->isChoice('b', ...)

        ...

        ->end();
```

> 注意, 用字符串作为选项序号的时候, 匹配时是不区分大小写的. 因为实际使用时发现精确匹配大小写反而容易引发歧义, 不符合直觉.

在 [问答文档](/zh-cn/dm/questions.md) 中会介绍, 各种类型的问题和答案是成对出现的. 例如```Commune\Chatbot\App\Messages\QA\Confirm``` 对应的回答 ```Commune\Chatbot\App\Messages\QA\Confirmation```. 因此在确认匹配成功时, 建议直接用回答的类名作为依赖注入的对象. 使用强类型能更好保证可读性, 并减少错误.

#### 匹配自定义问题

在问答场景中, 系统会保存上一轮机器人提出的问题到上下文, 使用它来判断当前用户的输入是否是一种回答. 从上下文中获取问题可通过```Dialog::currentQuestion()```方法.

但有可能某些情形下, 您需要使用自己定义的问题取代上下文记录的问题. 这时可以使用```Hearing::matchQuestion($question)``` 方法, 替换掉 Hearing 匹配时用的问题.

这样做有可能导致用户的迷惑, 请确认其中利弊.

#### 肯定与否定

机器人与用户对话, 最常见的情况是要用户进行确认. 然而对确认与否的回答却是五花八门的, 通常有以下几种情况 :

- 精确给出建议的字符, 例如 "yes" 和 "no"
- 用一个积极的表达, 或消极的表达来回复, 例如 ```好, 不错, 可以, 别, 不要"
- 重复词句来表达意图, 例如 "要不要加冰", 回答 "加"

为了适应这种复杂的情形, CommuneChatbot 使用 "情绪匹配". 简单而言, 把各种形式的回答, 统一归纳到某种情绪中. 并且对于确认类型的问答, 单独提供了 ```Hearing::isPositive()```和```Hearing::isNegative()```两个方法.

因此强烈建议肯定与否定的场景, 使用以下 API :

```php
    return $stage->buildTalk()
        ->askConfirm("请问是否需要加冰")

        ->hearing()

        // 用户表达肯定
        ->isPositive( ... )

        // 用户表达否定
        ->isNegative( ... )

        ...
        ->end();
```


### 7.3 匹配情绪

除了 Positive, Negative 两种标准情绪之外, 您还可以定义更多自己设计的 "情绪" 概念. 关于情绪模块的介绍, 请看 [相关文档](/zh-cn/dm/emotion.md).

对于自定义的情绪, 可以使用 ```Hearing::feels($emotionName, $action)``` 方法来进行匹配.

### 7.4 PHP匹配规则

指用 PHP 自己实现的匹配规则. 最基本的方法是```Hearing::expect(callable $prediction, callable $action)```. 通过运行第一个 callable 类型参数```$prediction``` 得到一个布尔值, 如果为真才会依赖注入地执行第二个参数```$action```.


以下几个方法也值得说明一下:

```Hearing::is($str, $action)``` : 名义上是字符串精确匹配, 实际上匹配的是```Message::getTrimmedText```(去掉了两边的空格和标点符号), 而且无关大小写.

```Hearing::isEmpty($action)``` : 调用了```Message::isEmpty()```方法. 不过单标点符号(例如 "." )的文字输入, 经过 ```getTrimmedText()```方法也会被省略掉, 从而可以当成 Empty 消息. 这在微信公众号这种不允许输入空消息的平台里是一种替代方法.

```Hearing::pregMatch($pattern, $keys, $action)``` : 一般规则机器人标配的正则模式. 如果正则捕获了用"()"定义的变量, 例如 ``` $pattern = "/^hel(lo) wor(ld)$/"; ```, 这些变量会依次赋值给```$keys```参数中的键名, 然后传入一个 ```Commune\Chatbot\App\Messages\ArrayMessage``` 对象作为 Message, 依赖注入给```$action```.

```Hearing::isCommand()``` : 这个方法可以尝试匹配一个 shell 风格的命令, 匹配成功会得到一个```Commune\Chatbot\Blueprint\Message\Transformed\CommandMsg``` 对象. 具体请看[command相关文档](/zh-cn/dm/commands.md).

```Hearing::soundLike($string, $action, $lang)``` : 使用了```Commune\Support\SoundLike\SoundLikeInterface``` 模块, 允许用 PHP 的手段判断用户输入和我们给出的字符串发音一致. 作为自然语言处理单元的补充方案 (例如语音单元永远把 "章节" 一词解析成 "张杰" ). 允许支持各种语言, 但目前仅支持中文汉语拼音, 通过 [overtrue/pinyin 库](https://packagist.org/packages/overtrue/pinyin) 实现字符串拼音化, 并比较一致性.

```Hearing::soundLikePart()``` : 同样是基于发音来匹配, 但允许用户的输入只匹配预期字符串的某一部分, 默认的四种规则定义在```Commune\Support\SoundLike\SoundLikeInterface``` 里.

```Hearing::matchEntity()``` : 允许基于本地语料库 (详情见[Corpus](/zh-cn/dm/corpus.md) 一节) 的词典来进行匹配检查, 原理类似查找敏感词. 例如调用```matchEntity('city')```方法, 预先又定义了名为 "city" 的城市词典, 就能从用户说的话中抽取类似 "长沙" 这类城市名的词出来. 这个功能也是自然语言处理单元的补充.

----

系统默认的 PHP 匹配规则, 通常要么用于比较简单的情景 (例如```Hearing::is```), 要么是对自然语言单元 ( NLU ) 的补充. 并不建议将它们作为主要的匹配逻辑.

此外有了 ```Hearing::expect ```方法, 您可以任意定义自己匹配规则, 最好是已注册到 IoC 容器中的服务及其动态方法 :

```php
    $hearing->expect([MyMatchClass::classs, $methodName], $callable);
```

甚至可以用第三方服务封装为 callable 对象, 作为 ```$prediction```传给 ```Hearing::expect```方法.

