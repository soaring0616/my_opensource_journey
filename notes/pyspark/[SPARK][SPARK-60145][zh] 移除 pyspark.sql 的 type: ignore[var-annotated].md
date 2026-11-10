# [SPARK][SPARK-60145] 移除 pyspark.sql 的 type: ignore[var-annotated]
    tags: Apache Spark 新手實驗日誌, Apache Spark, PySpark, open source, 實驗日誌, mypy, type annotation
    description: 第一次為 Apache Spark 清 type: ignore:把 PySparkPlotAccessor._backends 的 var-annotated 換成明確型別

> JIRA:[SPARK-60145](https://issues.apache.org/jira/browse/SPARK-60145)
> PR:[apache/spark#59351](https://github.com/apache/spark/pull/59351)
> 2026.10 版本

[TOC]

:::info
**這份日誌怎麼讀**
- ⚠️ = 推測或沒有驗證(含「對話裡找不到原始輸出」的項目)
- 程式碼區塊裡的輸出,是從當時的對話中找到的原始 log(過長的有節錄)
- 原始碼的行號連結,都指向上游 `master` 的 commit [`0aa4f91`](https://github.com/apache/spark/commit/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4)(2026-10-23)。**這個 PR 已經合併,所以該版本的第 106 行已經是修改後的樣子**;改動前的那一行來自我當時本機的 `git grep ... HEAD` 輸出
- 這份日誌的 PR 狀態、合併資訊,來自 PR 頁面的摘要,不是 verbatim
:::

---

## 1. 了解程式

### 1.1 找到目標

- 目標只有一行:`PySparkPlotAccessor` 的類別屬性 `_backends`,原本是空 dict 加一個 ignore。
- 位置(上游 `0aa4f91`):

| 內容 | 檔案:行 |
|---|---|
| 改動前:`_backends = {}  # type: ignore[var-annotated]`(來自我當時的 `git grep` 輸出);改動後:`_backends: dict[str, ModuleType] = {}` | [`python/pyspark/sql/plot/core.py:106`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/plot/core.py#L106) |
| `from types import ModuleType` | [`python/pyspark/sql/plot/core.py:19`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/plot/core.py#L19) |
| `_get_plot_backend(...) -> ModuleType` | [`python/pyspark/sql/plot/core.py:119`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/plot/core.py#L119) |

```python
# python/pyspark/sql/plot/core.py(上游 0aa4f91,已含本 PR)
106:    _backends: dict[str, ModuleType] = {}
...
119:    def _get_plot_backend(backend: Optional[str] = None) -> ModuleType:
120:        backend = backend or "plotly"
121:
122:        if backend in PySparkPlotAccessor._backends:
123:            return PySparkPlotAccessor._backends[backend]
```

為什麼挑這一個:workshop 當天的選題對話裡,我先統計 `pyspark.sql` 與 `pyspark.ml`(不含測試)的 `type: ignore` 錯誤碼分布。我撈到的輸出在尾端被截斷,**`var-annotated` 的數量落在截斷之後,看不到**:

```
$ git grep -ho "type: ignore\[[^]]*\]" -- python/pyspark/sql python/pyspark/ml ':(exclude)*/tests/*' \
  | sort | uniq -c | sort -nr
     75 type: ignore[arg-type]
     70 type: ignore[attr-defined]
     50 type: ignore[assignment]
     21 type: ignore[misc]
     15 type: ignore[call-overload]
     13 type: ignore[override]
     11 type: ignore[union-attr]
      9 type: ignore[return-value]
      6 type: ignore[list-item]
      5 type: ignore[operator]
      5 type: ignore[no-untyped-def]
      5 type: ignore[empty-body]
      3 type: igno...(之後被截斷)
```

⚠️ 最後是怎麼從這份清單落到 `var-annotated` 這個錯誤碼的,選題當下的判斷原文 Sonnet 沒撈到。能確定的是:`{}` 推論不出型別,是 ignore 裡根因最單一的一類(這句是推論,不是量到的)。
⚠️ 我記得沒錯的話，應該是這個類別是最少的，然後有讓 AI 檢查，看起來影響範圍很小 🤔

### 1.2 `var-annotated` 是什麼錯

- mypy 看到空的 `{}` 時,沒辦法推論 key、value 的型別,所以報 `var-annotated`。解法是明確標註,不是壓掉警告。
- ⚠️ 這個說明是我當時從 Claude 的解釋得到的,這份日誌沒有另外去查 mypy 文件。
- `python/mypy.ini` 開了 `warn_unused_ignores = True`([L23](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/mypy.ini#L23)),所以 ignore 一旦多餘,mypy 自己會報(這是從設定推的,我沒另外實驗)。

### 1.3 `_backends` 在哪裡被用到

我先搜整個 `python/pyspark`,看「有沒有人寫入」,因為值的型別應該由寫入點決定,不是靠猜 :

```
$ git grep -n "_backends" python/pyspark
python/pyspark/errors/error-conditions.json:1206:      "`<backend>` is not supported, it should be one of the values from <supported_backends>"
python/pyspark/pandas/plot/core.py:487:    _backends = {}  # type: ignore[var-annotated]
python/pyspark/pandas/plot/core.py:498:            return PandasOnSparkPlotAccessor._backends[backend]
python/pyspark/pandas/plot/core.py:509:                    PandasOnSparkPlotAccessor._backends[backend] = module
python/pyspark/pandas/plot/core.py:522:        if backend in PandasOnSparkPlotAccessor._backends:
python/pyspark/pandas/plot/core.py:523:            return PandasOnSparkPlotAccessor._backends[backend]
python/pyspark/pandas/plot/core.py:539:            PandasOnSparkPlotAccessor._backends["matplotlib"] = module
python/pyspark/pandas/plot/core.py:551:            PandasOnSparkPlotAccessor._backends["plotly"] = module
python/pyspark/pandas/plot/core.py:554:            PandasOnSparkPlotAccessor._backends[backend] = module
python/pyspark/pandas/tests/plot/test_series_plot.py:41:    def test_plot_backends(self):
python/pyspark/pandas/tests/plot/test_series_plot.py:50:    def test_plot_backends_incorrect(self):
python/pyspark/sql/plot/core.py:106:    _backends = {}  # type: ignore[var-annotated]
python/pyspark/sql/plot/core.py:122:        if backend in PySparkPlotAccessor._backends:
python/pyspark/sql/plot/core.py:123:            return PySparkPlotAccessor._backends[backend]
python/pyspark/sql/plot/core.py:130:                messageParameters={"backend": backend, "supported_backends": ", ".join(["plotly"])},
python/pyspark/sql/tests/plot/test_frame_plot.py:49:            messageParameters={"backend": "matplotlib", "supported_backends": "plotly"},
```

讀法 :

- **sql 版**:只有宣告(106)、讀取(122、123)和錯誤訊息的參數(130)。**整個函式沒有任何一行寫入 `_backends`**。
- **pandas 版**:有寫入點(509、539、551、554),也用同一個 `# type: ignore[var-annotated]`(487)。

pandas 版寫入的值,我看過 `sed -n '485,556p'` 的輸出,寫入的 `module` 是 `importlib` 載入的模組,或 `from pyspark.pandas.plot import plotly as module`(見上游 [`pandas/plot/core.py:509`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/pandas/plot/core.py#L509)、[`:539`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/pandas/plot/core.py#L539)、[`:551`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/pandas/plot/core.py#L551))。

### 1.4 value 型別怎麼決定

sql 版沒有寫入點,所以 value 型別不能「從程式碼推導」,是設計上的選擇。我最後選 `dict[str, ModuleType]`,依據有三個:

1. 第 123 行 `return PySparkPlotAccessor._backends[backend]` 回傳的就是這個 dict 的 value,而函式簽名 [`-> ModuleType`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/plot/core.py#L119) 本來就是契約。
2. pandas 版寫入的值也都是模組。
3. 這個檔案本來就用小寫的 `list[str]`(例如 `Union[str, list[str]]`,上游 [L136](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/plot/core.py#L136)),所以 `dict[...]` 不用新增 `Dict` import,風格也一致。

沒選的做法:

- **`Dict[str, Any]`**:mypy 一定過,但資訊量少,reviewer 可能問為什麼不寫精確。
- 只照抄 pandas 版:pandas 版自己也還有同一個 ignore,沒有可抄的標註。

類別層級的註解在 class body 執行時就會求值,所以要確認 runtime 沒壞。上游這個檔案裡沒有 `from __future__ import annotations`(我用 grep 確認過,沒有任何命中),`ModuleType` 又是頂層 import(L19),所以 `dict[str, ModuleType]` 會在載入檔案時真的被求值,測試 import 這個模組就會踩到(測試結果見 2.4)。

### 1.5 為什麼不一起改 pandas 版

`pandas/plot/core.py:487` 還有同樣的 ignore,我刻意留著:

- 它屬於 `pyspark.pandas`;`python/mypy.ini` 對 `pyspark.pandas.plot.*` 設了 [`disallow_untyped_defs = False`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/mypy.ini#L53-L54),對 `pyspark.pandas.*` 設了 [`strict_optional = False`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/mypy.ini#L117-L118),規則本來就比 `sql` 寬鬆,要不要動是另一個判斷。(設定值是讀檔案確認的)
- 縮小範圍比較容易被 review,PR 標題才能寫成「in `pyspark.sql`」。
- ⚠️ 「那個檔案的函式大多沒有型別標註」這句是 Claude 的說法,我沒有自己統計過。

驗證 pandas 版沒被我動到(本機 grep,改動後):

```
$ git grep -n "var-annotated" python/pyspark/sql
$                                      # 沒有輸出

$ git grep -n "var-annotated" HEAD -- python/pyspark/sql
HEAD:python/pyspark/sql/plot/core.py:106:    _backends = {}  # type: ignore[var-annotated]
```

第一行搜 working tree(我已經改掉),第二行搜已 commit 的狀態(改之前)。 兩個結果加起來才能說「這是 `python/pyspark/sql` 裡最後一個 `var-annotated`」。範圍只限 `pyspark.sql`,PR 與 JIRA 的標題都不能寫成整個 PySpark 沒有了。

### 1.6 順便發現的行為問題(不在這個 PR 範圍)

sql 版的 `_get_plot_backend` 讀了 `_backends`,但整個函式沒有寫入。最後的 `from pyspark.sql.plot import plotly as module; return module` 沒有存進 cache,所以第 122–123 行的 cache 分支在 sql 版看起來永遠不會命中。(這是讀程式碼加 1.3 的 grep 得到的)

這是行為,不是型別。我沒有改它,只打算在 PR 描述提一句,讓 reviewer 決定要不要另開 issue。⚠️ 這一句最後有沒有出現在送出的 PR 描述裡,我沒有逐字確認。

---

## 2. 解決步驟

### 2.1 確認沒人做過

先搜 JIRA(JQL)與 GitHub PR,再看有沒有前例。

**JIRA(進階搜尋)**

| # | JQL | 結果 |
|---|---|---|
| 0 | 直接在 JQL 模式的搜尋框打自由文字(內含 `_backends`) |  紅字 `Error in the JQL Query: Expecting operator but got '_backends'`;下面的 "No issues were found" 是查詢壞掉,不是沒有 ticket。⚠️ 當時輸入的完整字串沒撈到 |
| 1 | `text ~ "PySparkPlotAccessor"` |  0 筆 |
| 2 | `text ~ "plot backends type"` |  只有 SPARK-38637(`OptionError: No such option: 'mode.chained_assignment'`),無關 |
| 3 | `component = PySpark AND text ~ "type hint plot"` |  兩筆:SPARK-50003(pie plot doc examples,文件範例)、SPARK-32082(Project Zen,CLOSED / Done / Fix Version 3.4.0,已結案的總括 epic,不要掛在它下面) |
| 4 | `project = SPARK AND component = PySpark AND text ~ "var-annotated"` |  只有 SPARK-54065(Python 3.14 測試修復)與 SPARK-27927(k8s driver pod 卡住),都無關 |
| 5 | `key = SPARK-58244`(查前例的 Issue Type 與 parent ticket) | ⚠️ 給了指令,對話裡沒看到結果 |

⚠️ 上表 #1–#3 是我從對話轉述還原的,當時可能帶有 `project = SPARK AND` 前綴,原始截圖沒有保留。

**JIRA 結論**:沒有看到重複的 ticket。但 JIRA 的 `text ~` 是斷詞後的模糊搜尋(#2 拉出不相關的 SPARK-38637 就是例子),0 筆只能支持「沒有人用這個類別名稱開過 ticket」,不能排除別人用別的描述寫過同一件事。

**GitHub PR**,不要預設 `is:open`:

```
https://github.com/apache/spark/pulls?q=is%3Apr+PySparkPlotAccessor
```

 結果 Open 0、Closed 5,沒有已開出的同款 PR。GitHub 的 PR 搜尋是比對標題與內文,**不一定比對 diff 內容**,所以這也只是輔助證據。

**前例(本機 + 網頁)**

```
$ cd ~/src/spark
$ git log --oneline --grep="SPARK-58244"
7d8a19023a1 [SPARK-58244][PYTHON] Remove type: ignore[name-defined] for DataFrame.plot and streaming _test() in pyspark.sql
```

-  這個 commit 在我本機的 master 裡,代表 #57407 已經合併(GitHub 上顯示 "closed",是因為 committer 用腳本合併,不一定顯示 merged)。我原本只是推測,這條指令把它證實了。
- #57407 是以錯誤碼(`name-defined`)為單位清理 `pyspark.sql`,和我要做的是同一類工作。我拿它當標題、commit 訊息與 PR 描述的樣板,也用它的範圍框架:「移除 `pyspark.sql` 裡最後一個 `type: ignore[var-annotated]`」。
- ⚠️ 我當時被要求去看三件事:#57407 的 diff(`git show --stat 7d8a19023a1`)有沒有動到 `sql/plot/core.py`、SPARK-58244 的 JIRA 類型與 parent ticket、#57407 頁面 13 則留言的內容。這三項的結果,對話裡都沒看到。

### 2.2 建 JIRA

- SPARK-60145:Improvement / Minor / Component PySpark。
- Summary:`Remove type: ignore[var-annotated] in pyspark.sql`(比對 SPARK-58244 的命名風格,與 PR 標題一致)。初稿曾是 `Add type annotation for PySparkPlotAccessor._backends`。
- Description 兩三句就夠:`_backends` 目前用 ignore 繞過 mypy,改成 `dict[str, ModuleType]`,和 `_get_plot_backend` 的回傳型別一致。
- Affects Version:先查 master 版本 :

```
$ cd ~/src/spark
$ grep -n "SNAPSHOT" pom.xml | head -3
29:  <version>5.0.0-SNAPSHOT</version>
$ grep -n "__version__" python/pyspark/version.py
19:__version__: str = "5.0.0.dev0"
```

  兩者一致,master 對應 **5.0.0**。
  --> ⚠️ SPARK-58244 在 7/22 的 Fix Version 是 4.3.0，可能被改版號。

### 2.3 建環境與確認測的是哪份程式

- 開發環境:`~/src/spark`,uv 管理的 `pyspark-dev` venv(**沒有 pip**),Python 3.11.17(PySpark 啟動畫面顯示)。
- 先確認 `pyspark` 指到 repo 原始碼,不是別處安裝的版本 :

```
>>> import pyspark
>>> print(pyspark.__file__)
/home/soarb/src/spark/python/pyspark/__init__.py
```

- 查 protobuf 系列版本時,`python -m pip list` 報 `No module named pip`;單打 `pip` 則指到 `/usr/lib/python3/dist-packages/pip`(系統的 Python 3.14),不是 venv。**不要用它 install**,會裝到系統 Python。改用 venv 的 Python 直接查:

```bash
python -c "
import importlib.metadata as m
for p in ['protobuf', 'grpcio', 'grpcio-status', 'grpc-stubs', 'mypy-protobuf', 'types-protobuf']:
    try:
        print(p, m.version(p))
    except m.PackageNotFoundError:
        print(p, 'NOT INSTALLED')
"
```

| 套件 | 我的環境 | `dev/requirements.txt` |
|---|---|---|
| protobuf | 6.33.5 | `==6.33.5` |
| grpc-stubs | 1.24.11 | `==1.24.11`  |
| mypy-protobuf | 3.3.0 | `==3.3.0`  |
| grpcio / grpcio-status | 1.84.0 | `>=1.76.0`  在範圍內 |
| types-protobuf | 7.35.1.20260906 | ⚠️ 我印象中 requirements 沒釘它,是被別的套件帶進來的最新版 |

⚠️ 這張表的數字是對話轉述,原始輸出沒撈到。`uv pip install protobuf==6.33.5` 回報 "Checked 1 package",代表已經是該版本。

### 2.4 驗證

**ruff**:

```
$ dev/lint-python --ruff
...
ruff format checks passed.

all lint-python tests passed!
```

**mypy**(Python 3.11.17、mypy 1.19.1,與 `dev/requirements.txt` 一致):

```
$ dev/lint-python --mypy
starting mypy annotations test...
annotations failed mypy checks:
python/pyspark/sql/connect/client/core.py:1381: error: "FieldDescriptor" has no attribute "label"  [attr-defined]
python/pyspark/sql/connect/client/core.py:1392: error: "FieldDescriptor" has no attribute "label"  [attr-defined]
python/pyspark/sql/connect/client/core.py:1399: error: "FieldDescriptor" has no attribute "label"  [attr-defined]
Found 3 errors in 1 file (checked 1327 source files)
1
```

-  `sql/plot/core.py` 沒有任何錯誤。剩下的 3 個錯誤都在我沒碰的 `sql/connect/client/core.py`。
- ⚠️ 「這 3 個錯誤在**未修改的 master** 上也存在」這件事,我沒在撈到的對話裡看到在乾淨 master 上重跑 mypy 的原始輸出;能確定的只有它們都在我沒改的檔案裡。
- ⚠️ 這 3 個錯誤的**原因我沒有查出來**。推測是新版 `types-protobuf` 改了 `FieldDescriptor` 的 stub(`types-protobuf<7` 降版重跑的實驗,對話裡沒看到做過)。所以 PR 描述只能寫事實,不能寫原因。
- 我一度想把這 3 個錯誤一併修掉,被勸阻了:範圍不同,而且根因沒驗證;我接受這個理由。

**相關測試**(`python/pyspark/sql/tests/plot/test_frame_plot.py` 在第 49 行有測不支援的 backend,見 1.3 的 grep 輸出;我把它當成會碰到 `PySparkPlotAccessor` 的測試):

```
$ python/run-tests --testnames 'pyspark.sql.tests.plot.test_frame_plot'
Will test against the following Python executables: ['/home/soarb/venvs/pyspark-dev/bin/python3']
Will test the following Python tests: ['pyspark.sql.tests.plot.test_frame_plot']
/home/soarb/venvs/pyspark-dev/bin/python3 python_implementation is CPython
/home/soarb/venvs/pyspark-dev/bin/python3 version is: Python 3.11.17
Starting test(...): pyspark.sql.tests.plot.test_frame_plot (temp output: ...)
Finished test(...): pyspark.sql.tests.plot.test_frame_plot (14s)
Tests passed in 14 seconds
```

 通過。它證明 class body 裡的新註解在 runtime 能正常求值、模組能 import;**不證明** cache 行為(sql 版本來就沒有寫入點)。

### 2.5 分支、commit 與 PR

我當時照 #57407 的做法(分支名不帶 JIRA 編號),流程如下(這是對話裡給我的指令,不是我的終端機輸出):

```bash
git checkout -b typeignore-var-annotated
git status --short                    # 只應該有 core.py 一個 M,外加未追蹤的 .coverage(不要 add)
git add python/pyspark/sql/plot/core.py
git commit                            # 不帶 -m,會開編輯器,方便寫多行訊息
```

- commit / PR 標題:`[SPARK-60145][PYTHON] Remove type: ignore[var-annotated] in pyspark.sql`。
- 第一次 push 之前,要先在自己的 fork 啟用 GitHub Actions(Spark 的 PR 檢查是跑在貢獻者自己的 fork)。
- ⚠️ 分支名:對話裡建議 `typeignore-var-annotated`,PR 頁面摘要顯示的是另一個名字,我沒交叉確認,日誌以 PR 頁面為準,不寫死。

PR 描述骨架(依這份 repo 的 PR 範本;實際內容以我實際做過的事為準):

| 欄位 | 內容 |
|---|---|
| What | 把 `PySparkPlotAccessor._backends` 的 `# type: ignore[var-annotated]` 換成明確的 `dict[str, ModuleType]` |
| Why | value 型別和 `_get_plot_backend` 的回傳型別 `ModuleType` 一致;可選一句提 1.6 的行為觀察 |
| User-facing change | No |
| How tested | 見下方草稿 |
| Generative AI | 依實際情況如實回答 |

How tested 的草稿(對話裡給我的版本,**不要照抄 #57407 的 `Success: no issues found`**,因為我的 mypy 不是全綠):

```
### How was this patch tested?

- `dev/lint-python --ruff` passes.
- `dev/lint-python --mypy`: no errors in `python/pyspark/sql/plot/core.py`. The 3 remaining `"FieldDescriptor" has no attribute "label"` errors in `python/pyspark/sql/connect/client/core.py` also occur on an unmodified master in my environment, so they are unrelated to this change.
- `python/run-tests --testnames pyspark.sql.tests.plot.test_frame_plot` passes.
```

⚠️ 第二點的 "also occur on an unmodified master" 就是 2.4 標示的沒有原始輸出佐證的那一句;最終送出的版本我沒有逐字確認(PR 頁面只有摘要)。

### 2.6 PR 之後

-  上游 `0aa4f91` 的 `sql/plot/core.py:106` 已經是 `_backends: dict[str, ModuleType] = {}`,所以這個修改確實已經在 master 裡。
- PR 頁面顯示:2026-10-11 合併;merge commit `37913ef`,並 cherry-pick 為 `dd163c5`(⚠️ 只來自 PR 頁面。我嘗試 `git fetch --depth=1 origin 37913ef` 失敗(`couldn't find remote ref`),所以合併當下的 diff 無法自己驗證;cherry-pick 到哪個 branch 我也不知道)。
- 我另外用 PR 的 `.diff` 核對過改動內容,見 3.1。

---

## 3. 修改內容

### 3.1 改了什麼

PR #59351 的 diff(從 `.diff` 讀到的完整內容):

```diff
diff --git a/python/pyspark/sql/plot/core.py b/python/pyspark/sql/plot/core.py
index dfa4998b95f1d..8109d49c7bdcb 100644
--- a/python/pyspark/sql/plot/core.py
+++ b/python/pyspark/sql/plot/core.py
@@ -103,7 +103,7 @@ class PySparkPlotAccessor:
         "pie": PySparkTopNPlotBase().get_top_n,
         "scatter": PySparkSampledPlotBase().get_sampled,
     }
-    _backends = {}  # type: ignore[var-annotated]
+    _backends: dict[str, ModuleType] = {}
 
     def __init__(self, data: "DataFrame"):
         self.data = data
```

- 範圍:`python/pyspark/sql/plot/core.py` 一個檔案、1 行改 1 行;不需要新 import(`ModuleType` 本來就在)。
- 使用者可見行為沒有改變。

### 3.2 為什麼 ignore 要整個拿掉

目的是把「讓 mypy 閉嘴」換成「告訴 mypy 這個 dict 裝什麼」。註解寫好之後 ignore 就多餘了,而 `warn_unused_ignores = True` 會讓多餘的 ignore 變成新的錯誤,所以不是「加了註解再留著 ignore」。(第二句是從設定推的,我沒另外做「保留 ignore」的實驗 ⚠️。)

### 3.3 沒做的事

- pandas 版 `pandas/plot/core.py:487` 的同款 ignore:刻意不動(見 1.5)。
- sql 版 `_backends` 沒有寫入點、cache 不會命中:不屬於型別修改(見 1.6)。
- 3 個 `FieldDescriptor.label` 的 mypy 錯誤:不屬於這個 PR,根因也沒驗證(見 2.4)。

---

## 4. 踩過的坑

| 雷 | 症狀 | 正解 |
|---|---|---|
| 在 JQL 模式的搜尋框打自由文字 | 紅字 `Expecting operator but got '_backends'`,下面的 "No issues were found" 看起來像沒有 ticket | 搜尋框要用 `欄位 運算子 值` 的 JQL,例如 `project = SPARK AND text ~ "..."` |
| 把 JQL 的 0 筆當成「沒人做過」 | `text ~` 是斷詞後的模糊搜尋,另一個查詢還拉出不相關的 SPARK-38637 | 0 筆只支持「沒人用這個名稱開過 ticket」;再搭配 GitHub PR 搜尋與本機 `git log --grep` |
| 把 PR 顯示 "closed" 當成被拒絕 | #57407 在 GitHub 上顯示 closed,但其實已合併 | 用 `git log --oneline --grep="SPARK-xxxxx"` 確認是否已進 master(committer 用腳本合併) |
| 第一版 grep 只比對 `ignore[var-annotated` 開頭 | 抓不到 `ignore[attr-defined, var-annotated]` 這種組合寫法 | 另外用 `git grep -n "var-annotated" python/pyspark/sql` 不限定開頭,並搭配 `HEAD` 版本做前後對照 |
| venv 裡沒有 pip | `python -m pip list` 報 `No module named pip`,單打 `pip` 指到系統 Python 3.14 | 用 `importlib.metadata` 查版本;`uv pip install --python "$(which python)" ...` 才會裝進 venv |
| 以為 `mypy` 全綠才算過 | 乾淨的狀態下仍有 3 個 `FieldDescriptor.label` 錯誤,不是我造成的 | 逐一說明這 3 個錯誤在哪個檔案,只寫親眼看過的事實,不寫沒驗證的原因 |
| 想順手修掉那 3 個錯誤 | 範圍擴大,而且根因沒驗證 | 另案處理(例如先做 `types-protobuf<7` 降版實驗);這個 PR 只動一行 |
| 想一起改 pandas 版 | `pandas/plot/core.py` 有同款 ignore,但 mypy 規則本來就比較寬鬆 | PR 範圍縮在 `pyspark.sql`,標題寫「in `pyspark.sql`」,不要寫成整個 PySpark |
| 照抄前例的測試欄 | #57407 寫 `Success: no issues found`,我的結果不是全綠 | 只列自己真的跑過的指令與結果;改過程式碼就要重跑 |
| 把 `.coverage` 一起 add | `git status --short` 會看到未追蹤的 `.coverage` | 只 `git add python/pyspark/sql/plot/core.py` |
| `git commit` 不帶 `-m` 開了編輯器 | 不熟 nano | `Ctrl+O` 存檔、`Ctrl+X` 離開 |
| 同類 PR 同一天很多 | 不確定有沒有人撞車 | 開 PR 之前再做一次 JIRA、GitHub 與本機 `git log` 搜尋 |
