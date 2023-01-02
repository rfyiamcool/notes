## 基于 golang 的消息推送系统 gotify 的设计实现原理

### gotify 架构设计

`Gotify` 是一个基于 websocket 协议并支持多种设备类型接入的消息推送系统，后端使用 golang 开发. gotify 简化了消息的投递及推送的流程，客户端使用 websocket 协议连接并订阅，使用 rest http post 进行消息投递. 

官方的文档里有说这是一个简单的推送系统，在看完代码后发现，发现 gotify 关于推送功能的实现确实有些简单，不适合复杂的线上业务推送场景. gotify 在功能上有一些的缺失，比如它未实现消息回执ack确认，不支持离线消息，不支持按照 revison 来增量推送, 未解决慢消费者引起的阻塞问题等等. 另外他也不是分布式架构，当前属于单点架构.

![https://gotify.net/img/intro.png](https://gotify.net/img/intro.png)

虽然 gotify/server 有些功能缺失，但作者花费时间做开源还是很值得敬佩的，另外还对前端接入及安卓端做了实现，还有文档也较为完整. 😁

社区还有一个使用 golang 开发的推送系统 `ntfy.sh`, ntfy 推送系统的设计跟 gotify 差不多，相比 gotify 多了一个 ios 客户端实现, 社区关注度和文档相比 gotify 都要差一些.

**代码地址:**

- [gotify/server](https://github.com/gotify/server)
- [ntfy](https://github.com/binwiederhier/ntfy)

### 阻塞引发的问题

功能缺失还好，但有一个问题相对严重点. 调用 notify 接口给一个 userID 的所有客户端写消息时，如果某个客户端弱网慢消费, 短时间没法写入到 chan 缓冲, 那么就会陷入阻塞等待. 问题来了，在阻塞期间不仅其他人没法 notify 给其他客户端投递消息, 对于客户端的增删改也都是 hang 住的. 直到客户端 conn 写超时触发 close(write) 唤醒 notify 阻塞端.

但让人疑惑的是，client.write 管道已经关闭，那么继续写入必然 panic, 这个逻辑无疑是个bug，好奇为啥没人反应.

解决阻塞有几个方法:

- 加大 chan 缓冲, 单个缓冲对于高并发不友好.
- 写入 chan 时加入超时, 减少阻塞时长.
- 把数据放到 list 链表里，只使用 chan 做事件通知.
- 把 clients 结构做成多分片，增加性能, 尽减少锁竞争.

```go
type API struct {
	clients     map[uint][]*client
	lock        sync.RWMutex
	pingPeriod  time.Duration
	pongTimeout time.Duration
	upgrader    *websocket.Upgrader
}

func (a *API) Notify(userID uint, msg *model.MessageExternal) {
	a.lock.RLock()
	defer a.lock.RUnlock()
	if clients, ok := a.clients[userID]; ok {
		for _, c := range clients {
			c.write <- msg
		}
	}
}
```

### 推送功能实现

主要就两点，客户端如何监听消息，消息如何投递.

#### 客户端如何监听

websocket 客户端连接服务端后，server 针对该客户端进行协议升级，然后创建一个 client 对象, 然后注册到一个 map 里.

```go
type API struct {
	clients     map[uint][]*client
	lock        sync.RWMutex
	pingPeriod  time.Duration
	pongTimeout time.Duration
	upgrader    *websocket.Upgrader
}

func (a *API) register(client *client) {
	a.lock.Lock()
	defer a.lock.Unlock()
	a.clients[client.userID] = append(a.clients[client.userID], client)
}
```

对 client 对象开两个协程，读和写协程. 读协程只管排空，不做业务逻辑. 写协程就是从 write 管道里监听数据，然后写到 conn 里.

```go
func (a *API) Handle(ctx *gin.Context) {
	conn, err := a.upgrader.Upgrade(ctx.Writer, ctx.Request, nil)
	if err != nil {
		ctx.Error(err)
		return
	}

	client := newClient(conn, auth.GetUserID(ctx), auth.GetTokenID(ctx), a.remove)
	a.register(client)
	go client.startReading(a.pongTimeout)
	go client.startWriteHandler(a.pingPeriod)
}

func (c *client) startReading(pongWait time.Duration) {
	defer c.NotifyClose()
	c.conn.SetReadLimit(64)
	c.conn.SetReadDeadline(time.Now().Add(pongWait))
	c.conn.SetPongHandler(func(appData string) error {
		c.conn.SetReadDeadline(time.Now().Add(pongWait))
		return nil
	})
	for {
		if _, _, err := c.conn.NextReader(); err != nil {
			printWebSocketError("ReadError", err)
			return
		}
	}
}

func (c *client) startWriteHandler(pingPeriod time.Duration) {
	pingTicker := time.NewTicker(pingPeriod)
	defer func() {
		c.NotifyClose()
		pingTicker.Stop()
	}()

	for {
		select {
		case message, ok := <-c.write:
			if !ok {
				return
			}

			c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if err := writeJSON(c.conn, message); err != nil {
				printWebSocketError("WriteError", err)
				return
			}
		case <-pingTicker.C:
			c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if err := ping(c.conn); err != nil {
				printWebSocketError("PingError", err)
				return
			}
		}
	}
}

func (c *client) NotifyClose() {
	c.once.Do(func() {
		c.conn.Close()
		close(c.write)
		c.onClose(c)
	})
}
```

#### 用户怎么投递信息

用户可以通过 http post 接口来投递 message 消息, gotify server 收到消息后进行基本的鉴权和消息格式适配后，把消息再广播给 userID 对应的 ws 客户端.

`一个 userID 可以绑定多个 websocket client.`

```go

g.Group("/").Use(authentication.RequireApplicationToken()).POST("/message", messageHandler.CreateMessage)

func (a *MessageAPI) CreateMessage(ctx *gin.Context) {
	message := model.MessageExternal{}
	if err := ctx.Bind(&message); err == nil {
		application, err := a.DB.GetApplicationByToken(auth.GetTokenID(ctx))
		if success := successOrAbort(ctx, 500, err); !success {
			return
		}
		...
		a.Notifier.Notify(auth.GetUserID(ctx), toExternalMessage(msgInternal))
		ctx.JSON(200, toExternalMessage(msgInternal))
	}
}
```