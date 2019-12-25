# 工作站目录结构

工作站 [commune/studio-hyperf](http://packagist.org/packages/commune/studio-hyperf) 是基于 [Hyperf skeleton](https://github.com/hyperf/hyperf-skeleton) 项目改造而来.

目录结构如下 :

- ```/app/``` Hyperf 的应用代码, 用于实现 web 服务端
- ```/app-frontend/``` CommuneChatbot 的 web 版前端代码, 示范搭建前端网站
    - ```./public``` 前端代码编译后的文件路径.
- ```/app-studio/``` 对话机器人的逻辑源码, 命名空间 ```Commune\Studio```
    - ```./Abilities``` 权限功能的源码
    - ```./Commands``` 命令行功能源码
    - ```./Contexts``` 机器人多轮对话逻辑源码
    - ```./Listeners``` 事件机制的 listener
    - ```./Providers``` 各种默认提供的服务
        - ```./AbilitiesServiceProvider``` 注册权限
        - ```./EventServiceProvider``` 注册事件
        - ```./FeelingServiceProvider``` 情绪模块
        - ```./RenderServiceProvider``` 注册回复消息渲染
        - ```./FakeAbilitiesServiceProvider``` 在 tinker 中注册权限
    - ```./SessionPipes``` 多轮对话内核的管道
- ```/bin``` hyperf 的命令行
- ```/config``` 配置文件
    - ```./autoload```  hyperf 的启动加载配置
    - ```./commune``` 对话机器人的相关配置
        - ```./apps``` 应用级配置
        - ```./chatbots``` 机器人配置
        - ```./configs``` 关联配置, 用于定义某个具体功能
- ```/migrations``` 项目创建数据表用到的文件
- ```/rasa-demo``` 用于提供 [Rasa](/docs/components/rasa.md) 作为 NLU 服务的默认配置
- ```/resources``` 资源文件
    - ```./langs``` 机器人默认的语言包
    - ```./nlu``` 机器人默认的本地语料库
        - ```./entities``` 本地实体词典
        - ```./intents``` 本地意图语料库
        - ```./synonyms.yml``` 本地同义词词典
- ```/runtime```
    - ```./logs``` 默认日志文件目录
- ```/test``` 单元测试目录


项目的目录结构不是硬性的, 主要是为方便新人使用, 所以给出了```app-studio```, ```rasa-demo```, ```app-frontend``` 等目录.

希望开发者使用一个成熟度的 PHP IDE, 例如 [phpstorm](https://www.jetbrains.com/zh/phpstorm/) 来进行开发. 至少用类名, 或者用文件名来查找源码会非常方便.