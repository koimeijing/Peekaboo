# 竞品内容追踪器 - 项目记忆

## 项目概述
本项目是一个本地运行的竞品内容追踪与分析工具，单文件 HTML+CSS+JS，无后端。
文件位置：D:\All-AIProjects\competitor-tracker\index.html
使用浏览器直接打开：file:///D:/All-AIProjects/competitor-tracker/index.html

## 用户背景
视觉设计师，负责海外儿童沙盒游戏 AHA World 的视觉物料设计。
主要竞品：Toca Boca、Avatar World、米加小镇。
零代码基础，纯 vibe coding 路线。

## 技术栈
- 纯前端：HTML + CSS + JavaScript，单文件
- AI 分析：DeepSeek API（deepseek-chat 模型，max_tokens: 2400）
- 数据存储：localStorage（key: `cpt-tracker-v2`）+ 支持导出/导入 JSON 文件
- 代理：Clash Verge，端口 7897

## 已完成功能
1. 内容录入：品牌标签管理、平台标签管理、日期、链接、内容描述、图片上传/粘贴
2. **双图上传区**：内容截图 + 互动数据截图并排显示（各自独立上传/拖拽/Ctrl+V 粘贴）
3. AI 分析：调用 DeepSeek API，含图像识别（最多2张图同时传入）
4. 内容库：卡片展示、多维筛选、点击查看详情
5. 时间线：按发布日期分组的垂直时间线
6. 设置：DeepSeek API Key 管理、品牌/平台标签管理

## 默认预设
品牌：Toca Boca、Avatar World、米加小镇
平台：Instagram、App Store、Google Play、TikTok、小红书

## 数据结构（entry 对象关键字段）
```js
{
  id, brand, platform, date, link, description,
  imageBase64: '',           // 内容截图（压缩后 base64）
  engagementImageBase64: '', // 互动数据截图（压缩后 base64）
  manualTags: [],
  analysis: null,            // AI 分析结果对象（见下）
  createdAt, updatedAt
}
```

### 旧数据兼容字段（只读，不再写入）
- `engagement`：旧版互动数据文字，渲染时做 fallback 展示
- `analysis.threatLevel`：旧版营销力度字段，读取时 fallback 到 `marketingIntensity`
- `analysis.insight` / `analysis.recommendations`：旧字段，fallback 到新字段

## AI 分析结果对象（analysis）字段说明
| 字段 | 类型 | 说明 |
|---|---|---|
| `contentType` | string | 内容形式（具体描述） |
| `culturalContext` | string | 🌍 文化背景解读（节日/流行文化/社会话题的海外背景，照顾中国读者视角） |
| `trendSignal` | string | 📈 流行趋势判断（是否借势海外热点、受众国家、趋势热度阶段） |
| `commercialPurpose` | string | 💼 商业目的深挖（叙述文本，旧版为 string[] 标签数组） |
| `visualStrategy` | string | 🎨 视觉策略解码（色彩心理、构图意图、与商业目标的关系） |
| `emotionalTone` | string | 情感联结定位（一句话） |
| `targetAudience` | string | 目标受众（年龄+身份+国家/地区+痛点） |
| `engagementInsight` | string | 💬 传播力洞察（从截图读取数字或预判） |
| `tags` | string[] | 3-5个关键词标签 |
| `strategicInsight` | string | 💡 核心战略洞察（旧字段名 `insight`） |
| `ahaRecommendations` | string | 🎯 AHA World 启示（旧字段名 `recommendations`） |
| `marketingIntensity` | number | 营销力度 1-5 |
| `marketingIntensityReason` | string | 评分一句话依据（附在分数 badge 后） |

## AI Prompt 角色设定（当前版本）
```
你是一位拥有10年经验的海外儿童游戏市场分析师，同时具备跨文化营销专业知识。
服务对象：中国视觉设计师，负责 AHA World 视觉物料设计。
分析对象：Toca Boca、Avatar World、米加小镇等，核心用户为欧美/东南亚3-10岁儿童及其家长。
要求：有具体判断、有跨文化洞察、照顾中国读者对海外文化的认知差距，绝不泛泛罗列。
```

## 关键常量
```js
const STORAGE_KEY = 'cpt-tracker-v2';
const DEFAULT_BRANDS   = ['Toca Boca', 'Avatar World', '米加小镇'];
const DEFAULT_PLATFORMS = ['Instagram', 'App Store', 'Google Play', 'TikTok', '小红书'];
const MINT_LABEL = {1:'日常维护', 2:'轻度推广', 3:'常规推广', 4:'重点推广', 5:'重点战役'};
const MINT_CLASS = {1:'m1', 2:'m2', 3:'m3', 4:'m4', 5:'m5'};
```

## 关键全局状态
```js
let formState = { brand: '', platform: '', imageBase64: '', engagementImageBase64: '' };
let lastPasteZone = 'content'; // 'content' | 'engagement'，决定 Ctrl+V 粘贴到哪个图区
let currentModalId = null;
```

## 双图上传区实现要点
- **内容截图**：zone = `'content'`，元素 ID：`imgZone` / `imgPreview` / `imgPreviewImg` / `imgSizeHint` / `imgFileInput`
- **互动截图**：zone = `'engagement'`，元素 ID：`engZone` / `engPreview` / `engPreviewImg` / `engSizeHint` / `engFileInput`
- `handleImageFile(file, zone)` → 压缩后存入对应 formState 字段
- `showImagePreview(dataUrl, zone)` → 显示对应预览区
- `clearImage(zone)` → 清空对应区域
- `onmouseenter` 更新 `lastPasteZone`，实现 Ctrl+V 定向粘贴
- AI 发送时：最多2个 `image_url` 块（内容图 + 互动图），任一存在则先尝试带图，400/422 时降级纯文本重试

## CSS 分析框样式类
| 类名 | 颜色 | 用途 |
|---|---|---|
| `.cultural-box` | 蓝色系 | 文化背景解读 |
| `.trend-box` | 紫色系 | 流行趋势判断 |
| `.engagement-box` | 黄色系 | 传播力洞察 |
| `.insight-box` | 主色系 | 核心战略洞察 |
| `.recommendation-box` | 绿色系 | AHA World 启示 |

## 待优化方向
- 方向E：根据物料需求描述搜索视觉参考图，输出 Pinterest/Google 可用搜索词

## 注意事项
- 每次修改后用浏览器直接打开 index.html 验证，不要依赖 Claude Code 预览
- PowerShell 启动 Claude Code 前需设置代理：$env:HTTPS_PROXY="http://127.0.0.1:7897"
