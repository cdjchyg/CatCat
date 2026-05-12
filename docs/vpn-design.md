# 分布式中继 VPN 系统设计文档

> 版本：v0.1（草案）
> 状态：Draft，待评审
> 作者：—
> 最近更新：2026-05-12

---

## 1. 背景与目标

### 1.1 背景

业务需要一套自研 VPN，用户的真实出口 IP 不能由用户本机直接暴露，而是要由若干部署在不同位置的"出口机"代为收发流量。出于运维便利，希望统一由一台中心服务器调度全部出口机和客户端，客户端无需感知出口机的存在与变化。

### 1.2 目标

- **多端覆盖**：客户端支持 Windows 与 Android，服务端组件支持 Linux 与 Windows。
- **两级服务端**：
  - **主节点（Master）**：单点（逻辑上单点，可做主备），部署在 Linux，对外接受客户端连接、对内调度支节点。
  - **支节点（Branch / Exit Node）**：多实例，部署在 Windows，作为真实出口主动连接到主节点。
- **典型数据路径**：
  ```
  Client → Master → Branch → Internet → Branch → Master → Client
  ```
- **多用户并发**：单台主节点支持成百上千客户端同时在线，并能将不同客户端的流量路由到不同支节点。
- **安全**：链路全程加密、可鉴权；支节点不能直接被客户端访问。
- **运维友好**：支节点通常处在 NAT/防火墙后，只允许主动外连；不要求支节点有公网 IP。

### 1.3 非目标（当前版本不做）

- 站点到站点（site-to-site）的二层桥接。
- 多主节点之间的实时一致性集群（v0.1 用主备 + DNS / VIP 切换，不做强一致集群）。
- 用户计费 / 流量配额（预留接口，不在 MVP 里实现）。
- iOS、macOS、Linux 桌面客户端（保留扩展空间）。

---

## 2. 名词与缩写

| 名词 | 含义 |
| --- | --- |
| Master | 主节点，部署在 Linux，调度中心与汇聚点 |
| Branch | 支节点，部署在 Windows，真实出口 |
| Client | VPN 客户端（Windows / Android） |
| Tunnel | 客户端 ↔ Master 之间，或 Branch ↔ Master 之间的加密承载通道 |
| Flow | 一个客户端发起的 L4 连接（TCP/UDP 五元组） |
| TUN | Layer 3 虚拟网卡 |
| WinTun | Windows 下高性能 TUN 驱动 |
| VpnService | Android 系统 VPN API |

---

## 3. 总体架构

### 3.1 拓扑

```
                       +-------------------+
   Windows Client ---->|                   |<---- Branch (Windows) #1
                       |                   |
   Android Client ---->|   Master (Linux)  |<---- Branch (Windows) #2
                       |   公网 IP/域名     |
   Windows Client ---->|                   |<---- Branch (Windows) #N
                       +-------------------+
```

- 所有 TCP/UDP 连接都是 **客户端/支节点 → Master 方向主动发起**。支节点和客户端都不需要公网 IP。
- Master 是唯一需要公网可达的节点。
- 客户端在 Master 上有一条长连接（控制 + 数据复用），支节点在 Master 上也有一条长连接。

### 3.2 数据通路（典型 TCP 流）

```
Client App  ─►  TUN  ─►  Client Tunnel ─►  Master ─►  Branch ─►  目标服务器
   (App socket on real Internet)                                  ▲
Client App  ◄─  TUN  ◄─  Client Tunnel ◄─  Master ◄─  Branch ◄────┘
```

1. 客户端 App 发出的 IP 包被 TUN 接管。
2. 客户端隧道模块把 IP 包封装、加密后通过隧道发到 Master。
3. Master 根据"客户端 ↔ 支节点"路由表，找到目标支节点，转发数据。
4. 支节点用真实 socket 向目标服务器建立连接并代发；目标的回包也由支节点经隧道回到 Master，再回到客户端。

### 3.3 设计取舍：L3 转发 vs L4 代理

| 方案 | 说明 | 优点 | 缺点 |
| --- | --- | --- | --- |
| **A. 纯 L3 中继** | 客户端 TUN 抓到的 IP 包原样在 Master/Branch 间传输，Branch 用原始套接字/驱动重发 | 通用、协议无关 | Windows 上要写 WinDivert/Npcap 级别的发包逻辑，且需要做 SNAT、复杂；ICMP 等支持难 |
| **B. L3 抓包 + L4 代理转发** ★ | 客户端用 TUN 抓包，但在 Master 上用用户态 TCP/IP 栈（gVisor / lwIP）把包还原成 Flow；Branch 用普通 socket 代发；UDP 单独通道 | Branch 端无需任何驱动，纯用户态实现；移植性好 | 需要在 Master 跑用户态协议栈 |
| C. 客户端自己拆 L4 | 客户端不走 TUN，做应用级 SOCKS / HTTP 代理 | 极简单 | 无法对全局流量做接管，与"VPN 客户端"要求不符 |

**最终选择方案 B**：客户端仍呈现为系统级 VPN（用户体验和 OpenVPN/WireGuard 一致），Branch 端实现复杂度被压到最低（关键，因为 Branch 在 Windows 上，并要部署多份）。

---

## 4. 组件设计

### 4.1 客户端（Windows / Android）

#### 4.1.1 模块划分

```
+--------------------------------------------------+
|                    UI / 设置                      |
+--------------------------------------------------+
|  会话管理 | 鉴权 | 配置 | 日志                    |
+--------------------------------------------------+
|     隧道层（与 Master 的 QUIC/TLS 长连接）         |
+--------------------------------------------------+
|     TUN 适配层（WinTun / Android VpnService）     |
+--------------------------------------------------+
```

#### 4.1.2 Windows 客户端

- TUN 设备：**WinTun**（WireGuard 项目开源，性能好、签名驱动）。
- 路由接管：把默认路由 `0.0.0.0/0` 指向 TUN，再为 Master 的公网 IP 写一条 `/32` 例外，避免隧道把自身控制面流量也圈进来。
- DNS：默认劫持系统 DNS 到隧道内 DNS（由 Master / Branch 解析），可配置。
- 进程语言：Go 或 Rust（推荐 Go：跨平台、生态成熟、`golang.org/x/sys/windows` 操作 TUN 方便）。
- 自启动 & 服务化：以 Windows Service 运行核心逻辑，UI 进程通过本地 IPC（命名管道）控制。

#### 4.1.3 Android 客户端

- TUN：使用系统 `android.net.VpnService`，无需驱动安装；用户首次启用时弹出系统授权。
- 后台保活：前台服务 + 通知栏。
- 应用语言：Kotlin（UI） + 共享的 Go/Rust 隧道核心（通过 `gomobile` 或 `cargo-ndk` 打包为 .aar / .so）。
- 路由策略：默认全量代理；提供"分应用代理"（基于 `VpnService.Builder.addAllowedApplication`）。

#### 4.1.4 共享隧道核心

为了减少重复实现并便于审计，Windows / Android 共用同一份 Go 写的隧道核心库，导出 C ABI / Java 绑定。

核心库职责：
- 维护与 Master 的长连接（心跳、断线重连、握手）。
- 数据面：从 TUN 读取 IP 包 → 封装 → 写隧道；从隧道读 → 解封装 → 写 TUN。
- 控制面：登录、token 刷新、节点切换通知。

### 4.2 主节点 Master（Linux）

#### 4.2.1 职责

1. **接入层**：监听客户端的隧道连接（QUIC over UDP，回退 TLS over TCP）。
2. **支节点管理**：监听支节点反向注册，维护可用支节点列表。
3. **会话编排**：为每个新登录的客户端分配/选择一个支节点；按 Flow 粒度做绑定，避免一个用户被拆到不同出口（会破坏很多目标站点的会话）。
4. **协议栈**：用 **gVisor netstack**（Go）作为用户态 TCP/IP 栈，把客户端发来的 IP 包解析成 Flow（TCP/UDP/ICMP），再以 Flow 形式下发给支节点。
5. **鉴权与配额**：用户登录、token 颁发、按账号 ACL 限制可用支节点。
6. **可观测**：Prometheus metrics、结构化日志、审计日志。

#### 4.2.2 内部模块

```
+-----------------------------------------------------------+
|                       Admin / API                          |
+-----------------------------------------------------------+
| AuthN/AuthZ | UserStore | BranchRegistry | Scheduler       |
+-----------------------------------------------------------+
| ClientGateway (QUIC/TLS)    | BranchGateway (QUIC/TLS)     |
+-----------------------------------------------------------+
| Userland TCP/IP (gVisor netstack)                          |
+-----------------------------------------------------------+
| Flow Multiplexer  (Client <-> Flow <-> Branch)             |
+-----------------------------------------------------------+
```

#### 4.2.3 调度策略（Scheduler）

输入：当前在线的支节点列表（含负载、地理位置、健康状态）、客户端的标签 / 计划。

策略（优先级从高到低）：

1. **会话粘性**：同一客户端在会话存续期内尽量绑定同一支节点。
2. **目标粘性**：同一 (clientId, dstIP) 在短时间内（如 10 分钟）粘到同一支节点，避免目标站点 IP 跳变导致掉登录。
3. **负载均衡**：按支节点当前连接数 / 带宽加权随机。
4. **用户策略**：允许账号配置"必须使用某地区支节点"。

切换支节点：仅在支节点掉线或显式管理操作下触发；切换时已存在的 TCP Flow 会被 RST（无法热迁移），新 Flow 走新支节点。

#### 4.2.4 路由表（核心数据结构）

```
ClientConn:
  clientId, userId, tunnelCtx, virtualIP (从 100.64.0.0/10 池子分配)
BranchConn:
  branchId, tunnelCtx, capacity, healthMetrics
Flow:
  flowId            -- Master 内部全局递增
  clientId          -- 哪个客户端
  branchId          -- 路由到哪个支节点
  l4proto, srcIP/Port (虚拟), dstIP/Port (真实)
  state, lastActiveAt
```

通过两张索引：`flowId → Flow` 与 `(clientId, l4 五元组) → Flow`，实现 O(1) 转发。

### 4.3 支节点 Branch（Windows）

#### 4.3.1 职责

- 启动时**主动**连到 Master（NAT 友好）。
- 接收 Master 下发的 Flow 指令：
  - `OpenTCP(flowId, dstIP, dstPort)` → 本地用 `socket()` 建立真实连接。
  - `OpenUDP(flowId, dstIP, dstPort)` → 本地分配 UDP socket。
  - `Data(flowId, payload, direction)` → 透传字节。
  - `Close(flowId)` → 关闭对应 socket。
- 上报心跳、连接数、流量、CPU/内存。

#### 4.3.2 实现要点

- 语言：Go（与 Master 共享协议结构体，开发效率高）。Windows 上以 Service 方式运行。
- **无需驱动**：Branch 不抓包、不写包，纯应用层 socket，行为与普通客户端无异，绕开 Windows 防火墙/EDR 的高敏感操作。
- 出口 IP：默认走本机首选路由；可配置绑定特定网卡 / 多 IP 轮换。
- DNS：支节点用本地 DNS 解析（这才是用户真正想要的"出口地"的 DNS 视角）。
- 并发模型：每个 Flow 一个 goroutine + 一个 `net.Conn`，由统一的隧道多路复用器写回 Master。

#### 4.3.3 限制

- 因为支节点用普通 socket，目标若主动 ping ICMP 客户端，是不可代理的（这是绝大多数 SOCKS-like VPN 的共同限制）。
- 客户端来自的"IP 包"在 Master 处终结成 Flow，所以不能透传任意 L3 协议（如 GRE、ESP）。当前需求只涉及上网代理，可接受。

---

## 5. 通信协议

### 5.1 承载层

- **首选**：**QUIC**（IETF QUIC，0-RTT、内置 TLS 1.3、多路复用、连接迁移）。客户端 ↔ Master 与 Branch ↔ Master 都用 QUIC。
- **回退**：在严格防火墙网络下回退到 **TLS 1.3 over TCP 443**；同样的应用层协议。
- 证书：Master 使用受信任 CA 签发的证书；客户端与 Branch 出厂内置 Master 的公钥指纹（pinning），防 MITM。

### 5.2 应用层帧格式

应用层在 QUIC 流之上再走一个简单的二进制协议，字段使用 protobuf 序列化（便于演进）。

```
Frame {
  uint8   type;      // 见下
  uint64  flowId;    // 0 表示控制帧
  bytes   payload;   // protobuf 编码
}
```

`type` 枚举：

| 值 | 名称 | 方向 | 说明 |
| --- | --- | --- | --- |
| 0x01 | HELLO | C/B → M | 握手，附身份与版本 |
| 0x02 | HELLO_ACK | M → C/B | 分配虚拟 IP / branchId |
| 0x10 | OPEN_TCP | M → B | 让 Branch 建立 TCP 连接 |
| 0x11 | OPEN_UDP | M → B | 让 Branch 准备 UDP socket |
| 0x12 | OPEN_OK / FAIL | B → M | 建连结果 |
| 0x20 | DATA | 双向 | Flow 数据 |
| 0x21 | CLOSE | 双向 | 关闭 Flow |
| 0x30 | PING / PONG | 双向 | 心跳 |
| 0x40 | METRICS | B → M | 上报指标 |
| 0x50 | KICK | M → C | 服务端踢用户 |

### 5.3 客户端 ↔ Master 的数据帧

客户端发的 DATA payload 直接是 **原始 IP 包**（已经从 TUN 读到）。Master 把它喂给用户态 TCP/IP 栈解析。回程同理：Master 把 Branch 返回的 Flow 字节流通过协议栈封装成 IP 包写回客户端隧道。

这种 "客户端 ↔ Master 走 L3，Master ↔ Branch 走 L4" 的两段式设计是本架构的关键。

### 5.4 鉴权

- 用户名/密码或 OIDC 登录，颁发短期 JWT。
- 客户端在 HELLO 里带 JWT。Master 验签后建立会话。
- Branch 使用预共享的 `branchId + secret`（也可换证书），不接入用户体系。

### 5.5 心跳与重连

- 客户端 / Branch 每 15s 发 PING；30s 未收到 PONG → 重连。
- QUIC 连接迁移：移动客户端切换 Wi-Fi/4G 时利用 QUIC 的 connection migration 不中断隧道。
- 重连指数退避：1s, 2s, 4s, 8s, 最长 30s。

---

## 6. 关键流程

### 6.1 支节点上线

1. Branch 启动，读取本地配置（Master 地址、branchId、secret）。
2. 与 Master 建立 QUIC。
3. 发 HELLO（含 `branchId`、版本、机器指纹、可用出口 IP 列表）。
4. Master 校验 secret，写入 `BranchRegistry`，返回 HELLO_ACK。
5. 进入心跳 + 等待 OPEN_xxx 指令的循环。

### 6.2 客户端登录

1. 用户在 UI 输入账号；客户端调 Master 的 HTTPS REST `/login` 获取 JWT。
2. 客户端建立 QUIC 隧道，发 HELLO 携带 JWT 与设备信息。
3. Master 校验 JWT，从 `100.64.0.0/10` 池分配一个虚拟 IP（如 `100.64.3.7`），返回 HELLO_ACK。
4. 客户端用该虚拟 IP 配置 TUN，并设置默认路由。
5. 调度器为该客户端选定（默认）支节点，但**先不下发**，等出现第一个 Flow 再绑定。

### 6.3 一次 TCP 请求的完整生命周期

```
Browser ──TCP SYN──► TUN ──IP包──► Client隧道 ──DATA──► Master
                                                            │
                                  netstack 解析为 Flow ◄────┘
                                                            │
                Master 调度器选 Branch B1                    │
                                                            ▼
                          M ──OPEN_TCP(flowId,dstIP,dstPort)──► B1
                          M ◄──OPEN_OK──── B1 (socket 已建)
                          M ──DATA(flowId, "GET / ...")──► B1
                                          B1 socket.send 到目标
                                          B1 socket.recv 收到响应
                          M ◄──DATA(flowId, "HTTP/1.1 200")── B1
                                                            │
                  netstack 封成 IP 包                        │
                                                            ▼
                Master ──DATA──► Client隧道 ──IP包──► TUN ──► Browser
```

UDP 流程类似，但是无握手；Flow 在多少秒空闲后由 Master 端 timer 触发 CLOSE。

### 6.4 支节点掉线

- Master 心跳超时检测到 B1 掉线。
- 找到所有 `branchId == B1` 的 Flow，向客户端注入 RST（TCP）或直接释放（UDP）。
- 已绑定 B1 的客户端，下一个 Flow 时重新由调度器选支节点。
- 客户端**无感知**（除了短暂的连接重置，与正常网络抖动一致）。

### 6.5 客户端断线

- Master 检测到客户端隧道断开，回收虚拟 IP；通知所有相关 Branch 关闭这个客户端名下的全部 Flow。

---

## 7. 安全设计

| 风险 | 缓解措施 |
| --- | --- |
| 中间人攻击 | TLS 1.3 + 证书钉扎（pinning）|
| 凭据泄露 | JWT 短期有效 + 刷新 token；支持服务端踢人 |
| 重放攻击 | QUIC/TLS 内置 anti-replay；应用层帧带递增 seq |
| 支节点伪造 | Branch 双向 TLS 或预共享强 secret |
| 客户端互嗅 | 不同客户端的虚拟 IP 互不可见；Master 不做客户端间转发 |
| 支节点滥用 | 支节点上加出站 ACL（黑名单 / 白名单网段、端口）|
| 日志合规 | 默认只记录元数据（时间、源虚拟 IP、目的 IP/Port、流量），不记 payload；可关闭 |
| DDoS / 扫描 | Master 前置速率限制、登录失败惩罚、fail2ban |

---

## 8. 性能与可扩展性

### 8.1 容量目标（v1）

- 单 Master：≥ 2,000 并发客户端，≥ 50 支节点，聚合带宽 ≥ 1 Gbps。
- 单 Branch：≥ 500 并发 Flow，带宽视本地出口而定。
- 端到端延迟开销：在 Master 与 Branch 同区域时，相比直连增加 < 30ms（典型）。

### 8.2 关键优化

- Master 用 Go + netstack，连接读写采用每 Flow 一组 goroutine，避免单一大循环；I/O 路径不做拷贝（`io.Copy` + buffer 池）。
- 隧道层启用 QUIC 多流，避免单 Flow 拥塞影响其他 Flow（HOL 问题）。
- Branch 上对 socket buffer、`SO_SNDBUF/SO_RCVBUF` 显式调优。
- Master 跨 NUMA 节点时绑核（`GOMAXPROCS` + cgroups cpuset）。

### 8.3 横向扩展（v2+）

- Master 水平扩展：前置 L4 LB（如 IPVS），客户端按 sticky hash；Branch 同样按 hash 注册到某个 Master 实例。
- 状态共享：Flow 元数据在单个 Master 内即可，跨 Master 不必同步（客户端会话不漂移）。

---

## 9. 部署与运维

### 9.1 部署形态

| 角色 | 平台 | 形态 |
| --- | --- | --- |
| Master | Ubuntu 22.04 / Debian 12 | systemd unit + 二进制；可选 Docker 镜像 |
| Branch | Windows 10/11、Windows Server 2019+ | Windows Service（用 `nssm` 或自带 `service.go` 注册），MSI 安装包 |
| Windows Client | Windows 10/11 | MSI，含 WinTun 驱动 |
| Android Client | Android 9+ | APK |

### 9.2 配置

- Master `master.yaml`：监听端口、证书路径、用户库、虚拟 IP 池、调度策略。
- Branch `branch.yaml`：Master 地址、branchId、secret、出口 ACL、并发上限。
- 客户端：登录后配置由服务端下发（端点列表、DoH 选项、分流规则）。

### 9.3 监控

- Master 暴露 `/metrics`：在线客户端数、Flow 数、按 Branch 的吞吐、调度耗时直方图、错误码计数。
- Branch 主动上报：CPU、内存、连接数、本地出口探测延迟。
- 日志：JSON 行式，按天滚动。

### 9.4 升级

- 协议帧带版本号；Master 兼容 N、N-1 客户端 / Branch。
- 客户端启动时拉取最新版本号；不强制升级但提示。
- Branch 支持热升级（先停接受新 Flow，老 Flow 排空后替换）。

---

## 10. 技术选型汇总

| 层次 | 选型 | 备注 |
| --- | --- | --- |
| 隧道核心语言 | Go 1.22+ | 跨平台、netstack、gomobile |
| 用户态协议栈 | `gvisor.dev/gvisor/pkg/tcpip` (netstack) | 成熟、可独立使用 |
| QUIC 库 | `quic-go` | 支持 IETF QUIC、连接迁移 |
| Windows TUN | WinTun | WireGuard 维护，签名驱动 |
| Android TUN | `android.net.VpnService` | 系统级 API |
| Windows UI | WPF (.NET 8) 或 Wails (Go + WebView2) | 倾向 Wails，技术栈统一 |
| Android UI | Kotlin + Jetpack Compose | — |
| 序列化 | protobuf | — |
| 鉴权 | OIDC + JWT | 可对接企业 IdP |
| 持久化 | Master 用 SQLite（小规模）/ PostgreSQL | 用户、审计 |
| 部署 | systemd / Windows Service | — |
| CI | GitHub Actions | 多平台交叉编译 |

---

## 11. 实施阶段

> 不给具体日历周期；按"投入面"描述难度，每个阶段都内含设计 / 实现 / 测试 / 文档。

### 阶段 0：原型打通（最小可用链路）

涉及子系统：Master 网络层、Branch 网络层、客户端隧道核心（Windows 一端）。

- 用最简协议（无鉴权、无 TLS）跑通 Windows Client → Linux Master → Windows Branch → Internet 的 HTTP GET。
- 没有 UI，纯命令行；写死一个支节点。
- 目的是验证 netstack + 隧道帧格式是可行的。

### 阶段 1：MVP

涉及：协议增强、鉴权、调度、Android 客户端、Windows UI。

- 加 QUIC + TLS + JWT。
- Master 实现多 Branch 注册与基础调度。
- Android 端跑通。
- Windows UI、安装包。
- 基础观测（日志、Prometheus）。

### 阶段 2：生产就绪

- HA：Master 主备 + 健康检查；Branch 优雅升级。
- 安全加固：证书钉扎、审计日志、出站 ACL。
- 性能压测与调优，达到 §8.1 容量目标。
- 运维工具：MSI / APK 自动化构建发布；远程下发配置；远程吊销用户。

### 阶段 3：扩展能力

- 分应用代理（Android 已具备，Windows 用 WFP 实现）。
- 自定义分流规则（按域名 / 网段走 / 不走 VPN）。
- 多 Master 水平扩展。

---

## 12. 风险与未决问题

1. **Windows 上 WinTun 驱动需要数字签名**：发布前需取得 EV 代码签名证书或使用 WinTun 上游驱动（推荐后者，避免自签）。
2. **gVisor netstack 性能上限**：单实例 1 Gbps 量级可达；超过需要分片或换 eBPF/XDP 方案。需在阶段 0 末做压测确认。
3. **Android 后台被杀**：电池优化、白名单需要在文档里指导用户；前台服务尽量避免被系统回收。
4. **UDP / QUIC 流的会话识别**：第三方网站若在多个 UDP 包之间维持状态（如某些游戏），出口 IP 切换会导致断线；需通过"目标粘性"缓解。
5. **国内运营商 QoS / QUIC 被限速**：必须保留 TCP/443 回退路径。
6. **支节点 Windows 杀软误报**：Branch 应申请代码签名证书；并预提交到主流 AV 厂商。
7. **合规与日志保留**：不同部署地区要求不同，配置项要可关 / 可调。
8. **NAT 类型不友好的 Branch**：本设计已经把 Branch 设为出方向连接，绝大多数 NAT 都不是问题；极端对称 NAT 下 QUIC 也可工作（因为不打洞）。

---

## 13. 附录

### 13.1 Master 核心伪代码（节选）

```go
// 客户端隧道读循环
for frame := range clientConn.Read() {
    switch frame.Type {
    case DATA:
        ipPkt := frame.Payload
        // 喂给用户态协议栈
        netstack.Inject(clientId, ipPkt)
    case PING:
        clientConn.Write(Pong)
    }
}

// 用户态协议栈回调：新 Flow
netstack.OnNewFlow = func(f *Flow) {
    branch := scheduler.Pick(f.ClientId, f.DstIP)
    branch.Send(OpenTCP{f.Id, f.DstIP, f.DstPort})
    flowTable.Bind(f.Id, f.ClientId, branch.Id)
}

// 用户态协议栈回调：Flow 有上行数据
netstack.OnFlowData = func(f *Flow, data []byte) {
    branch := flowTable.GetBranch(f.Id)
    branch.Send(Data{f.Id, data})
}

// 支节点读循环
for frame := range branchConn.Read() {
    switch frame.Type {
    case DATA:
        f := flowTable.Get(frame.FlowId)
        // 写回用户态协议栈，由其封包 → 客户端
        netstack.WriteToFlow(f, frame.Payload)
    case CLOSE:
        netstack.CloseFlow(frame.FlowId)
    }
}
```

### 13.2 数据流时序图（TCP）

```
Client          Master              Branch         Internet
  | TUN: SYN     |                    |               |
  |--DATA(IP)--->|                    |               |
  |              | netstack: new flow |               |
  |              |---OPEN_TCP-------->|               |
  |              |                    |---SYN-------->|
  |              |                    |<--SYN/ACK-----|
  |              |<--OPEN_OK----------|               |
  |              | netstack: SYN/ACK  |               |
  |<--DATA(IP)---|                    |               |
  | TUN: ACK     |                    |               |
  |--DATA(IP)--->|                    |               |
  |              | netstack: data     |               |
  |              |---DATA------------>|               |
  |              |                    |---DATA------->|
  |              |                    |<--DATA--------|
  |              |<--DATA-------------|               |
  |<--DATA(IP)---|                    |               |
  | TUN: data    |                    |               |
```

### 13.3 端口与防火墙

| 角色 | 监听 | 出方向需要 |
| --- | --- | --- |
| Master | UDP/443 (QUIC), TCP/443 (回退), TCP/8443 (管理 API) | — |
| Branch | — | UDP/443、TCP/443 到 Master |
| Client | — | UDP/443、TCP/443 到 Master |

---

*以上为 v0.1 草案；评审后将拆分为子设计文档：协议详规、Master 详细设计、Branch 详细设计、客户端详细设计。*
