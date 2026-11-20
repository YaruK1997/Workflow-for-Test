# 軟體測試類型關聯圖

依軟體測試理論（ISTQB 分類、V-Model、測試金字塔、CI/CD 流程）整理以下九種測試的關係：Unit、Integration、System、E2E、Regression、Smoke、Sanity、Load/Stress、Usability。

---

## 1. 分類總覽：三個不同維度

這九種測試**不在同一個維度上**，最常見的混淆就是把它們排成一條線。

```
                                  軟體測試
                                     │
        ┌────────────────────────────┼─────────────────────────────┐
        ▼                            ▼                             ▼
 ┌──────────────┐           ┌─────────────────┐           ┌─────────────────┐
 │ Test Level   │           │ Test Type       │           │ Trigger/Purpose │
 │ 測多大範圍   │           │ 測什麼特性      │           │ 何時跑、為何跑  │
 └──────┬───────┘           └────────┬────────┘           └────────┬────────┘
        │                            │                             │
   Unit                      Functional                       Smoke
   Integration                 └─ E2E（完整使用者流程）       Sanity
   System                    Non-functional                   Regression
   (Acceptance)                ├─ Load / Stress（效能）
                               └─ Usability（易用性）
```

- **Level** 決定被測物的大小：一個函式、模組串接，或整個系統。
- **Type** 決定驗證的面向：功能對不對，還是快不快、好不好用。
- **Trigger** 決定在什麼時機跑、跑多少：可以套用在任何 Level 上。

---

## 2. V-Model：開發階段 ↔ 測試層級

```
  Requirements ································ Acceptance ── Usability
     ╲                                            ╱
      System Design ························ System ── E2E · Load/Stress
         ╲                                  ╱
          Architecture ·············· Integration
             ╲                        ╱
              Module Design ···· Unit
                 ╲              ╱
                       Coding

  左臂：定義「要做什麼」     ·····  左右互相對應
  右臂：驗證「做對了沒」
```

| 測試層級 | 驗證對象 | 掛在這層的測試 |
|---|---|---|
| Unit | 單一函式／元件（模組設計） | — |
| Integration | 模組之間的介面、API、DB（架構設計） | — |
| System | 整個系統是否符合規格（系統設計） | E2E、Load/Stress |
| Acceptance | 是否符合使用者真正的需求 | Usability |

---

## 3. 測試金字塔：數量、速度與成本

```
                        ▲  慢 · 貴 · 脆弱 · 最接近真實使用者
                       ╱ ╲
                      ╱E2E╲                    ┌───────────────────────┐
                     ╱─────╲                   │ 金字塔外：非功能測試  │
                    ╱       ╲                  │                       │
                   ╱ System  ╲                 │  Load / Stress        │
                  ╱───────────╲                │  Usability            │
                 ╱             ╲               │                       │
                ╱ Integration   ╲              │  不比數量，看時機：   │
               ╱─────────────────╲             │  上線前、大改版前     │
              ╱                   ╲            └───────────────────────┘
             ╱       Unit          ╲
            ╱───────────────────────╲
                        ▼  快 · 便宜 · 穩定 · 數量最多

   Regression ═══ 可橫跨每一層 ══▶  Unit、Integration、E2E 的測試都可以收進回歸測試集
```

---

## 4. 交付流程：各測試在 CI/CD 中的位置

```
  Commit / PR
      │
      ▼
 ┌──────────┐  fail
 │   Unit   │──────────▶ 退回開發者（秒級回饋）
 └────┬─────┘
      ▼
 ┌─────────────┐  fail
 │ Integration │───────▶ 退回開發者
 └──────┬──────┘
        ▼
   Build & Deploy → Test / Staging 環境
        │
        ▼
 ┌──────────┐  fail
 │  Smoke   │──────────▶ 拒收這個 build（不浪費 QA 時間）
 └────┬─────┘            廣而淺：能開站？能登入？能結帳？
      ▼
 ┌────────────────────────────────┐
 │ System Testing                 │
 │   ├─ Functional / E2E          │  完整使用者流程
 │   ├─ Regression（full suite）  │  舊功能沒被改壞
 │   └─ Load / Stress             │  負載上限、崩潰點
 └───────────────┬────────────────┘
                 ▼
 ┌────────────────────────────────┐
 │ Acceptance / Usability         │  真實使用者：好不好用
 └───────────────┬────────────────┘
                 ▼
             Release ─────────▶ Production Smoke（上線後快速確認）
                 │
                 │  發現 bug → Hotfix
                 ▼
 ┌──────────┐
 │  Sanity  │  窄而深：只驗證修正點與周邊邏輯
 └────┬─────┘
      ▼
 Regression（selective：只跑受影響的範圍）
      ▼
   Release
```

---

## 5. Smoke、Sanity、Regression 的差別

這三個最常被混用。用「廣度 × 深度」來區分：

```
                  Breadth（涵蓋範圍）
               窄 ◀────────────────────▶ 廣
          淺 ┌──────────────────────────────┐
             │                    Smoke     │
   Depth     │                              │
  （深度）   │                              │
          深 │  Sanity          Regression  │
             └──────────────────────────────┘
```

| | Smoke | Sanity | Regression |
|---|---|---|---|
| 時機 | 每次新 build | 小修正／hotfix 之後 | 任何變更之後 |
| 要回答的問題 | 這個 build 能不能測？ | 這個修正合理嗎？ | 舊功能有沒有被改壞？ |
| 範圍 | 全部主要功能、淺 | 單一區塊、深 | 全部或受影響範圍、深 |
| 是否有腳本 | 通常有、會自動化 | 常是非正式的手動測試 | 有腳本、高度自動化 |
| 關係 | Regression 的入口閘門 | 可視為 Regression 的子集 | — |
