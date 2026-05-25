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
- AI 分析：DeepSeek API（`deepseek-chat` / `deepseek-reasoner` 可切换，`max_tokens: 2400`）
- 数据存储：`localStorage`（key: `cpt-tracker-v2`）+ File System Access API 实时同步 + 导出/导入 JSON
- 代理：Clash Verge，端口 7897（PowerShell 启动前：`$env:HTTPS_PROXY="http://127.0.0.1:7897"`）

## 已实现功能

| Tab | 功能 |
|---|---|
| ✏️ 录入 | 品牌/平台选择、日期、链接、描述、内容截图多图上传、互动截图多图上传、AI 分析 |
| 🗂 收藏库 | 卡片网格、品牌pill筛选、多维筛选（平台/日期/分析状态/排序）、清除筛选、批量管理、点击查看详情、更新互动数据入口 |
| 📅 时间线 | 按发布日期分组的垂直时间线，支持搜索/平台/品牌筛选 |
| ⚙️ 设置 | API Key（密码可见切换）、AI 模型选择、测试连接、品牌管理、平台（只读）、本地文件绑定/新建、数据概览、清空数据、关于 |
| 🌙 深色模式 | Header 右侧按钮切换，持久化到 `db.settings.theme` |

## 关键常量
```js
const STORAGE_KEY      = 'cpt-tracker-v2';
const DEFAULT_BRANDS   = ['Toca Boca', 'Avatar World', '米加小镇'];
const FIXED_PLATFORMS  = ['Instagram', 'TikTok', 'Facebook', 'YouTube', '小红书', 'App Store', 'Google Play'];
const DEFAULT_PLATFORMS = FIXED_PLATFORMS; // 向后兼容别名
// 平台固定，不允许用户增删，loadFromStorage 每次强制覆盖
const MAX_CONTENT_IMAGES = 9;
const MAX_ENG_IMAGES     = 9;
const MINT_LABEL = {1:'日常维护', 2:'轻度推广', 3:'常规推广', 4:'重点推广', 5:'重点战役'};
const MINT_CLASS = {1:'m1', 2:'m2', 3:'m3', 4:'m4', 5:'m5'};
const MINT_DESC  = '1=日常维护 · 3=常规推广 · 5=重点战役';
const BRAND_PALETTE = [ // 品牌标签循环色（hash 取模）
  { bg:'#FFE8F3', text:'#C0174F' }, { bg:'#E5F3FF', text:'#0060C0' },
  { bg:'#FFF8CC', text:'#806000' }, { bg:'#FFF2E8', text:'#C04010' },
  { bg:'#E8FBE8', text:'#1A8020' }
];
```

## 全局状态
```js
let db = { entries: [], settings: { apiKey:'', model:'deepseek-chat', theme:'light', brands:[...], platforms:[...] } };
let syncFileHandle = null;          // File System Access API 文件句柄
let formState = { brand: '', platform: '', contentImages: [], engagementImages: [] };
let lastPasteZone = 'content';      // 'content' | 'engagement'，Ctrl+V 路由（录入 tab 内）
let currentModalId = null;
// 收藏库筛选
let _fbBrand = '';                  // 品牌pill选中值（空串=全部）
// 批量管理
let batchMode = false;
let batchSelected = new Set();
// Eng Panel
let engPanelEntryId = null;
let engPanelImages = [];            // 当前编辑的互动截图数组
// Lightbox
let _lbImages = []; let _lbIndex = 0;
```

## 数据结构（entry 对象）
```js
{
  id, brand, platform, date, link, description,
  contentImages: [],           // 内容截图数组，最多9张，base64
  imageBase64: '',             // contentImages[0] 的副本，向后兼容
  engagementImages: [],        // 互动数据截图数组，最多9张，base64
  engagementImageBase64: '',   // engagementImages[0] 的副本，向后兼容
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
db.settings.platforms = [...FIXED_PLATFORMS]; // 平台强制覆盖
db.entries.forEach(e => {
  if (!e.contentImages)    e.contentImages    = e.imageBase64          ? [e.imageBase64]          : [];
  if (!e.engagementImages) e.engagementImages = e.engagementImageBase64 ? [e.engagementImageBase64] : [];
});
```

## 图片上传系统

### 内容截图（多图，最多9张）
- 元素：`#contentImagesGrid`（3列1:1网格，JS 动态渲染）
- 文件 input：`#imgFileInput`（`multiple`）
- 关键函数：`renderContentImagesGrid()` / `_addContentImagesFromFiles(files)` / `removeContentImage(index)`
- 支持：多选文件、拖拽追加、Ctrl+V 粘贴追加（`lastPasteZone='content'`时）
- 点击缩略图 → `openLightbox(formState.contentImages, i)` 全屏查看

### 互动数据截图（多图，最多9张）
- 元素：`#engImagesGrid`（3列1:1网格，同 content 完全一致）
- 文件 input：`#engImgFileInput`（`multiple`）
- 关键函数：`renderEngImagesGrid()` / `_addEngImagesFromFiles(files)` / `removeEngImage(index)`
- 支持：多选文件、拖拽追加、Ctrl+V 粘贴追加（鼠标悬停区域时 `lastPasteZone='engagement'`）
- 点击缩略图 → `openLightbox(formState.engagementImages, i)` 全屏查看

### Lightbox（全图查看）
- HTML：`#lightboxOverlay` > `.lightbox-content` > `#lightboxImg`
- 函数：`openLightbox(images, index)` / `closeLightbox()` / `shiftLightbox(dir)`
- 入口函数：`openEntryLightbox(entryId, index)`（内容图）/ `openEntryEngLightbox(entryId, index)`（互动图）
- 键盘：ESC 关闭，← → 切换

### Paste 路由逻辑
```
粘贴事件 →
  lightboxOverlay.open → 忽略
  engOverlay.open      → _addEngPanelImagesFromFiles（engPanel 检测靠 overlay 状态，不靠 lastPasteZone）
  录入 tab 激活 →
    lastPasteZone === 'content'    → _addContentImagesFromFiles
    lastPasteZone === 'engagement' → _addEngImagesFromFiles
```
> **注意**：`lastPasteZone` 只有 `'content'` 和 `'engagement'` 两个有效路由值。engPanel 区域的 `onmouseenter` 会设置 `lastPasteZone='engPanel'`，但 paste handler 不读取它，engPanel 的粘贴靠 `engOverlay.open` 单独拦截。

### 图片压缩
```js
compressImage(dataUrl, maxW, quality)
// 内容图：maxW=1200, quality=0.80
// 互动图：maxW=900,  quality=0.75
```

## 更新互动数据面板（Eng Panel）
- HTML：`#engOverlay`（class: `eng-overlay`）> `.eng-panel`（z-index: 600，高于 modal 的 500）
- 触发：收藏库卡片「📊 更新互动数据」按钮 → `openEngPanel(id)`
- 状态：`engPanelEntryId` + `engPanelImages[]`（加载时从 entry.engagementImages 复制）
- 面板渲染：`renderEngPanelGrid()` — 动态写入 `#engZoneWrap`，含 `content-img-grid` + 隐藏 file input `#engPanelFileInput`
- 保存：`saveEngPanel()` → `updateEntry()` 写入数组 → `closeEngPanel()` → `analyzeEntry(id)` 自动重新分析
- ESC 优先关闭此面板（ESC handler 中先判断 engOverlay 再判断 modal）

## 收藏库

### 品牌筛选 Pills
- HTML：`<div id="fb-brand-pills" class="fb-brand-pills">`
- 渲染函数：`populateBrandFilters()` — 生成"全部品牌" + db.settings.brands 的 pill 按钮
- **关键**：onclick 必须用 `data-brand` 属性传参（不能用 `JSON.stringify` 内联，双引号破坏属性）：
  ```js
  `<button class="fb-bp" data-brand="${escAttr(b)}" onclick="setFbBrand(this.dataset.brand)">`
  ```
- 切换函数：`setFbBrand(val)` → 更新 `_fbBrand` → 重渲 pills → `renderLibrary()`
- 筛选逻辑：精确匹配（`e.brand === opts.brand`），空串显示全部
- 需调用 `populateBrandFilters()` 的时机：`renderAll()`、切换到收藏库 tab、`tsConfirmAdd('brand')`、`mgrRemove('brand', …)`、`init()`
  - ⚠️ `mgrAdd('brand')` **不**调用 `populateBrandFilters()`，仅调 `populatePlatformFilters()`

### 清除筛选
- `clearFilters()` — 重置 fb-search/fb-from/fb-to/fb-platform/fb-analyzed/fb-sort，同时 `_fbBrand=''` + `populateBrandFilters()` + `renderLibrary()`

### 批量管理
- 入口：收藏库 `#batchModeBtn`（stats区右侧），点击 → `enterBatchMode()`
- 状态：`batchMode: boolean` + `batchSelected: Set<id>`
- 操作栏：`#batchBar`（粉色玻璃条），含"已选N条" + "🗑 删除选中" + "取消"
- 卡片变化：添加 `.batch-mode` / `.batch-selected` class，左上角 `.cc-checkbox`，点击切换勾选
- 关键函数：`enterBatchMode()` / `exitBatchMode()` / `renderBatchBar()` / `toggleBatchSelect(id, event)` / `deleteBatchSelected()`
- 删除：`confirm()` 确认后批量 splice，`saveToStorage()`，`exitBatchMode()`，`renderAll()`

## 设置页

### AI 配置
- API Key：`<input id="s-apikey" type="password">` + 👁 切换可见 `toggleApiKeyVis()`
- 模型选择：`<select id="s-model">` — `deepseek-chat`（默认推荐）/ `deepseek-reasoner`
- 测试连接：`testApiKey()` — 向 `/v1/chat/completions` 发 `max_tokens:10` 的 "hi" 请求，结果显示在 `#apiTestResult`
- 保存：`saveSettings()` — oninput/onchange 实时写入 `db.settings.apiKey` / `db.settings.model`

### 本地文件绑定（File System Access API）
- `bindExistingFile()` — `showOpenFilePicker` → 合并导入 + 绑定句柄
- `createNewFile()` — `showSaveFilePicker` → 写入当前数据 + 绑定句柄
- `syncToFile()` — 每次 `saveToStorage()` 后自动调用，向绑定文件写入 JSON
- `updateFileStatus()` — 更新 Header 绿色「已绑定」状态条和设置页标签
- 仅支持 Chrome / Edge（用 `'showOpenFilePicker' in window` 检测）

### 关于 Peekaboo
- 设置页最底部有一张中英双语描述卡，介绍工具定位

## 深色模式
- 切换：`toggleTheme()` → 写 `document.documentElement` 的 `data-theme` 属性 → 存 `db.settings.theme`
- CSS：`[data-theme="dark"]` 覆盖 `:root` 中的表面色变量（`--bg`、`--bg-card`、`--border`、`--text`、`--shadow` 等）
- 按钮：Header 右侧 `🌙` / `☀️`，id=`themeBtn`
- init 时读取 `db.settings.theme` 恢复状态

## AI 分析

### Vision 图片发送
- 内容图最多取前3张 + 互动图最多取前2张，拼成 `image_url` blocks
- 任一图存在则先带图请求；遇 400/422 降级纯文本重试

### buildPrompt 规则
- **角色设定**：4条基础 contextLines，第5条【重要规则】禁止假设/推测/捏造任何互动数字
- **小红书条件**：`isXiaohongshu = entry.platform.includes('小红书')` 为真时，追加第6条【小红书平台特别说明】（18-25岁年轻女性成年玩家，非儿童/家长框架）
- **无互动数据时**：`engagementGuide` 要求模型明确说明"未提供互动数据，无法进行数据分析"，禁止估算
- **小红书专项**：`isXiaohongshu` 为真时，JSON schema 插入 `xiaohongshuField`（5维：溯源/趋势/圈层/圈内文化/热点判断）

### analysis 对象字段
| 字段 | 说明 | 字数要求 |
|---|---|---|
| `contentType` | 内容形式 | 简短具体 |
| `culturalContext` | 🌍 文化背景解读（①②③④） | — |
| `trendSignal` | 📈 流行趋势判断（①②③） | — |
| `commercialPurpose` | 💼 商业目的深挖（string；旧版为 string[]，兼容展示） | 80-100字 |
| `visualStrategy` | 🎨 视觉策略解码（①②③④⑤，⑤为美术向深度解析） | **150-180字** |
| `emotionalTone` | 情感联结（一句话） | — |
| `targetAudience` | 目标受众 | — |
| `engagementInsight` | 💬 传播力洞察（无数据时说明无法分析） | — |
| `xiaohongshuInsight` | 📕 小红书专项（仅平台含"小红书"时生成） | — |
| `tags` | string[]，3-5个关键词 | — |
| `strategicInsight` | 💡 核心战略洞察（旧字段名 `insight`） | 100-120字 |
| `ahaRecommendations` | 🎯 AHA World 启示（旧字段名 `recommendations`，①②③） | 120-150字 |
| `marketingIntensity` | number 1-5 | — |
| `marketingIntensityReason` | 评分依据一句话 | — |

### renderAnalysisHTML 渲染规则
- 所有长文本字段用 `formatAnalysisText(s)` 处理，不再用 `escHtml`
- `formatAnalysisText`：先 `escHtml` → 再把 `**text**` 转 `<strong>`（用 `[\s\S]*?` 支持跨行） → 再在 ②③④⑤ 前插入 `<br>`
- `xiaohongshuInsight` 渲染：红色系专属样式 `rgba(255,50,80,0.07)` 背景，📕 图标

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
/* dark mode 覆盖：--bg:#111110; --bg-card:rgba(28,28,26,0.82); --border:#2E2E2A; */
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
- `.fb-bp`（品牌筛选pill）：`rgba(255,255,255,0.65)` + active 态粉色系

### 批量管理样式
- `.batch-bar`：粉色系玻璃条，`display:none` / `.open` 时 `display:flex`
- `.cc-checkbox`：卡片左上角 22px 圆角方框
- `.content-card.batch-selected`：粉色边框描边 + 顶边高亮 + 阴影

### AI 分析框
| 类名 | 色系 |
|---|---|
| `.cultural-box` | 蓝色系 |
| `.trend-box` | 粉色系 |
| `.engagement-box` | 橙色系 |
| `.insight-box` | 黄色系 |
| `.recommendation-box` | 绿色系 |
| 小红书专项（inline style） | 红色系 `rgba(255,50,80,…)` |

## Logo 文件（competitor-tracker 目录）
- `logo-color.png`（header 使用）
- `logo-black.png` / `logo-white.png` / `logo-stacked.png` / `logo-black-color.png`

## 注意事项
- 修改后用浏览器直接打开 index.html 验证，不要依赖 Claude Code 预览
- PowerShell 启动前需设置代理：`$env:HTTPS_PROXY="http://127.0.0.1:7897"`
- 平台列表固定，不要改为可编辑
- `imageBase64` / `engagementImageBase64` 只作为向后兼容副本，真正的数据在数组字段
- 品牌pill onclick 必须用 `data-brand` + `this.dataset.brand`，不能用 `JSON.stringify` 内联（双引号破坏 HTML 属性）
- `formatAnalysisText` 的 bold 正则必须用 `[\s\S]*?` 而非 `.+?`（`.` 不匹配换行）
