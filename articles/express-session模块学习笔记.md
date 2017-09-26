#express-session模块学习笔记(init版)

---
架构：
+ session：创建session对象
  + cookie：创建cookie对象，挂载到session对象
+ store：单件，所有session共用一个，由req.sessionStore暴露store
+ memory：继承自store，用于具体存储

##session(主函数):
传入options对象，返回一个函数（同样名为session）  
store = new MemoryStore  
初始化store.generate

####session逻辑结构（返回的函数，中间件函数）
脉络：挂载sessionID，session对象到req对象上
+ 初始化判断
+ 初始化设置
+ 重置res.end，调用res.send等时会运行此函数
+ 若session不存在则生成一个session
+ 若session存在则将session对象挂在到req对象上，供后续代码使用

满足以下某一个条件，则直接next()
+ req.session存在
+ storeReady为非（此值由disconnect与connect事件影响，store.on注册 ）
+ pathname匹配失败
+ secret未设置


req.sessionStore = store; (store为对象)   
rawCookie = req.cookies[key] (key为session使用的关键字)   
若unsignedCookie不存在，rawCookie存在，则设置unsignedCookie(与rawCookie关联)   

重置res.writeHead  
**重置res.end**:  
运行此函数后res.end复原，express框架下res.send最终都会调用res.end   
若session存在，则req.session.resetMaxtAge();  
req.session.save(function)  
(即req.sessionStore.sessions[sid]=sessJSON形式，store为全局初始化创建，所有session共用同一个store)  
function为无错误则res.end（发送数据）


req.sessionID = unsignedCookie;  
如果req.sessionID为空，则生成session，并next();  

运行store.get(req.sessionID,function)生成session object  
**function**:  
将sess对象挂载到req对象上  

+ 存在err：next(err);  
+ 无err，sess不存在：generate();next;  
+ 无err，sess存在：  
运行store.createSession(req,sess);  
originalID = req.sessID;  
originalHash = hash(sess);  
next();  

---
##session.js:
####Session:
生成一个新对象，Object.req = req，Object.id = req.sessionID  

####resetMaxAge:  
更新session.cookie.maxAge为originalMaxAge(创建cookie对象时设置，option.cookie对象中设置数值)  

####save:
调用this.req.sessionStore.set(this.id,fn);
return this;

---

##cookie.js
####Cookie:
传入cookie(配置对象)，初始化配置

---

##store.js
####Store:
空的

####createSession:
重新加载配置，将新生成session对象挂在到req上  
sess.cookie  
req.session  
sess.cookie.originalMaxAge  
sess.cookie.expires  

返回req.session

---

##memory.js(继承自store)
####MemoryStore:
创建新对象，设置对象sessions为空
####MemoryStore.prototype.get:
通过sessionID获得 一个sess  

传入req.sessionID和一个函数  
sess = this.sessions[req.sessionID]   
若sess不存在则直接，fn()  

否则：  
sess由JSON形式解码
expires = sess.cookie.expires(这是一个时间)  

若expires不存在或当前时间小于expires  
则fn(null,sess)  

否则this.destroy(req.sessionID,fn)  

####MemoryStore.prototype.set:
存储一个session

this.sessions[sessionID] = sess转化为JSON形式