# 陈三姐病历管理

> 胰腺癌患者的系统性病历管理与可视化报告项目。以**原始单据为输入源、Excel 为唯一数据录入端**，通过脚本流水线生成**单文件离线 HTML 报告**。

## 重要声明（隐私）

本项目包含**完整医疗隐私数据**（身份证号、医保卡号、详细住址、化验数值、费用明细等），**仅存于本私有仓库 / 本地目录**。

- 切勿直接推送到任何公开仓库。
- 如需对外展示，须走 **发布清单 `5、静态页面/deploy/publish.manifest.json` + `scripts/publish/pub1_generate_out.py`（按清单生成 out/）+ `scripts/publish/pub2_mask_pii.py`（PII 脱敏校验，MASK=0 才放行）** 的脱敏发布链路（见「公开同步」一节）。旧同步配置 `sync-config.json` 已于 2026-09-10 删除。

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
├── 3、外部资料/       只读：CSCO 2024 指南 / NCCN 胰腺癌 2026.V3（2026-09-10 新增）/ 胰腺术后饮食
├── 4、信息记录/       ★ 全部 8 个 Excel（唯一数值 SSOT）：关键指标监测 / 费用记录 / 用药记录 /
│                        化疗药品矩阵 / 血压体温每日监测 / 用药打卡记录（后两者各含 A4 打印模板）
│                        └── .ocr/  OCR 工作区：中转/ 入私有仓（用户裁决：私有仓不脱敏）；.bin/、待处理/ 不入仓
├── 5、静态页面/       ★ 前端工程：index.html + data/ + assets/ + views/ + deploy/（详见下文）
├── 6、暂存/           ★ 前端成品迭代工作区，按任务分子目录；`归档/` 存「现阶段不迭代」的沉淀产物（登记簿见其 README.md）
├── scripts/          流程脚本（dev / gen / ocr / pipeline / publish / util，详见 `scripts/README.md`）
├── diagrams/         架构图 SVG（数据流转 / 公开同步脱敏链路）
├── AGENTS.md         项目规则（数据所有权边界、高危操作清单）★ 开发前必读
└── .gitignore        隐私数据 + 备份 + macOS 元数据忽略规则
```

> ⚠️ 旧目录名已作废（2026-09-10 六目录重构）：`8、静态页面`→`5、静态页面`；`9、暂存`→`6、暂存`；`1、病历记录`→`1、病历`；`2、用药记录`+`3、指标监测`+`4、费用记录`→`4、信息记录`；`6、指南`+`7、饮食`→`3、外部资料`。

### 5、静态页面/ 详解（核心产品代码）

| 子目录 / 文件 | 作用 |
|---|---|
| `index.html` | **主报告**（pipe2 产出，GH Pages 标准入口名，6 个 tab）|
| `data/` | 数据中间产物（含敏感数值）：indicators / urine / fees / meds / chemo_drugs .json（构建产物，禁手改；已 gitignore）|
| `assets/` | **唯一静态资源目录**：`style.css` + `app.js` + `images/长海医院门诊检查流程照片/`（10 张 JPG）|
| `views/` | **叙述型 tab 固化模板**（timeline / chemotherapy / diet / notes 4 个 .html；叙述内容直接改这里）|
| `deploy/` | 发布与构建状态：`publish.manifest.json`（**发布清单 SSOT**）+ `baseline/`（数值基线 + 结构指纹 + 构建 hash）+ `README.md` |

> 注：构建流水线脚本均在项目根 `scripts/pipeline/`（不在前端工程内）；旧 `src/assets/`、`静态文件/`、`public/`、`sync-config.json` 均已于 2026-09-10 清理。

---

## 三、数据流转（核心）

数据**单向**流动：`原始单据 / Excel → JSON → HTML`，叙述型内容则固化在 `views/`（一次性从旧版 HTML 迁移，内容不改）。

![数据流转架构](diagrams/%E6%95%B0%E6%8D%AE%E6%B5%81%E8%BD%AC%E6%9E%B6%E6%9E%84.svg)

**各环节职责：**

| 阶段 | 脚本 / 目录 | 职责 |
|---|---|---|
| ① 录入 | `4、信息记录/` 的 8 个 Excel | 唯一数值录入端（指标、费用、用药、化疗药品矩阵、血压体温）|
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
# 0. 先把化验/费用录入 4、信息记录 的 Excel（唯一录入端）
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

> 说明：`data/` 与 `deploy/baseline/` 均由脚本生成，**不要手改**；改数值一律改 Excel。

### 本地预览

```bash
cd "5、静态页面"
python -m http.server 8088 --bind 127.0.0.1
# 浏览器访问 http://127.0.0.1:8088/index.html
```

### 修改任何东西前必读

1. `AGENTS.md`（数据所有权边界 + 高危操作清单——涉及 `1、病历` 只读区与 Excel SSOT 的改动须谨慎）
2. `5、静态页面/README.md`（子项目细目：views / data / assets / deploy 各自职责与关键工作流）
3. `scripts/README.md`（脚本分区 + 逐脚本说明 + 依赖安装）
4. `6、暂存/归档/REFACTOR_SOP-归档-2026-09-06.md`（2026-09-06 前端重构 SOP 完整备份；**已归档、不再更新**，其 Phase 3/4 已由现行发布链路取代）

---

## 五、公开同步（隐私脱敏链路）

> 让公开项目 `chenlong220192/chensanjie`（GitHub Pages）看起来像一个正常前端项目——**只同步去隐私后的静态 HTML**。

![公开同步脱敏链路](diagrams/%E5%85%AC%E5%BC%80%E5%90%8C%E6%AD%A5%E8%84%B1%E6%95%8F%E9%93%BE%E8%B7%AF.svg)

- 发布清单（SSOT）：`5、静态页面/deploy/publish.manifest.json`——增删发布页面只改清单 `enabled`，不改脚本
- 一键发布：`bash .workbuddy/skills/release-publish/scripts/pub_run_full.sh`（清单驱动 → 生成 out/ → `pub2_mask_pii.py` 脱敏校验 → MASK=0 才 push；回滚见 `release-rollback` skill）
- 当前状态：**已启用** —— 公开仓 `chenlong220192/chensanjie` 已多次发布（最近 2026-09-15）；清单内 `enabled: true` 的条目见该文件
- 归档参考：旧版同步说明（`sync-config.json` 时代）存 `6、暂存/归档/SYNC_GUIDE-旧版2026-09-06.md`，**已作废，勿据此操作**

**切勿**绕过清单直接推送整个仓库。

---

## 六、前端项目化进度

| Phase | 状态 | 内容 |
|---|---|---|
| A | ✅ | CSS/JS 拆出 HTML（`index.html` + `assets/`，GH Pages 标准入口名）|
| B | ✅ | 补齐 `pipe_run_rebuild.sh` 一键构建 / `dev_serve_preview.py` 预览服务器 / head meta+favicon（2026-09-06，按「不破坏双击直开」原则，不引入运行时 fetch）|
| C | ✅ | 脱敏脚本（`scripts/publish/pub2_mask_pii.py`，MASK=0 才放行 push）|
| D | ✅ | 推送公开仓库（`chenlong220192/chensanjie`）——已多次发布，最近 2026-09-15 |

---

## 七、医疗动态速览（非技术）

近期关键节点见 `AGENTS.md`「待完成事项」区——当前关键决策点：**09-17 第三疗程·第 2 次化疗（累计第 8 次）**。化疗前需主治就 **WBC 3.07↓（给药门槛边缘，WBC ≥3.0 / 中性粒 ≥1.5）** 与 **CA19-9 42.77↑（仍高于 37）** 判定是否按期给药；尿路感染持续随访（09-15 尿培养细菌 + 真菌均未生长，可乐必妥临床有效 → 续服至尿常规正常）。数据更新以实际检查结果为准。

---

*README 由整体 review 产出 · 图示源文件存于根目录 `diagrams/` · 2026-09-10 六目录体系 + 发布清单驱动 · 2026-09-17 按最新状态全面校订（脚本命名 pipe*、公开同步已启用、失效引用修复）*
