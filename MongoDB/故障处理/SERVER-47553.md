#MongoDB #Crash
# mongos crashes due to client disconnecting when signing keys being refreshed

## 问题概述

近日，生产一套MongoDB4.2的分片环境出现Mongos Crash的问题，同时日志出现如下信息

```
[conn132492] operation was interrupted because a client disconnected

[conn132492] terminate() called. No exception is active 0x55aa2cd50fe1 0x55aa2cd
```

发现官方在[SERVER-47553]中已经解决，问题的描述如下：

> When constructing a command response, mongos may need to refresh its set of signing keys if the key for the cluster time isn’t currently known. If the client disconnects before this refresh, the resulting error is not properly handled and causes mongos to crash.

原文大意就是mongos在构造命令的响应时，如果客户端的cluster time对应的signing key在mongos本地还没有时，mongos就会去刷新signing keys。如果在刷新完成前，客户端断开连接，由此产生的错误没有进行正确处理就会导致mongos crash。临时的解决方案就是重启config server

## signing key是什么

signing keys是一种hash消息认证码，主要用于对客户端连接的消息进行合法性校验。monogs在回复客户端时会携带该key，后续客户端的请求也需要携带该key，mongos收到key之后会进行校验，如果通过则执行请求，同时根据请求的cluster time推进当前的逻辑时钟。signing keys默认有效期为90天，保存在config server的admin.system.keys集合里面

```
conf:PRIMARY> db.system.keys.find()

{ "_id" : NumberLong("6769505655549067294"), "purpose" : "HMAC", "key" : BinData(0,"QvMiPe8PTTgYRcXMAANFaoiQz5M="), "expiresAt" : Timestamp(1583924359, 0) }

{ "_id" : NumberLong("6769505655549067295"), "purpose" : "HMAC", "key" : BinData(0,"VxoXIXvT1WSj5pcjWzrA9+RWWGY="), "expiresAt" : Timestamp(1591700359, 0) }
```

问题出现的时间正好在第二条的expiresAt之后，当重启config之后会生成新的signing key，因此问题就在于config没有正常生成singing key

## 问题真的解决了么

根据文章[官方CS BUG导致mongos不可用问题定位记录](https://mongoing.com/archives/75350)信息，即使在合并官方path之后，虽然不会出现mongos crash的问题，但还是会出现mongos实例不可用的问题，因此问题并没有真正的被解决，具体原因可以看看该文章。最后官方在4.2.10的版本中才真正解决该问题，因此建议升级该版本或更高版本，避免受该问题影响。

