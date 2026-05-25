# Peekaboo — 项目记忆

## 基本信息
- **工具名称**：Peekaboo（原名"竞品内容追踪器"）
- **Slogan**：Peek into what makes content work / 让我看看什么在偷偷发力～
- **文件路径**：`D:\All-AIProjects\competitor-tracker\index.html`
- **打开方式**：浏览器直接打开 `file:///D:/All-AIProjects/competitor-tracker/index.html`
- **架构**：纯前端单文件，HTML + CSS + JS，无后端，无构建工具

## 用户背景
视觉设计师，负责海外儿童沙盒游戏 **AHA World** 的视觉物料设计。
主要竞品：Toca Boca、Avatar World、米加小镇。零代码基础，vibe coding 路线。

## 技术栈
- AI 分析：DeepSeek API（`deepseek-chat` 模型，`max_tokens: 2400`）
- 数据存储：`localStorage`（key: `cpt-tracker-v2`）+ 导出/导入 JSON
- 代理：Clash Verge，端口 7897（PowerShell 启动前：`$env:HTTPS_PROXY="http://127.0.0.1:7897"`）

## 已实现功能

| Tab | 功能 |
|---|---|
| ✏️ 录入 | 品牌/平台选择、日期、链接、描述、多图上传、AI 分析 |
| 🗂 收藏库 | 卡片网格、多维筛选、点击查看详情、更新互动数据入口 |
| 📅 时间线 | 按发布日期分组的垂直时间线 |
| ⚙️ 设置 | API Key、品牌管理、平台（只读）、本地文件绑定、数据概览 |

## 关键常量
```js
const STORAGE_KEY      = 'cpt-tracker-v2';
const DEFAULT_BRANDS   = ['Toca Boca', 'Avatar World', '米加小镇'];
const FIXED_PLATFORMS  = ['Instagram', 'TikTok', 'Facebook', 'YouTube', '小红书', 'App Store', 'Google Play'];
// 平台固定，不允许用户增删，loadFromStorage 每次强制覆盖
const MAX_CONTENT_IMAGES = 9;
const MINT_LABEL = {1:'日常维护', 2:'轻度推广', 3:'常规推广', 4:'重点推广', 5:'重点战役'};
const MINT_CLASS = {1:'m1', 2:'m2', 3:'m3', 4:'m4', 5:'m5'};
const BRAND_PALETTE = [ // 品牌标签循环色（hash 取模）
  { bg:'#FFE8F3', text:'#C0174F' }, { bg:'#E5F3FF', text:'#0060C0' },
  { bg:'#FFF8CC', text:'#806000' }, { bg:'#FFF2E8', text:'#C04010' },
  { bg:'#E8FBE8', text:'#1A8020' }
];
```

## 全局状态
```js
let formState = { brand: '', platform: '', contentImages: [], engagementImageBase64: '' };
let lastPasteZone = 'content'; // 'content' | 'engagement'，Ctrl+V 路由
let currentModalId = null;
let engPanelEntryId = null;
let engPanelImageBase64 = '';
let _lbImages = []; let _lbIndex = 0; // Lightbox 状态
```

## 数据结构（entry 对象）
```js
{
  id, brand, platform, date, link, description,
  contentImages: [],           // 内容截图数组，最多9张，base64（新）
  imageBase64: '',             // contentImages[0] 的副本，向后兼容旧渲染
  engagementImageBase64: '',   // 互动数据截图，单张 base64
  manualTags: [],
  analysis: null,              // AI 分析结果（见下）
  createdAt, updatedAt
}
```

### 旧数据兼容（只读，不再写入）
- `imageUrl`：极旧版字段，渲染时 fallback
- `engagement`：旧版互动数据文字，渲染时 fallback 展示
- `analysis.threatLevel` → fallback 到 `marketingIntensity`
- `analysis.insight` / `analysis.recommendations` → fallback 到新字段名

### loadFromStorage 迁移逻辑
```js
// 平台强制覆盖
db.settings.platforms = [...FIXED_PLATFORMS];
// 旧单图迁移
db.entries.forEach(e => {
  if (!e.contentImages) e.contentImages = e.imageBase64 ? [e.imageBase64] : [];
});
```

## 图片上传系统

### 内容截图（多图，最多9张）
- 元素：`#contentImagesGrid`（3列1:1网格，JS 动态渲染）
- 文件 input：`#imgFileInput`（`multiple`）
- 关键函数：`renderContentImagesGrid()` / `_addContentImagesFromFiles(files)` / `removeContentImage(index)`
- 支持：多选文件、拖拽追加、Ctrl+V 粘贴追加
- 点击缩略图 → `openLightbox(formState.contentImages, i)` 全屏查看

### 互动数据截图（单张，不变）
- 元素：`#engZone` / `#engPreview` / `#engPreviewImg` / `#engSizeHint` / `#engFileInput`
- 关键函数：`handleImageFile(file)` / `showEngImagePreview(dataUrl)` / `clearImage()`
- `onmouseenter` 将 `lastPasteZone` 设为 `'engagement'`

### Lightbox（全图查看）
- HTML：`#lightboxOverlay` > `.lightbox-content` > `#lightboxImg`
- 函数：`openLightbox(images, index)` / `closeLightbox()` / `shiftLightbox(dir)` / `openEntryLightbox(entryId, index)`
- 键盘：ESC 关闭，← → 切换

### Paste 路由逻辑
```
粘贴事件 →
  lightbox 开着 → 忽略
  engPanel 开着 → handleEngPanelFile
  录入 tab 激活 →
    lastPasteZone === 'content'     → _addContentImagesFromFiles
    lastPasteZone === 'engagement'  → handleImageFile
```

### 图片压缩
```js
compressImage(dataUrl, maxW, quality)
// 内容图：maxW=1200, quality=0.80
// 互动图：maxW=900,  quality=0.75
```

## 更新互动数据面板（Eng Panel）
- HTML：`#engOverlay` > `.eng-panel`（z-index: 600，高于 modal 的 500）
- 触发：收藏库卡片「📊 更新互动数据」按钮 → `openEngPanel(id)`
- 保存后自动调用 `analyzeEntry(id)` 重新分析
- ESC 优先关闭此面板

## AI 分析

### Vision 图片发送
- 内容图最多取前3张 + 互动图1张，拼成 `image_url` blocks
- 任一图存在则先带图请求；遇 400/422 降级纯文本重试

### analysis 对象字段
| 字段 | 说明 |
|---|---|
| `contentType` | 内容形式 |
| `culturalContext` | 🌍 文化背景解读 |
| `trendSignal` | 📈 流行趋势判断 |
| `commercialPurpose` | 💼 商业目的深挖（string；旧版为 string[]，兼容展示） |
| `visualStrategy` | 🎨 视觉策略解码 |
| `emotionalTone` | 情感联结（一句话） |
| `targetAudience` | 目标受众 |
| `engagementInsight` | 💬 传播力洞察 |
| `tags` | string[]，3-5个关键词 |
| `strategicInsight` | 💡 核心战略洞察（旧字段名 `insight`） |
| `ahaRecommendations` | 🎯 AHA World 启示（旧字段名 `recommendations`） |
| `marketingIntensity` | number 1-5 |
| `marketingIntensityReason` | 评分依据一句话 |

## 视觉设计系统

### 品牌色
```css
--pink: #FF8DBB; --blue: #59ACFF; --yellow: #FFE359; --orange: #FF6F42; --green: #62D963;
```

### 关键 Token
```css
--bg: #F6F7FB;              /* body 背景实为 #F0F2FA（body 样式内直接写死）*/
--bg-card: rgba(255,255,255,0.72);
--border: #E8E4DD;
--r: 12px; --rl: 22px;
```

### 背景装饰 Blobs（`position:fixed; z-index:0`）
- `.b1`：右上，粉色，opacity 0.65
- `.b2`：左下，蓝色，opacity 0.55
- `.b3`：页面中部，黄色，opacity 0.35
- 所有内容层加 `position:relative; z-index:1`

### 玻璃拟态规范
- `.card`：`rgba(255,255,255,0.72)` + `blur(12px)` + `border-radius:22px`
- `.content-card`：`rgba(255,255,255,0.55)` + `blur(16px)` + `border-radius:20px`
- `.stat-card` / `.tl-card`：`rgba(255,255,255,0.60)` + `blur(12px)`
- `.modal` / `.eng-panel`：`rgba(255,255,255,0.85)` + `blur(20px)`
- `.ts-pill`：`rgba(255,255,255,0.65)` + `blur(10px)` + inset 高光

### 标签/徽章色系
所有 `.tag` 变体和 `.m1`–`.m5` 均为对应色系的极淡半透明底色（非实色），保持通透感。

### AI 分析框
| 类名 | 色系 |
|---|---|
| `.cultural-box` | 蓝色系 |
| `.trend-box` | 粉色系 |
| `.engagement-box` | 橙色系 |
| `.insight-box` | 黄色系 |
| `.recommendation-box` | 绿色系 |

## Logo 文件（competitor-tracker 目录）
- `logo-color.png`（header 使用）
- `logo-black.png` / `logo-white.png` / `logo-stacked.png` / `logo-black-color.png`

## 注意事项
- 修改后用浏览器直接打开 index.html 验证，不要依赖 Claude Code 预览
- PowerShell 启动前需设置代理：`$env:HTTPS_PROXY="http://127.0.0.1:7897"`
- 平台列表固定，不要改为可编辑
- 修改 JS 时注意 `imageBase64` 只作为 `contentImages[0]` 的向后兼容副本，真正的数据在 `contentImages[]`
