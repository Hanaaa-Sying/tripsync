# CLAUDE.md — 一拍迹合（TripSync）项目规范

> 这是给 Claude Code 看的项目说明 + 固定工作流。每次开工前先读本文件。

---

## 一、项目背景

**项目名**：一拍迹合 · AI 个人向旅行规划（原名「旅搭子」）
**性质**：抖音 AI 创变者黑客松·广州站 2026 参赛作品
**GitHub**：https://github.com/Hanaaa-Sying/tripsync.git
**一句话**：把抖音旅行攻略视频，变成一条贴合用户旅行人格、还能和搭子同步的**可执行行程**——省去「收集 → 筛选 → 记录 → 整合」的繁琐工作流。

### 目标人群
出行前会上抖音精选刷攻略视频做策划的用户。

### 解决的痛点
- 视频要逐条看、记地名、查坐标，极其麻烦；
- 通用攻略千人一面，网红路线未必合用户口味和节奏；
- 多人出行各存各的，行程难对齐。

### 端到端流程
1. **旅行人格测试**：8 道题（排序 / 滑块 / 单选），四维度模型（记录↔沉浸、精致↔本地、计划↔灵感、覆盖↔深度）→ 映射到 6 种旅行人格。
2. **视频解析**：粘贴抖音链接 → Agent Reach 调 douyin-mcp 无水印下载 → ffmpeg 抽音频 → 小米 MiMo ASR 转写 → LLM 提取景点（名称 / 类型 / 坐标），多视频并行。
3. **联网补全**：豆包（火山方舟）Responses API 的 `web_search`，按「城市 + 天数」补高频打卡地做兜底。
4. **人格化筛选**：LLM 结合人格画像选点并给推荐理由，地图高亮。
5. **搭子同步**：输入伙伴 UID，共享并对齐双方人格与计划。
6. **导出行程**。

### 6 种旅行人格（配方见 `backend/data/mbti_data.py` + `tests/test_mbti_recipes.py`）
摄影师 DRPT · 特种兵 DRPC · 生活家 ILST · 架构师 IRPT · 悠享家 IRSC · 美食家 ILSC

### 技术栈
- **后端**：Flask + SQLite（入口 `app.py`，路由 `backend/routes/`，业务 `backend/services/`，数据 `backend/data/`）
- **前端**：原生 HTML / CSS / JS（`frontend/`）
- **LLM**：OpenAI 兼容接口，默认豆包 / 火山方舟 Responses API（`backend/config.py` 的 `DOUBAO_MODEL`，pro 版联网更稳）
- **互联网接入**：Agent Reach（`agent-reach/`）—— douyin-mcp（视频解析）/ xhs-cli（小红书）/ Exa（搜索），配置见 `config/mcporter.json`
- **当前开发分支**：`main`（个人 solo 仓库，单人维护）

### 跑起来
```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env          # 填 LLM API Key、MIMO_API_KEY
bash agent-reach/setup.sh     # 安装抖音/小红书/搜索工具
bash run.sh                   # 或 python3 app.py → http://localhost:5000
```

---

## 二、固定工作流（每次改动后按顺序执行，不得跳过）

1. **改代码 / 改内容**。

2. **记录 `debug.md`**：本次若**遇到并解决了 Bug、或发现了一条新规则 / 隐患**，按 `debug.md` 现有格式追加或更新对应条目（写清：Bug 现象 → 根因 → 修复状态 → 在哪个文件改的 → 是否仍需真机验证）。纯功能新增不进 debug.md。

3. **记录 `log.md`**：把本次做了什么追加进 `log.md`（这是「目前已经做了什么」的总账，格式见该文件抬头）。**每次改动都要写**，无论是 Bug 修复还是功能新增。

4. **询问是否推送**：改完、记完，**主动询问用户是否可以推送到 GitHub**（`origin` = `https://github.com/Hanaaa-Sying/tripsync.git`）。**得到明确确认后**才执行：
   ```bash
   git add -A
   git commit -m "简要描述本次修改"
   git push origin main
   ```
   > `origin` 已指向个人仓库 `Hanaaa-Sying/tripsync`，主分支为 `main`，直接推即可。
   > ⚠️ 切勿提交 `.env` / `config/mcporter.json`（含真实密钥，已在 `.gitignore` 中屏蔽）。

---

## 三、参考文档

- **[`debug.md`](debug.md)** —— 调试与 Bug 修复记录。开工前先扫一遍「核对结论速览」表，避免重复踩坑（重点：人格匹配、海报截图、视频景点坐标补全、豆包 web_search、city 写死上海等已知项）。
- **[`log.md`](log.md)** —— 已做事项总账，记录每次改动「做了什么 + 在哪改的」。
- **[`README.md`](README.md)** —— 面向外部的项目说明与 Agent Reach 接入文档。
- **[`项目介绍.md`](项目介绍.md)** —— 产品视角的完整介绍（痛点 / 用户价值 / 创新点 / 延展潜力）。
