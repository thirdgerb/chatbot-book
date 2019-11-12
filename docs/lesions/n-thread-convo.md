# 第六课 : 不相依赖的 N 阶多轮对话

上一课我们讨论了, 多个 ```Context``` 通过 ```dependOn``` 方式嵌套, 形成依赖关系的 N 阶多轮对话. 它们在上下文中的递归栈称之为 ```Thread```. 同一个```Thread``` 遇到 ```cancel``` 等各种退出事件时, 会退出整个 ```Thread```.

而实际的业务场景中, 更常出现的是不相依赖的多轮对话嵌套. 用户和机器人在讨论语境 A 的时候, 用户可能临时插入一个语境 B, 当语境 B 结束了, 需要重新返回到语境 A.

如果语境 B 遇到了 ```cancel```等事件时, 也不应该影响到语境 A.


在 CommuneChatbot 中, 实现语境依赖的 api 是 ```Stage::dependOn``` 方法, 对应回调事件 ```Stage::onIntended```.

而定义不相依赖的语境跳转, 则是用 ```Stage::sleepTo``` 方法, 对应回调事件 ```Stage::onFallback```. 让我们用具体的例子看看.


## 创建 UserMenu 语境

我们先创建一个新的 ```Context```, 用它来提供类似菜单的功能.

创建文件, ```touch BASE_PATH/demo/src/UserMenu.php```.

编辑文件的内容如下:

```php
<?php


namespace Commune\Demo;


use Commune\Chatbot\App\Callables\Actions\Redirector;
use Commune\Chatbot\OOHost\Context\Depending;
use Commune\Chatbot\OOHost\Context\Exiting;
use Commune\Chatbot\OOHost\Context\OOContext;
use Commune\Chatbot\OOHost\Context\Stage;
use Commune\Chatbot\OOHost\Dialogue\Dialog;
use Commune\Chatbot\OOHost\Dialogue\Hearing;
use Commune\Chatbot\OOHost\Directing\Navigator;
use Commune\Components\Demo\Cases\Maze\MazeInt;
use Commune\Chatbot\App\Callables\StageComponents\Menu;

/**
 * 用户的菜单
 *
 * @property-read UserInfo $user 仍然依赖用户信息.
 */
class UserMenu extends OOContext
{

    public function __construct(UserInfo $user)
    {
        // 原始 __construct 方法接受一个 map
        // map 里的值都会付给当前 Context, 然后可以用 $context->{$key} 的方式来调用
        // 注意 map 里的值应该全部都可以序列化
        // 最好是基础值(is_scalar), Message 对象, Context 对象
        // 它们会序列化后保存在 session 的上下文里.
        parent::__construct(get_defined_vars());
    }

    public static function __depend(Depending $depending): void
    {
    }

    public function __exiting(Exiting $listener): void
    {
    }

    public function __onStart(Stage $stage): Navigator
    {
        return $stage->buildTalk([
                'name' => $this->user->name
            ])
            ->askChoose(
                '您好, %name%, 请选择您需要的操作:',
                [
                    1 => '迷宫小游戏',
                    2 => '返回'
                ]
            )
            ->hearing()

            // 进入迷宫游戏
            ->todo(Redirector::goStage('maze'))
                ->isChoice(1)

            // 返回
            ->todo(Redirector::goCancel())
                ->isChoice(2)
                // 如果是 "cancel" 字符串, 也可以直接执行返回.
                ->is('cancel')

            ->otherwise()

            // 拒答
            ->end(function(Dialog $dialog){
                $dialog->say()->warning('对不起, 您的选项不存在');
                // 重复语境
                return $dialog->repeat();
            });
    }

    public function __onMaze(Stage $stage) : Navigator
    {public function __onStart(Stage $stage): Navigator
         {
             $menu = (new Menu(
                     '您好, %name%, 请选择您需要的操作:',
                     [
                         // 直接用语境作为值, 会自动调用该语境的 desc
                         MazeInt::class,

                         // 用 stage 名称作为值
                         '迷宫' => 'maze',

                         // 用闭包作为值
                         '返回' => Redirector::goCancel()
                     ]

                 ))
                 ->withSlots(['name' => $this->user->name])
                 ->onHearing(function(Hearing $hearing) {
                     $hearing->defaultFallback(function(Dialog $dialog){
                         $dialog->say()->warning('对不起, 您的选项不存在');
                         // 重复语境
                         return $dialog->repeat();
                     });
                 });

             return $stage->component($menu);
         }

        // 通过 sleep to, 进入到迷宫游戏
        return $stage->sleepTo(
            MazeInt::getContextName(),
            // 迷宫游戏退出后的回调方法
            function(Dialog $dialog){
                $dialog->say()->info('(迷宫游戏结束)');

                // 回到 start 环节.
                return $dialog->goStage('start');
            }
        );
    }

}
```

> 注意这一次我们给 UserMenu 增加了 __construct 方法, 用于示范语境与语境之间直接传值的做法.

这样, 我们就获得了一个对话式的小菜单, 可以进入到系统自带的迷宫小游戏.

> 要退出迷宫小游戏, 可以输入```#cancel``` 或者 "退出".


## 将 UserMenu 添加到 WelcomeUser

接下来, 我们要把 ```UserMenu``` 添加到上一课的 ```WelcomeUser``` 中.

修改之前的 ```WelcomeUser::__onFinal``` 方法 :

```php
    public function __onFinal(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ->askChoose(
                '请选择您想要的功能:',
                [
                    1 => '进入菜单',
                    // 字符串也可以用来做选项
                    'q' => '退出',
                ]
            )
            ->hearing()
            ->isChoice(1, Redirector::goStage('menu'))
            ->isChoice('q', Redirector::goFulfill())
            ->end();

    }

    /**
     * 示范通过 sleep, 进入菜单语境.
     * @param Stage $stage
     * @return Navigator
     */
    public function __onMenu(Stage $stage) : Navigator
    {
        return $stage

            // 示范 fallback 事件被触发.
            ->onFallback(Talker::say()->info('(触发了 fallback 事件)'))

            ->sleepTo(
            // 和 dependOn 一样, 直接用类名, 或者 contextName, 就可以指定目标语境
            // 不过我们这次为了示范 Context::__construct 的用法, 允许传入一个语境实例
                new UserMenu($this->user),

                // 回调的时候, 返回 final stage
                Redirector::goStage('final')
            );
    }

```

我们预期的情况是, 当填写完用户信息之后, 会让我们选择输入 "1" 进入 ```UserMenu``` 语境. 而在 ```UserMenu``` 语境中, 我们如果选择了 "返回" 或者输入 "cancel", 将回到 ```WelcomeUser``` 语境的 ```final``` 阶段. 而不会连累整体退出.

在命令行输入 ```php demo/console.php nOrder``` 以测试结果.


## sleepTo 的原理

上一课我们讨论过, ```Stage::dependOn``` 可以构建语境之间的依赖, 依赖关系的递归栈称之为 ```Thread```.

而 ```Stage::sleepTo``` 将会用目标 ```Context``` 创建一个新的 ```Thread``` , 而当前的```Thread```则进入```sleeping```状态.

只有新的```Thread```执行了```fulfill```, ```cancel```等事件退出时, 才会通过```fallback```事件重新唤醒```sleeping```中的```Thread```.

所以同一个会话中, 可能同时存在多个```Thread```, 一个时间只有一个```Thread```是活跃状态(```active```). 它们一起构成了会话 (```Session```) 的进程 (```Process```).


## 使用 Stage::component 来简化代码

CommuneChatbot 追求对多轮对话完全可编程, 在此基础上可以再封装各种工具和配置以简化代码.

在前面的小课中, 我们已经见过了几种工具:

* Redirector : 封装了重定向的 callable
* Talker : 用链式调生成对白的 callable
* BuildTalk : 用链式调用封装 ```Stage::talk```

接下来示范 ```Stage::component``` 的做法, 它可以把```Stage```的常见逻辑封装成独立的组件.

让我们重构 ```UserMenu::__onStart``` 如下 :

```php
    public function __onStart(Stage $stage): Navigator
    {
        // 定义了 Menu 组件.
        $menu = (new Menu(
                '您好, %name%, 请选择您需要的操作:',
                [
                    // 直接用语境作为值, 会自动调用该语境的 desc
                    MazeInt::class,

                    // 用 stage 名称作为值
                    '迷宫' => 'maze',

                    // 用闭包作为值
                    '返回' => Redirector::goCancel()
                ]

            ))
            ->withSlots(['name' => $this->user->name])
            ->onHearing(function(Hearing $hearing) {
                $hearing->defaultFallback(function(Dialog $dialog){
                    $dialog->say()->warning('对不起, 您的选项不存在');
                    // 重复语境
                    return $dialog->repeat();
                });
            });

        // 用组件去 build stage
        return $stage->component($menu);
    }

```


在命令行输入 ```php demo/console.php nOrder``` 以测试结果.

在了解了组件化的原理之后, 您也可以按自己的需要任意定义组件.