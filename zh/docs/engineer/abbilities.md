# 权限机制

CommuneChatbot 的权限机制基于 IoC 容器来实现, 任何一种权限, 都是一个继承自```Commune\Chatbot\Blueprint\Conversation\Ability```的 interface, 例如:

```php
<?php

namespace Commune\Chatbot\App\Abilities;

/**
 * 检查用户是不是超级管理员
 */
interface Supervise extends Ability
{
}

```


然后可以在任何地方通过 Conversation 来调用权限判断

```
    /**
     * @var Commune\Chatbot\Blueprint\Conversation\Convosation $convo
     */

     // 检查用户是不是超级管理员.
     $isSupervisor = $convo->isAbleTo(Supervise::class);
     assert(true === $isSupervisor);
```


然后系统会先检查权限的实现是否存在, 然后再调用权限的实现去二次检查.

```php

    public function isAbleTo(string $abilityInterfaceName): bool
    {
        // 权限名需要是 Ability 的子类
        if (!is_a($abilityInterfaceName, Ability::class, TRUE)) {
            return false;
        }

        // 权限 interface 是否已经在 IoC容器里注册了服务.
        if (!$this->has($abilityInterfaceName)) {
            return false;
        }

        /**
         * @var Ability $ability
         */
        $ability = $this->make($abilityInterfaceName);
        return $ability->isAllowing($this);
    }
```

之所以如此设计权限, 因为 CommuneChatbot 是一个高度组件化的项目.
我们可能需要在组件A里定义一种权限, 但并不需要关心在别的平台上如何实现它. 这样组件A只需要提供

## 注册权限

注册一种权限的实现有三步. 第一步是定义出一种权限, 如上文所述.

第二步是提供权限的实现, 例如:


```php
class IsSupervisor implements Supervise {

    public function isAllowing(Conversation $conversation) : bool;
    {
        ...
    }
}
```

第三步是通过 Service Provider 通过配置文件注册到 IoC 容器里, 推荐是进程级容器.

```php
class IsSupervisorServiceProvider extends ServiceProvider
{
    const IS_PROCESS_SERVICE_PROVIDER = true;

    public function register()
    {
        $this->app->singleton(Supervise::class, IsSupervisor::class);
    }

}
```