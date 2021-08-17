## 为 golang GRPC 配置 SAN 证书

### 被干掉的 X509

在 golang grpc tls 里不能直接使用 X509 生成的证书, 不然会报如下的错误.

```
rpc error: code = Unavailable desc = connection error: desc = "transport: authentication handshake failed: x509: certificate relies on legacy Common Name field, use SANs or temporarily enable Common Name matching with GODEBUG=x509ignoreCN=0"
```

原因在于 go1.15 之后不在支持 x509 的加密套件，需要改用 SAN 证书.

当然, 如果不想改用 SAN 证书, 可以加入一个GODEBUG环境变量, 命令如下:

```
GODEBUG=x509ignoreCN=0 go run xxx.go

or

export GODEBUG="x509ignoreCN=0"
go run xxx.go
```

### 生成 SAN 证书

#### 1. 安装 openssl

```bash
yum -y install openssl
```

#### 2. 生成普通的key:

```bash
openssl genrsa -des3 -out server.key 2048
```

输入密码，后面要用到.

#### 3. 生成ca的crt：

```bash
openssl req -new -x509 -key server.key -out ca.crt -days 3650
```

#### 4. 生成csr

```bash
openssl req -new -key server.key -out server.csr
```

#### 5. 配置 openssl.cnf 文件

* 把系统的 openssl.cnf 拷贝到当前目录.
* 找到 [ CA_default ], 打开 copy_extensions = copy
* 找到[ req ],打开 req_extensions = v3_req # The extensions to add to a certificate request
* 找到[ v3_req ],添加 subjectAltName = @alt_names
* 添加新的标签和标签字段

```bash
[ alt_names ]
DNS.1 = *.org.haha.com
DNS.2 = *.xiaorui.cc
DNS.3 = *
```

`*` 代表任意ServerName.

#### 6. 生成证书私钥test.key：

```bash
openssl genpkey -algorithm RSA -out test.key
```

#### 7. 通过私钥test.key生成证书请求文件test.csr：

```bash
openssl req -new -nodes -key test.key -out test.csr -days 3650 -subj "/C=cn/OU=myorg/O=mycomp/CN=myname" -config ./openssl.cnf -extensions v3_req
```

#### 8. 文件介绍

test.csr是上面生成的证书请求文件, ca.crt/server.key是CA证书文件和key, 用来对test.csr进行签名认证.

#### 9. 生成SAN证书：

```
openssl x509 -req -days 365 -in test.csr -out test.pem -CA ca.crt -CAkey server.key -CAcreateserial -extfile ./openssl.cnf -extensions v3_req
```

#### 10. 在代码中使用 SAN 证书

服务器加载代码:

```
creds, err := credentials.NewServerTLSFromFile("test.pem", "test.key")
```

客户端加载代码:

```
creds,err := credentials.NewClientTLSFromFile("test.pem", "*.xiaorui.cc")
```

### refer

- [https://blog.csdn.net/cuichenghd/article/details/109230584](https://blog.csdn.net/cuichenghd/article/details/109230584)
- [https://www.cnblogs.com/jackluo/p/13841286.html](https://www.cnblogs.com/jackluo/p/13841286.html)
- [https://www.796t.com/article.php?id=249208](https://www.796t.com/article.php?id=249208)

