# 常见问题

本篇文档记录常见问题的回答. 如有问题, 欢迎在 [issue](https://github.com/thirdgerb/chatbot/issues) 提出, 或者加入 QQ 群: 907985715

<h3>为什么使用 PHP 来开发? </h3>

项目还在开发阶段, 就多次遇到有朋友问为什么使用 PHP 来开发.
因此开辟一个专门的页面来回答 :)

最根本的原因, 还是作者本人最熟练的编程语言是 PHP, 用它来验证多轮对话引擎的思路最快.
使用其它语言, 在用到高级特性时有一个爬坡的成本. 在各种功能模块上更难达到最佳实践.

此外, 设计项目时也做过语言的调研. 结论是 PHP 胜任并适合对话机器人项目 :

* 验证对话引擎原型, 需要随时开发随时调试, 编译型语言不如脚本语言方便.
* 对话本基于文本, 本身是弱类型场景. 频繁类型转换带来较高开发成本, 语言最好支持弱类型.
* 设计思路结合了面向对象, 与函数式的风格. PHP 语法对两种都有不错的兼容.
* 考虑 PHP 语言流行度, 所有平台 (公众号, 智能音箱) 等都有对接 SDK
* PHP 的功能类库生态比较成熟
* Swoole + Hyperf 提供协程服务端, 性能优异

PHP 用于对话引擎最大的不足, 在于缺乏自然语言相关生态,
与 NLP 相关的功能只好调用第三方 API.
不过作为一个多轮对话管理系统, 本来 NLU 就应该解耦为微服务, 所以也不是问题.