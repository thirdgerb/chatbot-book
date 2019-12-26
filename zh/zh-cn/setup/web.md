# 搭建 Web bot

Web 版对话机器人, 通常用 JS 开发的浏览器应用作为前端, 通过 http 与服务端通信.
使用无状态的 Http 协议, 机器人的逻辑会相对简单一些.
更好的做法是用 WebSocket 通信, 这样的机器人可以拥有双工的能力.

CommuneChatbot 开箱自带的 Web 机器人是基于 Http 的.
组件是 ```Commune\Platform\Web\WebComponent```.
它给出了 Web 版机器人的一个示范.
用类似的思路, 可以将对话机器人用于网页上的任何对话界面中.

## 编译前端界面

CommuneChatbot 自带的 Web 机器人, 前端代码在 ```BASE_PATH/app-frontend``` 下.

需要进入该目录, 先安装依赖.

```
npm install
```

可以测试编译代码 :

```
npm run dev
```

也可以监视代码, 改动后自动编译 :

```
npm run watch
```

生成生产级的代码, 请使用 :

```
npm run prod
```

生成的前端资源会进入 ```BASE_PATH/app-frontend/public/``` 目录.
可以通过 Nginx 将该目录对外暴露.

前端工程主要使用了以下项目:

- [vue](https://vuejs.org/) : 大名鼎鼎的 MVVM 框架
- [vuetify](https://vuetifyjs.com/) : 非常优秀的 martial design 风格界面
- [axios](https://github.com/axios/axios) : 处理 http 请求
- [Bubble](https://github.com/dmitrizzle/chat-bubble) : 用来做对话交互效果
- [LaraveMix](https://laravel-mix.com/) : 简化 Webpack 的配置

具体效果请查看 [官方网站](https://communechatbot.com).

这个前端工程是为示范的目的, 作者用几天时间临时开发的.
如有问题烦请反馈.
在生产环境中使用的 Web 对话机器人, 建议参考本项目的实现, 另行专门开发.
