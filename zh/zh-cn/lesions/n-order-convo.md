# 第五课 : 依赖关系的 N 阶多轮对话

关于复杂多轮对话问题, CommuneChatbot 项目有自己的一套解释, 您可以通过 [这篇文章](/zh-cn/core-concepts/complex-conversation.md) 了解.

简单而言, 当一个多轮对话 A 的某个节点, 嵌套了另一个多轮对话 B , 这就构成了一个 二阶多轮对话. 而 B 可以再嵌套 C, C 可以再嵌套 D ... 从而构成了 N 阶多轮对话.

由于每一个节点都可以有分叉, 走向不同的 1阶多轮对话, 因此整个多轮对话看起来是 ```树状```的.

但实际上, 子级多轮对话的结果, 又可以反过来影响父级多轮对话的走向, 因此整个多轮对话是 ```图结构```的.

接下来我们了解一下 CommuneChatbot 如何实现这种```图结构``` 的多轮对话吧.


## 定义依赖 UserInfo 的 WelcomeUser 语境


我们先创建一个欢迎用户的语境, 并且要求这个语境要先获取用户的信息.

创建文件 ```touch BASE_PATH/demo/src/WelcomeUser.php```

编辑内容:

```php
<?php


namespace Commune\Demo;


use Commune\Chatbot\OOHost\Context\Depending;
use Commune\Chatbot\OOHost\Context\Exiting;
use Commune\Chatbot\OOHost\Context\OOContext;
use Commune\Chatbot\OOHost\Context\Stage;
use Commune\Chatbot\OOHost\Directing\Navigator;
use Commune\Chatbot\App\Callables\Actions\Redirector;

/**
 * 欢迎用户的测试用例
 *
 * @property-read UserInfo $user  增加一个注解, 方便IDE识别
 */
class WelcomeUser extends OOContext
{
    public static function __depend(Depending $depending): void
    {
        // 标记 user entity 依赖另一个 Context
        $depending->onContext('user', 'demo.lesions.user-info');
    }

    public function __exiting(Exiting $listener): void
    {
    }

    public function __onStart(Stage $stage): Navigator
    {
        return $stage->buildTalk()
            // 打招呼
            ->info(
                '欢迎您, %name%',
                [
                    // 直接链式调用 UserInfo 的参数
                    'name' => $this->user->name
                ]
            )
            // 打完招呼直接结束.
            ->fulfill();
    }
}
```


> 注意, 我们这里用了 ```Depending::onContext``` 方法, 将上一课的 ```UserInfo``` 作为一个 ```Entity``` 定义成了 ```WelcomeUser``` 的依赖.


然后我们把它也注册到场景中, 修改 ```BASE_PATH/src/configs/config.php```, 在 ```host``` 数组里添加:

```php
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

            // n 阶多轮对话的测试用例.
            'nOrder' => \Commune\Demo\WelcomeUser::class,
        ],
```

接下来请运行 ``` php demo/console.php nOrder``` 查看效果.

> 通过 depend entity 的方式, 让 UserInfo 作为实体被 WelcomeUser 所持有. 只要 WelcomeUser 没有被垃圾回收, UserInfo 就会一直存在, 可以反复使用.

## 填槽型的 N 阶多轮对话

上一课我们讨论过使用 ```Context::__depend``` 方法可以快速定义填槽式的任务型多轮对话.

而在 CommuneChatbot 中, 另一个多轮对话也可以视作一个 ```Entity```, 被其它的```Context``` 所依赖. 被依赖的```Context```还可以继续依赖其它的```Context```, 从而分形几何式地嵌套下去.

因此, 仅仅使用```Context::__depend``` 的形式就可以做出一个```树状``` 的 ```N阶多轮对话```, 只不过一个槽位 (slot) 是通过另一个多轮对话来实现的. 这个依赖树理论上不限深度, 会随着对话推进, 用先序遍历的方式层层递归地回到根节点.

这样就构成一个依赖型的 N阶多轮对话树, 而它的递归栈在项目中称之为 ```Thread```


## 用 Stage::dependOn 定义更灵活的依赖关系

填槽的策略只能实现比较简单的多轮对话, 因为所有的参数输入都集中在任务的头部, 而任务只执行一轮.

而现实中的多轮对话任务, 往往任务本身就是多轮的交互, 任何一个环节都可能依赖别的语境(```Context```). 这就无法靠填槽的形式去实现.

在 CommuneChatbot 中的最佳实践, 是把每个依赖产生的环节, 定义为一个独立的 ```Stage```, 并给予独立的回调机制.

让我们用这种思路重构 ```WelcomeUser``` 如下:


```php
<?php


namespace Commune\Demo;


use Commune\Chatbot\OOHost\Context\Depending;
use Commune\Chatbot\OOHost\Context\Exiting;
use Commune\Chatbot\OOHost\Context\OOContext;
use Commune\Chatbot\OOHost\Context\Stage;
use Commune\Chatbot\OOHost\Dialogue\Dialog;
use Commune\Chatbot\OOHost\Directing\Navigator;

/**
 * 欢迎用户的测试用例
 * @property UserInfo|null $user  增加一个注解, 方便IDE识别
 */
class WelcomeUser extends OOContext
{
    public static function __depend(Depending $depending): void
    {
    }

    public function __exiting(Exiting $listener): void
    {
    }

    public function __onStart(Stage $stage): Navigator
    {
        return $stage->buildTalk()
            ->info('这里是欢迎用户测试用例')
            // 然后进入询问用户信息的环节
            ->goStage('askUserInfo');
    }

    /**
     * 询问用户信息
     * @param Stage $stage
     * @return Navigator
     */
    public function __onAskUserInfo(Stage $stage) : Navigator
    {
        return $stage

            // dependOn 的回调事件是 onIntended, 这里也测试一下
            ->onIntended(Talker::say()->info('(拿到了dependOn的回调)'))

            // 定义 dependOn
            ->dependOn(
            'demo.lesions.user-info',

            // 等价于 onIntended 回调事件.
            function(Dialog $dialog, UserInfo $userInfo) : Navigator {

                // 将拿到的结果进行赋值.
                $this->user = $userInfo;

                // 重定向到 final
                return $dialog->goStage('final');
            }
        );
    }

    public function __onFinal(Stage $stage) : Navigator
    {
        return $stage->buildTalk()
            ->info(
                '您好! %name%',
                [ 'name' => $this->user->name ]
            )
            ->fulfill();
    }

}
```

然后我们执行 ```php demo/console.php nOrder``` 查看效果.

使用这种方式, 就可以在多轮对话的任何一个环节, 定义一个依赖关系的多轮对话嵌套了.


## 逃离多轮对话陷阱

当用户进入一个 N 阶多轮对话后, 可能因为各种各样的原因 (不耐烦, 或者有别的事情) 希望能够中途退出来. 而机器人自己也可能要拒绝用户访问某个多轮对话. CommuneChatbot 将常见的情况总结为:

* cancel : 用户主动退出一个多轮对话
* quit : 用户退出所有的会话
* reject : 系统拒绝用户访问一个对话
* failure : 系统运行发生错误, 导致一个多轮对话自动退出.

问题的关键不在于用何种方式退出多轮对话, 而在于退出到哪一个节点.

CommuneChatbot 认为, 由__依赖关系__形成的 ```Thread```, 在任何一个```stage``` 节点尝试退出时, 应该退出整个 ```Thread```.

让我们为此写一个测试场景. 先修改上一课的类 ```UserInfo```, 我们给它新增一个方法:

```php
    /**
     * Hearing 方法为当前 Context 里所有的 hearing api 添加公共的匹配逻辑
     * @param Hearing $hearing
     */
    public function __hearing(Hearing $hearing) : void
    {
        $hearing

            // 模拟用户取消
            ->todo(function(Dialog $dialog) {
                return $dialog->cancel();
            })
                // 精确匹配字符串 'cancel'
                ->is('cancel')

            // 模拟退出整个会话
            ->todo(function(Dialog $dialog) {
                return $dialog->quit();
            })
                // 精确匹配字符串 'quit'
                ->is('quit')

            // hearing 内部的 to do api 必须以 otherwise 结尾.
            ->otherwise();

    }

```

> ```Context::__hearing```  方法会让当前 ```Context``` 在任何时候调用 ```Dialog::hear``` 的时候都执行这个方法, 为 ```Hearing``` api 增加公共逻辑.


然后我们运行 ```php demo/console.php nOrder```, 然后在询问姓名的环节尝试输入 :

* "cancel"
* "quit"

查看效果. 可以看到, 输入 "cancel" 和 "quit" 时, 都导致了整个会话的退出. 因为在退出 ```UserInfo``` 时, 依赖它的 ```WelcomeUser``` 也一起结束了, 当多轮对话根节点结束时, 就会调用 ```quit``` 退出会话.


## 拦截退出语境事件

多个```Context``` 嵌套构成的依赖语境, 在遇到退出事件时, 会整体地退出. 但有时候我们并不希望如此简单, 而是父级语境也能够知晓退出事件, 并有所反应.

比如上文的例子, 我们可能希望用户只要在 ```UserInfo``` 语境中输入了名字, ```WelcomeUser``` 语境就不需要退出.

这个需求可以通过 ```Context::__exiting``` 方法实现. 让我们修改 ```WelcomeUser```  类的 ```__exiting``` 方法如下:

```php
   /**
    * 退出事件拦截
    * @param Exiting $listener
    */
   public function __exiting(Exiting $listener): void
   {
       $listener
           // 拦截依赖流程的 cancel 事件
           ->onCancel(function(Dialog $dialog, Context $context) : ? Navigator {
               $dialog->say()->info('(侦测到 cancel 指令)');

               // 如果 name 有值, 就拦截事件.
               if (
                   $context instanceof UserInfo
                   && $context->hasAttribute('name')
               ) {
                   $dialog->say()->info('(userInfo::name 参数存在, 直接进入下一步"final")');
                   $this->user = $context;
                   // 重定向到 farewell
                   return $dialog->goStage('final');
               }

               // 如果 name 没值, 不做任何额外操作.
               return null;
           })
           // 拦截全局的 quit 事件.
           ->onQuit(function(Dialog $dialog){
               $dialog->say()->info('(侦测到 quit 指令, 不做拦截)');
               return $dialog->quit(true);
           });
   }
```

然后运行 ```php demo/console.php nOrder``` , 在询问姓名时我们给出答案, 但在询问 ```email``` 时我们输入 ```"cancel"```, 以查看效果.


## [下一课 : 不相依赖的 N 阶多轮对话](/zh-cn/lesions/n-thread-convo.md)