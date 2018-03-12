## WebSocket学习笔记
wss://echo.websocket.org 我能够websocket连接   
但是不能连接wss://127.0.0.1:80 与wss://127.0.0.1:8080    

然后我发现ws://127.0.0.1可以正常连接  
显然是wss 是 websocket security 类似于http和https，   
不能互相连    

设置var connection = request.accept('echo-protocol',request.origin);时无法交互，因协议不对应(echo-protocol)，改为request.accept(null,request.origin)即可，猜测为所有协议均能交互   

现在我们前端显示不了图片，前端img标签设置不对？得到的数据类型是blob？   
现在成功了，图片src赋值为blob即可显示图片     

https://stackoverflow.com/questions/46557485/difference-between-ws-and-wss ws与wss的区别   
https://stackoverflow.com/questions/6182315/how-to-do-base64-encoding-in-node-js node的base64  
http://blog.csdn.net/kkkkk4400/article/details/16980647 js操作二进制   
https://developer.mozilla.org/en-US/docs/Web/API/FileReader filereader     
https://developer.mozilla.org/en-US/docs/Web/API/Blob blob