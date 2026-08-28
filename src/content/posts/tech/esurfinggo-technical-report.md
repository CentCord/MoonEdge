---
title: "EsurfingGo 技术解析：中国电信校园网认证与保活机制"
published: 2026-08-28
description: "深入分析 EsurfingGo 的强制门户检测、加密会话建立、票据获取、登录认证与心跳保活完整流程。"
tags: ["Go", "网络认证", "Captive Portal", "ESurfing", "校园网"]
category: "技术解析"
image: ""
---

# EsurfingGo 技术解析：中国电信校园网认证与保活机制

## 摘要

`EsurfingGo` 是一个使用 Go 语言实现的中国电信天翼校园网（ESurfing）自动认证客户端。它能够在检测到强制门户（Captive Portal）后自动完成登录，并通过周期性心跳维持在线状态，支持断线自动重连与多网卡绑定。本文从源码层面详细解析其网络检测、会话加密、票据获取、登录认证与心跳保活的完整技术流程。

---

## 1. 项目概述

### 1.1 背景

在许多高校和公共场所，中国电信提供的校园网接入采用 Web Portal 认证方式。用户连接 Wi-Fi 或有线网络后，首次访问互联网会被重定向到认证页面，需要手动输入账号密码登录。`EsurfingGo` 将该过程自动化，适用于路由器、服务器或个人电脑长期挂机场景。

### 1.2 核心功能

- **自动门户检测**：通过访问小米 captive 探测地址判断网络状态。
- **自动认证**：解析 portal 页面配置，完成加密登录。
- **心跳保活**：按服务端指定间隔发送心跳，断线后自动重连。
- **多拨支持**：可绑定指定网络接口，实现多网卡同时在线。
- **跨平台部署**：Go 编译为单文件二进制，支持 Windows、Linux、macOS 及 ARM 路由器。

### 1.3 依赖

```go
// go.mod
require (
    github.com/emmansun/gmsm v0.41.1  // 国密 SM4 / ZUC
    github.com/google/uuid v1.6.0      // UUID 生成
)
```

---

## 2. 项目结构

```
.
├── main.go              // 程序入口、命令行解析、信号处理
├── client.go            // 认证主循环与登录逻辑
├── session.go           // ZSM 解析与加密会话管理
├── states.go            // 线程安全全局状态
├── constants.go         // 常量定义
├── iface.go             // 网络接口列举与绑定
├── network/
│   ├── client.go        // HTTP 客户端与请求封装
│   └── connectivity.go  // 强制门户检测与短信验证码
├── cipher/
│   ├── cipher.go        // 加密算法工厂
│   ├── keydata.go       // 各算法密钥
│   └── ...              // AES、3DES、SM4、ZUC、ModXTEA 实现
├── utils/               // 工具函数
└── model/               // 数据模型
```

---

## 3. 认证整体流程

认证流程可以概括为五个阶段：

1. **门户检测**：判断当前是否已联网，或是否需要认证。
2. **配置解析**：从 portal 页面提取认证接口地址、用户 IP、AC IP 等。
3. **会话初始化**：向服务端请求 ZSM 数据，确定本次加密算法。
4. **票据获取**：使用加密会话请求 ticket。
5. **登录与保活**：携带 ticket 登录，获取心跳地址后周期性保活。

```text
启动
  │
  ▼
访问 generate_204
  │
  ├── HTTP 204 ────────────────► 已联网，轮询检测
  │
  └── HTML / 重定向 ───────────► 解析 portal 配置
            │
            ▼
      提取 auth-url、ticket-url、userIp、acIp
            │
            ▼
      请求 ZSM，初始化加密会话
            │
            ▼
      getTicket ──► login ──► 获取 keep-url / term-url
            │
            ▼
      心跳保活（失败则回到检测阶段）
```

---

## 4. 强制门户检测

### 4.1 探测原理

客户端使用小米标准的 Captive Portal 探测 URL：

```go
const CaptiveURL = "http://connect.rom.miui.com/generate_204"
```

该地址设计为：

- 如果设备已联网，返回 **HTTP 204 No Content**。
- 如果设备处于强制门户网络，会被重定向到认证页面，返回 HTML。

### 4.2 代码实现

检测逻辑位于 `network/connectivity.go:56` 的 `DetectConfigWithClient`：

```go
func DetectConfigWithClient(httpClient *http.Client, state StateProvider, verbose bool) ConfigResult
```

主要步骤：

1. 发送 GET 请求到 `captiveURL`，携带 `User-Agent`、`Accept`、`Client-ID` 头。
2. 如果返回 `204`，直接认为已联网。
3. 如果返回 HTML，尝试在页面内容中定位门户配置：

```html
<!--//config.campus.js.chinatelecom.com
<config>
    <auth-url>...</auth-url>
    <ticket-url>...</ticket-url>
    <funcfg>...</funcfg>
</config>
//config.campus.js.chinatelecom.com-->
```

4. 如果页面没有直接返回配置，可能是 JS 跳转，代码会解析 `location.href="..."` 等模式并跟随，最多尝试 6 次。

### 4.3 提取关键信息

从 `ticket-url` 的 query 参数中解析用户 IP 和 AC IP：

```go
parsedURL, _ := url.Parse(result.TicketURL)
result.UserIP = parsedURL.Query().Get("wlanuserip")
result.AcIP   = parsedURL.Query().Get("wlanacip")
```

同时从 HTTP 响应头或 portal 配置中提取 `CDC-Area`、`CDC-SchoolId`、`CDC-Domain`，用于后续请求路由。

---

## 5. 短信验证码流程

部分学校账号登录需要短信验证码。`client.go:177` 的 `checkSMSVerify` 函数处理该逻辑：

```go
if network.CheckVerifyCodeStatus(...) && network.GetVerifyCode(...) {
    log.Println("This login requires a SMS verification code.")
    // 从标准输入读取验证码
}
```

两个网络接口定义在 `network/connectivity.go:379`：

- `CheckVerifyCodeStatus`：查询是否需要验证码。
- `GetVerifyCode`：触发服务端下发短信。

请求体为 JSON：

```json
{
  "schoolid": "...",
  "username": "...",
  "timestamp": "172...",
  "authenticator": "..."
}
```

其中 `authenticator` 的计算方式为：

```go
hash := md5.Sum([]byte(schoolID + timestamp + "Eshore!@#"))
authenticator := strings.ToUpper(hex.EncodeToString(hash[:]))
```

这是一个简单的时间戳 + 共享密钥的校验机制。

---

## 6. 加密会话初始化（ZSM）

### 6.1 什么是 ZSM

ZSM 是服务端返回的一段二进制数据，包含了本次连接使用的加密算法标识（`algoID`）和密钥信息。客户端解析后创建对应的加解密器。

### 6.2 请求 ZSM

`client.go:195`：

```go
body, err := network.PostRaw(
    c.httpClient,
    c.states.GetTicketURL(),
    c.states.GetAlgoID(), // 初始为 "00000000-0000-0000-0000-000000000000"
    c.states,
)
c.session.Initialize(body)
```

### 6.3 ZSM 解析

`session.go:31` 的 `load` 方法解析二进制格式：

```text
[0..3]        4 字节头部
[3]           1 字节 keyLen
[4..4+keyLen] 密钥数据
[...]         分隔符 '$'
[...]         36 字节 algoID（UUID 字符串）
[...]         ']' 字符
[...]         后续数据
```

解析出 `algoID` 后，通过 `cipher.NewCipher(algoID)` 创建算法实例。

### 6.4 支持的加密算法

`cipher/cipher.go` 实现了工厂模式，支持 9 种算法：

| 算法 UUID | 算法 |
|---|---|
| `CAFBCBAD-B6E7-4CAB-8A67-14D39F00CE1E` | AES-CBC |
| `A474B1C2-3DE0-4EA2-8C5F-7093409CE6C4` | AES-ECB |
| `5BFBA864-BBA9-42DB-8EAD-49B5F412BD81` | 3DES-CBC |
| `6E0B65FF-0B5B-459C-8FCE-EC7F2BEA9FF5` | 3DES-ECB |
| `B809531F-0007-4B5B-923B-4BD560398113` | ZUC-128 |
| `F3974434-C0DD-4C20-9E87-DDB6814A1C48` | SM4-CBC |
| `ED382482-F72C-4C41-A76D-28EEA0F1F2AF` | SM4-ECB |
| `B3047D4E-67DF-4864-A6A5-DF9B9E525C79` | ModXTEA |
| `C32C68F9-CA81-4260-A329-BBAFD1A9CCD1` | ModXTEA-XTEAIV |

密钥数据硬编码在 `cipher/keydata.go` 中。

### 6.5 双重 AES-CBC 示例

以 AES-CBC 为例，`cipher/aescbc.go:41`：

```go
func (a *AESCBC) Encrypt(text string) string {
    r1 := a.aesEncrypt([]byte(text), a.key1)
    r2 := a.aesEncrypt(r1, a.key2)
    return hexEncode(r2)
}
```

即先用 `key1` 加密一次，再用 `key2` 加密一次，最后返回大写十六进制字符串。解密则反向操作。

---

## 7. 票据获取（getTicket）

### 7.1 请求 XML

`client.go:207` 构造如下 XML：

```xml
<?xml version="1.0" encoding="utf-8"?>
<request>
    <user-agent>CCTP/android64_vpn/2093</user-agent>
    <client-id>%s</client-id>
    <local-time>%s</local-time>
    <host-name>%s</host-name>
    <ipv4>%s</ipv4>
    <ipv6></ipv6>
    <mac>%s</mac>
    <ostag>%s</ostag>
    <gwip>%s</gwip>
</request>
```

填充字段：

- `client-id`：随机生成的 UUID。
- `local-time`：北京时间格式化字符串。
- `host-name` / `ostag`：随机 10 位字符串。
- `ipv4`：从 ticket-url 解析出的 `wlanuserip`。
- `mac`：随机生成的 MAC 地址。
- `gwip`：从 ticket-url 解析出的 `wlanacip`。

### 7.2 加密传输

```go
encrypted, _ := c.session.Encrypt(payload)
data, _ := network.Post(c.httpClient, c.states.GetTicketURL(), encrypted, c.states, nil)
decrypted, _ := c.session.Decrypt(data)
ticket := extractXMLTag(decrypted, "ticket")
```

请求体是十六进制密文字符串，响应也是十六进制密文，解密后提取 `<ticket>`。

---

## 8. 登录认证（login）

### 8.1 请求 XML

`client.go:250`：

```xml
<?xml version="1.0" encoding="utf-8"?>
<request>
    <user-agent>CCTP/android64_vpn/2093</user-agent>
    <client-id>%s</client-id>
    <ticket>%s</ticket>
    <local-time>%s</local-time>
    <userid>%s</userid>
    <passwd>%s</passwd>
    <verify>%s</verify>  <!-- 可选 -->
</request>
```

用户名、密码、短信验证码会做 XML 转义，防止 `&`、`<` 等字符破坏 XML 结构。

### 8.2 解析登录响应

解密后提取三个关键字段：

```go
c.keepURL   = extractXMLTag(decrypted, "keep-url")
c.termURL   = extractXMLTag(decrypted, "term-url")
c.keepRetry = extractXMLTag(decrypted, "keep-retry")
```

### 8.3 登录成功判定

`client.go:165`：

```go
if c.keepURL == "" {
    log.Println("[Client] KeepUrl is empty, login may have failed.")
    c.session.Free()
    c.states.SetRunning(false)
    return
}

c.tick = time.Now().UnixMilli()
c.states.SetLogged(true)
```

只有服务端返回了 `keep-url`，才认为登录成功并进入心跳阶段。

---

## 9. 心跳保活机制

### 9.1 触发逻辑

`client.go:57` 主循环中：

```go
if c.session.IsInitialized() && c.states.IsLogged() {
    nowMs := time.Now().UnixMilli()
    retryMs, err := strconv.ParseInt(c.keepRetry, 10, 64)
    if err == nil && retryMs > 0 && nowMs - c.tick >= retryMs * 1000 {
        if err := c.heartbeat(c.states.GetTicket()); err != nil {
            c.states.SetLogged(false)
            c.statusChanged = true
            continue
        }
        c.tick = time.Now().UnixMilli()
    }
    time.Sleep(1 * time.Second)
    continue
}
```

- 每秒检查一次。
- 当距离上次心跳或登录超过 `keep-retry` 秒时，触发心跳。
- 心跳失败则重置登录状态，下一次循环重新进入检测和认证流程，实现自动重连。

### 9.2 心跳请求体

`client.go:302`：

```xml
<?xml version="1.0" encoding="utf-8"?>
<request>
    <user-agent>CCTP/android64_vpn/2093</user-agent>
    <client-id>%s</client-id>
    <local-time>%s</local-time>
    <host-name>%s</host-name>
    <ipv4>%s</ipv4>
    <ticket>%s</ticket>
    <ipv6></ipv6>
    <mac>%s</mac>
    <ostag>%s</ostag>
</request>
```

心跳使用 ticket 作为身份凭证，不再携带用户名密码。

### 9.3 动态心跳间隔

`client.go:337`：

```go
interval := extractXMLTag(decrypted, "interval")
if interval != "" {
    c.keepRetry = interval
}
```

服务端可以在心跳响应中下发新的 `<interval>`，客户端会更新下一次心跳间隔。

---

## 10. HTTP 请求封装

所有认证相关 POST 都经过 `network/client.go:179` 的 `Post()` 或 `PostRaw()`，统一封装请求头：

| 请求头 | 来源/含义 |
|---|---|
| `User-Agent` | `CCTP/android64_vpn/2093` |
| `Accept` | 固定 Accept 字符串 |
| `Content-Type` | `application/x-www-form-urlencoded` |
| `CDC-Checksum` | 请求体 MD5 大写十六进制 |
| `Client-ID` | `states.GetClientID()` |
| `Algo-ID` | `states.GetAlgoID()` |
| `CDC-SchoolId` | 学校路由 ID |
| `CDC-Domain` | 域名路由 |
| `CDC-Area` | 区域路由 |

### 10.1 重定向拦截器

`network/client.go:34` 的 `redirectInterceptor` 自定义处理重定向：

1. 读取响应头中的 `CDC-Area`、`CDC-SchoolId`、`CDC-Domain` 并写入状态。
2. 后续请求自动带上这些头，确保路由正确。
3. 最多跟随 5 次重定向。

---

## 11. 网卡绑定与多拨

### 11.1 列出网卡

`iface.go:19` 的 `ListNetworkInterfaces()` 使用 Go 标准库 `net.Interfaces()`，过滤掉回环接口和未启用接口。

### 11.2 绑定指定网卡

`iface.go:72` 的 `NewBoundHTTPTransport()`：

1. 根据网卡名获取接口。
2. 获取接口上第一个 IPv4 地址。
3. 构造 `net.Dialer`，设置 `LocalAddr` 为该 IPv4。
4. 所有 HTTP 连接的 TCP 连接都从该地址发起。

这种方式跨平台有效，不依赖操作系统特定的 socket option。

---

## 12. 安全下线

`main.go:97` 注册信号监听：

```go
sigCh := make(chan os.Signal, 1)
signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
go func() {
    <-sigCh
    if session.IsInitialized() && states.IsLogged() {
        client.Term()
        session.Free()
    }
    os.Exit(0)
}()
```

`client.go:345` 的 `Term()` 构造与心跳类似的 XML，但 POST 到 `term-url`，通知服务端主动下线。

---

## 13. 线程安全设计

`states.go` 使用 `sync.RWMutex` 保护所有全局状态，每个字段都有 getter/setter。例如：

```go
func (s *States) GetClientID() string {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return s.clientID
}

func (s *States) SetClientID(v string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.clientID = v
}
```

这保证了主循环、心跳 goroutine、信号处理 goroutine 并发访问状态时的安全性。

---

## 14. 总结

`EsurfingGo` 的认证流程可以概括为：**探测 → 解析 → 会话 → 票据 → 登录 → 心跳**。整个流程基于状态机驱动，任何阶段失败都会回到检测阶段重新尝试，从而实现长期稳定的自动在线。

其技术亮点包括：

- 使用 `generate_204` 标准机制检测 Captive Portal。
- 通过 ZSM 二进制数据动态协商 9 种加密算法之一。
- 自定义 `redirectInterceptor` 处理电信校园网特有的 CDC 路由头。
- 心跳保活 + 失败自动重连，适合无人值守部署。
- 网卡绑定机制简单有效，支持跨平台多拨。

对于希望理解校园网认证协议、开发类似自动登录工具，或学习 Go 网络编程的开发者，`EsurfingGo` 是一个结构清晰、值得参考的实战项目。

---

## 参考

- 项目仓库：[EsurfingGo](https://github.com/xxmod/EsurfingGo)
- 协议逻辑继承自：[Rsplwe/EsurfingDialer](https://github.com/Rsplwe/EsurfingDialer)
- 小米 Captive Portal 探测：`http://connect.rom.miui.com/generate_204`
