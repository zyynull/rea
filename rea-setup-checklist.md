# REA 配置清单(面向 Demon's Souls 重制版静态逆向)

> 用途:在**你自己的机器**上搭好 REA 二进制分析链,用静态逆向做 Mod / 互通研究。
> 目标:Demon's Souls Remake(PS5, 2020)。REA 本身不提供 PC 运行时,只提供静态分析。

---

## 0. 前置判断(先定宿主,决定走哪条 provider)

| 你的宿主      | 深分析 provider  | 成熟度                                           |
| ------------- | ---------------- | ------------------------------------------------ |
| **Linux x64** | **Ghidra**(推荐) | REA 唯一正式支持的深分析平台                     |
| macOS         | Hopper           | 唯一支持 GUI 写入 / `.hop`                       |
| Windows x64   | Ghidra           | **实验性 P0**,仅限原生 x86-64 PE,无 GUI / 无写入 |

REA **不负责**的前置(属于你自己):

1. 合法自持副本;
2. 已越狱主机自行 dump / 解密出 **x86-64 ELF**;
3. 处理厂商私有加密容器。

这两步卡住,后面全无意义。且涉及技术保护措施,请按当地法规自行把关,**仅限个人研究 / 个人使用,不分发本体或解密密钥**。

---

## 1. 运行时

```bash
node --version    # 必须满足 ^22.19.0 || >=24.11.0
npm  --version
```

> 最常见坑:仓库 `.npmrc` 设了 `engine-strict=true`,Node 版本不符会直接 `EBADENGINE` 卡死 `npm ci`。
>
> - 反例:Node **24.1.0** 不满足(既不在 22.x,又 `< 24.11.0`)
> - 安全选择:`22.19.0`、`22.20.x`,或 `>=24.11.0`

```bash
# 举例(nvm)
nvm install 22.19.0 && nvm use 22.19.0
```

---

## 2. 仓库安装与构建

```bash
git clone <rea-repo> rea && cd rea
npm ci                 # .npmrc engine-strict 生效,Node 版本必须先对齐
npm run build          # 产出 dist/;否则 scripts/rea.mjs 报 "compiled runtime is missing"
node scripts/rea.mjs doctor --json   # 首次自检
```

---

## 3A. Ghidra 路线(Linux x64,推荐)

| 要求   | 值                    | 说明                 |
| ------ | --------------------- | -------------------- |
| Ghidra | **12.1.2**(精确)      | 版本不符会被拒绝     |
| JDK    | **21**(64 位完整 JDK) | 硬约束,`25` 不被接受 |

```bash
# 假设已解压 Ghidra 12.1.2 与 JDK 21
export GHIDRA_INSTALL_DIR=/abs/path/ghidra_12.1.2_PUBLIC
export JAVA_HOME=/abs/path/jdk-21        # 缺失时 doctor 退回 PATH 里的 java/javac
```

## 3B. Hopper 路线(macOS)

```bash
export HOPPER_LAUNCHER_PATH="/Applications/Hopper Disassembler.app/Contents/MacOS/hopper"
export HOPPER_TARGET_PATH=/abs/path/target        # 可选;target-free 会话改用 open_binary
```

> REA 的 installer **不会**安装/升级 Homebrew、Node、Java、Ghidra 等无关软件;Hopper 授权需你自备。
> `rea setup` 会**先打印计划并要求批准**才写入文件或安装 Hopper。

---

## 4. 环境变量清单(本任务最小集)

| 变量                                 | 必需性          | 用途                            |
| ------------------------------------ | --------------- | ------------------------------- |
| `GHIDRA_INSTALL_DIR`                 | ✅(Ghidra 路线) | 指向解压后的 Ghidra 12.1.2      |
| `JAVA_HOME`                          | ✅(Ghidra 路线) | 指向 **JDK 21**                 |
| `REA_ANALYSIS_PROVIDER`              | 可选            | 固定为 `ghidra`(默认 `auto`)    |
| `HOPPER_LAUNCHER_PATH`               | Hopper 路线必需 | Hopper 可执行文件路径           |
| `REA_INVESTIGATION_INPUT_ROOTS_JSON` | 可选            | 授权用于静态分析的绝对根目录    |
| `REA_ILSPY_CMD_PATH`                 | 可选            | 仅 .NET managed 分析,本任务无关 |

---

## 5. 自检命令(按顺序)

```bash
node scripts/rea.mjs doctor --json      # 期望:ghidra 各检查项全 OK
node scripts/rea.mjs providers --json   # 期望:analysis_provider_candidates 里 ghidra 为 available
```

> 注意:`providers` 在无目标(target-free)时 `analysis_provider_binding` **恒为 null**,这是设计如此(绑定是 per-target 会话行为);判断 provider 是否可用要看 `analysis_provider_candidates`,不要看 binding。

`doctor` 里这两条**可以先不管**(与二进制分析无关):

- `hopper` 失败 —— Linux 上正常(仅 macOS)
- `skill:identity` 失败 —— 跑一次 `rea setup` 即可消除

---

## 6. 用无害二进制先验证链路(强烈建议)

碰游戏之前,先拿一个本地小 ELF 跑通整条静态分析链:

```bash
node scripts/rea.mjs analyze /bin/true
node scripts/rea.mjs search  /bin/true "usage"
```

跑通 = Ghidra provider 端到端可用。**再**换成解密后的游戏 ELF。

---

## 7. MCP 接入(可选)

```bash
npm run config:print -- /path/to/decrypted-elf
# 或构建 stdio server 配置后接入你的 MCP 客户端
```

---

## 8. 资源与边界(影响分析策略)

- REA 的 Ghidra 分析是**有界**的:`300s` 自动分析、`2 CPU`、`2G` JVM 堆、单函数反编译 `30s`、串行队列 `32`。
- 3A 游戏主程序远超该规模 → **原生 Ghidra / IDA 做全量装载初筛**,**REA 做定向函数档案与调用链**。
- 无 PC 运行时:Mod 只能在越狱主机侧验证,无法产出可分发 PC 版。

---

## 9. 快速排查表

| 症状                             | 原因                                 | 处理                          |
| -------------------------------- | ------------------------------------ | ----------------------------- |
| `npm ci` → `EBADENGINE`          | Node 版本不符 + `engine-strict=true` | 切到 `22.19.0` 或 `>=24.11.0` |
| `compiled runtime is missing`    | 没构建                               | `npm run build`               |
| `doctor` ghidra `not_configured` | `GHIDRA_INSTALL_DIR` 未设            | 设置后复验                    |
| `doctor` ghidra 仍失败           | JDK 不是 **21**(如 25)               | 换 JDK 21,设 `JAVA_HOME`      |
| 版本不符被拒                     | Ghidra 不是 **12.1.2**               | 换精确版本                    |
| Windows 上深分析异常             | P0 实验性,仅 PE                      | 改用 Linux x64 + Ghidra       |
