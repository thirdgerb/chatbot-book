# 配置中心抽象层

CommuneChatbot 实现了配置中心抽象层策略. 从结果上来看, 可以通过```Commune\Support\OptionRepo\Contracts\OptionRepository``` 类来获取已注册的```Option``` 实例.

```php
    public function dependencyInjection(OptionRepository $repo)
    {
        if ($repo->has($categoryName, $optionId)) {
            // $categoryName : 配置的分类名称
            // $optionId : 配置枚举值的ID
            $option = $repo->get($categoryName, $optionId);
        }
    }
```

CommuneChatbot 的各种组件, 都大量使用了这个机制, 用来存储各种```Option枚举值```.

更细致的 api, 请查阅 ```Commune\Support\OptionRepo\Contracts\OptionRepository``` 类.

## 为何要用配置中心

在一个分布式服务端的系统中, 常常需要通过```分布式配置中心```, 例如```etcd```, ```zookeeper```等来获取配置. 从而做到一处修改, 立刻同步到所有的服务器.

配置中心不仅对于```服务化治理```很重要, 也可以应用到业务中. 比如在一个配置中心后台定义了一个秒杀活动的配置, 立刻所有服务器就上线了这个活动.

[studio-hyperf](https://github.com/thirdgerb/studio-hyperf)
底层依赖的 [Hyperf 框架](https://hyperf.wiki/#/zh/config) 对各种配置中心有很好的协程客户端支持,
详情可见 [相关文档](https://hyperf.wiki/#/zh/config-center)

## 什么是配置中心抽象层

配置中心用于系统配置时还好办, 要用于业务逻辑时就需要一个抽象层来解耦.

当我们的系统, 或者业务需要一个分布式配置中心时, 底层可能使用的是```etcd```, ```zookeeper```, 甚至是```redis```和```mysql```. 每一种存储介质都有一套独立的 api, 当需要把配置从```redis``` 迁移到 ```etcd```时, 有可能要改动所有的调用代码, 还需要迁移数据.

所谓的配置中心抽象层, 指的是不从存储介质 (```Storage```) 直接获取数据, 而是通过一个统一的抽象层:

```php
    // 直接从 redis 获取配置
    $option = $redis->get('optionId');

    // 直接从 etcd 封装里获取配置
    $option = $etcd->get('optionId');

    // 都改成从抽象层获取配置
    $option = $repo->get('categoryName', 'optionId');
```


抽象层可以实际上定义了一个存储介质(```Storage```)的管道

```php

    // 预先注册了配置的分类元数据
    $repo->register([
        // 定义了分类名
        'categoryName' => 'name',

        // 定义了获取配置的 storage 管道
        'storagePipeline' => [
            'redisStorage',
            'etcdStorage',
        ],

        // 定义了一个方便修改, 编辑的根 storage
        'rootStorage' => 'mysqlStorage',
    ]);

```


然后从这个管道中来读写数据:

```php

    // 从 storage 管道中获取配置
    public function find(
        string $category, // 配置的分类名称
        string $optionId // 配置的ID
    ): Option
    {
        // 获取配置的元数据
        $meta = $this->getCategoryMeta($category);

        // 没有该配置的 storage 保存在这个栈中.
        $noDataStack = [];

        $result = null;

        // 按顺序遍历 storage
        foreach ($this->getStoragePipeline($meta) as $storage) {

            $result = $storage->get($meta, $storageMeta, $optionId);

            // 数据已经找到, 跳出循环.
            if (isset($result)) {
                break;
            }

            // 将没有数据的 storage 入栈
            array_push($noDataStack, $storage);
        }

        // 如果所有管道都没有找到.
        if (!isset($result)) {

            // 从根介质中获取数据
            $rootStorage = $this->getRootStorage($meta);
            $result = $rootDriver->get($meta, $storageMeta, $optionId);
        }
        
        // 实在没有数据, 直接返回
        // 如果第一层介质就找到了数据, 也直接返回
        if (empty($result) || empty($noDataStack)) {
            return $result;
        }

        // 如果有存储介质没有数据, 则把找到的数据缓存进去.
        // 从前往后存, 这样下一个请求拿到数据更快.
        foreach ($noDataStack as $storage) {
            $storage->save($meta, $storageMeta, $result);
        }

        return $result;

    }

```


在这个策略中, ```etcd, redis, zookeeper``` 都抽象成了存储介质```Storage```. 在 ```Storage``` 构成的管道中, 上层的```Storage``` 就相当于缓存层. 而最底层的 ```Storage``` 可以是一个拥有独立后台, 方便修改的存储介质.

而且当数据需要从原来的 AStorage 迁移到新的 BStorage, 只需要将 BStorage 作为 AStorage 的上层, 再调用 ```OptionRepository::sync``` 方法就可以.


## 哪里用到了配置中心抽象层

CommuneChatbot 有许多组件都需要用到枚举值式的配置. 例如:

* __corpus__ : 本地语料库
* __StoryComponent__ : 对话式情景游戏, 用```ScriptOption```作为剧本

系统需要一些开箱自带的配置, 以文件 (```.yml```) 的形式保留在代码版本库中.

但这些模块正式使用时, 可能需要一个基于```redis, mysql, etcd``` 等各种介质的分布式存储方案. 以做到后台一处修改, 所有服务端都同时生效.

所以这样的配置, 基本都通过配置中心抽象层 (```OptionRepository```) 来获取, 然后先记录在 ```.yml``` 文件中. 未来只需要修改 ```CategoryMeta``` 配置, 就可以将这些配置转移到别的存储介质中去了.


## 注册 CategoryMeta

配置中心抽象层```Commune\Support\OptionRepo\Contracts\OptionRepository``` 需要在启动前注册```Commune\Support\OptionRepo\Options\CategoryMeta``` 来定义各种配置的分类.

注册流程可以通过 ```ServiceProvider``` 来实现:


```php
class CategoryMetaServiceProvider extends ServiceProvider
{
    const IS_PROCESS_SERVICE_PROVIDER = true;

    /**
     * @param ContainerContract $app
     */
    public function boot($app)
    {
        $repo = $app->get(OptionRepository::class);
        $repo->registerCategory(new CategoryMeta([
        
            // category 的名称
            'name' => ScriptOption::class,
            
            // category 读取出来的 Option 类名
            'optionClazz' => ScriptOption::class,
            
            // 根 storage 的配置
            'rootStorage' => [
                // 使用 yaml storage 从文件中读取配置
                'meta' => YamlStorageMeta::class,
                
                // 定义获取文件的路径
                'config' => [
                    'path' => __DIR__ . '/resources/stories/',
                    'isDir' => true,
                ],
            ],
            
            // 缓存层的 storage 不一定要有
            'storagePipeline' => [
            ],
        ]));
    }

    public function register()
    {
    }
}

```

## 在 ComponentOption 中注册

可以在```ComponentOption::bootstrap```方法中调用```ComponentOption::loadOptionRepoCategoryMeta```方法配置的分类元数据, 从而在系统启动时完成加载. 这是更为标准的做法.

例如:

```php
/**
 * StoryComponent 是一个情景互动游戏的示范.
 * 可参考本模块开发类似的互动游戏.
 * 也可以封装出基于配置的引擎.
 *
 * ...
 * @property-read MetaHolder $rootStorage 配置文件根仓库的元配置.
 * @property-read MetaHolder[] $storagePipeline 配置文件缓存仓库的元配置.
 *
 */
class StoryComponent extends ComponentOption
{

    public static function stub(): array
    {
        ...
    }


    protected function doBootstrap(): void
    {
        ...

        // 注册所有的脚本配置.
        $data = $this->toArray();
        $this->loadOptionRepoCategoryMeta(
            new CategoryMeta([
                'name' => ScriptOption::class,
                'optionClazz' => ScriptOption::class,
                'rootStorage' => $data['rootStorage'] ?? [],
                'storagePipeline' => $data['storagePipeline'] ?? [],
            ])
        );
    }

}
```

## 定义 Storage 介质

定义一个 ```OptionStorage``` 需要实现两个接口: 

* ```Commune\Support\OptionRepo\Contracts\OptionStorage``` : 定义存取数据的逻辑
* ```Commune\Support\OptionRepo\Options\StorageMeta``` : 定义该介质的可配置参数 

CommuneChatbot 系统自带的存取介质有以下几种:

* Yaml 文件 :
    - storage : ```Commune\Support\OptionRepo\Storage\Yaml\YamlRootStorage```
    - meta : ```Commune\Support\OptionRepo\Storage\Yaml\YamlStorageMeta```
* Json 文件 :
    - storage : ```Commune\Support\OptionRepo\Storage\Json\JsonRootStorage```
    - meta : ```Commune\Support\OptionRepo\Storage\Json\JsonStorageMeta```
* PHP 数组文件 :
    - storage : ```Commune\Support\OptionRepo\Storage\Arr\PHPRootStorage```
    - meta : ```Commune\Support\OptionRepo\Storage\Arr\PHPStorageMeta```
* PHP 静态变量 :
    - storage : ```Commune\Support\OptionRepo\Storage\Memory\MemoryRootStorage```
    - meta : ```Commune\Support\OptionRepo\Storage\Memory\MemoryStorageMeta```