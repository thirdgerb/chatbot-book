# 第一课, hello world


## 检查依赖

入门课程使用项目框架 [chatbot](https://github.com/thirdgerb/chatbot) 直接运行.

必要依赖:

* php版本 ```>= 7.2``` 
* composer 
* 必要扩展 (最好有所有常用扩展) 
    * mb_string
    * SPL
    * json
    * Reflection
* 建议扩展:
    * php-intl + ICU 以实现文本国际化

强烈建议使用一个优秀的 PHP IDE 来开发, 例如 [phpstorm](https://www.jetbrains.com/zh/phpstorm/) . 因为本项目针对 IDE 做了大量的优化, 基于 IDE 可以极大地减少写代码和查阅 API 的时间.

## clone 仓库

创建目录, 并从 github 克隆仓库

    git clone https://github.com/thirdgerb/chatbot.git 

进入目录 (接下来称之为 ```BASE_PATH``` ), 安装依赖
    
    cd chatbot
    composer install -vvv 

如果```composer install``` 执行很慢, 建议使用[composer中国镜像](https://pkg.phpcomposer.com/)

## 运行检查

在项目目录中运行

    php demo/console.php 

查看是否能正常启动 demo 机器人. 如有未安装的依赖, 请按提示补全. 

## 创建 Hello world 语境

创建 ```HelloWorld.php ``` 文件

    touch BASE_PATH . /demo/src/HelloWorld.php


我们将用这个文件定义第一个上下文语境 (Context). 复制以下代码到 HelloWorld.php :

```php
    <?php
    namespace Commune\Demo;

    use Commune\Chatbot\OOHost\Context\Depending;
    use Commune\Chatbot\OOHost\Context\Exiting;
    use Commune\Chatbot\OOHost\Context\OOContext;
    use Commune\Chatbot\OOHost\Context\Stage;
    use Commune\Chatbot\OOHost\Dialogue\Dialog;
    use Commune\Chatbot\OOHost\Directing\Navigator;
    use Commune\Chatbot\Blueprint\Message\Message;

    /**
     * 创建 hello world 文件
     */
    class HelloWorld extends OOContext
    {
        public static function __depend(Depending $depending): void
        {
        }

        public function __exiting(Exiting $listener): void
        {
        }

        public function __onStart(Stage $stage): Navigator
        {
            // 任何时候都执行的逻辑
            $stage->always(function(Dialog $dialog){
                // dialog 说 hello world
                $dialog->say()->info('hello world!');
            });

            // 等待用户下一次输入
            return $stage->dialog->wait();
        }

    }
```

这样就完成了机器人最初的定义.

## 注册语境

接下来您需要把这个 hello world 语境注册到 demo 机器人的启动配置里.

找到并编辑文件:

    BASE_PATH/demo/configs/config.php

找到 host (多轮对话内核) 的配置, 并添加:


    'host' => [

        'rootContextName' => \Commune\Components\Demo\Contexts\DemoHome::class,

        // 添加教程机器人, 作为一个启动场景.
        'sceneContextNames' => [

            // test 是场景名, 用类名来标记 Context
            'test' => \Commune\Demo\HelloWorld::class,
        ],


## 运行机器人

在 shell 中运行命令:

    php demo/console.php test

就可以看到一个只会说 "hello world" 的机器人了.

想要退出这个机器人, 除了用 'ctrl + c' 直接退出外, 可以输入 ```#quit``` 命令退出.

这是 CommuneChatbot 的命令行机制. 更多命令请输入 ```#help``` 查看.


## 关于场景

CommuneChatbot 有一个 "场景" (scene) 的概念. 简单来说, 同一个机器人, 从不同的 "场景" 进入, 可以指定不同的 "语境" (Context) 来应答.

默认的场景是```root```, 所以运行命令 ```php demo/console.php``` 不需要传入场景名称.

当执行 ```php demo/console.php test``` 时, 就进入了 ```test``` 场景, 这时便启动了之前注册的 ```Commune\Demo\HelloWorld``` 语境.


## [下一课 : 定义单轮对话](/zh-cn/lesions/single-turn-convo.md)