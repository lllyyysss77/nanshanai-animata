# AI影视制作系统 - 完整优化方案

## 文档信息

- **项目名称**: Nanshan AI Animata (南山 AI 短剧版)
- **版本**: v1.1.0
- **生成日期**: 2026-03-09
- **优化目标**: 系统性解决解析性能、代码质量、架构设计问题

---

## 第一部分：现状分析

### 1.1 项目定位

**核心功能**: 本地优先的AI影视资产生成平台

- 剧本解析（小说→结构化数据）
- 资产管理（角色、场景、分镜）
- 图像/视频生成
- 任务队列管理

**技术架构**:

- 前端: React 19 + TypeScript + Vite
- 存储: OPFS (本地文件系统)
- AI集成: 支持40+模型

### 1.2 核心问题诊断

#### 问题1: 剧本解析严重低效 🔴 **致命**

**现象**:

- 300字小说消耗25,000+ tokens
- 执行5+次API调用
- 总耗时511秒（8.5分钟）

**根本原因**:

```typescript
// 当前实现: 短文本也执行完整5步流程
const [metadata, globalContext] = await Promise.all([
  this.extractMetadata(content),      // 1次API
  this.extractGlobalContext(content)  // 1次API（多余）
]);
const characters = await this.extractAllCharacters(...);  // 1-2次API
const scenes = await this.extractAllScenes(...);          // 1次API
const shots = await this.generateAllShots(...);           // 1次API
```

**问题**:

1. 并行调用metadata和globalContext（2次API）
2. extractMetadata内部又调用globalContext（第1896行）
3. 角色和场景分别提取（2次API）
4. 每次调用携带庞大的Prompt（schema定义+示例）

#### 问题2: FileSystem API权限失效 🔴 **严重**

**现象**:

- 刷新页面后无法自动连接
- 必须重新选择工作文件夹

**根本原因**:

- 浏览器FileSystem API权限在页面刷新后失效
- autoConnect尝试恢复但权限验证失败

#### 问题3: 代码质量债务 🔴 **中等**

**已清理**:

- ✅ 30个过时测试文件
- ✅ 重复类型定义
- ✅ 未使用函数parseMetadataOnly
- ✅ 25个类型错误

**待解决**:

- 140个类型错误（主要是AI核心模块）
- 8个@deprecated配置仍在使用
- 重复代码块（哈希函数、提取逻辑）

---

## 第二部分：系统性优化方案

### 2.1 架构层优化

#### 优化1.1: 重构解析策略模式

**目标**: 根据文本长度智能选择解析策略

**当前问题**:

```typescript
// 当前: 虽然检测到"短文本"，但仍执行5步流程
Strategy selected: fast
Reason: 短文本快速路径 (293 < 800 字)
Total duration: 511s  // 实际并不fast
```

**优化方案**:

```typescript
// 新增: 超短文本专用解析方法
async parseUltraShortScript(content: string): Promise<ScriptParseState> {
  // 单次API调用，同时提取所有信息
  const combinedPrompt = `
    分析以下剧本，提取元数据、角色、场景和分镜：

    剧本内容：${content}

    请以JSON格式返回：
    {
      "title": "标题",
      "characters": [{"name": "角色名", "description": "描述"}],
      "scenes": [{"name": "场景名", "description": "描述"}],
      "shots": [{"sceneName": "场景名", "description": "分镜描述"}]
    }
  `;

  const result = await this.callLLM(combinedPrompt, { maxTokens: 2000 });
  return this.parseCombinedResult(result);
}
```

**策略选择器**:

```typescript
function selectParseStrategy(content: string): ParseStrategy {
  const length = content.length;

  if (length < 500) {
    return 'ultra-short'; // 1次API调用
  } else if (length < 2000) {
    return 'short'; // 2次API调用
  } else if (length < 10000) {
    return 'medium'; // 3-4次API调用
  } else {
    return 'long'; // 完整5步流程
  }
}
```

**预期效果**:
| 文本长度 | 当前API调用 | 优化后 | 改善 |
|---------|------------|--------|------|
| <500字 | 5次 | 1次 | -80% |
| <2000字 | 5次 | 2次 | -60% |
| <10000字 | 5次 | 3-4次 | -20-40% |

#### 优化1.2: 修复FileSystem权限持久化

**目标**: 刷新页面后自动恢复访问权限

**优化方案**:

```typescript
// 改进autoConnect实现
async autoConnect(): Promise<boolean> {
  // 1. 从IndexedDB恢复handle
  const savedHandle = await this.getSavedDirectoryHandle();
  if (!savedHandle) return false;

  // 2. 请求权限（关键修复）
  const permission = await savedHandle.requestPermission({ mode: 'readwrite' });
  if (permission !== 'granted') {
    await this.clearSavedDirectoryHandle();
    return false;
  }

  this.directoryHandle = savedHandle;
  return true;
}
```

**备选方案**: 使用OPFS作为主存储

```typescript
// OPFS不需要用户授权，自动持久化
const root = await navigator.storage.getDirectory();
const scriptsDir = await root.getDirectoryHandle('scripts', { create: true });
```

### 2.2 性能层优化

#### 优化2.1: Prompt优化

**目标**: 减少Prompt tokens，提高响应速度

**当前问题**:

- Prompt tokens: 1720
- Completion tokens: 5823
- 每次请求携带完整的schema定义和示例

**优化方案**:

```typescript
// 为短文本创建简化版Prompt
const SHORT_METADATA_PROMPT = `
分析剧本，提取基本信息：
- 标题
- 角色列表（姓名即可）
- 场景列表（名称即可）

剧本：{content}

返回JSON：{"title": "...", "characterNames": [], "sceneNames": []}
`;

// 长文本使用完整Prompt
const FULL_METADATA_PROMPT = `
// 完整的schema定义、示例、规则...
`;

// 动态选择Prompt
const prompt =
  content.length < 500
    ? SHORT_METADATA_PROMPT.replace('{content}', content)
    : FULL_METADATA_PROMPT.replace('{content}', content);
```

**预期效果**:

- Prompt tokens: 1720 → 300 (-82%)
- 响应时间: 86s → 20s (-77%)

#### 优化2.2: 并行优化

**目标**: 合理控制并发，避免API限流

**当前问题**:

```typescript
// 当前: 并行提取角色和场景
const [characters, scenes] = await Promise.all([
  this.extractAllCharacters(...),  // 1-2次API
  this.extractAllScenes(...)       // 1次API
]);
// 总计: 2-3次API同时执行，容易触发限流
```

**优化方案**:

```typescript
// 方案1: 短文本合并提取
if (content.length < 1000) {
  const combined = await this.extractCharactersAndScenesCombined(content);
  return { characters: combined.characters, scenes: combined.scenes };
}

// 方案2: 串行执行避免限流
const characters = await this.extractAllCharacters(...);
await delay(500);  // 短暂延迟避免限流
const scenes = await this.extractAllScenes(...);
```

#### 优化2.3: 缓存优化

**目标**: 避免重复解析相同内容

**优化方案**:

```typescript
// 增强缓存机制
class ParseCache {
  private cache = new Map<string, CacheEntry>();

  async getOrParse(
    content: string,
    parseFn: () => Promise<ScriptParseState>
  ): Promise<ScriptParseState> {
    const hash = this.hashContent(content);
    const cached = this.cache.get(hash);

    if (cached && !this.isExpired(cached)) {
      console.log('[ParseCache] Cache hit');
      return cached.data;
    }

    const result = await parseFn();
    this.cache.set(hash, { data: result, timestamp: Date.now() });
    return result;
  }
}
```

### 2.3 代码质量优化

#### 优化3.1: 统一类型定义

**已完成**:

- ✅ RuleViolation统一从ShortDramaRules导入
- ✅ 删除types.ts中的重复定义

**待完成**:

- 修复140个剩余类型错误
- 统一AI核心模块的类型定义

#### 优化3.2: 移除废弃配置

**现状**:
8个@deprecated配置仍在使用：

- useDurationBudget
- targetPlatform
- paceType
- useDynamicDuration
- useShotQC
- qcAutoAdjust
- qcTolerance
- useProductionPrompt

**迁移方案**:

```typescript
// 阶段1: 标记废弃（已完成）
/**
 * @deprecated 2.0移除：基于字数的时长预算规划
 * 请使用creativeIntent替代
 */
useDurationBudget?: boolean;

// 阶段2: 创建迁移工具
function migrateConfig(oldConfig: OldConfig): NewConfig {
  return {
    ...oldConfig,
    creativeIntent: {
      filmStyle: oldConfig.targetPlatform || 'short-drama',
      // ...其他字段映射
    }
  };
}

// 阶段3: 逐步替换使用点
// 将使用旧配置的地方迁移到creativeIntent
```

#### 优化3.3: 提取公共函数

**重复代码**:

```typescript
// 两处DJB2哈希实现
// ParseCache.hash() 第933-941行
// ScriptParser.hashContent() 第1386-1394行
```

**优化方案**:

```typescript
// 创建utils/hash.ts
export function djb2Hash(str: string): number {
  let hash = 5381;
  for (let i = 0; i < str.length; i++) {
    hash = (hash << 5) + hash + str.charCodeAt(i);
  }
  return hash;
}

// 两处都改为使用公共函数
import { djb2Hash } from '../utils/hash';
```

---

## 第三部分：实施路线图

### 阶段1: 紧急修复（1-2天）

**目标**: 解决最影响用户体验的问题

| 任务                      | 优先级 | 预期效果                   |
| ------------------------- | ------ | -------------------------- |
| 实现parseUltraShortScript | P0     | 短文本解析时间: 511s → 30s |
| 修复FileSystem权限        | P0     | 刷新后自动恢复             |
| 添加Prompt优化            | P1     | Token消耗减少50%           |

### 阶段2: 架构重构（3-5天）

**目标**: 系统性优化解析流程

| 任务           | 优先级 | 预期效果         |
| -------------- | ------ | ---------------- |
| 实现策略选择器 | P0     | 智能选择解析策略 |
| 重构并行逻辑   | P1     | 避免API限流      |
| 增强缓存机制   | P1     | 减少重复解析     |

### 阶段3: 代码质量（持续）

**目标**: 提高可维护性

| 任务             | 优先级 | 预期效果     |
| ---------------- | ------ | ------------ |
| 修复剩余类型错误 | P2     | 类型检查通过 |
| 迁移废弃配置     | P2     | 代码更清晰   |
| 提取公共函数     | P3     | 减少重复代码 |

---

## 第四部分：预期成果

### 性能指标

| 指标          | 当前   | 优化后 | 改善     |
| ------------- | ------ | ------ | -------- |
| 300字解析时间 | 511s   | 30s    | -94%     |
| Token消耗     | 25,000 | 3,000  | -88%     |
| API调用次数   | 5+     | 1-2    | -70%     |
| 刷新后恢复    | 手动   | 自动   | 用户体验 |

### 代码质量指标

| 指标       | 当前 | 优化后 | 改善 |
| ---------- | ---- | ------ | ---- |
| 类型错误   | 140  | 0      | -140 |
| 测试文件   | 30   | 0      | -30  |
| 重复代码块 | 7    | 0      | -7   |
| 代码覆盖率 | N/A  | 待补充 | -    |

---

## 第五部分：风险评估

### 高风险

1. **FileSystem API权限修复** - 可能受浏览器安全策略限制
   - 缓解: 准备OPFS备选方案

2. **Prompt简化** - 可能影响解析质量
   - 缓解: A/B测试验证效果

### 中风险

1. **废弃配置迁移** - 可能影响现有用户配置
   - 缓解: 提供自动迁移工具

### 低风险

1. **代码重构** - 可能影响稳定性
   - 缓解: 充分测试，逐步上线

---

## 附录

### A. 关键代码位置

| 功能           | 文件路径                 | 行号范围  |
| -------------- | ------------------------ | --------- |
| 解析策略选择   | services/scriptParser.ts | 3560-3600 |
| 短文本优化路径 | services/scriptParser.ts | 3177-3270 |
| FileSystem连接 | services/storage.ts      | 400-500   |
| Prompt定义     | services/scriptParser.ts | 337-450   |
| 任务配置       | services/scriptParser.ts | 306-332   |

### B. 测试策略

1. **单元测试**: 为核心解析函数编写测试
2. **集成测试**: 验证完整解析流程
3. **性能测试**: 对比优化前后的耗时和token消耗
4. **兼容性测试**: 验证不同浏览器的FileSystem支持

### C. 监控指标

1. **解析耗时** - 按文本长度分档统计
2. **Token消耗** - 每次解析的token数
3. **API成功率** - 失败率和重试次数
4. **缓存命中率** - 重复内容的缓存效果

---

**文档结束**

_本方案基于对项目的全面分析，提供了系统性的优化路径。建议按阶段实施，每个阶段完成后进行验证和评估。_
