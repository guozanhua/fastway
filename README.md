
前端协议
=======

上下行：

```
+------------------------+---------------------+---------+
| Packet Length (4 byte) | Backend ID (8 byte) | Message |
+------------------------+---------------------+---------+
```

特殊约定：

0. `Packet Length` = 8 + len(`Message`)
1. 客户端发送 `{Packet Length = 0, Backend ID = N}` 的消息，用来显式关闭跟指定后端之间端虚拟连接
2. 网关发送 `{Packet Length = 0, Backend ID = N}` 的消息，用来告知客户端某个**使用过的**后端节点失效
3. 网关之发送 `{Packet Length = 0, Backend ID = MAX(uint64)}` 的消息，用来进行检查客户端连接是否存活
4. 客户端发送 `{Packet Length = 0, Backend ID = MAX(uint64)}` 的消息，用来回应网关的检查


后端协议
=======

握手过程：

0. 网关和后端都持有一个共同的握手验证秘钥
1. 网关在接受到新的后端连接之后，向新连接发送一个`uint64`范围内的随机数作为挑战码

	```
	+-------------------------+
	| Challenge Code (8 byte) |
	+-------------------------+
	```

2. 后端收到挑战码后，拿出秘钥，计算 `MD5(挑战码 + 秘钥)`，得到验证码
3. 后端将验证码和后端节点ID一起发送给网关

	```
	+-------------------------------------+
	| MD5 (16 byte) | Backend ID (8 byte) |
	+-------------------------------------+
	```
	
4. 网关收到消息后同样计算 `MD5(挑战码 + 秘钥)`，跟收到的验证码比对是否一致
5. 验证码比对一致，网关将新连接登记为对应节点ID的连接

上下行：

```
+---------------+------------------------+--------+
| Type (1 byte) | Packet Length (4 byte) | Packet |
+---------------+------------------------+--------+
```

消息类型和对应消息内容格式：

+ Type = 0，发送给指定客户端

	```
	+---------------------+---------+
	| Session ID (8 byte) | Message |
	+---------------------+---------+
	```

+ Type = 1，断开指定客户端

	```
	+---------------------+
	| Session ID (8 byte) |
	+---------------------+
	```

+ Type = 2，PING / PONG

优化细节
=======

网关为每个客户端连接分配一个`Receive Buffer`，只有当`Receive Buffer`容量不足以装下将要读取的消息时才会重新申请内存。

对于同一个客户端，网关每次只为其转发一个消息，消息还没转发成功时不会读取后续消息，所以`Receive Buffer`是被顺序使用的，并且可以被重复使用。

协议设计时有意的对齐前端上行消息和后端上行消息格式，后端上行消息只多了头部一个字节的消息类型字段：

```
                +------------------------+---------------------+---------+
                | Packet Length (4 byte) | Backend ID (8 byte) | Message |
                +------------------------+---------------------+---------+

+---------------+------------------------+---------------------+---------+
| Type (1 byte) | Packet Length (4 byte) | Session ID (8 byte) | Message |
+---------------+------------------------+---------------------+---------+
```

在读取前端上行消息时，在`Receive Buffer`头部预留一个字节，读取完整前端消息后，将`Receive Buffer`中的`Backend ID`填充为当前客户端的`Session ID`就可以直接作为后端上行消息转发出去，不需要再多一次数据拷贝。
