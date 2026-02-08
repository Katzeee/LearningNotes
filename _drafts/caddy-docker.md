我需要用caddy docker去反代我的unraid，我现在的设置是这样，好像无法成功，我的unraid.example.com这个域名只有AAAA的解析，没有A的解析，我应该怎么设置，我只需要使用ipv6能访问即可，没有ipv4地址，证书是已经申请成功了，只是反代好像一直没成功，上面这个4443的一直无法成功访问，下面这个5555的一切正常

unraid.example.com:4443 {
        reverse_proxy 192.168.1.200:8080 {
                flush_interval -1

                header_up X-Real-IP {remote_host}
                header_up X-Forwarded-For {remote_host}
                header_up X-Forwarded-Proto {scheme}

                transport http {
                        read_timeout 300s
                        write_timeout 300s
                        dial_timeout 30s
                }
        }

        header {
                Strict-Transport-Security "max-age=31536000; includeSubDomains"
                X-Frame-Options "DENY"
                X-Content-Type-Options "nosniff"
                -Server
        }
}

unraid.example.com:5555 {
        reverse_proxy 127.0.0.1:3005 {
                flush_interval -1

                header_up X-Real-IP {remote_host}
                header_up X-Forwarded-For {remote_host}
                header_up X-Forwarded-Proto {scheme}

                transport http {
                        read_timeout 300s
                        write_timeout 300s
                        dial_timeout 30s
                }
        }

        header {
                Strict-Transport-Security "max-age=31536000; includeSubDomains"
                X-Frame-Options "DENY"
                X-Content-Type-Options "nosniff"
                -Server
        }
}




你这个场景里 **AAAA-only（只有 IPv6）本身不会导致 Caddy 反代失败**：Caddy 只要你的域名 **A 或 AAAA** 指到这台机器就行，不要求两者都存在。([Caddy Web Server][1])
所以问题更可能出在 **4443 端口“外部根本没打到 Caddy”**（端口映射/占用/IPv6 防火墙/上游不可达）——因为同域名的 5555 能正常，说明 DNS / 证书 / IPv6 通路大概率是通的。

下面按“最常见原因 → 你最快能定位”的顺序给你一套排查/修复清单：

---

## 1) 先确认：4443 这个端口到底有没有对外开放（IPv6 防火墙最常见）

IPv6 没有 NAT 的“端口转发”概念，**很多路由器/防火墙默认会挡入站**，你现在很像是 **只放行了 5555，没放行 4443**。

**外网（IPv6）测试：**

* 用手机 5G（有 IPv6）或外部 VPS 运行：

  * `nc -6vz unraid.example.com 4443`
  * `curl -6vk https://unraid.example.com:4443/`

如果这里是 **timeout / no route / filtered**，那就不是 Caddy 配置问题，而是：

* 路由器的 IPv6 入站防火墙规则放行 TCP 4443
* 服务器本机（或 Unraid 上的安全策略/插件防火墙）放行 TCP 4443

---

## 2) 确认 Docker 端口映射：你可能根本没把 4443 映射出来，或被占用导致绑定失败

在 Unraid 的 Docker 页面里，检查 Caddy 容器是否同时暴露了：

* Host Port `4443` → Container Port `4443`（TCP）
* Host Port `5555` → Container Port `5555`（TCP）

在命令行也可以看：

* `docker ps --format 'table {{.Names}}\t{{.Ports}}' | grep -i caddy`

另外确认宿主机有没有进程占用 4443：

* `ss -lntp | grep 4443`

> 你现在“5555 正常、4443 不通”最符合 **4443 没映射出来** 或 **被占用/被防火墙挡** 这两类。

---

## 3) 如果你在用 Cloudflare 橙云代理：4443/5555 都不在支持端口里（这点非常关键）

如果你的 `unraid.example.com` 是 **Cloudflare 橙云（Proxied）**，Cloudflare 只代理固定的一组 HTTP/HTTPS 端口；**4443 不在 HTTPS 支持列表里**，因此会直接失败。Cloudflare 支持的 HTTPS 端口包含 `443、2053、2083、2087、2096、8443` 等，但**不包含 4443/5555**。([Cloudflare Docs][2])

解决办法二选一：

1. 把这个 DNS 记录改成 **灰云（DNS only）**，让客户端直连你的 IPv6
2. 把端口改成 Cloudflare 支持的，例如把 4443 改成 **8443**（HTTPS 支持）([Cloudflare Docs][2])
   然后 Caddyfile 用：

   ```caddy
   unraid.example.com:8443 {
       reverse_proxy 192.168.1.200:8080
   }
   ```

> 你说“5555 正常”，如果你确实在用 Cloudflare，那大概率 5555 那条记录是灰云直连；4443 可能是橙云（或走了 CF Access），就会出现“一条正常一条不正常”的诡异现象。([Cloudflare Docs][2])

---

## 4) 再看反代上游：从 Caddy 容器里能不能访问到 `192.168.1.200:8080`

进入 Caddy 容器测试：

* `docker exec -it <caddy容器名> sh`
* `curl -v http://192.168.1.200:8080/`

如果这里都访问不了，那就是：

* 容器网络模式/路由问题（bridge/macvlan/ipvlan）
* 或者 Unraid WebUI 根本不是在 `8080`（或只监听在某个接口上）

---

## 5) 关于“只用 IPv6”：Caddy 端基本不用特别改，但你可以显式只监听 IPv6

一般不需要；但如果你想明确只在 IPv6 上监听，可以加 `bind`：([Caddy 服务器][3])

```caddy
unraid.example.com:4443 {
    bind [::]
    reverse_proxy 192.168.1.200:8080
}
```

---

## 6) Docker 本身的 IPv6 支持（仅当你发现 Docker 端口根本不在 IPv6 上监听时才需要看）

按 Docker 官方文档，Linux 上 Docker 可以启用 IPv6，并且端口发布可以同时在 IPv4/IPv6 上可用；还给了用 `curl http://[::1]:80` 验证的方式。([Docker Documentation][4])
不过你既然 5555 已经能从 IPv6 访问，通常说明 Docker/宿主的 IPv6 入站链路是 OK 的，**更可能是“4443 没映射/没放行/被代理限制”**。

---

# 你现在最该做的 3 件事（按优先级）

1. **外网 IPv6** 对 `4443` 做 `nc -6vz` / `curl -6vk`，看是 *timeout* 还是 *refused*（定位防火墙 vs 端口绑定）
2. 在 Unraid Docker 页面/`docker ps` 里确认 **4443 是否真的映射出来**
3. 如果用了 Cloudflare 橙云：把记录改灰云，或把端口换到 **8443**（Cloudflare 支持）([Cloudflare Docs][2])

如果你愿意把你 Caddy 容器的 `docker run` / `docker-compose.yml`（或 Unraid Docker 模板里端口映射截图文字）贴出来，我可以直接指出是哪一步没配到位。

[1]: https://caddyserver.com/docs/automatic-https "Automatic HTTPS — Caddy Documentation"
[2]: https://developers.cloudflare.com/fundamentals/reference/network-ports/ "Network ports · Cloudflare Fundamentals docs"
[3]: https://caddyserver.com.cn/docs/caddyfile/directives/bind?utm_source=chatgpt.com "bind (Caddyfile 指令) — Caddy 文档 - Caddy 服务器"
[4]: https://docs.docker.com/engine/daemon/ipv6/?utm_source=chatgpt.com "Use IPv6 networking | Docker Docs"


没有cloudflare，docker也没暴露什么东西，5555就是正常

version: "3.8"

services:
  caddy:
    container_name: caddy
    build:
      context: /mnt/user/misc/caddy
      dockerfile: Dockerfile
      args:
        # Build-time proxy for RUN steps (xcaddy/go downloads)
        HTTP_PROXY:  ${HTTP_PROXY}
        HTTPS_PROXY: ${HTTPS_PROXY}
    image: caddy-alidns:local
    restart: unless-stopped
    network_mode: host
    environment:
      ALIYUN_ACCESS_KEY_ID: ${ALIYUN_ACCESS_KEY_ID}
      ALIYUN_ACCESS_KEY_SECRET: ${ALIYUN_ACCESS_KEY_SECRET}
    volumes:
      - /mnt/user/appdata/caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - /mnt/user/appdata/caddy/data:/data
      - /mnt/user/appdata/caddy/config:/config

访问4443就是502，得到的caddy log是这样
{"level":"debug","ts":1770550817.6351426,"logger":"http.handlers.reverse_proxy","msg":"selected upstream","dial":"192.168.1.200:8080","total_upstreams":1}
{"level":"debug","ts":1770550825.0531328,"logger":"http.handlers.reverse_proxy","msg":"upstream roundtrip","upstream":"192.168.1.200:8080","duration":7.417769923,"request":{"remote_ip":"192.168.1.77","remote_port":"55565","client_ip":"192.168.1.77","proto":"HTTP/3.0","method":"GET","host":"unraid.example.com:4443","uri":"/","headers":{"Sec-Fetch-Mode":["navigate"],"Sec-Fetch-Site":["none"],"Accept":["text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7"],"Upgrade-Insecure-Requests":["1"],"Sec-Fetch-Dest":["document"],"Accept-Encoding":["gzip, deflate, br, zstd"],"Sec-Ch-Ua-Mobile":["?0"],"Sec-Ch-Ua-Platform":["\"Windows\""],"Priority":["u=0, i"],"Via":["3.0 Caddy"],"Accept-Language":["zh-CN,zh;q=0.9,ja;q=0.8,en;q=0.7"],"Sec-Ch-Ua":["\"Chromium\";v=\"142\", \"Google Chrome\";v=\"142\", \"Not_A Brand\";v=\"99\""],"X-Forwarded-Proto":["https"],"X-Real-Ip":[""],"Sec-Fetch-User":["?1"],"Early-Data":["1"],"X-Forwarded-For":[""],"User-Agent":["Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 Safari/537.36"],"X-Forwarded-Host":["unraid.example.com:4443"]},"tls":{"resumed":true,"version":772,"cipher_suite":4865,"proto":"h3","server_name":"unraid.example.com"}},"error":"net/http: HTTP/1.x transport connection broken: unexpected EOF"}
{"level":"error","ts":1770550825.05334,"logger":"http.log.error","msg":"net/http: HTTP/1.x transport connection broken: unexpected EOF","request":{"remote_ip":"192.168.1.77","remote_port":"55565","client_ip":"192.168.1.77","proto":"HTTP/3.0","method":"GET","host":"unraid.example.com:4443","uri":"/","headers":{"Sec-Fetch-Site":["none"],"User-Agent":["Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 Safari/537.36"],"Upgrade-Insecure-Requests":["1"],"Sec-Fetch-Dest":["document"],"Sec-Ch-Ua-Mobile":["?0"],"Accept-Encoding":["gzip, deflate, br, zstd"],"Accept":["text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7"],"Sec-Fetch-Mode":["navigate"],"Sec-Fetch-User":["?1"],"Accept-Language":["zh-CN,zh;q=0.9,ja;q=0.8,en;q=0.7"],"Sec-Ch-Ua":["\"Chromium\";v=\"142\", \"Google Chrome\";v=\"142\", \"Not_A Brand\";v=\"99\""],"Sec-Ch-Ua-Platform":["\"Windows\""],"Priority":["u=0, i"]},"tls":{"resumed":true,"version":772,"cipher_suite":4865,"proto":"h3","server_name":"unraid.example.com"}},"duration":7.418263241,"status":502,"err_id":"7a8x31zz3","err_trace":"reverseproxy.statusError (reverseproxy.go:1390)"}



你这个 **4443 的问题跟“只有 AAAA 记录 / 只用 IPv6”没关系**——因为请求已经打到 Caddy 了（日志里有 `proto":"HTTP/3.0" ... "host":"unraid.example.com:4443"`），说明 **DNS、证书、4443 端口监听都没问题**。现在的 502 是 **Caddy → 上游 `192.168.1.200:8080` 通信失败**，并且是：

> `net/http: HTTP/1.x transport connection broken: unexpected EOF`

这类 EOF 最常见原因是：**你以 HTTP 的方式去连了一个“其实是 HTTPS / h2c”的上游端口**，或者上游在复用连接时直接断开导致 Go 的 HTTP transport 报错（也会变成 502）。Caddy 官方文档也明确：默认是明文 HTTP；用 `https://` 会启用到上游的 TLS；`h2c://` 会按明文 HTTP/2 处理。 ([Caddy Web Server][1])

---

## 1) 先用 3 条命令把 “8080 到底是什么协议” 定死（在跑 Caddy 的那台机子上）

```bash
curl -v http://192.168.1.200:8080/
curl -vk https://192.168.1.200:8080/
curl -v --http2-prior-knowledge http://192.168.1.200:8080/
```

* **如果第二条（https）能返回页面/302，而第一条（http）不行**：说明 8080 是 HTTPS（或被强制 HTTPS）
* **如果第三条能通**：说明是 **h2c（明文 HTTP/2）**
* **如果第一条其实能通**：那就不是协议错配，去看第 3 部分（keepalive/头/重定向）

---

## 2) （最可能）上游其实是 HTTPS：把 upstream 写成 `https://`，必要时关验证

Caddy 文档：`https://` 会自动启用到上游的 TLS；如果上游是自签证书，可用 `tls_insecure_skip_verify`（不推荐用于公网，但内网排障可以先这样）。 ([Caddy Web Server][1])

把你的 4443 改成这样试：

```caddy
unraid.example.com:4443 {
    reverse_proxy https://192.168.1.200:8080 {
        transport http {
            tls_insecure_skip_verify
        }

        # 你现在用的 {remote_host} 看起来没有被替换成功（日志里是空字符串）
        # 建议用新版占位符
        header_up X-Real-IP {http.request.remote.host}
        header_up X-Forwarded-For {http.request.remote.host}
        header_up X-Forwarded-Proto {http.request.scheme}
        header_up X-Forwarded-Port {http.request.port}
        header_up X-Forwarded-Host {http.request.host}

        # 如果上游证书按 SNI/Host 分流，可能需要覆写 Host（官方文档有提到这种情况）
        # header_up Host {upstream_hostport}
    }

    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Frame-Options "DENY"
        X-Content-Type-Options "nosniff"
        -Server
    }
}
```

> 说明：你日志里 `X-Real-Ip` / `X-Forwarded-For` 都是空，基本可以判定 `{remote_host}` 没被正确替换；用 `{http.request.remote.host}` 更稳。

另外：如果你不想 `tls_insecure_skip_verify`，Caddy 也支持“信任上游证书/指定 SNI”，官方示例在 reverse_proxy 文档里（`tls_trust_pool` / `tls_server_name`）。([Caddy Web Server][1])

---

## 3) 如果上游是 h2c：用 `h2c://`（最省事）

官方文档明确：用 `h2c://` 会把 transport 的 HTTP versions 调整成允许明文 HTTP/2。 ([Caddy Web Server][1])

```caddy
unraid.example.com:4443 {
    reverse_proxy h2c://192.168.1.200:8080
}
```

---

## 4) 如果确认上游就是普通 HTTP：关掉 keepalive 试试（排“复用连接被上游干掉”）

Caddy 文档里提到：当 HTTP/1.1 上游的 keepalive 策略不匹配时，可能出现连接问题并导致 502。 ([Caddy Web Server][1])

```caddy
reverse_proxy 192.168.1.200:8080 {
    transport http {
        keepalive off
        read_timeout 300s
        write_timeout 300s
    }
}
```

---

## 5) 顺手确认一件事：你 Unraid 的管理端口/协议设置是否和你想的一致

Unraid 官方文档里提到 WebGUI 的 HTTPS 端口默认 443，也常被改到 3443/4443/5443 这种（尤其是端口被 Docker 占用时）。([Unraid 文档][2])
所以你现在写死 `192.168.1.200:8080` 很可能根本不是你以为的那个“Unraid WebUI（HTTP）”。

---

### 你下一步怎么最快定位

把你在跑 Caddy 的宿主机上执行的这两条输出贴出来就够了（不用全贴，贴关键几行）：

* `curl -v http://192.168.1.200:8080/`
* `curl -vk https://192.168.1.200:8080/`

我就能直接告诉你应该用 **http:// / https:// / h2c://** 哪一种，以及是否要配 `tls_server_name`/`Host`。

[1]: https://caddyserver.com/docs/caddyfile/directives/reverse_proxy "reverse_proxy (Caddyfile directive) — Caddy Documentation"
[2]: https://docs.unraid.net/unraid-connect/remote-access/?utm_source=chatgpt.com "Remote access - Unraid Docs"



协议不会有问题的，就是经典http啊，我现在怀疑这可能和unraid的某些设置相关，你去了解一下。unraid自己是有一个地方可以提供https访问的，但是问题是他的设置里面不能填证书，就是Management Access那里，是可以使用unraid自己的https的，只是证书好像不可选

root@Xunraid:~# curl -v http://192.168.1.200:8080/
* Uses proxy env variable no_proxy == '127.0.0.1,localhost'
* Uses proxy env variable http_proxy == 'http://192.168.1.x:xxxx'
*   Trying 192.168.1.x:xxxx...
* Established connection to 192.168.1.2 (192.168.1.2 port 7891) from 192.168.1.200 port 60860 
* using HTTP/1.x
> GET http://192.168.1.200:8080/ HTTP/1.1
> Host: 192.168.1.200:8080
> User-Agent: curl/8.16.0
> Accept: */*
> Proxy-Connection: Keep-Alive
> 
* Request completely sent off


你这条线索其实已经把原因暴露得很明显了：**不是 Unraid 的 WebGUI 协议问题，而是“代理环境变量把你访问 192.168.1.200:8080 的流量劫持去走了 192.168.1.x:xxxx 这个 HTTP 代理”**。

你自己贴的 curl 输出里写得很清楚：

* `Uses proxy env variable http_proxy == 'http://192.168.1.x:xxxx'`
* 所以它并没有直连 `192.168.1.200:8080`，而是先连到 `192.168.1.x:xxxx` 再请求 `http://192.168.1.200:8080/`

这会导致两件事同时成立：

1. **5555 正常**（因为上游是 `127.0.0.1:3005`，通常在 `NO_PROXY` 里会被排除掉，不走代理）
2. **4443 反代 502/EOF**（因为上游是 `192.168.1.200:8080`，不在 `NO_PROXY`，走代理后代理不转发/断开，就会出现 `unexpected EOF`）

Go 的 HTTP 客户端（Caddy 也是 Go 写的）会按环境变量 `HTTP_PROXY/HTTPS_PROXY/NO_PROXY` 来决定是否走代理。([Go Packages][1])

---

## 先用两条命令验证“直连 8080 其实是好的”

在 Unraid 上执行（关键是**绕过代理**）：

```bash
curl --noproxy '*' -v http://192.168.1.200:8080/ --max-time 5
curl --noproxy '*' -v http://127.0.0.1:8080/ --max-time 5
```

只要这里能正常返回（哪怕是 302/登录页 HTML），就能确定：**Unraid WebGUI 的 HTTP 本身没问题**，问题就是代理环境变量影响了 Caddy→上游。

---

## 最快的修法（推荐）：把 upstream 改成 127.0.0.1（直接绕开代理）

因为 Caddy 就跑在同一台 Unraid 上（`network_mode: host`），你完全可以这样写：

```caddy
unraid.example.com:4443 {
    reverse_proxy 127.0.0.1:8080
}
```

这通常“一改就好”，而且和你 5555 那条为什么正常是同一逻辑。

> 你日志里 `X-Real-Ip` / `X-Forwarded-For` 为空，说明你用的 `{remote_host}` 占位符也没按预期生效；但这不会导致 502，只是建议顺手修一下。

---

## 根治法：在 caddy 容器里正确设置 NO_PROXY（或清空代理变量）

先看 caddy 容器里到底有没有这些变量：

```bash
docker exec -it caddy sh -lc 'env | grep -i proxy'
```

如果你看到了 `HTTP_PROXY/http_proxy` 之类，那就按你的需求二选一：

### A) 你不需要容器走代理：直接清空

```yaml
services:
  caddy:
    environment:
      HTTP_PROXY: ""
      HTTPS_PROXY: ""
      http_proxy: ""
      https_proxy: ""
      NO_PROXY: "127.0.0.1,localhost"
      no_proxy: "127.0.0.1,localhost"
```

### B) 你需要容器出网走代理：但内网地址必须 bypass

```yaml
services:
  caddy:
    environment:
      NO_PROXY: "127.0.0.1,localhost,192.168.0.0/16,10.0.0.0/8,172.16.0.0/12,192.168.1.200"
      no_proxy: "127.0.0.1,localhost,192.168.0.0/16,10.0.0.0/8,172.16.0.0/12,192.168.1.200"
```

（这正是 Go `ProxyFromEnvironment` 机制的用法：`NO_PROXY` 命中的地址不走代理。([Go Packages][1])）

---

## 你提到的 Unraid “Management Access 里 HTTPS 不能选证书”这件事：官方就是这么设计的

Unraid 的自定义证书**不是在 UI 里选文件**，而是把证书 bundle 放到固定路径：

* 到 **Settings → Management Access**
* `Use SSL/TLS` 设为 `Yes`
* `Local TLD` 要和证书域名匹配
* 把证书上传到：`/boot/config/ssl/certs/[servername]_unraid_bundle.pem`
  而且证书不匹配会被 Unraid 删除并回退。([Unraid 文档][2])

另外，如果你想让后端稳定给 Caddy 反代，官方也明确支持把 WebGUI 设成 **HTTP-only**（`Use SSL/TLS = No`）。([Unraid 文档][2])

---

## 我会怎么建议你最终落地（最稳）

1. Unraid：`Use SSL/TLS = No`（让后端始终是 HTTP，避免任何重定向/严格模式干扰）([Unraid 文档][2])
2. Caddy：TLS 终止在 Caddy；上游写 `127.0.0.1:8080`
3. 容器里把 `NO_PROXY` 配好（或清空代理变量），以后不会再遇到这种“内网地址被代理劫持”的 502/EOF

你把 `docker exec caddy env | grep -i proxy` 的输出贴一段（打码也行），我可以直接告诉你现在是哪一种情况：**容器确实带了代理变量**，还是 **只有宿主机带了但容器没有**。

[1]: https://pkg.go.dev/golang.org/x/net/http/httpproxy?utm_source=chatgpt.com "httpproxy package - golang.org/x/net/http/httpproxy - Go Packages"
[2]: https://docs.unraid.net/unraid-os/system-administration/secure-your-server/securing-your-connection/ "Securing your connection | Unraid Docs"



我的docker compose

```
version: "3.8"

services:
  caddy:
    container_name: caddy
    build:
      context: /mnt/user/misc/caddy
      dockerfile: Dockerfile
      args:
        # Build-time proxy for RUN steps (xcaddy/go downloads)
        HTTP_PROXY:  ${HTTP_PROXY}
        HTTPS_PROXY: ${HTTPS_PROXY}
    image: caddy-alidns:local
    restart: unless-stopped
    network_mode: host
    environment:
      ALIYUN_ACCESS_KEY_ID: ${ALIYUN_ACCESS_KEY_ID}
      ALIYUN_ACCESS_KEY_SECRET: ${ALIYUN_ACCESS_KEY_SECRET}
    volumes:
      - /mnt/user/appdata/caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - /mnt/user/appdata/caddy/data:/data
      - /mnt/user/appdata/caddy/config:/config

```

env file 
```
# Build/runtime proxy (set only what you use)
HTTP_PROXY=http://192.168.1.x:xxxx
HTTPS_PROXY=http://192.168.1.x:xxxx

# Aliyun DNS
ALIYUN_ACCESS_KEY_ID=
ALIYUN_ACCESS_KEY_SECRET=
```


caddyfile
```
{
  email your-email@example.com
  acme_dns alidns {
    access_key_id {env.ALIYUN_ACCESS_KEY_ID}
    access_key_secret {env.ALIYUN_ACCESS_KEY_SECRET}
  }
  debug
}

unraid.example.com:4443 {
        reverse_proxy 127.0.0.1:8080 {
                flush_interval -1

                header_up X-Real-IP {remote_host}
                header_up X-Forwarded-For {remote_host}
                header_up X-Forwarded-Proto {scheme}

                transport http {
                        read_timeout 300s
                        write_timeout 300s
                        dial_timeout 30s
                }
        }

        header {
                Strict-Transport-Security "max-age=31536000; includeSubDomains"
                X-Frame-Options "DENY"
                X-Content-Type-Options "nosniff"
                -Server
        }
}

unraid.example.com:5555 {
        reverse_proxy 127.0.0.1:3005 {
                flush_interval -1

                header_up X-Real-IP {remote_host}
                header_up X-Forwarded-For {remote_host}
                header_up X-Forwarded-Proto {scheme}

                transport http {
                        read_timeout 300s
                        write_timeout 300s
                        dial_timeout 30s
                }
        }

        header {
                Strict-Transport-Security "max-age=31536000; includeSubDomains"
                X-Frame-Options "DENY"
                X-Content-Type-Options "nosniff"
                -Server
        }
}
```