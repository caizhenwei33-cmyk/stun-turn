# STUN/TURN 服务器搭建教程

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> 一套完整的 STUN/TURN 服务器 (coturn) 搭建与配置指南，涵盖原理讲解、Docker 部署、原生安装、安全加固及常见问题排查。
>
> A complete guide to building and configuring a STUN/TURN server (coturn), covering principles, Docker deployment, native installation, security hardening, and troubleshooting.

---

## 📑 目录 / Table of Contents

- [1. 概述 / Overview](#1-概述--overview)
- [2. 原理讲解 / How It Works](#2-原理讲解--how-it-works)
  - [2.1 STUN 协议](#21-stun-协议)
  - [2.2 TURN 协议](#22-turn-协议)
  - [2.3 交互流程](#23-交互流程)
  - [2.4 应用场景](#24-应用场景)
- [3. 准备工作 / Prerequisites](#3-准备工作--prerequisites)
- [4. 原生安装 coturn / Native Installation](#4-原生安装-coturn--native-installation)
  - [4.1 Ubuntu/Debian 安装](#41-ubuntudebian-安装)
  - [4.2 CentOS/RHEL 安装](#42-centosrhel-安装)
  - [4.3 基本配置](#43-基本配置)
  - [4.4 启动与验证](#44-启动与验证)
- [5. Docker 部署 / Docker Deployment](#5-docker-部署--docker-deployment)
  - [5.1 使用 docker run](#51-使用-docker-run)
  - [5.2 使用 docker-compose](#52-使用-docker-compose)
  - [5.3 验证 Docker 部署](#53-验证-docker-部署)
- [6. 配置详解 / Configuration Reference](#6-配置详解--configuration-reference)
  - [6.1 认证配置](#61-认证配置)
  - [6.2 端口与网络](#62-端口与网络)
  - [6.3 TLS/SSL 配置](#63-tlsssl-配置)
  - [6.4 速率限制与安全](#64-速率限制与安全)
  - [6.5 日志与调试](#65-日志与调试)
- [7. 完整配置示例 / Full Config Examples](#7-完整配置示例--full-config-examples)
  - [7.1 基础配置（无认证）](#71-基础配置无认证)
  - [7.2 生产配置（TLS + 认证）](#72-生产配置tls--认证)
- [8. 常用命令 / Common Commands](#8-常用命令--common-commands)
- [9. 客户端集成 / Client Integration](#9-客户端集成--client-integration)
  - [9.1 WebRTC 浏览器端](#91-webrtc-浏览器端)
  - [9.2 Python 示例](#92-python-示例)
  - [9.3 iOS/Android](#93-iosandroid)
- [10. 常见问题 / FAQ](#10-常见问题--faq)
- [11. 安全最佳实践 / Security Best Practices](#11-安全最佳实践--security-best-practices)
- [12. 参考资源 / References](#12-参考资源--references)

---

## 1. 概述 / Overview

### 中文

STUN（Session Traversal Utilities for NAT）和 TURN（Traversal Using Relays around NAT）是 WebRTC、VoIP 和 P2P 通信中不可或缺的协议。它们帮助设备在复杂的网络环境（如防火墙、NAT）中建立直接或中继连接。

- **STUN**：客户端通过向 STUN 服务器发送请求，获取自己的公网 IP 地址和端口，从而建立直接的 P2P 连接。适用于端点处于简单 NAT 后方的场景。
- **TURN**：当 P2P 直连失败时（如对称 NAT），TURN 服务器作为中继转发媒体流。TURN 会消耗服务器带宽，但保证连接可靠性。

本教程使用 [coturn](https://github.com/coturn/coturn) —— 最流行的开源 STUN/TURN 服务器实现。

### English

STUN (Session Traversal Utilities for NAT) and TURN (Traversal Using Relays around NAT) are essential protocols for WebRTC, VoIP, and P2P communications. They help devices establish direct or relayed connections across complex network environments like firewalls and NATs.

- **STUN**: The client sends a request to a STUN server to discover its own public IP address and port, enabling direct P2P connections. Works when endpoints are behind simple NATs.
- **TURN**: When P2P direct connections fail (e.g., symmetric NAT), the TURN server relays media traffic. TURN consumes server bandwidth but guarantees connectivity.

This tutorial uses [coturn](https://github.com/coturn/coturn) — the most popular open-source STUN/TURN server implementation.

---

## 2. 原理讲解 / How It Works

### 2.1 STUN 协议

#### 中文

STUN 基于请求-响应模型。客户端向 STUN 服务器发送 Binding Request，服务器回复 Binding Response，其中包含客户端在服务器端看到的公网 IP 和端口（称为"反射地址"）。

STUN 协议的特点：
| 特性 | 说明 |
|------|------|
| 传输层 | 默认使用 UDP (端口 3478)，也支持 TCP |
| 报文结构 | 固定头部 20 字节 + 属性（Attribute），如 MAPPED-ADDRESS |
| 消息类型 | Binding Request (0x0001)、Binding Response (0x0101)、Binding Error Response (0x0111) |
| RFC 标准 | RFC 5389 (STUN)、RFC 8489 (STUN 修订版) |

#### English

STUN follows a request-response model. The client sends a Binding Request to the STUN server, which replies with a Binding Response containing the client's public IP and port as seen by the server (called the "reflexive address").

STUN protocol features:
| Feature | Description |
|---------|-------------|
| Transport | Defaults to UDP (port 3478), also supports TCP |
| Message Structure | 20-byte fixed header + Attributes (e.g., MAPPED-ADDRESS) |
| Message Types | Binding Request (0x0001), Binding Response (0x0101), Binding Error Response (0x0111) |
| RFC Standards | RFC 5389 (STUN), RFC 8489 (STUN bis) |

### 2.2 TURN 协议

#### 中文

TURN 在 STUN 基础上增加了中继功能。客户端通过 TURN 服务器分配一个中继地址（Relay Address），然后让对端向该地址发送数据，由服务器转发。

TURN 的关键概念：
| 概念 | 说明 |
|------|------|
| Allocation | 客户端在 TURN 服务器上分配的中继资源 |
| Permission | 允许向特定对端地址发送数据的权限 |
| Channel | 通过信道号（Channel Number）减少报文开销的优化机制 |
| 带宽消耗 | 每个中继流消耗双倍带宽（上行 + 下行转发） |

TURN 使用端口 3478 (UDP/TCP) 和 5349 (TLS)。分配的中继端口范围建议在 49152-65535 之间。

#### English

TURN extends STUN with relay functionality. The client requests an allocation from the TURN server, obtaining a relay address. The remote peer sends data to this relay address, and the server forwards it.

Key TURN concepts:
| Concept | Description |
|---------|-------------|
| Allocation | A relay resource allocated by the client on the TURN server |
| Permission | Authorization for a specific peer address to send data |
| Channel | Optimized mechanism using channel numbers to reduce overhead |
| Bandwidth | Each relay stream consumes 2x bandwidth (upstream + downstream relay) |

TURN uses ports 3478 (UDP/TCP) and 5349 (TLS). The recommended relay port range is 49152-65535.

### 2.3 交互流程 / Interaction Flow

```
Client A                    STUN/TURN Server                Client B
    |                             |                             |
    |--- STUN Binding Request --->|                             |
    |<-- STUN Binding Response ---|                             |
    |    (Public IP:Port A)       |                             |
    |                             |                             |
    |--- TURN Allocate Request -->|                             |
    |<-- TURN Allocate Response --|                             |
    |    (Relay Address: R_A)     |                             |
    |                             |                             |
    |--- CreatePermission(B) ---->|                             |
    |<-- Permission Response ---- |                             |
    |                             |                             |
    |===== Data via Relay ========|======= Data via Relay =====|
```

### 2.4 应用场景 / Application Scenarios

| 场景 | 使用协议 | 说明 |
|------|---------|------|
| WebRTC 视频通话 | STUN + TURN | 首选 STUN 直连，失败时回退到 TURN |
| VoIP 电话系统 | STUN + TURN | 确保 SIP 信令和 RTP 媒体流穿越 NAT |
| 在线游戏 | STUN | 减少延迟，尽可能使用 P2P 直连 |
| IoT 设备通信 | TURN | 设备位于受限网络中，需要中继 |
| P2P 文件传输 | STUN + TURN | 大文件使用直连，中继作为备选 |

---

## 3. 准备工作 / Prerequisites

### 中文

在开始之前，请确保：

- **一台公网服务器**（VPS 或云服务器），建议至少 1GB 内存
- **开放防火墙端口**：
  - 3478/UDP,TCP — STUN/TURN 服务端口
  - 5349/UDP,TCP — STUN/TURN TLS 端口
  - 49152-65535/UDP — TURN 中继端口范围
- **域名**（可选，用于 TLS 证书）
- **root 或 sudo 权限**

### English

Before you begin, ensure you have:

- **A public server** (VPS or cloud instance), recommended 1GB+ RAM
- **Open firewall ports**:
  - 3478/UDP,TCP — STUN/TURN service port
  - 5349/UDP,TCP — STUN/TURN TLS port
  - 49152-65535/UDP — TURN relay port range
- **A domain name** (optional, for TLS certificates)
- **Root or sudo access**

---

## 4. 原生安装 coturn / Native Installation

### 4.1 Ubuntu/Debian 安装

```bash
# 更新包管理器
sudo apt update

# 安装 coturn
sudo apt install -y coturn

# 检查版本
turnserver --version

# 启用 coturn 服务（编辑配置文件）
sudo sed -i 's/#TURNSERVER_ENABLED=1/TURNSERVER_ENABLED=1/' /etc/default/coturn
```

### 4.2 CentOS/RHEL 安装

```bash
# 安装 EPEL 源（如尚未安装）
sudo yum install -y epel-release

# 安装 coturn
sudo yum install -y coturn

# 检查版本
turnserver --version

# 启用服务
sudo systemctl enable coturn
```

### 4.3 基本配置

创建配置文件 `/etc/turnserver.conf`：

```ini
# === 基础设置 ===
listening-device=eth0
listening-ip=0.0.0.0
listening-port=3478
tls-listening-port=5349
relay-device=eth0
relay-ip=0.0.0.0
relay-threads=4

# === 外部 IP 配置 ===
# 如果你的服务器有多个 IP，或者位于 NAT 后方，需要指定外部 IP
external-ip=YOUR_SERVER_PUBLIC_IP

# === 中继端口范围 ===
min-port=49152
max-port=65535

# === 认证配置 ===
fingerprint
lt-cred-mech
userdb=/var/lib/turn/turndb
user=testuser:testpass

# === 日志配置 ===
verbose
syslog
```

### 4.4 启动与验证

```bash
# 启动 coturn
sudo systemctl restart coturn
sudo systemctl enable coturn

# 查看状态
sudo systemctl status coturn

# 查看端口监听
ss -tulpn | grep turnserver

# 测试 STUN 功能（使用 trickle ICE）
# 访问 https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/
# 输入 STUN 地址：stun:YOUR_SERVER_IP:3478

# 命令行测试（安装 stuntman-client）
sudo apt install -y stuntman-client
stunclient YOUR_SERVER_IP 3478
```

---

## 5. Docker 部署 / Docker Deployment

### 5.1 使用 docker run

```bash
# 拉取 coturn 镜像
docker pull coturn/coturn

# 运行 STUN/TURN 容器
docker run -d --name=coturn \
  --network=host \
  -e DETECT_EXTERNAL_IP=yes \
  coturn/coturn \
  -n \
  --listening-ip=0.0.0.0 \
  --listening-port=3478 \
  --tls-listening-port=5349 \
  --min-port=49152 \
  --max-port=65535 \
  --fingerprint \
  --lt-cred-mech \
  --user=testuser:testpass \
  --realm=example.com \
  --log-file=stdout
```

参数说明：
| 参数 | 说明 |
|------|------|
| `--network=host` | 使用宿主机网络，避免端口映射问题 |
| `-n` | 不加载默认配置文件 |
| `--lt-cred-mech` | 使用长期凭证机制 |
| `--fingerprint` | 在消息中添加指纹属性 |
| `--realm` | 认证域 |

### 5.2 使用 docker-compose

创建 `docker-compose.yml`：

```yaml
version: '3.8'

services:
  coturn:
    image: coturn/coturn:latest
    container_name: coturn
    network_mode: host
    restart: unless-stopped
    environment:
      - DETECT_EXTERNAL_IP=yes
    volumes:
      - ./turnserver.conf:/etc/coturn/turnserver.conf:ro
      - ./certs:/etc/coturn/certs:ro
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

同时创建 `turnserver.conf` 在同目录：

```ini
listening-ip=0.0.0.0
listening-port=3478
tls-listening-port=5349
relay-ip=0.0.0.0
external-ip=YOUR_SERVER_PUBLIC_IP
min-port=49152
max-port=65535
fingerprint
lt-cred-mech
user=testuser:testpass
realm=example.com
log-file=stdout
cert=/etc/coturn/certs/fullchain.pem
pkey=/etc/coturn/certs/privkey.pem
```

启动：

```bash
docker-compose up -d
docker-compose logs -f
```

### 5.3 验证 Docker 部署

```bash
# 检查容器运行状态
docker ps -a | grep coturn

# 查看日志
docker logs coturn | tail -20

# 测试 STUN
docker exec coturn turnadmin -l
```

---

## 6. 配置详解 / Configuration Reference

### 6.1 认证配置 / Authentication

coturn 支持多种认证方式：

```ini
# === 长期凭证机制 (推荐) ===
lt-cred-mech
# 用户数据库文件
userdb=/var/lib/turn/turndb
# 添加用户（使用 turnadmin）
# turnadmin -a -u username -p password -r example.com

# === 静态凭证（仅测试用）===
# user=username:password

# === REST API 认证（配合 Web 服务）===
# use-auth-secret
# static-auth-secret=YOUR_SHARED_SECRET

# === 无需认证（仅 STUN，公共服务）===
# no-auth
```

管理用户：

```bash
# 添加用户
turnadmin -a -u alice -p pass123 -r example.com -b /var/lib/turn/turndb

# 删除用户
turnadmin -d -u alice -r example.com -b /var/lib/turn/turndb

# 列出所有用户
turnadmin -l -b /var/lib/turn/turndb
```

### 6.2 端口与网络 / Ports & Network

```ini
# 监听设备
listening-device=eth0

# 监听地址
listening-ip=0.0.0.0

# STUN/TURN 端口（默认 3478）
listening-port=3478

# TLS 端口（默认 5349）
tls-listening-port=5349

# 备用端口
# alternate-server=YOUR_SERVER_IP:3479
# alt-tls-listening-port=5350

# 中继端口范围
min-port=49152
max-port=65535

# TURN 中继地址（可指定多个）
relay-ip=0.0.0.0
# relay-ip=192.168.1.100

# 外部 IP（用于 NAT 场景）
external-ip=203.0.113.10
# 内部:外部 映射
# external-ip=192.168.1.100/203.0.113.10
```

### 6.3 TLS/SSL 配置 / TLS Configuration

使用 Let's Encrypt 获取免费证书：

```bash
# 安装 certbot
sudo apt install -y certbot

# 获取证书
sudo certbot certonly --standalone -d turn.example.com

# 证书路径
# /etc/letsencrypt/live/turn.example.com/fullchain.pem
# /etc/letsencrypt/live/turn.example.com/privkey.pem
```

配置文件中使用：

```ini
# TLS 证书路径
cert=/etc/letsencrypt/live/turn.example.com/fullchain.pem
pkey=/etc/letsencrypt/live/turn.example.com/privkey.pem

# 要求客户端使用 TLS
# no-tlsv1
# no-tlsv1_1

# 设置密码
# cipher-list="ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384"
```

### 6.4 速率限制与安全 / Rate Limiting & Security

```ini
# === 速率限制 ===
# 每分钟最多分配数
total-quota=100

# 每个用户最多分配数
user-quota=10

# 相同 IP 的最大分配数
# max-allocate-per-ip=5

# === 访问控制 ===
# 允许特定 IP 范围
allowed-peer-ip=0.0.0.0-255.255.255.255

# 拒绝特定 IP 范围
denied-peer-ip=10.0.0.0-10.255.255.255
denied-peer-ip=172.16.0.0-172.31.255.255
denied-peer-ip=192.168.0.0-192.168.255.255

# === 安全加固 ===
# 无 TLS 连接可选（可选）
# no-tls

# 无 DTLS（可选）
# no-dtls

# 移动端优化
# mobility

# 关闭中继信息泄露
# no-stun-backward-compatibility
```

### 6.5 日志与调试 / Logging & Debug

```ini
# 详细日志
verbose

# 超详细调试
# debug

# 日志到 syslog
syslog

# 日志到文件
# log-file=/var/log/turnserver/turnserver.log

# 简单日志（仅少量信息）
# simple-log
```

查看日志：

```bash
# 查看 syslog
sudo journalctl -u coturn -f

# 查看 Docker 日志
docker logs -f coturn

# 实时监控连接数
watch -n 1 'ss -tulpn | grep turnserver | wc -l'
```

---

## 7. 完整配置示例 / Full Config Examples

### 7.1 基础配置（无认证）

适用于仅需要 STUN 功能的场景，如公共 STUN 服务。

`/etc/turnserver.conf`:

```ini
# === 基础 STUN 服务 ===
listening-device=eth0
listening-ip=0.0.0.0
listening-port=3478

# 无需认证
no-auth

# 仅 STUN，禁用 TURN 中继
# （通过不设置 min-port/max-port 和 user 来禁用 TURN）

# 日志
verbose
syslog
```

### 7.2 生产配置（TLS + 认证）

适用于 WebRTC 生产环境。

`/etc/turnserver.conf`:

```ini
# === 基础网络 ===
listening-device=eth0
listening-ip=0.0.0.0
listening-port=3478
tls-listening-port=5349
relay-device=eth0
relay-ip=0.0.0.0
external-ip=YOUR_SERVER_PUBLIC_IP

# === 中继端口 ===
min-port=49152
max-port=65535
relay-threads=0

# === TLS/SSL ===
cert=/etc/letsencrypt/live/turn.example.com/fullchain.pem
pkey=/etc/letsencrypt/live/turn.example.com/privkey.pem
cipher-list="ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384"

# === 认证 ===
fingerprint
lt-cred-mech
userdb=/var/lib/turn/turndb
realm=turn.example.com

# === 用户配额 ===
total-quota=300
user-quota=10

# === 流量控制 ===
# 每个分配的带宽限制（bps）
# bps-capacity=0   # 0 表示不限制

# === 日志 ===
verbose
syslog

# === 安全 ===
no-multicast-peers
allowed-peer-ip=0.0.0.0-255.255.255.255
denied-peer-ip=10.0.0.0-10.255.255.255
denied-peer-ip=172.16.0.0-172.31.255.255
denied-peer-ip=192.168.0.0-192.168.255.255
```

---

## 8. 常用命令 / Common Commands

| 命令 | 说明 |
|------|------|
| `turnserver -c /etc/turnserver.conf` | 手动启动 turnserver |
| `turnadmin -a -u user -p pass -r realm` | 添加认证用户 |
| `turnadmin -d -u user -r realm` | 删除用户 |
| `turnadmin -l` | 列出所有用户 |
| `turnadmin -A -u user -p pass -r realm` | 添加管理员用户 |
| `turnutils_uclient -t -T -u user -w pass SERVER_IP` | 客户端测试工具 |
| `stunclient SERVER_IP 3478` | 使用 stunclient 测试 STUN |
| `ss -tulpn \| grep turnserver` | 查看监听端口 |
| `journalctl -u coturn -f` | 查看 coturn 日志 |
| `turnadmin -P -u user -r realm` | 修改用户密码 |

### 使用 turnutils_uclient 进行压力测试

```bash
# 安装工具（coturn 源码编译）
# 测试 TURN 中继
turnutils_uclient -t -T -u testuser -w testpass YOUR_SERVER_IP

# 测试 TLS 连接
turnutils_uclient -t -S -u testuser -w testpass YOUR_SERVER_IP

# 多客户端模拟（100 个并发连接）
turnutils_uclient -t -T -c 100 -u testuser -w testpass YOUR_SERVER_IP
```

---

## 9. 客户端集成 / Client Integration

### 9.1 WebRTC 浏览器端

```javascript
// ICE Server 配置
const iceServers = {
  iceServers: [
    {
      urls: 'stun:turn.example.com:3478'
    },
    {
      urls: ['turn:turn.example.com:3478', 'turns:turn.example.com:5349'],
      username: 'testuser',
      credential: 'testpass'
    }
  ]
};

// 创建 RTCPeerConnection
const pc = new RTCPeerConnection(iceServers);

// 监听 ICE 候选
pc.onicecandidate = (event) => {
  if (event.candidate) {
    console.log('ICE Candidate:', event.candidate);
  }
};

// 查看 ICE 连接状态
pc.oniceconnectionstatechange = () => {
  console.log('ICE State:', pc.iceConnectionState);
};
```

### 9.2 Python 示例

```python
import asyncio
from aiortc import RTCPeerConnection, RTCSessionDescription
from aiortc.contrib.media import MediaPlayer

async def create_peer_connection():
    pc = RTCPeerConnection()

    # 配置 ICE 服务器
    pc._iceTransport._ice_gatherer._ice_servers = [
        {
            'urls': ['stun:turn.example.com:3478']
        },
        {
            'urls': ['turn:turn.example.com:3478'],
            'username': 'testuser',
            'credential': 'testpass'
        }
    ]

    @pc.on('iceconnectionstatechange')
    def on_ice_state_change():
        print(f'ICE state: {pc.iceConnectionState}')

    # 添加媒体轨道
    player = MediaPlayer('/dev/video0', format='v4l2')
    pc.addTrack(player.video)

    return pc

pc = asyncio.run(create_peer_connection())
```

### 9.3 iOS/Android

**iOS (Swift)**:

```swift
import WebRTC

let iceServers: [RTCIceServer] = [
    RTCIceServer(urlStrings: ["stun:turn.example.com:3478"]),
    RTCIceServer(
        urlStrings: ["turn:turn.example.com:3478"],
        username: "testuser",
        credential: "testpass"
    )
]
let config = RTCConfiguration()
config.iceServers = iceServers
```

**Android (Java/Kotlin)**:

```kotlin
val iceServers = listOf(
    PeerConnection.IceServer.builder("stun:turn.example.com:3478").createIceServer(),
    PeerConnection.IceServer.builder("turn:turn.example.com:3478")
        .setUsername("testuser")
        .setPassword("testpass")
        .createIceServer()
)
```

---

## 10. 常见问题 / FAQ

### Q1: 为什么客户端无法连接 STUN 服务器？

**可能原因**：
1. 防火墙未开放端口 3478/UDP
2. 服务器绑定了错误的 IP 地址
3. NAT 环境未正确配置 `external-ip`

**排查方法**：
```bash
# 检查端口是否开放
nc -zuv YOUR_SERVER_IP 3478
# 或从外部测试
stunclient YOUR_SERVER_IP 3478
```

### Q2: TURN 中继无法工作？

**可能原因**：
1. 中继端口范围（49152-65535）未在防火墙开放
2. 未添加认证用户
3. 中继 IP 配置错误

**解决方法**：
```bash
# 确保防火墙开放中继端口
sudo ufw allow 49152:65535/udp
# 检查用户是否配置正确
turnadmin -l
```

### Q3: 如何选择合适的 min-port/max-port？

- 生产环境建议使用 49152-65535 范围
- 单服务器支撑连接数 ≈ (max - min) / 2（每个分配需要 2 个端口）
- 如果用户量少（<100 并发），可以使用 49152-50000

### Q4: 带宽消耗如何计算？

```
总带宽 (Mbps) = 同时连接数 × 每路流媒体带宽 × 2

例如：100 个并发通话，每路 2 Mbps
总消耗 = 100 × 2 × 2 = 400 Mbps（约 50 MB/s）
```

### Q5: Docker 部署时无法检测外部 IP？

```bash
# 方法 1：通过环境变量
docker run -e DETECT_EXTERNAL_IP=yes ...

# 方法 2：手动指定
docker run -e EXTERNAL_IP=YOUR_SERVER_IP ...

# 方法 3：在配置文件中指定
# external-ip=YOUR_SERVER_IP
```

### Q6: 如何轮换 TLS 证书？

```bash
# 使用 certbot 续期后自动 reload
sudo certbot renew --post-hook "systemctl reload coturn"

# Docker 方式：挂载证书目录，续期后重启容器
docker restart coturn
```

### Q7: turnserver 启动时报 "Unsupported relay address family"

**原因**：IPv6 地址族不支持，但服务器尝试使用 IPv6。
**解决**：添加 `--no-ipv6` 参数或在配置中加入 `no-ipv6`。

### Q8: 如何监控 TURN 使用情况？

```bash
# 查看活跃分配数
ss -tulpn | grep turnserver | wc -l

# 查看日志中的分配/释放事件
grep "allocation" /var/log/syslog | tail -20

# 使用 Prometheus + Grafana（需额外部署 exporter）
```

---

## 11. 安全最佳实践 / Security Best Practices

| 实践 | 说明 |
|------|------|
| 使用 TLS/DTLS | 始终启用 5349 端口的 TLS 加密 |
| 启用认证 | 长期凭证机制（lt-cred-mech） |
| 限制配额 | 设置 total-quota 和 user-quota |
| 封锁私有 IP | 拒绝 10/8, 172.16/12, 192.168/16 的 peer IP |
| 定期轮换密码 | 使用 turnadmin 定期更新用户密码 |
| 监控带宽 | 部署带宽监控，防止滥用 |
| 使用防火墙 | 仅开放必要端口 |
| 开启日志 | 保留至少 30 天日志用于审计 |
| 定期更新 | 保持 coturn 版本最新 |
| 禁用不需要的功能 | 如不需要 IPv6，添加 no-ipv6 |

```bash
# 安全加固 checklist
# 1. 防火墙规则
sudo ufw allow 3478/udp
sudo ufw allow 3478/tcp
sudo ufw allow 5349/tcp
sudo ufw allow 49152:65535/udp

# 2. 禁止 root 运行（使用专用用户）
sudo useradd -r -s /bin/false turnserver

# 3. 文件权限
sudo chown -R turnserver:turnserver /var/lib/turn
sudo chmod 700 /var/lib/turn
```

---

## 12. 参考资源 / References

- [coturn 官方仓库](https://github.com/coturn/coturn)
- [RFC 5389 — STUN](https://datatracker.ietf.org/doc/html/rfc5389)
- [RFC 8489 — STUN bis](https://datatracker.ietf.org/doc/html/rfc8489)
- [RFC 5766 — TURN](https://datatracker.ietf.org/doc/html/rfc5766)
- [RFC 8155 — TURN URI](https://datatracker.ietf.org/doc/html/rfc8155)
- [WebRTC ICE](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection)
- [trickle ICE 测试工具](https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/)

---

## ☕ 支持 / Support

如果这个教程对你有帮助，欢迎请我喝杯咖啡：

**USDT (TRC20)**
```
TVbQerV1SF4MXB1JCcAzQxarewHwEPYTKm
```
