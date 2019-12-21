# Cache

作为多轮对话机器人框架, CommuneChatbot 并不干涉具体业务的实现, 开发者可以通过[服务注册](/docs/engineer/di.md) 机制定义所需的缓存模块, 数据库模块等.

而 CommuneChatbot 自身的一些功能也依赖缓存. 例如用户发来消息后, 对 Chat 加锁阻塞, 防止分布式系统上同时收到多个用户消息, 导致 "裂脑" 的现象.

因此系统必须要注册 ```Commune\Chatbot\Contracts\CacheAdapter``` 服务以启动, 具体 API 可以查看接口文件.

CacheAdapter 可以通过方法 ```CacheAdapter::getPSR16Cache()``` 获取一个[psr-16](https://www.php-fig.org/psr/psr-16/) 的 ```Psr\SimpleCache\CacheInterface``` 实例.
它与 psr-16 主要的区别在于, 需要定义 ```CacheAdapter::lock()```  与 ```CacheAdapter::unlock()``` 方法. 未来还可能加入计数器.




