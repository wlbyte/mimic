# Mimic 动态对端缓存（Peer Cache）机制与功能实现汇总

本文档记录了在 Mimic（UDP-to-TCP 混淆网关）中新增的 **动态对端 IP/Port 缓存（Peer Cache）** 功能的完整设计、代码实现、问题排查以及使用方法，以便后续会话快速掌握上下文。

---

## 1. 需求与设计背景

### 1.1 业务诉求
传统 Mimic 部署需要通过 `-f local=... -f remote=...` 精确指定对端（Remote）的公网 IP 和端口。但在客户端位于动态 NAT、移动漫游网络或多客户端汇聚的网关场景下：
- 对端的公网 IP 和端口是动态变化的，无法静态写死在配置文件中。
- 频繁变动会导致必须不断修改配置文件并重启 Mimic，从而引发既有流量闪断。

### 1.2 目标机制
1. **入方向自动学习（Ingress Learn）**：
   服务端接收并解包 TCP->UDP 时，把对端的源公网 IP + 端口动态缓存在 BPF Map 中。
2. **出方向定向混淆（Egress Match & Obfuscate）**：
   服务端发送业务 UDP 时，先匹配 `local=` 监听端口；匹配成功后，优先匹配配置好的 `remote=`；若未配置 `remote=`，则匹配缓存中的对端 IP + 端口。命中且活跃才执行 UDP->TCP 混淆，未命中直接裸 UDP 放行。
3. **参数控制**：
   通过 `-c / --cache` 开关控制是否启用该功能；通过 `-t / --cache-ttl` 控制超时时间。
4. **即时恢复与优雅下线**：
   客户端正常退出时主动发 TCP RST，服务端捕获并**立即删除缓存**，避免 60 秒黑洞期；对于异常掉电等非正常退出，由可调的 TTL 超时保底。

---

## 2. 核心架构与底层实现

### 2.1 BPF Map 定义 (`bpf/main.h`, `bpf/main.c`)
- **Map 名称**：`mimic_peer_cache`
- **Map 类型**：`BPF_MAP_TYPE_LRU_HASH`（容量 10240 条，系统超额时自动 LRU 淘汰）
- **数据结构**：
  ```c
  struct peer_endpoint_key {
    struct in6_addr ip; // IPv4 转换为 IPv4-mapped IPv6，IPv6 直接存储
    __u16 port;         // 主机字节序（ntohs）
    __u16 _pad;         // 对齐填充，确保 BPF Map Key 无脏字节
  };

  struct peer_info_val {
    __u64 last_seen;    // 最后活跃时间戳（bpf_ktime_get_boot_ns()）
  };
  ```

### 2.2 入方向逻辑（XDP Ingress - `bpf/ingress.c`）
1. **纯 UDP 极速透传**：
   `if (ip_proto != IPPROTO_TCP) return XDP_PASS;`
   非 TCP 流量（如客户端退出后直接发来的裸 UDP）在第 1 行直接放行，0 损耗直达应用。
2. **动态学习（缓存写入）**：
   在握手成功且校验通过的 TCP 数据路径上（`will_drop == false` 且 `use_cache == true`）：
   将对端源公网地址（`ipv4_mapped(ipv4->saddr)` 或 `ipv6->saddr`）与源端口（`ntohs(tcp->source)`）写入 `mimic_peer_cache`，并记录 `last_seen`。
3. **秒级下线感知（RST 即删）**：
   在 RST/FIN 处理路径（`tcp->rst && use_cache`）中：
   捕获客户端退出时发来的 `TCP_FLAG_RST` 报文，直接调用 `bpf_map_delete_elem(&mimic_peer_cache, &peer_key)`，彻底消除客户端正常退出后的网络黑洞。

### 2.3 出方向逻辑（TC Egress - `bpf/egress.c`）
开启 `use_cache` 时的严格匹配流：
```
                       [出方向 UDP 数据包]
                                |
                   [1. 匹配 local= (源端口/IP)]
                                |
              +-----------------+-----------------+
              | 匹配失败                          | 匹配成功
              v                                   v
        [TC_ACT_OK 放行]                [2. 匹配 remote= (目的)]
        (不篡改,发裸 UDP)                         |
                                 +----------------+----------------+
                                 | remote= 匹配                    | remote= 未配置/未命中
                                 v                                 v
                       [使用 remote 策略]                 [3. 查 peer_cache]
                       (不查 cache,直接混淆)                       |
                                                   +---------------+---------------+
                                                   | 命中且未超时                   | 未命中或超时
                                                   v                               v
                                          [使用 local 策略混淆]              [TC_ACT_OK 放行]
                                          (UDP -> TCP 伪装)                 (不篡改,发裸 UDP)
```

超时判定使用可配置的 TTL（`bpf/egress.c`）：
```c
__u32 ttl = cache_ttl > 0 ? (__u32)cache_ttl : 60;
if (val && time_diff(SECOND, tstamp, val->last_seen) < ttl) {
  settings = &local_info->settings;
} else {
  return TC_ACT_OK;
}
```

### 2.4 用户态接线 (`src/`)
- `src/main.h`：`struct run_args` 增加 `bool use_cache;` 与 `int cache_ttl;`；声明 `parse_cache_ttl`。
- `src/run.c`：
  - 命令行参数注册：`-c, --cache` 与 `-t, --cache-ttl=SECONDS`。
  - 参数注入：通过 libbpf skeleton 写入全局变量 `skel->bss->use_cache` 和 `skel->bss->cache_ttl`。
  - 缺省安全保底：未配置 `cache_ttl` 时默认置为 60 秒。
- `src/config.c`：
  - 配置文件支持 `cache = true|false` 与 `cache_ttl = 1~86400`。

---

## 3. 关键问题与排错记录

### 3.1 LLVM 投机指针算术导致 BPF Verifier 拒绝（Permission Denied -13）
* **现象**：
  加载 BPF 程序时报错：`R1 pointer arithmetic on map_value_or_null prohibited, null-check it first`。
* **根因**：
  原 `matches_whitelist()` 中使用了三元表达式：
  `return result ? &result->settings : NULL;`
  clang `-O2` 优化时将其编译成了「先给指针加上 128 偏移（`r1 += 128`），再通过分支将 NULL 赋为 0」。在 eBPF Verifier 看来，这是对尚未判空的 `PTR_TO_MAP_VALUE_OR_NULL` 进行了指针算术运算，触发硬性拦截。
* **解决**：
  重构 `bpf/main.h` 中的 `matches_whitelist()`，改成逐分支显式返回：
  ```c
  struct filter_info* result = bpf_map_lookup_elem(&mimic_whitelist, &local);
  if (result) return &result->settings;
  result = bpf_map_lookup_elem(&mimic_whitelist, &remote);
  if (result) return &result->settings;
  return NULL;
  ```
  保证所有指针偏移严格发生在判空分支之后，彻底消除 Verifier 报错。

### 3.2 客户端退出后的网络断流问题与演进分析
1. **初始问题**：
   客户端退出 mimic 后，服务端出向由于命中尚未过期的缓存，仍将回包强转为 TCP，导致客户端应用无法解析，造成持续 60 秒的单向断流。
2. **方案对比与取舍**：
   - *方案 A（XDP 入向探测裸 UDP 删缓存）*：分析后发现，在公网 NAT 场景下，客户端切回纯 UDP 后其映射的外网源端口大概率漂移，导致 key 不匹配而无法清理。
   - *方案 B（收到客户端 TCP RST 时删缓存）*：**最终采纳**。客户端退出 mimic 时会走原连接发送 `TCP_FLAG_RST`，NAT 映射未变，能 100% 精准匹配并立即删除 cache，做到 0 毫秒恢复。
   - *兜底策略（可调 Cache TTL）*：针对 `kill -9` 或断电等收不到 RST 的异常场景，将超时时间开放为参数 `-t/--cache-ttl`，用户可根据业务需求缩短（如设为 3~5 秒）。

---

## 4. 涉及文件列表

| 文件路径 | 修改性质 | 核心修改说明 |
|---|---|---|
| `bpf/main.h` | 结构体 & 逻辑 | 声明 `mimic_peer_cache`、`use_cache`、`cache_ttl`；重构 `matches_whitelist` 解决 verifier 报错 |
| `bpf/main.c` | 全局定义 | 实例化 `mimic_peer_cache SEC(".maps")`、`use_cache`、`cache_ttl` |
| `bpf/egress.c` | 出向混淆 | 实现「local 必配 -> remote 优先 -> 查 cache -> 兜底裸 UDP」的匹配逻辑 |
| `bpf/ingress.c` | 入向学习/清理 | 数据路径写入 `peer_cache`；RST 快速路径主动删除 `peer_cache` |
| `src/main.h` | 头文件声明 | `run_args` 增加配置字段；声明 `parse_cache_ttl` |
| `src/run.c` | 用户态控制 | 增加 `-c` 和 `-t` 命令行选项；写入 BPF skeleton 全局变量 |
| `src/config.c` | 配置解析 | 增加 `cache` 和 `cache_ttl` 配置文件字段支持；新增 `parse_cache_ttl` 函数 |

---

## 5. 使用说明与命令示例

### 5.1 命令行启动（服务端推荐配置）
```bash
# 只需配置本地服务监听端口（如 45000），开启动态学习，缓存有效时间 10 秒
mimic run eth0 -f local=0.0.0.0:45000 -c -t 10
```

### 5.2 配置文件形式 (`mimic.conf`)
```ini
link_type = eth
filter = local=0.0.0.0:45000
cache = true
cache_ttl = 10
```
运行方式：
```bash
mimic run eth0 -F mimic.conf
```

### 5.3 常用调试与验证命令
```bash
# 1. 编译验证
make build-cli

# 2. BPF Verifier 加载校验（需 root）
bpftool prog load out/mimic.bpf.o /sys/fs/bpf/test_mimic && rm -f /sys/fs/bpf/test_mimic

# 3. 查看 peer_cache 运行中条目
bpftool map dump name mimic_peer_cache
```
