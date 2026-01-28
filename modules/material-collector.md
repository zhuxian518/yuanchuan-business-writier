# 素材采集模块

> **职责**：自动联网搜索、提取和验证商业分析所需的核心素材，构建结构化素材库

---

## 🎯 核心功能

素材采集模块负责：
1. 根据配置对象确定搜索策略
2. 使用`web_fetch`工具进行分层搜索
3. 提取关键数据点并结构化存储
4. 对素材进行信源分级和可信度评估
5. 生成素材收集报告供后续使用

---

## 📊 素材分类体系

### 一级素材：基础信息层

**目标**：快速建立研究对象的基本认知

**包含内容**：
- 公司/行业基本信息
- 关键时间节点
- 核心人物
- 主营业务/产品

**搜索策略**：

```typescript
// 搜索优先级
const tier1Sources = [
  "官方网站/About页面",
  "工商注册信息（天眼查/企查查类）",
  "上市公司公告/招股书",
  "百度百科/维基百科（仅作参考，需验证）"
];

// 搜索关键词模板
const searchKeywords = [
  `"${companyName}" 公司简介`,
  `"${companyName}" 成立时间 创始人`,
  `"${companyName}" 主营业务`,
  `"${companyName}" 上市 招股书`
];
```

**数据提取**：

```typescript
interface BasicInfo {
  companyName: string;
  englishName?: string;
  foundedYear: number;
  founders: string[];
  headquarters: string;
  industry: string;
  mainBusiness: string;
  listedStatus: 'listed' | 'unlisted' | 'preparing';
  stockCode?: string;
  employees?: number;
}
```

---

### 二级素材：核心数据层

**目标**：获取可量化的关键指标

**包含内容**：
- 财务数据（营收、利润、现金流）
- 业务数据（用户量、GMV、市占率）
- 估值数据（市值、融资轮次）
- 增长数据（同比、环比）

**搜索策略**：

```typescript
// 数据源优先级（按可信度排序）
const tier2Sources = [
  {
    type: "财报原文",
    priority: 1,
    keywords: [`"${companyName}" 年报 PDF`, `"${companyName}" 财报 investor`]
  },
  {
    type: "证券交易所公告",
    priority: 1,
    keywords: [`"${companyName}" site:sse.com.cn`, `"${companyName}" site:szse.cn`]
  },
  {
    type: "券商研报",
    priority: 2,
    keywords: [`"${companyName}" 研报 投资评级`]
  },
  {
    type: "权威媒体报道",
    priority: 2,
    keywords: [`"${companyName}" 营收 利润 site:36kr.com`]
  }
];
```

**数据提取**：

```typescript
interface DataPoint {
  metric: string;           // 指标名称（如"2023年营收"）
  value: number | string;   // 数值
  unit: string;             // 单位（元、亿、%等）
  timePoint: string;        // 时间点（2023Q4、2023年等）
  comparisonType?: 'YoY' | 'QoQ' | 'vs_competitor';
  comparisonValue?: number; // 对比值（同比增长50%）
  source: string;           // 来源URL
  sourceType: '财报' | '公告' | '研报' | '新闻';
  credibility: 1 | 2 | 3 | 4 | 5;  // 可信度评级
  verified: boolean;        // 是否已交叉验证
  note?: string;            // 备注（如"不同机构测算有差异"）
}

// 示例
const exampleDataPoint: DataPoint = {
  metric: "小米SU7 BOM成本",
  value: 26,
  unit: "万元",
  timePoint: "2024年3月",
  source: "https://example.com/report",
  sourceType: "研报",
  credibility: 3,
  verified: false,
  note: "XX券商测算，仅供参考"
};
```

---

### 三级素材：深度分析层

**目标**：获取行业洞察、专家观点、深度报道

**包含内容**：
- 行业报告（艾瑞、IDC、Gartner等）
- 深度报道（36氪、晚点、财新等）
- 创始人访谈/演讲
- 用户评价/社交媒体反馈

**搜索策略**：

```typescript
const tier3Sources = [
  {
    type: "深度报道",
    outlets: ["36kr.com", "latepost.com", "caixin.com"],
    keywords: [`"${companyName}" 深度 独家`, `"${companyName}" 调查`]
  },
  {
    type: "创始人访谈",
    keywords: [`"${founderName}" 访谈`, `"${founderName}" 演讲 实录`]
  },
  {
    type: "行业报告",
    keywords: [`"${industry}" 行业报告 2023`, `"${industry}" 市场研究`]
  },
  {
    type: "用户反馈",
    outlets: ["zhihu.com", "weibo.com"],
    keywords: [`"${companyName}" 用户体验`, `"${companyName}" 评价`]
  }
];
```

**内容提取**：

```typescript
interface DeepArticle {
  title: string;
  url: string;
  author?: string;
  outlet: string;           // 媒体名称
  publishDate: string;
  keyInsights: string[];    // 关键洞察（AI提取）
  quotes: Array<{           // 关键引语
    speaker: string;
    content: string;
    context: string;
  }>;
  credibility: number;
  verified: boolean;
}
```

---

## 🔍 搜索执行流程

### 阶段1：搜索规划

根据配置对象生成搜索计划：

```typescript
function generateSearchPlan(config: ArticleGenerationConfig): SearchPlan {
  const plan: SearchPlan = {
    tier1: [],  // 基础信息搜索任务
    tier2: [],  // 核心数据搜索任务
    tier3: []   // 深度分析搜索任务
  };
  
  // 根据生成级别决定搜索深度
  if (config.generation.level === 'basic') {
    plan.tier1 = generateBasicSearchTasks();
    plan.tier2 = generateCoreDataSearchTasks();
    // 跳过tier3
  } else if (config.generation.level === 'advanced') {
    plan.tier1 = generateBasicSearchTasks();
    plan.tier2 = generateCoreDataSearchTasks();
    plan.tier3 = generateDeepSearchTasks(2);  // 搜索2轮
  } else {  // professional
    plan.tier1 = generateBasicSearchTasks();
    plan.tier2 = generateCoreDataSearchTasks();
    plan.tier3 = generateDeepSearchTasks(4);  // 搜索4轮，更全面
  }
  
  return plan;
}
```

---

### 阶段2：并行搜索

使用`web_fetch`工具并行执行搜索任务：

```typescript
async function executeTier1Search(subject: string): Promise<BasicInfo> {
  // 搜索任务1：官方网站
  const officialSite = await webFetch(
    `https://www.google.com/search?q="${subject}"+官网`,
    "提取公司名称、成立时间、创始人、主营业务"
  );
  
  // 搜索任务2：工商信息
  const businessInfo = await webFetch(
    `https://www.tianyancha.com/search?key=${subject}`,
    "提取注册资本、法人、注册地址"
  );
  
  // 搜索任务3：上市信息
  const listingInfo = await webFetch(
    `https://www.google.com/search?q="${subject}"+上市+招股书`,
    "提取上市时间、股票代码、募资规模"
  );
  
  // 合并结果
  return mergeBasicInfo([officialSite, businessInfo, listingInfo]);
}
```

**实际调用示例**：

```typescript
// 示例：搜索小米汽车基础信息
const xiaomiCarBasic = await webFetch(
  "https://www.xiaomiev.com/about",
  "提取小米汽车的成立时间、隶属公司、工厂位置、产品线"
);

// 示例：搜索小米汽车财务数据
const xiaomiCarFinancial = await webFetch(
  "https://www.google.com/search?q=小米汽车+销量+订单+2024",
  "提取2024年的销量数据、订单数、交付量"
);
```

---

### 阶段3：数据提取与结构化

从搜索结果中提取关键信息并结构化存储：

```typescript
function extractDataPoints(searchResult: string, sourceURL: string): DataPoint[] {
  const dataPoints: DataPoint[] = [];
  
  // 正则提取所有数字+单位组合
  const patterns = [
    /(\d+\.?\d*)(亿元|万元|元|亿美元|万美元)/g,
    /(\d+\.?\d*)(亿|万|%)/g,
    /(\d+\.?\d*)(辆|台|人|家|个)/g
  ];
  
  patterns.forEach(pattern => {
    const matches = searchResult.matchAll(pattern);
    for (const match of matches) {
      dataPoints.push({
        metric: extractMetricName(match.input, match.index),  // AI识别指标名
        value: parseFloat(match[1]),
        unit: match[2],
        timePoint: extractTimePoint(match.input),  // AI识别时间点
        source: sourceURL,
        sourceType: inferSourceType(sourceURL),
        credibility: rateCredibility(sourceURL),
        verified: false
      });
    }
  });
  
  return dataPoints;
}
```

---

### 阶段4：多源交叉验证

对关键数据进行多源验证：

```typescript
function crossVerifyData(dataPoints: DataPoint[]): DataPoint[] {
  // 按指标分组
  const grouped = groupBy(dataPoints, 'metric');
  
  Object.keys(grouped).forEach(metric => {
    const points = grouped[metric];
    
    // 如果只有单一信源，标记为"待验证"
    if (points.length === 1) {
      points[0].verified = false;
      points[0].note = "单一信源，需进一步验证";
    }
    
    // 如果有多个信源，检查一致性
    else {
      const values = points.map(p => p.value);
      const variance = calculateVariance(values);
      
      // 差异小于10%，视为一致
      if (variance < 0.1) {
        points.forEach(p => {
          p.verified = true;
          p.note = `已通过${points.length}个信源交叉验证`;
        });
      }
      // 差异较大，标记冲突
      else {
        points.forEach(p => {
          p.verified = false;
          p.note = `多源数据存在差异，范围：${Math.min(...values)}-${Math.max(...values)}`;
        });
      }
    }
  });
  
  return dataPoints.flat();
}
```

---

## 📦 素材库数据结构

最终生成的结构化素材库：

```typescript
interface MaterialDatabase {
  // 元信息
  meta: {
    subject: string;
    collectedAt: string;
    searchDepth: 'basic' | 'advanced' | 'professional';
    totalSources: number;
  };
  
  // 基础信息
  basicInfo: BasicInfo;
  
  // 核心数据点
  dataPoints: DataPoint[];
  
  // 深度文章
  articles: DeepArticle[];
  
  // 关键引语
  quotes: Array<{
    speaker: string;
    role: string;
    content: string;
    source: string;
    date: string;
  }>;
  
  // 行业背景
  industryContext?: {
    marketSize: string;
    growthRate: string;
    competitors: string[];
    trends: string[];
  };
  
  // 验证报告
  verificationReport: {
    totalDataPoints: number;
    verifiedCount: number;
    conflictCount: number;
    singleSourceCount: number;
  };
}
```

---

## 🎯 信源可信度评级标准

复用`references/04-fact-checking-standards.md`的分级标准：

| 级别 | 信源类型 | 可信度 | 使用方式 |
|------|---------|-------|----------|
| ⭐⭐⭐⭐⭐ | 上市公司财报、政府文件、法院判决 | 最高 | 直接引用 |
| ⭐⭐⭐⭐ | 券商研报、主流媒体(新华社/财新)、公司官方公告 | 高 | 标注来源后引用 |
| ⭐⭐⭐ | 行业媒体(36氪/晚点)、专业机构报告 | 中等 | 需交叉验证 |
| ⭐⭐ | 自媒体、社交媒体、用户评论 | 较低 | 仅作参考，不作核心依据 |
| ⭐ | 匿名爆料、网络传言 | 极低 | 不使用 |

```typescript
function rateCredibility(sourceURL: string): 1 | 2 | 3 | 4 | 5 {
  // 一级信源（⭐⭐⭐⭐⭐）
  if (sourceURL.includes('investor.') || 
      sourceURL.includes('.gov.') ||
      sourceURL.includes('sse.com.cn') ||
      sourceURL.includes('szse.cn')) {
    return 5;
  }
  
  // 二级信源（⭐⭐⭐⭐）
  if (sourceURL.includes('xinhua') ||
      sourceURL.includes('caixin.com') ||
      sourceURL.endsWith('.com.cn')) {  // 公司官网
    return 4;
  }
  
  // 三级信源（⭐⭐⭐）
  if (sourceURL.includes('36kr.com') ||
      sourceURL.includes('latepost.com') ||
      sourceURL.includes('iresearch.')) {
    return 3;
  }
  
  // 四级信源（⭐⭐）
  if (sourceURL.includes('zhihu.com') ||
      sourceURL.includes('weibo.com')) {
    return 2;
  }
  
  // 默认低可信度
  return 1;
}
```

---

## 🚨 特殊情况处理

### 情况1：搜索失败或信息不足

```typescript
if (searchResult === null || searchResult.length < 100) {
  return {
    status: 'insufficient',
    message: `关于"${subject}"的公开信息较少，建议：
    1. 调整选题为更知名的公司/事件
    2. 选择"用户提供素材"模式，您提供核心信息
    3. 降低生成级别为"基础版"（减少数据需求）`
  };
}
```

---

### 情况2：数据冲突严重

```typescript
if (conflictRate > 0.3) {  // 超过30%的数据冲突
  return {
    status: 'conflict',
    message: `发现多个关键数据存在冲突，建议：
    1. 手动审核素材清单，选择最可信的数据
    2. 在文章中标注"不同机构测算有差异"
    3. 联系我们人工核查（专业版服务）`
  };
}
```

---

### 情况3：敏感信息处理

```typescript
const sensitiveKeywords = ['裁员', '欠薪', '诉讼', '破产', '跑路'];

if (containsSensitiveInfo(searchResult)) {
  return {
    status: 'sensitive',
    message: `检测到敏感信息，将谨慎处理：
    1. 仅使用权威媒体报道（⭐⭐⭐⭐以上）
    2. 必须有官方回应或法律文书支撑
    3. 文中标注"据XX报道"，不做主观判断`
  };
}
```

---

## 📊 素材收集报告

搜索完成后，生成可视化报告：

```markdown
# 素材收集报告

**研究对象**：小米汽车
**搜索时间**：2025-01-28 15:30
**搜索深度**：进阶版

---

## 📈 收集概览

| 类别 | 数量 | 已验证 | 待验证 | 冲突 |
|------|------|--------|--------|------|
| 基础信息 | 1套 | ✅ | - | - |
| 核心数据 | 27个 | 18个 | 7个 | 2个 |
| 深度文章 | 15篇 | - | - | - |
| 关键引语 | 8条 | 6条 | 2条 | - |

---

## ⭐ 信源分布

| 级别 | 数量 | 占比 |
|------|------|------|
| ⭐⭐⭐⭐⭐ | 5个 | 18% |
| ⭐⭐⭐⭐ | 12个 | 44% |
| ⭐⭐⭐ | 10个 | 37% |

---

## ⚠️ 需要关注

**数据冲突**：
- "小米SU7 BOM成本"：3个信源分别为22万/26万/28万（差异27%）
  建议：使用中位数26万，标注"据XX券商测算"

**单一信源**：
- "小米汽车2024年目标销量"：仅来自1篇媒体报道
  建议：标注"据XX报道"，不作为核心论据

---

## ✅ 核心数据清单（已验证）

1. 小米SU7售价：21.59万元起（官方发布会）⭐⭐⭐⭐⭐
2. 首月订单量：超10万张（雷军微博）⭐⭐⭐⭐⭐
3. 小米2023年营收：2710亿元（年报）⭐⭐⭐⭐⭐
4. 小米汽车累计投入：100亿+（官方公告）⭐⭐⭐⭐⭐
...

---

**素材库已生成**：`material-db-xiaomi-car-20250128.json`
**下一步**：生成文章大纲
```

---

## 🔧 使用方式

### 在SKILL.md中调用

```markdown
## 步骤2：素材收集

加载[module:material-collector]：

1. 根据配置对象生成搜索计划
2. 执行分层搜索（基础→核心→深度）
3. 提取并结构化数据
4. 多源交叉验证
5. 生成素材收集报告
6. 询问用户是否满意，或需要补充

确认素材充足后，进入下一步：生成大纲
```

---

## 💡 优化建议

1. **搜索效率**：并行执行多个搜索任务，缩短等待时间
2. **缓存机制**：已搜索过的公司信息缓存7天，避免重复搜索
3. **增量搜索**：如用户不满意，可针对性补充搜索，而非全部重来
4. **人工审核**：专业版支持人工审核关键数据，确保100%准确

---

**状态**：模块设计完成，等待集成到SKILL.md主流程
