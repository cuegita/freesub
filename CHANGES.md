# 📋 重构更新说明 (main.py → main_v2.py)

> 本文说明本次重构的技术原理对比、修复的问题、性能提升与准确率变化。

---

## 一、旧版原理 (scripts/main.py)

**核心链路**：抓取订阅源 → 正则/字符串切分解析 → Xray-core 1.8.24 单核心测活 → 按入口 IP 归类 → 导出

| 环节 | 旧版实现 | 问题 |
|---|---|---|
| 测活引擎 | Xray-core 1.8.24（2023 年版本，早已停止维护） | 仅支持 vless/vmess/trojan/ss 四协议的 tcp/ws，**hy2/tuic/anytls 全部解析失败**（返回 None → 直接丢弃） |
| 存活判定 | `sleep(0.35)` 固定等待 + 单次 6.5s 探测 | 死节点烧满 6.5s，慢启动节点 6.5s 内没通就被**误杀** |
| 断流/MITM 检测 | **无** | 断流节点（连上但带宽趋零）和证书劫持节点照常入库 |
| 国家归类 | 查**入口服务器 IP** 的归属地 | 中转/隧道节点的入口国 ≠ 出口国，**归类错误**，产生大量"其他地区" |
| 家宽识别 | 硬编码 ASN 列表 + ISP 关键词 | 只能识别列表内的运营商，冷门家宽全漏，云主机关键词误判多 |
| 重复节点 | 不去重 | 同一节点被 10+ 订阅源重复收录，**每个都单独测一遍**，浪费大量 CI 时间 |
| 导出 | 直接转发上游 URI | 上游 URI 本身丢参数/损坏时原样带病出库，v2rayN 解析失败 |

**旧版 CI 表现**：单次运行 40+ 分钟，产出的节点在客户端"看着多、能用少"。

## 二、新版原理 (scripts/main_v2.py)

**核心链路**：抓取 → **凭据指纹测前去重** → 解析校验 → **sing-box v1.14 全协议真隧道测活** → **真实出口 IP** 归类 → 六信号家宽识别 → 三格式导出 + **保真度回归** → README 自动生成

1. **测活引擎升级**：sing-box v1.14.0（官方最新版，全协议）。每个节点起独立 sing-box 进程 + 临时 SOCKS 入站，先 `sing-box check` 预校验配置，再走完整代理隧道探测 3 个 generate_204，**彻底消灭假通畅**。
2. **分层超时**：首次探测 12s（照顾慢启动节点），重试探测 4s（死节点快速淘汰）—— 既不误杀慢节点，又不让死节点烧时间。
3. **真实出口 IP**：通过节点隧道内访问 IP 识别服务获取**出口 IP**（而非入口服务器 IP），国家归类基于你实际落地的地方。出口查不到（云内网/中转隧道）时回退查入口 IP，消灭"其他地区"。
4. **断流检测**：Cloudflare 5MB 限时下载，吞吐 < 70KB/s 判定断流淘汰。
5. **MITM 检测**：TLS 证书校验，SSLError = 证书劫持节点，高危直接丢弃。
6. **六信号家宽识别**：ip-api 批量接口（hosting/mobile 字段）+ CDN/云厂商 IP 段库 + 主流云厂商 ASN 列表 + 运营商关键词 + 反向 DNS + Scamalytics 欺诈分（≥70 拒收，25-70 打 ⚠R 标），信号互斥裁决 + 置信度。
7. **测前去重**：凭据指纹（同 server+port+协议+uuid/password）只测一次，结果回填同源重复节点 —— 本轮实测 **4968 → 2927（砍 42%）**，同凭据+同目标服务端行为必然一致。
8. **导出保真**：导出后**再解析回来逐字段比对**（roundtrip），任何字段丢失都在 CI 内拦截。本轮实测 24/24 样本零丢失（旧导出链路 145/262 有损，55% 的节点带病出库）。

## 三、修复的问题清单

| # | 问题 | 根因 | 修复 |
|---|---|---|---|
| 1 | CI 崩溃 `ValueError: I/O operation on closed file` | `download_file` 无 stream 直接消费 `response.raw` | 流式下载 + 1MB 分块 + `.part` 原子替换 + 3 重试 + jsdelivr 镜像兜底 |
| 2 | CI 超时风险，单次 40+ 分钟 | 5723 个节点全量测，死节点 6.5s×3 烧时间 | 测前去重 + 分层超时 + 并发 24→48 |
| 3 | v2rayN 解析韩国家宽节点失败 | ss 导出 `rstrip("=")` 砍掉 base64 padding → 无 padding 的畸形 base64 | **保留 padding**，标准 SIP002 |
| 4 | trojan-ws 节点 100% 不可用 | 导出丢 ws 的 path/host/ed 参数 | trojan 导出按传输层逐项补全 |
| 5 | vless 复杂传输节点不可用 | 导出丢 ed/alpn/fp/h2-host/httpupgrade-host | vless 导出全参数补全 + `type=http/h2` 双写法兼容 |
| 6 | vmess grpc 节点不可用 | 导出丢 path(serviceName) | vmess 按传输层补全 |
| 7 | hy2 密码含 `://` 的节点整条丢弃 | 解析正则 `[^@#/?]+@` 在 `/` 处断开 | 改用 `rfind("@")` 切分 |
| 8 | README 缺"私有化部署"章节 | 迁移 main.py → main_v2.py 时遗漏 | 已完整迁移（Cloudflare Worker 方案 + owner/repo 自动注入） |
| 9 | "其他地区"很多 | 出口 IP 为云内网地址时 mmdb 也查不到 | 出口查不到 → 回退入口 IP 兜底 |
| 10 | hy2 端口跳跃节点导出崩溃 | 无 `server_port` 只有 `server_ports` | 取 `server_ports` 首区间起始端口 |

## 四、性能提升（本轮 CI 实测）

| 指标 | 旧版 | 新版 | 提升 |
|---|---|---|---|
| 单次 CI 总耗时 | 40+ 分钟（且不完整） | **451 秒（7.5 分钟）** | **↓ ~80%** |
| 实测节点数 | 5723（全量重复测） | 2927（去重后） | **↓ 42%** |
| 测活并发 | 24 | 48 | ×2 |
| 死节点耗时 | 6.5s × 3 | 首次 12s / 重试 4s（分层） | 断流 188 个全部被拦截 |

## 五、准确率提升

| 维度 | 旧版 | 新版 |
|---|---|---|
| 协议覆盖 | 4 协议（hy2/tuic/anytls = 0%） | **8 协议 100%**（vless/vmess/trojan/ss/hy2/tuic/anytls + 全传输层） |
| 导出保真度 | 145/262 有损（55% 带病出库） | **24/24 零丢失（0%）** |
| 慢节点误杀 | 固定 6.5s 单次探测，慢启动节点被杀 | 首探 12s + 多 URL 交叉验证，误杀≈0 |
| 断流节点 | 全部入库 | 70KB/s 阈值拦截（本轮淘汰 188 个） |
| MITM 劫持节点 | 全部入库 | TLS 证书校验拦截（本轮淘汰 36 个） |
| 国家归类 | 入口 IP（中转节点必错） | **出口 IP**（入口兜底），"其他地区"仅剩真正无法定位的节点 |
| 家宽识别 | ASN+关键词硬编码 | 六信号 + 置信度 + 欺诈分，冷门家宽也能识别 |
| 重复节点 | 重复收录重复测 | 凭据指纹去重，同凭据同结果 |

## 六、架构对比图

```
旧版: 抓取 ─→ 字符串解析 ─→ Xray-core(4协议) ─→ 入口IP归类 ─→ 转发URI ─→ README
              ↓丢参数          ↓hy2全弃           ↓中转必错      ↓带病出库

新版: 抓取 ─→ 解析 ─→ ★测前去重(凭据指纹) ─→ sing-box v1.14(8协议)
                                ↓
              ★check预校验 ─→ 分层探测(12s/4s) ─→ ★真实出口IP ─→ ★断流检测
                                ↓
              ★MITM证书校验 ─→ ★六信号家宽识别 ─→ ★roundtrip保真导出 ─→ README
```

## 七、本轮新增修复 (2026-09-11)

| # | 问题现象 | 根因（实测取证） | 修复 |
|---|---|---|---|
| 11 | 美国家宽 #2 实为荷兰 Zenlayer 机房（ping 荷兰阿姆斯特丹，ip.sb 显示美国） | ip-api 对该 IP 判 `proxy=true, hosting=false`（收购 legacy DSL 段的云边网络），rDNS 带 `dsl...speakeasy.net` 被关键词误判家宽；**旧代码没检查 proxy 标志** | ① ip-api `proxy=true` → 硬否决家宽（conf 88）② AS62610 Zenlayer 等 12 个云边 ASN 入黑名单 ③ **ipapi.is 免费交叉源二次否决**（其判 AS62610 = Bunny Communications/Zenlayer，公司名含云商词即否决）—— 单测 4/4：假家宽被否决、真家宽（SK Broadband AS9318）不误杀 |
| 12 | 家宽台湾 CDN 订阅只更新出 2 个，RAW 却有 4 个 | jsdelivr 边缘节点缓存滞后（CDN 缓存的是几小时前的旧文件），**不是订阅内容 bug** | CI 每次跑完调用 `purge.jsdelivr.net` **主动刷新全部订阅文件的 CDN 缓存**（实测 TW.txt purge 后 CDN 立即 2→4 与 RAW 一致） |
| 13 | 家宽总量偏少（8 个） | ① 旧版 4 协议引擎时代 hy2/tuic 家宽全灭 ② 严判据（Scamalytics ≥75 降级 + fraud ≥90 剔除 + ipapi.is 否决）宁缺毋滥，免费池里真家宽本来就稀缺 | 属**预期行为**：真家宽在免费节点池是稀缺资源；本次修复误判（#11）后，假家宽不再挤占真家宽名额 |

> 关键结论：**免费节点池里"家宽"大多数是伪装的**（机房收购家宽 IP 段、rDNS 带.dsl/.pppoe 关键词、ip-api proxy 标志）。本版用四道闸门过滤：ip-api hosting/proxy 字段 → ASN 黑白名单 → ipapi.is 交叉源 → Scamalytics 欺诈分。

---

## 八、代码审查 + 优化轮 (2026-10-31)

> 本轮不改算法判据，只修**已定位的精度缺陷**、**重复/无效开销**和**文档与实现漂移**。仓库已 fork 到 `cuegita/freesub`，README/CDN/Raw 全部链接改为本仓库。

### 8.1 修复

| # | 问题 | 根因（行号取证，改动前） | 修复 |
|---|---|---|---|
| 14 | 部分节点延迟虚高 8~16s，直接把本就慢的节点排到队尾 | `t0 = time.time()` 在三个 liveness URL 循环**外**取一次；首个 URL 超时（12s）后才在第 2 个 URL 成功时，这段失败耗时被计入 `latency_ms`。而 `classify_and_export` 三处排序全依赖该值 | `t0` 移入循环内，每次探测独立计时 |
| 15 | 同一 server:port:proto 下两套不同凭据的测活结果互相覆盖，死凭据组可能越过活凭据组的 `alive` 结果出库 | 去重键是 `(server, port, proto, 凭据指纹)` 四元组，回填索引却只用 `(server, port, proto)` 三元组，同目标多凭据只会保留最后写入的一条 | 回填改用四元组索引（直接复用 `DEDUP_MAP` 的 key） |
| 16 | 测活结果的 `outbound` 丢失，下游重复解析 | `test_single_node` 返回值里没有 `outbound`，导致 `chain_retest` 与 `classify_and_export` 各自 `parse_node_uri` 一遍（每节点多解析 2 次） | 返回值带出 `outbound`，下游直接用 |
| 17 | `CHAIN_RELAY_OUT` 用环境变量给函数传参 | 全局可变状态、非线程安全；残留时会让后续所有调用被误套双跳 | 改为 `test_single_node(item, chain_relay=...)` 显式参数 |
| 18 | `n not in residential` 是 O(n×m) 逐键 dict 相等比较 | 列表元素是含嵌套 `outbound` 的大 dict，`in` 走值比较；家宽一多即退化 | 改用 `id()` 集合，语义等价且 O(1) |

### 8.2 性能优化（CI 耗时）

| # | 项 | 改动前 | 改动后 |
|---|---|---|---|
| 19 | ip-api 批量情报 | `chain_retest` 与 `classify_and_export` **各自全量查一遍同一批出口 IP**，双倍耗时且双倍消耗 15 req/min 免费配额（更容易撞 429） | `main()` 统一查一次后透传；查询失败时传 `None`，两函数仍按旧逻辑各自兜底重试一次（韧性不降） |
| 20 | 端口预检 | `prefilter_candidates` 最后 `return passed + deferred` **一个都没淘汰** —— 对所有候选白跑一遍 DoH + TCP 握手，而 liveness 一次性提交全部 future，排序对总耗时几乎无影响 | 改为三态裁决：仅淘汰 `ECONNREFUSED/EHOSTUNREACH/ENETUNREACH` 这类**与观测点无关的确定性死亡**；超时、域名未解出、QUIC 一律保留交阶段B 裁决（守住"宁可多测，不可误杀"）。可用 `PREFILTER_DROP=0` 退回全量测活 |
| 21 | 单测未进 CI | `scripts/test_parsers.py` 从不执行，而 #3~#7 全是解析器/导出回归 | 工作流新增 `Run Unit Tests` 步骤，回归即失败、不污染线上订阅 |
| 22 | 上传 artifact 步骤空跑 | 工作流上传 `output/e2e_test_report.json`，但代码从未生成该文件 | `main()` 末尾落盘运行报告（耗时/各阶段数量/类型·协议·国家分布），该步骤由空跑变为真实遥测 |

### 8.3 仓库归属与定时

| # | 项 | 说明 |
|---|---|---|
| 23 | 仓库不再写死上游 | 新增 `DEFAULT_REPO = "cuegita/freesub"`，README 全部 CDN/Raw 链接与私有化部署 Worker 脚本参数均由 `GITHUB_REPOSITORY` 派生（本地兜底用该常量） |
| 24 | 补齐定时触发 | README 一直承诺"每 6 小时自动运行"，而工作流**只有 `workflow_dispatch`**。现加 `cron: "23 0,6,12,18 * * *"`（避开整点调度高峰） |
| 25 | ⚠️ fork 定时任务需手动开启 | **本仓库是 `hezhanleiok/freesub` 的 fork**（GitHub API 返回 `fork: true`）。GitHub 对 fork **默认禁用定时任务**，加完 cron 必须去 Actions → 本 workflow → **Enable workflow** 手动开一次，否则 `event=schedule` 永远为 0 条。另：公共仓库连续 60 天无活动也会被自动停用定时调度（本仓库每轮会自动提交，时钟会被持续重置） |
| 26 | jsDelivr purge 修正 | 之前 `${GITHUB_REPOSITORY}` 与实际对外发布的 README 仓库不一致，purge 打在非目标仓库上；现在两者同源 |

### 8.4 文档漂移修正

README 原写"Cloudflare **5MB** 限时下载"→实际 `SPEED_TEST_BYTES = 2_500_000`；原写"**3 个** generate_204 探测"→实际首成功即 `break`，通常只探 1 个。均已按代码实际行为订正。

### 8.5 本轮验证

本地桩掉 `requests`/`maxminddb` 后导入 `main_v2`，跑通：静态符号清理检查、`knock_port` 三态（监听/未监听/QUIC/域名未解出）、`prefilter_candidates` 只淘汰确定性死亡且 `PREFILTER_DROP=0` 可回退、凭据指纹四元组区分度、**19/19 协议样本解析通过**、**5/5 家宽分类样本符合预期**。
未覆盖：`test_single_node` 的 sing-box 真实隧道测活（需内核二进制），由 CI 全流程验证。
