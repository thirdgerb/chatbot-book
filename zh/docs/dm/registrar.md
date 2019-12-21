# Registrar

Registrar 指的是```Commune\Chatbot\OOHost\Context\ContextRegistrar```,
是系统存放 ```Commune\Chatbot\OOHost\Context\Definition``` 的容器,
而 Definition 用于存放 Context 的上下文逻辑.

> 开发者通常不用操心 ContextDefinition 或 Registrar, 但需要做高级功能的开发, 还是需要了解细节的.

编程语言的面向对象是在内存中进行管理的, 同样是一个类对应多个实例, 在存储时会拆分为 instance (实例), class (类的定义), classLoader (类的管理) 等, 通过唯一的 className 进行关联.

CommuneChatbot 中, Context (上下文) 虽然也通过面向对象来定义,
 但实例的数据是分布式存储的, 必须经由 Session 还原,
 因此有点像 __用基于内存的面向对象编程语言 (PHP),  实现一个基于分布式存储的面向对象结构__ .

用面向对象的思维来理解各方的关系:

- ContextName : 相当于 className (类名), 唯一
- Context 对象 : 相当于 instance (类的实例)
- Definition : 相当于 class (类的定义)
- Registrar : 相当于 classLoader (类的管理)

因此, 我们把 PHP 对类的操作, 和 Context 相关操作可以做一个比较:

```php

    /**
     * @var Commune\Chatbot\OOHost\Context\Context $context
     * @var Commune\Chatbot\OOHost\Context\Definition $definition
     * @var Commune\Chatbot\OOHost\Context\ContextRegistrar $registrar
     * @var Object $object
     */

    $className = get_class($object); // 获取 object 实例的类名
    $context->getName();   // 获取 context 实例的 contextName

    $object = new ClassName($args); // 实例化一个类
    $context = (new ContextName($args))->toInstance($session); // 实例化 Context 对象

    $reflection = new ReflectObject($object); // 通过实例获取类的反射
    $definition = $context->getDef(); // 通过 context 实例获取 definition

    $object = $reflection->newInstance($args); // 通过反射实例化类
    $context = $definition->newContext($args); // 通过 definition 实例化 context

    $reflection->hasProperty($name); // 通过反射获取属性的反射
    $definition->hasEntity($name); // 通过 Definition 获取 Entity 对象

    $reflection->getMethod($name); // 通过反射获取 method 的反射
    $definition->getStage($name); // 通过 Definition 获得 Stage 对象

    $reflection->getName(); // 通过反射获取类名
    $definition->getName(); // 通过 Definition 获得 ContextName

    $reflection->getDocComment(); // 通过反射获取类的注解
    $definition->getDesc(); // 通过 Definition 获取 Context 的介绍

    class_exists($className); // 判断类是否存在
    $registrar->hasDef(); //判断 Definition 是否存在

    $reflection = new Reflection($className); // 获取类的反射
    $definition = $registrar->getDef($contextName); // 获取 Definition
```

## 1. ContextDefinition

在 [Context](/docs/dm/context.md) 一节中介绍过, ``````Commune\Chatbot\OOHost\Context\Context``` 类用于定义一个上下文,
这是定义上下文最简洁直观的方式. 它包含三方面的信息:

- 上下文记忆
- 对话逻辑 (stage, entity等)
- 对话轨迹

Context 的实例用于保存上下文记忆, 而对话逻辑可以通过 [Stage](/docs/dm/stage.md) 方法来定义, 这些方法会被抽取为```callable```对象, 保存在```Commune\Chatbot\OOHost\Context\Definition``` 里.
它是一个全局单例.

Context 里通过 Stage 定义上下文逻辑, 必须要写成方法, 而且是写死的.

如果想通过配置或别的接口, 批量生成多个 Context, 并动态地生成上下文逻辑,
就不能依赖写死的类方法.

因此, 用 ContextDefinition 来持有上下文逻辑, 类似函数式编程, 是动态性更好的方法.

### Definition 的唯一性

每一个独立的上下文, 都拥有全局唯一的 ```Commune\Chatbot\OOHost\Context\Definition```实例.
它们通过 ContextName (上下文名称) 做唯一的标志.

### Definition 的功能

想要了解 Definition 的功能, 请查看```Commune\Chatbot\OOHost\Context\Definition``` 接口. 简单而言, Definition 管理 Context 的上下文逻辑, 包括:

- [Stage](/docs/dm/stage.md) : 定义上下文逻辑
- Entity : 定义实体
- 关键属性 :
    - name : 作为 Context 唯一ID ContextName, 标记 Context 的类型.
    - desc : Context 的介绍, 可以在对话中直接使用
    - clazz : 实例化后对应的 PHP 类名.
    - tags : Definition 的标签, 用于给 Context 分类, 比如管理对话, 配置对话等.

### 定义上下文逻辑

Definition 的标准实现是 ```Commune\Chatbot\OOHost\Context\ContextDefinition```.
它在初始化的时候, 会从对应 Context 类中抽取 Stage 定义和 Entity 定义, 并持有它们.

当然, 也可以自定义 Definition 类, 通过实现 :

- ```Definition::hasStage($name)```
- ```Definition::getStage($name)```
- ```Definition::hasEntity($name)```
- ```Definition::getEntity($name)```

等方法, 在 Definition 内部定义所有的逻辑, 完全不依赖 Context 类.
还可以进一步的, 将 Entity 和 callable 对象在启动时注册到 Definition 中 :

```php

    // 注册 Entity 实例
    $definition->addEntity(new Commune\Chatbot\OOHost\Context\Entities\PropertEtt('name'));

    // 注册 callable 对象作为 stage
    $definition->setStage($name, $callable);

```

这样的做法更接近函数式编程, 把 Context 类当成一个结构体来使用.

究竟采用面向对象做法 (在 Context 类里定义所有内容), 还是函数式做法 (定义 Context 结构体, 处理逻辑定义在 ContextDefinition 中), 取决于开发者的偏好.

前者代码管理更加直观, 明确, 友好; 而后者拥有更强的动态性.

### 使用 Definition

在使用 ```Commune\Chatbot\OOHost\Dialogue\Dialog``` 对象调度多轮对话时,
基本都是这样的逻辑:

1. 根据 ContextName, 在 Registrar 中找到 ContextDefinition 对象.
2. ContextDefinition 对象找到当前 Stage 的执行方法.
3. 根据状态生成 Stage 对象, 作为参数传递给 Stage 的执行方法.

详见 ```Commune\Chatbot\OOHost\Context\Helpers\ContextCaller```, 和 ```Commune\Chatbot\OOHost\Directing\AbsNavigator```.

### 通过配置定义 Definition

让我看一个用配置生成 Context 的例子, 这是通过 yaml 文件配置自动生成对话式游戏的案例:

```
// 定义脚本名称
id : story.examples.sanguo.changbanpo
title : 大战长坂坡
version : 1.0
messagePrefix : storyComponent.sanguo.changbanpo
// 定义影响剧情分支的道具
itemDef :
  - id : helpJian
    title : 帮助简雍
    enums :
      - 1
      - 0
  ...
// 定义章节
episodes :
  // 定义 episode context
  - id: searchGan
    option : 第一章
    title : 寻找甘夫人
    // 定义多个 stages
    stages :
      // 定义 stage "go"
      - id : go
        title : 出发救主
        // 定义 stage 的对话内容
        stories :
          - playZhao
          - background
          - fightWholeNight
        // 定义 stage 的问答内容
        confirms :
          - query: ifGoFindGan
            yes : meetCivilians
            no : endDieRegrat

      // 定义 stage "meetCivilians"
      - id : meetCivilians
        title : 遇到平民
        stories :
          - startSearchGan
          - meetCivilians
        confirms :
          - query : shouldHelpCivilians
            yes : meetJianYong
            no : missJianYong
    ...
```

这些配置会被读取, 每一节 (episode) 都会作为一个独立的上下文, 将逻辑解析到```Commune\Components\Story\Basic\EpisodeDefinition``` 对象中.

### 获取 Definition

上文已经介绍过, Definition 之于 Context, 相当于类的反射之于类.

获取类的反射, 可以通过类的实例, 或者通过 classLoader (PHP 的classLoader 是全局唯一的).

因此获取 Definition 也可以通过 Context 实例和 Registrar :

```php

    // 类比 $refleciton = new ReflectionObject($object);
    $definition = $context->getDef();

    // 类比 $reflection = new ReflectionClass($className);
    $definition = $registrar->getDef($contextName);
```


## 2. ContextRegistrar

上文介绍过, ContextRegistrar 有点类似于类的 classLoader,
作为全局单例 (进程级单例) 存储所有的 ContextDefinition.

它的基本定义是 : ```Commune\Chatbot\OOHost\Context\ContextRegistrar```.

### 树状结构

我们知道, Context 有各种功能特殊的子实现, 主要是 Intent 和 Memory.
有时候我们查找 Definition, 只想查找 Intent, 或者 Memory;
不想, 也不需要从所有的 Context 中查找.

为适应这类需求, Registrar 实际上是一个树状结构. 可以用树上的任何一个分支节点进行查找, 查找时会尝试遍历所有子节点寻找目标 Definition.

这样的分支节点, 都被定义为 ```Commune\Chatbot\OOHost\Context\ParentContextRegistrar```, 使之具有遍历的功能.

而 Definition 并不存在分支节点上, 而是存在叶节点中. 所以每一个分支节点至少有一个叶节点, 称之为 "Root".

于是整个 Registrar 的抽象结构为 :

+   ```Commune\Chatbot\OOHost\Context\Contracts\RootContextRegistrar```
    +   ```Commune\Chatbot\OOHost\Context\Contracts\RootMemoryRegistrar```
    +   ```Commune\Chatbot\OOHost\Context\Contracts\RootIntentRegistrar```

具体实现为 :

+ ```RootContextRegistrarImpl```
    + ```RootContextRegistrarDefault```
    + ```RootMemoryRegistrarImpl```
        -   ```MemoryRegistrarDefault```
        -   ...
    + ```RootIntentRegistrarImpl```
        -   ```IntentRegistrarDefault```
        -   ...

### 获取 Registrar 实例

ContextRegistrar 作为进程级单例,
可以通过 IoC 容器 (常用 Conversation, 或 ```$app->getProcessContainer()```),
或依赖注入来获取 :

```php

    $contextRepo = $conversation->get(Commune\Chatbot\OOHost\Context\Contracts\RootContextRegistrar::class);

    $memoryRepo = $conversation->get(Commune\Chatbot\OOHost\Context\Contracts\RootIntentRegistrar::class);

```

更常见的方式是通过 Session 直接获取:

```php

    $contextRepo = $session->contextRepo;
    $intentRepo = $session->intentRepo;
    $memoryRepo = $session->memoryRepo;
```

如果想要注册 Definition 或者叶子节点的 Registrar, 则可以通过进程级的 ServiceProvider 直接从 IoC 容器中获取:

```php


class SomeServiceProvider extends ServiceProvider
{
    const IS_PROCESS_SERVICE_PROVIDER = true;

    public function boot(ContainerContract $app)
    {
        $contextRepo = $app[Commune\Chatbot\OOHost\Context\Contracts\RootContextRegistrar::class];


        // 注册 Definiton
        $contextRepo->registerDef(...);

        // 注册一个 Registrar 作为子节点.
        $contextRepo->registerSubRegistrar($registrarId);
    }
}


```

### 启动加载

ContextRegistrar 树上所有的节点, 都需要保证在 ```$chatApp->bootApp()``` 环节, 作为进程级的单例完成加载.

这通常可以通过进程级的 ServiceProvider 来实现.

```php

class MyIntentRepoServiceProvider extends ServiceProvider
{
    const IS_PROCESS_SERVICE_PROVIDER = true;

    public function register()
    {
        $this->app->singleton(
            MyIntentRepo::class,
            ...
        );
    }

    public function boot(ContainerContract $app)
    {
        $intentRepo = $app[Commune\Chatbot\OOHost\Context\Contracts\RootIntentRegistrar::class];

        // 在 IntentRegistrar 的根节点中注册当前容器.
        $intentRepo->registerSubRegistrar[MyIntentRepo::class];

        $myRepo = $app[MyIntentRepo::class];

        // 注册 Definion
        $myRepo->registerDef(...);
    }
}
```

至于注册 Definition 到 Registrar, 除了用 ServiceProvider, 还可以有各种策略.
例如从配置文件中加载, 例如```Commune\Components\UnheardLike\Libraries\UnheardRegistrar```, 完全通过配置文件, 自动生成 Definition.

### Context 自动注册

所有实现了 ```Commune\Chatbot\OOHost\Context\SelfRegister``` 的 Context 类,
可以在系统启动时把自己注册到某一个指定的 Registrar 中.

前提是在配置文件的 ```$chatbotConfig->host->autoloadPsr4``` 位置 (见```Commune\Chatbot\Config\Children\OOHostConfig```),
按 psr-4 规范指定了这些自注册的类所在目录, 那样系统启动时会自动扫描到它们, 并运行加载.