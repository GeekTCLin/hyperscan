三种精度模式用于控制 Hyperscan 在**流模式**下如何跟踪**匹配起始位置**，是**精度与资源消耗**之间的权衡。让我详细解释：

## 🎯 核心概念：SOM（Start of Match）

- **SOM** = 匹配开始的位置偏移量
- 在流式扫描中，一个匹配可能**跨越多个数据块**
- 为了准确报告"匹配从哪个位置开始"，需要保存历史状态

## 📊 三种 SOM 精度模式对比

| 模式 | 精度范围 | 内存占用 | 适用场景 |
|------|----------|----------|----------|
| **`HS_MODE_SOM_HORIZON_LARGE`** | 全精度，无限制 | **最高** | 需要绝对准确SOM的场景 |
| **`HS_MODE_SOM_HORIZON_MEDIUM`** | 2³²字节（约4GB）范围内 | 中等 | 大多数流式处理场景 |
| **`HS_MODE_SOM_HORIZON_SMALL`** | 2¹⁶字节（64KB）范围内 | **最低** | 内存敏感或匹配间隔短的场景 |

## 🔍 详细解释

### 1. **`HS_MODE_SOM_HORIZON_LARGE`** - 全精度模式
```c
// 无论匹配在多久之前开始，都能准确报告起始位置
// 示例：匹配从数据流开头开始，经过TB级数据后结束，仍能准确报告SOM
hs_compile("pattern.*match", HS_FLAG_SOM_LEFTMOST, 
           HS_MODE_STREAM | HS_MODE_SOM_HORIZON_LARGE, ...);
```
- **特点**：使用最多的流状态内存
- **保证**：**永远**返回准确的匹配起始偏移量
- **代价**：内存开销最大

### 2. **`HS_MODE_SOM_HORIZON_MEDIUM`** - 中等精度模式
```c
// SOM精度限制在4GB范围内
// 示例：如果匹配结束在位置1,000,000,000，那么SOM必须在[996,000,000, 1,000,000,000]范围内才准确
hs_compile("pattern.*match", HS_FLAG_SOM_LEFTMOST,
           HS_MODE_STREAM | HS_MODE_SOM_HORIZON_MEDIUM, ...);
```
- **特点**：比大模式使用更少的流状态
- **限制**：SOM精度限制在匹配结束位置**前4GB**范围内
- **超出范围**：如果匹配开始位置超出此范围，报告的SOM可能不准确

### 3. **`HS_MODE_SOM_HORIZON_SMALL`** - 有限精度模式
```c
// SOM精度限制在64KB范围内
// 示例：适合短间隔匹配，如日志分析中的近场模式
hs_compile("pattern.{0,100}match", HS_FLAG_SOM_LEFTMOST,
           HS_MODE_STREAM | HS_MODE_SOM_HORIZON_SMALL, ...);
```
- **特点**：使用最少的流状态内存
- **限制**：SOM精度限制在匹配结束位置**前64KB**范围内
- **适用**：匹配开始和结束位置很近的场景

## 💡 使用要点

### 必要条件
```c
// 必须同时设置以下两个条件才能使用SOM功能：
// 1. 使用HS_FLAG_SOM_LEFTMOST标志
// 2. 选择任意一个SOM_HORIZON模式
hs_compile(patterns, HS_FLAG_SOM_LEFTMOST, 
           HS_MODE_STREAM | HS_MODE_SOM_HORIZON_MEDIUM, ...);
```

### 选择策略
- **需要绝对准确SOM** → 选择 `LARGE` 模式
- **处理大数据流，内存敏感** → 选择 `MEDIUM` 模式（覆盖99%场景）
- **内存极度受限，匹配间隔短** → 选择 `SMALL` 模式
- **不需要SOM功能** → 不设置任何SOM_HORIZON模式

## 🚀 实际影响示例

假设处理一个10GB的数据流：
- 使用 `LARGE`：无论匹配从哪个位置开始，都能准确报告SOM
- 使用 `MEDIUM`：如果匹配从数据流**开头**开始（距离>4GB），报告的SOM可能不准确
- 使用 `SMALL`：只有匹配在最后64KB内开始的，SOM才准确

这些模式让你在**内存开销**和**匹配精度**之间做出合适的权衡。