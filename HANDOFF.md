# OmniSearch 项目交接说明

> 给下一台电脑的 AI Agent 看：这是一份完整的项目交接文档。读完这份文档你就能无缝接手继续开发。

---

## 0. 一句话概括

**OmniSearch** 是一个纯前端的**聚合搜索 PWA**，支持多搜索引擎切换、拍照识图搜同款、以及用多模态 AI 对图片提问（识别品牌型号 + 推荐香港购买渠道）。已部署上线，手机/电脑都能用。

- **线上地址**：https://yj438863687.github.io/omni-search/
- **GitHub 仓库**：https://github.com/yj438863687/omni-search
- **技术栈**：纯 HTML + CSS + 原生 JS，**零依赖、零构建、无后端**
- **部署方式**：GitHub Pages（`main` 分支根目录自动部署）

---

## 1. 文件结构

```
omni-search/
├── index.html      # 全部代码都在这一个文件里（HTML+CSS+JS，约 924 行）
├── sw.js           # Service Worker，离线缓存
├── manifest.json   # PWA 清单（图标、名称、主题色）
├── icon.svg        # 矢量图标（源文件）
├── icon-192.png    # PWA 图标 192
├── icon-512.png    # PWA 图标 512
├── .env            # ⚠️ 本地密钥存档（已被 .gitignore 忽略，不会上传）
├── .gitignore      # 忽略 .env
└── HANDOFF.md      # 本文件
```

**重要**：整个应用逻辑集中在 `index.html` 的 `<script>` 标签内。修改时注意文件较大，用 grep 定位函数再改，不要整文件重写。

---

## 2. 功能清单（当前已完成）

### 2.1 搜索核心
- 8 个搜索引擎一键切换：百度、必应、Google、DuckDuckGo、知乎、B站、GitHub、淘宝
- 实时搜索建议（百度 JSONP 接口，失败自动降级为本地历史匹配）
- 聚合模式：勾选后同时打开百度+必应+Google 三家结果
- 搜索历史：只存本地 `localStorage`（最近12条），可清空
- 键盘快捷键：`Tab` 切换引擎、`↑↓` 选建议、`Enter` 搜索

### 2.2 拍照识图（搜同款）
- **手机端**：调用系统相机（`<input capture="environment">`）
- **电脑端**：`getUserMedia` 实时取景 + 快门按钮 + 空格键快门 + 前后摄像头切换
- 照片自动压缩到最长边 1000px 的 JPEG
- 4 个识图引擎提交：
  - **Google Lens**：multipart 文件 POST 到 `lens.google.com/v3/upload`
  - **必应识图**：base64 表单 POST 到 `bing.com/images/search?...sbiupload`
  - **Yandex**：multipart 文件 POST 到 `yandex.com/images/search?rpt=imageview`
  - **百度识图**：不支持跨站直传，只能打开其上传页让用户手动操作

### 2.3 AI 图片问答
- 拍完照可对图片提问，支持多轮对话
- 使用**智谱 GLM-4V-Flash**（免费视觉模型），**Key 已内置在代码中**（见下方第 4 节）
- 系统提示词已针对"识别品牌型号 + 香港购买渠道"优化
- AI 回答末尾输出 `PRODUCTS: 品牌 型号` 结构化标记，App 解析后生成：
  - **4 张同款商品缩略图**（来自必应图片 `tse2.mm.bing.net/th?q=...`，直接内嵌显示）
  - 购买按钮：Google购物（香港区）、HKTVmall、Farfetch、淘宝
- 兜底机制：AI 忘记输出标记时，从正文正则抓取 50+ 常见品牌名
- 快捷问题按钮：这是什么品牌型号 / 给我看同款图片 / 香港哪里买 / 上面文字写了什么 / 相似替代款

### 2.4 PWA
- Service Worker 缓存壳资源，可离线打开
- 支持"添加到主屏幕"，全屏运行如原生 App
- 深色/浅色主题（跟随系统 + 手动切换）

---

## 3. 关键代码位置速查（index.html）

| 功能 | 位置 |
|------|------|
| 搜索引擎配置 `ENGINES` 数组 | 约 410 行 |
| 本地存储 key 定义 `LS` | 约 420 行 |
| 搜索执行 `doSearch()` | 约 443 行 |
| 历史记录 `readLocalHistory/saveHistory/renderHistory` | 约 464 行起 |
| 搜索建议 `fetchSuggest()` | 约 500 行起 |
| AI 预设 `PRESETS` / 内置 Key `DEFAULT_AI_CFG` | 约 585 行 |
| AI 系统提示词（brand/香港提示词） | 约 614 行 |
| AI 发送 `sendAsk()` | 约 593 行 |
| AI 回答渲染 `renderAiAnswer()`（含缩略图+购买按钮） | 约 660 行 |
| 摄像头逻辑 `openCamera/startStream/camShutter` | 约 780 行起 |
| 图片压缩 `handleImage()` | 约 810 行 |

**注意**：行号会随改动偏移，请用 grep 定位函数名。

---

## 4. API Key 说明（重要）

### 当前状态
- **智谱 GLM-4V-Flash 的 API Key 已硬编码在 `index.html` 里**（`DEFAULT_AI_CFG` 常量），用户打开即用，无需配置。
- Key 值也存档在本地 `.env` 文件的 `ZHIPU_API_KEY=` 一行（**该文件已被 .gitignore 忽略，不在 GitHub 上**）。

### 风险与后续建议
- ⚠️ 因为代码是公开的，Key 理论上可被他人从源码中提取。GLM-4V-Flash 是**免费模型**，被人用不产生费用，风险低。
- **如果将来要换成付费模型**，务必改为 **Cloudflare Worker 中转**：把 Key 藏在 Worker 环境变量里，前端只调用 Worker 地址。这是下一步该做的架构升级。
- 用户也可在 App 内 ⚙️AI设置 里填自己的 Key（存 localStorage，覆盖内置的）。

### 关于 DeepSeek
- 用户有 DeepSeek Key，但**DeepSeek 官方 API 没有视觉模型**（只有 `deepseek-chat` 和 `deepseek-reasoner`，均纯文本），所以不能直接用来识图。
- 如果要用上，可做"两段式"：免费视觉模型描述图片 → DeepSeek 深度分析。**目前未实现**，属于可选方向。

---

## 5. 部署方式

### 当前：GitHub Pages
- 仓库 `main` 分支根目录即发布源
- 推送后约 1-2 分钟自动更新

### 更新流程
```bash
cd omni-search
# 修改代码...
git add . && git commit -m "描述改动" && git push
# 等待 1-2 分钟，访问 https://yj438863687.github.io/omni-search/ 验证
```

### 本地预览
```bash
cd omni-search
python3 -m http.server 8899
# 浏览器打开 http://localhost:8899
```

### 手机访问（局域网调试）
```bash
ipconfig getifaddr en0   # 查 Mac 的局域网 IP
# 手机同 WiFi 打开 http://<IP>:8899
```

---

## 6. 修改时的注意事项（踩过的坑）

1. **Service Worker 缓存**：每次发布新版本，**必须**把 `sw.js` 里的 `const CACHE = 'omnisearch-vN'` 版本号 +1，否则用户拿到旧缓存。
2. **强制刷新**：验证线上更新时，用带时间戳的 URL 绕过缓存，如 `...?v=$(date +%s)`。
3. **`renderAiAnswer` 的 `:has()` 选择器**：`.msg.ai:has(.prod-box)` 用于让带产品卡片的回答气泡变宽，现代浏览器支持，旧浏览器会降级（不影响功能）。
4. **必应缩略图接口**：`https://tse2.mm.bing.net/th?q=<关键词>&w=220&h=220&c=7&rs=1&p=0&pid=1.7&mkt=zh-HK`，返回真实商品图，`p` 参数 0-3 取不同图。加了 `onerror` 隐藏加载失败的图。
5. **禁止 AI 编造链接**：提示词里明确要求 AI 不要输出网址（实测小模型会编造假链接），只输出品牌型号，由 App 生成真实链接。
6. **手机/电脑摄像头分流**：通过 UA + `navigator.maxTouchPoints` 判断设备类型，手机走系统相机，电脑走 getUserMedia。
7. **JS 语法自检**：改完可跑
   ```bash
   python3 -c "import re; html=open('index.html').read(); open('/tmp/c.js','w').write(re.search(r'<script>(.*?)</script>',html,re.S).group(1))" && node --check /tmp/c.js
   ```

---

## 7. 已知待办 / 可优化方向

- [ ] **Cloudflare Worker 中转 Key**（若要上付费模型，必须做）
- [ ] 支持电脑端 `Ctrl+V` 粘贴截图直接识图
- [ ] 识图历史记录（当前只存文字搜索历史）
- [ ] 接入淘宝"拍立淘"搜购物同款
- [ ] AI 回答里的关键词一键转为搜索
- [ ] 打包成安卓 APK（可用 PWABuilder：先上线→输入网址→自动生成 APK）
- [ ] 两段式：免费视觉模型描述 + DeepSeek 分析（用户有 DS Key）

---

## 8. 用户背景与偏好

- 用户是中文用户，在香港（涉及本地购买渠道推荐）
- 偏好**极简、直接**的沟通，喜欢马上能看到结果
- 项目驱动方式：一步步来，做完一步确认一步
- 之前的 Google 账号被停用，正在注册新 Gmail（与本项目无关，但注意 GitHub 账号是 `yj438863687`）

---

## 9. 快速上手（新电脑）

```bash
# 1. 克隆仓库
git clone https://github.com/yj438863687/omni-search.git
cd omni-search

# 2. 本地预览
python3 -m http.server 8899
# 打开 http://localhost:8899

# 3. 改代码（主要在 index.html）

# 4. 发布
git add . && git commit -m "改动说明" && git push
```

**注意**：`.env` 文件不在仓库里（被忽略），如果新电脑需要智谱 Key，从旧电脑拷 `.env`，或直接用 `index.html` 里已内置的 Key。

---

*文档生成时间：2026-09-19*
*最新提交：1472d65 强化本地历史记录隐私说明与读取容错*
