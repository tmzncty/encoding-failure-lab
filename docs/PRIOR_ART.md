# Prior Art / 已有标准、工具与边界

本项目不是从零开始研究字符编码。字符编码标准、检测、转换和 mojibake 修复已经有成熟生态。本文件的作用是提醒后续开发者和 AI：**先复用已有能力，再做“错误机制解释”。**

## 1. WHATWG Encoding Standard

- https://encoding.spec.whatwg.org/

现代 Web 平台中的许多 legacy encoding decoder 语义都由 WHATWG Encoding Standard 统一定义。

它已经定义：

- encoding label；
- encoder / decoder；
- decoder state；
- replacement / fatal 等错误模式；
- legacy encodings 的 Web 兼容行为。

### 对本项目的意义

不要凭印象自己定义“GBK decoder 应该怎样报错”“Shift-JIS 哪些 bytes 合法”。

浏览器展品应尽量以 WHATWG / Unicode 的行为作为基线，并明确不同语言运行时可能存在实现差异。

---

## 2. Unicode Standard：U+FFFD 与转换错误

- Unicode Core Specification: https://www.unicode.org/versions/latest/
- Display Problems: https://www.unicode.org/help/display_problems.html
- Unicode Security Considerations: https://www.unicode.org/reports/tr36/

Unicode 文档明确讨论了：

- 错误编码转换导致 mojibake；
- 无法转换的输入可用 U+FFFD REPLACEMENT CHARACTER 表示；
- 非法输入不应被静默删除；
- replacement 是处理错误数据的标准策略之一。

### 本项目最重要的观察

一旦原始非法 bytes 被替换成 U+FFFD，当前 Unicode text 通常已经不再携带“原来到底是哪几个 bytes”的完整信息。

因此 U+FFFD 是非常适合作为本项目的**不可逆边界展品**。

---

## 3. ftfy

- https://github.com/rspeer/python-ftfy
- https://ftfy.readthedocs.io/

`ftfy` 已经非常成熟地处理多类 Unicode glitch 和 mojibake，并特别强调：

- 可以恢复多层 mojibake；
- 使用启发式避免误改正常文本；
- 目标是修复现实世界里的错误文本。

### 不重复

不要重新实现一个 `fix_text()`。

### 本项目增量

`ftfy` 回答：

> “能不能把这段文本修回来？”

Encoding Failure Lab 更关注：

> “可能经历了什么转换链？”
> “为什么这条链可逆？”
> “在哪一步之后已经不可逆？”

如果需要自动修复作为对照，可以调用/引用 ftfy，而不是复制其启发式。

---

## 4. charset-normalizer / chardet 类工具

- charset-normalizer: https://github.com/jawah/charset_normalizer
- chardet: https://github.com/chardet/chardet

这类项目已经解决：

> 给一段未知 bytes，猜测它最可能是什么字符编码。

### 不重复

不要把本项目变成“编码检测 benchmark”。

### 值得展示

编码检测本质上通常是推断，而不是恢复文件中丢失的 metadata。

一个很适合的实验是：

```text
同一组 bytes
→ 在多个编码下都合法
→ detector 给出概率/候选
→ 用户看到“合法”并不等于“原始意图正确”
```

---

## 5. iconv / ICU / iconv-lite

成熟的编码转换层已经广泛存在：

- GNU libiconv: https://www.gnu.org/software/libiconv/
- ICU Conversion: https://unicode-org.github.io/icu/userguide/conversion/
- iconv-lite: https://github.com/ashtuchkin/iconv-lite

### 项目决策

需要 legacy encoding 的 encode/decode 时，优先使用成熟库或平台 API。

本项目自己的核心应是：

```text
transformation graph
loss tracking
trace visualization
counterfactual replay
```

而不是 character mapping tables。

---

## 6. Mojibake repair websites

互联网上已经存在大量：

- UTF-8/GBK/Big5 转换器；
- mojibake repair；
- 自动遍历错误编码链的修复工具。

这些工具证明“修乱码”本身并不稀缺。

### 本项目不得退化成

```text
[text box]
[Fix]
```

如果 UI 最终只剩一个修复按钮，停止开发并重新审视目标。

---

## 7. 本项目真正值得写的共用层

### Transformation graph

明确表达：

```text
bytes --decode(encoding,error-mode)--> text
text  --encode(encoding,error-mode)--> bytes
```

并允许错误链：

```text
text
→ UTF-8 bytes
→ wrong decode
→ wrong text
→ encode again
→ bytes
→ wrong decode again
```

### Loss provenance

每一步记录：

- input hash / representation；
- operation；
- encoding；
- error mode；
- output；
- reversible status；
- loss reason。

### Branching replay

同一组 bytes 同时尝试：

```text
UTF-8
GBK
Big5
Shift-JIS
Windows-1252
```

用户看到不同分支，而不是只得到“最佳答案”。

---

## 8. 必须区分的概念

### Encoding failure ≠ font failure

字符正确但字体没有 glyph，不属于编码错误。

### Encoding failure ≠ normalization

NFC/NFD/NFKC/NFKD 是 Unicode normalization 问题。后期可以作为“另一个可能改变 code-point identity 的转换”展出，但必须明确不是字符编码本身。

### Encoding failure ≠ escaping

HTML entity、URL percent-encoding、JSON escape、Base64 都不是字符编码，但现实错误链中常与字符编码混在一起。后期可作为邻接层加入，必须标注层次。

---

## 9. 开工前强制查重清单

- [ ] 目标只是 encode/decode 吗？若是，优先平台 API / ICU / iconv。
- [ ] 目标只是猜编码吗？若是，优先 charset-normalizer / chardet。
- [ ] 目标只是修 mojibake 吗？若是，先测试 ftfy。
- [ ] 是否保留原始 bytes？
- [ ] 是否明确 byte domain / text domain？
- [ ] 是否记录 error mode？
- [ ] 是否能指出信息损失发生在哪一步？
- [ ] 是否把字体问题误判成编码问题？
- [ ] 是否把“看起来正常”误认为“原始编码一定正确”？

本仓库存在的理由不是提供第 N 个 converter，而是让转换失败本身成为可观察对象。
