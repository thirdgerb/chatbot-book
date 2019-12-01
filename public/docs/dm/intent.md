# Intent

意图 (intent) 是机器人理解用户输入的最基本抽象. 无论用户输入的是语音, 文字, 图片, 表情, 链接, 手势... 都可以抽象为 "意图".

在 CommuneChatbot 项目中, 用 "intentName" + "entities" ( 意图名 + 实体值 ) 来描述单一意图. 本质上类似于```方法名 + 参数```, 是人类对机器人下达指令的最小完整单元.

与浏览器的表单, shell 里的命令行不同, 作为意图参数的 "entities" 往往需要经过多轮对话来完善. 所以意图被封装成为```Commune\Chatbot\OOHost\Context\Intent\IntentMessage```对象, 既是一个输入消息 ```Message```, 又是一个上下文单元```Context```.