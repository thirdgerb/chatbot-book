# Questions

多轮对话通常有用户主导, 和机器人主导两种情况. 当机器人试图主导对话时, 最基本的做法就是通过问答.

当机器人提出一个问题后, 用户的反应可能是 :

- 正确回答问题
- 错误回答问题
- 提出与问题无关的别的需求
- 给出无法理解的反应

因此机器人一方要有多种响应策略. 因此问答在 CommuneChatbot 中建立了一套管理体系.


## Question & Answer

CommuneChatbot 对问答建立了两个核心的抽象:

- 问题抽象 : ```Commune\Chatbot\Blueprint\Message\QA\Question```
- 回答抽象 : ```Commune\Chatbot\Blueprint\Message\QA\Answer```

可以通过接口文件, 了解问答的设计理念.

两者都继承自 ```Commune\Chatbot\Blueprint\Message\Message```, 是一种消息的类型. Question 发送给用户, 而 Answer 则是对用户消息的解析.

其中最核心的一块, 是所有类型的问题, 都允许给出 suggestions (建议);
而任何类型的回答, 都允许是 suggestions (建议) 的一个 choice (选择).

这种拥有 Suggestion 的消息, 被打上 tag  ```Commune\Chatbot\Blueprint\Message\Tags\Conversational```,
表示用户可以对话式地回复这个消息. 该 tag 往往用于渲染回复.

## 问答生命周期

当机器人调用 ```Dialog::reply()``` 方法回复一个 Question 对象时, Dialog 会把 Question 存储到当前 Thread (见[对话生命周期](/zh-cn/dm-lifecircle.md))中, 详见```Commune\Chatbot\OOHost\Dialogue\DialogImpl::reply()```.

因此当前的 Question 对象会跟随上下文轨迹保存. 当用户返回一个消息时, 可以调取该 Question 来检查消息是不是一个答案. 例如 :

```php

    public function __onStart(Stage $stage) : Navigator
    {
        return $stage->talk(

            // 机器人发言环节
            function (Dialog $dialog) {

                $question = new Choose(
                    '请问需要哪种口味?',
                    [
                        1 => '苹果',
                        2 => '香蕉',
                        3 => '哈密瓜',
                    ]
                );

                $dialog->reply($question);

                // 等待用户回答
                return $dialog->wait();
            },

            // 用户回答环节
            function (Dialog $dialog) {

                // 从上下文中还原 Question
                $question = $dialog->currentQuestion();

                // question 解析 Session, 判断是否有答案
                $answer = $question->parseAnswer($dialog->session);

                // 答案存在
                if (isset($answer) ) {
                    ...

                // 答案不存在
                } else {
                    ...
                }

            }
        );
    }
```

这种写法很麻烦, 而实际上提问可以通过```Commune\Chatbot\OOHost\Dialogue\DialogSpeech``` 对象来简化:

```php

    /**
     * @var Commune\Chatbot\OOHost\Dialogue\Dialog $dialog
     */
    $dialog
        ->say()

        // 传入 Question 对象
        ->ask($question)

        // 要求一个文字回答, 不限内容
        ->askVerbal(
            $query,
            $suggestions
        )

        // 要求作出一个选择, 只能选择给出选项
        ->askChoose(
            // 问题
            $query,
            // 选项
            $choices,
        )

        // 要求确认一个 Intent 的内容
        ->askConfirmIntent(
            $query,
            $intentMessage
        )

        // 更多 API 请查看接口文件
        ...
```

而回答的处理, 可以通过 [Hearing API](/zh-cn/dm/hearing.md) 来简化 :

```php

    /**
     * @var Commune\Chatbot\OOHost\Dialogue\Dialog $dialog
     */
    return $dialog->hear()
        ...
        // 是建议中的一个选项
        ->isChoice($a, $actionA)

        // 包含建议中的某个选项
        ->hasChoice($b, $actionB)

        // 用情绪来匹配, 确认类问题的回答也会认为是积极或消极情绪
        ->isPositive($action)

        // 拿到了 Answer 对象
        ->isAnswer($action)

        ...
        ->end();
```

而问答的结合, 可以通过 Stage builder 高度简化代码 :

```php

    public function __onAskName(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            // 可以回答任何字符串, 但也给出建议
            ->askVerbal(
                '请问怎么称呼您?',
                [
                    1 => '张三',
                    2 => '李四',
                    3 => '王五',
                ]
            )
            ->hearing()

            // 根据建议来匹配
            ->isChoice(1, $actionZhangSan)
            ->isChoice(2, $actionLiSi)
            ->isChoice(3, $actionWangWu)

            // 回答了, 但不是选项之一
            ->isAnswer(function(VbAnswer $answer, Dialog $dialog) {
                ...
            })

            // 如果没有使用正确的方式回答..
            ->end(function() { ... });
    }

```

> Suggestions 也可以使用 [Translator](/zh-cn/engineer/translator.md) 进行转义, 因此可以用 replyId (详见 [Reply 一节](/zh-cn/engineer/replies.md) ) 作为 Suggestion 的值.


## "建议" 的匹配规则

系统默认的问答对大多是基于语言的, 并会给出建议答案; 那么, 什么时候认为用户的回答命中了一个 "建议" 呢?

默认的规则定义在 ```Commune\Chatbot\App\Messages\QA\VbQuestion::parseAnswer()``` 方法中. 简单而言:

- 回答必须是文字 (或语音转文字)
- 回答会被去掉两边的空格和标点符号
- 匹配规则 :
    - 允许使用意图 ```Commune\Components\Predefined\Intents\Dialogue\OrdinalInt``` 指定第几个
    - 答案唯一地匹配 (局部匹配, 或完整匹配) Suggestions 其中之一的索引, 不考虑大小写, 允许弱类型
    - 答案唯一地匹配 (局部匹配, 或完整匹配) Suggestions 其中之一的内容
    - 空答案 (```$message->isEmpty()``` 则命中默认值 ```$question->getDefaultValue()```)


匹配规则具体例子请查看单元测试 ```Commune\Test\Chatbot\App\Messages\VbQuestionTest```.

如果默认匹配规则不符合实际需求, 就需要自己定义别的 Question 对象.

### 序数词匹配

当提问存在建议时, 用户常常会用序数词来表示答案, 例如 "第一个", "第二个", "最后一个", "倒数第二个".

这种情况, 我们使用意图 ```Commune\Components\Predefined\Intents\Dialogue\OrdinalInt``` 来匹配. 只要回答解析出了 OrdinalInt 这个意图, 而且 ```$ordinalInt->ordinal``` 的值在 "建议" 的范围内, 就会将之作为答案.

### 标注答案别名

继承自 ```Commune\Chatbot\App\Messages\QA\VbQuestion``` 的问题, 还拥有别名机制用于匹配答案.

简单来说, 用户常常会重复问题中的个别字词, 来表示某种答案. 例如:

```
机器人: 请问要不要加冰呢?

用户: 要 / 好 / 嗯 / 可以 (匹配了 AffirmInt 意图)
用户: 加 / 加冰 (重复问题的谓词, 也表示答案)
```

这类情况下用 suggestions 也很麻烦, 因此我们允许用标注的方式 :

```php
    $question = new Confirm(

        // 实际的问题仍然是 "请问要不要加冰", [] 内是标注的别名, () 是别名对应的值
        // 从标注起点开始匹配, 加, 加冰 的回答都会变成 y
        '请问要不要[加冰](y)'?
    )
```

答案如果从左到右匹配到了标注字符串的若干个字符, 就会把作者的回答改为预定义的回答.
这种功能可以很好弥补 NLU 的能力.

## 问答对

在 CommuneChatbot 中, 问题和答案的类通常是成对出现的. 一个问题类对应一个回答类.
例如:

- 接口:
    - 问题抽象 : ```Commune\Chatbot\Blueprint\Message\QA\Question```
    - 回答抽象 : ```Commune\Chatbot\Blueprint\Message\QA\Answer```
- 基类:
    - 问题 : ```Commune\Chatbot\Framework\Messages\QA\AbsQuestion```
    - 回答 : ```Commune\Chatbot\Framework\Messages\QA\AbsAnswer```
- 语言问答:
    - 问题 : ```Commune\Chatbot\App\Messages\QA\VbQuestion```
    - 回答 : ```Commune\Chatbot\App\Messages\QA\VbAnswer```
- 要求确认:
    - 问题 : ```Commune\Chatbot\App\Messages\QA\Confirm```
    - 回答 : ```Commune\Chatbot\App\Messages\QA\Confirmation```
- 要求单选:
    - 问题 : ```Commune\Chatbot\App\Messages\QA\Choose```
    - 回答 : ```Commune\Chatbot\App\Messages\QA\Choice```
- 要求多选:
    - 问题 : ```Commune\Chatbot\App\Messages\QA\Selects```
    - 回答 : ```Commune\Chatbot\App\Messages\QA\Selection```

每种问题都有其使用场景, 在 ```Question::parseAnswer()``` 方法中定义解析逻辑, 通过解析 Session 获得回答 ``` $answer = $question->parseAnswer($session);```.

开发者可以自由定义问答对, 用于特殊场景中. 尽管默认的问答多是文字的, 基于```VerbalMsg```, 但问题和回答的抽象设计并不依赖文字, 可以是 图片/语音/文件/信号/动作 等等... 只需要实现了问答的抽象接口.

由于 Question 会被序列化存储到上下文记忆中, 所以必须确保 Question 对象可以序列化, 而且体积不会过分庞大.


### VbQuestion

VbQuestion 的问答对是 :

- 问题 : ```Commune\Chatbot\App\Messages\QA\VbQuestion```
- 回答 : ```Commune\Chatbot\App\Messages\QA\VbAnswer```

它的作用类似于网页表单元素 ```input```. 它接受用户的任何文字 (或语音转文字) 作为答案, 同时可以定义建议.

### Confirm

机器人要求用户 "确认" 一个信息, 是最常见的问答场景, 也是匹配形式最多样的一个场景.

- 问题: ```Commune\Chatbot\App\Messages\QA\Confirm```
- 回答: ```Commune\Chatbot\App\Messages\QA\Confirmation```

问题的建议选项通常是 "y" 和 "n". 这时, 回答合法的选项只有 true 和 false 两种, 其实用 ```Answer::getChoice()``` 来表示, 值是 ```0``` 或者 ```1```.

当使用 ```Commune\Chatbot\OOHost\Dialogue\DialogSpeech::askConfirm()``` 来提问时, 回答的默认值不是使用 "y" 和 "n", 而是使用机器人配置的 ```$this->dialog->session->chatbotConfig->defaultMessages->yes``` 和 ```$this->dialog->session->chatbotConfig->defaultMessages->no```.
具体可见 ```Commune\Chatbot\OOHost\Dialogue\DialogSpeechImpl```.

有很多种 "意图" 都可以用来表达 "确认" 或 "否认". 因此 CommuneChatbot 封装了两种广义的情绪 :

- 积极: ```Commune\Chatbot\OOHost\Emotion\Emotions\Positive```
- 消极: ```Commune\Chatbot\OOHost\Emotion\Emotions\Negative```

所以在使用 Confirm 提问后, 建议使用 Hearing API 的情绪方法来判断答案:

```php

    public function __onConfirm(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ->askConfirm('请问您喜欢运动吗?')
            ->hearing()

            // 不需要用 isChoice(1), 都整合进去了
            ->isPositive($positiveAction)

            // isChoice(0) 也整合进去了
            ->isNegative($negativeAction)

            ->end();
    }
```

### Choose

让用户在有限功能中做出选择, 类似菜单的操作, 是引导用户行为最好的办法. 问答对是:

- 问题 : ```Commune\Chatbot\App\Messages\QA\Choose```
- 回答 : ```Commune\Chatbot\App\Messages\QA\Choice```

```php
    public function __onChoose(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ->askChoose(
                '请选择您喜欢的颜色',
                [
                    ...
                ]
            )
            ->hearing()
            ...
            ->isChoice($a, $actionA)
            ->isChoice($b, $actionB)
            ->isChoice($c, $actionC)
            ...
            ->end();
    }
```

值得一提的是, 单项选择经常会用到 "功能菜单" 对话中, 机器人提示几种功能, 让用户选择, 根据用户的选择进入相关多轮对话.

这些目标 "功能", 实际上可能是一个方法, 可能是一个 "Stage", 还可能是一个 "Context".
有很多代码可能都是重复的.
因此封装了一个 ```Commune\Chatbot\App\Callables\StageComponents\Menu``` 组件, 可以极简地在对话中实现功能菜单 :

```php

    public function __onMenu(Stage $stage) : Navigator
    {
        $menu = new Menu(
            '请您选择以下功能: ',
            [
                // 直接用 ContextName 作为值
                // 会给用户显示 $context->getDef()->getDesc() 作为选项
                // 如果选择, 直接用 sleepTo 跳转到目标语境.
                MazeInt::class,

                // 用 stageName 作为值, 表示 Dialog::goStage 到目标 stage
                // key 是显示给用户的选项.
                '前往 someStageName' => 'someStageName',

                // 用 callable 对象作为值,
                // 表示命中后直接执行该对象
                // key 也是现实给用户的选项
                '执行 action 方法' => [$this, 'action']
            ]
        );

        // 用 menu 去加载 stage
        return $stage->component($menu);
    }


```

### Selects

多选功能的问答对是 :

- 问题 : ```Commune\Chatbot\App\Messages\QA\Selects```
- 回答 : ```Commune\Chatbot\App\Messages\QA\Selection```

它通常表示用户一句话回答了所有选项, 并用某个固定符号隔开.

由于这并不符合自然对话中常见的多选方式, 因此并不作为推荐方案, 请慎重使用.

理想的方案, 多选本身是一个 Loop 类型的多轮对话, 既允许一句话选多个, 也允许一句话选一个. 用户用特定的话语表示选择结束.


### 意图相关问答

[Intent 一节](/zh-cn/dm/intent.md) 介绍过, 意图本身可以是一个多轮对话, 要填满所有必填的实体 (Entity) 才能正式使用.

一部分 NLU 服务, 都是围绕意图实体的 "填槽" (slot filling) 来推进多轮对话的.
为了与它们相结合, CommuneChatbot 提供了一些上下文相关的问答功能.

可见: ```Commune\Chatbot\OOHost\Dialogue\DialogSpeech```, 和 ```Commune\Chatbot\OOHost\Dialogue\DialogSpeechImpl```.

具体而言, 有:

- askIntentEntity : 询问 Entity 的一个实体值, 等同 slot filling.
- askConfirmIntent : 要求用户确认一个意图.
- askConfirmEntity : 要求用户确认意图的一个实体值.
- askSelectEntity : 要求用户为一个实体选择多个值.
- askChooseEntity : 要求用户为一个实体选择单个值.
- askChooseIntents : 要求用户作出选择, 用意图作为选项.

查看源码可以了解具体的实现. 但对 CommuneChatbot 而言, 这些方法不是必要的.
它们过度依赖 Intent 的定义, 反而失去了简洁和灵活性.
如果不是必须要和 NLU 做配合 (NLU可以主动处理这些 Question), 并不推荐使用.

## 渲染 Question

Question 会像其它```Commune\Chatbot\Blueprint\Message\ReplyMsg``` 一样, 渲染后发送给客户端.

然而 Question 属于交互类消息, 实现了 ```Commune\Chatbot\Blueprint\Message\Tags\Conversational```,
这意味着不同的客户端有不同的渲染形式.

最简单的方式, 是用 ```[1] 选项1``` 的方式, 用字符串来表示选项. 例如微信公众号.

但在支持触屏, 点击的客户端, 选项通常会渲染成按钮. 例如 https://communechatbot.com

在有触屏的智能音箱中, 带选项的消息会被渲染成有按钮的 Card.

诸如此类. 因此, Question 的渲染是消息渲染里最基本的一环. 具体请看 [回复渲染文档](/zh-cn/engineer/replies.md).

## 用 Context 代替问答

问答对是一种语义高度明确的单轮对话, 实现机器人对会话的引导, 以及用户提供信息给机器人.

然而具体场景中的问答可能非常复杂, 远非一个单轮对话可以涵盖 (例如多选). 因此我们鼓励用小型的多轮对话来定义各种类型的问答, 并形成规范.

这样的问答单元, 写出来可能是 :

```php
    pubic function __onSomeStage(Stage $stage) : Navigator
    {
        return $stage

            ->dependOn(

                // 问题做成了 Context
                new ConfirmContext($quetion, $suggestions),

                // 在回调中处理答案
                function(ConfirmContext $confirmation, Dialog $dialog) {

                    // 有答案时
                    if ($confirmation->hasAnswer) {
                        ...

                    // 没有答案时
                    } else {
                        ...

                    }
                }
            );
    }
```