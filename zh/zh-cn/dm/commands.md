# Command

CommuneChatbot 允许在多轮对话中定义[Symfony Console](https://symfony.com/doc/current/console.html) 风格的命令, 可以把对话机器人当成 Shell 来使用. 这种机制可以用于机器人的调试, 在运维对话机器人中可能有更广泛的用途.

## 系统自带的命令

系统自带了两套命令, 提供一些基础功能. 运行一个对话机器人, 输入 ```#help``` 可以查看用户权限可用的命令. 输入 ```/help``` 则可以查看管理员权限的命令, 前提是当前用户拥有管理员权限.

注意, 区分两套命令的方式是前缀字符串 ```#``` 或者 ```/```.


## 定义 SessionCommand

您可以创建类继承```Commune\Chatbot\OOHost\Command\SessionCommand```以实现标准的命令.

需要实现```SessionCommand::handle```方法, 通过字符串常量 ```SessionCommand::SIGNATURE```定义命令格式, 通过```SessionCommand::DESCRIPTION```定义命令的简介.

### 定义命令格式

通过```SessionCommand::getCommandDefinition()```方法, 可以得到一个```Commune\Chatbot\OOHost\Command\CommandDefinition```对象, 命令的规则定义都通过它实现.

所以修改```SessionCommand::getCommandDefinition()```方法可以得到自己的命令规则.

CommuneChatbot 提供了更为简单的方式来定义一个命令的格式, 使用了 Laravel 的```Illuminate\Console\Parser``` 用一个字符串来生成 CommandDefinition.

简单而言, 您可以用以下方式来定义命令格式:

```php

class RunningSpyCmd extends SessionCommand
{

    // 运行示例:
    // "runningSpy -d"
    // "runningSpy"
    //
    const SIGNATURE = 'runningSpy
        {--d|detail : 查看所有选项的详情}
    ';

    ...
}
```

或者:

```php
class ContextRepoCmd extends SessionCommand
{

    // 运行示例:
    // "contextRepo '' 1 20 -i"
    // "contextRepo configure -t"
    //
    const SIGNATURE = 'contextRepo
    {domain? : 命名空间, 默认为空, 表示所有}
    {page? : 查看第几页, 默认为0}
    {limit? : 每页多少条, 默认为0, 表示所有}
    {--i|intent : 仓库为intent}
    {--m|memory : 仓库为memory}
    {--t|tag : 不按domain查询,转为按tag查询.常见tag如"manager"}
    {--p|placeholder : 只查看 placeholder 的情况}
';

    const DESCRIPTION = '查看已注册的 context';

    ...
}
```

来定义一个更为复杂的命令. 用字符串定义命令格式更详细的规范请查看 [Laravel Artisan 文档](https://laravel.com/zh-cn/6.x/artisan#defining-input-expectations)

所有的 SessionCommand 都支持```-h```参数, 可以运行```runningSpy -h ```查看使用规则.

### 获取命令参数

在运行一个命令时, ```SessionCommand::handle```方法会被执行, 这时用户的字符串命令会封装为一个```Commune\Chatbot\Blueprint\Message\Transformed\CommandMsg```对象传入, 可以像数组那样使用其中的 ```Argument``` 与 ```Option``` 值.

```php
class ContextRepoCmd extends SessionCommand
{
    ...

    public function handle(CommandMsg $message, Session $session, SessionCommandPipe $pipe): void
    {
        // 数组方式获取 argument
        $domain = $message['domain'] ?? '';

        $page = intval($message['page']);
        $page = $page > 0 ? $page : 1;

        $limit = intval($message['limit']);
        $limit = $limit > 0 ? $limit : 0;

        // 获取 option 需要用完整名称, 加上 -- 前缀
        if ($message['--memory']) {
            ...

        } elseif($message['--intent']) {
            ...

        } else {
            ...

        }

        ...
    }
```

### 在命令中让机器人说话

```SessionCommand::say``` 方法可以让命令获得```Commune\Chatbot\OOHost\Dialogue\DialogSpeech```对象, 可以用它来让机器人说话:

```php
class WhoAmICmd extends SessionCommand
{
    const SIGNATURE = 'whoami';

    const DESCRIPTION = '查看用户自己的数据';

    public function handle(CommandMsg $message, Session $session, SessionCommandPipe $pipe): void
    {
        $user = $session->conversation->getUser();

        $this->say()->info(
            "您的数据: \n"
            . $user->toPrettyJson()
        );
    }
}
```

### 不影响多轮对话

许多命令仅起到调试的功能, 我们可能不希望它干扰到多轮对话上下文. 而 ```SessionCommand::beSneak``` 属性可以起到这个作用. 当它为```True```的时候, Session 不会存储任何状态. 虽然命令给予了用户反馈, 但机器人会当没有发生过这一轮对话.

### 注册到 CommandPipe

SessionCommand 需要通过中间件加载到系统中才能起作用. 请检查对话机器人的配置文件, 注意查看```host```的配置中是否有命令类的中间件.

在 studio-hyperf 中, ```BASE_PATH/configs/commune/chatbots/demo.php``` 定义了两个默认的命令管道 :

- 用户命令: ```Commune\Studio\SessionPipes\UserCommandsPipe```
- 管理员命令: ```Commune\Studio\SessionPipes\AnalyseCommandsPipe```

它们负责检查用户的消息是否符合命令的格式, 如果符合某个命令, 则用命令去响应用户. 您可以在这里添加自己的自定义命令.

定义一个命令管道, 需要创建类继承```Commune\Chatbot\OOHost\Command\SessionCommandPipe```, 并将它注册到机器人配置数组的```$chatbotConfig->host->sessionPipes```中.

```SessionCommandPipe``` 有两个关键属性 :

- SessionCommandPipe::$commandMark : 决定触发命令必要的前缀字符. 常用```# / .```这三类字符.
- SessionCommandPipe::$commands : 在这个数组中加上命令类名, 则注册了该命令.

> 注册到 SessionCommandPipe 中的命令都可以实现构造方法依赖注入,
> 可以按需注入命令所需的服务. 但要注意, 请不要在 ioc 容器中将命令注册为单例.
> 避免协程并发情况下互相污染.

### 查看帮助

用户可以用 ```help``` 命令查看某一个 SessionCommandPipe 里注册命令的介绍.
注意要加上该管道定义的前缀字符.

用户输入 ```#help``` 只能看到用户的命令, 有权限情况下输入```/help``` 才能看到管理员命令.

另外 SessionCommand 也允许用 ```-h``` 参数查看具体命令的说明. 例如让用户输入```/contextRepo -h```. 说明的内容通过```SessionCommand::SIGNATURE```来定义.


### 使用 Intent 作为命令

用户的意图 (intent) 有多种解析的方法, 在 [Intent](/zh-cn/dm/intent.md) 一节中我们会讨论通过命令格式来匹配意图的方法.

凡是定义了命令解析方式的意图, 也可以作为命令, 用类名字符串注册到 ```SessionCommandPipe::$commands``` 中, 从而作为一个独立的命令来使用. 例如 :

```php
class QuitInt extends NavigateIntent
{

    // 定义了意图的命令名称.
    const SIGNATURE = 'quit';

    const DESCRIPTION = '退出当前会话';

    public static function getContextName(): string
    {
        return StringUtils::normalizeContextName('navigation.'.static::SIGNATURE);
    }

    public function navigate(Dialog $dialog): ? Navigator
    {
        return $dialog->quit();
    }

}
```

### 在 Hearing API 中使用 SessionCommandPipe

所有的```SessionCommandPipe```都是一个 Session 的中间件, 实现了 ```Commune\Chatbot\OOHost\Session\SessionPipe```. 因此可以像中间件一样, 用到 Hearing 中间:

```php

    public function __onStart(Stage $stage) : Navigator
    {
        return $stage->talk(
            ...,
            function(Dialog $dialog) {
                return $dialog
                    ->hear()

                    // 使用命令管道作为中间件.
                    ->middleware(UserCommandPipe::class)
                    ...
                    ->end();
            }
        );
    }
```

这样可以在更具体的业务场景中使用整套的自定义命令.


## 在 Hearing API 中使用命令

Hearing API 有```Commune\Chatbot\OOHost\Dialogue\Hearing::isCommand```方法, 可以传入一个 ```SessionCommand::SIGNATURE```这样的字符串用于匹配命令.

如果匹配到了命令, 无论参数是否正确, 都会得到一个 ```Commune\Chatbot\Blueprint\Message\Transformed\CommandMsg```对象, 作为消息传递给处理消息的 callable.


