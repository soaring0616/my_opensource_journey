# 實驗紀錄：[Pyrefly][Issue 1482] 讓 `pyrefly.toml` 支援 `only`（等同 CLI `--only`）

Nov 10, 2026 · @anna


> pyrefly: issue [#1482](https://github.com/facebook/pyrefly/issues/1482)（為 CLI `--only` 加上設定檔對應選項）
> PR: https://github.com/meta/pyrefly/pull/5553（原為 facebook/pyrefly，GitHub 會自動轉址）
> 狀態：2026-11-05 完成 rebase 與驗證並開 PR（#5553）→ 11/10 rchen152、stroxler approve，rchen152 以 meta-codesync 匯入內部 → 已 merge（依你所述；merge 日期與 commit hash 待補，我抓到的 PR 頁面仍顯示 Open）
> 2026.11 版本；rebase 基底 upstream/main `cfac5854e`


## 1. Issue #1482：需求與約定

CLI 有 `--only`，只顯示指定的錯誤種類；希望設定檔也能設定同樣的事，不用每次下指令。

- 已確認（10/21 讀碼）：最接近的範本是 `min-severity`。CLI 欄位在 `OutputArgs`，設定欄位在 `ConfigFile`，在 `OutputArgs::resolve` 合併，CLI 優先。`--only` 原本只有 CLI 這一條路。
- 已確認：`ErrorKind` 已經 derive `Deserialize`（kebab-case），所以設定檔不需要自訂解析。

跟 maintainer 在 issue 裡約定的 6 點：

| # | 約定 | 結果 |
| --- | --- | --- |
| 1 | 和 CLI 一樣在輸出階段過濾 | 完成 |
| 2 | kebab-case 陣列鍵：`only = ["bad-assignment"]` | 完成 |
| 3 | CLI 的值整個取代設定檔的值 | 完成（單元測試涵蓋 4 種組合） |
| 4 | 專案層級，sub-config 不能覆寫 | 完成（sub-config 的 `only` 被忽略並警告） |
| 5 | 測試確認 `--suppress-errors` 和 `--update-baseline` 都跟著過濾 | 完成（兩個 scrut 測試） |
| 6 | 更新 `schemas/pyrefly.json` 與 `configuration.mdx` | 完成 |

額外加的（不在 6 點內）：`only = []` 視為設定錯誤。見 §3。


## 2. 最終設計與行為

branch 上的 6 個 commit（squash 前，rebase 後的 hash）：

| hash | 內容 |
| --- | --- |
| `5ce53c396` | 從 TOML 解析 `only` |
| `4b67f967c` | 把 `only` 接到診斷流程，CLI 優先 |
| `6aea5731f` | 測試 `--suppress-errors` 與 `--update-baseline` 都跟著過濾 |
| `59b92b948` | 載入設定時拒絕空陣列 |
| `ff69555ef` | 文件與 JSON schema |
| `4fbcefc92` | 合併巢狀 `if`（clippy::collapsible_if） |

行為（都有實驗或測試支持）：

- 鍵 `only`，型別 `Option<Vec<ErrorKind>>`。
- 合併：`self.only.clone().or_else(|| config.and_then(|config| config.only.clone()))`，CLI 有值就用 CLI，否則用設定檔。
- 專案層級：寫在 `[[sub-config]]` 會被忽略，並出現 WARN「Extra keys found in sub config matching ...: only」。
- CLI `--only` 沒給值或給 `""`：clap 報錯，exit 2。
- `only = []`：設定錯誤，輸出 `ERROR <path>: ...` 與 `Fatal configuration error`，exit 1，不檢查任何檔案（只驗證過 `pyrefly check`，LSP 沒驗證）。


## 3. `only = []` 的決定

**實驗（11/4）**：檔案 `x: str = 1`，設定 `only = []`（加驗證之前）。

| 設定 | 輸出 | exit code |
| --- | --- | --- |
| `only = []` | `INFO 0 errors` | 0 |
| 空設定檔（對照組） | 1 個 `bad-assignment` 錯誤 | 1 |

對照組在同一個檔案報了錯，所以 `0 errors` 確實是 `only = []` 造成的。

**讀碼**：`only_filter` 是 `Option<SmallSet<&ErrorKind>>`，用 `only.contains(&e.error_kind())` 篩選。空集合的 `contains` 永遠是 `false`，每個錯誤都被丟掉。

**決定**：在 `ConfigFile::from_file` 載入時拒絕。理由：空陣列會讓檢查永遠通過，而且沒有警告；CLI 端 `--only` 沒給值本來就被 clap 拒絕。

- 這是超出約定的決定，PR 的 Summary 和 Open question 4 都明講，讓 maintainer 可以要求改成「視為未設定」或允許。
- maintainer 的回覆：PR 頁面上沒有針對 `only = []` 的專門回覆；rchen152 在 11/10 approve，被 approve 的版本已包含 `59b92b9` 的拒絕行為，沒有要求修改。


## 4. 測試

rebase 到 upstream/main（`cfac5854e`）**之後**重跑的結果：

| 項目 | 結果 |
| --- | --- |
| 單元測試 | `test_only_config_parsing`、`test_only_empty_list_is_rejected`、`only_cli_and_config_precedence` 都 `ok` |
| 完整 `cargo test -- --skip deep_phi_chain_term_expr` | exit 0；主 crate 10420 passed、0 failed；用時 548 秒 |
| scrut（全部） | 231 案例，225 通過，6 失敗（見下） |
| scrut（我改的 4 個檔案） | 63 案例，60 通過，3 失敗（都在下表） |
| jsonschema | `Ran 62 tests ... OK` |
| conformance | Python 3.12，exit 0，`git status --short` 為空 |
| fmt | stable `cargo fmt -- --check` 無差異；本機 nightly rustfmt 壞掉（`librustc_driver` 找不到） |
| clippy | exit 0；1 個警告在 `coverage.rs`（`large_enum_variant`，不在 diff 內） |

scrut 的 6 個失敗，都與 `only` 無關：

| 位置 | 原因 |
| --- | --- |
| `errors.md:185` | Python 3.9 的 `typing` 沒有 `reveal_type` |
| `config.md:93` | Python 3.9 的 `typing` 沒有 `assert_type` |
| `errors.md:316` | 手動執行時 `$TEST_PY` 沒設定，`dirname` 缺參數 |
| `basic.md:24` | `$TEST_PY` 沒設定，`cp` 缺參數 |
| `basic.md:37` | `$PYREFLY_PY` 沒設定 |
| `find.md:30` | `$TYPESHED_ROOT` 沒設定，`touch` 變成 `/pyrefly.toml` |

新增的 scrut 區塊（4 個，皆通過）：

| 檔案 | 標題 | 目錄 |
| --- | --- | --- |
| `test/errors.md` | `only` in config file filters reported errors | `only_config` |
| `test/suppress.md` | `--suppress-errors` respects the `only` filter from the configuration | `suppress_only_config` |
| `test/baseline_reporting.md` | `--update-baseline` respects the `only` filter from the configuration | `baseline_only_config` |
| `test/config.md` | `only` in config file should not be empty | `only_cannot_be_empty` |

schema 負向案例：`invalid_type_for_only`、`only_empty_array`（每個案例會產生 2 個測試，分別對 `pyrefly.toml` 與 `pyproject.toml`）。

未驗證：

- LSP / IDE：沒實測。用 `git grep -F '.only'` 搜尋，沒找到讀設定檔 `only` 的 LSP 程式碼（推論，只搜欄位存取，解構寫法不會被抓到）。
- 完整 `test.py`：`--test` 步驟在 cargo test 中止，改成逐步執行。
- `--extension`、`--tensor-shapes` 沒跑。

反向驗證（PR 描述所寫，細節以你的終端紀錄為準）：暫時拿掉實作，單元測試應失敗；空設定檔讓 scrut 看到未過濾的錯誤；暫時拿掉 `minItems: 1`，schema 負向測試失敗。


## 5. 程式碼位置

行號以 rebase 後的 branch（基底 `cfac5854e`）為準，只列驗證過的；其他以名稱為準。

| 位置 | 內容 |
| --- | --- |
| `pyrefly/lib/commands/check.rs:645-647` | `OutputArgs` 的 `resolve`：CLI 與設定檔合併 |
| `pyrefly/lib/commands/check.rs:2105-2107` | `only_filter` 改讀 `defaults.only` |
| `pyrefly/lib/commands/check.rs:3452-3469` | 單元測試 `only_cli_and_config_precedence` |
| `OutputDefaults`（`check.rs`） | 新增 `only` 欄位 |
| `crates/pyrefly_config/src/config.rs` | 新增 `only` 欄位、`ConfigFile::from_file` 裡的空陣列驗證、`use crate::error_kind::ErrorKind;` |
| `schemas/pyrefly.json` | 根層級 `only`，`minItems: 1`（沒放進 `$defs.configBase`，因為是專案層級） |
| `schemas/validate_schemas.py` | 負向案例；案例名稱不能以 `test_` 開頭，否則測試名稱會變成 `test_test_...` |
| `website/docs/configuration.mdx` | `### only` 條目 |


## 6. 時間線

| 日期 | 事件 |
| --- | --- |
| 10/21 | 讀碼、提出設計與 5 個待問問題 |
| 11/4 | `only = []` 實驗；實作與測試 |
| 11/5 | rebase（落後 upstream 125 個 commit）、重跑所有驗證、推 branch |
| 11/5 | 開 PR #5553（目前頁面顯示非 Draft；開啟當下是否為 Draft 待補）；bot 回報符合 contributing checklist、mypy_primer 沒有型別檢查結果變動；標籤 `cla signed`、`size/m`、`waiting-for-merge`；assignee rchen152 |
| 11/10 | approve |
| 11/10 | (應該很快就merge了） |


## 7. rebase 事件

upstream 125 個 commit，其中 `check.rs`（468 行變動）、`config.rs`、`configuration.mdx` 動過；schemas 與 scrut 測試檔沒動。

- 第 2 個 commit 在 `check.rs` 衝突，只有 1 處（約 7 行）：upstream 重構成 `shown_kind`／`CollectedDiagnostic`，已經刪掉 `let errors = collected;`。解法：採用 upstream 的結構，只套上我的改動，也就是 `only_filter` 的來源從 `self.output` 換成 `defaults`。
- 手動編輯衝突時，把共用的 `.as_ref()` 和 `.map(...)` 兩行也刪了，編譯前補回。
- rebase 結束時，**接線那一行是舊的**（HEAD 裡 `only_filter` 還在讀 `self.output.only`），差異只出現在暫存區（`git diff --cached`）。原因沒有查明。
- 修法：先建備份 branch，`git commit --fixup=<第 2 個 commit>`，再 `GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash upstream/main`。
- 驗證：`git log --oneline upstream/main..HEAD` 是 6 個 commit、`git status --short` 為空、`git show HEAD:pyrefly/lib/commands/check.rs | grep -n -A2 'let only_filter'` 看到 `defaults`。
- 教訓：解完衝突之後，先確認 HEAD 的內容，再跑測試；測試通過不等於接線還在。


## 8. 踩過的坑

- scrut：指令續行要用行首的 `> `（行內換行會切斷指令）；用到 `*` 的行要加 `(glob)`；`$PYREFLY` 在新的 shell 是空的，會變成 `command not found: check`、exit 127。
- 手動跑 scrut 要設 `$PYREFLY`、`$JQ`；有些案例還要 `$TEST_PY`、`$PYREFLY_PY`、`$TYPESHED_ROOT`。症狀是 `-c: command not found`、`usage: dirname`、`touch: /pyrefly.toml`，一看就知道是變數沒設。
- Python 3.9：`typing` 沒有 `reveal_type`/`assert_type`；conformance 腳本要 3.10 以上（`Path | None`），而且 `tempfile.NamedTemporaryFile(delete_on_close=...)` 要 3.12。
- macOS 的 `git grep -E` 不支援 `\b`，結果是空的，不代表沒有；改用 `-F`。
- nightly rustfmt 的 dyld 錯誤之後，`echo $?` 仍是 0；不能當作格式通過。
- `GIT_EDITOR=true git rebase --continue` 可以不開編輯器沿用訊息；`git commit -F <檔案>` 取代 vim。
- rebase 時，`<<<<<<< HEAD` 是 upstream 的版本，`>>>>>>>` 才是自己的 commit，和 merge 相反。
- 兩個視窗同時對同一個 repo 跑 `test.py`（會格式化、會跑 cargo）很危險；`--fmt` 預設開啟。
- 測試裡 `{content:?}` 會把字串加上引號，造成測試因為錯的原因失敗。
- `git restore` 是從暫存區還原；如果暫存區的版本已經壞了，還原出來還是壞的。
- Docusaurus 的連結 `../error-kinds`（不加副檔名）在 GitHub 預覽顯示 404，是預期的。
- 完整 `cargo test` 會在 `deep_phi_chain_term_expr` 堆疊溢位中止（`SIGABRT`），要用 `--skip`。
- macOS 要設 `SDKROOT`；`ld: __eh_frame section too large` 和 `cargo fmt` 的 nightly 選項警告都無害。


## 9. 待辦與未決

PR 描述裡提出的 4 個問題：

| # | 問題 | 回覆 |
| --- | --- | --- |
| 1 | 設定檔的 `only` 要不要也讓 `has_daemon_incompatible_options` 回傳 `true`？ | 頁面上沒有回覆；approve 時未要求修改，方法維持不變 |
| 2 | JSON schema 缺 `min-severity`、`output-format` | 頁面上沒有回覆；沒有要求補進這個 PR |
| 3 | schema 的 `only` 用純字串，不用 enum，可以嗎？ | 頁面上沒有回覆；沒有要求改成 enum |
| 4 | `only = []` 拒絕、視為未設定、或允許？ | 頁面上沒有針對性回覆；approve 的版本即為拒絕空陣列 |

- [ ] 待補：merge 的 commit hash 與日期（我抓到頁面時仍是 Open；是否已落地請以 GitHub 頁面為準）
- [ ] CI：頁面只看得到 bot 的 checklist 與 mypy_primer 結果，看不到各項 CI 的明細；「單元、整合、schema、conformance 通過」是你本機的結果，不是 CI
- [ ] 清理本機 branch：`backup-issue-1482-before-rebase`、`backup-after-rebase`、`issue-1482-config-only`
- [ ] follow-up PR：沒有人要求；問題 1、2 目前沒有答覆，要不要另開 issue 由你決定


---

## Q1: 為什麼 `CheckArgs` 也要改

`only_filter` 是最後真正過濾輸出的東西。

- 舊寫法：`self.output.only`，只讀 CLI 的 `--only`。
- 如果只改 `OutputArgs` 的合併方法、沒改這一行，合併結果會存進 `defaults.only`，但過濾的程式碼仍然只看 CLI，設定檔的 `only` 就算被讀進來也沒有效果。
- 所以這一行是「接線」：讓過濾器吃合併後的值。
- 同一類的風險：搜尋有沒有別處還在讀 `output.only`（要用 `-F '.only'`，並注意舊寫法把 `self`、`.output`、`.only` 拆成三行）。

## Q2: `has_daemon_incompatible_options` 是什麼

已確認：

- 有兩個：`OutputArgs` 上的（私有），和 `CheckArgs` 上的 `pub` 版本，後者呼叫前者再加上 `self.behavior.has_custom_options()`。
- `pub` 版本在這個 repo 裡沒有呼叫者；daemon 的實作也不在這個 repo。
- 引入它的 commit：`04b2767c0 Allow supported daemon output options`（2026-09-01）；說明是「daemon 檢查可以用客戶端選的 `full-text`/`min-text`/`json` 輸出格式與最低嚴重度，其他需要 daemon 不支援行為的選項繼續拒絕」。
- 註解：「Whether the invocation uses an option unsupported by daemon checks」。
- `min_severity: _` 是明確忽略（該 commit 允許）；`only.is_some()` 在判斷式裡，所以 CLI `--only` 算不相容。

推論（未驗證）：呼叫者在 Meta 內部程式裡；這個函式只看 CLI 參數是刻意的還是沒考慮到設定檔，只有 maintainer 知道。這是 Open question 1。

## Q3: 怎麼判斷 scrut 失敗是環境問題

1. 看 stderr 的第一行。`-c: command not found`、`usage: dirname`、`touch: /pyrefly.toml: Read-only file system` 都是環境變數展開成空字串。
2. `Could not import X from typing`：Python 版本太舊。
3. 再確認失敗的區塊跟自己的改動有沒有關係（看標題與內容有沒有 `only`）。
4. 全部歸零之前，先設 `$PYREFLY`、`$JQ` 再重跑，失敗數會從 32 降到 6。