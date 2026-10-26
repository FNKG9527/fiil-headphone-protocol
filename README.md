# FIIL 耳机蓝牙协议逆向笔记

> 对 FIIL（斐耳）官方控制 App **FIIL+ 3.5.2** 的静态逆向成果整理，以及一台真实耳机上的逐条实机验证记录。
> 目标只有一个：**让第三方能在自己的设备上正确地说 FIIL 的协议**。

![License: CC BY-SA 4.0](https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg)
![Docs](https://img.shields.io/badge/docs-3%20%E4%BB%BD-blue.svg)
![Transport](https://img.shields.io/badge/transport-BLE%20GATT%20%7C%20SPP-orange.svg)
![Status](https://img.shields.io/badge/%E7%8A%B6%E6%80%81-%E5%AE%9E%E6%9C%BA%E9%AA%8C%E8%AF%81-green.svg)

---

## 这是什么

FIIL 耳机没有公开的开发者协议文档。官方 App 能做的每一件事——降噪档位、EQ、影院模式、低延迟、多点连接、语音提示音量——都跑在一条**私有二进制协议**上，而且这套协议在自家产品线里还分裂成了**三套并存的命令空间**。

本仓库把这条链路拆开写清楚：

| 你会拿到 | 说明 |
|---|---|
| **帧格式** | 三套命令空间的完整字节布局、长度字段的三种语义、组帧与流式重组 |
| **命令表** | 190 条 `FFCOMMAND`（GAIA）、Mate 的 `classifyId + cmdId` 二元组、豆包 `0x55AA` 帧 |
| **语义** | 每个可写项的取值范围、档位文案原文、线值与文案的对应（有几处是**反序**的） |
| **时序** | 每类命令的真实生效延迟、ACK 与状态回读的先后、断链与重连行为 |
| **踩坑** | 文档里被广泛抄错的地方，以及本项目实测证伪的结论 |

所有结论都标注了来源：要么是反编译产物的 `文件:行号`，要么是 `docs/evidence/` 下的原始抓包日志。**没有来源的结论不写。**

---

## 文档导航

| 文档 | 面向 | 内容 |
|---|---|---|
| [**PROTOCOL_FIIL.md**](docs/PROTOCOL_FIIL.md) | 想**读懂**协议的人 | 三套命令空间的完整规格：GAIA / Mate / 豆包，机型路由，GET 与 SET 的语义差异，未确认项清单 |
| [**PROTOCOL_PC.md**](docs/PROTOCOL_PC.md) | 想**动手发命令**的人 | 一台真实耳机上的 BLE GATT 实测手册：UUID、握手、18 条 GET 全表、五态降噪、电量解析、生效延迟 |
| [**REVERSE_ENGINEERING_REPORT.md**](docs/REVERSE_ENGINEERING_REPORT.md) | 想**知道 App 怎么做的**人 | FIIL+ 3.5.2 拆包分析：四条传输链路、190 条命令分类、机型对照表、反逆向设计点 |

想直接上手：先读 `PROTOCOL_PC.md` 的 **§0 结论速览** 和 **§4 连接初始化流程**，十分钟能发出第一条命令。
想理解全貌：读 `PROTOCOL_FIIL.md` 的 **§0 结论速览** 和 **§5 机型与路由**。

---

## 协议速览

```
帧布局（两套命令空间共用同一搬运通道）

  偏移  0     1     2      3       4         5        6          7..
       seq   cid   type   00   lenField   cmdId   payloadLen   payload

  seq       1 字节滚动计数器（低 4 位）
  cid       命令族编号。GET 固定 39 (0x27)；SET 用命令族编号
  type      0x01 REQUEST / 0x02 RESPONSE / 0x03 NOTIFY
  lenField  0 → 空帧（耳机不认识该命令）
            1 → 6 字节短帧，第 6 字节是成功标志（SET 的应答）
            ≥2 → 总长 = 5 + lenField（所有 GET 应答与请求）
```

几个最容易写错的点，先说在前面：

- **`lenField == 1` 不是「长度 1 的空帧」**，第 6 字节是成功标志。按「<2 就当 5 字节空帧」切帧会把 SET 的成功字节整个吞掉。
- **`seq` 不能用来匹配请求与响应**。耳机用自己的滚动计数器回包。
- **`type` 必须同时接受 `0x01/0x02/0x03`**。只认 `0x01` 会丢掉全部回包——这是「界面永远显示未连接」的经典原因。
- **连接后第一帧必须是 `setUnlock`**，否则耳机对所有命令静默。
- **SET 生效是异步的**：降噪约 1.75–2.0 s，影院 0.5 s，低延迟 0.45 s。写完立刻读回，读到的一定是旧值。
- **多点连接切换必然断链**。这是固件行为，不是 bug。重连后必须重做 `setUnlock`。

---

## 覆盖范围

| 维度 | 已覆盖 |
|---|---|
| 命令空间 | GAIA（16 位命令字）、Mate（`classifyId + cmdId`）、豆包 `0x55AA`（结构未展开） |
| 传输层 | 经典蓝牙 SPP/RFCOMM、BLE GATT（`fdb3` / `ff17` 写 / `ff18` 收） |
| 实机验证 | 一台 Mate 机型（earType 46 / Key Max，固件 3.0.1.1）：BLE GATT 全链路、18 条 GET、5 态降噪、3 档提示音音量、多点切换断链时序 |
| 静态推导 | 7 个机型命令类的覆盖差异、机型 → earType 映射、四套并行传输栈 |
| **未覆盖** | 其他机型的真机验证、EQ 自定义曲线写入、豆包语音帧的 payload 结构、OTA 流程 |

未确认的一律在文档里标 `未确认`，并汇总在各文档的「未确认项清单」里。**请不要把未确认项当结论用。**

---

## 仓库结构

```
.
├── README.md                        # 本文件
├── LICENSE                          # CC BY-SA 4.0
├── NOTICE                           # 商标与素材归属声明
├── DISCLAIMER.md                    # 使用边界与合规声明
├── CONTRIBUTING.md                  # 如何提交勘误
├── CODE_OF_CONDUCT.md
├── SECURITY.md
├── CHANGELOG.md
└── docs/
    ├── README.md                    # 文档索引（含阅读顺序）
    ├── PROTOCOL_FIIL.md             # 协议规格总纲
    ├── PROTOCOL_PC.md               # BLE GATT 实测手册
    ├── REVERSE_ENGINEERING_REPORT.md# 拆包分析报告
    ├── evidence/                    # 原始抓包日志（已脱敏）
    └── reference/
        └── _cmd_table.txt           # 190 条 FFCOMMAND 原始导出
```

---

## 数据脱敏说明

本仓库的文档源自一台真机的调试过程，公开前做过一次**不可逆的替换**：

| 类别 | 处理 |
|---|---|
| 蓝牙地址 / MAC | 全部替换为文档保留地址段（`B0:F1:A3:00:00:0x`、`D8:80:83:00:00:0x`、`16:F3:78:00:00:0x`、`62:F8:9E:00:00:0x`），保留 OUI 与末字节关系以维持原有推理逻辑 |
| 扫描记录里的第三方设备 | 同上处理，只保留地址结构，不保留任何可关联到真实设备的取值 |
| 本机绝对路径 | 改写为仓库内相对路径（`jadx_out/…`、`apktool_out/…`、`docs/evidence/…`） |
| 构建环境路径 | 替换为 `<JAVA_HOME>` 等占位符 |
| 包名等个人标识 | 删去私有模块名与包名，改为「第三方 Android 控制端」等中性描述 |
| 个人视角措辞 | 改写为中性表述（「本次实测」「该机型」） |

**不包含**：APK 原文件、反编译产物、App 资源、任何密钥或证书、任何账号或个人信息。
APK 的哈希值保留在 `REVERSE_ENGINEERING_REPORT.md` 的样本信息一节，仅用于标识被分析的版本。

如果你在仓库里仍然发现了可关联到具体个人的信息，请按 [SECURITY.md](SECURITY.md) 的方式联系，我们会立即处理。

---

## 使用边界

这些文档的目的是**互操作性研究与本地控制**。允许：为自己的耳机写控制端、做协议学习、写测试工具。

不允许：绕开固件校验或做未授权 OTA、把 FIIL 的私有实现或资源当作自己的成果再分发、在商业产品里声称与 FIIL 官方兼容。

完整条款见 [DISCLAIMER.md](DISCLAIMER.md)。

---

## 许可证

文档以 **[CC BY-SA 4.0](LICENSE)** 发布——可自由复制、修改、商用，但必须署名并以相同许可证分发。

FIIL、FIIL+ 及相关名称是斐耳科技的商标。本仓库与斐耳科技无任何关联，未获其授权或背书，仅用于互操作性研究。详见 [NOTICE](NOTICE)。

---

## 参与

发现错误、有别的机型的实测数据、或者某个「未确认」项你已经验证过了——欢迎提 issue 或 PR，格式见 [CONTRIBUTING.md](CONTRIBUTING.md)。**带抓包日志的勘误优先合并。**

