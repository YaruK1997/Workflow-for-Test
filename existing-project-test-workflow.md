# 既存專案測試導入與執行工作流規範（草案 v0.4）

- **狀態**：草案（Draft），尚未實作（本文件定義之 `ci/` 腳本與擷取器尚待依 §13 優先序開發）
- **日期**：2026-11-20
- **依據文件**：
  - `docs/00_software-testing-types.md`：九種測試類型的三維分類、V-Model、測試金字塔、CI/CD 位置與 Smoke / Sanity / Regression 差異
  - `web-cicd-pipeline.md`：以腳本為主的確定性閘門、Exit Code、瑕疵工單格式、`gates.yml`、`ci/impact-map.yml`、資安掃描（G4）與受保護路徑（§11.5）
  - `../Workflow-for-Agent/workflow.md`：R1–R3 測試分組、`Given / When / Then` 可測性規範、7 類邊界清單、`T-R*` 命名與職責分離原則
  - `test-onboarding-workflow.md`（v0.1）：吸收 PR＋CODEOWNERS 閘門契約、風險計分公式、骨架進度索引、`test.fail()` 契約與 `diff / rebaseline / drift` 機制
- **適用範圍**：已經存在程式碼庫與／或既有規格文件之專案（Brownfield），特別是 Web 系統與前後端應用（非 Web 專案於配置表內以注記對照）；支援完整導入流程與小型/M3 精簡快軌
- **變更紀錄**：
  - `v0.1`：初版
  - `v0.2`：規劃長文暫不寫入版控 `docs/`；`T-CHR-*` 預設列為待確認事項無強制期限；新增 `recon` 預掃描；Load/Stress 與真人 Usability 置於 Wave 3；非 Web 工具表內注記
  - `v0.3`：納入反目標、`recon --diff/--rebaseline/drift`、AC 來源標註與裁決碼、公式化風險分數、Ratchet 日期里程碑、`scaffold`/`case_index` 與 `test.fail()`
  - `v0.4`：修正閘門與腳本可驗證性缺口——恢復 `ci/onboarding/` 結構化閘門契約檔（`scope.yml`、`gaps.yml`、`plan-gate.yml`、`triage.yml`）並走 PR＋CODEOWNERS 保護；修正 `test.fail()` 意外通過（XPASS）為獨立不計次 Exit Code `7` 且限由 `Test Author`／審查者解除；補齊 `Wave 0`–`Wave 3` 明確通過條件、資安基線盤點、共享環境（`shared-staging`）隔離模式、分支／入口覆蓋指標、Nuxt 支援、軟性老化警告（`[STALE]`）、精簡版流程，以及「已知限制」與「待決事項」；修正 `--since` 時間錨點與 `baseline_run.json` 分離以確保 `recon.json` 確定性，並消除 `scaffold` 覆寫既有測試檔之矛盾與 Worker/Wave 命名衝突

---

## 0. 決策摘要

| # | 決策項目 | 結論 |
|---|---|---|
| 1 | 事實來源認定 | **Spec 與 Code 並讀、互不盲信**：兩者衝突列為 `CONFLICT` 交人工以標準裁決碼寫入 `gaps.yml`，嚴禁模型自行臆測何者為真；AC 標註 `spec / observed / assumed` |
| 2 | 確定性盤點與基線分離 | **靜態盤點與動態實測分離**：`recon.json` 只記錄基於目標 `commit_sha` 時間錨點之靜態事實（100% 可重現）；測試跑批與資安掃描另寫 `baseline_run.json` 與 `security_baseline.json` |
| 3 | 版控邊界（精簡控制檔 vs 長文不入 `docs/`） | **長篇探索筆記與報告暫置 `.scratch/`（不入 `docs/`）；僅 4 份極簡閘門控制檔（`ci/onboarding/{scope,gaps,plan-gate,triage}.yml`）及 `docs/stage_state.json` 納入版控並走 PR＋CODEOWNERS 保護**，讓腳本可確定性驗證閘門 A/B/C |
| 4 | 導入策略與門檻 | 依**成熟度（M0–M3）**區分完整版或精簡版；風險權重預設值可於 `scope.yml` 覆寫；Ratchet 同時把關 **Line＋Branch 覆蓋率**與里程碑；Load/Stress 與真人 Usability 置於 **Wave 3** |
| 5 | 骨架設計與特徵測試 | `scaffold` **只建立尚未存在的新測試檔**（`test.fixme()`），絕不修改既有檔案；`T-CHR-*` **不設強制期限，預設列待確認事項**並啟用軟性老化提醒（`[STALE]`） |
| 6 | 失敗處置與 XPASS 權限 | 失敗須於 `triage.yml` 全部分類（缺漏回 Exit `5`）；已知延後缺陷標 `test.fail()` 並綁 Issue；**意外通過回傳專用 Exit `7`（不計入修復迴圈），且禁止 `Code Fixer` 自行拆標記** |

---

## 1. 設計原則與可量測反目標

### 1.1 設計原則

1. **雙源校準，拒絕確認偏誤**：既存專案常見「文件過時」或「程式碼暗藏補丁」，盤點期必須做 Spec ↔ Code 落差比對並透過 PR 裁決，不偏信任何一方。
2. **先腳本確定觀測、後模型語意推演**：檔案結構、路由擷取、模糊用語、Git 變更與風險分數一律由 `recon` 腳本輸出（同一 Commit 兩次輸出 hash 一致），動態跑批另行輸出，確保基準可比較。
3. **閘門契約入版控，長篇過程留暫存**：為兼顧「不於 `docs/` 堆積未定稿長文」與「腳本必須讀得到閘門狀態」，僅將 4 份結構化控制檔置於受 CODEOWNERS 保護的 `ci/onboarding/` 並以 PR 簽核；詳細盤點筆記、設計追溯草稿與報告暫置 `.scratch/`。
4. **先止血建基線，再漸進拉高門檻（Ratchet）**：同時納管行覆蓋率與分支覆蓋率，從 `baseline_run.json` 地板起步，守住變更品質（`diff_min`）並依里程碑推進。
5. **測試分類嚴格對齊三維模型**：依 `docs/00_software-testing-types.md` 區分 **Level**、**Type** 與 **Trigger**，遵守測試金字塔與快速失敗漏斗。
6. **明確職責分離與獨立抽查**：撰寫實作修補者（`Code Fixer`）不得修改 `gates.yml`、測試斷言或拆掉 `test.fail()`；AI 撰寫之測試強制依 `scope.yml` 抽查比例由人類審查。

### 1.2 可量測反目標（Anti-Goals）

出現以下任一量化指標，即視為導入流程失控，必須暫停推進並修正機制：

| 反目標 | 說明 | 確定性量測紅線（觸發即警示／阻擋） |
|---|---|---|
| **1. 紅燈失控且無法歸因** | 測試大量失敗，分不清是實作 Bug、環境錯還是寫錯 | `triage.yml` 存在**未分類失敗數 > 0**（Exit `5`），或無連結 Issue 之 `test.fail()` 數 **> 0** |
| **2. 只看 Spec 漏掉線上行為** | 測試計畫遺漏實際正在提供服務的路由或 API | `recon.json` 之 `entrypoints` 未映射至任何 `FR-xx` / `T-CHR-*` 且未列入 `scope.yml` 排除清單之比例 **> 0%** |
| **3. 盤點結果無法重現** | 同一個程式碼狀態跑出不同盤點清單，導致 `diff` 失效 | 同一 `commit_sha` 連續兩次執行 `node ci/run.mjs recon` 之輸出檔 **SHA-256 不一致** |

---

## 2. 流程總覽

```text
  [既存專案：既有 Spec / 文件（可能殘缺或過時） + 既有程式碼庫 + 既有測試（可能為零或部分損壞）]
                                             │
                                             ▼
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │ 階段 1：範圍鎖定、確定性盤點與 Spec-Code 對齊                                │
 │  1.0 onboard init ─▶ 人工填寫 ci/onboarding/scope.yml（範圍/關鍵/抽查率/環境）│
 │  1.1 確定性預掃描：node ci/run.mjs recon ─▶ .scratch/reports/recon.json      │
 │      （基於目標 commit 時間錨點、Stack Adapter 擷取、計算暫定風險分數）       │
 │  1.2 動態基線與資安觀測：node ci/run.mjs baseline --with-security            │
 │      ─▶ .scratch/reports/baseline_run.json + security_baseline.json          │
 │  1.3 AI/人補齊 Spec 萃取（AC 標註 spec / observed / assumed）與落差比對      │
 │  ──▶ 提交 PR：ci/onboarding/gaps.yml                                         │
 └───────────────────────────────────────────┬─────────────────────────────────┘
                                             ▼
 ═════════[ 閘門 A（PR + CODEOWNERS）：gaps.yml 裁決與 assumed 確認 ]═════════
  檢查指令：node ci/run.mjs onboard lint --gate A
  缺任一裁決碼（bug/spec-outdated/not-implemented/confirm-chr/wont-test）或     
  有未確認 assumed AC ─▶ Exit 5 擋下 PR                                         
 ════════════════════════════════════════════╤═══════════════════════════════════
                                             ▼ (PR Approved & Exit 0)
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │ 階段 2：初步測試計劃與門檻設計                                               │
 │  2.1 風險分級（P0–P3）× 三維測試配置表（含非 Web 工具表內注記）               │
 │  2.2 環境模式（isolated vs shared-staging）＋ Mock 邊界 ＋ 可測性修繕清單    │
 │  2.3 Ratchet 門檻（Line＋Branch floor / diff_min / milestones）              │
 │  2.4 Wave 0–3 執行路線圖與通過條件                                           │
 │  ──▶ 計畫草稿暫置 .scratch/planning/；提交 PR：plan-gate.yml + gates.yml 草案│
 └───────────────────────────────────────────┬─────────────────────────────────┘
                                             ▼
 ═════════[ 閘門 B（PR + CODEOWNERS）：測試計劃與門檻核可 ]═════════════════════
  檢查指令：node ci/run.mjs onboard lint --gate B
  未填寫 approved_by / approved_at 或門檻欄位缺漏 ─▶ Exit 5 擋下 PR             
 ════════════════════════════════════════════╤═══════════════════════════════════
                                             ▼ (PR Approved & Exit 0)
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │ 階段 3：骨架生成、案例設計與品質審查（Gate 2）                               │
 │  3.1 node ci/run.mjs scaffold ─▶ 僅建立「新測試檔」（test.fixme()），禁改舊檔 │
 │  3.2 三軌填充（EC 7 類設計技巧表；T-CHR-* 標待確認事項＋軟性老化追蹤）       │
 │  3.3 node ci/run.mjs status ─▶ 維護 .scratch/reports/case_index.json         │
 │  3.4 Gate 2 雙軌審查：lint-tests（腳本紅旗）＋ 人類依 scope 抽查率反套套邏輯 │
 └───────────────────────────────────────────┬─────────────────────────────────┘
         ▲                                   │
         │ (導入期新 Commit：recon --diff / --rebaseline 保留裁決更新基線)
         │                                   ▼
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │ 階段 4：分波漏斗實測、分類閘門 C 與閉環交接                                  │
 │  4.1 Wave 0 基線恢復與 KNOWN_ISSUE 標記（test.fail() + Issue 連結）          │
 │  4.2 分層漏斗：L0 靜態 ─▶ L1 Unit ─▶ L2 Integ ─▶ L3 Build ─▶ L4 Smoke        │
 │                ─▶ L5 E2E & Regression×2 ─▶ [Wave 3] L6 Load/Stress/Usability │
 │      （Hotfix 支線：Build ─▶ Sanity[impact-map] ─▶ Selective Regression）    │
 │  4.3 失敗處置：Code Fixer 修碼重跑，或依規定登錄 ci/onboarding/triage.yml    │
 │      （預期失敗意外通過 ─▶ Exit 7 [不計修復次數]，交 Test Author 解除標記）   │
 └───────────────────────────────────────────┬─────────────────────────────────┘
                                             ▼
 ═════════[ 閘門 C（PR + CODEOWNERS）：波次失敗 100% 分類檢查 ]═════════════════
  檢查指令：node ci/run.mjs triage --check
  該波存在未分類失敗 ─▶ Exit 5 擋下；全數過關或分類完成始得進下一波 / handoff  
 ════════════════════════════════════════════╤═══════════════════════════════════
                                             ▼ (Wave 0–3 完成 & handoff)
                    [併入 gates.yml ＋ G0 加入 drift 警告 ＋ 交棒 web-cicd-pipeline]
```

---

## 3. 角色定義與職責分離

本工作流支援「單一 Agent 分階段切換乾淨上下文」或「多 Agent 分工」（對應 `../Workflow-for-Agent/multi_agent_workflow.md` 的四類專職 Worker 能力定位；為避免與導入波次混淆，本文不使用 `W1–W4` 代稱角色）。

| 角色 | 核心職責 | 允許讀寫範圍 | 嚴禁行為（職責分離） |
|---|---|---|---|
| **Explorer**（盤點探索者） | 讀取 `recon.json`、`baseline_run.json`、`security_baseline.json` 與 Spec/Code，草擬 `gaps.yml` | 全域唯讀；草稿寫 `.scratch/planning/` 與 `ci/onboarding/gaps.yml`（待 PR） | 嚴禁修改業務碼、既有測試或於 `docs/` 新增說明長文 |
| **Test Architect**（測試計畫設計者） | 校準風險分級、訂定計劃與 `plan-gate.yml`、`gates.yml`、`impact-map.yml`、執行升級 RCA | `.scratch/planning/`、`ci/onboarding/{plan-gate,triage}.yml`、`gates.yml`、`ci/impact-map.yml` | 嚴禁直接修改 `src/` 掩飾測試失敗 |
| **Test Author**（測試案例作者） | 依計劃填充 `scaffold` 骨架與新測試（標註 `author: ai|human`）；**當 `Exit 7 (XPASS)` 發生時，核實修復後移除 `test.fail()`** | `tests/**`（以新增檔或依核可清單修改）、`.scratch/planning/` | **嚴禁照抄 `src/` 內部運算當斷言**；不得修改 `src/` |
| **Code Fixer**（代碼修復者） | 依工單修復 `src/` 缺陷或補齊可測性介面（健康端點、`data-testid`、DI seam） | `src/**`（與明確指派之局部單元測試） | **嚴禁修改 `gates.yml`、刪除失敗測試、新增或移除 `test.fail()`，或擅改 `.security-allowlist.yml`** |
| **Script Runner**（確定性腳本） | 執行 `recon`、`baseline`、`lint`、`scaffold`、`lint-tests`、`status`、`triage`、`run`、`handoff`、`drift` | 讀設定與 `tests/**`；寫 `.scratch/`、`tests/` 全新骨架檔、`docs/stage_state.json` | 嚴禁呼叫 AI 判定結果；嚴禁覆寫或自動附加修改既有手寫測試檔 |
| **Human Owner / Reviewer**（人工決策與抽查者） | 填寫 `scope.yml`；於 PR 核可閘門 A / B / C；**依 `ai_review_sample_rate` 人工審查 AI 寫的測試反套套邏輯**；確認 `T-CHR-*` 與 Wave 3 真人 Usability | 所有 PR 簽核、`ci/onboarding/*.yml`、`gates.yml` | — |

---

## 4. 成熟度分級、環境模式與精簡版流程

### 4.1 既存專案成熟度分級（`M0`–`M3`）

| 等級 | 既存專案現況特徵 | 階段 1–2 調整策略 | 階段 3–4 重點順序 |
|---|---|---|---|
| **M3（高規範）** | 已有結構化 Spec（如 `docs/01_spec.md`）與可執行之測試套件 | 直接比對 Spec-Code 差異，可啟用 **§4.3 精簡快軌** | 補齊未達標 EC、雙跑防抖、`security_baseline` 清理與 CI 接軌 |
| **M2（分散文件）** | 有 PRD、README、OpenAPI/Schema 與部分測試 | 萃取正規化 `FR/AC`，標註 `spec`/`observed`/`assumed` 交閘門 A | Wave 0 處理既有壞測與資安告警 → Wave 1 Smoke+P0 → Wave 2–3 |
| **M1（有碼無文）** | 幾乎無文字 Spec，但程式碼運作中；零或少量測試 | 以 `recon` 路由/API 逆向列舉行為交閘門 A 確認或標 `confirm-chr` | Smoke + `T-CHR-*`（預設待確認事項）鎖住現況 → 逐步補齊 EC |
| **M0（混沌遺留）** | 無 Spec、測試全壞或零測試、強耦合、缺啟動文件 | `scope.yml` 先排除凍結舊區；Wave 0 列為必要前置修復 | 先恢復健康端點與 Smoke，以 `floor`＋`diff_min` Ratchet 起步 |

### 4.2 共享 Staging 環境隔離模式（`env_isolation`）

當專案無法為每個 PR 或本地測試起一套獨立資料庫與微服務，而必須共用 `shared-staging` 環境時，於 `scope.yml` 設定 `env_isolation: shared-staging`，強制套用以下防護：

1. **測試資料租戶／前綴隔離**：所有測試建立之資料（帳號、訂單等）強制帶入 `TEST_<run_id>_` 唯一識別前綴，並在測試結束後精準刪除自建 ID；**嚴禁執行全表 `TRUNCATE`、清庫 Seed 或重置全域系統設定**。
2. **併發排隊鎖**：CI 與腳本執行階段對 `shared-staging` 啟用單一執行鎖（如 GitHub `concurrency` 鎖定環境），避免兩次執行互相汙染測試資料。
3. **高風險與壓測隔離**：`shared-staging` **禁止執行 Stress（打到崩潰點）或破壞性故障注入（斷網/殺進程）**；Load/Stress 必須切至獨立壓測環境，或在明確維護時窗加鎖進行。

### 4.3 精簡版流程（Fast-Track Mode）

針對 **`M3` 專案** 或 **小型單體服務（`recon.json` 偵測 `entrypoints ≤ 10`）**，可在 `scope.yml` 啟用 `mode: lightweight`，縮減儀式成本但不犧牲確定性：
- **合併閘門**：將「閘門 A（`gaps.yml`）」與「閘門 B（`plan-gate.yml`）」合併為**單一 PR** 一次簽核。
- **合併波段**：將 `Wave 1`（Smoke+P0）與 `Wave 2`（EC+E2E）合併為單一實測波段；無長期日期 `milestones`，Ratchet 直接採 `floor`＋`diff_min`＋`target`。
- **維持不變**：`recon.json` 確定性、`scaffold` 不改舊檔、`triage.yml` 閘門 C、AI 人工抽查與 `Exit 7` 規則一律保留。

---

## 5. 階段 1：範圍鎖定、確定性盤點與對齊

### 5.1 必填範圍檔：`ci/onboarding/scope.yml`

由 `node ci/run.mjs onboard init` 建立範本並填入當前 `HEAD` 作為初始 `baseline_sha`，交由 Human Owner 補充必要欄位後納入版控（受 CODEOWNERS 保護）。**未提供有效 `scope.yml` 時，`recon` 與 `onboard lint` 拒絕往下推進。**

```yaml
# ci/onboarding/scope.yml（受 CODEOWNERS 保護）
version: 1
mode: standard                    # standard | lightweight（小型或 M3 專案可用精簡版）
baseline_sha: "<auto-filled-sha>" # init 時鎖定之基準 commit；僅 --rebaseline 可更新
env_isolation: isolated           # isolated | shared-staging

include:
  - src/server/api/
  - src/pages/
exclude:
  - src/legacy/deprecated-admin/

critical:                         # 業務關鍵流程，計算風險時直接提升業務關鍵度並優先列 P0
  - name: auth-login
    paths: [src/server/api/auth/, src/pages/login]
  - name: checkout
    paths: [src/server/api/checkout/, src/pages/checkout]

# 風險分數權重（暫定預設值，可依專案特性覆寫；見 §12 待決事項 1）
risk_weights:
  churn: 0.30
  fix_density: 0.25
  uncovered: 0.25
  business_critical: 0.20

# Gate 2 人工抽查與軟性老化門檻
ai_review_sample_rate:
  p0: 1.0                         # P0 由 AI 撰寫之測試必須 100% 經人類審查反套套邏輯
  default: 0.3                    # P1–P3 至少抽查 30%
aging_warn_days:
  characterization: 30            # T-CHR-* 超過 30 天未轉正發出 [STALE] 軟性警告（不阻擋）
  expected_fail: 30               # test.fail() 超過 30 天未修復發出 [STALE] 軟性警告
```

### 5.2 確定性預掃描與增量追趕（`node ci/run.mjs recon`）

- **實作規格**（對齊 `web-cicd-pipeline.md` §5.1）：
  - 模組：`ci/lib/recon.mjs`（含 `ci/lib/recon/stacks/` 框架擷取器）；Node 24 LTS（ESM `.mjs` + `// @ts-check`），僅依賴 `yaml` 與內建模組；`spawn` 一律 `shell: false` + timeout。
  - **純靜態、100% 確定性**：`recon` **絕不執行測試、絕不以「今天日期」為基準、絕不寫入浮動時間戳至 `recon.json` 本體**（執行紀錄時間寫至 `.scratch/logs/recon_<時間戳>.log`）。同一 `commit_sha` 執行兩次，`recon.json` 檔案 SHA-256 必須完全一致。
  - **Commit 時間錨點規則**：所有 `--since-days <D>` 統計一律以**目標 `commit_sha` 本身之 commit 時間（`git show -s --format=%ct <sha>`）往回推 `<D>` 天**，且只遍歷 `<sha>` 之祖先歷史；或使用固定 `--since-commits <N>`。保證換一天重跑同一 Commit 結果不變。
  - **Stack Adapter 涵蓋**：內建規劃包含 `nextjs`（App/Pages Router）、**`nuxt`（Nuxt 3 `pages/` + Nitro `server/api/`）**、`sveltekit`、`express/fastify`、`openapi` 及 `cli/python-fallback`；未匹配框架退回 OpenAPI/Schema＋通用入口掃描並於輸出標註 `fallback`。
  - **資安靜態檢核**：掃描 `.env.example` key 名稱（**嚴禁讀取 `.env` 機密值**），同時檢查 Git 追蹤清單是否誤含 `.env`、有無 `.security-allowlist.yml` 及 `Dockerfile` non-root `USER`。

- **指令介面**：

  ```text
  node ci/run.mjs recon [--sha <commit>] [--since-commits 200 | --since-days 90] [--scope ci/onboarding/scope.yml] [--out .scratch/reports/recon.json]
  node ci/run.mjs recon --diff [--base <sha>]
  node ci/run.mjs recon --rebaseline
  ```

- **輸出契約（`.scratch/reports/recon.json` — 純靜態事實，不含動態測試時間戳）**：

  ```json
  {
    "schema_version": 3,
    "target_sha": "<sha>",
    "commit_timestamp": 1795147200,
    "since_anchor": { "mode": "commits", "value": 200 },
    "runtime_and_scripts": {
      "ecosystems": ["node", "docker"],
      "package_manager": "pnpm",
      "scripts": { "dev": "...", "build": "...", "test": "...", "lint": "...", "typecheck": "..." },
      "env_keys": ["DATABASE_URL", "REDIS_URL", "JWT_SECRET"]
    },
    "git_conventions": { "detected": "conventional-emoji", "samples": 50, "ratio": 0.92, "uses_version_prefix": false },
    "security_static": {
      "tracked_env_files": [],
      "has_allowlist": true,
      "dockerfile_non_root": true,
      "has_lockfile": true
    },
    "spec_candidates": [
      { "path": "docs/api.md", "kind": "doc", "sha256": "...", "lines": 240, "vague_term_hits": [{ "line": 48, "term": "適當" }] }
    ],
    "code_surface": {
      "stack_adapter": "nuxt-nitro",
      "entrypoints": [{ "kind": "http", "method": "POST", "path": "/api/orders", "file": "server/api/orders.post.ts", "line": 1 }],
      "external_deps": ["postgres", "redis", "stripe-sdk", "system-clock"],
      "has_healthcheck": true,
      "has_container": true
    },
    "hotspots": [
      { "path": "server/api/orders.post.ts", "commits": 38, "fix_commits": 11, "complexity": 14, "risk_score_static": 0.78 }
    ],
    "existing_tests": {
      "frameworks": ["vitest", "playwright"],
      "files": ["tests/unit/auth.test.ts"],
      "tags": { "@smoke": 0, "@e2e": 2, "@regression": 0 }
    }
  }
  ```

- **導入期增量追趕與持續漂移**：
  - `recon --diff`：比對 `scope.yml` 的 `baseline_sha` 與當前 `HEAD`，標出新增路由/API（列 `Code_Only` 待補）、刪除（標 `removed` 待清理骨架）與介面變更（標 `stale` 需複核 AC）。
  - `recon --rebaseline`：人工決定時機後將 `scope.yml` 的 `baseline_sha` 更新為 `HEAD`，**保留既有 `gaps.yml` 裁決與測試**。
  - `drift`（CI `G0`）：交接後於 PR 檢查新增路由/API 是否缺少對應測試或 FR，發出**非阻擋性警告**。

### 5.3 動態基線實測與資安基線盤點（獨立輸出，不混入 `recon.json`）

為避免測試執行與即時弱點資料庫破壞 `recon.json` 的確定性，動態量測由獨立指令執行並寫入獨立檔案：

```text
node ci/run.mjs baseline [--with-security]
```

1. **測試基線實測（`.scratch/reports/baseline_run.json`）**：
   - 以專案現有測試指令跑批（含 JSON reporter 與覆蓋率），記錄 `executed_at`、`commit_sha`、通過/失敗/Flaky/環境阻礙數量，以及 **全域與核心模組之 `line_pct` 與 `branch_pct`**（作為 Ratchet `floor` 依據）。
2. **既存專案資安基線盤點（`.scratch/reports/security_baseline.json`，加 `--with-security` 觸發）**：
   - 以 **advisory（只盤點不阻擋）** 模式執行 `gitleaks`、`semgrep`/`bandit` 與 `pnpm audit`/`trivy fs`，將既有漏洞依 `Critical / High / Medium / Low` 數量與清單匯總。
   - **目的**：讓團隊在階段 2 就看見「歷史欠債有多少弱點」，於 `Wave 0`–`Wave 2` 先行修補或依規定登錄 `.security-allowlist.yml`（含理由、核可者與到期日），避免交接給 `web-cicd-pipeline.md` 的 `G4` 時第一次執行即整片紅燈卡死。

### 5.4 多來源 Spec 萃取、公式風險計分與落差對齊

1. **Spec 蒐集與來源標註**：從 `recon.json` 之 `spec_candidates` 萃取臨時 `FR-xx` / `AC-xx` / `NFR-xx`；每條 AC 強制標註「正常 / 異常」及來源：
   - `spec`（既有文件或 OpenAPI/Schema）
   - `observed`（從程式碼或實測觀察推得，如 `Code_Only`）
   - `assumed`（AI 依常識補全，**必須於閘門 A 人工逐條確認**）
2. **風險分數與 `P0`–`P3` 分級**（結合 `recon.json` 與 `baseline_run.json` 覆蓋率，權重取自 `scope.yml` 預設暫定值）：
   $$\text{risk} = w_{\text{churn}} \cdot \text{修改頻率} + w_{\text{fix}} \cdot \text{fix密度} + w_{\text{cov}} \cdot (1 - \text{覆蓋率}) + w_{\text{biz}} \cdot \text{業務關鍵度}$$
   - 各項依專案內最大值正規化到 `0–1`；無法辨識 `fix` commit 慣例時 `fix_density` 權重歸零並等比重分配其餘權重。
   - 暫定門檻：`P0 ≥ 0.70`（或命中 `scope.yml` 之 `critical`）、`P1 ≥ 0.50`、`P2 ≥ 0.30`、`P3` 其餘。
3. **Spec ↔ Code 落差比對**：分類為 `Aligned`、`Spec_Only`、`Code_Only`、`Conflict`（兩者矛盾時嚴禁自選邊站）。

### 5.5 閘門 A 檔案契約與檢查（`ci/onboarding/gaps.yml`）

Explorer 完成比對後，將所有 `Conflict`、`Spec_Only`、`Code_Only` 與 `assumed` AC 寫入版控檔案 **`ci/onboarding/gaps.yml`** 並開立 PR（受 CODEOWNERS 保護）。**所有項目必須由人工填入裁決碼，未填齊時由 `node ci/run.mjs onboard lint --gate A` 回傳 Exit `5`。**

```yaml
# ci/onboarding/gaps.yml（PR 提交 + CODEOWNERS 保護）
version: 1
baseline_sha: "<sha>"
items:
  - id: GAP-01
    kind: conflict                # conflict | spec_only | code_only | assumed_ac
    fr: FR-05
    spec_ref: "docs/order.md#L42: 30天內可退貨"
    code_ref: "server/api/refund.post.ts#L18: >14天回400"
    decision: spec-outdated       # bug | spec-outdated | not-implemented | confirm-chr | wont-test | confirmed(僅assumed)
    decided_by: "@owner"
    decided_at: "2026-11-20"
    note: "現行商業規則已改為14天，排程更新舊文件"
  - id: GAP-02
    kind: code_only
    fr: FR-09
    spec_ref: null
    code_ref: "server/api/legacy-export.get.ts#L1"
    decision: confirm-chr         # 列為 T-CHR-* 待確認事項
    decided_by: "@owner"
    decided_at: "2026-11-20"
    note: "先以特徵測試保護，待產品確認是否保留"
```

---

## 6. 階段 2：產生初步測試計劃與閘門 B

### 6.1 初步測試計劃編製規則

Test Architect 讀取通過閘門 A 的 `gaps.yml`、`recon.json`、`baseline_run.json` 與 `security_baseline.json`，草擬計劃（長篇說明置於 `.scratch/planning/02_test_plan.md` 或對話，關鍵結構化契約寫入 `plan-gate.yml` 與 `gates.yml` 草案）：

1. **測試範圍與資源配置**：依 `P0–P3` 決定覆蓋深度（P0 必備 Unit + Integ + `@smoke` + `@e2e` + 7 類 EC + `@regression`；P1 涵蓋 Unit + Integ + 關鍵 E2E + `@regression`；P2 以 Unit + API Integ 為主；P3 靜態檢查或 `T-CHR-*`）。
2. **三維測試類型配置表（對齊 `docs/00_software-testing-types.md`，含非 Web 表內注記）**：

   | 測試類型 | 層級 / 維度 | 目標對象與原則（含非 Web 情境） | 波段與預算 | 建議標記 / 工具（含非 Web 注記） |
   |---|---|---|---|---|
   | **Unit** | Level: Unit | 純邏輯、權限規則、狀態轉移 | Wave 1 起；無 IO、秒級 | Vitest / Pytest<br>_（非 Web：`pytest`、`go test` 等原生測試器）_ |
   | **Integration** | Level: Integration | API Route ↔ DB/Redis 契約、異常注入<br>_（非 Web：Subprocess/Pipe、DB/檔案/管線契約）_ | Wave 1–2；Container、分鐘內 | Vitest / Pytest + Docker<br>_（非 Web：Testcontainers 或暫存目錄/SQLite）_ |
   | **Smoke** | Trigger: Smoke | 開站、Healthcheck、登入、P0 關鍵鏈路<br>_（非 Web：CLI `--help`/主指令、管線最小樣本）_ | Wave 1 起；**≤ 15 條、≤ 5 分鐘** | Playwright `@smoke`<br>_（非 Web：CLI/API 冒煙腳本，同標 `@smoke`）_ |
   | **E2E** | Type: Functional (System) | 跨頁/跨服務完整使用者旅程<br>_（非 Web：完整 CLI 工作流或管線輸出驗證）_ | Wave 2 起；`retries: 0` | Playwright `@e2e`<br>_（非 Web：CLI runner / Golden 輸出比對）_ |
   | **Regression** | Trigger: Regression | P0/P1 全套件 + `T-BUG-*` 歷史案例（跨各層） | Wave 3 完整化；連跑 2 次防抖 | `@regression`（同跨 Web 與非 Web） |
   | **Sanity** | Trigger: Sanity | 熱修復（Hotfix）區域之窄而深驗證 | Hotfix 隨時；依 `impact-map` | 模組 Tag（如 `@auth` / `@parser`） |
   | **Load / Stress** | Type: Non-functional | 高併發 API/頁面（p95 與 1.5× 容量潰點）<br>_（非 Web：管線吞吐量、記憶體壓力上限）_ | **Wave 3**；禁對 `shared-staging` 壓崩 | k6<br>_（非 Web：Locust、`pytest-benchmark`、`hyperfine`）_ |
   | **Usability** | Type: Non-functional (Acceptance) | 自動化 a11y/效能 + 真人任務探索腳本<br>_（非 Web：CLI 文件、錯誤訊息可讀性與 DX；無 UI 時 a11y 標 `N/A`）_ | **Wave 3**；Preview / Staging | axe-core / LHCI / 人工記錄單<br>_（非 Web：文件測試 + DX 檢核清單）_ |

3. **Ratchet 多維覆蓋門檻（Line＋Branch＋入口，不單看行覆蓋率）**：

   ```yaml
   # gates.yml 草案（經閘門 B 核可後啟用）
   unit:
     coverage:
       mode: ratchet
       floor:    { line_min: 41, branch_min: 32 }          # 取自 baseline_run.json 無條件捨去
       diff_min: { line_min: 80, branch_min: 70, core_line_min: 90 }
       milestones:
         - { by: 2026-12-31, line_min: 55, branch_min: 45 }
       target:   { line_min: 80, branch_min: 70, core_line_min: 90, core_modules: [server/api/auth/] }
   integration:
     entrypoint_coverage_min: 100                          # P0/P1 entrypoints 皆具整合或系統測試覆蓋
     ec_mapping_min: 100
   ```
   > `Wave 2–3` 可針對 `P0` 純業務邏輯模組選用變異測試（Mutation Testing，如 Stryker / `mutmut`）驗證斷言有效性。

4. **環境邊界與資安清償計畫**：依 `scope.yml` 的 `env_isolation`（`isolated` vs `shared-staging`）訂定 Test Doubles 邊界、可測性修繕項目，以及處理 `security_baseline.json` 既有高風險弱點的波段排程。

### 6.2 閘門 B 檔案契約與檢查（`ci/onboarding/plan-gate.yml`）

提交 PR（包含 `plan-gate.yml`、`gates.yml` 草案與 `ci/impact-map.yml` 初稿），經人工核可並由 `node ci/run.mjs onboard lint --gate B` 驗證通過（未填核可人與日期回傳 Exit `5`）：

```yaml
# ci/onboarding/plan-gate.yml（PR 提交 + CODEOWNERS 保護）
version: 1
baseline_sha: "<sha>"
gates_ref: "gates.yml"
impact_map_ref: "ci/impact-map.yml"
env_isolation: isolated           # 須與 scope.yml 一致
security_remediation_wave: 1      # 既有 Critical/High 資安基線修補或豁免登錄期限（Wave 0..3）
smoke_candidates_count: 12        # 上限 15
approved_by: "@owner"
approved_at: "2026-11-21"
```

---

## 7. 階段 3：骨架生成、案例設計與 Gate 2 審查

### 7.1 三軌推導與 EC 設計技巧對照

1. **軌道 A：Spec 黑盒推導**：依已確認 `FR-xx`/`AC-xx` 轉寫為可斷言之 `Given / When / Then`（正常類 `T-R1-AC-xx`、異常類 `T-R2-AC-xx`），**避免先看內部實作算式再倒推斷言**。
2. **軌道 B：Code & 風險白盒推導**：針對 `P0/P1` 模組檢核 7 類邊界（`T-R2-EC-xx`，不適用標 `N/A + 理由`）與歷史修復（`T-BUG-xx`），對應設計技巧：

   | EC 類別 | 建議測試設計技巧 |
   |---|---|
   | 空值 / 缺失、非法格式 | **等價類劃分** + 異常輸入注入集 |
   | 極大值 / 邊界值 | **邊界值分析**（`min−1` / `min` / `max` / `max+1`） |
   | 權限不足 / 越權 | **決策表**（角色 × 操作 × 資源歸屬） |
   | 資料遺失 / 斷線、併發衝突 | **狀態轉換分析 + 異常注入 / 併發模擬** |
   | 網路 / 相依逾時 | **時間與網路 Mock 控制**（Fake Timers / MSW fault） |

3. **軌道 C：遺留無規格區特徵測試（`T-CHR-*`，預設待確認＋軟性老化追蹤）**：
   - 以 Golden Master 鎖定現行輸入輸出，於測試註解標註 `CHARACTERIZATION (pending owner confirmation)`。
   - **不設強制轉正期限**；但若超過 `scope.yml` 的 `aging_warn_days.characterization` 天數仍未確認，系統僅於報告與 `status` 標註 **`[STALE]` 軟性提醒**（不阻擋建置），直到人工確認轉正為 `T-R1-*`/`T-R2-*` 或移除。

### 7.2 骨架寫入鐵律（`scaffold`）與進度追蹤

1. **`node ci/run.mjs scaffold` 安全寫入規則（徹底不碰既有檔案）**：
   - 依通過閘門 B 的清單與 `case_index.json`，為尚未對應的條目建立**全新的測試檔案**（如 `tests/e2e/<flow>.generated.spec.ts`），內部以 `test.fixme('T-R1-AC-xx ... @tag', ...)` 標記與 `Given/When/Then` 註解。
   - **絕對禁止覆寫或自動附加（append）內容到任何既有手寫測試檔**！若預定檔名已存在，腳本自動改建 `<name>.scaffold.<ext>` 並輸出提示，由人/AI 視需要整合；若既有測試已涵蓋某條目，只需人工/AI 在該測試標題加上 `T-R*` ID，腳本掃描時即自動納管、不重複生成骨架。
2. **進度索引（`.scratch/reports/case_index.json` + `node ci/run.mjs status`）**：
   - 由腳本掃描 `tests/**` 與計畫產出，追蹤每條 `T-*` 的 `status`（`stub` / `written` / `passing` / `expected_fail`）、`author`（`ai` / `human`）、`created_at` 與老化狀態 `[STALE]`。

### 7.3 Gate 2 測試設計雙軌品質審查（解決「AI 審 AI」盲區）

大量執行前必須同時通過「腳本靜態檢查」與「人類實質抽查」：

1. **腳本確定性檢查（`node ci/run.mjs lint-tests`）**：
   - 已填寫（非 `fixme`）測試至少含 1 個有效斷言；無 `.only`；測試 ID 唯一且 Tag 合法；`@smoke` ≤ 15 條；無 `sleep` / `waitForTimeout` 等硬等待模式；無寫死外部真實網域；無未經閘門 A 裁決之 `CONFLICT` / `assumed`。
2. **獨立反套套邏輯審查（強制人類參與）**：
   - 依 `scope.yml` 的 `ai_review_sample_rate`（例如 `P0: 1.0`、`default: 0.3`），對所有 `author: ai` 標記的測試於 PR 中強制由 **Human Reviewer** 實質抽查：核對「斷言期待值是否獨立推導自需求契約，而非複製受測函式的計算公式」，以及是否遵守 `shared-staging` 資料隔離規範。

---

## 8. 階段 4：分波漏斗實測、閘門 C 與閉環

### 8.1 Wave 0–Wave 3 分波範圍與明確通過條件

| 波次 | 測試範圍 | 通過／放行條件（進入下一波必備） |
|---|---|---|
| **Wave 0** | 環境恢復與基線止血 | 1. 測試環境與 Healthcheck 可穩定啟動（符合 `env_isolation` 模式）<br>2. 完成 `baseline_run.json` 與 `security_baseline.json`<br>3. 既有 `PRE_EXISTING_FAIL` 及閘門 A 裁決之 `bug` 已登錄 `triage.yml` 並標 `test.fail()`＋Issue；當期 Critical/High 資安項已依 `plan-gate.yml` 排解或登錄有效豁免 |
| **Wave 1** | 最小防線：`@smoke`（≤15條、≤5m）＋ `P0` 正常路徑 Unit/Integ | 1. `P0` 正常路徑無未完成骨架（或已標 `wont-test`）<br>2. `L0–L4 Smoke Gate` 通過；其餘失敗 100% 完成 `triage.yml`（無未處理阻擋級 `IMPL_DEFECT`）<br>3. Line＋Branch 覆蓋率不低於 `floor` |
| **Wave 2** | 深度邊界：`P0/P1` 異常＋7 類 EC（`T-R2-*`）＋ `@e2e`（`retries: 0`） | 1. `P0/P1` 異常與 EC 對應率達 `100%`（含 `N/A + 理由`）<br>2. `L0–L5` 通過或所有失敗 100% 完成 `triage.yml`<br>3. 變更覆蓋率達 `diff_min`（Line＋Branch） |
| **Wave 3** | 全量 `@regression`（連跑 2 次）＋ **Load/Stress（k6）＋ Usability（a11y/LHCI/視覺/真人）** | 1. 全套件連跑 2 次 **0 Flaky**（`Exit 3 = 0`）<br>2. Load/Stress 達標或記錄基線（`shared-staging` 依 §4.2 限制）<br>3. axe 0 Critical/Serious、LHCI ≥ 90、完成《真人易用性測試任務單》<br>4. 閘門 C 通過，具備 `handoff` 併入 `gates.yml` 資格 |

> **Hotfix 專用支線**（沿用 `docs/00_software-testing-types.md` §4）：小修正／Hotfix 通過 `L0–L3` 後，執行 **`Sanity`（依 `ci/impact-map.yml` 窄而深驗證修正點）→ `Selective Regression`**；未映射路徑退回全量 Regression。

### 8.2 分層漏斗執行序列

```text
  [node ci/run.mjs run --wave <0..3>]
        │
        ▼
  (L0) 靜態與中繼資料檢查：Lint · TypeCheck · Secrets · lint-tests · trace-check
        ▼
  (L1) Unit 測試（R1 功能 + Ratchet [Line/Branch floor & diff_min] 檢查）
        ▼
  (L2) Integration 測試（R2 邊界異常 + Service Container + Entrypoint/EC 覆蓋率）
        ▼
  (L3) Build & 部署（isolated 或 shared-staging 加鎖）─▶ Health Probe (200 OK)
        ▼
  (L4) Smoke Gate（@smoke，≤ 15 條、≤ 5 分鐘）──▶ [Fail 立即中止！不跑 L5/L6]
        ▼
  (L5) System E2E & Regression 雙跑防抖（retries: 0，連跑 2 次）
        ▼
  (L6) [僅 Wave 3] Load / Stress（k6）＋ a11y / LHCI / 視覺回歸 ＋ 真人 Usability 任務單
```

### 8.3 指令總表與統一 Exit Code（解決 `XPASS` 計數衝突）

#### 完整指令集（統一前綴 `node ci/run.mjs`）

| 指令 | 階段 | 核心作用 |
|---|---|---|
| `onboard init` | 階段 1 | 初始化 `ci/onboarding/scope.yml`、鎖定 `baseline_sha`、確認 `docs/stage_state.json` |
| `recon [--sha ...]` / `recon --diff` / `recon --rebaseline` | 階段 1–3 | 純靜態確定性盤點（`.scratch/reports/recon.json`）、導入期增量比對與更新基準 |
| `baseline [--with-security]` | 階段 1 | 動態跑批既有測試與資安掃描（`.scratch/reports/baseline_run.json` + `security_baseline.json`） |
| `onboard lint --gate <A\|B\|all>` | 閘門 A / B | 驗證 `gaps.yml`（裁決碼與 `assumed`）及 `plan-gate.yml` 完整性，缺漏回 **Exit `5`** |
| `scaffold` | 階段 3 | 僅建立「新測試檔」之 `test.fixme()` 骨架，絕不覆寫或附加修改既有檔案 |
| `lint-tests` | Gate 2 | 靜態紅旗檢查（斷言 ≥ 1、禁 `.only`、Smoke 預算、禁 `sleep`、ID 唯一） |
| `status` | 階段 3–4 | 更新 `.scratch/reports/case_index.json`，列示進度與 `[STALE]` 老化提醒 |
| `run --wave <0..3>` / `<gate>` | 階段 4 | 執行分層漏斗、解析 JSON/SARIF 產出報告，並偵測 `test.fail()` 意外通過 |
| `triage --check` | 閘門 C | 核對該波所有失敗是否皆已登錄 `ci/onboarding/triage.yml`，未分類回 **Exit `5`** |
| `handoff` | 交接期 | 檢查 Wave 0–3 交接條件（無未完成骨架），合併 `gates.yml` 並啟用 `G0 drift` |
| `drift` | CI `G0` | 持續偵測未對應 FR 或測試的新增路由/API/元件（非阻擋警告） |

#### 統一 Exit Code（對齊 `web-cicd-pipeline.md` 並修正 `XPASS` 誤計次問題）

| Code | 意義 | 計入修復迴圈？ | 路由對象 / 處理動作 |
|---|---|---|---|
| `0` | 目標閘門或波次通過 | 否 | 推進下一階段或關閉工單 |
| `1` | 測試失敗 / 格式錯誤 / 審查重大問題 | **是** | 依 `triage.yml` 路由 `Code Fixer`（`IMPL_DEFECT`）或 `Test Author`（`TEST_DEFECT`） |
| `2` | 覆蓋率（Line/Branch/Entrypoint）或 EC 對應率不足 | **是** | 路由 `Test Architect` 補測或判定死碼 |
| `3` | 不穩定測試（L5 雙跑結果不一致） | **是** | 路由 `Test Architect` 做 RCA 歸屬判定 |
| `4` | 同一瑕疵累計 3 次修復未通過 | **是（升級）** | 升級 `Test Architect` 根因分析 |
| `5` | 停止條件，或 **閘門 A/B/C 檢查存在未裁決 / 未分類項目** | 否 | 暫停交 `Human Owner`（PR 裁決或重置計數） |
| `6` | 工具、腳本或環境錯誤（自動重跑 1 次） | 否 | 路由 `route:infra` 排除環境異常 |
| **`7`** | **`XPASS`：標記 `test.fail()` 的預期失敗測試意外通過** | **否（關鍵修正）** | **不計入 `count_exit_codes` 修復次數，不觸發升級**；通知 **`Test Author`／審查者**（禁止 `Code Fixer` 自拆）核實修復後移除 `test.fail()` 並關閉 Issue |

### 8.4 閘門 C 檔案契約：`ci/onboarding/triage.yml` 與故障處置

每波測試若有失敗，必須全數登錄於版控檔案 **`ci/onboarding/triage.yml`**（PR + CODEOWNERS 保護），經 `node ci/run.mjs triage --check` 確認 0 未分類項目後，始可完成該波次：

```yaml
# ci/onboarding/triage.yml（PR 提交 + CODEOWNERS 保護）
version: 1
wave: 1
failures:
  - test_id: T-R2-EC-12
    fingerprint: a1b2c3
    class: IMPL_DEFECT            # IMPL_DEFECT | KNOWN_ISSUE | TEST_DEFECT | ENV_DEFECT | SPEC_DRIFT
    disposition: fix-now          # fix-now（交 Code Fixer） | defer-xfail（核可延後，標 test.fail()）
    issue: "#42"
    by: "@owner"
    created_at: "2026-11-22"
    note: "密碼超過 128 字元回傳 500，應為 400"
```

- **雙軌處置與權限**：
  - `IMPL_DEFECT`（`disposition: fix-now`）：交 `Code Fixer` 修復 `src/` 後從 `L0` 重跑驗證，計數更新於 `docs/stage_state.json`。
  - `KNOWN_ISSUE` / 核可之 `defer-xfail`：於測試加上 `test.fail()` 並綁定 Issue 編號；若該 Issue 狀態已為 `CLOSED` 而測試檔仍留 `test.fail()`，或超過 `aging_warn_days.expected_fail` 天數，`status` 標記 `[STALE]` 警示；一旦實作修復使該測試通過，腳本回傳 **Exit `7`（`XPASS`）**，交 **`Test Author`** 核對後移除 `test.fail()`（`Code Fixer` 無權動此標記）。
  - `SPEC_DRIFT`：凍結為 `PENDING_OWNER`，交 Human Owner 回到 `gaps.yml` 重新裁決。

### 8.5 執行報告與交接（`.scratch/reports/test_report.md`）

報告維持寫入 `.scratch/reports/test_report.md`（不污染版控 `docs/`），除彙整各層通過率、覆蓋率（Line/Branch）變化、`OPEN` 工單與 `handoff` 就緒檢查外，**獨立置頂彙整兩類軟性老化／待確認事項**：
1. **`T-CHR-*` 特徵測試待確認清單**（標註超過 `aging_warn_days` 之 `[STALE]` 項目，不設強制到期阻擋）。
2. **`test.fail()` 預期失敗與 `security_baseline` 殘留項目**（含 `[STALE]` 提醒與 `.security-allowlist.yml` 到期日）。

---

## 9. 與現有規範之整合對照

| 本工作流環節 | 對應 `docs/00_software-testing-types.md` | 對應 `web-cicd-pipeline.md` | 對應 `../Workflow-for-Agent/workflow.md` & `test-onboarding-workflow.md` |
|---|---|---|---|
| **階段 1（盤點與閘門 A）** | 盤點現有測試覆蓋之 Level / Type / Trigger | §5.1 腳本規範、§11.2 慣例偵測、G4 資安前置預警 | 吸收 `test-onboarding` 的 `scope.yml`、確定性盤點、風險公式、`gaps`＋閘門 A（改進 `--since` 錨點與 `baseline_run.json` 分離） |
| **階段 2（計劃與閘門 B）** | §1–§3 三維模型與測試金字塔（表內注記非 Web） | 產出 `gates.yml`（Ratchet Line+Branch）與 `impact-map` | 承接 3A 規劃；保留 PR 核可閘門 B（`plan-gate.yml`），支援 `shared-staging` 與精簡快軌 |
| **階段 3（骨架與 Gate 2）** | §5 Smoke / Sanity / Regression 設計邊界 | `@smoke` / `@e2e` / `@regression`、Wave 3 k6 與 axe/LHCI | 沿用 GWT、7 類邊界；吸收 `scaffold` 與 `case_index`（修正為絕不改既有檔，並補人工抽查防套套邏輯） |
| **階段 4（實測與閘門 C）** | §4 CI/CD 漏斗順序與 Hotfix Sanity 支線 | 共用 Exit Code `0–6`（擴充 `7` XPASS）、`docs/stage_state.json`、`DEF-*` | 沿用 3B 雙跑防抖與修碼閉環；吸收 `triage.yml` 閘門 C 與 `test.fail()`（修正 `Code Fixer` 權限與不誤觸升級計次） |

---

## 10. 產出結構與版控邊界總覽

嚴格區分「**受 PR＋CODEOWNERS 保護之精簡控制契約（入版控）**」與「**長篇探索筆記、日誌與動態報告（留 `.scratch/`，不污染 `docs/`）**」：

```text
existing-project/
├── gates.yml                            # [入版控 / CODEOWNERS] 測試門檻（Line+Branch floor / diff_min / target）
├── .security-allowlist.yml              # [入版控 / CODEOWNERS] 資安豁免清單（含理由、核可者與到期日）
├── docs/
│   └── stage_state.json                 # [入版控 / CODEOWNERS] 本地修復迴圈與升級計數狀態（對齊 web-cicd-pipeline）
├── ci/
│   ├── run.mjs                          # 單一入口（onboard / recon / baseline / scaffold / lint-tests / status / run / triage / handoff / drift）
│   ├── impact-map.yml                   # [入版控 / CODEOWNERS] 路徑 → Sanity / Regression Tag 映射
│   ├── onboarding/                      # [入版控 / CODEOWNERS] 四份精簡閘門契約檔（PR 提交）
│   │   ├── scope.yml                    #   必填：範圍、關鍵流程、暫定風險權重、AI 抽查率、環境模式、老化警告天數
│   │   ├── gaps.yml                     #   閘門 A：Conflict / Spec_Only / Code_Only 裁決碼與 assumed 確認
│   │   ├── plan-gate.yml                #   閘門 B：測試計劃核可簽核與環境／資安清償波次
│   │   └── triage.yml                   #   閘門 C：各波失敗歸因分類與 KNOWN_ISSUE / defer-xfail 登錄
│   └── lib/
│       ├── recon.mjs (+ stacks/)        # 階段 1 確定性預掃描（含 nextjs / nuxt / sveltekit / express / openapi / fallback）
│       ├── baseline.mjs                 # 動態基線測試與資安預掃描（--with-security）
│       ├── onboard-lint.mjs             # 閘門 A / B / C 與 Gate 2 靜態紅旗檢查（lint-tests）
│       └── scaffold.mjs                 # 安全骨架生成（僅建新檔）與 case_index 統計
├── tests/                               # [入版控] 實際測試套件（unit / integration / e2e / load）
└── .scratch/                            # [.gitignore 排除，不入版控]
    ├── cache/                           # 符號索引與快取
    ├── planning/                        # 規劃期長篇草稿（盤點分析 / 計畫長文 / 設計追溯表）
    ├── logs/                            # Raw 執行日誌（recon_*.log, defects.jsonl）
    └── reports/                         # recon.json、baseline_run.json、security_baseline.json、case_index.json、test_report.md
```

---

## 11. 已知限制（Known Limitations）

1. **規範尚未實作（Draft Only）**：本文件定義之 `ci/lib/recon.mjs`、`baseline.mjs`、`onboard-lint.mjs`、`scaffold.mjs` 等腳本尚未實作，目前無法直接執行自動化檢查；必須先依 §13 優先序落地腳本與合約測試後才具備確定性卡控能力。
2. **Stack Adapter 初期覆蓋有限**：如 Next.js、Nuxt（Nitro）、SvelteKit 或特定後端框架之自動化路由擷取須逐一開發；未支援框架在 adapter 補齊前僅能依賴 OpenAPI/Schema 與通用掃描，人工盤點比例會提高。
3. **`T-CHR-*` 特徵測試可能暫時固化舊 Bug**：在無 Spec 遺留區建立 Golden Master 雖能防止無意改壞，但同時也可能保護到原本錯誤的實作邏輯，故仍需依賴 `[STALE]` 提醒與人類定期審視轉正。
4. **風險公式受限於 Git 歷史與測試基準品質**：若專案長期採 Squash / 籠統 commit 訊息或無法辨識 `fix` 類型，風險公式之 `fix_density` 權重會歸零，退化為僅依賴變更次數、覆蓋缺口與人工標記之 `critical`。
5. **`shared-staging` 環境天然限制**：在無法取得隔離環境的專案中，高併發競態測試與 Stress 壓崩測試仍受限於共享環境安全性，無法在日常 PR 中無條件執行。

---

## 12. 待決事項（Open Questions）

1. **風險公式權重校正**：`scope.yml` 預設之 `0.30 / 0.25 / 0.25 / 0.20` 為暫定初始值，需在第一個實際導入的既存專案完成 Wave 2 後，對照實際線上漏網缺陷與命中率重新校準。
2. **Stack Adapter 實作優先序**：除 Nuxt / Next.js / Express 外，首波應優先支援哪幾種框架或非 Web 生態系（例如 FastAPI / Django / CLI 工具）？
3. **老化軟性警告門檻（`aging_warn_days`）**：`T-CHR-*` 與 `test.fail()` 預設 `30` 天觸發 `[STALE]` 警告是否合適，或應依專案發布節奏（例如以 Sprint / Release 數計算）？
4. **`test.fail()` 數量上限**：是否需要在 `gates.yml` 為單一專案的 `KNOWN_ISSUE` / `test.fail()` 總數設定上限（例如不得超過核心測試 10%），防止團隊過度把新發現的缺陷往 `defer-xfail` 堆放？
5. **Exit Code `7` 與 `web-cicd-pipeline.md` 同步**：`web-cicd-pipeline.md` §5.4 目前僅定義 Exit Code `0–6`；在實作導入腳本前，須確認 `web-cicd-pipeline.md` 是否同步納入 Exit Code `7`（`XPASS`），或將 `XPASS` 獨立回報於 `status` / PR 警示註記而不佔用總閘門 Exit Code。

---

## 13. 實作優先順序（Implementation Roadmap）

| 優先序 | 實作目標 | 驗收與完成條件 |
|---|---|---|
| **P1** | `ci/onboarding/` 四份契約解析器與 `onboard lint --gate A/B`、`triage --check` | 缺裁決碼、`assumed` 未確認、核可欄位空缺或未分類失敗時確定回傳 Exit `5` |
| **P2** | `recon.mjs`（含 commit 時間錨點、`--diff`/`--rebaseline`、第一個 `stacks/`）與 `baseline.mjs`（含 `--with-security`） | 同 `commit_sha` 執行兩次 `recon.json` SHA-256 一致；`baseline_run.json` 與 `security_baseline.json` 獨立輸出 |
| **P3** | `scaffold.mjs`（安全建新檔規則）、`lint-tests`（Gate 2 紅旗）、`status`（`[STALE]` 追蹤） | 絕不修改既有手寫測試檔；正確統計 `case_index.json` 與 AI 抽查清單 |
| **P4** | 分波漏斗執行器、Ratchet（Line＋Branch）檢核、`Exit 7 (XPASS)` 與 `handoff` / `G0 drift` | `test.fail()` 意外通過回傳 Exit `7` 且不計入 `stage_state.json` 修復迴圈；完整通過後無縫接軌 `web-cicd-pipeline.md` |
