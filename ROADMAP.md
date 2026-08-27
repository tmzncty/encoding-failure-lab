# Roadmap

Encoding Failure Lab 按“状态模型 → 单步错误 → 多步错误 → 多编码 → 邻接转换”推进。

## M0 — Transformation Model

目标：先让每一次 encode / decode 都成为可记录、可测试的状态转换。

- [ ] `docs/MODEL.md`；
- [ ] `src/core/types.ts`；
- [ ] `src/core/trace.ts`；
- [ ] `src/core/loss.ts`；
- [ ] byte/text domain 严格区分；
- [ ] loss level L0-L4；
- [ ] deterministic JSON trace。

### 最小数据结构

至少能表达：

```text
Bytes
Decode(encoding,errorMode)
Text
Encode(encoding,errorMode)
Bytes
```

并保存每一步的 input/output 表示。

### M0 验收

给定固定输入，trace JSON 必须稳定；UI 不参与正确性判断。

---

## M1 — Classic Mojibake

第一件可玩的展品：

```text
中文
→ UTF-8 bytes
→ wrong single-byte decode
→ mojibake
→ reverse hypothesis
→ 中文
```

要求：

- 显示 code points；
- 显示 hex bytes；
- 显示每步 encoding；
- 标记 `recoverable-with-hypothesis`；
- 支持 step forward / backward；
- 不做“一键自动修复”为主交互。

---

## M2 — Point of No Return

围绕 U+FFFD 做不可逆实验。

至少包含：

1. 非法 UTF-8 sequence；
2. replacement decode；
3. 多个不同原始 byte sequences 映射到带 U+FFFD 的结果；
4. 证明当前 text 已不足以唯一确定原始 bytes。

### 验收

用户能清楚回答：

> 为什么“�”不是乱码原字节的某种神秘字符，而是错误处理后留下的占位符？

---

## M3 — Double Mojibake

支持多步链：

```text
text
→ encode A
→ decode B
→ encode C
→ decode D
```

功能：

- trace timeline；
- branch from any step；
- 手工修改 encoding hypothesis；
- compare two recovery paths；
- 标记哪条链精确恢复，哪条只是产生“看起来合理”的文本。

可用 ftfy 作为对照，不复制其实现。

---

## M4 — CJK Failure Gallery

加入：

- GBK / GB18030；
- Big5；
- Shift-JIS；
- EUC-JP（视平台支持）；
- Windows-1252 作为常见西文错误基线。

重点不是收集所有编码，而是挑：

- 合法但错义；
- 非法 byte sequence；
- 多字节边界错位；
- 相同 bytes 在多编码下都合法；
- detector 无法凭逻辑绝对确定原意。

---

## M5 — Stateful Encoding

进入 ISO-2022-JP 等状态型 decoder。

展示：

```text
byte
↓
decoder state changes
↓
same byte value may be interpreted under different active states
```

要求显式显示 decoder state，防止把所有 encoding 都讲成静态查表。

---

## M6 — Adjacent Failure Layers

后期可选：

- BOM / endian；
- Unicode normalization；
- HTML entities；
- URL percent-encoding；
- JSON escape；
- filesystem filename conversion。

这些必须作为**邻接转换层**，不能混称“字符编码”。

特别适合做：

> 字符本身没错，但连续跨层以后为什么仍然会坏？

---

## M7 — Public Exhibit

部署 GitHub Project Pages。

页面建议：

```text
/                     首页
/classic-mojibake      经典乱码链
/replacement-loss      U+FFFD 不可逆实验
/double-mojibake       双重乱码
/cjk                    CJK 编码故障
/stateful               状态型编码
/about                   方法与来源
```

静态站优先，不需要后端。

---

## AI 可直接领取的第一批任务

### Task A — 写 MODEL

读取 README 和 `docs/PRIOR_ART.md`，设计 `docs/MODEL.md`。

约束：

- 只定义 byte/text domain、operation、trace、loss；
- 不做 UI；
- 不写自动修复器；
- 给出至少 10 个边界测试用例。

### Task B — Mojibake Golden Vectors

建立 `fixtures/`，收集一批可人工验证的转换链：

- UTF-8 ↔ Windows-1252 类；
- 中文 UTF-8 错解；
- double mojibake；
- invalid UTF-8 + replacement。

每条必须同时保存原 bytes 和期望 trace。

### Task C — U+FFFD loss proof

写一个最小实验，找到至少两组不同 invalid byte inputs，在 replacement decode 后无法从 text 唯一恢复原 bytes。

输出测试 + 教学说明。

### Task D — Browser capability survey

调查浏览器 `TextDecoder` 当前支持的 WHATWG encoding label，以及 `TextEncoder` 的限制；评估 legacy encode 是否需要 `iconv-lite` 或其他成熟依赖。

输出 `research/browser-encoding-capabilities.md`，未经调查不要自己写 mapping table。

### Task E — First Pages prototype

只有在 A/B/C 完成以后开始。

做一个完全静态的 stepper：

```text
Text → Bytes → Wrong Decode → Mojibake
```

每步显示 hex / code points / loss state。

---

## Stop Conditions

以下任一发生就停止当前实现方向：

- 产品退化为一个 converter；
- 产品退化为一键 mojibake fixer；
- 自己维护大型字符映射表而成熟库已经提供；
- 没有保存原始 bytes 就开始声称“恢复原文”；
- 把 U+FFFD 当作可逆编码；
- 把 font rendering、escape、normalization 全混叫 encoding；
- UI 动画成为唯一状态真相来源。

最终目标不是：

> “我能把乱码修好。”

而是：

> **“我能告诉你文本到底在哪一步坏了，以及为什么这一次还能救、下一次却不能。”**
