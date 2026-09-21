---
name: spec-to-tests
description: 從規格文件產生可追溯、可供後續自動化轉換使用的測試案例規格。讀入 Bible／PRD／SRS／前後端系統規格書，先把前端 SRS 的每個 UI control 盤點成 inventory 並逐格標記十二個規則面向，再逐欄位、API、DB、權限、錯誤碼與狀態轉換正規化需求，展開驗收條件與八種測試維度，輸出具 fixture、actor、entrypoint、inputs、observations、oracles、cleanup 與完整追溯的 testcases.json。本 skill 不產生或執行自動化腳本；結構化欄位是提供下游工具使用的測試設計資訊。文件已有驗收資產時直接萃取；衝突與規格／資料／環境／依賴／自動化缺口分型記錄而不猜測。支援 baseline 重跑。Use when generating test cases or acceptance criteria from specs, building traceability or field-rule matrices, preparing structured metadata for downstream test automation, reviewing spec coverage and conflicts, validating the test design contract, or re-running revised specs.
---

# 規格轉測試案例

把任意規格文件轉成 `testcases.json`：需求 → 驗收條件 → 一對多測試案例 → 下游自動化所需的結構化對映，外加可追溯且可解除的阻擋清單。

本 skill 的終點是測試案例與執行對映資料，不負責產生測試程式碼、控制瀏覽器、呼叫 API、執行 SQL 或判定實際測試結果；這些工作由 `test-to-automation` 或其他執行工具負責。

## 五條硬性規則

這五條凌駕一切效率考量，違反其中任何一條就是產出失敗。

0. **範圍之外的不進來。** 先用 `project` 代號界定本次要測的模組，只有明確涉及該模組的段落才成為需求。Bible 的系統邊界敘述、名詞定義、命名規範、Business Rules 表這類**講方向與講語彙**的內容，是需求的佐證來源，不是需求本身。落不到本模組的具體畫面、API 或資料表，就不准變成需求。
1. **衝突不得自行裁決。** 文件之間任何矛盾都要寫進 `issues`。可以依仲裁規則挑一方讓流程往下走，但絕不可默默採用後照常產測試案例。
2. **現成的驗收資產一律萃取，不重新發明。** 文件裡已經寫好的驗收條件、測試案例、待確認項，照原意搬進來並在 `from` 記下原始編號，只能補欄位、不得改寫語意。五條規則裡最容易被整批跳過的就是這條，所以步驟 4 要求先輸出一張盤點表把每個編號逐條交代完。
3. **禁用樣板句。** 步驟與預期結果要具體到陌生人能照著執行並自己判斷通過與否。`scripts/check.py` 會逐條比對禁用清單，命中即錯。
4. **下游工具不得猜測。** 每條測試都要明列 actor、fixture、entrypoint、inputs、observations、oracles 與 cleanup；Save／Finish、checkpoint、頁籤、橘點與 rollback 另列前後狀態。這些是測試設計 metadata，不是可執行腳本。規格沒有寫就記 issue，不得虛構。

## 流程

### 1. 確認輸入與輸出

問清楚要分析哪些文件；使用者沒指定就掃描工作目錄找規格文件（`.md`／`.docx`／`.pdf`），列出來請他確認範圍。

輸出寫到這些文件的共同根目錄，檔名 `testcases.json`。若該檔已存在，先問清楚是要重跑比對（把舊檔當 baseline）還是覆蓋重來——**不要直接覆蓋**。

要重跑就讀 `references/rerun.md`；首次產生則完全不套用重跑規則，也不要寫 `change` 欄位。

### 2. 盤點文件

逐份判定型別（`BIBLE`／`PRD`／`SRS`／`OTHER`）並登錄進 `docs`。

依 `references/method.md` 的來源章節盤點規則填 `docs[].sections`，逐節標記 `covered`、`out-of-scope`、`duplicate` 或 `unverifiable`。採用內容要指向產出 ID；不採用內容要有理由。

`path` 照磁碟上的實際檔名寫，**一律用正斜線 `/`**（Windows 上也是），且**不要從功能代號或別份文件的引用去拼**——目錄名與模組代號長得像但未必相同。寫完先確認檔案真的存在：

```bash
ls -1 <每一個 path>
```

`docs` 的 ID 照固定順序配發：**先 `BIBLE`、再 `PRD`、再 `SRS`、最後 `OTHER`，同型別內按 path 字串排序**。順序隨機的話，下次重跑會整批對不上 baseline。

再用 grep 抓標題結構建立地圖，不要整份讀進來：

```bash
grep -n "^#\{1,4\} " <file>
```

### 3. 界定範圍

在讀任何內容之前先劃線，否則後面每一步都會超收。

從 PRD／SRS 的功能代號決定 `project`（例如 `EPROZ00200`），這就是本次的範圍。接著逐份文件標記哪些章節在範圍內：

- **PRD／SRS**：整份都在範圍內（它們本來就是為這個模組寫的）。
- **Bible**：只有明確提到本模組代號、本模組畫面或本模組 API 的段落在範圍內。講整個專案願景、演進順序、角色總覽、名詞字典、命名規範的段落**一律在範圍外**。

範圍外的段落只有兩個用途：當作範圍內需求的佐證來源（寫進該需求的 `src`），或當作發現跨文件衝突的依據（寫進 issue）。**不得自成需求。**

判準只有一句：這句話失效時，本模組會出現什麼看得到的異常？答不出來就是範圍外。答不出來卻又寫得像規則的，記一則 `UNVERIFIABLE` issue。

`check.py` 會擋：只源自 BIBLE 的需求超過兩成就報 `SCOPE_LEAK` 錯誤。

### 4. 萃取現成的驗收資產

讀 `references/method.md` 的「萃取文件裡現成的驗收資產」，把文件裡已有的驗收條件、測試案例、待確認項先撿出來。這一步要在自己動手寫任何東西之前做完，否則會重複發明。

先用 grep 把編號抓齊，不要靠印象撿：

```bash
grep -rnoE '\b[A-Z][A-Z0-9-]*-[0-9]{3}\b' <每一份文件> | sort -u
```

**抓完先輸出一張萃取盤點表再往下做**（格式看 `references/method.md`），逐條交代每個編號落到哪，或為什麼不採用：

| 原編號 | 出處章節 | 落到哪 | 不採用的理由 |
| --- | --- | --- | --- |
| TC-004 | PRD `8. 測試案例草案` | STT-TC-031（`from: TC-004`） | — |
| AC-BIBLE-005 | Bible `共同驗收範例` | 不採用 | CA 分案，不在本模組範圍 |

編號要全列，不能只列採用的。PRD／SRS 的資產成批不採用，代表範圍界定做錯了——`check.py` 會用 `EXTRACT_THIN` 擋。

**盤點表只出現在回覆裡，不要倒進 `issues`。**「AC-BIBLE-003 不採用，因為不在本模組範圍」是盤點紀錄，不是待確認事項；把這種條目寫成 issue 會讓 PM／SA 拿到一份八成是雜訊的清單。`issues` 只留真的需要有人去定案、而且指得出影響哪條需求或驗收條件的項目（`refs` 要有值）——`check.py` 會用 `ISSUE_ORPHAN` 擋，且 `refs` 全空的 issue 不計入萃取覆蓋率。

### 5. 需求正規化

一條需求 = 一個可驗證的斷言。顆粒度與合併判準看 `references/method.md`，兩件事最容易做錯：

- **顆粒度**：一條需求對應「一個欄位的一條驗證規則／一個 API 的一種行為／一張表的一次寫入」。不要一條塞多個規則，也不要把同一個欄位的同一條規則拆成好幾條。
- **合併**：同一個欄位、同一支 API、同一張表的同一件事出現在多份文件，就是**同一條需求**，合併成一條、`src` 收多筆。逐份文件各掃一遍再各記一條，是最常見的失敗模式——`check.py` 會用 `MERGE_NONE` 擋下來。

**先輸出一張「斷言對象 → 出現在哪些文件哪一節 → 合併成哪條」的對照表再寫需求**（格式看 `references/method.md`）。一列 = 一條需求，這張表就是顆粒度的錨；先寫需求再回頭補 `src`，順序反了就不是在合併。驗收條件／需求的比值應落在 **1.2～1.8**。

比值只能驗展開，驗不到顆粒度——**一整節壓成一條需求，所有比值都還是漂亮的**。所以再對一次絕對量：

- **每一份 PRD／SRS 都要有需求引用得到。** 有文件零引用，就是只掃了其中一份就收工（`DOC_UNCITED` 錯誤）。
- **每份文件被引用到的章節數要對得上它的規則密度。** 一份寫滿欄位規格表與 DB 影響矩陣的 SRS 只被引用三四節，代表節裡的每條欄位規則、每次落表寫入都被吞掉了（`REQ_COARSE` 警告）。

衝突依 `references/method.md` 的仲裁規則處理並記 issue。

寫完 `requirements` 就先落檔，不要等全部做完才寫。

**有前端 SRS 時，先做 UI control inventory，再寫需求。** 依 `references/method.md` 的「UI control inventory」把欄位表裡每一個實體 control 各建一筆 `uiControls`（地址四層是四筆、幣別與金額是兩筆、按鈕也算），十二個規則面向每格標 `specified`、`not-applicable`（附理由）或 `issue`（指向 ISS），寫進 `testcases.json` 後跑一次 `check.py`。`check.py` 會自己從前端 SRS 抽出 control 清單來比對，少一個報 `UI_CONTROL_MISSING`；**inventory 沒過檢查之前不要開始寫 `ui-field` 需求**。

之後每個 `specified` 格對應一條 requirement，`target` 寫 `{ ui-field, <field>, <aspect> }` 且與格子一致；一格一條，不要把六個地址欄位的 cascade 合成一條（`UI_TARGET_COMPOSITE`）。這一步做完，前端 SRS 的三四十個欄位才會逐一進到需求裡，而不是整節壓成幾條。

### 6. 驗收條件

每條需求至少一條驗收條件，`then` 必須是看得到的結果。

**`then` 不得寫成提問。**「需先確認應寫入 00 還是 01」不是結果，是問題。碰到跨文件衝突的正確順序是：依仲裁規則挑一方 → `then` 寫出**那一方**的具體可判定結果 → 記 `CONFLICT` issue → 掛 `blocked`。`blocked` 標記的是「這個答案還沒定案」，不是「我寫不出結果」。

`blocked` 的驗收條件**不展開測試案例**——讓缺口被看見，比產出一條基於猜測的假案例好。但缺口要有節制：**有衝突不等於該擋住**，仲裁規則的存在就是為了讓流程往下走。`blocked` 佔比超過 15% 報 `BLOCK_HEAVY` 警告、超過 25% 報錯。

### 7. 一對多展開測試案例

對每條驗收條件套 `references/method.md` 的展開矩陣，逐一檢查 `happy`／`boundary`／`negative`／`auth`／`state`／`txn`／`concurrent`／`integration` 八個維度，適用的每個維度至少一條。

**測試案例數量應該明顯多於驗收條件數量。** 收尾時對這三個數，任何一個沒過就是沒有真的展開，回頭重做：

- 測試案例／驗收條件 **不低於 1.5**（`TC_PER_AC`）
- 可測驗收條件裡「只有一條測試案例」的比例 **不超過 50%**，理想在三分之一以下（`EXPANSION_FLAT`）
- 規格提到序號／併發／rollback／批次，就不該有 `concurrent`／`txn`／`integration` 掛零（`DIM_MISSING`）

第三點常常和步驟 4 連動：**文件裡現成的測試案例往往正好是這幾個維度**（併發取號、複製交易 rollback），萃取漏掉，展開通常也跟著漏掉。

同時要有節制：沒有值域限制就不要硬湊 `boundary`，沒有併發語意就不要硬湊 `concurrent`。每條都要能說出它在驗哪個具體風險。

**每批展開完，先在回覆裡輸出一張判定表再寫檔**，逐條逼自己說明八個維度各自的取捨：

| AC | happy | boundary | negative | auth | state | txn | concurrent | integration |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| STT-AC-012 | TC-031 | TC-032/033（金額上下界） | TC-034/035（兩個錯誤碼各一） | TC-036（CR 直呼 API） | 不適用：只有一種前置 | 不適用：無寫入 | 不適用：無併發語意 | 不適用：不跨系統 |

不適用要寫出原因，不能留白。`negative` 那格特別要對照規格的**處理結果代碼表**逐碼盤點——每個獨立錯誤碼各一條，這是最常整批漏掉的一格。

寫 `expected` 時再過一次：它必須包含 `then` 沒寫的東西（具體代碼、訊息、欄位值、觀察位置）。如果 `expected` 裡有一條讀起來就是 `then` 換句話說（前面加「此維度的可判定結果為：」這種引導語也一樣算），那條就沒寫完。

同樣要過一次骨架：**把「QA 依「X」設定本案例輸入值」這種固定頭尾抽掉，剩下的內容還能不能讓陌生人執行？** 每句字面不同不代表不是模板，`check.py` 會用 `TEMPLATE_SHAPE` 比頭尾指紋。`steps` 每一步要指名真正的角色（`AO（002）`、`CR（003）`），寫「以規格指定角色執行」等於沒寫。

每條 test 依 `references/schema.md` 補齊 `automation` 對映資料。先定 mode、actor、fixture 與 entrypoint，再填結構化 inputs、observations、oracles 與 cleanup。不要生成測試程式碼或實際執行案例。狀態或交易案例補 `stateTransition`。逐碼輸出錯誤碼盤點表；每個錯誤碼各一條 negative，並描述 response、message、data、DB、audit 與 log mask 中規格適用的觀察點。

### 8. 檢查與回報

```bash
python3 <skill-dir>/scripts/check.py <path>/testcases.json
```

錯誤全部清乾淨才算完成，警告要逐條看過再決定接受或修正。這幾個錯誤碼各自對應一個前面的步驟，出現就回到那一步重做，不要在原地修修補補：

| 錯誤碼 | 意思 | 回到哪一步 |
| --- | --- | --- |
| `SCOPE_LEAK`／`BIBLE_ONLY_REQ` | 把 Bible 的方向、名詞、Business Rules 當成需求了 | 3 界定範圍 |
| `EXTRACT_THIN`／`EXTRACT_SERIES_MISS` | 文件裡現成的驗收條件、測試案例、待確認項沒撿，自己重新發明了一套 | 4 萃取，補完盤點表 |
| `MERGE_NONE`／`MERGE_THIN` | 逐份文件各掃一遍，沒做跨文件合併 | 5 需求正規化 |
| `DOC_UNCITED`／`REQ_COARSE` | 有 PRD／SRS 沒進到需求裡，或一整節壓成一條需求 | 5 需求正規化 |
| `ISSUE_ORPHAN` | 把萃取盤點表的「不採用理由」倒進 `issues` | 4 萃取，理由寫回盤點表 |
| `AC_PER_REQ`／`REQ_UNCOVERED` | 需求切太細或驗收條件沒補齊 | 5、6 |
| `AC_THEN_QUESTION` | `then` 寫成提問，沒依仲裁規則收斂出可判定的結果 | 6 驗收條件 |
| `BLOCK_HEAVY` | 把「有衝突」直接當成「擋住」，缺口過量 | 6，重看仲裁規則 |
| `ECHO_AC` | 測試案例只是把驗收條件複製一次 | 7 展開矩陣 |
| `TC_PER_AC`／`EXPANSION_FLAT`／`DIM_MISSING` | 貼著一對一，八個維度沒真的逐格過 | 7 展開矩陣 |
| `TEMPLATE_REPEAT`／`TAIL_TEMPLATE` | 同一句話反覆出現，是填空模板 | 7、可測性紅線 |
| `TEMPLATE_SHAPE`／`SETUP_ECHO` | 頭尾固定、中間換填空的骨架句，或把同一句前置條件加個殼再寫一次 | 7、可測性紅線 |
| `TRACE_MISMATCH` | `title`／`risk.why` 寫的 AC 編號與 `ac` 欄位對不上，複製貼上只改了一半 | 7 展開矩陣 |
| `RERUN_WIPE` | 文件沒改版卻把 baseline 整批標成 `REMOVED`，身分比對沒做 | `references/rerun.md` |
| `NEGATIVE_THIN`／`NEGATIVE_MISSING` | 失敗路徑整批沒展開 | 7 的 `negative` 那格 |
| `RISK_SKEW` | 幾乎全標 High，風險分級沒有鑑別度 | `references/schema.md` 的判斷依據 |
| `DOC_PATH_MISSING`／`DOC_PATH_SEP`／`DOC_ORDER` | path 拼錯、用了反斜線、ID 配發順序不固定 | 2 盤點文件 |
| `SOURCE_SECTION_MISSING` | 文件章節未逐一交代採用或不採用理由 | 2 來源章節盤點 |
| `UI_FIELD_COVERAGE`／`TARGET_KIND` | requirement 缺少結構化斷言對象或欄位規則面向 | 5 UI control inventory |
| `UI_INVENTORY_MISSING`／`UI_CONTROL_MISSING` | 有前端 SRS 卻沒建 inventory，或 SRS 欄位表的 control 沒進 `uiControls` | 5 UI control inventory |
| `UI_ASPECT_UNACCOUNTED`／`UI_NA_REASON_MISSING`／`UI_ISSUE_REF_MISSING`／`UI_ASPECT_SOURCE_MISSING` | 十二個面向有留白、不適用沒理由、issue 沒指向 ISS、結論沒來源 | 5 UI control inventory |
| `UI_TARGET_COMPOSITE`／`UI_REQUIREMENT_MISSING`／`UI_REQUIREMENT_MISMATCH` | 一筆代表多個欄位、specified 沒有需求、需求 target 與 inventory 對不上 | 5 UI control inventory |
| `UI_CASE_MISSING` | specified 面向的需求有驗收條件卻沒有任何測試案例 | 7 展開矩陣 |
| `ROLE_MATRIX_MISSING`／`ERROR_CODE_MISSING`／`CROSS_MODULE_GAP` | 權限、錯誤碼或跨模組規則未展開對應測試 | 7 展開矩陣 |
| `FIXTURE_MISSING`／`ORACLE_MISSING`／`CLEANUP_MISSING`／`EXECUTION_CONTRACT_MISSING` | 自動化工具仍需猜測測資、入口、觀察點、判定或清理方式 | 7 自動化執行契約 |
| `STATE_TRANSITION_MISSING` | 狀態案例未列初始、觸發、完成與 rollback 狀態 | 7 狀態轉換模型 |
| `ISSUE_CATEGORY`／`ISSUE_EXECUTION_CONTRACT` | issue 未分型或缺解除與重跑條件 | `references/method.md` 的待確認事項 |

要用眼睛看追溯關係，用瀏覽器開 `<skill-dir>/assets/viewer.html` 載入 `testcases.json`：RTM 頁籤把需求 → 驗收條件 → 測試案例展成心智圖，被 issue 擋住的驗收條件標成紅色缺口；UI Coverage 頁籤是 control × 十二面向的熱圖，一眼看出哪個欄位哪個面向沒有測試案例，點格子可反查 REQ → AC → TC；Issues 頁籤列出全部待確認事項，並反查各自擋住哪些驗收條件。純檢視，不會動到檔案。

回報時給使用者：各集合的數量與展開比例、UI control 數與十二面向三種狀態的分布、萃取了幾筆、被擋住的缺口有幾條、待確認清單摘要（尤其是需要誰去確認）。

## 分批策略

文件動輒上千行、需求動輒數十條，一次全做完會失控。照這個順序分批，每批寫檔後跑一次 `check.py`：

1. 一次做完 `docs`、範圍清單、來源章節／現成資產盤點表與 `issues` 初稿
2. 有前端 SRS 時，一次做完 `uiControls` 全集並落檔，跑 `check.py` 到沒有 `UI_*` 錯誤
3. 先輸出斷言對象對照表，再一次做完 `requirements` 全集（`ui-field` 需求逐格對應 inventory）
4. 分批補 `criteria`，每批約 10 條需求
5. 分批展開 `tests` 與 `automation`，每批約 5 條驗收條件；同步維護錯誤碼與狀態轉換盤點表
6. 補上過程中新發現的 issue，最後整體檢查

`check.py` 對半成品會報覆蓋率警告，那是正常的，做完再看最後一次的結果。

## 參考資料

- `references/schema.md` — 輸出 JSON 的欄位定義、ID 規則、`uiControls` 與 aspects 結構、風險矩陣、完整範例。寫第一筆記錄前先讀。
- `references/method.md` — 文件分層仲裁、萃取規則、需求正規化、UI control inventory、驗收條件要求、展開矩陣、可測性紅線、待確認事項分類。整條流程的判斷依據都在這裡。
- `references/rerun.md` — 增量重跑。只在有 baseline 時讀。
- `scripts/check.py` — 驗證輸出。收尾必跑。
- `assets/viewer.html` — 產出後的檢視器。單檔零依賴，瀏覽器開啟後載入 `testcases.json` 即可看 RTM 心智圖、UI Coverage 熱圖、各集合表格與待確認清單。