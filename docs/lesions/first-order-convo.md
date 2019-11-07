# 第三课 : 定义一阶多轮对话

如果完成一个任务需要经过多个单轮对话, 但没有嵌套其它的多轮对话, CommuneChatbot 称之为 "一阶多轮对话".

在本节中我们将了解一个一阶多轮对话的定义方法.

##  创建新的测试场景

为了不和一二章的内容相混和, 我们先创建一个新的测试场景.

您需要先创建文件:

    touch demo/src/FirstOrderConvo.php

往里面添加以下内容:

```php
<?php


namespace Commune\Demo;


use Commune\Chatbot\App\Callables\Actions\Redirector;
use Commune\Chatbot\App\Callables\Actions\Talker;
use Commune\Chatbot\OOHost\Context\Depending;
use Commune\Chatbot\OOHost\Context\Exiting;
use Commune\Chatbot\OOHost\Context\OOContext;
use Commune\Chatbot\OOHost\Context\Stage;
use Commune\Chatbot\OOHost\Dialogue\Dialog;
use Commune\Chatbot\OOHost\Directing\Navigator;

/**
 * 一阶多轮对话的示例.
 */
class FirstOrderConvo extends OOContext
{
    public static function __depend(Depending $depending): void
    {
    }

    public function __exiting(Exiting $listener): void
    {
    }

    /**
     * 启动环节.
     *
     * @param Stage $stage
     * @return Navigator
     */
    public function __onStart(Stage $stage): Navigator
    {
        return $stage->talk(
            function(Dialog $dialog) {

                $dialog->say()->info('输入任何信息进入下一步');

                // 等待用户输入
                return $dialog->wait();
            },
            function(Dialog $dialog) {

                // 用 goStage 方法指定跳转的目标 stage
                return $dialog->goStage('final');
            }
        );
    }

    /**
     * 最终步, 结束对话.
     * @param Stage $stage
     * @return Navigator
     */
    public function __onFinal(Stage $stage) : Navigator
    {
        $name = $stage->name;
        $stage->dialog->say()->info("到达了 $name 环节, 流程退出.");

        // 结束流程.
        return $stage->dialog->fulfill();
    }


}
```

然后我们把这个语境也注册到机器人配置中去:

修改文件 ```BASE_PATH/demo/configs/config.php``` :

在数组 'host' 中添加

```php
    'host' => [
        'rootContextName' => \Commune\Components\Demo\Contexts\DemoHome::class,

        // 新手教程: 添加机器人, 作为一个启动场景.
        'sceneContextNames' => [

            // test 是场景名, 用类名来标记 Context
            'test' => \Commune\Demo\HelloWorld::class,

            // 一阶多轮对话 测试用例.
            'firstOrder' => \Commune\Demo\FirstOrderConvo::class,
        ],
```


然后运行 ```php demo/console.php firstOrder``` 查看效果.

可以看到, 机器人接受到任何信息之后, 就跳到了 ```__onFinal ``` 方法. 这个方法用 ```__on``` 做前缀来标记方法名. 而 stage 自己的名称则是 'final'.


## 使用 Builder 工具来简化代码.

您可能觉得每次都要写一个闭包 (```function(){}```) 很麻烦, 其实结合 PHP 的 callable 机制, 可以设计各种 Builder 工具来简化代码.

CommuneChatbot 项目自带多种生成 callable 对象的常用工具, 都在 ```Commune\Chatbot\App\Callables``` 命名空间下. 我们使用两个最常见的工具来优化一下 ```__onStart ``` 方法:

```php
    /**
     * 启动环节.
     *
     * @param Stage $stage
     * @return Navigator
     */
    public function __onStart(Stage $stage): Navigator
    {
        return $stage->talk(
            Talker::say()->info('输入任何信息进入下一步'),
            Redirector::goStage('final'),
        );
    }

```

然后运行 ```php demo/console.php firstOrder``` 查看效果.


## 定义多个 Stage

接下来, 我们试着多定义两个stage, "step1" 和 "step2", 让对话沿着顺序依次往后走.


```php
    /**
     * 启动环节.
     *
     * @param Stage $stage
     * @return Navigator
     */
    public function __onStart(Stage $stage): Navigator
    {
        return $stage->talk(
            Talker::say()->info('输入任何信息进入下一步'),
            Redirector::goStage('step1')
        );
    }

    /**
     * @stage   用 stage 注解来定义, 而不是用前缀
     * @param Stage $stage
     * @return Navigator
     */
    public function step1(Stage $stage) : Navigator
    {
        $name = $stage->name;
        return $stage->talk(
            Talker::say()
                ->info("进入了 stage $name")
                ->info('输入任何信息进入下一步'),
            Redirector::goStage('step2')
        );
    }

    /**
     * 第二步
     * @param Stage $stage
     * @return Navigator
     */
    public function __onStep2(Stage $stage) : Navigator
    {
        $name = $stage->name;
        return $stage->talk(
            Talker::say()
                ->info("进入了 stage $name")
                ->info('输入任何信息进入下一步'),
            Redirector::goStage('final')
        );
    }


    /**
     * 最终步, 结束对话.
     * @param Stage $stage
     * @return Navigator
     */
    public function __onFinal(Stage $stage) : Navigator
    {
        $name = $stage->name;
        $stage->dialog->say()->info("到达了 $name 环节, 流程退出.");

        // 结束流程.
        return $stage->dialog->fulfill();
    }
```

然后运行 ```php demo/console.php firstOrder``` 查看效果.

您可能注意到, ```step1``` 这次不是用 ```__on``` 前缀来标记的, 而是用了一个注解 ``` @stage ``` 来标记它.

CommuneChatbot 有很多注解的使用办法, 在后面会介绍. 但标记 Stage 建议还是用 ```__on``` 前缀, 因为这种做法可读性更好, 不会和其它的方法名混在一起, 难以辨别.


## 定义有分支的多轮对话

在上一种多轮对话实现中, 每个 stage 都必须知道自己的下一步是什么, 而且下一步是唯一的. 而现实中的多轮对话, 往往存在各种各样的分支, 可以根据逻辑来切换.

让我们再修改一下 case, 实现一个有分支的多轮对话.

```php
    /**
     * 启动环节.
     *
     * @param Stage $stage
     * @return Navigator
     */
    public function __onStart(Stage $stage): Navigator
    {
        return $stage->talk(
            Talker::say()->info('输入任何信息进入下一步'),
            Redirector::goStage('step1')
        );
    }

    /**
     * @param Stage $stage
     * @return Navigator
     */
    public function __onStep1(Stage $stage) : Navigator
    {
        $name = $stage->name;

        return $stage

            // 定义了 onStart 事件, 只在start stage 时执行.
            ->onStart(
                // say 方法传入数组(模板slots), 可以让所有后续的 info, warning等方法共用.
                Talker::say(['name' => $name])
                    ->info("进入了 stage $name")
            )

            // talk
            ->talk(
                function(Dialog $dialog) : Navigator {

                    // 向用户抛出选择题.
                    $dialog->say()->askChoose(
                        '请选择您想走的路线',
                        [
                            1 => '路线1',
                            2 => '路线2',
                        ]
                    );

                    return $dialog->wait();
                },

                function(Dialog $dialog)  : Navigator {

                    return $dialog->hear()

                        ->isChoice(1, function(Dialog $dialog){

                            // 使用 goStagePipes 可以定义一个stage 管道
                            return $dialog->goStagePipes(['step2', 'step3', 'final']);
                        })

                        // Redirector 对 goStagePipes 的封装.
                        ->isChoice(2, Redirector::goStageThrough(['step3', 'step2', 'final']))
                        ->end();

                }
            );
    }

    public function __onStep2(Stage $stage) : Navigator
    {
        // 这次我们封装一个内部方法, 来复用代码.
        return $this->goStep($stage);
    }

    public function __onStep3(Stage $stage) : Navigator
    {
        return $this->goStep($stage);
    }

    /**
     * 进入到下一步.
     * @param Stage $stage
     * @return Navigator
     */
    protected function goStep(Stage $stage) : Navigator
    {
        $name = $stage->name;
        return $stage->talk(
            Talker::say()
                ->info("经过了 stage : $name")
                ->info("输入任何信息进入下一步"),
            function (Dialog $dialog) : Navigator {

                // 使用 next 方法, 如果管道中有下一步, 则进入下一步
                // 否则执行 fulfill 方法.
                return $dialog->next();
            }
        );
    }

    /**
     * 最终步, 结束对话.
     * @param Stage $stage
     * @return Navigator
     */
    public function __onFinal(Stage $stage) : Navigator
    {
        $name = $stage->name;
        $stage->dialog->say()->info("到达了 $name 环节, 流程退出.");

        // 结束流程.
        return $stage->dialog->fulfill();
    }
```


然后运行 ```php demo/console.php firstOrder``` 查看效果. 可以看到第一步要求我们选择路线, 选择会导致后面的流程不一样.

在这个示范中, 我们还引入了几个新的知识点:

__Dialog::goStagePipes__ : 这个方法可以将多个 ```stage``` 组合成一个管道, 每一个 stage 只用调用 ```Dialog::next``` 方法就可以进入下一步, 而不需要知道下一步是什么. 这样就解耦了中间节点和下文.

__askChoose__ : 机器人引导用户提供信息时, 需要通过```提问```的方式. 将提问进行封装, 可以优化各种匹配逻辑, 例如示例中用了 ```Hearing::isChoice``` 方法. 用户输入 '1', '路线1' 都会匹配到相通结果.

我们还定义了一个 ```goStep``` 方法用来复用代码. ```Context``` 作为一个类, 可以用各种方法来实现封装和解耦.


## 使用链式工具 BuildTalk 来优雅地定义对话

```Stage::talk() ``` 是定义单轮对话最基础的做法, 允许自己定义各种编程逻辑. 但绝大多数的 stage, 只需要发送和接受信息, 基础的做法会导致大量冗余代码.

CommuneChatbot 提供了 BuildTalk 的链式调用, 能让您非常优雅地描绘一个多轮对话. 让我们再次重构上面这个例子.

```php
    /**
     * 启动环节.
     *
     * @param Stage $stage
     * @return Navigator
     */
    public function __onStart(Stage $stage): Navigator
    {
        // 创建 builder
        return $stage->buildTalk()
            // start 阶段
            ->info('输入任何信息进入下一步')

            // 通过 hearing, 进入 callback 阶段
            ->hearing()
            ->end(Redirector::goStage('step1'));
    }

    /**
     * @param Stage $stage
     * @return Navigator
     */
    public function __onStep1(Stage $stage) : Navigator
    {
        $name = $stage->name;

        return $stage->buildTalk(['name' => $name])
            ->info("进入了 stage $name")
            ->askChoose(
                '请选择您想走的路线',
                [
                    1 => '路线1',
                    2 => '路线2',
                ]
            )
            ->hearing()
            ->isChoice(1, Redirector::goStageThrough(['step2', 'step3', 'final']))
            ->isChoice(2, Redirector::goStageThrough(['step3', 'step2', 'final']))
            ->end();

    }

    public function __onStep2(Stage $stage) : Navigator
    {
        // 这次我们封装一个内部方法, 来复用代码.
        return $this->goStep($stage);
    }

    public function __onStep3(Stage $stage) : Navigator
    {
        return $this->goStep($stage);
    }

    /**
     * 进入到下一步.
     * @param Stage $stage
     * @return Navigator
     */
    protected function goStep(Stage $stage) : Navigator
    {
        $name = $stage->name;
        return $stage->buildTalk()
            ->info("经过了 stage : $name")
            ->info("输入任何信息进入下一步")
            ->hearing()
            ->end(Redirector::goNext());
    }

    /**
     * 最终步, 结束对话.
     * @param Stage $stage
     * @return Navigator
     */
    public function __onFinal(Stage $stage) : Navigator
    {
        $name = $stage->name;
        return $stage->buildTalk()
            ->info("到达了 $name 环节, 流程退出.")
            ->fulfill();
    }

```

这样是不是简洁, 优雅多了?

```Stage::buildTalk``` 方法使用强类型约束, 有很严谨的语法结构. 详情请查看:

* ```Commune\Chatbot\OOHost\Context\Stages\OnStartStage```
* ```Commune\Chatbot\OOHost\Context\Stages\OnCallbackStage```

如果您有很好的 IDE 支持, 可以放心使用这种方式来定义 stage. 它反而能让您的代码更严谨.


## [下一课 : 定义 N 阶多轮对话](/docs/lesions/n-order-convo.md)