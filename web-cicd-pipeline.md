# Web 系統 CI/CD 流程規範（草案 v0.2）

- **狀態**：草案（Draft），尚未實作
- **日期**：2026-11-20
- **依據文件**：
  - `docs/00_software-testing-types.md`：九種測試類型的層級、類型與觸發時機
  - `../Workflow-for-Agent/workflow.md`：五階段協作流程、閘門腳本、Exit Code、瑕疵工單規範
- **適用範圍**：以 Git 為核心、以容器或 Serverless 部署的 Web 系統
- **變更紀錄**：
  - v0.1：初版
  - v0.2：修復迴圈計數同時提供 `branch` 與 `release` 兩種範圍，可單用或併用（§8.6）；commit 格式改為先從歷史 commit 偵測專案慣例，只有慣例本身帶版本號時才要求 `[vX.Y.Z]` 前綴（§11.2）

---

## 0. 決策摘要

| # | 決策項目 | 結論 |
|---|---|---|
| 1 | Git 平台 | **GitHub 為主**；以 adapter 與 `render` 保留切換到 GitLab / Gitea / Forgejo 的空間 |
| 2 | 腳本語言 | **Node**（ESM `.mjs` + JSDoc `// @ts-check`） |
| 3 | 部署目標 | **容器為預設**；Serverless 以 deploy adapter 保留擴充空間 |
| 4 | 執行主體 | **以腳本為主**；AI agent 不進入管線，只在管線外撰寫程式與修復工單 |

---

## 1. 設計原則

1. **以腳本為主**：所有判定都由腳本的 Exit Code 搭配 `gates.yml` 決定。CI YAML 只是外殼，每個 job 只呼叫一行 `node ci/run.mjs <gate> --ci`。本地與 CI 執行同一支指令，結果一致。
2. **AI 不進入管線**：管線中沒有任何一步呼叫 AI，也不讓 AI 判定結果。workflow.md 中 R3 的 AI Reviewer 改為人工 PR 核可加上確定性檢查；AI review 可選，但只能留下 advisory 留言，不影響 Exit Code。
3. **Git 是唯一事實來源**：觸發、版本號、門檻、豁免、流程定義、權責名單全部放在 repo 內。Hooks 用 `core.hooksPath` 設定，不需要額外套件。
4. **平台差異收進 adapter**：Git 平台、工單系統、部署目標、CI 執行環境各有一個 adapter。gate 邏輯完全不知道底下是什麼平台，換平台時只需改設定與重新產生 CI YAML。
5. **先判定、後發佈**：Exit Code 在發 issue、留言、寫紀錄之前就已決定。平台 API 失敗只產生警告，不改變判定結果。
6. **Build once, deploy many**：只 build 一次，preview、staging、production 都部署同一個不可變的 artifact。
7. **測試理論對應**：Smoke 是入口閘門；Sanity 只用在 hotfix；Regression 橫跨各層（見 `docs/00_software-testing-types.md`）。

---

## 2. 流程總覽：Git 事件如何觸發各層管線

```
 開發者（人或 AI，在管線外）
   │ git commit
   ▼
 ┌─ L0 本地 Git Hooks（.githooks/，shell 殼呼叫 node）
 │   pre-commit : format · lint(staged) · gitleaks --staged
 │   commit-msg : 依專案慣例檢查（由歷史 commit 自動偵測，見 §11.2）
 │   pre-push   : typecheck · unit(changed) · version 一致性
 └─────────────────────────────────────────────────────────────
   │ git push feature/* → PR → dev
   ▼
 ┌─ L1 PR Pipeline（required check：ci-result）
 │   plan         依事件、分支、git diff 決定本次要跑哪些 gate
 │   G0 meta      commit 格式 · 版本一致 · spec-lint · trace-check · render --check
 │   G1 static    ESLint · TypeCheck · Stylelint                ┐
 │   G2 unit      Unit + Coverage                               │ 平行執行
 │   G3 integ     Integration（API / DB service）               │
 │   G4 security  Secrets · SAST · SCA · 豁免清單驗證           ┘
 │   G5 build     只 build 一次，輸出 ARTIFACT_ID
 │   G6 preview   臨時環境 → Smoke(@smoke)       ── 失敗即停止
 │   G7 system    E2E · Regression ×2（防抖）· a11y · Lighthouse · 視覺回歸
 │   ci-result    匯總所有 gate 結果（唯一的 required check）
 └─────────────────────────────────────────────────────────────
   │ ci-result 通過，且有 1 人 approve（CODEOWNERS）
   ▼ merge → dev
 ┌─ L2 Staging Pipeline
 │   deploy staging → Smoke → Full Regression → Load（k6 baseline）
 │   → 結果寫入 release ledger（供 L3 檢查）
 └─────────────────────────────────────────────────────────────
   │ release PR：dev → main
   ▼ tag vX.Y.Z
 ┌─ L3 Release Pipeline
 │   檢查該 commit 的 staging 紀錄（Regression 與 Load 都必須通過）
 │   → image build → image scan → Stress（k6）
 │   → 人工核可（environment protection）
 │   → deploy production（blue/green 或 canary）→ Prod Smoke
 │   → 健康檢查連續 3 次失敗：自動 rollback
 └─────────────────────────────────────────────────────────────

 ┌─ Hotfix（從 main 切出 hotfix/*）
 │   PR → main：G0–G5 → Sanity（依 impact-map 選測）→ Selective Regression
 │   merge → tag → L3（略過 Stress）→ Prod Smoke
 │   → 自動開 PR 回併 dev
 └─────────────────────────────────────────────────────────────

 ┌─ Nightly（schedule，UTC 18:00 = 台灣 02:00）
 │   Full Regression（跨瀏覽器）· Load · 相依套件 CVE 重掃 · flaky 統計
 └─────────────────────────────────────────────────────────────
```

---

## 3. 測試類型與管線位置

| 測試類型 | 層級 | 觸發時機 | 建議工具 | 判定方式 | 是否阻擋 |
|---|---|---|---|---|---|
| Unit | Unit | pre-push（只跑變更相關）、G2 | Vitest + v8 coverage | 通過率 100%；line ≥ 80、core ≥ 90 | 是 |
| Integration | Integration | G3 | Vitest + service container（DB / Redis）、API contract | 通過率 100% | 是 |
| Smoke | System | 每次部署後（preview / staging / prod） | Playwright `@smoke`（≤ 15 條、≤ 5 分鐘） | 通過率 100% | 是，失敗即停止後續 |
| Sanity | System（局部） | hotfix | 依 impact-map 選出對應 tag | 通過率 100% | 是 |
| E2E | System | G7、staging | Playwright `@e2e`，`retries: 0` | 通過率 100% | 是 |
| Regression | 跨各層 | PR 全量、hotfix 選擇性、staging、nightly | `@regression` 加上歷史 bug 測試 | 連跑 2 次結果一致，0 flaky | 是 |
| System（相容性） | System | nightly | 跨瀏覽器 matrix、RWD viewport | 通過率 100% | 開 issue，並阻擋 release |
| Load | System | staging 合併後（短版）、nightly | k6 thresholds | p95 ≤ NFR、錯誤率 ≤ 1% | 阻擋 release |
| Stress | System | release 前（hotfix 略過） | k6 逐步加壓到崩潰點 | 崩潰點 ≥ NFR 容量 × 1.5 | 阻擋 release |
| Usability（自動化指標） | Acceptance | G7 | axe-core、Lighthouse CI、Playwright 視覺回歸 | a11y 0 serious/critical；LH a11y ≥ 90 | 是 |
| Usability（真人測試） | Acceptance | 每個 minor release 前 | 人工 session，以 issue template 記錄 | 人工判定 | 不在管線內 |

**與 workflow.md R1–R3 的對應**：

| workflow.md | 本規範 | 說明 |
|---|---|---|
| R1 功能正確性 | G2 Unit 加上正常類 AC 的 E2E | 測試名稱帶 `T-R1-AC-xx` |
| R2 邊界與異常 | G3 Integration 加上異常類 AC 與 EC 測試 | 測試名稱帶 `T-R2-AC-xx`、`T-R2-EC-xx` |
| R3 回歸與驗收 | G7 Regression 連跑 2 次、AC 勾選表 | AI 獨立審查改為人工 PR 核可（見 §12） |

**補充**：
- 視覺回歸的 snapshot 一律在 Linux 容器內產生與比對，避免不同作業系統的字型渲染差異造成誤判。
- G7 連跑 2 次都使用同一份 artifact。兩次之間只要有任何一個測試結果不一致，就判定為 flaky（exit 3）。

---

## 4. 架構分層與 adapter

```
                      ci/pipeline.yml（唯一的流程定義，與平台無關）
                                 │
                     node ci/run.mjs render
                                 │
          ┌──────────────────────┼───────────────────────┐
          ▼                      ▼                       ▼
   .github/workflows/*     .gitlab-ci.yml（保留）   .gitea/workflows/*（保留）
          │  每個 job 都只有：node ci/run.mjs <gate> --ci
          ▼
 ┌─ ci/run.mjs ──────────────────────────────────────────────
 │  ctx       CI 執行環境：事件、base/head、sha、run URL、PR 編號、輸出變數
 │  platform  Git 平台：PR 留言、Release、部署紀錄、保護規則、核可
 │  issues    工單系統：預設沿用 platform，可單獨改成 Jira
 │  deploy    部署目標：container（預設）或 serverless（保留）
 └───────────────────────────────────────────────────────────
          │
          ▼
   gates / parsers / defect / impact / plan  ← 這一層完全不知道底下是哪個平台
```

| Adapter | 預設 | P1 實作 | 保留擴充 |
|---|---|---|---|
| `ctx` | 依環境變數自動偵測 | github、local | gitlab、gitea |
| `platform` | github | github、local | gitlab、gitea/forgejo |
| `issues` | `inherit`（沿用 platform） | github、local | gitlab、gitea、jira |
| `deploy` | container | compose（P2） | k8s（P4）、serverless（P6） |

**換平台時要改的地方**：

| 項目 | 換 Git 平台 | 換部署目標 |
|---|---|---|
| `ci/pipeline.yml`、`ci/gates/*`、`tests/**` | 不用改 | 不用改 |
| `gates.yml` 的測試門檻 | 不用改 | 不用改 |
| CI YAML | 重新執行 `render` | 不用改 |
| `platform.provider`、平台 token | **要改** | 不用改 |
| `deploy.target` 與 provider 設定、部署 secrets | 不用改 | **要改** |
| 框架 build 輸出設定（如 Nitro preset、SvelteKit adapter、Next `output`） | 不用改 | **要改** |

---

## 5. 腳本規格

### 5.1 執行環境與依賴

| 項目 | 規格 |
|---|---|
| Runtime | Node 24 LTS，由 `.node-version` 與 `engines` 固定版本 |
| 入口 | `node ci/run.mjs <gate>`（別名 `pnpm ci:run <gate>`） |
| 套件管理 | pnpm（`corepack enable`，以 `--frozen-lockfile` 安裝） |
| 寫法 | `.mjs` 加 JSDoc，檔頭標 `// @ts-check`；不需 build，也能被 TypeCheck 檢查 |
| 第三方依賴 | 只有 `yaml`（固定版本） |
| 內建模組 | HTTP 用 `fetch`；hash 用 `node:crypto`；參數解析用 `node:util` 的 `parseArgs`；子行程用 `node:child_process` |
| 子行程規則 | `spawn` 一律 `shell: false`，並設定 timeout |
| Exit Code | 設定 `process.exitCode`，不呼叫 `process.exit()`，避免輸出被截斷 |
| 腳本本身的測試 | `node --test tests/ci/` |

**CI 工具與 App 的依賴分開管理**：`ci/` 是獨立的 pnpm workspace 套件，有自己的 `package.json`，只依賴 `yaml`。App 升級套件不會影響閘門腳本，`ci/**` 也在 CODEOWNERS 保護範圍內。

### 5.2 目錄結構

```
repo/
├── .node-version
├── package.json                "prepare": "git config core.hooksPath .githooks"
├── pnpm-workspace.yaml         packages: [".", "ci"]
├── gates.yml                   門檻與 adapter 設定（受 CODEOWNERS 保護）
├── .security-allowlist.yml     資安豁免（受 CODEOWNERS 保護）
├── .githooks/                  pre-commit · commit-msg · pre-push
├── .github/                    由 render 產生：workflows/ · CODEOWNERS · ISSUE_TEMPLATE/
├── ci/
│   ├── package.json            只依賴 yaml
│   ├── run.mjs                 單一入口
│   ├── bootstrap.sh            安裝固定版本的非 Node 工具並驗證 sha256
│   ├── tools.lock.json         非 Node 工具的版本與 sha256
│   ├── pipeline.yml            唯一的流程定義
│   ├── policy.yml              分支保護、核可人數、required check、環境
│   ├── owners.yml              CODEOWNERS 單一來源
│   ├── impact-map.yml          路徑 → 測試 tag
│   ├── overrides/<platform>/   平台專屬寫法（必要時才使用）
│   ├── render/                 github.mjs（P1）· gitlab.mjs · gitea.mjs（P7）
│   ├── gates/                  g0-meta · g1-static · g2-unit · g3-integ · g4-security
│   │                           g5-build · g6-smoke · g7-system · sanity · load · stress
│   │                           deploy · rollback · ci-result
│   └── lib/
│       ├── exitcodes.mjs
│       ├── config.mjs          讀取並驗證 gates.yml
│       ├── plan.mjs            事件、分支、diff → 要執行的 gate 清單
│       ├── impact.mjs          git diff → impact-map → 測試 tag
│       ├── parsers/            vitest · playwright · eslint · sarif · audit · k6 · lhci · axe
│       ├── defect.mjs          解析結果 → 標準化 defect JSON
│       ├── release.mjs         release manifest 的產生與讀寫
│       ├── ctx/                github · local（P1）· gitlab · gitea（P7）
│       ├── platform/           contract · github · local（P1）· gitlab · gitea（P7）
│       ├── issues/             contract · github · local（P1）· gitlab · gitea · jira（P7）
│       └── deploy/             contract · index · container/compose（P2）· container/k8s（P4）
│                               serverless/vercel …（P6）
├── deploy/
│   ├── Dockerfile              multi-stage，以 non-root 執行
│   ├── compose.yml             preview 與 staging 用
│   ├── compose.bluegreen.yml   反向代理（Caddy / Traefik）切換 blue/green
│   └── k8s/                    保留：kustomize base 與 overlays
└── tests/
    ├── unit/ · integration/ · e2e/ · load/
    └── ci/                     腳本測試與 contracts/
```

### 5.3 腳本約定

- **輸入**：`gates.yml`、git 狀態、`ctx` 提供的環境資訊。
- **輸出**：
  - Exit Code
  - `.scratch/reports/<gate>.json`：標準化結果
  - `.scratch/logs/<gate>_<時間戳>.*`：原始輸出
- **只解析機器格式**：JSON 與 SARIF。不以文字比對來判定結果。
- **未列入計畫的 gate**：讀取 plan 結果，若本 gate 不在計畫內，以 exit 0 加上 `skipped` 狀態結束。
- **工具錯誤自動重跑**：遇到 exit 6 時，腳本自己重跑一次（不依賴 CI 平台的 retry 功能）；仍失敗才回傳 6。
- **artifact 一致性**：G5 記錄 `ARTIFACT_ID`，之後每次部署前都核對 hash，不一致就 exit 6。

### 5.4 Exit Code 與路由

沿用 workflow.md 的定義。多項同時成立時，回傳優先序最高者，但所有工單都會寫入。

| Code | 意義 | 優先序 | Issue label | 後續處理 |
|---|---|---|---|---|
| 0 | 通過 | — | — | 自動關閉對應 issue |
| 1 | 測試失敗 / 弱點 / 審查重大問題 | 4 | `route:dev` | 開發者修復 |
| 2 | 覆蓋率或 EC 對應率不足 | 5 | `route:test-design` | 補測試；判定為死碼則移除 |
| 3 | flaky（兩次結果不一致） | 3 | `route:test-design` | 判定歸屬：測試 / 實作 / 環境 / 規格 |
| 4 | 同一瑕疵累計 3 次修復未通過 | 2 | `route:test-design` + `escalated` | 根因分析 |
| 5 | 達到停止條件 | 1 | `route:owner` | 人工裁決 |
| 6 | 工具或環境錯誤（已自動重跑 1 次） | 0（最高） | `route:infra` | 排除環境問題後重跑 |

### 5.5 各工具的報表格式

Node 沒有內建 XML parser，所以一律使用各工具的 JSON 輸出。

| 來源 | 輸出格式 |
|---|---|
| Vitest | `--reporter=json` |
| Coverage（v8） | `json-summary` |
| Playwright | `--reporter=json` |
| ESLint | `-f json` |
| gitleaks / semgrep / trivy | SARIF 或 JSON |
| `pnpm audit` | `--json` |
| k6 | `--summary-export` |
| Lighthouse CI / axe | JSON |

### 5.6 非 Node 工具

| 工具 | 安裝方式 |
|---|---|
| gitleaks、trivy、k6 | `ci/bootstrap.sh` 下載固定版本的 binary，以 `ci/tools.lock.json` 的 sha256 驗證 |
| semgrep | 本身需要 Python，改用官方 Docker image（固定 tag 與 digest）以 `docker run` 執行 |
| Playwright 瀏覽器 | `pnpm exec playwright install --with-deps <browsers>`，版本跟著 lockfile |
| Lighthouse CI、axe | npm 套件 `@lhci/cli`、`@axe-core/playwright`，列在 App 的 devDependencies |

---

## 6. 流程定義與 CI 外殼

### 6.1 `ci/pipeline.yml`

```yaml
pipelines:
  pr:
    on: { change_request: [dev, main] }
    jobs:
      - plan
      - { id: g0-meta, needs: [plan] }
      - { parallel: [g1-static, g2-unit, g3-integ, g4-security], needs: [g0-meta] }
      - { id: g5-build,  needs: [g1-static, g2-unit, g3-integ, g4-security] }
      - { id: g6-smoke,  needs: [g5-build], env: preview }
      - { id: g7-system, needs: [g6-smoke], env: preview }   # hotfix 時由 plan 改跑 sanity
      - { id: ci-result, needs: all, always: true }
  staging:
    on: { push: [dev] }
    jobs: [plan, g5-build, deploy-staging, g6-smoke, regression-full, load, ledger-write]
  release:
    on: { tag: "v*.*.*" }
    jobs: [plan, ledger-check, g5-build, image-scan, stress, approve, deploy-prod, g6-smoke, ledger-write]
  nightly:
    on: { schedule: "0 18 * * *" }
    jobs: [plan, regression-matrix, load, sca-rescan, flaky-stats]
  maintenance:
    on: { manual: { inputs: [action, scope, ref, reason] } }   # GitHub 對應 workflow_dispatch
    jobs: [counters]                                           # 例如 counters reset（§8.6）
artifacts: { path: .scratch/, retention_days: 14 }
```

### 6.2 產生 CI YAML

- `node ci/run.mjs render --platform github` 產生 `.github/workflows/*.yml`、`.github/CODEOWNERS`、`.github/ISSUE_TEMPLATE/`。
- **G0 會執行 `render --check`**：產出的檔案與 repo 內容不一致就 exit 1，避免有人直接手改 YAML。
- 需要平台專屬寫法時，放在 `ci/overrides/<platform>/`，產生時合併，並在產出檔案中標示來源。
- 產生 YAML 時，若平台有原生的條件語法（例如 GitHub 的 `if:`），會加上條件，省下啟動 runner 的時間。沒有這類語法的平台，則由腳本自行略過（見 §5.3）。

### 6.3 唯一的 required check：`ci-result`

`ci-result` 匯總所有 gate 的結果，各平台的保護規則只需要指向這一個 check，不必隨 gate 增減而改設定。

| 平台 | 對應設定 |
|---|---|
| GitHub | Rulesets 的 required status check：`ci-result` |
| GitLab | 「Pipelines must succeed」 |
| Gitea / Forgejo | Branch protection 的 status check pattern：`ci-result` |

### 6.4 產出範例：`.github/workflows/pr.yml`（節錄）

```yaml
# 由 node ci/run.mjs render --platform github 產生，請勿手動修改
name: pr
on:
  pull_request:
    branches: [dev, main]
permissions:
  contents: read
concurrency:
  group: pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  plan:
    runs-on: ubuntu-latest
    outputs:
      plan: ${{ steps.p.outputs.plan }}
    steps:
      - uses: actions/checkout@<sha>          # v4，第三方 action 一律以 commit SHA 固定
        with: { fetch-depth: 0 }
      - uses: actions/setup-node@<sha>
        with: { node-version-file: .node-version }
      - run: ./ci/bootstrap.sh
      - id: p
        run: node ci/run.mjs plan --ci        # 由 ctx 寫入 $GITHUB_OUTPUT

  gate:
    needs: plan
    strategy:
      fail-fast: false
      matrix:
        gate: [g1-static, g2-unit, g3-integ, g4-security]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>
        with: { fetch-depth: 0 }
      - uses: actions/setup-node@<sha>
        with: { node-version-file: .node-version }
      - run: ./ci/bootstrap.sh
      - run: node ci/run.mjs ${{ matrix.gate }} --ci
      - if: always()
        uses: actions/upload-artifact@<sha>
        with: { name: "${{ matrix.gate }}", path: .scratch/, retention-days: 14 }

  # g0-meta、g5-build、g6-smoke、g7-system 以 needs 依序串接，每個 job 同樣只有一行 run.mjs

  ci-result:
    if: always()
    needs: [plan, gate]                       # 實際產出會列出全部 job
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>
      - uses: actions/setup-node@<sha>
        with: { node-version-file: .node-version }
      - run: node ci/run.mjs ci-result --ci
        env:
          CI_NEEDS: ${{ toJSON(needs) }}
```

---

## 7. Git 平台 adapter

### 7.1 介面

| 方法 | 用途 |
|---|---|
| `capabilities()` | 宣告平台支援哪些能力，供 G0 選擇降級方式 |
| `summary(markdown)` | 輸出本次執行的結果摘要 |
| `comment(changeRequest, body)` | 在 PR / MR 留言 |
| `openChangeRequest(head, base, title, body)` | 開 PR（用於 hotfix 回併 dev） |
| `approvals(changeRequest)` | 查詢核可人數與核可者 |
| `release.create(tag, notes, assets)` | 建立 Release 並附加檔案 |
| `ledger.write(env, manifest)` / `ledger.read(env, n)` | 讀寫部署紀錄 |
| `meta.read(changeRequest)` / `meta.write(changeRequest, data)` | 讀寫 PR 上的計數資料，以帶標記的專用留言保存（見 §8.6） |
| `applyPolicy(policy)` | 套用分支保護、核可、環境設定 |

### 7.2 各平台能力與降級方式

| 能力 | GitHub（P1） | GitLab（P7） | Gitea / Forgejo（P7） | 平台不支援時的降級方式 |
|---|---|---|---|---|
| 結果摘要 | Step Summary | MR 留言 | PR 留言 | `.scratch/summary.md` 作為 artifact |
| Release 與 asset | Releases API | Releases API | Releases API | 只用 git tag |
| 部署紀錄 | Deployments API | Environments / Deployments API | 無 | **git notes `refs/notes/releases`**，只要有 git 就能用 |
| 正式環境人工核可 | Environment 的 required reviewers | Protected environments（需付費方案） | 無 | 手動觸發部署；腳本檢查觸發者是否在 `ci/owners.yml` 名單中 |
| PR 強制核可 | Rulesets | Approval rules（需付費方案） | Branch protection | `ci-result` 以 API 查詢核可人數，不足就 exit 1；核可後需重跑 `ci-result` |
| 保護規則 | Rulesets | Protected branches | Branch protection | `apply-policy` 輸出設定清單，交由人工設定 |
| CODEOWNERS | `.github/CODEOWNERS` | `CODEOWNERS` | `CODEOWNERS` | 由 `ci/owners.yml` 產生 |
| Issue template | `.github/ISSUE_TEMPLATE/` | `.gitlab/issue_templates/` | `.gitea/ISSUE_TEMPLATE/` | 由同一份來源產生 |
| Container registry | GHCR | GitLab Registry | Gitea Packages | 在設定中手動指定 |

G0 會比對 `platform.capabilities()` 與設定。例如設定要求正式環境核可、但平台不支援時，會自動改用降級方式，並在 summary 中寫明改用了哪一種做法。

### 7.3 `ci/policy.yml`

```yaml
branches:
  dev:  { require_change_request: true, required_checks: [ci-result], approvals: 1, codeowners: true, allow_force_push: false }
  main: { require_change_request: true, required_checks: [ci-result], approvals: 1, codeowners: true, allow_force_push: false }
environments:
  preview:    { approval: none }
  staging:    { approval: none, branches: [dev] }
  production: { approval: required, reviewers_from: owners.release, branches: [main], tags: ["v*.*.*"] }
```

由 `node ci/run.mjs platform apply-policy` 套用。這個指令需要 admin 權限，只需執行一次；之後修改 `policy.yml` 時再重新套用。

### 7.4 GitHub 實作重點

| 項目 | 規格 |
|---|---|
| Token 權限 | 預設 `contents: read`。staging、release、nightly 才開 `issues: write`、`deployments: write`；PR 只開 `pull-requests: write`（選用留言） |
| Token 名稱 | 腳本統一讀取 `CI_PLATFORM_TOKEN`，由 render 把它對應到 `GITHUB_TOKEN` |
| Fork PR | fork 來的 PR 拿到的 token 是唯讀的；本規範本來就不在 PR 階段開 issue，兩者自然相容 |
| `concurrency` | 同一個 PR 有新 push 時，取消舊的執行 |
| SARIF → Code Scanning | 選用。private repo 需要 GHAS 授權；判定仍以腳本 Exit Code 為準 |
| Action 版本 | 第三方 action 一律以 commit SHA 固定 |

---

## 8. Issue 多平台設計

### 8.1 資料流

```
 gate 腳本 → 標準化 defect JSON → IssueRouter → issues adapter
                                     │
          provider: inherit → 沿用 platform.provider；也可指定 jira / local
                                     │
   ┌──────────┬──────────┬───────────────┬──────────┬───────────────────────┐
   github     gitlab     gitea/forgejo   jira       local（備援）
   REST       REST v4    REST v1         REST v3    .scratch/issues/*.md + defects.jsonl
```

### 8.2 標準化 defect 欄位

| 欄位 | 說明 |
|---|---|
| `fingerprint` | `sha256(gate + test_id + 正規化後的斷言或 rule_id + 檔案)`，用來去重 |
| `ticket_id` | `DEF-{gate}-{YYYYMMDD}-{fingerprint 前 6 碼}` |
| `kind` | `FAIL` / `ERROR` / `VULN` / `PERF` / `A11Y` |
| `gate`、`test_type`、`test_id` | 例如 `G7`、`e2e`、`T-R1-AC-03` |
| `spec_ref` | `{ fr, ac, ec }`，對應 `docs/01_spec.md` |
| `location` | `檔案:行號` |
| `message` | 精簡錯誤訊息或斷言差異 |
| `severity` | `Critical` / `High` / `Medium` / `Low` |
| `route` | `dev` / `test-design` / `owner` / `infra` |
| `retry_count` | 同一瑕疵的累計修復次數 |
| `commit`、`branch`、`run_url` | 追溯資訊 |
| `repro` | 重現指令，例如 `node ci/run.mjs g7-system --grep "T-R1-AC-03"` |
| `snippet` | 失敗處前後 25–50 行 |
| `status` | `OPEN` / `IN_PROGRESS` / `RESOLVED` / `CLOSED` |

**工單編號的變更**：workflow.md 使用 `DEF-3B-YYYYMMDD-XXX` 流水號。平行執行的 CI 會讓流水號衝突，所以改用 fingerprint 前 6 碼。

### 8.3 Adapter 介面

`find(fingerprint)`、`create(defect)`、`update(issue, defect)`（留言並更新計數）、`close(issue, evidence)`、`ensureLabels()`。

### 8.4 各平台差異

| 項目 | GitHub | GitLab | Gitea / Forgejo | Jira | local |
|---|---|---|---|---|---|
| Label 格式 | `stage:G7` | `stage::G7`（scoped） | `stage/G7` | `stage-G7` | 欄位 |
| 去重查詢 | 列出所有 `ci-defect` open issue，本地比對標題中的 `[fp:xxxxxx]` | `?search=` | `?q=` | JQL `labels = fp-xxxxxx` | grep |
| Metadata 存放 | body 結尾的隱藏註解 | body 隱藏註解 | body 隱藏註解 | 自訂欄位或 label | JSON |
| 認證 | `GITHUB_TOKEN` | Project Access Token（`CI_JOB_TOKEN` 不能建立 issue） | `GITEA_TOKEN` | email + API token | 不需要 |

**GitHub 去重不使用 Search API**：Search API 的認證上限是每分鐘 30 次，失敗多時容易撞到。改為每次執行時一次列出所有 `ci-defect` open issue（分頁取完），在本地比對。

**Metadata 格式**：`<!-- ci-defect-meta {"fp":"a1b2c3","retry":2,"first_seen":"…"} -->`

**Label 規格**（由 `node ci/run.mjs issues ensure-labels` 一次建立）：
- `ci-defect`
- `stage:G0`–`stage:G7`、`stage:load`、`stage:stress`
- `route:dev`、`route:test-design`、`route:owner`、`route:infra`
- `severity:critical`、`severity:high`、`severity:medium`、`severity:low`

### 8.5 行為規則

1. **PR 失敗不開 issue**：只寫結果摘要與 artifact（可選擇加 PR 留言）。只有 dev、main、nightly、release 失敗才開 issue（`issues.open_on`）。
2. **重複發生時沿用原 issue**：以 fingerprint 比對，`retry_count` 加 1，並留言附上本次的 run URL 與 snippet。
3. **升級與停止**：`retry_count` 達到 `escalate_after`（3）時 exit 4；達到 `stop_same_defect`（6）時 exit 5。跨瑕疵的累計修復迴圈另外計數，見 §8.6。
4. **自動關閉**：staging 或 nightly 通過時，若某個 open issue 的 test_id 本次有執行且通過，就附上 commit SHA 與 run URL 關閉。
5. **平台 API 失敗**：只發警告，把 local issue 上傳成 artifact，不改變 Exit Code。
6. **Issue template 只維護一份**：`node ci/run.mjs sync-templates` 產生各平台的 template，其中包含真人 Usability 測試的回報格式。

### 8.6 修復迴圈計數（branch / release 兩種範圍）

workflow.md 規則 7 規定「每次退回都計入累計修復迴圈」，但不同專案對「累計」的範圍需求不同。因此本規範同時提供兩種範圍，由 `issues.loop_limits.total.scope` 選擇，可以單用，也可以併用。這組計數與單一瑕疵的計數（`escalate_after`、`stop_same_defect`）互相獨立。

**兩種範圍的比較**：

| 項目 | `branch` | `release` |
|---|---|---|
| 計數範圍 | 單一分支，從第一次 PR 執行到合併或關閉 | 上一次 production 部署成功之後，到下一次部署成功為止 |
| 計入事件 | 該分支 PR pipeline 的 `ci-result` 為 exit 1–4 | dev、staging、nightly pipeline 的 `ci-result` 為 exit 1–4 |
| 不計入 | exit 0、5、6；同一個 commit 重跑 | exit 0、5、6；同一個 commit 重跑；PR pipeline 的失敗 |
| CI 模式存放位置 | PR 上帶標記的專用留言（`platform.meta`） | 追蹤 issue `[ci-cycle] since vX.Y.Z`（label `ci-cycle`），計數存在 metadata |
| 降級存放位置（API 不可用時） | 不計數，`ci-result` 在摘要中註明 | git notes `refs/notes/ci-loops` |
| local 模式存放位置 | `docs/stage_state.json` 的 `loops.branch.<分支名>` | `docs/stage_state.json` 的 `loops.release` |
| 達到上限時 | `ci-result` 回傳 exit 5，PR 被擋下並標 `route:owner` | 追蹤 issue 標 `route:owner`；L3 的 `ledger-check` 回傳 exit 5，擋下 release |
| 自動歸零 | 分支合併或關閉 | production 部署成功：關閉目前的追蹤 issue，下一次計數時開新的 |
| 人工歸零 | owner 執行 `counters reset`（見下方） | 同左 |

追蹤 issue 使用獨立的 `ci-cycle` label，不帶 `ci-defect`，因此不會被 §8.5 的瑕疵自動關閉規則誤關。

**選擇建議**：

| 專案型態 | 建議 scope | 原因 |
|---|---|---|
| 依 workflow.md 逐一交付功能、分支壽命較長 | `[branch]` | 等同 workflow.md「單一階段內的累計」，問題能對應到單一功能 |
| 多分支頻繁合併到 dev、接近 trunk-based | `[release]` | 個別分支壽命短，問題多出現在整合之後，需要看整體 release 品質 |
| 外包或多團隊協作，各自負責分支但共用 release | `[branch, release]` | 分支層級追蹤個別交付，release 層級把關整體；兩者各自計數，先達上限者觸發 exit 5 |

**計數規則**：
1. 只有 `ci-result` 會更新計數，各個 gate 不直接寫入，避免重複計算。
2. 以「commit SHA + 範圍」去重：同一個 commit 重跑（例如排除環境問題後重跑）不會重複計數。metadata 會保留最近已計數的 SHA 清單。
3. exit 5 本身不再計數。exit 6 是工具或環境錯誤，不代表程式碼被退回，也不計數。
4. 要計入哪些 Exit Code 由 `count_exit_codes` 設定，預設與 workflow.md 相同（1–4）。

**人工歸零**：

```
node ci/run.mjs counters reset --scope branch|release [--ref <分支名>] --reason "<裁決說明>"
```

- **CI 模式**：由 `maintenance` pipeline 手動觸發（GitHub 為 `workflow_dispatch`）。腳本檢查 `ctx.actor` 是否在 `ci/owners.yml` 的 `loop_reset` 名單中，不在名單中就 exit 1。
- **稽核紀錄**：每次歸零都在 PR 或追蹤 issue 留言，記錄執行者、歸零前的數值、裁決說明與時間。
- **local 模式**：直接修改 `docs/stage_state.json`。這個檔案納入版控並受 CODEOWNERS 保護，所以歸零會出現在 PR diff 中，必須經過核可。

---

## 9. 部署 adapter

### 9.1 核心想法：部署以外的環節只依賴兩個值

```
            ┌─ deploy adapter（唯一跟部署平台有關的地方）
            │
 G5 build ──┤  container : OCI image  → ghcr.io/…@sha256:…
            │  serverless: prebuilt bundle → 平台 deployment id
            │
            └─▶ 對外只輸出兩個值：
                  ARTIFACT_ID  （不可變的產出物識別碼）
                  BASE_URL     （部署後的網址）
                       │
     ┌─────────────────┼──────────────────────────┐
     ▼                 ▼                          ▼
  Smoke / Sanity    E2E / Regression       Load / Stress / a11y / Lighthouse
  （只需要 BASE_URL，完全不知道部署平台是什麼）
```

### 9.2 介面

| 方法 | container（預設） | serverless（保留） |
|---|---|---|
| `capabilities()` | 支援 rolling、blue-green、canary | 依各 provider 而定 |
| `build()` | `docker buildx build`，推送到 registry，回傳 digest | 框架以 serverless 輸出 build，執行 `<provider> build`，回傳 bundle hash |
| `deploy(env, artifact)` | compose 或 k8s 套用指定 digest，回傳 URL | `<provider> deploy --prebuilt`，回傳 URL |
| `promote(env, strategy)` | 切換 blue/green 流量，或依比例放大 canary | alias 或網域切換，或平台的漸進釋出功能 |
| `rollback(env)` | 讀取上一筆 release 紀錄，重新部署前一個 digest | 讀取上一筆 release 紀錄，切回前一個 deployment |
| `preview.up()` / `preview.down()` | 在 CI runner 內 `docker compose up`，`BASE_URL=http://localhost:3000` | 平台原生的 preview deployment |
| `artifactScan()` | `trivy image`（阻擋 High / Critical） | 略過（SCA 已在 G4 掃過 lockfile） |

**策略與能力的檢查**：G0 比對 `deploy.strategy` 與 adapter 的 `capabilities()`，設定了不支援的策略就 exit 1，不會等到部署時才失敗。

### 9.3 各管線環節的差異

| 管線環節 | container | serverless |
|---|---|---|
| G5 build | image + digest | prebuilt bundle + hash |
| G6 preview | runner 內 compose | 平台 preview URL |
| L2 staging | compose 或 k8s 部署 | provider 部署到 staging alias |
| L3 image scan | `trivy image` | 略過 |
| L3 Stress | 打 staging | 打 staging，並受 `stress.max_cost_guard` 限制（Serverless 依用量計費且有 rate limit） |
| L3 promote | blue/green 切換或 canary 逐步放量 | alias 切換或平台漸進釋出 |
| 自動 rollback | 部署前一個 digest | 切回前一個 deployment |
| Smoke / Regression / Load / a11y | 相同（只依賴 `BASE_URL`） | 相同 |

### 9.4 Release 紀錄（rollback 依據）

- 每次部署時，`ci/lib/release.mjs` 產生 `release-manifest.json`：

  ```json
  {
    "version": "v1.4.0",
    "commit": "<sha>",
    "env": "production",
    "target": "container",
    "provider": "compose",
    "artifact_id": "ghcr.io/<owner>/<repo>@sha256:…",
    "gates": { "regression": "pass", "load": "pass", "stress": "pass" },
    "gates_hash": "<sha256>",
    "deployed_at": "2026-11-20T10:00:00Z",
    "run_url": "…"
  }
  ```

- 寫入 `platform.ledger`：GitHub 使用 Deployments API（manifest 放在 payload）。不支援的平台降級為 git notes `refs/notes/releases`。
- production 的 manifest 另外附加為 Release asset，永久留存。
- `rollback(env)` 與 L3 的 `ledger-check` 都只讀這份紀錄，不向部署平台查歷史，所以換平台時邏輯不必改。
- 沒有 token 時降級為 local 模式，寫到 `.scratch/releases/`。

---

## 10. 設定檔

### 10.1 `gates.yml`

```yaml
version: 1

platform:
  provider: github                  # github | gitlab | gitea | local
  detect: [config, ci-env, git-remote]
  registry: auto                    # 依平台自動對應：GHCR / GitLab Registry / Gitea Packages

commit:                             # 見 §11.2
  convention: auto                  # auto | workflow-version | conventional-emoji | conventional | gitmoji | custom
  custom_regex: null                # convention: custom 時必填
  types: [feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert]
  detect:
    base: default-branch            # 偵測時取樣的分支
    samples: 50                     # 取最近 N 筆非 merge commit
    min_samples: 10                 # 少於此數視為「沒有歷史」
    min_ratio: 0.8                  # 單一慣例需達到的比例
    fallback: workflow-version      # 沒有歷史時採用的慣例

spec:
  required: true                    # 依 workflow.md 開發的專案須有 docs/01_spec.md
  path: docs/01_spec.md
  vague_terms: [TBD, 待定, 適當, 盡量, 快速, 合理, 大約, 若干, 等等, "...", etc, appropriate]

unit:        { coverage: { line_min: 80, core_min: 90, core_modules: [src/server/] } }
integration: { services: [postgres, redis] }
smoke:       { tag: "@smoke", max_duration_s: 300 }
e2e:         { tag: "@e2e", browsers: [chromium], retries: 0 }
regression:  { tag: "@regression", runs: 2 }
sanity:      { impact_map: ci/impact-map.yml, unmapped: full-regression }
compat:      { browsers: [chromium, firefox, webkit], viewports: [375x812, 1440x900] }
a11y:        { standard: wcag21aa, block_impact: [critical, serious] }
lighthouse:  { min: { performance: 80, accessibility: 90, best-practices: 90 } }
visual:      { max_diff_pixel_ratio: 0.01 }
load:        { script: tests/load/baseline.js, p95_ms: 500, error_rate_max: 0.01 }
stress:      { script: tests/load/stress.js, min_capacity_ratio: 1.5, max_cost_guard: null }

security:
  block_severity: [Critical, High, Medium]
  allowlist: .security-allowlist.yml   # 每筆需有理由、核可者、到期日；過期即失效

deploy:
  target: container                 # container | serverless
  strategy: blue-green              # rolling | blue-green | canary（由 G0 驗證是否支援）
  container:
    registry: auto
    runtime: compose                # compose（預設）| k8s
    dockerfile: deploy/Dockerfile
    preview: runner-compose
    canary: { steps: [10, 50, 100], hold_s: 300 }
  serverless:                       # 保留：介面與合約測試先完成，provider 之後實作
    provider: vercel                # vercel | netlify | cloudflare | aws-lambda
    prebuilt: true                  # 堅持 build once：CI 自己 build，平台只負責部署
  environments:
    preview:    { url: from-adapter }
    staging:    { url: https://staging.example.com }
    production: { url: https://example.com }
  health:   { path: /api/health, expect: 200, consecutive: 3, timeout_s: 60 }
  rollback: auto

issues:
  provider: inherit                 # inherit | jira | local
  open_on: [dev, main, nightly, release]
  loop_limits:
    escalate_after: 3               # 單一瑕疵：升級門檻（exit 4）
    stop_same_defect: 6             # 單一瑕疵：停止門檻（exit 5）
    total:                          # 累計修復迴圈（見 §8.6）
      scope: [branch]               # [branch] | [release] | [branch, release]
      branch:  { max: 10 }
      release: { max: 30 }
      count_exit_codes: [1, 2, 3, 4]
```

### 10.2 `ci/impact-map.yml`

Sanity 與選擇性 Regression 使用：

```yaml
- paths: ["src/pages/checkout/**", "src/server/api/order/**"]
  tags:  ["@checkout", "@order"]
- paths: ["src/server/api/auth/**"]
  tags:  ["@auth"]
- paths: ["package.json", "pnpm-lock.yaml", "*.config.{js,mjs,ts}", "deploy/**"]
  tags:  ["@regression"]     # 影響全域，跑全量
# 對應不到的路徑一律跑全量 Regression，寧可多跑也不漏跑
```

### 10.3 `.security-allowlist.yml`

```yaml
- id: CVE-2026-12345              # 規則 ID 或 CVE
  path: pnpm-lock.yaml
  reason: 僅影響 server-side render 以外的路徑，已確認未使用
  approved_by: <owner>
  expires: 2027-01-31
```

缺少任一欄位或已過期的豁免一律無效，該項重新阻擋。

---

## 11. Git 規範

### 11.1 分支

| 分支 | 來源 | 合併到 | 觸發 |
|---|---|---|---|
| `feature/*`、`fix/*` | `dev` | `dev`（PR） | L1 |
| `dev` | — | `main`（release PR） | L2 |
| `main` | — | — | 打 tag 觸發 L3 |
| `hotfix/*` | `main` | `main`（PR），之後自動開 PR 回併 `dev` | L1（Sanity 模式） |

### 11.2 Commit 訊息：先偵測專案慣例

workflow.md 規定每個 commit 都要以 `[vX.Y.Z]` 開頭，但既有專案多半已有自己的 commit 慣例。為了不和既有歷史衝突，**commit 前會先檢查歷史 commit，確認專案慣例後才依該慣例驗證**。只有在專案慣例本身就帶版本號時，才要求版本號前綴。

**可辨識的慣例**（由上而下比對，取第一個符合的）：

| 名稱 | 格式 | 範例 |
|---|---|---|
| `workflow-version` | `[vX.Y.Z] type(scope): subject` | `[v1.5.1] fix(cli): 修復版本斷言` |
| `conventional-emoji` | `type(scope): <emoji> subject` | `test: ✅ 新增登入流程的 E2E 測試` |
| `conventional` | `type(scope): subject` | `feat(auth): 新增登入頁` |
| `gitmoji` | `<emoji> subject` | `✨ 新增登入頁` |
| `custom` | `commit.custom_regex` | 由專案自訂 |

```regex
workflow-version   : ^\[v\d+\.\d+\.\d+[\w.-]*\]\s+(<types>)(\([\w.-]+\))?!?:\s+.+
conventional-emoji : ^(<types>)(\([\w.-]+\))?!?:\s+(\p{Extended_Pictographic}|:\w+:)\s*.+
conventional       : ^(<types>)(\([\w.-]+\))?!?:\s+.+
gitmoji            : ^(\p{Extended_Pictographic}|:\w+:)\s+.+
```

`<types>` 取自 `commit.types`，regex 以 Unicode 模式（`u` flag）執行。

**偵測流程**（`commit.convention: auto` 時）：

```
 commit-msg hook / G0
   │
   ▼
 取樣：git log --no-merges -n <samples> <base>
   │   G0 只取 PR base 分支的歷史，排除 PR 自己的 commit，避免 PR 自行改變慣例
   ▼
 樣本數 < min_samples？
   ├─ 是 → 視為新專案，採用 fallback（預設 workflow-version），並提示
   └─ 否 → 逐筆比對，統計各慣例的符合比例
             │
             ├─ 最高比例 ≥ min_ratio → 採用該慣例驗證本次 commit
             └─ 都未達 min_ratio   → 不擋 commit，只發警告
                                     並建議執行 commit lock 指定慣例
```

**偵測結果範例**：

```
$ node ci/run.mjs commit detect
base: origin/dev   samples: 50 (no-merges)
  conventional-emoji   46 / 50   92%   ← 採用
  conventional          2 / 50    4%
  unmatched             2 / 50    4%
version prefix: 未使用 → 版本一致性只檢查 tag ↔ package.json（§11.3）
```

**相關指令與規則**：
- `node ci/run.mjs commit detect`：只輸出偵測結果，不修改任何檔案。
- `node ci/run.mjs commit lock [--convention <name>]`：把偵測結果（或指定的慣例）寫入 `gates.yml` 的 `commit.convention`。`gates.yml` 受 CODEOWNERS 保護，所以鎖定慣例必須經過 PR 核可。鎖定之後就不再自動偵測。
- **本地 hook 的快取**：偵測結果以 base 分支的 commit SHA 為 key，快取在 `.git/ci-cache/commit-convention.json`，避免每次 commit 都重新掃描歷史。
- **CI 檢查**：G0 用同一套偵測邏輯，檢查 PR 範圍內的每一個 commit（`git log base..head`）。本地與 CI 的結果一致。
- **偏移提示**：已鎖定慣例時，G0 會順便統計近期歷史的符合比例。比例低於 `min_ratio` 時，只在 summary 中提示慣例可能已改變，不阻擋。
- **偵測邏輯的測試**：`tests/ci/commit-convention.test.mjs` 涵蓋各慣例的正反例、emoji 與 `:shortcode:` 寫法、樣本不足、比例未達門檻，以及排除 PR 自身 commit 的情境。

### 11.3 版本一致

- 一律檢查：tag 與 `package.json` 的 `version` 必須相同，由 L3 檢查。
- 只有慣例為 `workflow-version` 時，才另外檢查 commit 前綴與 `package.json` 的 `version` 一致，由 hook 與 G0 檢查。
- 版本一律由 `node ci/run.mjs bump patch|minor|major` 遞增。

### 11.4 Hooks

`.githooks/pre-push` 範例：

```sh
#!/bin/sh
exec node ci/run.mjs hook:pre-push "$@"
```

- 不使用 husky，以 `git config core.hooksPath .githooks` 啟用。
- 根目錄 `package.json` 的 `"prepare"` 會在 `pnpm install` 時自動設定。
- Hooks 只是提早回饋，不能取代 CI：本地可以用 `--no-verify` 跳過，CI 不行。

### 11.5 受保護路徑

以下路徑列入 `ci/owners.yml`，修改時必須經過指定人員核可，落實 workflow.md 的職責分離原則：

- `gates.yml`、`.security-allowlist.yml`
- `ci/**`、`.githooks/**`、`.github/**`
- `deploy/**`
- `docs/stage_state.json`（local 模式的計數檔）

---

## 12. 移除 AI 依賴：與 workflow.md 的對照

| workflow.md 原做法 | 原本由誰執行 | 本規範改法 |
|---|---|---|
| 階段 1 R1：模糊用語、FR 是否有 AC、NFR 是否量化 | 模型自測 | `spec-lint`：regex 加表格解析 |
| 階段 1 R2：EC 7 類是否判定、衝突表是否有空格 | 模型 | 腳本檢查表格是否填完整（不判斷內容是否合理） |
| 階段 1 R3：冷讀 | 子代理人 | 保留為選用，人或 AI 都可以，不放進管線 |
| 追溯矩陣 FR → AC/EC → 測試 ID | 模型 | `trace-check`：掃描測試檔中的 `T-R*` ID，與 spec 比對 |
| 階段 2 人工審核 | 人 | PR 核可，加上腳本檢查 `docs/02_human_review.md` 的 Approved 欄位 |
| 3B R3 獨立審查 | AI Reviewer | **移除**；改為人工 PR 核可。AI review 可選，只留 advisory 留言，不影響 Exit Code |
| exit 2 / 3 / 4 交階段 3A 判定 | AI | 腳本只貼 `route:*` label 並指派 CODEOWNERS，不限定由誰處理 |
| 工單產生與計數 | 腳本 | 沿用。單一瑕疵的計數存在 issue metadata；累計修復迴圈支援 `branch` / `release` 兩種範圍（§8.6）；local 模式沿用 `docs/stage_state.json` |
| 工單編號 `DEF-3B-YYYYMMDD-XXX` | 腳本 | 改為 `DEF-{gate}-{YYYYMMDD}-{fp 前 6 碼}` |
| 版本號一致性 | 人 | `version-check`（G0 / L3） |
| 每個 commit 強制 `[vX.Y.Z]` 前綴 | 規範 | 先從歷史 commit 偵測專案慣例；只有慣例本身帶版本號時才要求前綴（§11.2） |
| 階段 5 容器化與部署 | 模型 | 由 `deploy` adapter 與 `Dockerfile` 執行；模型只在撰寫階段參與 |

---

## 13. 刻意保留給人工的環節

| # | 環節 | 原因 |
|---|---|---|
| 1 | PR 核可 | 架構與需求是否合理，無法完全以腳本判定 |
| 2 | 正式環境部署核可 | 上線時機屬於業務決策 |
| 3 | 修改 `gates.yml`、豁免清單、`ci/**` | 防止實作者自行放寬閘門 |
| 4 | Exit 5（停止條件）裁決與計數歸零 | 需要權衡修復方向或接受風險；歸零只能由 owner 執行，並留下稽核紀錄 |
| 5 | 真人 Usability 測試 | 自動化只能檢查代理指標（a11y、效能） |
| 6 | Spec 冷讀（選用） | 需要理解需求語意 |

---

## 14. 合約測試

`tests/ci/contracts/` 內放四組測試，每組都用假的 HTTP 與 executor，對該類別的所有 adapter 跑同一套測試：

| 檔案 | 驗證內容 |
|---|---|
| `platform.contract.test.mjs` | `capabilities()` 結構、summary、留言、ledger 讀寫、降級行為 |
| `issues.contract.test.mjs` | 去重、計數遞增、升級與停止門檻、自動關閉、API 失敗時不改變 Exit Code |
| `deploy.contract.test.mjs` | build 回傳 `ARTIFACT_ID`、deploy 回傳 `BASE_URL`、artifact hash 核對、rollback 讀取 ledger |
| `loops.contract.test.mjs` | `branch` / `release` 各自計數；併用時先達上限者觸發 exit 5；同一 commit 重跑不重複計數；exit 5、6 不計數；自動與人工歸零；非 owner 歸零會被拒絕 |

新增平台或部署目標時，只要通過合約測試和 `render --check`，就能接上整條管線。

---

## 15. 導入順序

| 階段 | 內容 | 完成條件 |
|---|---|---|
| P1 | Hooks、G0、G1、G2、G4；`ctx` / `platform` / `issues` 的 github 與 local adapter；`render --platform github`；commit 慣例偵測；`branch` 範圍迴圈計數；四組合約測試骨架 | PR 會被 `ci-result` 擋下，失敗時產生結果摘要 |
| P2 | G3、G5、G6；deploy 介面與 compose adapter | PR 能在 runner 內起站並通過 Smoke |
| P3 | G7 全套；staging pipeline；release ledger（GitHub Deployments 與 git notes 降級）；`release` 範圍迴圈計數 | 合併到 dev 後自動部署 staging 並跑完整 Regression |
| P4 | Load / Stress；L3 blue-green 與自動 rollback；k8s runtime（選用） | tag 後能經核可部署 production，失敗時自動 rollback |
| P5 | Nightly；flaky 統計；template 與 CODEOWNERS 同步 | nightly 失敗會開 issue，通過會自動關閉 |
| P6 | Serverless deploy adapter（從 Vercel 開始） | 通過 deploy 合約測試 |
| P7 | GitLab、Gitea 的 ctx / platform / render adapter；Jira issue adapter | 通過 platform 與 issues 合約測試，以及 `render --check` |

---

## 16. 已知限制與待決事項

1. **GitLab 付費功能**：Protected environments 與 Approval rules 需要付費方案，免費方案會使用 §7.2 的降級方式。腳本強制核可的做法在核可後需要重跑 `ci-result`。
2. **Serverless 的 Stress 成本**：需要依 provider 決定 `max_cost_guard` 的計算方式，P6 再定案。
3. **跨瀏覽器視覺回歸**：snapshot 只在 chromium 上比對；其他瀏覽器只做功能相容性測試，以免維護多組 snapshot。
4. **Fork PR 的 `branch` 計數**：fork 來的 PR 拿到的 token 是唯讀的，無法寫入計數留言，因此 `branch` 範圍計數不會生效，`ci-result` 會在摘要中註明。不建議為此改用 `pull_request_target`，因為這會讓不受信任的程式碼取得寫入權限。
