# Encoding Failure Lab · 编码故障实验室

> 这不是又一个“乱码修复器”。
>
> 本项目要让人看见：**一段文本在 bytes、字符、编码标签和转换步骤之间走错一步以后，到底发生了什么；哪些错误还能逆推，哪些信息已经永久丢失。**

## 为什么做这个

平时遇到乱码，我们看到的是结果：

```text
中文 → ä¸­æ–‡
```

但真正发生的事情是：

```text
Unicode text
    ↓ UTF-8 encode
E4 B8 AD E6 96 87
    ↓ 被错误地按 Windows-1252 / 兼容单字节编码解释
ä ¸ ­ æ – ‡
    ↓ 重新保存
新的 bytes
    ↓ ...
```

“乱码”不是一种错误，而是一条**转换历史**。

本仓库希望把这条历史拆开，让每一步都可观察、可复现、可判断损失。

## 明确不做

以下工具已经有成熟实现，本项目不重复：

- 通用字符编码转换器；
- 自动猜测文件编码；
- “一键修复所有乱码”；
- 复制 `iconv` / ICU / WHATWG decoder；
- 重新实现 `ftfy`。

本项目关注的是：

> **为什么会错，以及错到哪一步以后已经救不回来。**

## 核心模型

必须始终区分两个世界：

```text
BYTE DOMAIN                       TEXT DOMAIN

[E4 B8 AD E6 96 87]  --decode-->  "中文"
        ^                            |
        |                            |
      encode <-----------------------+
```

任何实验都要显式记录：

- 当前对象是 bytes 还是 Unicode text；
- 使用了什么 encoding label；
- 使用 encode 还是 decode；
- 错误模式是 fatal、replacement 还是其他策略；
- 这一步是否仍然可逆；
- 是否引入了不可恢复的信息丢失。

## 第一批展品

### Exhibit 1 — 一次经典 mojibake

展示：

```text
中文
→ UTF-8 bytes
→ 错误单字节 decode
→ ä¸­æ–‡
→ 逆操作
→ 中文
```

重点：为什么这种错误有时仍然可以恢复？

### Exhibit 2 — U+FFFD：信息什么时候真正消失

给一段非法/不匹配的 byte sequence：

```text
raw bytes
→ decoder(replacement)
→ �
```

然后让用户尝试反推原 bytes。

核心问题：

> U+FFFD 只告诉我们“这里有东西坏了”，却通常不再保存原始 byte identity。

### Exhibit 3 — Double Mojibake

展示一段文本被连续错误 encode/decode 两次以后，为什么 `ftfy` 一类工具有时仍可恢复，但并非所有链都可恢复。

### Exhibit 4 — GBK / Big5 / Shift-JIS 错位

不要只讲西文 Windows-1252。

重点观察：

- 多字节边界；
- 字节落入另一个编码的合法区间；
- “看起来像正常汉字”的错误为什么反而更危险；
- 哪些路径产生 replacement，哪些路径产生合法但错误的字符。

### Exhibit 5 — Stateful Encoding

后期加入 ISO-2022-JP 等带状态切换的编码，展示：

> decoding 不一定只是“两个 bytes 查一个表”，decoder 本身也可能有状态。

## 损失等级

建议统一使用：

```text
L0 exact
   原始信息完整，操作可精确逆转

L1 recoverable-with-hypothesis
   信息仍在，但必须猜对曾经使用的错误编码链

L2 ambiguous
   多条原始路径可能得到同一个当前结果

L3 substituted
   原始非法 bytes 已被 U+FFFD / ? 等替换

L4 destroyed
   删除、覆盖、不可逆标准化或其他操作导致原始区别消失
```

这里的等级描述**信息状态**，不是给文本质量打分。

## 建议架构

第一版优先纯浏览器运行：

```text
Transformation model
        ↓
Encoding adapters
        ↓
Step-by-step trace
        ↓
Interactive visualization
```

核心逻辑与 UI 分离。

每一步应该产生类似：

```json
{
  "op": "decode",
  "encoding": "utf-8",
  "errorMode": "replacement",
  "inputKind": "bytes",
  "outputKind": "text",
  "loss": "L3",
  "note": "invalid byte subsequence replaced by U+FFFD"
}
```

## 展示原则

网页不是一个大文本框 + Convert 按钮。

理想交互是：

```text
原文
  ↓
code points
  ↓
bytes
  ↓
[故意选择错误 decoder]
  ↓
错误文本
  ↓
[继续保存 / 再错误解码]
  ↓
当前结果
```

每一步都能展开看：

- code point；
- hex bytes；
- encoding；
- decoder state；
- loss level。

## 研究问题

- 为什么某些 mojibake 容易修复，另一些不可修复？
- “合法字符但语义错误”和“非法 byte sequence”有什么根本区别？
- U+FFFD 为什么既是安全机制，也是信息损失边界？
- 一个 encoding detector 能知道“这是原始编码”，还是只能给出概率判断？
- stateful encoding 为什么让错误链更复杂？
- Unicode 解决了字符编号统一，为什么没有消灭 encoding failure？

## 第一阶段

- [ ] `docs/PRIOR_ART.md`；
- [ ] `docs/MODEL.md`：byte/text domain + transformation + loss taxonomy；
- [ ] UTF-8 ↔ Windows-1252/Latin-1 类 mojibake demo；
- [ ] U+FFFD irreversible-loss demo；
- [ ] double-mojibake trace；
- [ ] GitHub Pages 静态展示。

## 项目原则

**永远保留转换链。**

如果一个工具只告诉用户“修好了”，却不能回答：

> 哪一步错了？为什么还能修？哪些信息已经没了？

那它不属于这个项目。
