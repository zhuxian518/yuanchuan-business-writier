# 事实核查模块

> **职责**：对生成的文章进行四轮系统化核查，确保数据准确、逻辑自洽、引用规范

---

## 🎯 核心功能

事实核查模块负责：
1. 自动识别文章中的所有待核查事实
2. 执行四轮核查流程（数据→时间线→实体→引用）
3. 与素材库交叉比对验证
4. 生成参考文献列表
5. 标注未验证或存在争议的内容

---

## 🔍 四轮核查法

复用`references/04-fact-checking-standards.md`和`references/05-fact-check-execution-guide.md`的方法论。

### 第一轮：数据核查

**目标**：确保每个数字都有据可查

**核查内容**：
- 所有数值（营收、销量、市值、增长率等）
- 单位是否正确（亿/万/元）
- 时间点是否明确（2023年/2023Q4/2023年12月）
- 统计口径是否一致（营收vs净利润、大定vs小定）

**核查流程**：

```typescript
class DataVerifier {
  // Step 1: 提取文章中的所有数据点
  static extractDataPoints(article: string): DataPoint[] {
    const patterns = [
      /(\d+\.?\d*)(亿元|万元|元|亿美元|万美元)/g,
      /(\d+\.?\d*)(亿|万|个|辆|%)/g,
      /(20\d{2})年.*?(\d+\.?\d*)(亿|万|元)/g
    ];
    
    const dataPoints: DataPoint[] = [];
    patterns.forEach(pattern => {
      const matches = article.matchAll(pattern);
      for (const match of matches) {
        dataPoints.push(parseDataPoint(match));
      }
    });
    
    return dataPoints;
  }
  
  // Step 2: 与素材库比对
  static verify(dataPoint: DataPoint, materialDB: MaterialDB): VerificationResult {
    const matchingSources = materialDB.dataPoints.filter(
      dp => dp.metric === dataPoint.metric && 
            dp.timePoint === dataPoint.timePoint
    );
    
    if (matchingSources.length === 0) {
      return { status: 'not_found', message: '素材库无对应数据，需补充信源' };
    }
    
    // 检查数值一致性
    const isConsistent = matchingSources.some(
      source => Math.abs(source.value - dataPoint.value) < dataPoint.value * 0.05
    );
    
    if (!isConsistent) {
      return { 
        status: 'mismatch', 
        message: `文章数据${dataPoint.value}与素材库${matchingSources[0].value}不一致`
      };
    }
    
    return { status: 'verified', sources: matchingSources };
  }
}
```

**核查输出示例**：

```markdown
### 第一轮：数据核查

✅ 已核查：23个数据点
⚠️ 需修正：2个

**需修正项**：

1. 第3段："小米2023年营收2800亿元"
   - 问题：数值错误
   - 正确值：2710亿元
   - 来源：小米集团2023年年报 P12

2. 第7段："特斯拉2023年销量200万辆"
   - 问题：数值不精确
   - 正确值：180.9万辆
   - 来源：特斯拉2023年Q4财报

❓ 待补充信源：1个

3. 第10段："小米汽车首月订单突破10万张"
   - 问题：素材库无对应数据
   - 建议：补充搜索"雷军微博 小米SU7 订单"
```

---

### 第二轮：时间线核查

**目标**：确保时间节点准确，逻辑自洽

**核查内容**：
- 所有时间点（年份、月份、具体日期）
- 事件先后顺序
- "之前/之后/同期"等相对时间关系

**核查流程**：

```typescript
class TimelineVerifier {
  // Step 1: 提取时间节点
  static extractTimePoints(article: string): TimePoint[] {
    const patterns = [
      /(20\d{2})年(\d{1,2})月(\d{1,2})日/g,  // 2024年3月28日
      /(20\d{2})年(\d{1,2})月/g,              // 2024年3月
      /(20\d{2})年/g,                          // 2024年
      /(20\d{2})Q([1-4])/g                     // 2024Q1
    ];
    
    return extractMatches(patterns, article);
  }
  
  // Step 2: 检查逻辑一致性
  static checkConsistency(timePoints: TimePoint[]): Issue[] {
    const issues: Issue[] = [];
    
    // 检查：事件A说在事件B之前，但时间点却晚于B
    timePoints.forEach((tp, i) => {
      if (tp.relativeClause && tp.relativeClause.includes('之前')) {
        const referenceEvent = findReferenceEvent(tp.relativeClause, timePoints);
        if (referenceEvent && tp.date > referenceEvent.date) {
          issues.push({
            type: 'timeline_conflict',
            description: `逻辑矛盾：事件"${tp.event}"标注为"在XX之前"，但实际时间晚于参考事件`
          });
        }
      }
    });
    
    return issues;
  }
  
  // Step 3: 与素材库比对
  static verify(timePoint: TimePoint, materialDB: MaterialDB): VerificationResult {
    // 查找素材库中的对应事件
    const match = materialDB.articles.find(
      article => article.title.includes(timePoint.event)
    );
    
    if (!match) {
      return { status: 'not_found' };
    }
    
    // 检查时间是否一致
    if (match.publishDate !== timePoint.date) {
      return {
        status: 'mismatch',
        message: `文章中时间${timePoint.date}与素材库${match.publishDate}不一致`
      };
    }
    
    return { status: 'verified' };
  }
}
```

**核查输出示例**：

```markdown
### 第二轮：时间线核查

✅ 已核查：15个时间点
⚠️ 需修正：1个

**需修正项**：

1. 第5段："2023年，小米SU7发布"
   - 问题：时间错误
   - 正确时间：2024年3月28日
   - 来源：小米汽车官方发布会

❓ 逻辑检查：0个问题
```

---

### 第三轮：实体核查

**目标**：确保人名、公司名、产品名无误

**核查内容**：
- 人名拼写（雷军/李想/马斯克）
- 职位准确性（CEO/创始人/董事长）
- 公司名全称（小米集团/小米科技/北京小米科技有限责任公司）
- 产品名规范（小米SU7/小米SU7 Max）

**核查流程**：

```typescript
class EntityVerifier {
  // Step 1: 提取实体
  static extractEntities(article: string): Entity[] {
    // 使用NER（命名实体识别）或正则
    const personPattern = /(雷军|李想|马斯克|[A-Z][a-z]+\s[A-Z][a-z]+)/g;
    const companyPattern = /(小米|特斯拉|比亚迪|[A-Z][a-z]+\sInc\.?)/g;
    
    return {
      persons: extractMatches(personPattern, article),
      companies: extractMatches(companyPattern, article)
    };
  }
  
  // Step 2: 检查拼写和规范性
  static verify(entity: Entity, materialDB: MaterialDB): VerificationResult {
    // 检查是否有常见错误
    const commonErrors = {
      '雷君': '雷军',
      '李详': '李想',
      '小米科技': '小米集团',
      'Telsa': 'Tesla'
    };
    
    if (commonErrors[entity.name]) {
      return {
        status: 'spelling_error',
        suggestion: commonErrors[entity.name]
      };
    }
    
    // 检查职位是否准确
    if (entity.type === 'person' && entity.title) {
      const correctTitle = materialDB.basicInfo.founders.find(
        f => f.name === entity.name
      )?.title;
      
      if (correctTitle && correctTitle !== entity.title) {
        return {
          status: 'title_mismatch',
          message: `文章中职位"${entity.title}"与实际"${correctTitle}"不符`
        };
      }
    }
    
    return { status: 'verified' };
  }
}
```

**核查输出示例**：

```markdown
### 第三轮：实体核查

✅ 已核查：人名12个，公司名8个
⚠️ 需修正：1个

**需修正项**：

1. 第8段："小米科技创始人雷军"
   - 问题：公司名称不规范
   - 建议：改为"小米集团创始人雷军"
   - 原因：2018年已改名为"小米集团"

❓ 职位检查：0个问题
```

---

### 第四轮：引用核查

**目标**：确保所有引用有出处，格式规范

**核查内容**：
- 直接引语（"..."）是否标注来源
- 关键数据是否标注信源
- 是否有参考文献列表
- 引用格式是否规范

**核查流程**：

```typescript
class CitationVerifier {
  // Step 1: 提取所有引用
  static extractCitations(article: string): Citation[] {
    const quotePattern = /["「](.*?)["」]/g;  // 直接引语
    const dataPattern = /据.*?(报道|报告|财报|研报|显示)/g;  // 间接引用
    
    return {
      directQuotes: extractMatches(quotePattern, article),
      indirectCitations: extractMatches(dataPattern, article)
    };
  }
  
  // Step 2: 检查是否有出处
  static verify(citation: Citation, materialDB: MaterialDB): VerificationResult {
    // 检查引语是否在素材库中
    const found = materialDB.quotes.find(
      q => q.content.includes(citation.content)
    );
    
    if (!found) {
      return {
        status: 'source_missing',
        message: '引语缺少来源，需补充出处'
      };
    }
    
    // 检查文中是否标注了来源
    const hasSourceTag = citation.context.includes('据') || 
                         citation.context.includes('表示') ||
                         citation.context.includes('说');
    
    if (!hasSourceTag) {
      return {
        status: 'tag_missing',
        message: '引语未标注来源，建议加入"据XX报道"或"XX表示"'
      };
    }
    
    return { status: 'verified', source: found.source };
  }
  
  // Step 3: 生成参考文献
  static generateReferences(citations: Citation[], materialDB: MaterialDB): string[] {
    const references: string[] = [];
    let index = 1;
    
    // 为每个引用生成参考文献条目
    citations.forEach(citation => {
      const source = findSource(citation, materialDB);
      if (source) {
        references.push(
          `[${index}] ${source.title}, ${source.author}, ${source.outlet}, ${source.date}, ${source.url}`
        );
        index++;
      }
    });
    
    return references;
  }
}
```

**核查输出示例**：

```markdown
### 第四轮：引用核查

✅ 已核查：直接引语8条，数据引用15处
⚠️ 需修正：2条

**需修正项**：

1. 第6段："雷军曾说过：'这是我最后一次创业，不能留遗憾。'"
   - 问题：引语缺少来源
   - 建议：补充"据XX访谈"或"在XX演讲中表示"
   - 素材库中有对应引语：2023年小米汽车技术发布会

2. 第12段："据知情人士透露..."
   - 问题："知情人士"不是有效信源
   - 建议：如无法追溯到权威来源，删除此句

✅ 参考文献已生成（12条）
```

---

## 📋 综合核查报告

四轮核查完成后，生成综合报告：

```markdown
# 事实核查报告

**文章标题**：小米汽车亏本卖，雷军到底在下什么棋？
**字数**：8520字
**核查时间**：2025-01-28 16:30
**核查人员**：AI自动核查

---

## 📊 核查概览

| 核查轮次 | 核查项 | 已验证 | 需修正 | 待补充 |
|---------|--------|--------|--------|--------|
| 第一轮：数据 | 23个 | 21个 | 2个 | 0个 |
| 第二轮：时间线 | 15个 | 14个 | 1个 | 0个 |
| 第三轮：实体 | 20个 | 19个 | 1个 | 0个 |
| 第四轮：引用 | 23处 | 21处 | 2处 | 0个 |

**总计**：81项核查，75项通过，6项需修正

---

## ⚠️ 需要修正的6项

### 数据类（2项）

1. **第3段**："小米2023年营收2800亿元"
   - ❌ 错误值：2800亿元
   - ✅ 正确值：2710亿元
   - 📎 来源：小米集团2023年年报 P12

2. **第7段**："特斯拉2023年销量200万辆"
   - ❌ 不精确：200万辆
   - ✅ 精确值：180.9万辆
   - 📎 来源：特斯拉2023年Q4财报

### 时间线类（1项）

3. **第5段**："2023年，小米SU7发布"
   - ❌ 错误时间：2023年
   - ✅ 正确时间：2024年3月28日
   - 📎 来源：小米汽车官方发布会

### 实体类（1项）

4. **第8段**："小米科技创始人雷军"
   - ❌ 名称过时："小米科技"
   - ✅ 当前名称："小米集团"
   - 📎 备注：2018年已更名

### 引用类（2项）

5. **第6段**："雷军曾说过：'这是我最后一次创业，不能留遗憾。'"
   - ⚠️ 缺少出处
   - ✅ 建议：补充"在2023年小米汽车技术发布会上表示"

6. **第12段**："据知情人士透露..."
   - ⚠️ 信源不明确
   - ✅ 建议：如无法追溯来源，删除此句

---

## 📚 参考文献（已生成12条）

[1] 小米集团2023年年度报告, 小米集团, 2024-03-19, https://ir.mi.com/...

[2] 特斯拉2023年第四季度及全年财报, Tesla Inc., 2024-01-24, https://ir.tesla.com/...

[3] 小米汽车技术发布会实录, 小米汽车, 2023-12-28, https://www.xiaomiev.com/...

[4] 小米SU7订单突破10万张, 雷军微博, 2024-04-03, https://weibo.com/leijun/...

[5] 小米SU7成本拆解分析, XX券商研究所, 2024-03-30, [研报链接]

[6] 中国新能源汽车市场研究报告2023, 中汽协, 2024-01-15, http://www.caam.org.cn/...

... (共12条)

---

## ✅ 核查结论

**整体评估**：文章事实基础扎实，92.6%的事实点已验证通过

**修正建议**：
1. 修正上述6项错误/不规范表述
2. 补充2处引用来源
3. 在文末添加参考文献列表

**风险提示**：
- 无高风险错误（如重大数据错误、时间线冲突）
- 无敏感信息处理不当
- 无抄袭风险

**核查通过**：✅ 修正后可发布
```

---

## 🔧 自动修正功能

对于明确的错误，系统可自动修正：

```typescript
function autoFix(article: string, issues: Issue[]): string {
  let fixedArticle = article;
  
  issues.forEach(issue => {
    if (issue.type === 'data_mismatch' && issue.autoFixable) {
      fixedArticle = fixedArticle.replace(
        issue.wrongValue,
        issue.correctValue
      );
    }
  });
  
  return fixedArticle;
}
```

**示例**：

```markdown
🔧 自动修正完成（3处）

✅ "2800亿元" → "2710亿元"
✅ "200万辆" → "180.9万辆"
✅ "2023年" → "2024年3月28日"

⚠️ 以下2处需人工确认：
- 引语来源补充
- "据知情人士"的处理方式
```

---

## 🚀 使用方式

### 在SKILL.md中调用

```markdown
## 步骤6：事实核查

加载[module:fact-checker]：

1. 执行四轮核查（数据→时间线→实体→引用）
2. 与素材库交叉验证
3. 生成修正建议
4. 自动修正明确错误
5. 生成参考文献列表
6. 输出综合核查报告

核查通过后，进入下一步：输出文章
```

---

## 💡 优化建议

1. **AI辅助识别**：使用NLP技术自动提取实体和时间点，提高准确率
2. **增量核查**：每生成一段就核查一段，而非全文生成后再核查
3. **人工复核**：专业版支持人工复核关键数据，确保100%准确
4. **历史记录**：记录常见错误，建立错误库，避免重复犯错

---

**状态**：模块设计完成，等待集成到SKILL.md主流程
