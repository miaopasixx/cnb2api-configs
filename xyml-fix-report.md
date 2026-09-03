# XYML/DSML 标签泄露问题排查与修复报告

## 问题现象

客户端（本地 IDE）通过 ToolForge 中间件调用 CNB API 时，模型响应中出现了未解析的 XYML/DSML 标签，直接泄露到客户端输出中。

### 泄露标签示例

**标准格式：**
```
<|XYML|tool_calls>
  <|XYML|invoke name="Bash">
    <|XYML|parameter name="command">...</|XYML|parameter>
  </|XYML|invoke>
</|XYML|tool_calls>
```

**全角字符变体：**
```
<｜ ｜ DSML｜ ｜ >
</｜ ｜ DSML｜ ｜ >
```

**混合格式（DSML 和 XYML 嵌套）：**
```
<｜ ｜ DSML｜ ｜ |XYML|tool_calls>
<｜ ｜ DSML｜ ｜ |XYML|invoke name="Read">
</｜ ｜ DSML｜ ｜ |XYML|tool_calls>
```

---

## 架构背景

```
客户端 -> ToolForge(:18080) -> cnb2api-gateway(:7863) -> CNB API
```

- **ToolForge**：前置中间件，负责上下文压缩、协议转换、函数调用管理
- **cnb2api-gateway**：Go 语言反向代理网关，连接池化管理
- **CNB API**：上游 AI 模型服务

---

## 排查过程

### 第1步：确认服务状态

**发现：** 服务器运行两个 Docker 容器：
- `cnb2api-toolforge`（端口 18080）
- `cnb2api-gateway`（端口 7863）

**结论：** 服务正常运行，问题出在代码逻辑层面。

---

### 第2步：检查 fc_mode 配置

**文件：** `/tmp/cnb2api/docker/config.yaml`

**发现：** 配置文件中 `fc_mode: auto`，但代码中判断的是 `fc_mode == "prompt"`。

**根因1：配置值与代码不匹配**
- 配置文件注释说有效值是 `auto | prefer_native | force_prompt`
- 但 orchestrator.py 代码中判断的是 `fc_mode == "prompt"`
- `force_prompt` 和 `prompt` 是两个不同的值

**修复：** 将 `fc_mode` 改为 `prompt`
```bash
sed -i 's/fc_mode: auto/fc_mode: prompt/' /tmp/cnb2api/docker/config.yaml
```

---

### 第3步：发现 passthrough 模式绕过所有过滤

**文件：** `/app/app/fc/policy.py` 第24-25行

**发现：** 当请求中没有工具定义时，fc_mode 被强制设为 `passthrough`
```python
if not req.tools:
    return "passthrough"  # 纯透传，不做任何过滤
```

**根因2：passthrough 模式完全绕过 XYML 解析和过滤**
- 非流式请求没有携带工具定义
- ToolForge 自动切换到 passthrough 模式
- 后端响应原样透传给客户端，XYML 标签直接泄露

**修复：** 禁用 passthrough 分支
```python
if not req.tools and False:  # 添加 and False 使条件永远为假
    return "passthrough"
```

---

### 第4步：发现修改了错误的函数

**文件：** `/app/app/engine/orchestrator.py` 和 `/app/app/stream/openai_sse.py`

**发现：** 最初在 `_stream_gemini` 函数中添加了过滤逻辑，但该函数只处理 Gemini 协议路径。用户实际使用的是 OpenAI 兼容接口，走的是 `stream_prompt_fc` 函数。

**根因3：过滤代码添加在了错误的函数中**
- `_stream_gemini`：Gemini 协议路径（未被使用）
- `stream_prompt_fc`：OpenAI 兼容接口路径（实际使用）

**修复：** 在 `openai_sse.py` 的 `stream_prompt_fc` 函数中添加过滤逻辑

---

### 第5步：发现正则表达式完全无法匹配

**文件：** `/app/app/engine/orchestrator.py` 和 `/app/app/stream/openai_sse.py`

**发现：** 正则表达式要求两个连续分隔符，但实际标签只有一个

**错误的正则：**
```python
r'<[|｜] ?[|｜] ?(XYML|DSML)[|｜] ?[|｜] ?[^>]*>'
#        ^^^^^^^^ 要求两个分隔符
```

**实际标签格式：**
```
<|XYML|tool_calls>
#     ^ 只有一个分隔符
```

**根因4：正则表达式语法错误，完全无法匹配任何 XYML 标签**

**修复：** 去掉多余的分隔符要求
```python
r'<[|｜] ?(XYML|DSML)[|｜][^>]*>'
#       ^ 只要求一个分隔符
```

---

### 第6步：发现正则不支持关闭标签

**发现：** 修正后的正则只能匹配开标签（`<|XYML|...>`），无法匹配关闭标签（`</|XYML|...>`）

**根因5：正则缺少对关闭标签斜杠的匹配**

**修复：** 添加 `/?` 来匹配可选的斜杠
```python
r'</?[|｜] ?(XYML|DSML)[|｜][^>]*>'
#   ^^ 匹配可选的斜杠，同时支持开标签和关闭标签
```

---

### 第7步：发现混合格式标签无法匹配

**发现：** 模型输出中出现了 DSML 和 XYML 标签嵌套的混合格式
```
<｜ ｜ DSML｜ ｜ |XYML|tool_calls>
```

**根因6：正则要求协议名紧跟在分隔符后面，但混合格式中协议名出现在标签中间位置**

当前正则：`</?[|｜] ?(XYML|DSML)[|｜][^>]*>`
- 期望格式：`<|XYML|...>`（协议名紧跟分隔符）
- 实际格式：`<｜ ｜ DSML｜ ｜ |XYML|tool_calls>`（协议名在中间）

**修复：** 使用更宽松的正则，匹配任意包含 XYML 或 DSML 的标签
```python
r'</?[|｜] ?[^>]*(XYML|DSML)[^>]*>'
#          ^^^^^ 匹配任意字符直到协议名
#                       ^^^^^ 匹配协议名后的任意字符
```

---

## 最终正则表达式

```python
import re

pattern = r'</?[|｜] ?[^>]*(XYML|DSML)[^>]*>'
result = re.sub(pattern, '', text, flags=re.IGNORECASE)
```

**匹配范围（全部通过测试）：**
- `<|XYML|tool_calls>` ✅
- `<|XYML|invoke name="Bash">` ✅
- `</|XYML|invoke>` ✅
- `</|XYML|tool_calls>` ✅
- `<|DSML|>` ✅
- `</|DSML|>` ✅
- `<｜ ｜ DSML｜ ｜ >` ✅（全角变体）
- `<｜ ｜ DSML｜ ｜ |XYML|tool_calls>` ✅（混合格式）
- `<｜ ｜ DSML｜ ｜ |XYML|invoke name="Read">` ✅（混合格式）
- `</｜ ｜ DSML｜ ｜ |XYML|tool_calls>` ✅（混合格式关闭标签）

---

## 完整修复汇总

| 序号 | 文件 | 行号 | 修改内容 | 根因 |
|------|------|------|----------|------|
| 1 | config.yaml | - | fc_mode: auto → prompt | 配置值与代码不匹配 |
| 2 | fc/policy.py | 24-25 | 禁用 passthrough 分支 | 无工具定义时走透传模式 |
| 3 | engine/orchestrator.py | 263-269 | 添加正则清理逻辑 | 解析失败时标签泄露 |
| 4 | stream/openai_sse.py | 79-81 | 添加过滤函数 | 流式输出前未过滤 |
| 5 | engine/orchestrator.py | 266-269 | 修正正则分隔符数量 | 正则要求两个分隔符 |
| 6 | stream/openai_sse.py | 81 | 修正正则分隔符数量 | 正则要求两个分隔符 |
| 7 | stream/openai_sse.py | 81 | 添加关闭标签支持 | 正则不支持 `</` 前缀 |
| 8 | engine/orchestrator.py | 266-269 | 增强正则匹配范围 | 混合格式标签无法匹配 |
| 9 | stream/openai_sse.py | 81 | 增强正则匹配范围 | 混合格式标签无法匹配 |

---

## 正则表达式演进历程

| 版本 | 正则表达式 | 能匹配 | 不能匹配 |
|------|-----------|--------|----------|
| v1（错误） | `<[|｜] ?[|｜] ?(XYML\|DSML)[|｜] ?[|｜] ?[^>]*>` | 无 | 所有标签 |
| v2 | `<[|｜] ?(XYML\|DSML)[|｜][^>]*>` | 开标签 | 关闭标签、混合格式 |
| v3 | `</?[|｜] ?(XYML\|DSML)[|｜][^>]*>` | 开标签、关闭标签 | 混合格式 |
| v4（最终） | `</?[|｜] ?[^>]*(XYML\|DSML)[^>]*>` | 所有变体 | - |

---

## 测试验证

### 内部测试（容器内 Python）

```python
import re

test_cases = [
    '<｜ ｜ DSML｜ ｜ |XYML|tool_calls>',
    '<｜ ｜ DSML｜ ｜ |XYML|invoke name="Read">',
    '</｜ ｜ DSML｜ ｜ |XYML|tool_calls>',
    '<|XYML|tool_calls>',
    '</|XYML|invoke>',
    '<|DSML|>',
    '正常文本内容',
]

pattern = r'</?[|｜] ?[^>]*(XYML|DSML)[^>]*>'

for text in test_cases:
    result = re.sub(pattern, '', text, flags=re.IGNORECASE)
    changed = '已过滤' if result != text else '保留'
    print(f'{changed}: {text[:50]}')
```

**测试结果：全部通过**
- 混合格式标签 → 已过滤 ✅
- 标准格式标签 → 已过滤 ✅
- 全角变体标签 → 已过滤 ✅
- 正常文本内容 → 保留 ✅

---

## 经验教训

1. **配置值与代码逻辑必须一致**：配置文件中的值必须与代码中判断的值完全匹配
2. **修改代码前先确认代码路径**：不同协议/接口可能走不同的处理函数
3. **正则表达式必须经过实际数据测试**：不能假设正则能匹配，必须用真实数据验证
4. **分层防护更可靠**：在多个层面（policy、orchestrator、stream）添加过滤，提高容错性
5. **正则表达式需要逐步增强**：从简单到复杂，逐步覆盖更多变体格式
6. **注意字符集差异**：半角 `|` 和全角 `｜` 都需要匹配
