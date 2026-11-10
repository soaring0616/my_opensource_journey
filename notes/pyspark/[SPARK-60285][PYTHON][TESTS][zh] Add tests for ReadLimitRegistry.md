# [Apache Spark 新手實驗日誌][SPARK-60285] 新增ReadLimitRegistry 測試
    tags: Apache Spark 新手實驗日誌, Apache Spark, PySpark, open source, 實驗日誌, 測試覆蓋率
    description: 第一次為 Apache Spark 補測試的完整紀錄,含假設、驗證、踩雷與指令
---

> JIRA:SPARK-60285
> PR:[apache/spark#59550](https://github.com/apache/spark/pull/59550)
> 2026.10 版本

[TOC]

:::info
**這份日誌怎麼讀**
建構的時候有 Claude 小助手，然後寫 write-up 也是用 Claude 幫我整理的 
    （有故意用 Sonnet，還是得操一下我的小腦袋XD） 
- ⚠️ = 推測或沒有驗證 
- 程式碼區塊裡的輸出,是從當時的對話中找到的原始 log
    - 有時候是我沒貼給 Sonnet (我沒開Claude-code, 我是用 Claude Chat)
- 破壞實驗(2.4)的終端機輸出，對話搜尋只留下 Claude 當時的轉述，沒有原文，所以那一節的引文不是 verbatim
:::

---

## 1. 了解程式

### 1.1 覆蓋率缺口

- [Codecov](https://app.codecov.io/gh/apache/spark) 對 `python/pyspark/sql/datasource_internal.py` 顯示 100 行、53 行 miss、1 行 partial。
- 逐段對照後,缺口集中在 `ReadLimitRegistry`。
    - 在 sprint 聽說可以多找 `sql`，所以從[這裡](https://app.codecov.io/gh/apache/spark/tree/master/python%2Fpyspark%2Fsql)開始找
    - 記得沒錯是把整段讓sonnet，後來就被推薦這裡 （太菜了😂）

![Screenshot 2026-10-24 at 3.27.24 PM](https://hackmd.io/_uploads/H12lCRF3Gx.png)

- 內建五個型別中,既有 Python 測試只用到 `ReadAllAvailable` 與 `ReadMaxRows`;`ReadMinRows`、`ReadMaxFiles`、`ReadMaxBytes` 從未被測試建構。
    - 對照上游測試檔 [`test_python_streaming_datasource.py`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/tests/test_python_streaming_datasource.py):
    - 只 import `ReadAllAvailable`(L37)與 `ReadMaxRows`(L39);
    - 用到的地方是 [L225 `ReadMaxRows(2)`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/tests/test_python_streaming_datasource.py#L225)、[L229 `isinstance(limit, ReadAllAvailable)`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/tests/test_python_streaming_datasource.py#L229)、L533、L560。
    - 整份測試檔裡沒有出現 `ReadLimitRegistry` 這個名稱,也沒有任何一處用 dict 直接呼叫 `get` 去重建 limit。
    - ⚠️ 端到端的測試(L225 一帶)在 worker 子行程裡可能會走到 registry,但那個執行 coverage 量不到(這是 1.4 的推測,沒驗證)。
- `get` 的兩個錯誤路徑(`_type` 缺失、型別未註冊)沒有測試。


### 1.2 被檢查的原始碼
>zoom-in by instruction from Sonnet:

- 用 `sed` 看 import 與 registry 的位置(在 repo 根目錄執行;在 `python/` 底下執行會出現 `No such file or directory`):

```
$ grep -n "class ReadLimitRegistry\|def get\|def __register\|^import\|^from" python/pyspark/sql/datasource_internal.py
19:import copy
20:import json
21:from itertools import chain
22:from typing import Dict, Iterator, List, Optional, Sequence, Tuple, Type
24:from pyspark.errors import PySparkNotImplementedError
25:from pyspark.errors.exceptions.base import PySparkException
26:from pyspark.sql.datasource import (
32:from pyspark.sql.streaming.datasource import (
40:from pyspark.sql.types import StructType
92:    def getDefaultReadLimit(self) -> ReadLimit:
146:    def getCache(self, start: dict, end: dict) -> Optional[Iterator[Tuple]]:
172:class ReadLimitRegistry:
182:    def __register(self, read_limit_type: Type["ReadLimit"]) -> None:
188:    def get(self, params_with_type: dict) -> ReadLimit:
```

- 再用 `sed` 把 import 區塊(32 到 39 行)與 registry 本體(172 到 200 行)印出來:
     
```bash
sed -n '32,39p;172,200p' python/pyspark/sql/datasource_internal.py
```

registry 本體([`datasource_internal.py` L172-L199](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/datasource_internal.py#L172-L199)，節錄:

```python
class ReadLimitRegistry:
    def __init__(self) -> None:
        self._registry: Dict[str, Type[ReadLimit]] = {}
        # Register built-in ReadLimit types
        self.__register(ReadAllAvailable)
        self.__register(ReadMinRows)
        self.__register(ReadMaxRows)
        self.__register(ReadMaxFiles)
        self.__register(ReadMaxBytes)

    def __register(self, read_limit_type: Type["ReadLimit"]) -> None:
        name = read_limit_type.__name__
        if name in self._registry:
            raise PySparkException(f"ReadLimit type '{name}' is already registered.")
        self._registry[name] = read_limit_type

    def get(self, params_with_type: dict) -> ReadLimit:
        type_name = params_with_type["_type"]
        if type_name is None:
            raise PySparkException("ReadLimit type name is missing.")

        read_limit_type = self._registry.get(type_name)
        if read_limit_type is None:
            raise PySparkException("name '{}' is not registered. ...")
        ...
```

* 後面破壞實驗會用到的行號:

| 行 | 內容 |
|---|---|
| 179 | `self.__register(ReadMaxFiles)` |
| 185 | 重複註冊的 `raise` |
| 190 到 191 | `_type` 缺失的檢查 |
| 195 | 未註冊的 `raise` |
| 199 | `return read_limit_type(**params_without_type)`(把檢查拿掉後,`TypeError: 'NoneType' object is not callable` 就是從這行來的) |

### 1.3 驗證過的事實

- `ReadLimitRegistry` 在 `datasource_internal.py` 第 172 行。
- `get(self, params_with_type: dict) -> ReadLimit` 是**實例方法,收 dict**。
- dict 鍵名:`min_rows`、`max_rows`、`max_files`、`max_bytes`。
- 五個 `ReadLimit` 子類都是 `@dataclass`([`streaming/datasource.py`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/streaming/datasource.py#L40-L41) 的 `ReadAllAvailable` 等,上游行號 L41、L49、L64、L74、L84;我當時本機少 2 行:39、47、62、72)。所以 `==` 比較欄位值,測試才可以直接 `assertEqual(actual, expected)`。 這點是我後來讀上游原始碼才確認的;我當時那次 `grep` 的輸出裡看不到 `def __eq__`,合理的解釋是 `__eq__` 由 `@dataclass` 自動產生、檔案裡沒有明寫(這個解釋是推論;測試 `assertEqual` 通過是它的間接證據)。

### 1.4 未驗證的推測

- `python_streaming_source_runner.py` 在 worker 子行程使用 registry( 見 `git grep`;上游位置:[`streaming/python_streaming_source_runner.py` L82](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/streaming/python_streaming_source_runner.py#L82) 模組層級 `READ_LIMIT_REGISTRY = ReadLimitRegistry()`,[L192](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/streaming/python_streaming_source_runner.py#L192) `READ_LIMIT_REGISTRY.get(limit)`),但覆蓋率報表沒有這個檔案。
- ⚠️ 我當時猜 worker 端執行沒被量到,這解釋了為什麼 registry 是紅的。**沒有驗證**,所以 PR 描述只寫「Python 測試沒有直接覆蓋」,不寫「完全沒被測過」(Scala 端我沒查)。

---

## 2. 解決步驟

### 2.1 確定目標

- 在 Codecov 找覆蓋率低的檔案,目標確定為 `ReadLimitRegistry`。
- 只改 `test_python_streaming_datasource.py`。
- 環境:`~/src/spark`,`(pyspark-dev)` 虛擬環境(uv 管理,Python 3.11.17,見 2.3 的 traceback 路徑)。

### 2.2 確認沒人做過

假設:沒有人為 registry 寫過測試,也沒有進行中的工作。三條線:JIRA、GitHub PR、本機 git 歷史。

**JIRA**:`https://issues.apache.org/jira/issues/?jql=`,切到 Advanced(JQL)模式貼上。JQL 預設就包含已關閉的 issue,不用另加狀態條件。

第一組(找測試工作是否有人做):

```
project = SPARK AND text ~ "ReadLimitRegistry" ORDER BY created DESC
project = SPARK AND text ~ "datasource_internal" ORDER BY created DESC
project = SPARK AND component = "PySpark" AND text ~ "ReadLimit" ORDER BY created DESC
project = SPARK AND text ~ "SimpleStreamReaderWrapper" ORDER BY created DESC
project = SPARK AND text ~ "python streaming data source" AND text ~ "test" ORDER BY created DESC
```

第二組(查到 docstring 的 PR #56473 之後,在「Coverage 新手入門」那個對話拿到的):

```
key in (SPARK-56367, SPARK-55450, SPARK-55304)
project = SPARK AND text ~ "ReadLimitRegistry"
project = SPARK AND text ~ "ReadMaxFiles"
project = SPARK AND text ~ "python streaming data source" AND text ~ "test" AND resolution = Unresolved ORDER BY updated DESC
```

我日誌原稿裡記下的結果,是這一條:

```
project = SPARK AND text ~ "ReadLimit" ORDER BY created DESC
```

⚠️ 上面這些 JQL 是對話中 Claude 給的清單。我自己記下的結果只有「`text ~ "ReadLimit"` 無相關」;其他每一條有沒有逐一跑、各自的結果,對話紀錄裡沒留下。另外 `datasource_internal` 有底線,搜尋引擎可能拆成兩個詞,結果會比較雜,要看標題挑;component 名稱 `PySpark` 是當時的印象,報錯就改用下拉選單選。

**GitHub PR**(不要只看 open,已關閉的有時有「為什麼不做」的討論):

```
https://github.com/apache/spark/pulls?q=is%3Apr+ReadLimitRegistry
https://github.com/apache/spark/pulls?q=is%3Apr+datasource_internal
https://github.com/apache/spark/pulls?q=is%3Apr+is%3Aopen+test_python_streaming_datasource
https://github.com/apache/spark/pulls?q=is%3Apr+is%3Aopen+ReadLimit
```

有裝 `gh` 的話:

```bash
gh pr list --repo apache/spark --state all --search "ReadLimitRegistry" --limit 20
gh pr list --repo apache/spark --state all --search "datasource_internal" --limit 20
```

另外我在 GitHub 搜了 `is:pr limit.maxRows`,找到 docstring 修正已有人在做(見下)。

**本機 git 歷史**(網頁搜尋只找得到文字提到名稱的 PR;改了測試檔卻沒提名稱的會漏掉,所以本機這條最可靠):

```bash
cd ~/src/spark
git fetch upstream
git grep -n "ReadLimitRegistry" upstream/master -- python/
git log upstream/master --oneline -S"ReadMinRows" -- python/pyspark/sql/tests
git show --stat --format= cbcc7f8a7f7   # SPARK-55304
```

| 檢查 | 結果 |
|---|---|
| `git grep ReadLimitRegistry` | 只有類別定義與 runner 的 import / 使用,`tests/` 無  |
| `git log -S"ReadMinRows"`(測試目錄) | 無輸出,歷史上測試從未出現  |
| JIRA JQL `text ~ "ReadLimit"` | 無相關  |
| GitHub `is:pr is:open ReadLimit` | 無  |

旁支發現:相關的 docstring 錯誤(`limit.maxRows` vs `limit.max_rows`)已有人在修(SPARK-56367 / PR #56473,open)。決定**不碰**,避免重複。

### 2.3 寫測試

- 原則:每個斷言都要對照原始碼或 REPL,不憑印象(先驗證,再斷言)。
- 測試內容見第 3 節。
- 跑單一測試類別(在 `python/` 底下執行,因為 `python -m unittest` 要能 import `pyspark`):

```bash
cd ~/src/spark/python
python -m unittest pyspark.sql.tests.test_python_streaming_datasource.ReadLimitRegistryTests -v
```

- PR 描述裡的版本(格式是照 `AGENTS.md` 的範例推的,⚠️ 我沒有留下這行實際跑過的輸出):

```bash
python/run-tests --testnames "pyspark.sql.tests.test_python_streaming_datasource ReadLimitRegistryTests"
```

寫的過程中留下的三份失敗輸出:

**(1) 從錯的模組 import,整個測試檔載不進來**

```
$ python -m unittest pyspark.sql.tests.test_python_streaming_datasource.ReadLimitRegistryTests -v
test_python_streaming_datasource (unittest.loader._FailedTest.test_python_streaming_datasource) ... ERROR

======================================================================
ERROR: test_python_streaming_datasource (unittest.loader._FailedTest.test_python_streaming_datasource)
----------------------------------------------------------------------
ImportError: Failed to import test module: test_python_streaming_datasource
Traceback (most recent call last):
  File "/home/soarb/.local/share/uv/python/cpython-3.11.17-linux-x86_64-gnu/lib/python3.11/unittest/loader.py", line 162, in loadTestsFromName
    module = __import__(module_name)
             ^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/soarb/src/spark/python/pyspark/sql/tests/test_python_streaming_datasource.py", line 45, in <module>
    from pyspark.sql.streaming.datasource import ReadLimitRegistry
ImportError: cannot import name 'ReadLimitRegistry' from 'pyspark.sql.streaming.datasource' (/home/soarb/src/spark/python/pyspark/sql/streaming/datasource.py)


----------------------------------------------------------------------
Ran 1 test in 0.000s

FAILED (errors=1)
```

修法:`ReadLimitRegistry` 在 `pyspark.sql.datasource_internal`,不在 `pyspark.sql.streaming.datasource`。因為 import 在模組頂層,壞掉的不只是新測試,是整個檔案的既有測試。

**(2) 只在方法內補了一部分 import**

```
ERROR: test_read_limit_registry_buils_all_builtin_types (pyspark.sql.tests.test_python_streaming_datasource.ReadLimitRegistryTests.test_read_limit_registry_buils_all_builtin_types)
The registry rebuilds every builtin ReadLimit type and returns the correct string representation.
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/home/soarb/src/spark/python/pyspark/sql/tests/test_python_streaming_datasource.py", line 772, in test_read_limit_registry_buils_all_builtin_types
    ({"_type": "ReadMinRows", "min_rows": 10}, ReadMinRows(10)),
                                               ^^^^^^^^^^^
NameError: name 'ReadMinRows' is not defined

----------------------------------------------------------------------
Ran 1 test in 0.000s

FAILED (errors=1)
```

這份輸出同時暴露了草稿的 typo(`buils`)與 docstring 與實作不符。接著用 `grep` 看每個 `Read*` 類別的建構子,而不是憑印象猜參數(行號是我當時本機的版本;上游現在是 24、41、49、64、74、84):

```
$ grep -n "^class Read\|def __init__\|def __eq__" python/pyspark/sql/streaming/datasource.py
22:class ReadLimit:
39:class ReadAllAvailable(ReadLimit):
47:class ReadMinRows(ReadLimit):
62:class ReadMaxRows(ReadLimit):
72:class ReadMaxFiles(ReadLimit):
...
```

**(3) 多打一組括號**

`actual = registry().get(params)` 讓五組同時失敗,錯誤是 `'ReadLimitRegistry' object is not callable`。`registry` 已經是實例;改成 `registry.get(params)`。

> ⚠️ (3) 是 Claude 當時對輸出的描述,不是原始輸出。

### 2.4 破壞實驗(Break Test)

測試會通過,不代表它有用。暫時註解掉被測程式的關鍵行,確認對應測試**會失敗**,再還原。

| 實驗 | 動作 | 結果 |
|---|---|---|
| 測試一(內建型別) | 註解 `datasource_internal.py` 第 179 行 `self.__register(ReadMaxFiles)` | 只有 `read_limit='ReadMaxFiles'` 那組失敗(`errors=1`),訊息 `name 'ReadMaxFiles' is not registered.`,來自 `get` 第 195 行;其餘四組沒出錯  |
| 測試二(未註冊型別) | 註解未註冊檢查(`if read_limit_type is None:` 連同 `raise`) | 只有 `params={'_type': 'NoSuchLimit'}` 那組失敗(`errors=1`),`TypeError: 'NoneType' object is not callable`,來自第 199 行  |
| 測試二(`_type` 缺失) | 註解第 190 到 191 行 | ⚠️ Claude 建議過這個實驗(預期該組失敗,訊息變 `name 'None' is not registered.`),我沒留下結果,不確定有沒有做 |

⚠️ 以上「結果」欄是 Claude 當時對我貼出的輸出所做的轉述,不是原始輸出。

限制(當時就標過):

- 測試一我只看到 `ReadMaxFiles` 被抓到,其他四個型別沒試。邏輯上是同一套機制,但這是推論,不是驗證。
- 失敗型態是 `TypeError` 而不是斷言失敗,這是正常的,因為少了檢查之後程式會直接呼叫 `None`。

還原與確認(在 repo 根目錄,或在 `python/` 底下用不帶 `python/` 前綴的路徑):

```bash
git restore python/pyspark/sql/datasource_internal.py     # 在 repo 根目錄
git status                                                # 只應該看到測試檔被修改
git diff --stat
```

 實驗後 `git status` 只列出測試檔,`datasource_internal.py` 沒有出現,原始碼與 HEAD 一致。

以後每個新測試都做這一步,這是 reviewer 會在意的「這個測試防的是哪種回歸」的具體證據。

### 2.5 量覆蓋率

這條指令是從我的 shell history 找出來的(第 727 與 820 行),不是 Claude 猜的版本(猜的是 `--python-executables=python`,實際是 `python3`):

```bash
cd ~/src/spark
which python3        # 確認是 venv 的,不是系統的 python3.14
python/run-tests-with-coverage --testnames 'pyspark.sql.tests.test_python_streaming_datasource' --python-executables=python3
cd python
coverage report -m --include="*/datasource_internal.py"
```

關於腳本的行為(讀腳本得到的事實 ):

- 結果放在 `python/test_coverage/coverage_data`,每次執行會先清掉舊資料,不會累積。
- 腳本一開頭 `pushd "$FWDIR"`,最後的 `coverage combine` 在 `python/` 目錄裡執行,所以合併後的 `.coverage` 在 `python/`,不是 repo 根目錄。
- 腳本只產生 `python/coverage.xml`,不產生終端機報表,報表要自己跑 `coverage report`。

輸出(整份測試檔跑完,約 55 秒,`Tests passed`):

```
pyspark/sql/datasource_internal.py   100   35   24   2   57%
```

- 100 個 statement,未覆蓋 53 → 35,少了 18 個;`datasource_internal.py` 本機覆蓋率 **39% → 57%** 
- 18 剛好等於預期:registry 的 19 個紅行扣掉第 185 行(重複註冊)。⚠️ 這只是算術吻合,報表沒有 `-m`,沒有直接看到是哪些行。
- 要直接確認 Missing 欄,資料檔不在預設位置,指令是(⚠️ 我沒留下這條的輸出,`--data-file` 是否被我的 coverage 版本支援也沒驗證):

```bash
cd ~/src/spark/python
coverage report -m --data-file=test_coverage/coverage_data/coverage --include="*/datasource_internal.py"
```

- 預期 Missing 只剩 wrapper 的 158、161 到 163、169 行與 185 行。⚠️ 預期,不是實測。
- ⚠️ 39% 是 rebase 之前量的,與 57% 的基底不一定相同,所以 PR 描述寫「local run」,不寫成 CI 結果。Codecov 對這個 PR 的結果尚未確認。
- 覆蓋率不是目的。Reviewer 要看的是測試能防哪一種具體回歸(例如 registry 漏註冊某個型別,這在原始 feature PR 的 review 中真的發生過)。
- 清理:`python/test_coverage/`、`python/coverage.xml`、`python/unit-tests.log`、`python/target/`、`.coverage` 有沒有被 `.gitignore` 擋住,不確定,看 `git status`;`git add` 只指定測試檔,不要 `git add .`。`.coverage` 一直出現的話加進 `.git/info/exclude`。

### 2.6 從本機到 PR

1. **建 JIRA**:ASF 帳號 → `issues.apache.org/jira/projects/SPARK` → Create → Improvement,元件 PySpark。Priority、Fix Version、Assignee 不自己改 ⚠️(這是我對貢獻指南的印象,沒有核對 `CONTRIBUTING.md`)。
2. **commit**:標題格式 `[SPARK-60285][PYTHON][TESTS] Add tests for ReadLimitRegistry`
3. **rebase**:`git fetch upstream` 之後先看落後多少,不是 0 再 rebase,然後重跑兩個測試與 lint:

```bash
cd ~/src/spark
git fetch upstream
git log --oneline HEAD..upstream/master | wc -l
git rebase upstream/master
```

4. **找出 repo 用的 lint 工具**(一開始我以為是 `black` / `flake8`,Sonnet 當時給的指令也是這兩個,是猜錯的;實際是 `ruff`,釘在 0.14.8):

```bash
grep -in "ruff\|black\|flake8\|pycodestyle" pyproject.toml dev/lint-python dev/reformat-python dev/requirements.txt
cat dev/reformat-python
```

5. **lint 與格式化**(在 repo 根目錄):

```bash
./dev/lint-python --ruff
dev/reformat-python
ruff check --select I --fix     # import 排序
git diff                        # 確認沒有重複 import、只改了預期的檔案
```

6. **長度與非 ASCII 檢查**(Spark 的行長度上限是 100;`awk 'length>101'` 是因為 diff 的 `+` 前綴多佔一個字元):

```bash
cd ~/src/spark
git diff upstream/master...HEAD --name-only
git diff -U0 upstream/master...HEAD | grep "^+" | grep -v "^+++" | awk 'length>101 {print "too long: " $0}'
git diff -U0 upstream/master...HEAD | grep -nP "^\+.*[^\x00-\x7F]"
```

輸出 :第一個指令只列出 `python/pyspark/sql/tests/test_python_streaming_datasource.py` 一個檔案;後兩個指令沒有輸出(沒有超長行、沒有非 ASCII 字元)。

7. **開 PR**:base 是 `apache/spark` 的 `master`,head 是 `soaring0616:test-datasource-internal`。compare 連結容易開在 fork 內,要手動用 `apache/spark/compare/master...soaring0616:spark:<branch>`。

PR 描述重點:

- **What**:新增 `ReadLimitRegistryTests`,涵蓋五個內建型別與兩個錯誤路徑,test-only。
- **Why**:Python 測試覆蓋率報表顯示 registry 未覆蓋。
- **How tested**:`python/run-tests --testnames "pyspark.sql.tests.test_python_streaming_datasource ReadLimitRegistryTests"`;本機執行 `run-tests-with-coverage`,`datasource_internal.py` 覆蓋率 39% → 57%;每個測試都確認過「移除對應的註冊或檢查會失敗」。
- **AI 揭露**:`Generated-by:` 欄位要如實填寫,不能寫 `No`(測試設計、除錯與程式碼草稿用了 Claude 的協助);工具名稱與版本以實際使用為準。

PR 狀態: 直接讀上游 `0aa4f91`(2026-10-23)的 [`test_python_streaming_datasource.py`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/tests/test_python_streaming_datasource.py),裡面還沒有 `ReadLimitRegistryTests`,所以到那個 commit 為止這個 PR 還沒進 master。之後的狀態我沒查。

---

## 3. 修改內容

### 3.1 測試一:五個內建型別

```python
class ReadLimitRegistryTests(unittest.TestCase):
    def test_read_limit_registry_builds_all_builtin_types(self):
        """ReadLimitRegistry.get rebuilds each builtin ReadLimit from its dict form."""

        registry = ReadLimitRegistry()
        cases = [
            ({"_type": "ReadAllAvailable"}, ReadAllAvailable()),
            ({"_type": "ReadMinRows", "min_rows": 10}, ReadMinRows(10)),
            ({"_type": "ReadMaxRows", "max_rows": 20}, ReadMaxRows(20)),
            ({"_type": "ReadMaxFiles", "max_files": 30}, ReadMaxFiles(30)),
            ({"_type": "ReadMaxBytes", "max_bytes": 4096}, ReadMaxBytes(4096)),
        ]

        for params, expected in cases:
            with self.subTest(read_limit=params["_type"]):
                actual = registry.get(params)
                self.assertEqual(actual, expected)
```

為什麼可以直接 `assertEqual(actual, expected)`:`ReadLimit` 的子類是 `@dataclass`,自動有 `__eq__`(見 1.3)。

### 3.2 測試二:兩個錯誤路徑

草稿一開始接在測試一的同一個方法裡、還重新賦值 `cases`,被指出三個問題(方法名與實際行為不符、失敗時看不出是哪個行為壞、重用變數),拆成類別內的第二個方法:

```python
    def test_get_rejects_missing_or_unregistered_type(self):
        """ReadLimitRegistry.get raises PySparkException for a missing or unknown type."""
        registry = ReadLimitRegistry()
        cases = [
            ({"_type": None}, "ReadLimit type name is missing."),
            ({"_type": "NoSuchLimit"}, "name 'NoSuchLimit' is not registered."),
        ]
        for params, message in cases:
            with self.subTest(params=params):
                with self.assertRaises(PySparkException) as ctx:
                    registry.get(params)
                self.assertIn(message, str(ctx.exception))
```

| 輸入 | 預期訊息 |
|---|---|
| `{"_type": None}` | `ReadLimit type name is missing.` |
| `{"_type": "NoSuchLimit"}` | `name 'NoSuchLimit' is not registered.`(開頭) |

> ⚠️ 這段程式碼是對話中 Claude 提出、我採用的版本;我沒有重新對照 PR #59550 的 diff,與最後送進 PR 的版本是否逐字一致(變數名稱、空行)以 PR 為準。

### 3.3 刻意不測

`__register` 的重複註冊檢查(第 185 行)。它是 name-mangled 私有方法,公開介面觸發不到。

---

## 4. 踩過的坑

| 雷 | 症狀 | Correction |
|---|---|---|
| 執行位置搞混 | `ModuleNotFoundError: No module named 'pyspark'`;`grep: python/pyspark/...: No such file or directory`;破壞實驗後 `git restore python/pyspark/...` 失敗 | `git` / `grep` / lint 在 repo 根目錄;`python -m unittest` 在 `python/`;在 `python/` 底下的 git 路徑不加 `python/` 前綴 |
| 以為用 black / flake8 | lint 不通過 | * 這個 repo 用 `ruff`(釘在 0.14.8);先 `grep -in "ruff\|black\|flake8" pyproject.toml dev/lint-python dev/reformat-python` 確認  * * (後來聽了高天的演講才知道[這革命](https://github.com/apache/spark/pull/54840))|
| import 來源錯 | `ReadLimitRegistry` 從 `pyspark.sql.streaming.datasource` import,整個測試檔 `Failed to import test module`(見 2.3 (1)) | 從 `pyspark.sql.datasource_internal` import;先 `sed -n` 看原始碼的 import,再寫測試 |
| 只在方法內補一部分 import | `NameError: name 'ReadMinRows' is not defined`(見 2.3 (2)) | 五個名稱都要在檔案頂部 import,而且只 import 有用到的 |
| 建構子參數憑印象 | 四個型別缺建構子參數(只有 `ReadAllAvailable` 不用參數),`TypeError` | `grep` / REPL 實測 |
| `registry().get(...)` 多打括號 | 五組同時 `'ReadLimitRegistry' object is not callable`,輸出對 registry 的行為什麼都沒說 | `registry` 已經是實例,直接 `registry.get(...)` |
| 破壞實驗只註解 `raise` 那行 | 留下空的 `if` 區塊,`IndentationError`,整個測試檔無法載入,那次實驗無效 | 把 `if` 區塊整段一起註解 |
| 兩個測試擠在同一個方法 | 方法名與行為不符、失敗時看不出哪個行為壞 | 拆成兩個方法,每個方法一個行為 |
| docstring 不準、typo(`buils`) | 草稿沒複查 | 送出前複查 |
| 兩個頂層類別間距不足、docstring 後多餘空行、行尾空白 | ruff / 格式化不過(W291、E303 之類) | `dev/reformat-python` |
| 殘留不完整的第二個類別、檔尾半成品註解 | `SyntaxError` / 註解掉的程式碼會被 reviewer 退回 | `git diff` 檢查,commit 前刪掉 |
| import 重複 | 補名稱時多加了一個 `streaming.datasource` 區塊,用 `git diff` 才發現 | `git diff` 確認沒有重複 import;`ruff check --select I --fix` |
| 只看 `git log` 判斷有沒有人做過 | `git log` 只看得到已合併的工作 | open 的 PR 與 JIRA 另外搜;commit 標題不等於內容,用 `-S` 搜內容比讀標題可靠 |
| 以為覆蓋率腳本猜得出參數 | 猜了 `--python-executables=python`,實際是 `python3` | 先 `history \| grep -i coverage` 找原本的指令,用它 |
| fork 沒開 GitHub Actions | PR 的 CI 沒有跑 / 失敗(fork `soaring0616/spark` 的 Actions 預設是關的 ) | 到 fork 啟用 Actions,再推一個空 commit 觸發(第二個 commit `Trigger CI`) |
| 已推送的 commit 想 amend | 違反 repo 規範 | 只加新 commit(依 `AGENTS.md`) |
| `.coverage` 一直出現在 `git status` | 干擾 | 加進 `.git/info/exclude`;不要 `git add .` |
| PR 的 compare 連結開在 fork 內 | PR 開錯目標 | 手動用 `apache/spark/compare/master...soaring0616:spark:<branch>` |
| PR 描述 typo | `pyspark.sql.test...`(少一個 s)、`form its dict form` | 送出後立刻編輯描述,不需要 push |
| 本機完整 build 卡 45 分鐘 | `./build/sbt -Phive clean package` 無進度 | 放棄本機 build,改用 fork 的 CI |











