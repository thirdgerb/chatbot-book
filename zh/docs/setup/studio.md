# 安装工作站

工作站项目是 [commune/studio-hyperf](http://packagist.org/packages/commune/studio-hyperf), 使用 [swoole](https://swoole.com) + [hyperf](https://hyperf.io) 实现, 提供协程服务端的全栈开发框架.

要使用工作站, 您可能需要掌握 :

* swoole 基础知识
* php 协程相关知识
* redis 与 mysql 的部署与使用
* 服务部署与治理相关知识

## 检查依赖

安装工作站, 有以下必要的依赖 :

- PHP >= 7.3
- PHP 扩展
    - 所有常用 PHP 扩展
    - [swoole >= 4.4](https://www.swoole.com/)
    - [intl (国际化)](https://www.php.net/manual/en/intro.intl.php)
    - redis
    - (如有遗漏请告知)
- PHP Composer
- Mysql
- Redis


## 安装项目

创建一个目录, 在该目录下使用 composer 安装工作站源码

    // 用 composer 安装源代码. 可以自己指定目录名
    composer create-project commune/studio-hyperf

    // 进入安装目录
    cd studio-hyperf

    // 安装依赖
    composer install

如果 composer 安装的时间太长, 请考虑使用 [composer 中国镜像](https://pkg.phpcomposer.com/).

### 运行 demo 检查

工作栈提供了一个测试工具, 可以在命令行里查看对话, 默认不依赖数据库. 需要运行:

    php bin/hyperf.php commune:tinker

tinker 机器人具体的配置在文件 ``` BASE_PATH/configs/autoload/commune.php ``` 中.

## 配置环境变量

    // 复制环境变量文件.
    cp .env.example .env

编辑 .env 文件, 配置:

    ## hyperf app 的名称.
    APP_NAME=commune

    ## 系统默认的 mysql 连接配置
    DB_DRIVER=mysql
    DB_HOST=localhost
    DB_PORT=3306
    DB_DATABASE=commune
    DB_USERNAME=root
    DB_PASSWORD=
    DB_CHARSET=utf8mb4
    DB_COLLATION=utf8mb4_unicode_ci
    DB_PREFIX=

    ## 系统默认的 redis 连接配置
    REDIS_HOST=localhost
    REDIS_AUTH=
    REDIS_PORT=6379
    REDIS_DB=0

    # 机器人是否开启 Debug
    CHATBOT_DEBUG=true           ## 是否开启 debug 模式

    # 默认 #whoyourdaddy 指令使用的 token 值, 用于在对话中获取管理员身份
    SUPERVISOR_TOKEN=

    # 系统自带各种应用, 启动时的监听端口
    CHAT_TCP_PORTS=9501   ## tcp demo 端
    CHAT_DUEROS_PORT=9502 ## 百度音箱应用端口
    CHAT_WECHAT_PORT=9503 ## 微信公众号应用端口
    CHAT_WEB_PORT=9504    ## 网页版应用端口
    CHAT_API_PORT=9505    ## http api 应用端口

    # 百度智能音箱服务端的独立配置
    DUEROS_PRIVATE_KEY=         ## duerOS 私钥文件路径, 不填则不会进行鉴权
    DUEROS_REDIS_HOST=localhost
    DUEROS_REDIS_AUTH=
    DUEROS_REDIS_PORT=6379
    DUEROS_REDIS_DB=1           ## duerOS 应用使用的 redis 库, 用于隔离缓存

    # 微信公众号服务端的独立配置
    WECHAT_REDIS_HOST=localhost
    WECHAT_REDIS_AUTH=
    WECHAT_REDIS_PORT=6379
    WECHAT_REDIS_DB=0           ## 同 duerOS
    WECHAT_APP_ID=              ## 微信公众号, 或测试号的 app_id
    WECHAT_APP_SECRET=          ## 微信公众号, 或测试号的 app_secret
    WECHAT_TOKEN=               ## 鉴权所用的 token
    WECHAT_AES_KEY=             ## 加密的密钥

    # 如果使用 rasa 作为 nlu, 这是 rasa 提供的 http api 接口地址
    RASA_API=localhost:5005

在 .env 文件中定义的配置, 可以在代码中用 ``` env('WECHAT_APP_ID', ''); ``` 这样的方式获取. 详情见 [hyperf 环境变量](https://doc.hyperf.io/#/zh/config?id=环境变量)

## 初始化数据库

工作站自带的 SessionDriver 依赖数据表定义在目录 ```BASE_PATH/migrations``` 之下.

定义了数据库连接后, 需要执行命令以初始化数据库:

    php bin/hyperf.php migrate

更多指令请查看 [hyperf 数据库迁移](https://doc.hyperf.io/#/zh/db/migration)

## 启动服务

工作站通过 [hyperf 的命令](https://doc.hyperf.io/#/zh/command) 来启动服务.

系统自带的服务, 可查看配置文件 ``` BASE_PATH/configs/autoload/commune.php ```.

部分应用配置如下:

    // 可以通过 commune:start 命令启动的, 真实客户端.
    // 配置内容请查看 Commune\Hyperf\Foundations\Options\AppServerOption
    'apps' => [
        // 默认的tcp端. 通常供测试用.
        'tcp' => include BASE_PATH . '/config/commune/apps/tcp.php',

        // 系统自带的 web 端, 示范如何开发 web 端的对话机器人.
        'web' => include BASE_PATH . '/config/commune/apps/web.php',

        // 系统自带的 api 端, 可以像 mvc 框架那样通过 http api 访问机器人的数据
        'api' => include BASE_PATH . '/config/commune/apps/api.php',

        // 系统自带的 dueros 端, 可以连接小度音箱设备.
        'dueros' => include BASE_PATH . '/config/commune/apps/dueros.php',

        // 系统自带的 微信公众号服务端. 可作为微信公众号的服务.
        'wechat' => include BASE_PATH . '/config/commune/apps/wechat.php',


    ],

所有已配置的应用, 可以通过运行命令 ``` php bin/hyperf.php commune:start [appName] ``` 的方式启动:

    // 运行 duerOS 服务端
    php bin/hyperf.php commune:start dueros

    // 运行 wechat 服务端.
    php bin/hyperf.php commune:start wechat

在服务器上可以用[supervisor](https://doc.hyperf.io/#/zh/tutorial/supervisor) 之类的方式启动持久化的服务.


## 提供 http 服务

duerOS, wechat, web, api 这类基于http协议的客户端, 可以在浏览器地址栏里输入 ```localhost:9501```  的方式直接访问 ( .env 文件配置了每个应用的端口号 ). 但这样显然无法对外提供服务.

要提供正规的 http 服务, 可以使用 [nginx 反向代理](https://doc.hyperf.io/#/zh/tutorial/nginx), 或用别的方法, 需要您掌握了相关知识. 这里提供一个简单的 nginx 反向代理配置示范:

    # web 端
    upstream web {
        server 127.0.0.1:9530;
    }

    # api 端
    upstream api {
        server 127.0.0.1:9531;
    }

    server {
        listen 80;
        server_name communechatbot.test;

        location / {
            index index.html;
            root /var/www/studio-hyperf/app-frontend/public;
        }

        # web 访问路径. commmunechatbot.test/web/
        location /web {
            proxy_pass http://web;
        }

        # api 端访问路径 commmunechatbot.test/api/
        location /api {
            proxy_pass http://api;
        }

        access_log /var/log/nginx/commune_test.log;
        error_log  /var/log/nginx/commune_error.log;

    }

