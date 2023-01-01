## 源码分析 kubernetes ingress nginx 动态更新的实现原理

阅读本文前, 建议先阅读下 k8s ingress nginx controller 实现原理.

[kubernetes_ingress_nginx_controller_code](https://github.com/rfyiamcool/notes/blob/main/kubernetes_ingress_nginx_controller_code.md)

#### 动态更新的函数调用流程:

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301011133932.png)

#### 概述

通过 ingress-nginx 中 `nginx.tmpl` 的配置得知, nginx (openretry) 转发的逻辑是依赖 upstream 的 balancer_by_lua_block 指令实现的.

http 和 stream (tcp/udp) 在生成配置时, 在 upstream 段里都插入了 `balancer_by_lua_block` 指令用来实现自定义负载均衡逻辑, nginx 会依赖该 balancer 来获取转发的地址, 然后对该连接进行转发.

> 该 lua 转发模块代码位置是 `rootfs/etc/nginx/lua/balancer.lua`.

### balancer_by_lua_block 是怎么回事 ?

`balancer_by_lua_block` 是一个支持自定义负载均衡器的指令, 通常基于 nginx 的服务发现就是通过该指令实现的.

开发时一定要注意事项, balancer_by_lua_block 只是通过自定义负载均衡算法获取 peer 后端地址, 接着通过 `balancer.set_current_peer(ip, port)` 进行赋值. 后面连接的建立，连接池维护，数据拷贝转发等流程统统不在这里，而是由 nginx 内部 upstream 转发逻辑实现.

一句话，nginx 只是调用 `balancer_by_lua_block` 获取理想的后端地址而已.

下面是使用 `balancer_by_lua_block` 实现调度地址池的例子:

```c
upstream xiaorui_cc_backend {
    server 0.0.0.0;
    
    balancer_by_lua_block {
        local balancer = require "ngx.balancer"
        local host = {"s1.xiaorui.cc", "s2.xiaorui.cc"}
        local backend = ""
        local port = ngx.var.server_port
        local remote_ip = ngx.var.remote_addr
        local key = remote_ip..port
        
        # 使用地址 hash 调度算法
        local hash = ngx.crc32_long(key);
        hash = (hash % 2) + 1
        backend = host[hash]
        ngx.log(ngx.DEBUG, "ip_hash=", ngx.var.remote_addr, " hash=", hash, " up=", backend, ":", port)
        
        # 配置后端地址, nginx 进行转发时依赖该地址
        local ok, err = balancer.set_current_peer(backend, port)
        if not ok then
            ngx.log(ngx.ERR, "failed to set the current peer: ", err)
            return ngx.exit(500)
        end
    }
}

server {
	listen 80;
	server_name xiaorui.cc
	location / {
		proxy_pass http://xiaorui_cc_backend;
	}
}
```

lua-nginx-module 项目中关于 balancer_by_lua_block 实现:

[https://github.com/openresty/lua-nginx-module#balancer_by_lua_block](https://github.com/openresty/lua-nginx-module#balancer_by_lua_block)

### 在 nginx 里加入 balancer_by_lua_block 指令

在 `nginx.tmpl` 中加入了 `balancer_by_lua_block` 指令, 所以不管是 http 和 stream 段里的 upstream 转发, 不再走 server 配置, 而是走 `balancer_by_lua_block` 自定义流程.

```c
http {
    upstream upstream_balancer {
	    // 只是占位符, openretry 优先走 balancer_by_lua 逻辑块.
        server 0.0.0.1; # placeholder

        balancer_by_lua_block {
          balancer.balance()
        }

        {{ if (gt $cfg.UpstreamKeepaliveConnections 0) }}
        keepalive {{ $cfg.UpstreamKeepaliveConnections }};
        keepalive_time {{ $cfg.UpstreamKeepaliveTime }};
	...
        {{ end }}
    }

    ...

    server {
	...
    }
}

stream {
    upstream upstream_balancer {
	    // 同上, 只是占位符, 避免 nginx -t 检测出错.
        server 0.0.0.1:1234; # placeholder

        balancer_by_lua_block {
          tcp_udp_balancer.balance()
        }
    }

    ...

    server {
	...
    }
}

```

### 把变更信息通知给 nginx

- 检查 http backends 是否有变更, 当有变更时, 把 backends 数据通知给 nginx 的 `http://127.0.0.1:10246/configuration/backends` 接口上.
- 检查 tcp/udp strem backends 是否有变更, 发生变更时, 把 stream backends 数据发到 nginx 的 tcp `10247` 端口上.
- 当证书发生变更时, 发数据发到 nginx 的 `http://127.0.0.1:10246/configuration/servers` 接口上.

源码位置: [https://github.com/kubernetes/ingress-nginx/blob/4508493dfe7fb06206109a0a9dcc6025cc335273/internal/ingress/controller/nginx.go](https://github.com/kubernetes/ingress-nginx/blob/4508493dfe7fb06206109a0a9dcc6025cc335273/internal/ingress/controller/nginx.go)

```go
func (n *NGINXController) configureDynamically(pcfg *ingress.Configuration) error {
	backendsChanged := !reflect.DeepEqual(n.runningConfig.Backends, pcfg.Backends)
	// 当 endpoints 地址发生变更时
	if backendsChanged {
		// 动态修改 http 的 backends
		err := configureBackends(pcfg.Backends)
		if err != nil {
			return err
		}
	}

	streamConfigurationChanged := !reflect.DeepEqual(n.runningConfig.TCPEndpoints, pcfg.TCPEndpoints) || !reflect.DeepEqual(n.runningConfig.UDPEndpoints, pcfg.UDPEndpoints)
	// 当 endpoints 地址发生变更时
	if streamConfigurationChanged {
		// 动态修改 tcp 和 udp 的 backends 地址列表
		err := updateStreamConfiguration(pcfg.TCPEndpoints, pcfg.UDPEndpoints)
		if err != nil {
			return err
		}
	}

	serversChanged := !reflect.DeepEqual(n.runningConfig.Servers, pcfg.Servers)
	// 当 servers 地址发生变更时
	if serversChanged {
		// 动态修改证书相关配置
		err := configureCertificates(pcfg.Servers)
		if err != nil {
			return err
		}
	}

	return nil
}
```

这里拿 `configureBackends()` 变更配置来说. 组装 openresty 专用的 backends 数据, 然后序列化成 json, post 发给 openresty 的 `/configuration/backends` 接口上.

```go
func configureBackends(rawBackends []*ingress.Backend) error {
	backends := make([]*ingress.Backend, len(rawBackends))

	for i, backend := range rawBackends {
		luaBackend := &ingress.Backend{
			...
		}

		var endpoints []ingress.Endpoint
		for _, endpoint := range backend.Endpoints {
			endpoints = append(endpoints, ingress.Endpoint{
				Address: endpoint.Address,
				Port:    endpoint.Port,
			})
		}

		luaBackend.Endpoints = endpoints
		backends[i] = luaBackend
	}

	statusCode, _, err := nginx.NewPostStatusRequest("/configuration/backends", "application/json", backends)
	if err != nil {
		return err
	}

	if statusCode != http.StatusCreated {
		return fmt.Errorf("unexpected error code: %d", statusCode)
	}

	return nil
}

func NewPostStatusRequest(path, contentType string, data interface{}) (int, []byte, error) {
	url := fmt.Sprintf("http://127.0.0.1:%v%v", StatusPort, path)
	buf, err := json.Marshal(data)
	...

	res, err := client.Post(url, contentType, bytes.NewReader(buf))
	...

	body, err := io.ReadAll(res.Body)
	...
	return res.StatusCode, body, nil
}
```

### nginx 如何接收需要动态更新的配置 ？

上面代码是如何发送变更信息, 那么谁来接收动态数据的投递? 

`nginx.conf` 中定义了一个解决动态配置更新的 server 配置段, 其中变量 StatusPort 为 10246, 接口的 prefix 路径为 `/configuration`, 该接口定义了 content_by_lua_block 处理块.

当接口收到请求后, 调用自定义 lua 模块 `configuration.lua` 中 `configuration.call()` 入口方法.

```c
    server {
        listen 127.0.0.1:{{ .StatusPort }};

        keepalive_timeout 0;
        gzip off;

        access_log off;

        location {{ $healthzURI }} {
            return 200;
        }

        location /configuration {
            client_max_body_size                    {{ luaConfigurationRequestBodySize $cfg }};
            client_body_buffer_size                 {{ luaConfigurationRequestBodySize $cfg }};
            proxy_buffering                         off;

            content_by_lua_block {
              configuration.call()
            }
        }
    }
```

下面分析 `configuration.call()` 的实现原理. `call()` 中硬编码写了各个接口的处理方法.

当 `ngx.var.request_uri` 为 `/configuration/backends` 时候, 调用 `handle_backends` 方法处理该路由.

`handle_backends` 内部实现过程很简单, 先解析 request body, 然后把读到的 body 字符串放到共享存储 `configuration_data` 的 backends 键里, 然后更新下操作的时间戳.

`configuration_data` 是一个 ngx.shared.Dict 共享内存的字典存储结构, 其 set/get 操作是并发安全的. nx.shared.dict 内部通过红黑树实现的 hashmap, 使用 lru 实现的数据淘汰.

configuration_data:set 的时候没有 cjson 解析对象, 而是直接赋值json string.

```lua
function _M.call()
  if ngx.var.request_method ~= "POST" and ngx.var.request_method ~= "GET" then
    ngx.status = ngx.HTTP_BAD_REQUEST
    return
  end

  # 处理证书的 servers 
  if ngx.var.request_uri == "/configuration/servers" then
    handle_servers()
    return
  end

  # 处理通用配置 general
  if ngx.var.request_uri == "/configuration/general" then
    handle_general()
    return
  end

  # 处理证书的 http handler
  if ngx.var.uri == "/configuration/certs" then
    handle_certs()
    return
  end

  # 处理 backends http handler
  if ngx.var.request_uri == "/configuration/backends" then
    handle_backends()
    return
  end

  ngx.status = ngx.HTTP_NOT_FOUND
  ngx.print("Not found!")
end

local function handle_backends()
  # 获取当前 nginx 内的 backends 配置
  if ngx.var.request_method == "GET" then
    ngx.status = ngx.HTTP_OK
    ngx.print(_M.get_backends_data())
    return
  end

  # 读取 request body
  local backends = fetch_request_body()
  if not backends then
    ngx.status = ngx.HTTP_BAD_REQUEST
    return
  end

  # 把 backends 放到 ngx.shared 的 configuration_data 存储的 backends 键值里.
  local success, err = configuration_data:set("backends", backends)
  if not success then
    ngx.status = ngx.HTTP_BAD_REQUEST
    return
  end

  # 记录更新时间
  ngx.update_time()
  local raw_backends_last_synced_at = ngx.time()
  success, err = configuration_data:set("raw_backends_last_synced_at", raw_backends_last_synced_at)
  if not success then
    ngx.status = ngx.HTTP_BAD_REQUEST
    return
  end

  ngx.status = ngx.HTTP_CREATED
end
```

### nginx 内部如何解析同步配置 ?

#### init_worker

上面的 `handle_backends()` 只是从 http server 里获取请求的 json body 字符串, 然后把字符串写到 ngx.shared.dict 存储里. 那么谁来读取 ? 谁来 json decode ? 

控制器在 nginx.conf 配置文件中加入了 `init_worker_by_lua_block` 初始化块, 所以当 nginx 启动时会调用 `balancer.init_worker` 进行模块初始化.

先异步执行 sync_backends_with_external_name, 同步 service 类型为 external_name 的配置, 然后每隔一秒调用一次 sync_backends 和 sync_backends_with_external_name.

代码如下: `rootfs/etc/nginx/lua/balancer.lua::init_worker()`

```lua
function _M.init_worker()
  # 通过定时器实现异步执行 sync_backends_with_external_name
  local ok, err = ngx.timer.at(0, sync_backends_with_external_name)
  if not ok then
    ngx.log(ngx.ERR, "failed to create timer: ", err)
  end

  # 每秒调用一次 sync_backends
  ok, err = ngx.timer.every(BACKENDS_SYNC_INTERVAL, sync_backends)
  if not ok then
    ngx.log(ngx.ERR, "error when setting up timer.every for sync_backends: ", err)
  end

  # 每秒调用一次 sync_backends_with_external_name
  ok, err = ngx.timer.every(BACKENDS_SYNC_INTERVAL, sync_backends_with_external_name)
  if not ok then
    ngx.log(ngx.ERR, "error when setting up timer.every for sync_backends_with_external_name: ",
            err)
  end
end
```

#### sync_backends

`sync_backends()` 被定时器周期性调度, 从 `ngx.shared.dict` 获取 backends 数据, 反序列化后, 遍历所有的 backend 对象, 依次调用 `sync_backend` 来向 balancer 同步配置.

```lua
local function sync_backends()
  # 从 ngx.shared.dict 获取 backends 键值数据
  local backends_data = configuration.get_backends_data()
  ...

  # 把 json string 进行反序列为 json object 对象
  local new_backends, err = cjson.decode(backends_data)
  ...

  # 通过 sync_backend() 处理 backend 对象
  local balancers_to_keep = {}
  for _, new_backend in ipairs(new_backends) do
    if is_backend_with_external_name(new_backend) then
      ...
    else
      # 向 balancer 同步配置
      sync_backend(new_backend)
    end
    balancers_to_keep[new_backend.name] = true
  end
end

local function sync_backend(backend)
  # 如果 endpoints 为空, 则跳出.
  if not backend.endpoints or #backend.endpoints == 0 then
    balancers[backend.name] = nil
    return
  end

  # 简化 endpoints 数据结构, 把复杂的 struct 转成 lua table 数组.
  backend.endpoints = format_ipv6_endpoints(backend.endpoints)

  local implementation = get_implementation(backend)
  local balancer = balancers[backend.name]

  # 该 name 没有 balancer 对象, 就创建一个
  if not balancer then
    balancers[backend.name] = implementation:new(backend)
    return
  end

  # 调用 balancer 的 sync 方法, 把 backend.endpoints 更新到 peers 对象里.
  balancer:sync(backend)
end
```

#### ingress-nginx 的负载均衡算法 ?

`ingress-nginx` 当前实现了下面几种负载均衡算法. 默认算法为 `round_robin` 轮询. 这些 lua 算法模块位置在 `rootfs/etc/nginx`. 可以发现负载均衡算法中没有 `least_conn` 最少连接算法, 也没有 `p2c` (自适应负载均衡算法) 算法.

```c
sticky_persistent.lua
sticky_balanced.lua
sticky.lua
round_robin.lua
resty.lua
ewma.lua
chashsubset.lua
chash.lua
```

#### 为什么不能使用 nginx 自带的负载均衡算法 ?

为什么没有使用 nginx 丰富的负载均衡策略, 而是在 lua 中实现 ? 这是因为 ingress-nginx 是通过 `balancer_by_lua` 实现的地址池的负载均衡, 这已经跟 nginx upstream balancer 冲突了.

另外 `balancer_by_lua` 内部没有实现健康检查, 失败地址的剔除是通过 ingress-nginx 的 informer 实现的. 当监听到 apiserver 有地址发生变更, 则需要调用 `configureDynamically` 来通知 nginx 更新.

### 为什么需要动态更新 upstream backends ? 

🤔 思考中...

不管是 nginx 和 openresty 都只支持配置的 reload 热加载, 不支持动态更新的. 但社区中基于 openresty 的 kong 和 apisix 都支持多源的动态更新配置, 另外社区中也有支持动态更新的 lua 模块可以使用. 

#### nginx reload 优雅热加载带来的问题 ?

当 nginx 作为 ingress 角色时, 遇到频繁变更 service endpoints 的场景下, nginx reload 开销不会小的, 每次都需要 new worker 及 kill old worker, 旧 worker 的长请求不断又是个问题. 新 worker 是新的子进程没法继承旧 worker 的连接池, 所以需要重新建连连接和维护 upstream 连接池, 这都会影响性能和时延 latency.

如果 nginx 不支持动态更新, 在一个大集群的的上下线会引发 ingress-nginx 不断的 reload.

当在 ingress-nginx 支持 upstream 和证书的动态更新后, 新配置加载的开销会小很多. 只需要把更新的配置通知给 openresty 的动态配置接口就可以了. balancer.lua 模块会维护每个 backend 地址池的负载均衡逻辑.

#### Apisix 架构设计

apisix 对云原生环境适配比 nginx/kong 更加的友好, 虽然大家都是基于 nginx/openresty 开发, 但 apisix 实现了更好的动态配置更新, 更好的插件性能及控制面板. apisix 也提供了 ingress controller 控制器, 建议大家可以用用王院长和温铭出名的 apisix.

apisix 控制的的项目一直由社区的张晋涛推广, 看过一部分 apisix 和 controller 代码实现, 因为有 apisix 的云原生特性加持, 所以 `apisix-ingress-controller` 的实现显得更干练一些, 没有 ingress-nginx 那么的绕. 

[apisix-ingress-controller 项目地址](https://github.com/apache/apisix-ingress-controller)

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301011512369.png)

## 总结 

ingress-nginx 的实现原理就是控制器监听 kube-apiserver 资源更新, 当有资源更新时, 通过 nginx 配置模板生成配置文件, 然后 reload 热加载的过程. 

为了应对 kubernetes endpoints 变更引发的 nginx 频繁 reload,  所以 ingress-nginx 在 nginx 里使用 lua 实现了配置热更新的功能, 主要是针对地址池构建各负载算法的 balancer, 当 nginx location upstream 进行转发钱, 先从 balancer_by_lua 里获取转发的后端地址, 然后 nginx 再对该后端地址进行转发.  