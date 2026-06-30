# 使用 docker mihomo 为 cloudflared 添加前置代理
 
---
国内建立 cloudflare tunnel 隧道的节点经常是美西，导致延迟很高。一种解决方案是通过前置代理建立隧道，让 cloudflare 与距离近的代理节点（如香港节点等）之间建立隧道，这样就能缓解 tunnel 因物理距离导致延迟高的问题。  

> [Cloudflare Tunnel速度慢？尝试给它加个前置代理提高速度](https://blog.xmgspace.me/archives/cloudflare-tunnel-via-proxy.html)  
> [Cloudflare Tunnel前置代理支持](https://www.iots.vip/post/cloudflare-tunnel-proxy-support)  

网上虽有可实践的方案，但对大部分人（比如我 haha~）来说还是有点困难。为了操作的便捷性，尝试使用纯 docker 容器的方案为 cloudflare 添加前置代理，相比已有方案大概有以下好处：
- 纯 docker 方案，将影响限制在 docker 网络层
- 简单易操作
- 官方镜像，不用担心维护问题

## 1 mihomo + cloudflared 容器部署

使用 docker compose 部署，文件内容如下

```yaml
services:
  mihomo:
    container_name: mihomo
    image: metacubex/mihomo:latest
    restart: unless-stopped
    # 添加hosts解析，方便docker容器通过此解析访问宿主机
    # 如：http://docker-host:8080
    # docker-host 可以改成任意名字如 host.docker.internal
    extra_hosts:
      - docker-host:host-gateway
    ports:
      # mihomo 面板配置中自定义的端口
      # 使用 yourip:9888/ui 访问面板
      - 9888:9888
    # ！！此处必须手动指定任意 dns 否则 mihomo 劫持 dns 失效
    dns:
      - 127.0.0.1
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun
    volumes:
      - ./mihomo/config/mihomo:/root/.config/mihomo
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    # 共享 mihomo 容器网络
    network_mode: service:mihomo
    depends_on:
      - mihomo
    command: tunnel --no-autoupdate run --token [your token]
    restart: unless-stopped
    
```

## 2 mihomo 配置文件编写

在 `./mihomo/config/mihomo` ( ./ 指的是 compose 文件所在目录) 目录下创建 `config.yaml` 文件，写入以下内容

```yaml
# ==========================================================
#   使用 docker mihomo 将透明代理隔离在 docker 网络层, cloudflared 容器共享 mihomo 容器网络
#   目的是以尽量小的影响代理 tunnel
#   以下配置力求简洁
# ==========================================================
#   参考链接
#  【Cloudflare Tunnel速度慢？尝试给它加个前置代理提高速度】https://blog.xmgspace.me/archives/cloudflare-tunnel-via-proxy.html/
#  【Cloudflare Tunnel前置代理支持】https://www.iots.vip/post/cloudflare-tunnel-proxy-support
#  【mihomo 官方教程】https://wiki.metacubex.one/
#  【第三方配置教程】https://github.com/HenryChiao/MIHOMO_YAMLS/wiki/
# ==========================================================

# ==========================================================
#  1. 控制面板管理
# ==========================================================
# 自定义面板访问地址 （访问地址是 yourip:9888/ui）
external-controller: 0.0.0.0:9888
# 控制面板登录密码
secret: "123456" 
# Web UI 存放目录（相对于配置文件目录）
external-ui: ui
external-ui-url: "https://github.com/MetaCubeX/metacubexd/archive/refs/heads/gh-pages.zip"

# ==========================================================
#  2. 机场订阅
# ==========================================================
proxy-providers:
  main:
    type: http
    url: https://[订阅地址]
    # 更新间隔 1 天
    interval: 86400
    proxy: DIRECT
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 300
    override:
      skip-cert-verify: true
      udp: true

# ==========================================================
#  3. 基础参数
# ==========================================================
# 规则模式，此外还有global direct
mode: rule 
# 日志等级 silent/error/warning/info/debug
log-level: warning 
# 统一延迟测试
unified-delay: true       

# ==========================================================
#  4. 基础 DNS 模块
# ==========================================================
dns:
  enable: true
  ipv6: false
  # 开启 DNS 服务器监听
  listen: 0.0.0.0:53
  # 配合 tun 最好用 fake-ip
  enhanced-mode: fake-ip        
  fake-ip-range: 198.18.0.1/16
  # 优先使用 HTTP/3 协议（DoH3），更快
  prefer-h3: true
  # 默认 nameserver：国内 DNS，并发查询取最快结果
  nameserver:
    # 阿里 DoH
    - https://223.5.5.5/dns-query
    # 腾讯 DoH    
    - https://doh.pub/dns-query     
  # 用于解析 nameserver，fallback 以及其他 DNS 服务器配置的，DNS 服务域名
  # 只能使用纯 IP 地址，可使用加密 DNS
  default-nameserver:
    - 223.5.5.5
    - 8.8.8.8

# ==========================================================
#  5. tun 实现透明代理 
# ==========================================================
tun:
  enable: true
  # TCP 走 system，UDP 走 gvisor
  stack: mixed 
  # dns 劫持
  dns-hijack: ["any:53", "tcp://any:53"]
  # 自动识别出口网卡
  auto-detect-interface: true
  # 自动配置路由表
  auto-route: true 
  # 自动配置 iptables 以重定向 TCP 连接。
  auto-redirect: true 
  # 将所有连接路由到 tun 来防止泄漏
  strict-route: true 

# ==========================================================
#  6. 出站策略组
#     此配置目的是作为 cloudflared tunnel 前置代理
#     考虑到延迟并保持配置简约，这里只写香港｜日本｜新加坡
# ==========================================================
# 节点筛选正则表达式 - 基于地理位置和关键词过滤
FilterHK: &FilterHK '^(?=.*(?i)(港|🇭🇰|HK|Hong|HKG)).*$'
FilterSG: &FilterSG '^(?=.*(?i)(坡|🇸🇬|SG|Sing|SIN|XSP)).*$'
FilterJP: &FilterJP '^(?=.*(?i)(日|🇯🇵|JP|Japan|NRT|HND|KIX|CTS|FUK)).*$'
# 延迟测试锚点
BaseUT: &BaseUT {type: url-test, interval: 200, lazy: true, tolerance: 30, url: 'https://www.google.com/generate_204' }
# 代理分组
proxy-groups:
  - { name: 默认代理, type: select, proxies: [香港自动,狮城自动,日本自动,自动选择,香港节点,狮城节点,日本节点,手动选择,DIRECT] }
  - { name: 香港自动, <<: *BaseUT, include-all: true, filter: *FilterHK }
  - { name: 狮城自动, <<: *BaseUT, include-all: true, filter: *FilterSG }
  - { name: 日本自动, <<: *BaseUT, include-all: true, filter: *FilterJP }
  - { name: 自动选择, <<: *BaseUT, include-all: true }
  - { name: 香港节点, type: select, include-all: true, filter: *FilterHK }
  - { name: 狮城节点, type: select, include-all: true, filter: *FilterSG }
  - { name: 日本节点, type: select, include-all: true, filter: *FilterJP }
  - { name: 手动选择, type: select, include-all: true }

# ==========================================================
#  7. 代理规则
# ==========================================================
rules:
  # argotunnel
  - DOMAIN-KEYWORD,argotunnel,默认代理
  # 代理 region1.v2.argotunnel.com、region2.v2.argotunnel.com ip段
  # 实际上 tunnel 发起的链接是通过域名方式，这两条 ip 规则不会触发，可以当做保留
  - IP-CIDR,198.41.192.0/24,默认代理,no-resolve
  - IP-CIDR,198.41.200.0/24,默认代理,no-resolve
  # 其他规则 (可自行添加)
  - DOMAIN-KEYWORD,google,默认代理
  - DOMAIN-KEYWORD,cloudflare,默认代理
  - DOMAIN-KEYWORD,alpinelinux,默认代理
  # 剩余的其他请求全部默认直连
  - MATCH,DIRECT
```

## 3 验证

重启 docker 容器（mihomo + cloudflare），查看 docker 容器日志

```bash
docker logs cloudflared 2>&1 | grep "location"
```

观察日志判断是否成功

```bash
cloudflared  | INF Registered tunnel connection connIndex=1 connection=6d68b9c3-xxx68d152d3f event=0 ip=198.18.0.4 location=hkg09 protocol=quic
```

出现类似 `location=hkg09` （香港），说明前置代理生效。

## 4 补充

#### 关于 dcoker compose `dns` 配置

必须为容器指定任意上游 DNS ，否则 docker 默认 DNS 解析器（127.0.0.11）会导致 mihomo 劫持失败。查询原因貌似是因为 127.0.0.11 解析全程在回环接口，对 tun 来说是透明的，也就无从劫持。但是哪怕我随意指定一个同样回环接口的地址如 127.0.0.2（甚至没有 dns 服务器）也能正常被劫持。由于对于底层知识了解有限，只能总结现象：只要为 docker 默认 DNS 解析器设置了上游 DNS，查询外部域名时 docker 默认解析器进行 DNS 转发便能被 tun 捕获并劫持。

#### 关于域名嗅探

```yaml
# 官网配置片段示例
sniffer:
  enable: false
  # 是否使用嗅探结果作为实际访问，默认为 true
  override-destination: false
  sniff:
    HTTP:
      ports: [80, 8080-8880]
      override-destination: true
    TLS:
      ports: [443, 8443]
    QUIC:
      ports: [443, 8443]
```

经过验证使用 tun 不是必须要配合域名嗅探也能使域名规则生效。而且即便使用域名嗅探也需要注意在当前场景下最好指定 `override-destination: false` （需显式指定，默认是 true）。否则可能影响 tunnel 回源本地服务的过程。假设你有一个 tunnel 服务配置如下：

```text
完整主机名：a.b.com
服务 URL：http://docker-host:8080
```

当你在访问 `a.b.com` ，`cloudflared` 收到后，根据配置的服务 URL 回源，准备把流量转发给`originService=http://docker-host:8080`。此时这条在经过 tun 时，发现 HTTP 请求里的 `Host: a.b.com`，如果此时 `override-destination: true` 处于开启状态，mihomo 会把原本要去本地服务 `http://docker-host:8080` 的地址改写成 `http://a.b.com:8080` ，最终访问将偏离预期。
