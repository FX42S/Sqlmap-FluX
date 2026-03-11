# SQLMap-FluX sqlmap二开bypass版

旨在增强其对现代 WAF（Web 应用防火墙）和 IPS（入侵防御系统）的规避能力。
                                          by FhoeniX42S
---

## 第一阶段：静态指纹"大清洗"

### 1. 全局定界符随机化 (`lib/core/settings.py`)

**修改内容：**
- 添加了 `generateDynamicDelimiter()` 函数，生成模拟业务数据格式的随机定界符
- 支持多种格式：JSON 响应片段、Base64 风格、UUID 风格、随机十六进制
- 添加了 `generateRandomizedBoundaries()` 函数生成随机定界符前缀和后缀
- 替换了固定的 `KB_CHARS_BOUNDARY_CHAR` 和 `KB_CHARS_LOW_FREQUENCY_ALPHABET`

**规避原理：**
- 避免使用固定的 `q...q` 十六进制字符串模式
- 模拟正常的业务数据格式，如 `{"id":"abc123","data":"` 或 Base64 填充

### 2. User-Agent 与 Header 库更新 (`data/txt/user-agents.txt`)

**修改内容：**
- 添加了 Chrome 138-140 版本的 User-Agent（Windows 和 macOS）
- 添加了 Firefox 141-143 版本的 User-Agent
- 添加了 Safari 18.x 版本的 User-Agent
- 添加了 macOS ARM 架构的 Chrome User-Agent
- 保留原有 User-Agent 以确保兼容性

**规避原理：**
- 使用 2026 年主流浏览器的最新 User-Agent
- 降低被基于 User-Agent 的指纹库检测的概率

### 3. 垃圾参数干扰 (`lib/request/connect.py`)

**修改内容：**
- 在 `getPage()` 函数中添加了垃圾参数干扰逻辑
- 每次请求随机添加 1-2 个无关紧要的参数，如：
  - `_timestamp`: 当前时间戳
  - `_v`: 随机版本号
  - `_nonce`: 随机十六进制字符串
  - `_ref`: 随机引用字符串
  - `_cache`: 随机缓存标识
  - `_rid`: 随机请求 ID
  - `_t`: 随机时间戳
  - `_track`: 随机追踪标识

**规避原理：**
- 使每个 URL 的哈希值都不同
- 对抗简单的 URL 频率限制和重复请求检测
- 模拟正常用户行为中的追踪参数

---

## 第二阶段：Payload 逻辑"去工程化"

### 1. 重写检测模版 (`data/xml/payloads/boolean_blind.xml`)

**修改内容：**
- 将简单的 `AND 1=1` 替换为复杂数学表达式：
  - `AND (SELECT COUNT(*) FROM (SELECT 1 UNION SELECT 2) t) BETWEEN 1 AND 3`
  - `AND ABS(ROUND([RANDNUM]/[RANDNUM],0))=1`
  - `AND ASCII(SUBSTRING(CAST([RANDNUM] AS CHAR),1,1)) BETWEEN 48 AND 57`

- 将简单的 `OR 1=1` 替换为：
  - `OR CEIL(POWER([RANDNUM]/[RANDNUM],2))=1`
  - `OR LENGTH(CONCAT([RANDNUM],[RANDNUM]))>LENGTH(CAST([RANDNUM] AS CHAR))`

- 子查询中使用复杂表达式：
  - `AND [RANDNUM]=(SELECT (CASE WHEN (MOD(ABS([RANDNUM]-[RANDNUM]),2)=0) THEN ...))`
  - `OR [RANDNUM]=(SELECT (CASE WHEN (FLOOR(POWER(SQRT([RANDNUM]),2))=[RANDNUM]) THEN ...))`

**规避原理：**
- 避免使用简单的 `AND 1=1` 或 `AND [RANDNUM]=[RANDNUM]` 模式
- 使用数学函数（ABS, ROUND, CEIL, POWER, MOD, FLOOR, SQRT）构造等价条件
- 使用字符串函数（LENGTH, CONCAT, ASCII, SUBSTRING）增加复杂度

### 2. 随机化 SQL 关键字 (`lib/core/agent.py`)

**修改内容：**
- 添加了 `obfuscateSqlKeywords()` 函数
- 实现三种随机化技术：

  **a) 大小写变换：**
  - 30% 概率对 SQL 关键字进行随机大小写变换
  - 例如：`SELECT` → `SeLeCt`, `AND` → `aNd`

  **b) 内联注释插入：**
  - 15% 概率在关键字后插入随机内联注释
  - 格式：`SELECT /*!abc*/ FROM /*!xyz*/ WHERE ...`
  - 注释内容随机生成，长度 2-5 个字符

  **c) 关键字混淆池：**
  - 覆盖常用 SQL 关键字：SELECT, FROM, WHERE, AND, OR, UNION 等

**规避原理：**
- 打破 SQLMap 特征的固定模式
- 内联注释不影响 SQL 执行，但干扰基于正则的特征检测
- 大小写变换绕过简单的关键字匹配

---

## 第三阶段：行为模式"拟人化"

### 1. 非线性时间分布 (`lib/core/common.py`)

**修改内容：**
- 添加了 `poissonRandomDelay()` 函数实现泊松分布随机延迟
- 添加了 `humanLikeDelay()` 函数实现拟人化延迟

**泊松分布算法：**
- 使用 Knuth 算法生成泊松分布随机变量
- 参数 `lambda_param=2.0` 控制分布形状

**人类行为模式模拟：**
- **快速点击模式（20% 概率）：**延迟缩短 30-50%
- **正常加载模式（60% 概率）：**延迟在基础值附近波动
- **慢速加载模式（20% 概率）：**延迟增加 50-150%
- 添加高斯噪声增加随机性

**规避原理：**
- 模拟人类操作的自然随机性
- 对抗基于请求频率和序列的自动化检测
- 泊松分布符合人类点击行为模式

### 2. 探测逻辑乱序化 (`lib/controller/checks.py`)

**修改内容：**
- 在 `heuristicCheckSqlInjection()` 函数中添加：

  **a) "垫底"请求：**
  - 在注入探测前发送 1-3 个正常请求
  - 使用原始参数值，模拟正常用户行为

  **b) 随机延迟：**
  - "垫底"请求之间添加随机延迟
  - 延迟时间为配置延迟的 0.5-1.5 倍

  **c) 噪声参数注入（30% 概率）：**
  - 随机生成不存在的参数名（如 `_xyzabc`）
  - 添加随机值作为噪声
  - 日志记录用于调试

**规避原理：**
- "垫底"请求模拟正常浏览行为，降低注入探测的异常性
- 随机延迟打破请求的规律性
- 噪声参数干扰基于参数分析的检测系统

---

## 使用建议

### 推荐的命令行选项组合：

```bash
python sqlmap.py -u "http://target.com/page.php?id=1" \
    --random-agent \
    --delay=1 \
    --level=3 \
    --risk=2 \
    --tamper=randomcase,between
```

### 高级规避配置：

```bash
python sqlmap.py -u "http://target.com/page.php?id=1" \
    --user-agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36..." \
    --delay=2 \
    --time-sec=5 \
    --level=5 \
    --risk=3 \
    --threads=1 \
    --tamper=randomcase,between,charencode
```

---

## 注意事项

1. **性能影响：** 部分修改（如泊松延迟、垫底请求）会增加扫描时间，建议在需要高隐蔽性时使用

2. **兼容性：** 所有修改都保持与原有 SQLMap 功能的兼容性，不影响正常扫描逻辑

3. **可配置性：** 垃圾参数和噪声注入默认启用，可通过修改源码中的概率值调整

4. **测试建议：** 在生产环境使用前，建议在测试环境验证所有修改的有效性

---

## 修改文件列表

| 文件路径 | 修改阶段 | 修改类型 |
|---------|---------|---------|
| `lib/core/settings.py` | 第一阶段 | 添加动态定界符生成函数 |
| `data/txt/user-agents.txt` | 第一阶段 | 更新 User-Agent 列表 |
| `lib/request/connect.py` | 第一阶段/第三阶段 | 添加垃圾参数/泊松延迟 |
| `data/xml/payloads/boolean_blind.xml` | 第二阶段 | 重写检测模板 |
| `lib/core/agent.py` | 第二阶段 | 添加 SQL 关键字随机化 |
| `lib/core/common.py` | 第三阶段 | 添加泊松分布延迟函数 |
| `lib/controller/checks.py` | 第三阶段 | 添加探测逻辑乱序化 |

---

## 技术参考

- **泊松分布：** 用于建模单位时间内随机事件发生次数的概率分布
- **SQL 注释：** MySQL 支持 `/*!...*/` 格式的条件注释，可跨数据库兼容
- **数学函数：** ABS, ROUND, CEIL, FLOOR, POWER, MOD, SQRT 等函数在主流数据库中通用

---
**修改版本：** 基于 sqlmap 1.10.2.18

