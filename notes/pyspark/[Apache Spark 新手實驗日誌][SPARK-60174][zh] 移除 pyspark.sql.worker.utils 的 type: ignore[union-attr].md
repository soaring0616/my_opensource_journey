
# [Apache Spark 新手實驗日誌][SPARK-60174] 移除 pyspark.sql.worker.utils 的 type: ignore[union-attr]

> JIRA:[SPARK-60174](https://issues.apache.org/jira/projects/SPARK/issues/SPARK-60174)
> PR:[apache/spark#59376](https://github.com/apache/spark/pull/59376)(撰寫日誌時仍是 open)
> 2026.10 版本

    tags: Apache Spark 新手實驗日誌, Apache Spark, PySpark, open source, 實驗日誌, mypy, type annotation
    description: 第二次清 type: ignore:用局部變數縮窄 __spec__ 的 Optional,保留 try/except,行為不變


[TOC]

:::info
**這份日誌怎麼讀**
- ⚠️ = 推測或沒有驗證(含「對話裡找不到原始輸出」的項目)
- 程式碼區塊裡的輸出,是從當時的對話中找到的原始 log(過長的有節錄)
- 原始碼的行號連結,都指向上游 `master` 的 commit [`0aa4f91`](https://github.com/apache/spark/commit/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4)(2026-10-23)。
    - **#59376 還沒合併,所以該版本的 `worker/utils.py` 還是舊碼(ignore 還在)**,連結看到的就是「改動前」
- 1.6 與 2.5 有一處**事後更正**:原本我和 Claude 都認為「現有測試沒走到改動行」,這份日誌寫作時重新查了測試的啟動方式,結論很可能相反。這段我標成 ⚠️,還沒用真實測試驗證
- PR 狀態來自 PR 頁面摘要
:::

---

## 1. 了解程式

### 1.1 找到目標

這是第二個 `type: ignore` 清理 PR(第一個是 [SPARK-60145](https://issues.apache.org/jira/browse/SPARK-60145)),每個 PR 要獨立,所以新 branch 從 `upstream/master` 開,不要疊在上一個 PR 上:

```bash
cd ~/src/spark
git fetch upstream
git switch --no-track -c type-union-attr upstream/master
```

(`--no-track` 是避免這個 branch 的預設 upstream 指向 `apache/spark`,之後 push 時比較不會搞混。)

先找出所有 `union-attr` 並統計各檔數量。**我下的指令把最後一個路徑打成 `python/pyspark/error`,少了 `s`,所以漏掉 `errors/utils.py`**;第二個指令(完整路徑)才是對的:

```
$ git grep -n "type: ignore\[union-attr\]" -- python/pyspark/sql python/pyspark/core python/pyspark/ml python/pyspark/error | cut -d: -f1 | sort | uniq -c | sort -rn
      2 python/pyspark/sql/tests/connect/client/test_reattach.py
      2 python/pyspark/sql/streaming/listener.py
      2 python/pyspark/ml/torch/torch_run_process_wrapper.py
      1 python/pyspark/sql/worker/utils.py
      1 python/pyspark/sql/utils.py
      1 python/pyspark/sql/tests/connect/client/test_artifact.py
      1 python/pyspark/sql/context.py
      1 python/pyspark/sql/connect/session.py
      1 python/pyspark/ml/wrapper.py
      1 python/pyspark/ml/connect/tuning.py
      1 python/pyspark/ml/connect/io_utils.py
```

```
$ git grep -n "type: ignore\[union-attr\]" -- python/pyspark/sql python/pyspark/core python/pyspark/errors
python/pyspark/errors/utils.py:63:                    "true",  # type: ignore[union-attr]
python/pyspark/sql/connect/session.py:1334:                utl = SparkContext._jvm.PythonSQLUtils  # type: ignore[union-attr]
python/pyspark/sql/c...(其餘被截斷)
```

(輸出中的 `:...skipping...` 是 git 的 pager 造成的,不是內容。之後改用 `git --no-pager grep ...`,`--no-pager` 必須放在 `git` 和子指令之間。)

**挑目標的三個條件**:

1. 一個檔案只有 1 到 2 處,範圍好控制在單一 root cause。
2. 避開 `connect/`、`_pb2.py`,以及依賴 torch、grpc 的模組。
3. 修法不能改變 runtime 行為。用 `assert x is not None` 縮窄會把原本的 `AttributeError` 變成 `AssertionError`,reviewer 可能在意,所以傾向用 `if` 分支或調整型別宣告。

**候選的分類**(⚠️ 這是 Claude 根據路徑與單行內容的判斷,當時還沒看過上下文):

| 類型 | 位置 | 判斷 |
|---|---|---|
| 最有希望 | `sql/worker/utils.py:125` | `__spec__` 在 typeshed 是 `ModuleSpec \| None`,是典型的 Optional narrowing |
| 值得看 | `sql/utils.py:510` | 單一處,不涉及 JVM 或 connect,要先看 `getter_func` 的型別 |
| 值得看 | `errors/utils.py:63` | 單一處,但 ignore 標在 `"true",` 那一行,多半是跨行表達式 |
| 先避開 | `listener.py`(138、497)、`connect/session.py:1334` | 根因是 `SparkContext._jvm` 為 Optional,波及面大 |
| 先避開 | `context.py:206` | 判斷式裡一邊用 `cls`、一邊用 `SQLContext`,mypy 無法縮窄;改了可能影響子類別行為 |
| 避開 | `tests/connect/*`、`ml/*`、`connect/*` | 依賴 grpc、torch |

三個候選我各印了前後約 15 行來看(`sed -n 112,138p python/pyspark/sql/worker/utils.py`、`sed -n 495,515p python/pyspark/sql/utils.py`、`sed -n 50,75p python/pyspark/errors/utils.py`)。`sql/utils.py` 那個的輸出 :

```python
def remote_only(func: Union[FuncT, property]) -> Union[FuncT, property]:
    ...
    if isinstance(func, property):
        # If it's a property, we need to set the attribute on the getter function
        getter_func = func.fget
        getter_func._remote_only = True  # type: ignore[union-attr]
        return property(getter_func)
    else:
        func._remote_only = True  # type: ignore[attr-defined]
        return func
```

⚠️ `errors/utils.py` 那段的輸出,以及最後為什麼沒選 `sql/utils.py`、`errors/utils.py`,對話裡沒有完整的原文。能確定的是:`errors/utils.py` 的那一處後來有人用 #59365 做掉了(見 2.1);最後我選的是 `sql/worker/utils.py:125`。

選它的理由 /⚠️:

- 單一檔案、單一處,不碰 JVM、connect、grpc。
- 根因是 typeshed 把 `__spec__` 宣告成 `ModuleSpec | None`,是標準的 Optional 縮窄,不需要改別的檔案。
- 這個檔案最近的歷史跟 type ignore 無關,表示最近沒有人動過這一行 :

```
$ git --no-pager log --oneline -5 -- python/pyspark/sql/worker/utils.py
29a4be87d65 [SPARK-59135][CONNECT][PYTHON] Handle metadata in command response streams
c95d9696402 [SPARK-58986][PYTHON] Sort import for python files
b51665875a3 [SPARK-58551][PYTHON] Python Data Sources Limit Pushdown API
759637ae75b [SPARK-56387][PYTHON][FOLLOWUP] Keep the original profiler result and create v2 for new format
0bcb17581d6 [SPARK-56387][PYTHON] Refactor ProfileResults to a dict of dict
```

  (`git log` 只看得到已合併的內容,open PR 看不到,所以還要搜 GitHub 與 JIRA,見 2.1。)

### 1.2 `worker_run` 在做什麼

位置(上游 `0aa4f91`;此版本還沒有套用本 PR,所以 ignore 還在):

| 內容 | 檔案:行 |
|---|---|
| `def worker_run(main: Callable, infile: IO, outfile: IO) -> None:` | [`python/pyspark/sql/worker/utils.py:99`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/worker/utils.py#L99) |
| `if profiler in ("perf", "memory"):` | [`utils.py:114`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/worker/utils.py#L114) |
| `if main.__module__ == "__main__":` | [`utils.py:122`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/worker/utils.py#L122) |
| `sys.modules["__main__"].__spec__.name  # type: ignore[union-attr]` | [`utils.py:125`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/worker/utils.py#L125) |
| `worker_module = worker_module.split(".")[-1]` | [`utils.py:131`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/worker/utils.py#L131) |
| 傳給 `WorkerPerfProfiler` / `WorkerMemoryProfiler` | [`utils.py:134`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/worker/utils.py#L134)、[`:137`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/worker/utils.py#L137) |

```python
# python/pyspark/sql/worker/utils.py(上游 0aa4f91,改動前)
122:            if main.__module__ == "__main__":
123:                try:
124:                    worker_module = (
125:                        sys.modules["__main__"].__spec__.name  # type: ignore[union-attr]
126:                    )
127:                except Exception:
128:                    worker_module = "__main__"
129:            else:
130:                worker_module = main.__module__
131:            worker_module = worker_module.split(".")[-1]
```

- `worker_module` 只在 `conf.profiler` 是 `"perf"` 或 `"memory"` 時才會算,用途是當 profiler 的名稱(上表最後一列)。
- 這段的意圖:如果 `main` 定義在被直接執行的檔案(`__module__ == "__main__"`),就去 `__main__` 模組的 `__spec__.name` 找真正的模組名;找不到就用 `"__main__"`。其餘情況用 `main.__module__`。(讀程式碼)
- 呼叫 `worker_run` 的檔案 (`git grep -n worker_run 0aa4f91 -- 'python/pyspark/sql/worker/*.py'`):`analyze_udtf.py`、`commit_data_source_write.py`、`create_data_source.py`、`data_source_pushdown_filters.py`、`lookup_data_sources.py`、`plan_data_source_read.py`、`python_streaming_sink_runner.py`、`write_into_data_source.py`,共 **8 個**。⚠️ 當時對話裡說的是「9 個 sql worker」,計數的指令與範圍我沒撈到,可能範圍不同。
- 每個 worker 的結構都是 `def main(infile, outfile): worker_run(_main, infile, outfile)`,`_main` 定義在 worker 自己的檔案(例如 [`analyze_udtf.py`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/worker/analyze_udtf.py#L230) 第 230 行)。所以 `_main.__module__` 取決於這個 worker 檔案是怎麼被載入的。

### 1.3 為什麼 mypy 報 `union-attr`

1. **型別宣告**:typeshed 把 `ModuleType.__spec__` 宣告成 `ModuleSpec | None`([`stdlib/types.pyi`](https://github.com/python/typeshed/blob/main/stdlib/types.pyi),當時我讀到約第 331 行,這個行號會隨 typeshed 更新而變 ⚠️)。mypy 看的是這份宣告,不看 CPython 原始碼,所以即使知道 `-m` 時一定有值,它還是會報。
2. **mypy 設定**:[`python/mypy.ini`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/mypy.ini) 的 `strict_optional = True`(L19)讓 Optional 存取成為錯誤;`warn_unused_ignores = True`(L23)讓多餘的 ignore 本身變成錯誤。(設定值是讀檔案;因果是推論,我沒另外實驗)
3. **錯誤訊息**:只刪 ignore、不改程式碼時,mypy 的訊息是 `Item "None" of "ModuleSpec | None" has no attribute "name"  [union-attr]`。⚠️ 這個原始輸出我沒撈到,文字來自我寫的 PR 描述。
4. **執行時 `__spec__` 真的可能是 `None`**:我讀的 CPython 原始碼(`Objects/moduleobject.c`)在建立模組時把 `__spec__` 初始化成 `None`;它維持 `None`,除非模組是透過 import 系統載入的,例如用 `python -m` 執行時,`runpy` 先用 `find_spec` 找到 spec,再設到 `__main__` 上。(讀過原始碼)
   - ⚠️ 這幾支原始碼(`runpy.py`、`moduleobject.c`、`pylifecycle.c`、`importlib/_bootstrap.py`)是 Claude 用 WebFetch 讀 CPython `main` 分支(約 3.15 開發版),我的 venv 是 3.11.17,**沒有逐版比對**。
   - 小發現:`-m` 加上套件名稱時,`spec.name` 會帶 `.__main__`(`runpy` 會改去找 `<pkg>.__main__`);`.split(".")[-1]` 就是在取最後一段。
5. **`dis`**:我請 Claude 解釋能不能用 `dis` 看這段的行為。`dis` 只看得到位元組碼(一連串取值、以及 3.11 以後的 exception table 範圍),看不出型別問題。⚠️ 我實際跑 `dis` 的輸出,對話裡沒看到。

### 1.4 兩個問題分開看

我一度把問題理解成「`__spec__` 可能是 `None`」加上「`modules` 裡的 `__main__` 不一定有」,Claude 的整理是:

| | 性質 | 處理方式 |
|---|---|---|
| `__spec__` 為 `None` | 型別問題,也是常見的執行情況(預設狀態) | **這次修的重點**,改成明確的 `is not None` |
| `sys.modules["__main__"]` 不存在 | 罕見的防禦情況;mypy 不會抱怨(`sys.modules` 是 `dict[str, ModuleType]`) | **不動**,讓 `try/except` 繼續接住,以維持行為不變 |

-  第二列是順便被原本寬鬆的 `except Exception` 保護到。⚠️ 原作者是不是刻意考慮過它,我不知道。
- 所以 PR 描述的理由只寫「`__spec__` 可能為 `None`」;保留 `try/except` 是為了不改變原有的例外處理行為。

### 1.5 哪些啟動路徑會跑到改動的那幾行

兩種啟動方式(`main.__module__` 的值不同):

| 啟動方式 | `main.__module__` | 走哪個分支 | 出處 |
|---|---|---|---|
| 直接 `python -m <worker 模組>` | `"__main__"` | **改動的 `__spec__` 分支**;`spec` 非 `None`,結果是該 worker 的模組名 | [`PythonWorkerFactory.scala:271`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/core/src/main/scala/org/apache/spark/api/python/PythonWorkerFactory.scala#L271):`new ProcessBuilder(Arrays.asList(pythonExec, "-m", workerModule))` |
| 經 daemon 匯入 | `pyspark.sql.worker.xxx` | `else` 分支,改動的行不會跑 | [`PythonWorkerFactory.scala:347`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/core/src/main/scala/org/apache/spark/api/python/PythonWorkerFactory.scala#L347) 以 `-m daemonModule workerModule` 啟動 daemon;[`daemon.py:84`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/daemon.py#L84) `importlib.import_module(sys.argv[1])` |

- 表中「各自走哪個分支」是由 Python 的行為推出來的(被 `-m` 執行的檔案,`main.__module__` 是 `"__main__"`;被 import 的檔案是模組名),不是在 Spark 裡量到的。
- 工廠怎麼選路徑 :`useDaemon = !Utils.isWindows && useDaemonEnabled`([L110–112](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/core/src/main/scala/org/apache/spark/api/python/PythonWorkerFactory.scala#L110-L112));`create()` 在 `useDaemon` 時走 `createThroughDaemon()`,否則走 [`createSimpleWorker(blockingMode = false)`(L179)](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/core/src/main/scala/org/apache/spark/api/python/PythonWorkerFactory.scala#L179)。
- `createSimpleWorker` 的呼叫者 (`git --no-pager grep -n "createSimpleWorker" -- '*.scala' | head`,只有 6 行,沒被 `head` 截斷):

```
core/src/main/scala/org/apache/spark/api/python/PythonWorkerFactory.scala:179:      createSimpleWorker(blockingMode = false)
core/src/main/scala/org/apache/spark/api/python/PythonWorkerFactory.scala:254:  private[spark] def createSimpleWorker(
core/src/main/scala/org/apache/spark/api/python/StreamingPythonRunner.scala:93:    val (worker: PythonWorker, _) = workerFactory.createSimpleWorker(blockingMode = true)
core/src/test/scala/org/apache/spark/api/python/PythonWorkerFactorySuite.scala:37:  test("createSimpleWorker() fails with a timeout error if worker does not connect back") {
core/src/test/scala/org/apache/spark/api/python/PythonWorkerFactorySuite.scala:49:        workerFactory.createSimpleWorker(blockingMode = true) // blockingMode doesn't matter.
sql/core/src/main/scala/org/apache/spark/sql/execution/python/streaming/PythonStreamingSourceRunner.scala:114:    val (worker: PythonWorker, _) = workerFactory.createSimpleWorker(blockingMode = true)
```

- ⚠️ **沒有驗證**:上面 8 個 sql worker 在預設設定下到底走 `-m` 直接啟動還是走 daemon。`planner runner` 一類的 Scala 類別有沒有呼叫 `create()`、`git --no-pager grep -n '"-m"' -- '*.scala' | head` 的結果,對話裡都沒看到。所以 **PR 描述裡我沒寫「正式環境走 `spec` 分支」**;如果要補一句,只能寫「該區塊只在 `main` 定義在被直接執行的檔案裡(`__main__`)時才會走到,例如 `python -m` 直接啟動 worker」。

### 1.6 現有測試有沒有走到改動行(⚠️ 事後更正)

**當時的結論**(對話裡):`pyspark.sql.tests.test_python_datasource` 通過,但「沒有走到改動行,因為 `main.__module__` 是測試模組」。這是 Claude 從測試檔結構讀出來的判斷,對話裡明確標了「沒有實際量過」。我把它寫進了 PR 描述的測試欄。

**測試本身** (上游 `0aa4f91` 的 [`test_python_datasource.py`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/tests/test_python_datasource.py)):

| 內容 | 行 |
|---|---|
| `class PythonDataSourceWorkerUtilsTests(unittest.TestCase):` | [L71](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/tests/test_python_datasource.py#L71) |
| `test_profiler_accumulators_are_created_only_when_profiling_is_enabled` | [L72](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/tests/test_python_datasource.py#L72) |
| 巢狀函式 `def main(_infile, _outfile):`(定義在測試函式裡) | [L92](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/tests/test_python_datasource.py#L92) |
| `worker_utils.worker_run(main, infile, outfile)` | [L119](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/tests/test_python_datasource.py#L119) |
| `test_cases` 含 `"perf"`、`"memory"`(會進 `if profiler in ("perf", "memory")`) | [L138–L143](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/pyspark/sql/tests/test_python_datasource.py#L138-L143) |

- 用 `mock.patch.multiple` 把 `WorkerPerfProfiler`、`WorkerMemoryProfiler` 換成 `contextlib.nullcontext()`,只檢查它們被呼叫幾次;**不檢查傳進去的 `worker_module` 值**。

**事後查到的啟動方式**(2026-10-24 寫這份日誌時查的,不是當時的對話):

- [`python/run-tests.py:254`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/run-tests.py#L254) 設 `"SPARK_TESTING": "1"`,[`:311`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/python/run-tests.py#L311) `cmd = [os.path.join(SPARK_HOME, "bin/pyspark")] + test_name.split()`。
- [`bin/pyspark:94–98`](https://github.com/apache/spark/blob/0aa4f9136bccc40856a5af688e3d16f8b38ebfc4/bin/pyspark#L94-L98):`SPARK_TESTING` 有設時 `exec "$PYSPARK_DRIVER_PYTHON" -m "$@"`。
- 所以 `python/run-tests --testnames pyspark.sql.tests.test_python_datasource` 實際是 `python -m pyspark.sql.tests.test_python_datasource`,測試模組會被當成 `__main__` 執行,它裡面巢狀定義的 `main` 的 `__module__` 就是 `"__main__"`。

**用替身腳本重現機制**(我在沙箱的 Python 3.13.17 跑的,不是 Spark 本身):

```python
# pkg/t_mod.py(巢狀 main + 與 worker_run 相同的判斷)
import sys
import unittest


class T(unittest.TestCase):
    def test_x(self):
        def main(a, b):
            pass

        spec = sys.modules["__main__"].__spec__
        print("main.__module__ =", repr(main.__module__))
        print("__main__.__spec__.name =", repr(spec.name if spec is not None else None))
        if main.__module__ == "__main__":
            worker_module = spec.name if spec is not None else "__main__"
        else:
            worker_module = main.__module__
        print("branch taken:", "changed (__spec__) branch" if main.__module__ == "__main__" else "else branch", "->", worker_module.split(".")[-1])


if __name__ == "__main__":
    unittest.main()
```

```
$ python3 -m pkg.t_mod                 # run-tests 的啟動方式
main.__module__ = '__main__'
__main__.__spec__.name = 'pkg.t_mod'
branch taken: changed (__spec__) branch -> t_mod

$ python3 -m unittest pkg.t_mod        # 另一種啟動方式
main.__module__ = 'pkg.t_mod'
__main__.__spec__.name = 'unittest.__main__'
branch taken: else branch -> t_mod
```

**目前的判斷**:

- 用 `python/run-tests` 跑(我當時就是這樣跑的),`"perf"`、`"memory"` 兩個 subtest 很可能**有**走到改動的那幾行;只用 `python -m unittest` 或 pytest 跑,才會走 `else`。所以 PR 描述裡 "it does not exercise the changed lines" 這句,對我實際用的指令來說**很可能不成立**。
- ⚠️ 我沒有在真實的 Spark 測試裡量過(沒有插 print 或看 coverage)。上面證明的是機制,不是這個測試本身。
- 建議的驗證(**break test**,還沒做 ⚠️):暫時把改動裡的 `worker_module = ...` 改成 `work_module = ...`(也就是 3.2 版本 2 的那個 typo),再跑一次 `python/run-tests --testnames pyspark.sql.tests.test_python_datasource`。我**預期**測試會失敗(`worker_module` 沒被賦值,後面的 `worker_module.split(".")` 會出錯,`main` 也不會被呼叫到);若真的失敗,就證明測試走到了改動行。若沒失敗,代表原本的說法才對。驗完要把檔案還原。
- 比較便宜的替代:在改動行後面暫時加 `print(...)`,看 `python/target/.../*.log` 或 `unit-tests.log` 有沒有出現。

---

## 2. 解決步驟

### 2.1 確認沒人做過

同一天有很多人在做類似的 `type: ignore` 清理,所以動手前、建 JIRA 前、push 前都要搜。

**JIRA(進階搜尋)**

| # | JQL | 結果 |
|---|---|---|
| 1 | `project = SPARK AND text ~ "type: ignore" AND resolution = Unresolved` |  太多:`text ~` 是斷詞後的模糊搜尋,標點被忽略,而且搜 description 與 comment,`ignoreNullable`、`ignoreCorruptFiles` 之類的字都被撈出來 |
| 2 | `project = SPARK AND resolution = Unresolved AND summary ~ "\"type ignore\""` | ⚠️ 給了指令,對話裡沒看到結果 |
| 3 | `project = SPARK AND resolution = Unresolved AND component = PySpark AND (summary ~ "mypy" OR summary ~ "union-attr" OR summary ~ "\"ignore[\"")` | ⚠️ 給了指令,對話裡沒看到結果 |
| 4 | `project = SPARK AND resolution = Unresolved AND (text ~ "union-attr" OR text ~ "worker_run" OR text ~ "worker.utils")` |  229 筆,我只看得到第一頁。其中 **SPARK-60168 `Remove type: ignore[union-attr] for the JVM and gateway handles`**(別人開的、未認領,當時才建立 29 分鐘,處理 `sql/connect/session.py`、`sql/streaming/listener.py`、`ml/connect/io_utils.py`、`ml/wrapper.py`,手法是加 `assert` 並引用 SPARK-58346)。其他多半是 `worker`、`union` 被全文比對命中的雜訊(例如 SPARK-59906 是 smart test selection)。`"worker.utils"` 會被斷詞,範圍太廣 |
| 5 | `project = SPARK AND text ~ "union-attr" ORDER BY created DESC`(不加 `resolution = Unresolved`,連已 resolved 的也看) | ⚠️ 給了指令,對話裡沒看到結果 |
| 6 | `project = SPARK AND resolution = Unresolved AND text ~ "union-attr"` | ⚠️ 給了指令,對話裡沒看到結果 |

JQL 注意:`-` 會被當成排除運算子;含標點的片語要用 `\"` 包起來。

**GitHub PR**(web):

```
https://github.com/apache/spark/pulls?q=is%3Apr+is%3Aopen+worker%2Futils.py
```

⚠️ 我被要求「沒結果就回報沒有」,結果對話裡沒看到。

**GitHub PR(Claude 用 GitHub Search API 替我查,不是我自己跑的)**,可以得到 PR 編號、標題、作者與建立時間的結構化結果:

```
https://api.github.com/search/issues?q=repo:apache/spark+is:pr+is:open+__spec__&per_page=30
https://api.github.com/search/issues?q=repo:apache/spark+is:pr+is:open+label:PYTHON+worker+profiler&per_page=30
https://api.github.com/search/issues?q=repo:apache/spark+is:pr+is:merged+union-attr&per_page=30
https://api.github.com/search/issues?q=repo:apache/spark+is:pr+is:open+mypy+in:title,body+created:>=2026-10-01&per_page=15
```

另外還搜了「open 且含 `union-attr`」「open 且含 `worker_run`」「open 且含 `worker/utils.py`」(⚠️ 這三條的完整網址我沒撈到)。結果 :

| 搜尋 | 結果 |
|---|---|
| open + `union-attr` | 2 筆:#57356(SQL 的 UNION ALL,無關)、#59365 |
| open + `__spec__` | 0 筆 |
| open + `worker_run` | 0 筆 |
| open + `worker/utils.py` | 37 筆,我只看得到 14 筆(另外 23 筆沒看到);只有 #57461(INFRA 的 CI 選測)和這個檔案有關,與型別無關 |
| open + `mypy`(10 月起) | 12 筆,標題都和 worker 無關;我只打開檢查過下表四個的檔案清單,其餘八個只看了標題 |

| PR | 狀態 | 內容 | 碰到 `sql/worker/utils.py`? |
|---|---|---|---|
| #59365 | open | `errors/utils.py` 的 `is_debugging_enabled()`,移除 `union-attr`,用 `assert` | 否(是我前面擱置的候選 #3) |
| #59355 | open | ML、mllib 的 `assignment` | 否(3 個檔案) |
| #59364 | open | `no-untyped-def` | 否 |
| #59246 | open | worker 計時指標,10 個檔案 | 否(檔案清單裡沒有它) |
| #57521 | merged | 對話摘要把它列為相關/先例 | ⚠️ 細節沒撈到 |

同一天另有一批 type-ignore 清理 PR(#59343、#59348、#59349、#59369 等),沒有一個動 `sql/worker/utils.py`。

**限制**:

- GitHub 搜尋比對的是 PR 的標題與內文,**不是檔案路徑**。如果有人移除了這個 ignore 卻沒在文字裡提到 `union-attr`,我找不到。
- #59355 的標題在 API 結果是 "in ML and Connect",網頁顯示 "in ML",可能改過標題或快取。
- 所以結論只能是「目前沒找到重複」;建 ticket 前、開 PR 前各自己再查一次(SPARK-60174 底下有沒有留言也要看)。

### 2.2 建 JIRA

- SPARK-60174,Improvement / Minor / Component PySpark。
- 與 SPARK-60168 的關係:同類型(`union-attr`)、不同檔案。**這張是我的 sibling,不重複**。在 Description 提一句,並加一條 `relates to` 連結。這是 Claude 的建議,不是 Spark 的硬性規定(`relates to` 只是連結,沒有依賴);我沒查過 Contributing 文件有沒有規定。
- ⚠️ 我最後有沒有加上 `relates to` 連結,對話裡沒看到。
- 建議的 Description(英文):

```
`worker_run` in python/pyspark/sql/worker/utils.py suppresses a mypy
`union-attr` error on `sys.modules["__main__"].__spec__.name` with
`# type: ignore[union-attr]`, because `__spec__` is typed as
`ModuleSpec | None`.

Narrow the type explicitly by storing `__spec__` in a local variable and
checking `is not None`, then remove the `type: ignore`. The existing
`try/except` fallback to "__main__" is kept, so behavior is unchanged.

This follows SPARK-58346 and is a sibling of SPARK-60168, which covers
other `union-attr` ignores in different files.
```

- 建好之後 Claude 看我貼的截圖,指出三處:
  1. **Affects Version 是 4.4.0**,SPARK-60168 填的是 5.0.0。查 master 版本的第一個指令是錯的 :

```
$ grep -m1 "<version>" pom.xml
    <version>34</version>
```

     `34` 是上層 Apache parent pom 的版本,不是 Spark 的。正確是找 `SNAPSHOT` 那行:`grep -m1 "SNAPSHOT" pom.xml`,或 `grep -i version python/pyspark/version.py`。⚠️ 這兩個指令在這個 session 的輸出我沒撈到;同一天另一個 session 的結果是 `pom.xml:29: <version>5.0.0-SNAPSHOT</version>`、`version.py:19: __version__ = "5.0.0.dev0"`(見 SPARK-60145 日誌),所以 master 當時對應 5.0.0。**最後 Affects Version 改成什麼,我沒看到。**
  2. **Description 缺了後半段**(截圖停在 "Behavior is unchanged.",少了 `try/except` 與 sibling 那一段)。
  3. **排版**:`__spec__` 顯示成 `_spec_`(JIRA 把底線當成粗體/斜體標記吃掉了);我草稿每行都手動斷行,在 JIRA 裡變成不自然的換行。只是外觀,存檔後看預覽,不正常再處理;段落改成一整行比較好。

### 2.3 建 branch

```bash
$ git switch --no-track -c SPARK-60174-worker-utils-union-attr upstream/master
M       python/pyspark/sql/worker/utils.py
Switched to a new branch 'SPARK-60174-worker-utils-union-attr'
```

`M` 表示我的修改跟著過來了 。接著確認 branch 的基底是最新的 master:

```bash
git fetch upstream
git --no-pager log --oneline -1 upstream/master
git --no-pager merge-base --is-ancestor upstream/master HEAD && echo "up to date" || echo "behind"
# 若 behind:git rebase upstream/master
```

⚠️ 這三條的結果(`up to date` / `behind`)我沒看到;只看到 `git fetch upstream` 的前幾行(`remote: Enumerating objects: 32, done.` 等)。我另外問過「rebase 自己尚未 push 的 branch 會不會干擾社群」,答案是不會,因為還沒公開。

### 2.4 驗證

**ruff**:第二版有 typo,`dev/lint-python --ruff` 抓到 :

```
$ dev/lint-python --ruff
starting ruff test...
ruff checks failed:
F841 Local variable `work_module` is assigned to but never used
   --> python/pyspark/sql/worker/utils.py:125:21
    |
123 |                 try:
124 |                     spec = sys.modules["__main__"].__spec__
125 |                     work_module = spec.name if spec is not None else "__main__"
    |                     ^^^^^^^^^^^
126 |                 except Exception:
127 |                     worker_module = "__main__"
    |
help: Remove assignment to unused variable `work_module`

Found 1 error.
No fixes available (1 hidden fix can be enabled with the `--unsafe-fixes` option).
1
```

改正 typo 後 ruff 通過、mypy 沒有 `worker/utils.py` 的錯誤:⚠️ 對話摘要說兩者都通過,但原始輸出我沒撈到。mypy 的指令(輸出必須是空的,而且是在 ignore 已刪掉的前提下才算數):

```bash
dev/lint-python --mypy 2>&1 | grep "worker/utils.py"
```

(Claude 在沙箱用的 mypy 是 2.4.0,**不一定是 Spark 用的版本**,所以沙箱的 mypy 結果不算證據,以我在 repo 裡跑的為準。)

**`git diff --stat`**:預期 `1 file changed, 2 insertions(+), 3 deletions(-)`,與 PR 的 diff 一致 (見 3.1)。

**行為比對(4 情境)**。這個腳本只複製那一小段邏輯,**不是直接呼叫 `worker_run`**,所以它證明「這段邏輯的行為一樣」,不證明整個 worker 的行為。腳本與沙箱結果 :

```python
# /tmp/spec_check.py
import sys


def orig():
    try:
        return sys.modules["__main__"].__spec__.name
    except Exception:
        return "__main__"


def new():
    try:
        spec = sys.modules["__main__"].__spec__
        return spec.name if spec is not None else "__main__"
    except Exception:
        return "__main__"


def run_both(label):
    print(f"{label:28} orig={orig()!r:12} new={new()!r:12} same={orig() == new()}")


main_mod = sys.modules["__main__"]
run_both("real __spec__ as launched")

saved = main_mod.__spec__
main_mod.__spec__ = None
run_both("__spec__ = None")
main_mod.__spec__ = saved

del sys.modules["__main__"]
run_both("__main__ not in sys.modules")
sys.modules["__main__"] = main_mod

del main_mod.__spec__
run_both("no __spec__ attribute")
main_mod.__spec__ = saved
```

```bash
cd /tmp && python3 -m spec_check          # 對應 worker 被 JVM 以 -m 啟動
python3 /tmp/spec_check.py                # 對應直接跑檔案
```

沙箱結果(Claude 跑的):4 種情境在兩種啟動方式下,`orig` 與 `new` 全部相同(`same=True`)。`-m` 啟動時第一行是 `'spec_check'`,直接跑檔案時是 `'__main__'`,這也說明為什麼只有 `-m` 才會走 `spec.name`。

⚠️ **我自己在本機跑這個腳本的結果,對話裡沒看到**。Claude 反覆強調「你自己跑過的結果,才是你能放在 PR 上的證據」,沙箱的結果不算。

**測試**(通過,但是否走到改動行見 1.6):

```
$ python/run-tests --testnames pyspark.sql.tests.test_python_datasource
Output is in /home/soarb/src/spark/python/unit-tests.log
Will test against the following Python executables: ['/home/soarb/venvs/pyspark-dev/bin/python3']
Will test the following Python tests: ['pyspark.sql.tests.test_python_datasource']
/home/soarb/venvs/pyspark-dev/bin/python3 python_implementation is CPython
/home/soarb/venvs/pyspark-dev/bin/python3 version is: Python 3.11.17
Starting test(/home/soarb/venvs/pyspark-dev/bin/python3): pyspark.sql.tests.test_python_datasource (temp output: ...)

Finished test(/home/soarb/venvs/pyspark-dev/bin/python3): pyspark.sql.tests.test_python_datasource (224s)
Tests passed in 224 seconds
```

(通過只代表「沒壞」。測到了哪些行要看 1.6。)

### 2.5 commit 與 PR

```bash
git add python/pyspark/sql/worker/utils.py
git commit -m "[SPARK-60174][PYTHON] Remove type: ignore[union-attr] in pyspark.sql.worker.utils"
# push 之前再搜一次 worker/utils.py 與 union-attr 的 open PR,並看 SPARK-60174 底下有沒有留言
git push -u origin SPARK-60174-worker-utils-union-attr
```

(第一次 push 之前,要先在自己的 fork 啟用 GitHub Actions。)

- PR #59376,base 是 `apache/spark` 的 `master`。
- PR 頁面顯示(⚠️ 摘要,非 verbatim):標題在 2026-10-11 被加上 `[TEST]`,現在是 `[SPARK-60174][PYTHON][TEST] Remove type: ignore[union-attr] in pyspark.sql.worker.utils`(我不知道為什麼加 `[TEST]`;PR 的 diff 只有 `worker/utils.py` 一個檔案);PR 頁面揭露了 Claude Code (Sonnet 5.5);2026-10-15 有 reviewer 留下 review,並對另一位發出 review 請求;review 內容這份日誌沒收 ⚠️。
- ⚠️ 我送出的 PR 描述,最終版本我沒有逐字確認。這裡放對話裡反覆修改後的 What / Why / 測試欄(測試欄改了幾輪:精簡措辭;我一度刪掉最後一條裡「沒涵蓋改動」那半句,被指出會讓審查者誤以為測試涵蓋了改動,又補回):

```
### What changes were proposed in this pull request?
In `worker_run`, read `sys.modules["__main__"].__spec__` into a local
variable and check `spec is not None` before accessing `spec.name`, instead
of suppressing the mypy `union-attr` error with `# type: ignore[union-attr]`.
The surrounding `try/except Exception` is kept.

### Why are the changes needed?
`__spec__` is declared as `ModuleSpec | None` on `types.ModuleType` in typeshed (`stdlib/types.pyi`), so mypy reports `union-attr` on `.name`. At runtime CPython initializes `__spec__` to `None` for a module (`Objects/moduleobject.c`).

Narrowing the type explicitly lets mypy check this line instead of ignoring it. This follows SPARK-58346, which removed similar `union-attr` ignores.

### Does this PR introduce _any_ user-facing change?
No.

### How was this patch tested?
- `dev/lint-python --ruff` passes.
- `dev/lint-python --mypy` reports no error for `python/pyspark/sql/worker/utils.py`.
- With the ignore removed and no narrowing, it reported `Item "None" of "ModuleSpec | None" has no attribute "name"  [union-attr]`.
- Compared the old and new expressions in four cases: `__spec__` set (run with `python -m`), `__spec__` is `None`, `__main__` missing from `sys.modules`, and no `__spec__` attribute. The result is identical in all four.
- Existing `pyspark.sql.tests.test_python_datasource` passes (it does not exercise the changed lines, since `main.__module__` is the test module there).
```

- 為什麼 Why 欄的第二句只寫「CPython 把 `__spec__` 初始化成 `None`」:原本還有一句「它維持 `None`,除非模組是經 import 系統載入,例如 `python -m`」,是 Claude 根據前面的 CPython 研究寫的,不是當下重新查證的;沒把握就刪,讓每個說法都有來源可查。
- ⚠️ **最後一條(括號內)我現在認為很可能不正確**,見 1.6。建議做完 break test 後,改成「(…runs under `python/run-tests`, where the test module is `__main__`, so the changed branch is reached)」之類的說法,或在驗證前直接把括號刪掉。PR 還開著,可以編輯描述。

---

## 3. 修改內容

### 3.1 改了什麼

PR #59376 的 diff(從 `.diff` 讀到的完整內容):

```diff
diff --git a/python/pyspark/sql/worker/utils.py b/python/pyspark/sql/worker/utils.py
index 2069cdc9e6d0b..3952bdbfb8937 100644
--- a/python/pyspark/sql/worker/utils.py
+++ b/python/pyspark/sql/worker/utils.py
@@ -121,9 +121,8 @@ def worker_run(main: Callable, infile: IO, outfile: IO) -> None:
 
             if main.__module__ == "__main__":
                 try:
-                    worker_module = (
-                        sys.modules["__main__"].__spec__.name  # type: ignore[union-attr]
-                    )
+                    spec = sys.modules["__main__"].__spec__
+                    worker_module = spec.name if spec is not None else "__main__"
                 except Exception:
                     worker_module = "__main__"
             else:
```

- 範圍:`python/pyspark/sql/worker/utils.py` 一個檔案,+2 / −3;使用者可見行為沒有改變。
- `try/except Exception` 完全沒動。

### 3.2 我寫過的三個版本

| 版本 | 寫法 | 問題 |
|---|---|---|
| 1 | 在 `if` 條件裡加 `hasattr(sys.modules["__main__"], "__spec__")`,並把 `try/except` 拿掉(⚠️ 下方是 Claude 依我貼的兩行補成的**重建版**,不是我的原始檔案) | `__main__` 不在 `sys.modules` 時,`sys.modules["__main__"]` 在任何保護範圍之外,原本被 `except` 接住的 `KeyError` 變成直接丟出,**行為退步** |
| 2 | `spec = ...` 存局部變數,但把結果存進 `work_module`(少了 `er`) | ruff F841 抓到;而且這不只是 lint 問題:成功路徑下 `worker_module` 沒被賦值,後面的 `worker_module.split(".")[-1]` 會丟 `UnboundLocalError`,只有走到 `except` 才剛好有值,很容易漏掉 |
| 3(最終) | `spec = ...` 存局部變數,`worker_module = spec.name if spec is not None else "__main__"`,`try/except` 不動 | 4 情境行為一致 |

版本 1 的重建:

```python
if main.__module__ == "__main__" and hasattr(sys.modules["__main__"], "__spec__"):
    worker_module = sys.modules["__main__"].__spec__.name if sys.modules["__main__"].__spec__ is not None else "__main__"
else:
    worker_module = main.__module__
```

版本 1 的 4 情境比較(Claude 在沙箱跑的):

| 情況 | 原版 | 版本 1 |
|---|---|---|
| `__spec__` 是 `None` | `"__main__"` | `"__main__"` |
| `__spec__` 有值 | `"pkg.mod.worker"` | `"pkg.mod.worker"` |
| `__main__` 不在 `sys.modules` | `"__main__"` | **丟出 `KeyError`** |
| `__main__` 沒有 `__spec__` 屬性 | `"__main__"` | `"__main__"` |

- 在沙箱用 mypy 2.4.0 跑版本 1,**沒有報錯**,代表 mypy 對 `sys.modules["__main__"].__spec__` 這種字面 key 的縮窄是有效的;但不等於 Spark 的設定也會過,必須在 repo 跑 `dev/lint-python --mypy`。
- `hasattr` 可能是多餘的:依 CPython 的 `module_init_dict()`,每個 module 一建立就有 `__spec__`;它只在有人手動 `del` 時才有用。
- `sys.modules["__main__"]` 重複寫了三次,存成局部變數比較好讀,也更穩定。

### 3.3 最終版(3)的 4 情境比較

| 情況 | 原版 | 最終版 |
|---|---|---|
| `__spec__` 是 `None` | `"__main__"` | `"__main__"` |
| `__spec__` 有值 | `"pkg.mod.worker"` | `"pkg.mod.worker"` |
| `__main__` 不在 `sys.modules` | `"__main__"` | `"__main__"` |
| 沒有 `__spec__` 屬性 | `"__main__"` | `"__main__"` |

 沙箱結果;⚠️ 我自己的本機執行結果沒撈到(見 2.4)。

一個細節:原版在 `__spec__` 為 `None` 時,是靠 `AttributeError` 被 `except` 接住才走到 fallback;改成最終版後,同樣的結果改由 `if spec is not None` 直接得出,不再靠例外。這是「行為不變、但意圖更明確」的地方。

### 3.4 沒選的寫法

| 寫法 | 為什麼沒選 |
|---|---|
| `assert spec is not None` | 會把 `AttributeError` 變成 `AssertionError`;`except Exception` 兩者都會接住,但 PR 要宣稱「行為不變」就多了一個要論證的地方。(`assert` 是 SPARK-60168、#59365 的做法,那些地方沒有 `try/except`) |
| `spec = getattr(sys.modules.get("__main__"), "__spec__", None)` + 三元運算式(候選 B) | 更短,4 情境也都一樣;但**把 `except Exception` 整個拿掉**,等於改了例外處理的寬度,範圍比較大 |
| `isinstance(spec, ModuleSpec)` | 多一個 `import`,而且任何「有 `.name` 但不是真 `ModuleSpec`」的物件會改走 fallback(例如驗證腳本用的 `SimpleNamespace` 替身,結果會變成 `"__main__"`)。真實執行時 `__spec__` 一定是 `ModuleSpec`,實務差別不大,但驗證方法就要換掉替身。`is not None` 比較好,兩種都可以 |

---

## 4. 踩過的坑

| 雷 | 症狀 | 正解 |
|---|---|---|
| grep 路徑打錯 | `python/pyspark/error` 少了 `s`,漏掉 `errors/utils.py:63` | 比對兩個指令的輸出;完整路徑用第二個 |
| git 預設用 pager | 輸出出現 `:...skipping...` 和 `~`,卡在 `less` | `git --no-pager grep ...`(`--no-pager` 放在 `git` 與子指令之間);離開 `less` 按 `q` |
| JQL 搜 `type: ignore` | 一大堆跟 type ignore 無關的 ticket(`ignoreNullable` 等) | 用 `summary ~` 加 `\"type ignore\"` 片語,或搜 `union-attr` 這種特殊詞 |
| JQL `"worker.utils"` | 會被斷詞,229 筆 | 改搜特殊詞(`union-attr`),並看已 resolved 的 |
| 以為沒有結果就沒人做 | GitHub 搜尋比對標題與內文,不是檔案路徑 | 只能說「目前沒找到重複」;建 ticket 前、開 PR 前再各查一次 |
| 同一天出現同類 ticket | SPARK-60168 才建立 29 分鐘 | 在 Description 提一句 sibling,必要時加 `relates to`;重複檢查要在 push 前再做一次 |
| 第一版修法拿掉 `try/except` | `__main__` 不在 `sys.modules` 時直接丟 `KeyError` | `try/except` 不動,只改取值那一行 |
| 第二版 typo | `work_module` vs `worker_module`,成功路徑會 `UnboundLocalError` | 兩個 lint 都跑:mypy 管型別,ruff 管未使用變數 |
| 沙箱 mypy 版本不同 | 沙箱是 2.4.0,Spark 的 CI 不是 | 以 repo 裡 `dev/lint-python --mypy` 的結果為準 |
| 驗證腳本複製了邏輯 | `spec_check.py` 的 `orig()`、`new()` 不是呼叫 `worker_run` | 只能說「這段邏輯行為一樣」;沙箱結果不算證據,要自己跑 |
| `grep -m1 "<version>" pom.xml` | 輸出 `34`(上層 parent pom) | `grep -m1 "SNAPSHOT" pom.xml`,或看 `python/pyspark/version.py` |
| Affects Version 填 4.4.0 | 與同一天的 SPARK-60168(5.0.0)不一致 | 查 master 的版本再填;不確定就照同一天、同一個 master 上建的 ticket |
| JIRA 吃掉底線 | `__spec__` 顯示成 `_spec_` | 存檔後看預覽,不正常再處理;段落不要手動斷行 |
| 測試通過就寫「涵蓋了改動」 | `test_python_datasource` 通過只代表沒壞,不代表走到改動行 | 先弄清楚測試是怎麼被啟動的。**我這次反過來錯了**:從檔案結構推論「沒走到」,沒看 `run-tests.py` / `bin/pyspark` 是用 `python -m <測試模組>` 啟動(見 1.6)。結論要用 break test 或 print 驗證,不要只靠讀程式推論 |
| 把推論寫進 PR 描述 | 「正式環境走 `spec` 分支」沒有證據 | `-m` vs daemon 沒驗證(1.5),描述只寫有證據的 |
| 引用 CPython 研究 | 讀的是 `main` 分支(約 3.15 開發版),我的 venv 是 3.11.17 | 說明沒有逐版比對;想看自己版本:`python -c "import runpy; print(runpy.__file__)"` 然後打開看 |
