# debug.md — 调试与 Bug 修复记录

> 本文档由原先的 `log.md` / `log(1).md` / `log(4).md` 三份改动日志**合并整理**而成。
> 三份原始日志混杂了「功能新增」和「Bug 修复」两类内容，本文档**只抽取并核对其中真正的 Bug**，
> 逐条标注当前 `main` 分支代码里的**实际状态**（是否已修复、在哪修的）。
> 功能性改动（品牌改名、问卷交互、海报模板、搭子同步等）不在本文档范围内，如需查阅请看 git 历史。
>
> 最近一次核对：2026-06-09。核对方式：直接读当前代码 + `py_compile` / `node --check` / 跑 MBTI 配方测试。

---

## 核对结论速览

| # | Bug | 来源日志 | 当前状态 |
|---|-----|---------|---------|
| 1 | 旅行人格结果**恒为「漫步客」** | log.md 四·补 | ✅ 已修复（且被 6 人格最近原型匹配重写取代） |
| 2 | 图文海报功能完全不可用（html2canvas 从未加载） | log(4).md | ✅ 已修复 |
| 3 | 视频分析后地图页**「已提取 0 个景点」** | log(4).md | ⚠️ **本次补修**（原修复未落到此前的团队分支） |
| 4 | 表头答题进度轴偏左 | log.md 三·补2 | ✅ 已修复 |
| 5 | 老用户从行程页返回人格页**一片空白** | log.md 三·补 A | ✅ 已修复 |
| 6 | 豆包联网搜索 `web_search` 报 404 | log(1).md | ✅ 已修复（改用 pro 模型 + 原生 tools 参数） |
| 7 | 视频景点入库 `city` 写死「上海」 | log(4).md | 🟡 已知遗留（原作者主动暂缓，非上海城市才触发） |

---

## 1. 旅行人格结果恒为「漫步客」 ✅ 已修复

**Bug**：旧 `calculate_mbti()` 算出 4 字母代码（如 `DRPC`）后，用 `if mbti not in PERSONALITY_TYPES` 兜底；
但 `PERSONALITY_TYPES` 的键是**人格名**（`wanderer`…）而非代码，该条件**恒为真** → 永远 `return "IRST"`
→ 映射到漫步客。无论怎么答都是漫步客。

**修复历程**：
- log.md「四·补」先把兜底判断从 `PERSONALITY_TYPES` 改为 `MBTI_TO_PERSONA`（16 代码为键），让结果随答案变化。
- log(4).md 进一步把整个 `calculate_mbti()` **重写**为「6 人格最近原型匹配」：答案 → 连续维度向量 →
  与 6 条标准配方算欧氏距离 → 返回最近人格。彻底消除维度耦合导致的撞车。

**当前代码**：`backend/data/mbti_data.py:487` `calculate_mbti()` → `return match_persona_by_prototype(answers)`。
6 条配方（摄影师 DRPT / 特种兵 DRPC / 生活家 ILST / 架构师 IRPT / 悠享家 IRSC / 美食家 ILSC）
均有自动化测试 `tests/test_mbti_recipes.py`，本次核对**全部通过**。

---

## 2. 图文海报功能完全不可用 ✅ 已修复

**Bug**：导出分享页（Step 8）点「图文海报」走 `exportAsImage()` → 调 `html2canvas` 把海报 DOM 截成 PNG，
但 `index.html` 从未引入该库，导致永远走降级路径（下载 HTML 而非图片）。

**修复**（log(4).md）：
- `frontend/index.html` 引入 html2canvas CDN（当前代码 `index.html:15-16` ✓）；
- 重写 `exportAsImage()` 为 async/await + loading 态 + 字体就绪等待 + 错误降级（`app.js:2288` ✓）；
- 路线缩略图改为**本地 DOM/SVG 自绘**（`buildPosterRouteMap()`），零外部请求、零跨域，
  规避了高德静态图跨域 taint canvas 导致 `toDataURL` 抛 `SecurityError` 的隐患；
- 一并删除了无后端支撑的**假分享链接**（`generateShareLink/copyShareLink`）。

**仍需真机验证**：emoji/中文/圆点定位的截图效果需本地 `bash run.sh` → 走到 Step 8 实测。

---

## 3. 视频分析后地图页「已提取 0 个景点」 ⚠️ 本次补修

**Bug**：视频转写成功、后端日志显示「提取 N 个、写库 M 个」，但 Step 5 地图页仍显示「暂无景点数据」，
提示条写「已从视频中提取 0 个景点」。

**根因**：
1. 地图页只保留**有 `lat/lng`** 的景点（前端会按坐标过滤）；
2. `/api/video/analyze` 返回给前端的是 `unique_locations` **原始结果**，
   而坐标补全 `_fill_coords()` 此前**只在 `_save_locations_to_db()` 写库时执行**，
   补全后的坐标从未回写到接口响应里；
3. 于是接口返回的景点 `lat/lng` 为 `null` → 被前端全部过滤 → 「0 个」。

**修复状态**：log(4).md 记录过此修复，但**该修复并未落到此前的团队分支**——
核对时发现 `backend/routes/video.py` 的 `analyze()` 仍直接返回未补坐标的 `unique_locations`。
**本次（2026-06-09）已补上**：在 `analyze()` 去重之后、写库与 `return` 之前，加入坐标/类型补全循环：

```python
# 坐标/类型补全：在返回前就把 lat/lng 补齐，否则前端地图页会因坐标为空把景点全部过滤掉
for loc in unique_locations:
    lat, lng = _fill_coords((loc.get("name") or "").strip(), loc.get("lat"), loc.get("lng"))
    loc["lat"], loc["lng"] = lat, lng
    raw_type = (loc.get("type") or "").strip().lower()
    loc["type"] = raw_type if raw_type in ("landmark", "street", "food", "culture") \
        else _infer_type_from_keywords(loc.get("keywords", []))
```

这样接口响应与写库用的是**同一份已补全数据**，地图页不再被过滤成 0。
`_save_locations_to_db()` 内部仍调 `_fill_coords()`，但对已补全坐标无副作用（非空时原样返回），两条路径一致。
`python -m py_compile backend/routes/video.py` 通过。

**仍需真机验证**：需 agent-reach/mcporter 语音转写 + LLM 提取跑通真实抖音链接，确认地图正常打点、数量一致。

---

## 4. 表头答题进度轴偏左 ✅ 已修复

**Bug**：表头原为 flex 布局，`.step-nav` 受 `flex:1 + max-width:320px` 限制、右侧用 `margin-left:auto`，
导致进度轴整体偏左，不居中。

**修复**（log.md 三·补2）：`.header-content` 改为**三列网格** `grid-template-columns: 1fr auto 1fr`，
`.step-nav` 去掉 `flex:1` 改 `justify-self: center` + 固定宽度；768px 媒体查询显式回到 flex 配合 order 换行。

**当前代码**：`frontend/css/style.css:134` grid 三列 ✓，`:165` `justify-self: center` ✓。

---

## 5. 老用户从行程页返回人格页一片空白 ✅ 已修复

**Bug**：老用户登录后被直接带到行程规划页，旅行人格页（`#page-mbti`）可能从未渲染过；
点「← 返回上一步」回到人格页时 DOM 为空 → 一片空白。

**修复**（log.md 三·补 A）：新增 `goToPreviousStep()` + `ensureMbtiPageContent()`——
返回人格页前检查 DOM，若为空则：已测过就 `renderMBTIResult()` 展示结果，没测过就 `resetMBTIPage()` 重新出题。

**当前代码**：`frontend/js/app.js:198` `goToPreviousStep()` ✓、`:206` `ensureMbtiPageContent()` ✓、
`:845` `backToMbtiQuestions()`（结果页返回答题页同理）✓。

---

## 6. 豆包联网搜索 web_search 报 404 ✅ 已修复

**Bug**：`discover_trip_attractions()` 用豆包 Responses API 带 `"tools": [{"type": "web_search"}]` 联网搜索，
早期报 404。

**根因 / 排查结论**（log(1).md）：`web_search` 是 Responses API 的**原生工具参数**，
与火山方舟控制台「插件市场」/Bot 级插件是**两套不同系统**，开通插件市场并不能让原生 `tools` 生效。
需所选模型本身在 Responses API 层面支持 `web_search` tool type，且 **pro 版联网搜索更稳定**。

**当前代码**：`backend/config.py:55` `DOUBAO_MODEL` 默认值已是 `doubao-seed-2-0-pro-260215`（pro 版），
调用走 `{DOUBAO_BASE_URL}/responses` + 原生 `tools` 参数。
若联网搜索失败会降级到静态默认景点列表（`DEFAULT_RECOMMEND_ATTRACTIONS`），不会整体崩。

---

## 7. 视频景点入库 city 写死「上海」 🟡 已知遗留（暂缓）

**隐患**：`/api/locations/list` 按 `city` 查询，但视频景点入库时 `city` 被**写死成「上海」**
（`backend/routes/video.py` 的 `_save_locations_to_db()`，当前约 944 行）。
若行程城市非上海，走**回库读取**这条降级路径会查不到。

**为何暂缓**（log(4).md 原作者决定，本次维持）：
1. 正常路径走前端内存 `analyzedLocations`，不经过 city 查询，地图页已可正常显示；
2. 真修需把行程实际城市从前端/会话透传到 `analyze` 与 `_save_locations_to_db`，改动面较大；
3. 兜底坐标表 `_SHANGHAI_COORD_FALLBACK` 目前**只有上海数据**，光改 city 字段、非上海城市坐标
   仍会落到上海市中心，意义有限。

**后续若要支持多城市**：需同时补 ① city 字段透传 ② 多城市兜底坐标表，缺一不可。
当前产品定位为上海玩法，暂不影响主流程。

---

## 附：其它历史改动（非 Bug，仅备查）

三份原始日志还包含大量**功能性改动**，已合并进代码、不属于 Bug，简列于此便于追溯：

- **品牌改名**：旅搭子 → 一拍迹合，✈️ → 🗺️（标题/导航/登录注册/聊天面板）。
- **问卷交互**：排序题点击 → **拖拽排序**；滑块题加 5 节点 + 灰色占位「拖动圆点作答」区分是否作答；
  删除自动提交改为底部「查看结果」按钮 + 漏答滚动定位高亮。
- **行程规划页**：天数/预算改**数字输入框**；删除「同行人」选项，改由「是否有搭子」推导；
  生成行程后展示**人均花费估算**与预算对比徽章。
- **后端成本估算**：`shanghai_locations.py` 补 `avg_cost`；`itinerary.py` 新增 `_estimate_cost()`
  与同行人/预算加权；`llm_service.py` 提示词加入人均消费与预算要求。
- **旅行搭子闭环**：「添加搭子」→ 后续步骤贯穿展示搭子头像 → 导出页「同步计划给搭子」
  （真接口 `POST /api/buddy/share-plan`，写入搭子 `travel_history`）。
- **深度画像（选做）**：结果页给一段结构化 prompt 让用户复制给常用 AI，粘回 JSON 存入
  `state.mbtiResult.deep_profile`，参与推荐加权与 AI 提示词（复用 `PUT /api/auth/mbti` 持久化，无需新表）。
- **每步骤「返回上一步」按钮**（step 2~7）、结果页标题随答题/结果态切换。
