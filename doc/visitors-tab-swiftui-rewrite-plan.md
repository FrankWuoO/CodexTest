# Visitors Tab — SwiftUI 重寫計劃（GEN-12671）

> 撰寫時間：2026-06-02
> 對應 ticket：[GEN-12671 SwiftUI Reimplementation of Visitors tab](https://www.notion.so/2fd88fc53823802e86afdd9d110e6a16)（iOS · Priority: Max · Status: Backlog）
> 行為基準（behavior map）：[visitors-tab-architecture.md](visitors-tab-architecture.md) / `.html`（本計劃所有「現狀」引用皆以該文件為準）
> 相關：App 啟動流程重構（見 memory `app-lifecycle-refactor` / `/Users/frankwu/.claude/plans/app-lifecycle-starry-journal.md`）
> 範本：**Messages tab 的 `ConversationWireframe` 已完成同型 UIKit→SwiftUI 遷移**（Flagr-backed enum 切換 → ramp → 移除 v1，commits `43e8fe6ee` drop v1 support / `ad206228c` GEN-13822 v1 cleanup）；本計劃的「目的地架構」與切換機制以它為藍本（見 §2.2 / §4.1）。**只借架構、不借它的 parity 寬鬆**（§2.2 ⚠️）。
> 現況基準（2026-06-02）：**GEN-13799 (#2462)** 已將 `.flagrConfigUpdated` 集中到 [`AppMasterStore.onFlagrUpdate()`](BWF/App/Classes/AppMasterModules/AppMasterStore.swift#L205)（single ingestion point）、`hideVisitorsTab`→`rebuildVisitorsTab`、tab 改 build-once；本計劃已據此校正。
>
> **進度現況（2026-06-08，rebase 後）**：branch `refactor/visitor-tab-architecture-md`，已 rebase 到 `version-79/main`。commit stack（舊→新）：`6383fd126` 資料層/pager/You-Liked → `6bfc2d99d` 導航/profile-view → `f07001fa7` decline/like → `6d6a0e85c` 空狀態/analytics parity → `64d6253d9` **3c singleProfile feed** → `235cb3a9c` **3f filter chip-bar** → `a55813708` **reskin-drop + L10n** → `fbf5cf7e7` **orchestration 搬進 `VisitorsHostingController` + 抽共用 `FullProfileDetailsView`**。
> - **已完成**：Phase 0（harness 建好，未跑矩陣）、1（flag + 兩處分支）、3（grid/pager/singleProfile feed/filtersV1V2/空狀態）、4（closure 注入導航 + 詳情三明治）、reskin-drop、orchestration 重構（見 §4.1 實作後校正）。build 綠 + 等效對抗驗證 PASS + 23 個 package VM 測試綠。
> - **未完成**：Phase **2c/2d/2e**（slim facade / network wrapper transport / config runtime diff）、Phase **5 parity matrix（DoD 核心、最大塊）**、Phase **6 v1 移除**。P2 preselect 延後（§7）、P1 不修（Amplitude blocked）。
> - ~~⚠️ working tree 的 `.swiftui` 強制 hack **merge 前必還原**~~ → ✅ **2026-06-12 已還原**：getter 讀真 flag（`Implementation(rawValue:) ?? .uikit`），**B1 blocker 解除**；裝置已實測 uikit↔swiftui flag flip（flip 當下零 request、開 tab 才 fetch＝lazy 設計，確認可接受）。
> - ⬜ 裝置互動冒煙尚未做（headless 的 build/等效/單測都綠，runtime 行為待實機驗）。
> - ⚡ **2026-06-12 全面複稽核（39-agent）**：§8 修法逐項對抗驗證**全數通過**；另掃出 1 high + 10 medium 新缺口、10 個新碼 bug/風險、4 處文件勘誤、parity-driver fidelity bug——完整記錄已併入 [QA 計劃 §7 watchlist](visitors-parity-qa-plan.md)，決策摘要見本文件 **§8.5**。

---

## 0. Context（為什麼做、要達成什麼）

Ticket 要求用 SwiftUI **全新實作** Visitors tab，舊版保留為 baseline，用 Flagr key 切換新/舊。明確需求：

1. SwiftUI 重寫，**不沿用舊邏輯**。
2. `VisitorsManager` 換成「輕量、可注入功能的 view-model」。
3. 新 tab **只在拿到 Flagr config 之後** 才實例化；理想上做一個**單一 async Flagr 存取點**。
4. 舊實作保留，用一個 **Flagr key 切換新/舊**。
5. Network layer 包一層 wrapper（**本次 scope**）——作為 `VisitorsRepository` 的 transport（見 §3 風險 2 / §4.2 Phase 2）。

**Definition of Done**：新版行為 / Analytics events / Network requests 與舊版 **100% 一致**；重寫前先有 behavior map（✅ 已有架構文件）。

本計劃分兩部分：**(A) 使用者指定先調查的兩件事**（§1–§2），**(B) 據此制定的分階段重構計劃**（§4 起；重構踩雷點見附錄 A）。

---

## 1. 調查一：單一 async Flagr 存取點

### 1.1 現狀（已對程式碼驗證）

| 事實 | 來源 |
|---|---|
| `RemoteConfigProperties.shared` 是 singleton；`visitorsProperties` 是 `lazy var` | [RemoteConfigProperties.swift:135](BWF/App/Classes/Analytics/RemoteProperties/RemoteConfigProperties.swift#L135) / [:45](BWF/App/Classes/Analytics/RemoteProperties/RemoteConfigProperties.swift#L45) |
| `VisitorsProperties` 屬性是 `@UserDefault`-backed → **讀取是同步從 UserDefaults**（無阻塞 I/O） | [VisitorsProperties.swift](BWF/App/Classes/Analytics/RemoteProperties/VisitorsProperties.swift) |
| Flagr 抓取入口 `fetchFlagr(_ fetchedHandler: VoidCompletionHandler? = nil)`，**callback-based**（非 async/await），依登入狀態走 `fetchFlagrConfiguration` / `fetchFlagrBeforeLogin` | [FlagrConfigClient.swift:28](BWF/App/Classes/Analytics/Flagr/FlagrConfigClient.swift#L28) |
| `fetchFlagrConfiguration` **要求有效 location**（lat/lng != 0），否則 `handleFailurOfFlagrFetch()` | [FlagrConfigClient.swift:41-71](BWF/App/Classes/Analytics/Flagr/FlagrConfigClient.swift#L41-L71) |
| 抓完（**成功與失敗都**）發 `.flagrConfigUpdated` notification（main queue） | [FlagrConfigClient+Handler.swift:27-31](BWF/App/Classes/Analytics/Flagr/FlagrConfigClient+Handler.swift#L27-L31) |
| **已登入啟動：`showTabBar()`（建所有 tab）就在 `fetchFlagr` 的 completion 內** → tab 是 Flagr 解析後才建，不是「先用 default 建再 patch」 | [AppMasterStore.swift:79-84](BWF/App/Classes/AppMasterModules/AppMasterStore.swift#L79-L84) |
| 啟動的 5s timeout 是 **splash 等 location** 的 fallback（**非** Flagr timeout）；denied 則跳過、立刻 fast-fail | [SplashScreenPresenter.swift:79-100](BWF/App/Classes/SplashScreen/SplashScreenPresenter.swift#L79-L100) |

**結論**：「等 Flagr 再建 tab」對**已登入路徑已存在**——`fetchFlagr { showTabBar() }`（[AppMasterStore.swift:79-84](BWF/App/Classes/AppMasterModules/AppMasterStore.swift#L79-L84)）在 Flagr 完成（成功或 fast-fail）後才建 tab；bound = splash 的 location 5s timeout，逾時/失敗則用 UserDefaults 現值（lat=0,0 降級）。**ticket req 3 對主路徑已滿足，不需另造 async 存取點**；要補的是兩個缺口（§1.2）。

### 1.2 真正要補的兩個缺口（沿用既有 gate）

gate 已存在（§1.1），**沿用 `fetchFlagr { showTabBar }`** 即可。要修的是：

- **(a) 啟動 double-build（最該修）**：同一次 launch fetch 同時 (i) completion → `showTabBar()` 建一次（inline 先跑）、(ii) `notifyUpdateOfFlagr` 以 `main.async` 發 `.flagrConfigUpdated`（[+Handler.swift:27-31](BWF/App/Classes/Analytics/Flagr/FlagrConfigClient+Handler.swift#L27-L31)）→ `onFlagrUpdate` → `rebuildVisitorsTab` 又建一次（後跑）。→ 用 §3.1 的 diff/no-op 消掉：驅動本次 `showTabBar` 的那發 fetch，其 notification 不再 rebuild。
- **(b) 新用戶路徑沒 gate**：`startAsNewUser` 同步 `showTabBar()`（[AppMasterStore.swift:97](BWF/App/Classes/AppMasterModules/AppMasterStore.swift#L97)），不在 fetch completion 內 → onboarding 期間先建（app-lifecycle action item：延後它）。逾時/denied 會用 lat=0,0 降級值建，location 到後靠 runtime（§3.1）補正。

**新 tab 隔離**：注入 `VisitorsConfigProviding`，新 tab 只依賴它取已解析 config：

```swift
protocol VisitorsConfigProviding {
    func resolved() async -> VisitorsConfig   // 已登入：showTabBar 時 config 已就緒 → 同步讀 UserDefaults
}
```

對已登入路徑 config 已就緒、同步讀即可，loader 只在新用戶 / 邊緣情況需要。

### 1.3 gate 範圍（要不要擴到已登入以外）

choke point 是 `showTabBar()`（[AppMasterWireframe.swift:328-334](BWF/App/Classes/AppMasterModules/AppMasterWireframe.swift#L328-L334)：一次建好所有 tab 再 swap rootVC），且已在 `fetchFlagr` completion 內。fallback 幾乎免費（讀的本來就是 UserDefaults，無需另寫）。若要把「等 Flagr」從已登入擴到全路徑，才需要 `awaitFlagrResolved(timeout:)`（await 既有 `.flagrConfigUpdated` + timeout、**不自己 kick fetch**）：

| 範圍 | 做法 | 取捨 |
|---|---|---|
| **維持現狀**（只已登入有 gate，**建議**） | 沿用 `fetchFlagr { showTabBar }` + 消 double-build | 最小改動、符合本 ticket |
| 擴到整個 tab bar | `showTabBar` 前 `awaitFlagrResolved` | 一勞永逸消 3→5 跳動；屬 app-lifecycle 級、風險大 |

---

## 2. 調查二：Flagr key 切換新 / 舊

### 2.1 掛載點與切換點（已驗證）

Visitors tab 在 `AppMasterWireframe.setUpForOldMainUI()` 組裝，**有兩處實例化 `VisitorsContainerWireframe`，兩處都要分支**：

| 點 | 位置 | 角色 |
|---|---|---|
| 初次建立 | [AppMasterWireframe.swift:375](BWF/App/Classes/AppMasterModules/AppMasterWireframe.swift#L375) | `setUpForOldMainUI()` 內，建好後 `updateTabBar()` 掛進 `AppTabBarController` |
| 重建 | [AppMasterWireframe.swift:533](BWF/App/Classes/AppMasterModules/AppMasterWireframe.swift#L533) | `rebuildVisitorsTab()` 內（由 [`AppMasterStore.onFlagrUpdate()`](BWF/App/Classes/AppMasterModules/AppMasterStore.swift#L205) 呼叫） |

⚠️ **關鍵 rebuild loop（已集中於單一入口）**：`.flagrConfigUpdated` 現由 [`AppMasterStore.onFlagrUpdate()`](BWF/App/Classes/AppMasterModules/AppMasterStore.swift#L205) **單一入口** ingest（GEN-13799 把原散在 `AppMasterWireframe` / `AppMasterView` 的 observer 收斂於此）→ `updateTabs()` + `rebuildVisitorsTab()`（[AppMasterWireframe.swift:529](BWF/App/Classes/AppMasterModules/AppMasterWireframe.swift#L529)）+ `refreshSession()`。**目前仍是鈍的**：每次（含失敗）都無條件重建。這對舊版是「免費的動態切換」，**也是位置授權變化時更新 tab 的途徑**——新版要**保留此能力但在 `onFlagrUpdate` 內改成精準觸發**（見 §3.1）。

### 2.2 既有切換 precedent（沿用同一套機制）

- **整頁 UIKit→SwiftUI 切換 precedent（本計劃藍本）**：**Messages tab 的 `ConversationWireframe` 已做過一模一樣的事**——Flagr-backed `@UserDefault` enum `InboxProperties.version`（`baseline = "local_ww_tabs"` 走 UIKit storyboard、`matchesSeparated = "matches_list_square_tiles"` 走 `UIHostingController<ConversationsList.HostingView>`，**預設 `.baseline` = 舊版**）；**單一 wireframe** 在 `updateMessageView()` switch 該 enum 決定 `view`（型別 `UIViewController`，兩條路都塞得進），config 變了就 diff（`if inboxVersion != …`）後 rebuild + reassign。ramp 到 100% 後以 `43e8fe6ee`（drop v1 support）+ `ad206228c`（GEN-13822 v1 cleanup）**整批刪 v1**（`BWFMessagesListVC` 537 行、`BWFConversationsListCell`、`InboxUpsellHeaderView` + Flagr/UserDefaults key）。→ §2.3 flag、§4.1 目的地架構、§5 檔案改動皆以此為藍本。
  - ⚠️ **借結構、不借寬鬆**：`matchesSeparated` 同時改了**行為**（matches 分頁、方形 tile），Conversations 因此**沒做 parity harness**；本 ticket 的 DoD（§0）要 **100% 行為一致**（純重寫、不改 UX），所以 §2.3 的 flag 必須是**純技術切換**（不夾帶任何 UX 改動），且 §4.2 Phase 0 parity harness **不可省**。
- **變體切換 precedent（頁內、非整頁）**：`VisitorsContainerWireframe.init` 依 `filtersEnabled` / `filtersV2Enabled` 兩個 bool 決定 config（[VisitorsContainerWireframe.swift:24-32](BWF/App/Classes/Tabbar/VisitorsContainerWireframe.swift#L24-L32)）。`screenVersion`（baseline/singleProfile）也是同模式的 enum。
- **Flagr key 慣例**：key 定義在 `FlagrConfigClient+Keys.swift`，於 `FlagrConfigClient+Values.swift` 解析後寫進 `RemoteConfigProperties.shared.visitorsProperties.*`（`@UserDefault`-backed）。目前**無**任何新/舊整頁切換的 key（淨新增）。

### 2.3 建議：新增 `visitorsImplementation` flag

```swift
// VisitorsProperties.swift — 仿 InboxProperties.version（Messages tab 同款）/ screenVersion enum
enum Implementation: String { case uikit; case swiftui }
var implementation: Implementation { Implementation(rawValue: implementationRaw) ?? .uikit } // 預設 uikit（舊版）
```

> **純技術切換**：此 enum 只決定「UIKit 或 SwiftUI 實作」，**不得**像 Conversations 的 `InboxVersion`（值是 `matches_list_square_tiles` 這種功能名）夾帶功能/變體語意——變體仍走既有 `screenVersion` / `filtersEnabled`。確保 flag on↔off 唯一差異是實作技術、行為一致（DoD）。

在 **§2.1 兩處** 分支：

```swift
let wireframe: LegacyWireframe =
    RemoteConfigProperties.shared.visitorsProperties.implementation == .swiftui
        ? SwiftUIVisitorsWireframe(masterStore: masterStore, masterWireframe: wireframeF)
        : VisitorsContainerWireframe(masterStore: masterStore, masterWireframe: wireframeF)
```

預設 `.uikit` → flag off 時行為與今日**完全一致**，符合 DoD 的「可切換」與「不影響舊版」。

---

## 3. 關鍵風險與張力（先看這段）

1. **config 變更有兩個時刻；runtime 那條要從「鈍」變「精準」**：
   - **(a) 啟動**：tab 在 `fetchFlagr` completion 內建（§1.1），已是「等 config 才建」。
   - **(b) runtime**：位置授權 `denied/notDetermined → authorized` 會重打 Flagr → tab 須跟著變（[AppMasterView.swift:323-338](BWF/App/Classes/AppMasterModules/AppMasterView.swift#L323-L338) → `.flagrConfigUpdated` → `onFlagrUpdate` → `rebuildVisitorsTab`）。**DoD 要求保留此能力**，但現況**每次（含失敗）都無條件整棵 rebuild**，要改精準。

   **收斂模型（已決定）**——`.flagrConfigUpdated` 由 [`onFlagrUpdate()`](BWF/App/Classes/AppMasterModules/AppMasterStore.swift#L205) 單一 ingest（GEN-13799 已完成），不再讓各層各自 `addObserver`：
   - **觸發收斂 + diff**：只在真會改 config 的事件重打（位置授權變化即可，[316-338](BWF/App/Classes/AppMasterModules/AppMasterView.swift#L316-L338)；不盲抓 `didBecomeActive`）；新 config 先跟上次 snapshot **diff，相同則 no-op**（同時消掉啟動 double-build 與失敗 notification 的重建）。
   - **Master**：判斷顯示/隱藏哪些 tab（= 既有 `updateTabs()`，作用於動態 tab；Visitors 是 base tab 永遠在 → 不增刪、只 fan-out）+ 把 resolved config 經既有 store 分派（`addHandler(WhoViewedMeStore)`，[VisitorsWireframe.swift:102](BWF/App/Classes/Tabbar/VisitorsWireframe.swift#L102)）往下帶。
   - **Child**：拿上次 snapshot 自行決定 **keep / re-init**——`@Observable` 讓**值層變更自動 re-render（= keep）**，只在 **VM / 容器形狀變** 時 re-init：
     - **re-init**：`filtersEnabled`（3-tab↔1-tab）、`filtersV2Enabled`（V1↔V2 VM）→ 重建時**外層 Container 蓋 Loading overlay → 重畫**（遮住重建、順手蓋掉 tab toggle）。
     - **keep**：`screenVersion`、`decliningVisitorsPlace`、distance / worldwide / blur / 文案 / reskin …；最終分類 Phase 2/3 定案。
   - **收編範圍**：散在 [VisitorsContainer.swift:85](BWF/App/Classes/Modules/New/Visitors/VisitorsContainer.swift#L85)、[VisitorsManager.swift:296](BWF/App/Classes/Services/VisitorsManager/VisitorsManager.swift#L296) 的 observer 收進此路徑；cross-cutting service（IAP / LikesMission）與 tab 無關，維持自訂閱。
   - **互斥**：re-init 會 `new` child、於 `init` 讀新 config → 不再另行通知（否則 double）。
   - 這套同時是 **app-lifecycle 消除啟動 double-build** 的做法（不只 Visitors，每個子 tab 自行判斷）；SwiftUI 下結構變更也是宣告式 re-render（今天結構在 `init` 凍結才這麼鈍），此 edge case 反而是 SwiftUI 重寫的論據。

2. **`VisitorsManager` 拆三塊（已盤點：9 個外部 consumer 只讀唯讀聚合值）**：singleton（[VisitorsManager.swift:83](BWF/App/Classes/Services/VisitorsManager/VisitorsManager.swift#L83)）目前混了四件事——networking、per-distance cache、對外聚合值、analytics。盤點 9 個 visitors 模組外 consumer，**全部只讀聚合值**（`totalNewVisitors` badge ×4、`totalViewedMeVisitors`/`likesInVisitorsCount` paywall/browser ×4、`refreshVisitorUpsell` ×1）——**沒有一個碰 fetch 方法或 per-distance cache**。所以可乾淨拆：
   - **① Networking → `VisitorsRepository` protocol（架在 Network wrapper 上）**：`fetchVisitorsWithCount`（V4 `/visitors/list`）、`fetchVisitorsWithFilters`（V5 `/visitors`）、`deleteVisitor` + request models。只被 tab 內部用 → 安全抽出；新 `@Observable` VM 依賴此 protocol（可注入/可測 = ticket req 2）。**這就是 VisitorsManager「承攬的打 API」→ 折進 Network wrapper（ticket item 5）的接點**。
   - **② per-distance cache + 分頁 → 新 tab 的 `@Observable` VM**（local/worldwide/filtered、offset、dedup = tab 私有狀態）。
   - **③ 對外聚合值 → 保留一個 slim 唯讀共享 facade（不可刪）**：`totalNewVisitors` / `totalViewedMeVisitors` / `likesInVisitorsCount` / `refreshVisitorUpsell`。這才是 9 個 consumer 真正要的，與 networking / tab UI 無關。
   - **拆解順序（Network wrapper 本次 scope）**：**先抽 `VisitorsRepository` protocol、內部仍用現有 `serviceController`（行為 100% 不變）→ 新 VM 依賴它 → 本次再把 repository 的 transport 換成新 Network wrapper（ticket item 5）、不動 VM**。分兩步只是降風險（介面先穩、transport 後換），**不是**把 wrapper 推到別的 ticket。聚合值 facade 的數值改由 repository/VM fetch 完後回報（取代目前 VisitorsManager 內部更新）。

3. **變體爆炸（已確認：全部都還在用，不可下線）**：舊版 3 頂層 config（`default` / `filtersOnLikes(.v1)` / `.v2`）× 7 子變體（架構文件 §4）**都在線上使用**——`default` 與 `.v1` 不能砍。
   → 重寫**必須覆蓋全部變體並逐一驗 parity**，無法靠下線縮減重寫面。Phase 0 parity harness（§4.2）要**建變體矩陣**（3 config × screenVersion × decline × gender × premium …）逐格 diff，是 DoD 的硬需求。

---

## 4. 重構策略（分階段，對齊附錄 A）

> 每階段獨立可上 PR；flag 預設 off，整段期間舊版不受影響。

### 4.1 目的地架構（對齊 `ConversationWireframe`，並讓 v1 移除便宜）

> **已決定**：新版的目標形狀直接照搬 Messages tab 的 `ConversationWireframe`（§2.2 precedent）。這套形狀同時服務兩個目標——(a) flag off 時舊版 byte-for-byte 不變（DoD）、(b) ramp 100% 後移除 v1 是機械式刪除（§4.2 Phase 6）。

1. **單一 wireframe 分支 enum**：`SwiftUIVisitorsWireframe`（新）與 `VisitorsContainerWireframe`（舊）由 §2.3 的 `visitorsImplementation` enum 在 §2.1 兩處選擇；wireframe 的 `view` 型別需可同時容納 `UIHostingController` 與舊 UIKit VC（同 `ConversationWireframe` 的 `view: UIViewController`）。
2. **SwiftUI 自成 module**：`VisitorsRootView` + `@MainActor @Observable` view-model + repository protocol，放 **`Packages/DesignSystem/Sources/DesignSystem/Screens/Visitors/`**（仿 `ConversationsList`）——與 BWF/App 解耦、可 SwiftUI Preview、且日後移除 ≈ 刪一個資料夾。
3. **導航用 closure 注入，不另起正式 router**：wireframe 建 view-model 時把 `openProfileDetails` / `openPaywall` / `openMatch` / `openReport` … 以 closure 注入（仿 `ConversationWireframe` 的 `openProfile` / `openConversation`）；SwiftUI 層**完全不 import UIKit**、只呼叫 closure，push / present / modal 全由 wireframe 持有。**`BWFFullProfileDetailsVC` 等 detail / 跨模組畫面維持 UIKit**（同 Conversations 把 chat detail 留在 `BWFMessageDetailsVC`；對應 Phase 4 / 附錄 A #13）。
4. **config 變更走 §3.1 的收斂模型**（與導航 closure 正交）：值由 store/publisher 往下帶、子層自行 diff；**不要**在 SwiftUI 層各自 `addObserver(.flagrConfigUpdated)`。
5. **為「便宜移除」設計**：依賴單向（v1 不呼叫 v2）、SwiftUI 全在 package、wireframe branch 只在 §2.1 兩處 → 移除 v1 = 刪 branch + 刪舊 VC/storyboard + 刪 flag key，**對應 Conversations `43e8fe6ee` + `ad206228c` 的規模**。

> **實作後校正（2026-06-08，commit `fbf5cf7e7`）**：上述形狀大致達成，兩處實作偏離記錄如下：
> 1. **orchestration 落在 `VisitorsHostingController`，不在 wireframe**：`SwiftUIVisitorsWireframe` 瘦成只剩 LegacyWireframe conformance（`masterWireframe`/`view`）+ `MainActor.assumeIsolated` 建 Controller；所有 layout 生成 / 導航 closure / analytics / presenter / telemetry 搬進 **@MainActor 的 `VisitorsHostingController`**（既有 `UIHostingController` 子類，天生 @MainActor → **per-func `@MainActor` 28→0、`MainActor.assumeIsolated` 5→4**）。funcs 維持 `static`（這是 closure factory，static 捕獲 local → 無 retain cycle、免 `[weak self]`；評估後不改 instance）。SwiftUI Visitors 檔內聚於 `BWF/App/Classes/Tabbar/SwiftUIVisitors/`。
> 2. **pager 用 iOS 17 paging ScrollView + `.scrollPosition`**（非計劃寫的 `TabView(.page)`——後者首滑誤排版）。
> 導航 closure 注入、`BWFFullProfileDetailsVC` 維持 UIKit（其 SwiftUI-embed bridge `FullProfileDetailsView` 抽成共用檔，放 `BWF/App/Classes/Modules/FullViewProfileDetails/`，legacy `VisitorsWireframe` 也改用）皆如計劃。空狀態目前仍 logic-based、未整成 `enum`（附錄 A #8 延後）。

### 4.2 分階段

> 各 Phase 即時狀態見頂部「**進度現況**」。下表為原始計劃。

| Phase | 目標 | 對應附錄 A |
|---|---|---|
| **0. Parity harness** | 建 analytics / network 對照工具：side-by-side 跑新舊、diff 送出的 events 與 requests（DoD 核心） | #12 telemetry / Analytics 對齊 |
| **1. Flagr switch 骨架** | 加 `visitorsImplementation` flag（§2.3）+ 兩處分支；建空的 `SwiftUIVisitorsWireframe`（flag on 時只渲染 placeholder + loader）；接上 §1.2 的 `VisitorsConfigProviding`（沿用既有 `fetchFlagr { showTabBar }` gate、並消除啟動 double-build，見 §1.2） | #1 路由邊界、#11 config 餵成 `@Observable` |
| **2. 抽資料層（去 VisitorsManager，§3.2 三塊拆解）** | ① 抽 `VisitorsRepository` protocol（networking，內部先用現有 serviceController）→ 新注入式 `@Observable` VM 依賴它、接手三快取 + 分頁 → **再把 transport 換成本次 scope 的 Network wrapper（ticket item 5）**；② 保留 slim 唯讀 facade 給 9 個 consumer 的聚合值；③ config 做成 `@Observable`，runtime 變更走 §3.1 的 **diff + 分級 apply** | #2 Redux/StoreSubscriber、#14 Entity 換 Swift value type |
| **3. SwiftUI view tree** | grid → `LazyVGrid`；`MultiPageViewController` → `TabView(.page)`；segmented / filters header；**複用既有 SwiftUI**（`LikesFeed`、`FiltersViewV1/V2`、`FiltersEmptyView`、`DeclineVisitorLike`）；空狀態整成 `enum EmptyState` state machine | #3 collection 陷阱、#4 pager、#5 storyboard 解耦、#8 空狀態 enum、#10 theme via `@Environment` |
| **4. 導航 / Modal** | paywall / match / report / photo-verified 用 **closure 注入**（仿 `ConversationWireframe`，不另起正式 router；見 §4.1），保留 UIKit 入口；詳情頁三明治（附錄 A #13）最後處理 | #9 跨模組 modal、#13 wireframe SwiftUI 入口 |
| **5. Parity 驗證 + ramp** | 用 Phase 0 harness 驗 analytics/network 100% 一致；全回歸；逐步把 flag ramp up | DoD 全項 |
| **6. v1 移除（ramp 100% 後）** | 仿 Conversations `43e8fe6ee`（drop v1 support）+ `ad206228c`（GEN-13822 cleanup）：刪 `VisitorsContainerWireframe` 分支、Visitors 私有舊 VC（`BWFWhoViewedMeViewController` / `MultiPageViewController` + storyboard）、`visitorsImplementation` flag key；§4.1 的 package 邊界讓這步是機械式刪除（`BWFFullProfileDetailsVC` 因跨模組共用**不刪**） | DoD 全綠後 |

**先做的兩塊（低風險、解鎖後續）**：Phase 0 harness 與 Phase 1 骨架可並行；Phase 2 的 view-model 可仿 [LikesFeed.ViewModel](BWF/App/Classes/Modules/WhoViewedMe/SingleProfile/LikesFeed.ViewModel.swift)（已是 `@Observable`，是現成範本）。

---

## 5. 切換機制：具體檔案改動（Phase 1）

| 檔案 | 改動 |
|---|---|
| [VisitorsProperties.swift](BWF/App/Classes/Analytics/RemoteProperties/VisitorsProperties.swift) | 新增 `Implementation` enum + `@UserDefault` 屬性（預設 `.uikit`），仿既有 `screenVersion` |
| `FlagrConfigClient+Keys.swift` | 新增 key case，如 `visitors/implementation` |
| `FlagrConfigClient+Values.swift` | 解析該 key → 寫入 `visitorsProperties.implementation` |
| [AppMasterWireframe.swift:375](BWF/App/Classes/AppMasterModules/AppMasterWireframe.swift#L375)（`setUpForOldMainUI`）與 [:533](BWF/App/Classes/AppMasterModules/AppMasterWireframe.swift#L533)（`rebuildVisitorsTab`） | **兩處都**改成依 flag 選 `SwiftUIVisitorsWireframe` / `VisitorsContainerWireframe`（否則 `onFlagrUpdate` 一觸發 rebuild 就會退回舊版） |
| 新檔 `SwiftUIVisitorsWireframe.swift` | 實作 `LegacyWireframe`（`view` 回傳 `UIHostingController<VisitorsRootView>`），維持與 tab bar 掛載相容；**導航用 closure 注入**（仿 `ConversationWireframe`：建 VM 時塞 `openProfileDetails` / `openPaywall` / `openMatch` … closure，push/present 由 wireframe 持有，`BWFFullProfileDetailsVC` 維持 UIKit）——見 §4.1 |
| 新增 SwiftUI 模組 `Packages/DesignSystem/Sources/DesignSystem/Screens/Visitors/` | `VisitorsRootView` + `@MainActor @Observable` view-model + repository protocol（仿 `ConversationsList`），與 BWF/App 解耦、可 Preview、日後移除 = 刪資料夾 |

---

## 6. 驗證 / 待確認

**驗證（每階段）**
- Flag **off**：行為、送出的 analytics、network requests 與今日 byte-for-byte 一致（Phase 0 harness 跑 diff）。
- Flag **on**：新 tab 正常掛載、loader→內容、切 tab 不崩、空狀態正確。
- **位置授權 `denied/notDetermined → authorized` runtime 變更**：重打 Flagr 後 tab 正確更新（值層原地更新、結構層才重建），且不因初次 / 失敗 notification 重複重建（§3.1 的 diff）。
- 重點回歸：`visitors_pv` / `visitors_view` / `visitors_tab_switch`（一度移除 `6b348af01`、PM 要求加回 `9276a8053`）/ `logVisitorsProfileView` / `trackVisible/Hidden`（BehavioralTelemetry section change）仍送（附錄 A #12）。

**已決定（2026-06-02）**
1. **變體**：`default` / `filtersOnLikes(.v1)` / `.v2` **都還在用，不可下線** → 重寫須覆蓋全部變體並逐格驗 parity（§3 風險 3；Phase 0 建變體矩陣）。
2. **VisitorsManager 拆解**：networking → `VisitorsRepository`（架在 Network wrapper）、cache/分頁 → 新 VM、對外聚合值 → slim 共享 facade；先抽 protocol 後換 wrapper 實作（§3 風險 2）。
3. **app-lifecycle 順序**：重點是**消啟動 double-build**，做法為**每個子 tab 自行判斷是否替換 layout（含 Visitors）**= §3.1 收斂模型推廣到全 tab。
4. **runtime 重讀 config 的 UX**：外層 Container 蓋 Loading overlay → 結束後重畫 layout（§3.1 結構層分支）。
5. **Network wrapper：納入本次 scope** —— 作為 `VisitorsRepository` 的 transport；Phase 2 先抽 protocol、再換 transport（§3 風險 2 / §4.2 Phase 2）。
6. **config 變更職責分層**：**Master 判斷顯示/隱藏 tab + fan-out**；**child 自行決定 keep / re-init layout**（值層交 `@Observable` 自動）。初版 diff 分類見 §3.1：結構 flag（`filtersEnabled` / `filtersV2Enabled`）→ re-init；其餘（`screenVersion` / distance / blur / reskin …）→ keep。

**仍待確認**
- （無 block 開工項）逐 flag 最終分類在 Phase 2/3 抽 VM 時定案（§3.1）。

---

## 7. 延後決策：tab preselect（P2）與 You-Liked re-fire（P1）

> 2026-06-08 對 rebase 後現況 + 完整生命週期查證（workflow + 對抗驗證）後的兩個 parity 缺口結論。**P1 不修；P2（見下方 ⚠️ 翻案）。**

### ⚠️ 啟動 fetch 機制（2026-06-09 重大修正，推翻先前「lazy」前提）

legacy **啟動就 fetch visitors，與 tab 顯示無關** —— 觸發是 `VisitorsManager.init()` 註冊的一次性 `.flagrConfigUpdated` observer（[VisitorsManager.swift:295-308](BWF/App/Classes/Services/VisitorsManager/VisitorsManager.swift#L295)）：啟動時 singleton 被 badge 訂閱（[AppMasterView.swift:341](BWF/App/Classes/AppMasterModules/AppMasterView.swift#L341)）強制建立 armed → Flagr 完成 post `.flagrConfigUpdated` → **filters config 打 V5 `api/v5/users/visitors`（:517）；default config 打 V4 `/users/visitors/list`**（observer 一次性、自我移除）。frank 實機 v78 filters-V2 確認。**先前「lazy／只在切 tab 才打／counts 開 tab 前是 0」皆錯**（漏了這個 NotificationCenter observer；`BWFWhoViewedMeViewController.viewDidLoad` 只是讀已暖 cache:315-316）。

**新版 parity 破口（待實機確認 + Phase 5 驗）**：VM 在新版仍活著（badge 用），其啟動 fetch 照樣發 → 啟動 parity ✓。但新 tab 的 `VisitorsRepository`/`ListViewModel` **不訂閱 flagr、不啟動預載** → 開 tab 時自己**再打一次 V5**（legacy 重用 VM 暖 cache、不再打）→ **新版很可能「開 tab 多打一發 V5 + 首開 spinner」**。根因：legacy 一個 VM fetch 餵 aggregate+tab；新 tab 解耦資料層 → 兩發。真 parity 需「一次 fetch 餵兩邊」（= 聚合 store 設計）。**下一步：frank 實機確認新版開 Visitors tab 是否多打 V5、首開有無 spinner，再決定修法。**

### P1 — You-Liked `visitors_view` 重新出現時漏送 → **analytics 半邊已修（2026-06-13）；資料半邊（§9.1#1）仍 open**

legacy `BWFMyLikesVC.viewDidAppear:129-141` 每次出現都送 `you_liked` `visitors_view`（+ refetch + zero-coins）；新版原只在 `onTabChange`（`to==.youLiked`）+ pushed `YouLikedViewController.viewDidAppear` 送 → baseline pager 停在 youLiked 頁、bottom-tab 離開再回時漏送。
**修法（commit 待補）**：pager selection 抬到 `Visitors.ViewModel.selectedTab`（PagerView 保留 `@State` 為渲染源以維持 preselect 不跳，雙向鏡像到 ViewModel、等值 guard）；controller 存 viewModel、`viewDidAppear` 在 `selectedTab == .youLiked && youLikedList != nil` 時補送 `visitors_view(you_liked)`。順帶把 selection 變成可外部讀寫＝**deep-link 到 tab 的前置**（§8.3 規劃，現休眠）。
**仍 open**：legacy viewDidAppear 三件套裡的 **refetch + zero-coins**（§9.1#1）re-appear 時仍未補（每次重現都打 my_likes 的取捨，frank/PM 決定）。〔歷程：三事件曾移除 `6b348af01`、PM 確認仍在用而加回 `9276a8053`，本修法接續加回後解 P1 analytics 半邊。〕

### P2 — `preselectEligibleTabIfNeeded`（local 空 && worldwide 非空 → 開 Worldwide）→ **延後決定**

- legacy [VisitorsContainer.swift:90](BWF/App/Classes/Modules/New/Visitors/VisitorsContainer.swift#L90)（viewDidLoad）→ [:180-186](BWF/App/Classes/Modules/New/Visitors/VisitorsContainer.swift#L180) 同步讀 `VisitorsManager` singleton 決定；新版 pager 永遠開 `.local`（[Visitors.PagerView.swift:36](Packages/DesignSystem/Sources/DesignSystem/Screens/Visitors/View/Visitors.PagerView.swift#L36)）。
- **⚠️ 以下「幾乎不 fire」結論已被 2026-06-09 重大修正推翻**（見上方「啟動 fetch 機制」）：它基於「worldwide 在 viewDidLoad 時還空」的錯前提。實際上 default config 的**啟動 fetch（V4 local+worldwide）在首次顯示 tab 之前就暖好 singleton** → 首次顯示時 `viewDidLoad` 的 preselect 讀到暖資料 → **local 空 && worldwide 非空時其實會 fire**。所以 legacy preselect 在其目標情境**會觸發，P2 是真缺口、非 near-no-op**，下方「不移植」決議需重評。
- ~~**生命週期查證：legacy 其實幾乎不 fire。**~~（已作廢）preselect 在 viewDidLoad **同步**跑，而 worldwide fetch 是子 VC `viewDidAppear`（晚於 viewDidLoad）驅動。①首次進入：worldwide 還空 → 不 fire；②tab 再進入：容器 created-once-reused（`selectTab` 只設 selectedIndex）、viewDidLoad 不重跑 → 不 fire；③**唯一窗口 = `.flagrConfigUpdated`→`rebuildVisitorsTab`**（[AppMasterStore.swift:207](BWF/App/Classes/AppMasterModules/AppMasterStore.swift#L207) 確有呼叫）容器重建、viewDidLoad 重跑，且 `VisitorsManager` singleton 存活過重建（只登出才清）→ 若稍早逛過（worldwide 暖／local 空）才 fire。〔註：①的前提錯，故結論不成立〕
- **為何現架構不能乾淨修**：legacy 能 fire 正因 **singleton 比 view 長壽**；新架構 list 綁 wireframe（rebuild 造全新空 list），在那唯一窗口新 list 恰好空。`VisitorsRepository` 是 **stateless**（無快取，資料只在 `ListViewModel`）。→ 要不碰 `VisitorsManager` 又能 fire，**必須讓 list 資料活過 rebuild（session-scoped 快取）= Phase 2 的 slim facade／快取層**。同步讀新 list 永遠空＝死；async fetch-then-scroll 會 Local→Worldwide 跳＝regression（legacy 同步決定、從不跳）。
- **決定**：**現在維持「永遠開 Local」**（首次／每次 re-entry 都 byte-faithful，只差極罕見的 rebuild-暖快取 edge case）。**待 Controller-as-orchestrator 重構（§4.1 修正版／見下）+ 持久 Repository 快取（Phase 2）完成後再決定**：屆時可在 `VisitorsHostingController.viewDidLoad`（1:1 對應 legacy）同步讀**自家持久快取**決定初始 tab，不碰 VisitorsManager、不跳。若在那之前產品就要 parity，唯一忠實不跳法 = Controller `viewDidLoad` 一次性讀 `VisitorsManager.local/worldwideViewedMeVisitors`（對低耦合的**明確例外、產品決策**）。
- **機制前置**（兩條路都要）：pager selection 需從 `PagerView` 的 `@State` 抬到 `Visitors.ViewModel.selectedTab`（bindable），Controller 才設得動。
- **✅ 已解（2026-06-10，方案三：closure 延遲評估）**：上面兩條路都不需要了。`Layout.pager` 的 `initialSelection` 改為 **closure**，在 `PagerView` 首次 render（= hosting view 首次載入 = 1:1 對應 legacy `viewDidLoad` 時機）才讀 `VisitorsManager` 暖陣列 → 啟動 prefetch 已落地、不跳 tab、不需要 selection 抬升。讀 VisitorsManager 屬「明確例外」已註記在 code（與 legacy 同資料源）。

---

## 8. 新舊對照稽核與修正（2026-06-10）

> 來源：10 並行 agent 稽核（結論已併入 [QA 計劃 §7 watchlist](visitors-parity-qa-plan.md)）→ 16 項缺口逐項對抗驗證（10 real / 4 partial / 1 false-claim-but-real / 1 確認免修）→ 一次修正。Build 綠 + 39 測試綠（`defaultBaselineEmitsExpectedRequests` 首跑 flaky、重跑綠，為 app launch 時序非 regression）。**裝置 QA 未跑。**

### 8.1 已修（High H1–H10 全修）

| # | 缺口 | 修法（關鍵：不疊床架屋的 seam） |
|---|---|---|
| H1 | filters config 下 You Liked 入口消失 | 新檔 `VisitorsHostingController+YouLiked.swift`：右上鈕（樣式照抄 VisitorsContainer）→ push `YouLikedHostingController`（重用 pager 的 youLiked grid 組件；viewDidAppear = 照 MyLikesVC：visitors_view + 重抓 + 零幣刷新；hideProfile observer） |
| H2/H3 | 詳情 decline 死（position nil + 無 store handler）/ like・block 後列表不更新 | **`masterStore.addHandler(self)`**（= legacy WhoViewedMeStore 的註冊法）：controller 實作 `InputHandler` 接 `DeclineLike`（gate→paywall / log+POST→`DeleteSelected(isDeclined:)`）、`DeleteSelected`（legacy 移除規則：hidden∥declined∥interacted；declined → DailyPicks 通知）、`ShowPaywall`。`openDetail` 改傳真 `position` + 記錄 `VisitorsDetailContext`（= legacy `selectedViewer`，含其「保留最後值」的 quirk） |
| H4 | 中途買 premium 不解 blur/gate | `GridPresentation` struct→**@Observable class**；blur/gate/lastSeen 由 `refresh(_:)` 原地更新（`premiumStateUpdated` input + 每次 viewDidAppear，= legacy 兩條路）；onSelect gate 改 live `canInteract()`；CTA 顯示改 `presentation.nonPremiumGated` |
| H5 | grid like-back 鈕變裝飾 | `GridCell` reaction 圓徽改 Button；`CellActions.onLikeBack`（app 端先 `vibrateDeviceLite` 再走同一條 onLike，decline cell 不震 = legacy） |
| H6 | grid like 成功後移除用錯 key（真 bug） | `list.remove(id: viewer.uuid)` → `viewer.userId ?? ""`（uuid 是 decline key、永不等於 `Visitor.id`）。**feed 不加移除**（legacy feed 本就 fire-and-forget + advance，verifier 提議的 feed removal 是過度修） |
| H7 | 截圖保護消失 | controller 改繼承 **`LimitScreenshotViewController`**（ObjC bridge，root = SecureView；不能用 package 的 Swift 類因 ObjC generated header 看不見），SwiftUI screen 改 hosting child 內嵌；AppMasterView 兩處型別判斷 `UIHostingController<Visitors.Screen>` → `VisitorsHostingController` |
| H8 | tab 再進入不刷新 + 空列表永遠空 | `CheckForRefreshVisitors`（同 handler 接）→ 非空 distance lists `refreshFromTop()`（新 API：page-0 重抓、未見者**插頂**、不動分頁游標）、filtered list `loadFirstPage()`（legacy replace 分支）；空列表 → viewDidAppear `loadFirstPage`（= legacy VC.viewDidAppear） |
| H9 | feed 卡片 identity 永不相等（每次重建 VM + 重發 profile_view） | `CardFeedViewModel` 比較改自家 `currentVisitorId == visitor.id`。**不改共用 `ProfileMapper`**（verifier 的 Option A 會破 legacy feed 的 uuid 比較） |
| H10 | my_likes 首抓 page=0 vs legacy page=1 | youLiked loader closure `fetchMyLikes(page: page + 1)`（鏡像 legacy caller；repository 保持 manager passthrough）。**不改 `ListViewModel` 分頁語意**（verifier 提議會弄壞 distance 的 `offset=page*50`）。loader 並補 legacy 的 3 次 retry + 終敗 `error_dialog_shown` |

### 8.2 已修（Medium）

- **preselect 時機**（P2 真解）：`initialSelection` 改 closure（§7 P2 註記）。
- **`profile_details_blur_enabled` 未接**：`GridPresentation.detailsBlurred`（!premium && flag && !female）→ `GridCell` 渲染兩條 blur bars（寬 50%/75%、高 8、corner 4，照 xib）。
- **decline 不發 DailyPicks 通知**：grid decline 成功 + 詳情 decline（經 DeleteSelected）都補發 `.removeDeclinedVisitorsFromDailyPicksBatch(object: userId)`；feed 不發（= legacy）。
- **hasHadMinVisitor latch 弱化**：`recomputeBadge()`（= legacy dataRefresh sink 的位置語意）檢查 badgeLists 串接數 ≥ `min_visits` → 設 latch + `refreshVisitorUpsellSubject.send(true)`（subject 從 private 改 internal，一字）。
- **per-cell BehavioralTelemetry**：`CellActions.onCellVisible/onCellHidden` ← GridView cell `onAppear/onDisappear` → `trackVisible/trackHidden(id: fbId)`（distance tabs only，= legacy；You Liked legacy 本就沒有）。
- **You Liked 活性**：每次切入 `loadFirstPage()` 重抓 + 零幣 `updateUserCoins`（2s retry）+ `.hideProfile` observer 移除；錯誤 retry×3 + `error_dialog_shown`。
- **supercharge**：已 supercharged → 提示 HUD（不再直接開 IAP）；`SuperchargeIAPStatus` 成功/失敗 HUD（handler 接）。
- **NEW 標基準凍結**（原 Low）：`lastSeenAt` 隨 presentation 每次 appear 刷新,順手修掉。

### 8.2b 裝置 QA 抓到的兩個追修（2026-06-10 晚，frank 實機）

- **pushed You Liked 出現系統 back button**：legacy 沒有「特別隱藏」——`BWFMyLikesVC.setupNavigationCloseButton` 設 `leftBarButtonItem`（You Liked label），UIKit 預設 left item 取代 back。新版同樣設了（外加 `hidesBackButton`）仍冒出來——**pushed `UIHostingController` 會自行重繪 navigationItem**。修法：`YouLikedViewController` 改 plain UIViewController + hosting child（與主 controller 同款 pattern），navigationItem 全程 app 持有。
- **filter 與 list 之間的 padding 歸屬**：legacy 每個 grid scene 都有**靜態 10**（[Whoviewedme.storyboard:237](../BWF/App/Resources/Storyboards/Whoviewedme.storyboard#L237) collection top 約束、MyLikes xib 同款）+ **滾動 10**（sectionInset top）——滑動邊界在 chip bar 下方 10pt，內容從那裡滑入。新版原本只有滾動 10 → 邊界貼 chip bar。修法：`GridView` 在 `.clipped()` 外加靜態 `.padding(.top, 10)`（保留內部滾動 10）；`.single` header 補 legacy 容器的 safeArea+10（`Screen` 內 `header.padding(.top, 10)`）。pager 路徑同受惠（legacy 每頁 VC 本就有靜態 10）。
- **grid/feed loading 樣式**：legacy 所有 grid（distance/filters/MyLikes）loading 都是 `BWFProgressHUD` 的 **Lottie**（`03_Loading_B_Original_01`，68×48 loop）；新版原本是系統 `ProgressView`。**最終修法（2026-06-11 收斂，取代第一版 app 注入方案）**：package 早有官方 Lottie SwiftUI 支援與**同一顆動畫資產**（`AnimatedView`/`ActivityIndicatorViewModifier` + Media.xcassets dataset）→ 新增 package 內建 `LoadingIndicatorView`（官方 `LottieView` + `.asset(bundle:.module)` + **CA 引擎 `.automatic`**，main-thread burst 不凍幀）；`GridView`/`CardFeedView` 的 `.loading`/`.failed` **收進 package**（failed 的 retry 用 view 自己的 viewModel），`EmptyKind` 刪除、`emptyState` 注入縮為 `() -> AnyView`（只剩 `.empty`——gender/premium/IAP 矩陣才是 app 知識）。app 端 `VisitorsLoadingView`/`emptyState` builder 刪除、共用 `LottieView` wrapper 還原。**教訓**：app target 的自訂 `LottieView` 同名遮蔽官方 API（在 app 內編不過 ≠ 官方 API 不可用）；加注入通道前先查 package 既有件。
- **pushed You Liked 的叉叉**：第一版漏抄 legacy `makeCloseButton` 的 40×40 約束與 `configurationUpdateHandler`——`.plain` config 的 `baseBackgroundColor` 無效，**圓底其實是 handler 畫的**（白 0.1 / highlight 0.2）。已照抄補齊。
- **空狀態確認免修**：legacy MyLikes 空狀態 = `BWFMyLikesRanOutOfFriendsNewView`（🚦+Still Waiting?+desc+Browse Nearby）；新版 `StillWaitingEmptyView` 在 3d 時就是照它做的（同 chrome、文案注入），pushed 畫面已接、`openBrowseNearby` 行為一致（只切 tab 不 pop）。
- **首開 loading Lottie 凍幀（frank 實機）**：`LottieView` wrapper 用 `renderingEngine: .mainThread` → 進 tab 的 main-thread burst（SwiftUI 樹首建 + 背景圖 decode + Lottie JSON 解析 + response 50 筆 parse 都在 main）期間動畫排不進幀 → 定格後才作動。修：wrapper 加 `renderingEngine` 參數（預設 `.mainThread` 不影響他處），Visitors loading 傳 `.automatic`（CA 引擎、render-server 驅動，main 再忙不掉幀）。**注意：loading 階段本身仍是 legacy 沒有的（watchlist #4 暖快取未 seed），動畫順了但結構性差異還在——真解是 Tier 2 seed-unification。**

### 8.2c 購買 VIP 後的 parity（2026-06-11，frank 實機抓到；build 綠 + 26 測試綠）

- **流程澄清（重要）**：legacy 購買後**並沒有重抓 Flagr**——`AppMasterStore.onVIPUploadSuccess`（:396-409）做的是 `premiumStateUpdated` dispatch + `updateTabs()`（只切 freshFaces/hotPicks/doubleDown 顯隱）+ `updateUserCoins()` + `activateRemoteConfig()`（Firebase）。這些全是 **app 層共用路徑，新舊一模一樣會跑**；觀察到的網路活動是 coins/RemoteConfig。tab 內真正的差異只有下面三項，已修：
- **Boost Profile banner（補建）**：legacy `setBoostProfileHeader`（premium && 未 supercharged && 列表非空 → 97pt 白卡：自己頭像 45×56 + 星×2/雲×2/上箭頭 + `visitorsSuperchargeBannerTitle/Description`，點擊 → `supercharge_cta_tapped(source: visitors_upsell_banner)` → supercharge 流）。新版：package 新增 `Visitors.BoostProfileBanner`（裝飾資產 5 個 imageset 複製進 package `Media.xcassets/VisitorsBanner/`；頭像佔位用 package 既有 `avatarPlaceholder`，~~與 legacy `noImgAvatar` 視覺略異——記錄在案~~（⚠️ 2026-06-12 勘誤：兩 imageset 包的是 **byte-identical** 的 `ic_avatar.svg`，視覺差不存在））；`Content.grid` 增 `header: () -> AnyView?` 槽（per-render 評估），app 端 `boostBannerHeader(list:masterStore:)` 注入 **local/worldwide/filters** 三處（= legacy 同一 VC 三 distance 都有）；feed/You Liked 無（legacy 同）。`purchaseSupercharge` 通用化 `source:` 參數（empty state 與 banner 共用）。
- **supercharge 購買成功後不刷新**：`SuperchargeIAPStatus` handler 補 `refresh(presentation)`（= legacy :225 `checkAndUpdateEmptyStateViews` + `setBoostProfileHeader`）→ banner 隱藏、空狀態換 supercharged 文案。
- **空狀態在 premium 變化時不重算**（edge：空列表停在 GoPremium empty 時買 VIP）：`GridView` 的 `.empty` 分支補 observation 依賴（`.id(presentation.nonPremiumGated)`）→ presentation refresh 觸發空狀態重建、矩陣重讀（= legacy `checkAndUpdateEmptyStateViews` on premiumStateUpdated）。
- **掃描確認無其他 premium 缺口**：blur/tap-gate/CTA（live ✓ 前輪已修）、空狀態矩陣（live ✓）、supercharge 倒數（live ✓）、badge 清除（AppTabBarController 按 **tab index** 判斷、非 VC 型別 → 新版同樣吃到 ✓）、coins（You Liked 進場 ✓）。
- **banner 視覺追修（frank 實機）**：白卡圓角 = **8**（storyboard `layer.cornerRadius` 是 **integer** 且 keyPath 帶 `layer.` 前綴——上次 regex 只搜 `cornerRadius` 漏掉）；上邊距 = **10**（97pt 槽 bottom-pin 87pt 卡 → 槽頂留 10；`R0Y.top = topLayoutGuide+0`）。`.clipped()` → `.clipShape(RoundedRectangle(8))` + `.padding(.top, 10)`。
- **banner 對齊重寫（frank 實機：avatar 偏下）**：第一版抄的是 IB **渲染 frame**（stale；雲朵 image view 無尺寸約束、IB 標 ambiguous），不是**約束圖**。真實約束：avatar `centerY = card.centerY` + 自帶 `layer.cornerRadius=8`；starA `top=avatar.top, trailing=avatar.leading+6`；starB `bottom=avatar.bottom-3, leading=avatar.trailing-4`；arrow `centerY=title.centerY+8, trailing=title.leading+4`；雲朵釘卡片**底邊**（左 leading-12/bottom+6、右 trailing+14/bottom+6）跑 **SVG intrinsic size**（cloud_a 53×44、cloud_b 90×45，非 IB 的 63×48/110×54）；z-order = avatar/星 < 雲 < 文字。SwiftUI 端偏下主因：`.frame(height: 87)` 預設 **center** 對齊，ZStack 自然高度（文字決定）< 87 時整簇下沉。重寫成 alignment-driven（avatar `.frame(maxHeight:.infinity)` 置中、星星 overlay 在 avatar、arrow overlay 在 title、雲用 filling-frame 夾在 avatar 與文字之間）；卡片 clip 改回非 continuous 圓角（UIKit `layer.cornerRadius` 是 circular）。**教訓：storyboard parity 要讀 constraints + outlets + 資產 intrinsic size，IB frame 不可信。** 後續再把全部 `.offset` 換成 `.alignmentGuide`（frank 提議）：約束常數原樣入碼（如 starA `alignmentGuide(.leading) { $0[.trailing] - 6 }` = `trailing = avatar.leading + 6`），不再把裝飾物尺寸烘進 offset 常數，資產尺寸改了也不歪。
- **GoPremium 空狀態 full-bleed 補做（frank 實機：新版圖小、光暈有邊界感；原 3d-3b PARKED 項）**：legacy `GoPremiumView.xib`（8LW）= 圖**全寬出血**（leading/trailing 貼滿、無寬高約束 → 高吃 intrinsic **363pt**、`scaleAspectFill`+clip、`centerY = 容器中心 − 70`）+ 文案堆疊釘底（bottom 25、左右 16、spacing 14）疊在圖的下緣光暈上。**xib 的 `BWFSwipGradientView`（透明→黑 60% 釘底）不搬**——legacy controller 把它關了（`goPremiumView.gradientView.isHidden = true`，lazy var :156）；第一版照 xib 搬進來被 frank 實機抓到「下方多一層黑遮罩」後移除（又一個「xib 預設值 runtime 被關」案例，同 segment 底線教訓）。此狀態 `userInteractionState=false` → hitTest 只放行 CTA（背景點擊死的，新版不用補）。新版 `GoPremiumEmptyView` 由 180×180 置中改寫為 GeometryReader+ZStack 復刻上述構圖；CTA 維持 `ctaPrimaryButtonStyle`（~~52 vs legacy 46/corner23，既有記錄的簡化~~ ⚠️ 2026-06-12 勘誤：legacy **runtime 實為 52/corner16**（xib 值被 code 覆寫）→ 新版近乎全等、非簡化；殘差只剩 `CtaButtonStyle.makeBody` 硬編 16-**bold** 蓋掉 style 的 semibold——元件級既有 quirk，`CtaButton.swift:17`）。offers 態仍 deferred（3d-5）。
- **硬編碼色掃描（frank 問 banner 描述色有無 token 引出）**：banner 描述 #280E4A → `.midNight`；**SegmentHeader 底線 #FF006A 是真 parity bug** —— 那是 `CustomSegmentHeaderView` 的漸層**死預設**（`MultiPageSegmentController.swift:68` runtime 無條件覆寫 start=end=`primaryPink`，#FF0099 才會上畫面）→ 已改 `ReusableStyle.Color.primaryPink.flatColor`。GridCell 的 pill（(0.23,0.62,1)·0.6 / (1,0,0.41)·0.6）與底部漸層（(0.23,0.41,1)·0.7）= legacy 代碼字面值逐字移植（`BWFNewViewerCell:110/120`、`BWFProfileTransparentCell:212`），無 token 完全相等 → **保留硬編碼**（換近似 token 會改色破 parity）。**全範圍複掃（2026-06-12，frank 要求）**：再換 3 處 exact match —— SegmentHeader 標題 `.white`→`.neutralWhite`、底部髮絲線 `Color.white.opacity(0.08)`→`.neutralWhite.withOpacity(0.08).flatColor`（`Color(hex:)` 支援 8 位 RGBA，0.6→0x99 精確）、GridCell New tag `Color.black.opacity(0.6)`→`.primaryBlack.withOpacity(0.6)`。其餘確認乾淨：GridCell:203 是 runtime 內插運算非常數、`Color.clear`/Preview 鷹架不算、empty views/LoadingIndicator/LoadingFailed/app 端 SwiftUIVisitors 全 token 化。avatar placeholder 維持與 legacy 一字不差（白 glyph 白卡隱形；legacy 同；自己頭像 super-low URL 幾乎秒載）。
- **買完 Supercharge 後 banner 還在（frank 實機質疑）—— 「當下不消失」parity 一致；但後續觸發鏈新版有兩個真缺口（第二輪實機抓到、已修）**：legacy `SuperchargeIAPStatus` handler（:217-225）只做 HUD + `checkAndUpdateEmptyStateViews()`，**不呼 `setBoostProfileHeader`**；新舊同走 `AppMasterStateInput.SuperchargeIAP`（legacy `PurchaseSuperCharge` → `checkAndDisplaySuperChargePurchase` 也是 dispatch 它）。**⚠️ 時序更正（追查 InAppProductsService 後）**：`isSuperChargeUser` 在 `SuperchargeIAPStatus` dispatch **之前**就翻 true —— `uploadPurchaseToApi` 等 server 收據驗證回應 → `saveUserPurchased`（`BWFAccountController.m:388`，同步設 `user.isSuperChargeUser = true`）→ 才 `onComplete` → dispatch。所以舊版買完**當下**：空狀態**會**變電池（handler 的 `checkAndUpdateEmptyStateViews`，flag 已 true）、banner **不會**消（沒呼 setBoostProfileHeader、等下個觸發點）。新版（鏡像欄位修後）：handler 的 `refresh` 寫出 `superchargeUser` false→true 真值變 → **當下 banner 消失 + 空狀態變電池**（banner 比 legacy 早一拍消失 = 刻意小幅優於，frank 要求「錯誤狀態不在」）。
- **第二輪實機（default config、worldwide 空狀態買 supercharge）抓到的兩個新版缺口（已修，build 綠）**：
  1. **pager 切 local/worldwide 沒有任何 refresh**：legacy 每次換頁 = 子 VC `viewDidAppear`（:329-333：list 空→`fetchViewers` 重抓；非空→`checkIfPremiumUserAndUpdate`）；新版 `onTabChange` 原本只記 analytics。修：`onTabChange` 補「目標 tab list 空→`loadFirstPage`（youLiked 除外，既有分支已重抓）+ `refresh(presentation)`」。
  1c. **paywall 關閉後空狀態「可見 reload」（frank 實機）**：觸發鏈 = paywall dismiss → `viewDidAppear` → `reloadEmptyLists()` → 空列表 `loadFirstPage`。查證 legacy **同樣會 refetch**（同一個 viewDidAppear :329 路徑）→ 網路行為 parity；但 legacy 是**靜默**（`loadingViewToBeDisplayed(false)` 收 HUD、UI 不動、回來還是空就什麼都不變），新版 `ListViewModel.fetch` 無條件翻 `.loading` → 空狀態被翻成 Lottie 再翻回 = 可見閃爍。修：`fetch` 改 `if state != .loaded { state = .loading }` —— 已載入過的列表（content 或 empty）重抓一律靜默、維持現畫面；首載/failed 重試照樣顯示 loading；失敗語意不動（測試 :134/:163 仍綠）。`refreshFromTop` 本就靜默，三者一致。23/23 測試綠。
  1b. **↑refetch 部分隨後拿掉（frank：每次滑到空 worldwide 都重打 API 不可接受）**：查證 legacy **確實**每次滑到空 tab 都重打（viewDidAppear :329 → `requestNextPage` → `fetchWorldwideVisitors:359`，唯一 guard 是 in-flight），所以拿掉是**刻意偏離 legacy**（降低冗餘請求，方向同 double-fetch watchlist）；空 tab 復原仍靠 controller `viewDidAppear` 的 `reloadEmptyLists()`（= bottom-tab 回來時，這部分與 legacy 同）。`refresh(presentation)`（純本地）保留在 onTabChange。
  2. **banner / 空狀態 gating 讀不可觀察的 account flag**：flag 默默翻轉後沒有任何 observable「值變化」；實機證實 `refresh()` 的同值寫入**不足以**讓視圖重評估（我先前推論「@Observable 同值寫入也觸發」與實機不符——切 tab 會更新只是視圖重建順帶重讀）。修：`GridPresentation` 增 `premiumUser`/`superchargeUser` 鏡像欄位（refresh 寫入，flag 翻轉時是真值變）；`boostBannerHeader` 改 gate 在 presentation 鏡像上；GridView `.empty` 的 `.id` 擴成三 flag 陣列。效果：banner 消失/出現與空狀態電池切換在 ①pager 換頁 ②bottom tab 離開再回 ③premiumStateUpdated ④SuperchargeIAPStatus（若 flag 已翻）都確定性重評估 = legacy 觸發面。
- **同輪 sweep 抓到 singleProfile 真缺口（已修，build 綠）**：legacy feed host 釘在 `viewersList` frame（+SingleProfile:89-95），banner 的 97pt 帶在 collection 上方、`hideOuterEmptyStates` 不碰、`setBoostProfileHeader` 照跑 → **legacy 在 single_profile 下 banner 照樣顯示在 feed 上方**；新版 `Content.feed` 原本沒 header 槽。修法：`Content.feed` 增 `header: () -> AnyView?`（鏡像 grid），`CardFeedView` 以 `VStack(spacing:0)` 把 header 放 feed 上方（banner 自帶 top 10、card 自帶 padding 16，間距鏈與 legacy 約束等價）；gating list 照 legacy quirk —— filters+singleProfile 用 filtered list、default+singleProfile 各 tab 用**自己的** local/worldwide list（legacy 各子 VC 的 `visitorsList`，feed 內容才是 filtered）。另把 `boostBannerHeader` 加 `presentation` 參數並在 closure 內讀 presentation 註冊觀察 —— feed body 沒有其他 presentation 讀取，沒這個 read 的話 supercharge/premium refresh 不會讓 feed 的 banner 重評估（grid 是靠 body :102 既有讀取，原本就會）。（⚠️ 2026-06-12 勘誤：實際 gate/觀察讀的是 `premiumUser`/`superchargeUser` **鏡像欄位**（`VisitorsHostingController.swift:564-565`），非原文寫的 `nonPremiumGated`——後者只有 GridView 的 CTA overlay 在讀。機制正確、措辭錯。）

### 8.3 確認不修 / 已知簡化（記錄在案）

- **paywallProfile**：全 repo 只寫不讀，**確認死碼**，新版不搬；Phase 6 連 legacy 寫入一起刪。
- **You Liked `updateCells`/`flirtMessageSent` observers**：未搬——legacy 靠原地改 mutable entity + reload cell；新架構是 value snapshot，要等價只能重抓，成本/收益不符。切入 tab 本就重抓（上面已補），自然收斂。
- **my_likes 錯誤 UI**：事件對齊（error_dialog_shown）；視覺以 in-place `LoadingFailedView`（含 retry）替代 legacy 的 `BWFInAppNotification` banner。
- **re-entry refresh 失敗靜默**（保留現有內容）；**空列表 viewDidAppear 重抓涵蓋全部 lists**（legacy 只有可見 page 的 VC——小幅 over-fetch，同端點）。
- **MyLikesVC.setupUI 的無條件 updateUserCoins**：未搬（coins==0 進場會刷,夠用）。
- 對照文件其餘 deferred 項不變：GoPremium offers（3d-5）、full-bleed、detail swipe-paging、SDWebImage prefetch、Phase 2c/2d/2e、Phase 6。

### 8.4 對 verifier 修法的三個否決（修 A 壞 B 案例，留檔防回潮）

1. H9 勿改 `LikesFeed.ProfileMapper` 的 id —— legacy feed 比 `profile.id == viewer.uuid`，改了破 legacy。
2. H10 勿改 `ListViewModel.fetch` 先 +1 —— distance 列表 `offset = page*50` 全錯位。
3. H6 勿給 feed 加 like/decline 移除 —— legacy feed 本就不動 list,加了反而破 parity。

### 8.5 全面複稽核結果與待決清單（2026-06-12，39-agent workflow，對 working tree）

> 完整證據已併入 [QA 計劃 §7 watchlist](visitors-parity-qa-plan.md)；此處只記決策面。稽核盲區：analytics 逐事件清點 / lifecycle / 新碼 swiftuiCorrectness 三個 sweep 因 session 限額未跑完。

> **⚡ 真 Bug 第一輪修正（2026-06-12，build 綠 + 受影響測試全綠：ListViewModel 13/13、CardFeedViewModel 10/10、ParityDriver 3/3；逐項狀態見 [QA 計劃 §1 驗證狀態總表](visitors-parity-qa-plan.md)）**：對「新碼 bug」做了「修了不破 parity」的安全子集。
> - **已修 5**：B2（cursor reset 移到 isFetching guard 後、gate `isFromStart`）、B3（reloadEmptyLists 加 `.failed`）、B6（DeclineLike 先校正 `detailContext.viewer`）、B9（`.userLoggedOut` observer 清 buttonTipToken、不動持久旗標）、HARNESS（driver my_likes 改驅動 production 的 `page=1`、斷言 `.int(1)`）。
> - **刻意延後 2**：**B10**（badge race 須走下方 §8.5#2 的 2e→observer→單寫者順序；半修在 rebuild 瞬間清掉 badge 或被 VM refetch 蓋，更糟）、**B8**（複驗 partly——現行 handler 拓撲無受害者，現狀可接受）。
> - **複驗更正 3 處**：**B1 stale**（deinit removeObserver 早已存在＝非 bug）、**§9.2 V1 條件寫反**（tap-anywhere＋擋 scroll 是 visitorCount **≥6**、非 <6）、**B8 誇大**。

**驗證結論**：§8.1–8.2c 全部修法逐項對抗驗證**通過**（9 組、零 refuted）。兩個範圍保留：pager 內嵌 You Liked 的 bottom-tab 再進入不刷新（segment 切換有做）；DeleteSelected 只移 detailContext.list（legacy 雙列表都移 + 同步 VisitorsManager 陣列）。

**新缺口待決（修 or 收進 QA watchlist，逐項見對照文件 §9.2）**：
- **High ×1**：~~filters config 空列表缺 `hasActiveFilters` gate~~（✅ **2026-06-12 已修**：grid emptyState 補 gate、沒選 chip → distanceEmptyContent 矩陣；同輪一併修 **B5**：filteredList 移出 replaceLists）。
- **Medium network ×5**：CheckForRefresh 不重置 cursor、last-page raw-count 判斷在 hidden-heavy 頁分岔、fetch 失敗無全域 error banner、You-Liked 再進入截斷分頁 tail、非 premium 男可分頁/下拉（gate 在 `nonPremiumGated` 即可，S）。
- **Medium visual ×5**：GoPremium overlay 互動模型（tap-anywhere/scroll-block）、過期 supercharge 判定（non-nil vs 未來）、cell 第二層黑漸層+placeholder 底色、feed 空狀態置中 vs top+80+卡片多 16pt top、refresh/error HUD chrome。
- **Low**：banner 標題色 #FF0099 vs legacy #FF006A（**真缺口**，與 segment 底線案例方向相反——banner 無 runtime 覆寫）、字級漂移（標題 30 vs 28 等）、微動畫、`seenProfileOnVisitorsTab`、grid 側圖片 prefetch、空 filters_list 仍掛 chip bar、malformed response 重試語意。

**新碼 bug（非 parity，建議即修，見對照文件 §9.3）**：`.hideProfile` observer 無 deinit 移除（每次 rebuild 漏一個；YouLikedVC 有正確做）、`loadFirstPage` cursor-reset race（reset 在 isFetching guard 前）、failed-empty 永不自動恢復（reloadEmptyLists 不含 `.failed`）、hidden-decline 多發 DailyPicks 通知、default+singleProfile 的 filteredList 誤入 replaceLists、refreshFromTop 保留舊 Visitor 值＋GridCell `==` 忽略 interaction、SwiftUIVisitors 層零 logout 處理（static buttonTipToken 跨帳號）、badge 雙寫者 race 的實證路徑。

**刻意偏離（已拍板，防回潮——harness 跑出 diff 屬預期，見對照文件 §9.4）**：onTabChange 空 tab 不重抓、refreshFromTop 的 isFetching check（不複製 legacy 資料毀損 bug）、ShowPaywall 單發（legacy 每個已載入 grid VC 重複發）、supercharge 成功當下 banner 即消、singleProfile 首滑 worldwide 不多打 V5、per-list in-flight guard、已載入列表靜默重抓。**已全部收進 QA 計劃 §4 watchlist。**

**Parity harness 自身（先修再擴矩陣）**：driver 用 `page=0` 驅動兩邊 my_likes，但 production 兩邊都從 page=1 起（legacy 先 +1、新 loader `page+1`）→ **harness 驗的是 UI 永不發出的 request 流**；矩陣仍 1/10 格 network 半邊；N1/N4/B2 等 VM 層差異 repository-層 harness 天生蓋不到，擴格時要納入 re-entry/refresh 情境。

**優化建議（4 個審查彙整，behavior-preserving 優先）**：
1. **立即（S，兼 bug fix）**：hideProfile observer deinit、cursor reset 移到 isFetching guard 後、failed-empty 納入 reloadEmptyLists、`CardFeedViewModel.retry()`/`reload()` 去重、GridCell 的 interaction 漸層移出 init（現狀破壞 Equatable skip）、`GridPresentation.refresh` 同值寫入加 equality guard。
2. **資料層排序（⚠️ 2026-06-12 修訂：2e 已嘗試並拍板 revert）**：~~2e 先做~~——same-signature no-op guard 與 VM observer once-per-session gate 當天都做了、同日 `git checkout` 還原（從未 commit）。**Revert 理由**：mid-session flagr post 是罕見事件（觸發源已盤點：權限變更/設定頁改位置/DailyPicks denied-recovery resume，非每次 resume）；no-op 會切斷 swiftui 的位置變更資料刷新（uikit 有 VM→store sink、swiftui 沒有＝不對稱）與 badge「不進 tab 就更新」管道（frank 拍板：badge 被動保鮮＞省 2 發請求）；複雜度（signature＋CheckForRefresh 補丁＋B5 前置）配不上收益。**事實更正**：`recomputeBadge` 是 onChange-only、badge subject 跨 rebuild 存活→「rebuild 清 badge」不成立、2e 非 badge 單寫者前置。**修訂後順序**：**warm-seed**（首次 page-0 讀 VisitorsManager 暖陣列，殺 double-fetch；**務必 fire `onLoaded(isFromStart:true)`**——三事件已加回 `9276a8053`，onLoaded 又在餵 distance `visitors_view` + filters `no_results`，seed 不 fire 會破 parity）→ **§3.1 收斂模型（2e＋value-refresh 管道）在 warm-seed 之後一次做** → badge 單寫者 → **Phase 5 矩陣 → Phase 6**；2c 等 v1 死後用 trim-VM（非草稿的 aggregate store）、2d 在矩陣後以 request-capture diff 驗證 transport swap。
3. **Orchestrator（M，可後置）**：1143 行 controller 拆 ConfigMapper/LayoutBuilder/ActionHandler + closure-based dependency seam（factories 可單測）；You Liked/distance-tab wiring 各收成參數化 builder；`shouldShowDeclineLikes` 三種 freshness（init 凍結/live/detail-live）收成單一 policy；`topInsertLists`/`replaceLists` 收斂成單一 reentry 集合（動詞收進 `ListViewModel` 的 refreshStyle——2026-06-12 評估過可合、刻意不單獨動，留到這輪一起）。
4. **SwiftUI（需 Instruments 驗證再動）**：分頁觸發提前（≈legacy 預抓 lookahead，兼 parity）、blur 改 SDWebImage transformer、`.id([Bool])` 改顯式依賴、CardFeedView 改 containerRelativeFrame。
5. **EmptyState enum state machine（附錄 A #8）**：L，**Phase 6 後**再做（現在動會擾動 parity 面）。

---

## 附錄 A：重構踩雷點（migration map）

> 各 Phase 的「對應附錄 A」欄指向此處編號。重構**決策**已在 §3.1 / §4.1 / §4.2 定案；此處只留「從哪個 UIKit 構造遷出、SwiftUI 對應的坑」。

| # | 從（現狀 UIKit） | SwiftUI 遷移坑 / 對策 |
|---|---|---|
| 1 | `LegacyWireframe` 直接操作 nav stack（`showYouLiked` / `showViewerDetails` / `showPaywall` / `showMatchScreen`） | 走 closure 注入（§4.1 #3），**不**換 `NavigationStack` |
| 2 | `StoreSubscriber.addSubscriber` 在 `viewDidLoad`、`stateDidChange` 大 switch + 視覺副作用 | 換 `@Observable` VM（§4.1 #2）；副作用拆成語意事件流，別在 view body 寫 switch |
| 3 | `CVCellPresenter`（cell 配置/prefetch/間距/大小/可見性）+ scroll-driven 分頁 + `BWFWhoViewedMeTrailingView` footer + `UIRefreshControl` | `LazyVGrid` 無 cell-disappeared callback → telemetry 靠 `onAppear/onDisappear`；分頁靠最後一格 `onAppear` / `scrollPosition`；footer 用 `.fixedSize`；下拉用 `.refreshable` |
| 4 | `MultiPageViewController`（滑頁 / segment header / `scrollToPage` / `didSelectTab`）+ `preselectEligibleTabIfNeeded` hack + `disablePagerBouncing` | iOS 17 paging ScrollView + `.scrollPosition`（`TabView(.page)` 首滑誤排版）；無 `pageDidAppear` → tap/scrollPosition setter 補 `logVisitorsTabSwitch` + section-change telemetry（`logSectionChange`）；**preselect 見 §7（查證後不可用 `@State` 初值——資料非同步可得；延後決策）**；bounce 需新 scroll API / introspect |
| 5 | `BWFWhoViewedMeViewController` 由 storyboard 載入、大量 IBOutlet | 全重畫成 view tree，storyboard 可丟 |
| 6 | Filters 已 SwiftUI，但 V1/V2 各自 VM、V2 有 `applyValueChange` | 別硬合 V1/V2；移除 `UIHostingController` 包裝直接持 VM |
| 7 | SingleProfile feed 用 `objc_getAssociatedObject` 黏 host/VM/tipToken；`LikesFeed.ViewModel` 靠 11 個 callback 注入；`ButtonTipOverlay` present 在 window | 三者改 `@State`/`@Bindable`；callback 改 `@Environment` router；overlay 改 `.overlay`/`fullScreenCover` |
| 8 | 空狀態 `checkAndUpdateEmptyStateViews` 用 boolean + `isHidden`（singleProfile / filters 空 / female / male） | 整成單一 `enum EmptyState` |
| 9 | 跨模組 modal 走 `appCurrentController().present` | 無「找 top VC」好方法 → 保留 UIKit 入口 / router（§4.1 #3） |
| 10 | `Theme.poppins(isTreatmentOn:)` closure、view 內讀 `RemoteConfigProperties.shared` | 改 `@Environment` 注入 theme（可測） |
| 11 | container 訂閱 `.flagrConfigUpdated` 只 title + reskin | config 餵 `@Observable`；runtime 模型見 §3.1 |
| 12 | `trackVisible` / `trackHidden` 由 `CVCellPresenter` 驅動 | `GeometryReader` / `onAppear`-`onDisappear`；注意 `LazyVGrid` disappear 時機不準 |
| 13 | `presentProfileDetails` 三明治（SwiftUI 包 UIKit 包 SwiftUI） | 從外往內，`BWFFullProfileDetailsVC` 最後重寫（Phase 4） |
| 14 | `BWFAccountController`（ObjC）、`BWFWhoViewedMe`（ObjC entity） | **先把 Entity 換 Swift value type**，否則 `@Observable`/`Equatable`/diffable 卡住 |

> 原 §6.15「分階段策略」已落實為 §4.2 的 Phase 0–6，不另列。
