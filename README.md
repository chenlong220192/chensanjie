# 陈三姐病历管理

> 胰腺癌患者的系统性病历管理与可视化报告项目。以**原始单据为输入源、Excel 为唯一数据录入端**，通过脚本流水线生成**单文件离线 HTML 报告**。

## 重要声明（隐私）

本项目包含**完整医疗隐私数据**（身份证号、医保卡号、详细住址、化验数值、费用明细等），**仅存于本私有仓库 / 本地目录**。

- 切勿直接推送到任何公开仓库。
- 如需对外展示，须走 **发布清单 `5、静态页面/deploy/publish.manifest.json` + `scripts/publish/pub1_generate_out.py` + `scripts/publish/pub2_mask_pii.py`** 的脱敏发布流程（见「公开同步」一节）。旧同步配置 `sync-config.json` 已于 2026-09-10 删除。

---

## 一、项目定位

| 维度 | 说明 |
|---|---|
| 用途 | 患者家庭端的「病历 + 指标 + 费用」追踪与可视化 |
| 数据性质 | 100% 本地 / 私有仓库，含敏感医疗信息 |
| 核心产物 | `5、静态页面/index.html`——单文件离线 HTML 报告（6 个 tab，纯 SVG 图表，零第三方依赖） |
| 数据源 | 原始 PDF/TIFF 单据（只读）+ Excel 主数据（SSOT） |

---

## 二、目录结构总览

```
陈三姐病历/
├── 1、病历/          原始单据（PDF/TIFF/扫描件）· 只读输入源（自检 trigger 扫描对象）
├── 2、医保+门特/      大病登记回执等政策文件
├── 3、外部资料/       只读：CSCO 2024 指南 / NCCN 胰腺癌 2026.V3（2026-09-10 新增）/ 术后饮食指导
├── 4、信息记录/       ★ 全部 8 个 Excel（唯一数值 SSOT）：关键指标监测.xlsx、费用记录.xlsx、
│                        用药记录.xlsx、化疗药品矩阵.xlsx、血压体温每日监测.xlsx、用药打卡记录.xlsx + 打印模板
├── 5、静态页面/       ★ 前端工程：index.html + data/ + assets/ + views/ + deploy/（详见下文）
├── 6、暂存/           ★ 前端成品迭代工作区，按任务分子目录，迭代稳定后回迁 5、静态页面/
├── scripts/          流程脚本（phase*/release*/check*/tool*/skill*/dev_*）
├── diagrams/         架构图 SVG
├── AGENTS.md         项目规则（数据所有权边界、高危操作清单）★ 开发前必读
└── .gitignore        隐私数据 + 备份 + macOS 元数据忽略规则
```

### 5、静态页面/ 详解（核心产品代码）

| 子目录 / 文件 | 作用 |
|---|---|
| `index.html` | **主报告**（pipe2 产出，GH Pages 标准入口名，6 个 tab）|
| `data/` | 数据中间产物（含敏感数值）：indicators / urine / fees / meds / chemo_drugs .json（构建产物，禁手改）|
| `assets/` | **唯一静态资源目录**：`style.css` + `app.js` + `images/长海医院门诊检查流程照片/`（10 张 JPG）|
| `views/` | **叙述型 tab 固化模板**（timeline/chemotherapy/diet/notes 4 个 .html；叙述内容直接改这里）|
| `deploy/baseline/` | 验收安全网：`deploy/baseline.json`（数据基线）+ `fingerprint.json`（HTML 结构指纹）|
| `deploy/publish.manifest.json` | **发布清单 SSOT**：增删要发布的页面只改这里（当前启用 index.html + assets 两项）|
> 注：构建流水线脚本均在项目根 `scripts/`（不在前端工程内）；旧 `src/assets/`、`静态文件/`、`public/`、`sync-config.json` 已于 2026-09-10 清理。

---

## 三、数据流转（核心）

数据**单向**流动：`原始单据 / Excel → JSON → HTML`，叙述型内容则固化在 `views/`（一次性从旧版 HTML 迁移，内容不改）。

![数据流转架构](diagrams/%E6%95%B0%E6%8D%AE%E6%B5%81%E8%BD%AC%E6%9E%B6%E6%9E%84.svg)

**各环节职责：**

| 阶段 | 脚本 / 目录 | 职责 |
|---|---|---|
| ① 录入 | `4、信息记录/` 的 8 个 Excel | 唯一数值录入端（指标、费用、用药、化疗、血压体温）|
| ② 抽取 | `scripts/pipeline/pipe1_extract_data.py` | openpyxl 读 Excel → 规范化数值 → 生成 5 个 JSON 到 `data/` |
| ③ 数据层 | `5、静态页面/data/*.json` | 结构化中间产物（构建产物，禁手改）|
| ④ 构建 | `scripts/pipeline/pipe2_render_html.py` | JSON 驱动数据型 tab；读 `views/` 叙述模板 → 内联 `APP_DATA` 成单文件 |
| ⑤ 产物 | `index.html` | 主报告（GH Pages 入口）|
| 🔒 验收 | `pipe3_check_data_baseline.py` / `pipe4_check_narrative_assets.py` / `pipe0_snapshot_fingerprint.py` | 数据基线比对 / 叙述模板+JSON 对照产物（标题·金额·正文零丢失）/ 结构指纹 |

> 一键调用链：`bash scripts/pipeline/pipe_run_rebuild.sh`（pipe1 → pipe2 → pipe4 → pipe3 --write → pipe_check_integrity 三档自检）。

---

## 四、构建与维护流程

### 更新数据（每次检查 / 化疗 / 费用后）

```bash
# 0. 先把化验/费用录入 4、信息记录 / 4、信息记录 的 Excel（唯一录入端）
# 快捷方式：一条命令跑完 1-3 步（任一步失败即停）
bash "scripts/pipeline/pipe_run_rebuild.sh"

# 或分步执行：
# 1. 抽取 Excel → JSON
python scripts/pipeline/pipe1_extract_data.py
# 2. 构建 HTML（自动数据对账）
python scripts/pipeline/pipe2_render_html.py
# 3. 验收（标题/金额零丢失、结构完整）
python scripts/pipeline/pipe4_check_narrative_assets.py
```

### 本地预览

```bash
cd "5、静态页面"
python -m http.server 8088 --bind 127.0.0.1
# 浏览器访问 http://127.0.0.1:8088/index.html
```

### 修改任何东西前必读

1. `AGENTS.md`（数据所有权边界 + 高危操作清单——涉及 `1、病历` 只读区与 Excel SSOT 的改动须谨慎）
2. `5、静态页面/README.md`（子项目细目）
3. `6、暂存/REFACTOR_SOP-归档-2026-09-06.md`（重构 SOP 完整备份，Phase 3/4 未完）

---

## 五、公开同步（隐私脱敏链路）

> 让公开项目 `chenlong220192/chensanjie`（GitHub Pages）看起来像一个正常前端项目——**只同步去隐私后的静态 HTML**。

![公开同步脱敏链路](diagrams/%E5%85%AC%E5%BC%80%E5%90%8C%E6%AD%A5%E8%84%B1%E6%95%8F%E9%93%BE%E8%B7%AF.svg)

- 发布清单（SSOT）：`5、静态页面/deploy/publish.manifest.json`（增删发布页面只改清单，不改脚本）
- 流程说明：`5、静态页面/SYNC_GUIDE.md`
- 当前状态：**未启用**（`public_project.root` 为占位符，待提供公开仓库本地路径）

**切勿**绕过该配置直接推送整个仓库。

---

## 六、前端项目化进度

| Phase | 状态 | 内容 |
|---|---|---|
| A | ✅ | CSS/JS 拆出 HTML（`index.html` + `assets/`，GH Pages 标准入口名）|
| B | ✅ | 补齐 `pipe_run_rebuild.sh` 一键构建 / `dev_serve_preview.py` 预览服务器 / head meta+favicon（2026-09-06，按「不破坏双击直开」原则，不引入运行时 fetch）|
| C | ⏳ | 脱敏脚本（公开仓库隐私过滤）|
| D | ⏳ | 推送公开仓库（chensanjie）|

---

## 七、医疗动态速览（非技术）

近期关键节点见 `AGENTS.md`「待完成事项」区——当前关键决策点：**09-07 华山医院·胰腺外科 专家门诊**（根据 09-03 增强 CT 结果判断下一步治疗方案：手术 vs 继续化疗）。数据更新以实际检查结果为准。

*README 由整体 review 产出 · 图示源文件存于根目录 `diagrams/` · 目录与发布链路说明于 2026-09-10 同步更新（6 目录体系 + 发布清单驱动）*
