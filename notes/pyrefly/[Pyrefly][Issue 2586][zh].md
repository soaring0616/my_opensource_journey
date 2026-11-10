# 實驗紀錄：[Pyrefly][Issue 2586] Widen the synthesized __init__ parameter for fields with a custom pydantic schema

Oct 23, 2026 · @anna


> pyrefly: issue [#2586](https://github.com/facebook/pyrefly/issues/2586)（bad-argument-type on Annotated pydantic fields with custom schema）
> PR: https://github.com/meta/pyrefly/pull/5432（原為 facebook/pyrefly，GitHub 會自動轉址）
> 前一次嘗試：PR [#3798](https://github.com/facebook/pyrefly/pull/3798)（被 rchen152 關閉）
> 狀態：2026-10-28 開 PR → 2026-10-29 收到 review（Request changes）→ 2026-10-30 推 follow-up commit（1f07f21） → 10/31 reviewer approve（Looks good to me），標籤 `waiting-for-merge` 
> 2026.10 版本


## 1. Issue #2586：現象與根因

在 strict 的 pydantic model 裡，`dtype: Annotated[np.dtype, DTypeAdapter()]` 的 `__init__` 參數被 pyrefly 推成 `dtype[Any]`，傳 `">i4"` 就報 `bad-argument-type`。但 `DTypeAdapter` 透過 `__get_pydantic_core_schema__` 提供的是接受任意輸入的 plain validator，`">i4"` 在執行期是合法的。

- 回報者的期待：metadata 實作了 `__get_pydantic_core_schema__` 的欄位，建構子參數應寬鬆處理（接受 `Any`）。
- 維護者 migeed-z 的回應：還沒研究如何支援這個 hook；支援方式大概是放寬 init 參數型別。
- 根因：合成 `__init__` 的 `get_dataclass_init` 在 strict 模式下直接拿欄位宣告型別當參數型別。pyrefly 無從得知自訂 schema hook 會把輸入轉成什麼，照宣告型別檢查就會誤報。
- 為何只在 strict 出現：`as_param`（`class_field.rs:3967-3972`）在非 strict 時本來就把參數型別設成 `Any`。

## 2. PR #3798 與 review

PR 關閉未合併。方向被接受（放寬 init 參數），作法被否決（改 `converter_param`）。

PR 做的事：

- 在 `extract_pydantic_field_from_annotation`（`alt/class/pydantic.rs`）對 `Annotated[...]` 的每個 metadata 項做 `expr_infer`，再用 `has_static_attr(&ty, "__get_pydantic_core_schema__")` 判斷有沒有 hook。
- 偵測到 hook 就把該欄位 `DataclassFieldKeywords.converter_param` 設成 `Any`，間接讓 init 參數變 `Any`。
- 在 `binding/pydantic.rs` 加常數 `GET_PYDANTIC_CORE_SCHEMA`，加一個測試（strict model、`reveal_type(Model.__init__)` 看到 `dtype: Any`、其他欄位仍嚴格）。

rchen152 的 review：

| 意見 | 程式碼依據 | 影響 |
| --- | --- | --- |
| 把 `converter_param` 設成 `Any` 是錯的，它也決定屬性賦值的型別檢查 | `class_field.rs:5615`：`kws.converter_param.unwrap_or(attr_ty)` | 關閉 PR 的主因。`m.dtype = "亂寫"` 會變合法。留下的方向：只放寬 `__init__` 參數 |
| `has_static_attr` 在 metadata 型別是 `Any` 時回 true | `alt/attr.rs:830` | 偵測必須先排除 `Any` |
| 常數放在 alias generator 旁邊很怪（nit） | `binding/pydantic.rs:25-47` 是其他 pydantic 名稱常數 | 搬過去即可 |

## 3. 兩種修法：做法 A 與做法 B

沿用同一個位置，不碰 `converter_param`。有兩個做法：

### 做法 A：把欄位名加進 `pydantic_before_validator_fields`

如果要做，步驟是：

1. 這個集合目前在 `binding/class.rs:263` 由 `extract_field_validator_fields(&body)` 純靠語法算出（找 `@field_validator(..., mode='before'|'plain')`），經 `binding/binding.rs:3692` 存進 `ClassMetadata`（`alt/class/class_metadata.rs:196`）。
2. 偵測 hook 需要推論 metadata 項的型別（`DTypeAdapter()` 是什麼、它的 class 有沒有 `__get_pydantic_core_schema__`），binding 階段沒有型別資訊，做不到。只能改在 `class_metadata.rs` 計算 metadata 時，對每個欄位的 annotation 做推論，再把命中的欄位名併進集合。
3. 繼承要另外處理：`class_metadata.rs:1557-1587` 目前手動合併父類別的集合，新來源也要走這段。

為什麼不選：

- 依賴方向反了。`extract_pydantic_field_from_annotation(annot, name, metadata)` 以 `ClassMetadata` 為輸入；讓 metadata 反過來依賴欄位 annotation 的推論，形成循環（metadata 需要欄位推論，欄位推論需要 metadata）。
- 語意混濁。這個集合的意思是「有 `mode='before'|'plain'` 的 `@field_validator`」。塞進 schema hook 欄位後，名字和內容不符，之後維護的人會誤判。
- 工作量不比 B 少，還多了繼承合併。

### 做法 B（採用）：在 `DataclassFieldKeywords` 加一個 init-only 布林旗標

* `extract_pydantic_field_from_annotation` 偵測到 hook 時設 `true`；`get_dataclass_init` 的條件改成 `before_validator_fields.contains(&name) || field_flags.<flag>`。
    * 不碰 `converter_param`，屬性賦值的檢查維持原樣。
    * 繼承免費：旗標存在宣告該欄位的 class 的 `ClassField` 上，`get_dataclass_member` 透過 `get_field_from_mro`（`class_field.rs:4985`）往上找，`class Child(Parent)` 自動帶到。
    * 旗標型別用 `bool`。`Option<Type>` 可以之後裝更精確的型別，但現在沒這需求。
* 做法 B 要動的地方（由我讀過的程式碼確認）：
    * `crates/pyrefly_types/src/keywords.rs`：加欄位，並更新 `new()`。
    * `dataclass.rs` 的 `dataclass_field_keywords`：那裡是明確列出所有欄位的 struct 字面值，漏填會編譯失敗。
    * `pydantic.rs`：偵測並設旗標。
    * `get_dataclass_init`：讀旗標。
    * `class_field.rs` 的 `AssignedInBody` 路徑：重要，那裡目前從 annotation flags 只取走 `strict`（`.and_then(|flags| flags.strict)`），其餘全丟掉。不處理的話，有預設值的欄位（`x: Annotated[...] = default`）就不會生效。
* 結論：採用做法 B。
    * 共同前提：`get_dataclass_init`（`alt/class/dataclass.rs:1349`）已有一條「init 參數放寬成 `Any`」的現成路徑：`if dataclass.pydantic_before_validator_fields.contains(&name) { converter_param = Some(Any) }`。
    * 這裡的 `converter_param` 是區域變數，只餵給 `as_param`，不寫回欄位的 `DataclassFieldKeywords`，所以不影響屬性賦值。這正是 Rebecca 要的效果。兩種做法都是重用這條路，差別在「這個欄位有沒有命中」這個資訊從哪裡來。



## 4. 做法 B 的改動點與三個補充

改動點共 5 處，其中第 5 處有兩個洞。

| # | 檔案 / 函式 | 改什麼 |
| --- | --- | --- |
| 1 | `crates/pyrefly_types/src/keywords.rs` `DataclassFieldKeywords` | 加布林欄位 `pydantic_has_custom_schema`（初版叫 `init_param_any`，review 後改名）；`new()` 補預設 `false` |
| 2 | `alt/class/dataclass.rs:913` `dataclass_field_keywords` | 結構字面值補新欄位。整個 repo 只有這裡和 `new()` 用字面值建構，漏填會編譯失敗 |
| 3 | `alt/class/pydantic.rs` `extract_pydantic_field_from_annotation` | 對每個 metadata 項找 `__get_pydantic_core_schema__` 是哪個 class 定義的（defining class）：定義在 pydantic 以外就設旗標；定義在 pydantic 內只放行 `BeforeValidator` / `PlainValidator` / `WrapValidator`；metadata 不是 class（含推論成 `Any`）一律不算 |
| 4 | `alt/class/dataclass.rs:1349` `get_dataclass_init` | 條件加 `\|\| field_flags.<flag>` |
| 5 | `alt/class/class_field.rs` `AssignedInBody` 路徑 | 兩處，見補充一 |

### 補充一：`AssignedInBody` 有兩個洞，不是一個

- `class_field.rs:1781-1790`：`annotation_flags` 只保留 `strict`，其他全丟。`x: Annotated[T, Hook()] = 預設值` 會丟旗標。
- `class_field.rs:1845-1850`：右邊是 `= Field(...)` 時，`flags` 來自那個 `Field` 呼叫，只把 `annotation_flags.strict` 合併進去。後面的 `flags.or_else(|| annotation_flags...)`（1936-1942）在 `flags` 已是 `Some` 時不會執行。`x: Annotated[T, Hook()] = Field(default=...)` 也會丟旗標。

兩處都要照 `strict` 的方式一併處理。

### 補充二：偵測規則（after review）

- 初版（`1af9a20`）：排除 `Any` 後，用 `has_static_attr(&ty, "__get_pydantic_core_schema__")` 判斷。
- 問題：pydantic 內建的 `AfterValidator`、`UuidVersion` 等也有這個 hook，會被一起放寬。但它們是在型別驗證之後、或建立在宣告型別的驗證之上，輸入型別仍是宣告型別，不該放寬。
- 定案（`1f07f21`，Reviewer 提的規則）：
    - 看 hook 的 defining class：在 pydantic 以外 → 算；在 pydantic 內 → 不算，除非是 `BeforeValidator` / `PlainValidator` / `WrapValidator`。
    - metadata 不是 class（包含推論成 `Any` 的，例如 `Field(gt=0)`）→ 永遠不算。這取代了初版「排除 `Any`」的特判。
    - 繼承：看 defining class，所以 `class MyUuid(UuidVersion)` 沒覆寫 hook 時仍視為 pydantic 的；覆寫了就算。
- pydantic 官方文件的語意：

| validator | 收到的輸入 | 放寬 init？ |
| --- | --- | --- |
| After | 驗證後的值（型別不變） | 否 |
| Before / Plain / Wrap | 原始輸入（`Any`） | 是 |
| `UuidVersion` | 透過 `handler(source)` 委派給宣告型別，再加限制 | 否 |

- 為什麼 reviewer 選這三個：reviewer 沒有明說。
- `FieldInfo` 沒有這個 hook，`Annotated[int, Field(...)]` 不會誤觸。


### 補充三：旗標只在 strict 模式有作用

`as_param`（`class_field.rs:3967-3972`）在非 strict 時本來就回 `Any`。所以 #2586 只在 strict model 出現，測試也必須用 strict（`ConfigDict(strict=True)` 或 `Field(strict=True)`），否則測試會過但沒驗到東西。

## 5. 測試計畫與現況

放在 `pyrefly/lib/test/pydantic/field.rs`，用 `pydantic_testcase!`。fixture 沒有 `functional_validators.py`，所以 `BeforeValidator` 等類別在測試裡沒有定義。

> 2026-10-31 補註：下表行號是動工前的，加了新欄位和 review 修改後已位移，以函式名稱為準。

- [x] strict + `Annotated[DType, DTypeAdapter()]` 無預設值：`test_annotated_custom_schema_widens_init_param`
- [x] pydantic 內 hook 不放寬（`UuidVersion`）：同一個測試裡的 `Model2`
- [x] 有預設值 `= DType()`：`..._with_default`
- [x] 繼承 `Child(Parent)`：`..._inherited`
- [x] lax 模式、落在轉換表的型別（如 `int` → `LaxInt`）：`..._widens_lax_conversion_table_type`
- [x] 不是 hook 的 metadata（`Field(gt=0)`，推論成 `Any`）：`test_annotated_metadata_without_custom_schema`
- [x] `= Field(default=...)`（補充一的第二個洞）：待對照 field.rs 確認有沒有涵蓋
- [x] 屬性賦值仍報錯（`m.dtype = ">i4"`）：待對照 field.rs 確認
- [x] `BeforeValidator` / `PlainValidator` / `WrapValidator` 三個例外：決定不測。reviewer 認為補最小 stub 是正確作法，但現有覆蓋率已足夠，可省略。已知限制：這三個例外目前沒有測試。

結果：`cargo test -p pyrefly --lib pydantic::field` → 40 passed。

<!-- ## 6. 開工前要問 Rebecca 的事

1. pydantic 內建 validator 類別（After / Before / Plain / Wrap）要不要一起放寬？還是第一版只處理使用者自訂 class？
2. 旗標名稱偏好，例如 `init_accepts_any` 或 `pydantic_custom_schema`。
3. `GET_PYDANTIC_CORE_SCHEMA` 常數放 `binding/pydantic.rs:25-47` 那一群可以嗎？ -->

## 6. 程式碼位置與連結

行號以 facebook/pyrefly main 的 commit `7573d4fd`（2026-10-23）為準。

| 位置 | 內容 |
| --- | --- |
| `alt/class/dataclass.rs:1349` | `get_dataclass_init` 中 before-validator 欄位放寬 init 參數的現成路徑 |
| `alt/class/dataclass.rs:913` | `dataclass_field_keywords` 的結構字面值 |
| `alt/class/pydantic.rs:700` | `extract_pydantic_field_from_annotation` |
| `alt/class/class_field.rs:1697-1709` | `DeclaredByAnnotation` 路徑呼叫上述函式 |
| `alt/class/class_field.rs:1781-1790`、`1845-1850`、`1936-1942` | `AssignedInBody` 路徑的兩個洞 |
| `alt/class/class_field.rs:3967-3972` | `as_param`：非 strict 回 `Any` |
| `alt/class/class_field.rs:4985` | `get_field_from_mro`：繼承自動帶旗標 |
| `alt/class/class_field.rs:5615` | 屬性賦值用 `kws.converter_param` |
| `alt/attr.rs:830` | `has_static_attr` |
| `binding/class.rs:263` | `extract_field_validator_fields`（語法層） |
| `alt/class/class_metadata.rs:1557-1587` | before-validator 集合的繼承合併 |
| `binding/pydantic.rs:25-47` | pydantic 名稱常數 |
| `crates/pyrefly_types/src/keywords.rs:143` | `DataclassFieldKeywords` |

## 7. Review 第一輪（2026-10-28 ~ 10-31）

### 7.1 時間線

| 日期 | 事件 |
| --- | --- |
| 10/28 | 開 PR #5432（`1af9a20`） |
| 10/29 | rchen152 Request changes，兩則 inline 留言；標籤變 `waiting-for-author`；mypy_primer 對開源語料無影響 |
| 10/30 | 推 follow-up（`1f07f21`）；重寫 PR 描述；留言說明並詢問是否補 stub |
| 10/31 | rchen152 approve：「Looks good to me」；不需補 `functional_validators` stub；被誤按 resolve 的 thread 她確認已處理；標籤加上 `waiting-for-merge` |


### 7.2 reviewer 的兩個要求

1. `init_param_any` 改名為 `pydantic_has_custom_schema`。名字應描述欄位的事實，而不是目前 `__init__` 的行為，之後用途變了名字才不用改。
2. pydantic 自己定義的 `__get_pydantic_core_schema__` 不該放寬，除了 `BeforeValidator`、`PlainValidator`、`WrapValidator`。

### 7.3 最終設計

- 旗標只被 `get_dataclass_init` 讀。`get_dataclass_replace` 和屬性賦值仍讀 `converter_param`，所以 `m.dtype = ">i4"` 仍會報錯。
- `AssignedInBody` 路徑兩處都要帶旗標：建 `annotation_flags` 時（過濾條件加上新旗標並複製），以及合併進右邊 `Field(...)` 的 flags 時（`|=`）。
- 偵測規則見 §4 補充二。

### 7.4 驗證

- `cargo test -p pyrefly --lib pydantic::field` → 40 passed。
- 反向驗證：暫時把偵測條件改成 `true ||`，測試報 expected 4、got 3 錯誤。這證明 `Model2`（`UuidVersion`）的排除案例真的在測東西。

### 7.5 踩過的坑

- `//!` 是 inner doc comment，放在程式碼中間會 E0658；要寫 `// !` 或直接刪掉舊行。
- `# E:` 的 `#` 後面要有空格；比對是子字串（backtick 要一致）；預期錯誤數必須等於實際數。
- 在 zsh 貼指令：`*.rs` 要加引號或改用 `git grep`；貼上的 `#` 註解會被當成指令。
- `git add` 之後又改檔，要再 add，並用 `git diff --cached` 確認。
- 換掉 `dataclass.rs` 的舊註解時，不小心刪掉原本對 `@field_validator(mode='before'|'plain')` 的說明，後來補回。
- 完整 `cargo test` 會卡在 `deep_phi_chain_term_expr`（main 上本來就這樣），要用 `--skip`。
- macOS 要設 `SDKROOT`；linker 的 `__eh_frame` 訊息和 `cargo fmt` 的 nightly 選項警告都無害。
- 不小心按了 `Resolve conversation`：已留言請 reviewer reopen。
- 比較兩個 base 不同的 branch 時，`git diff A B` 會混入 upstream 的變動；要看某個 branch 自己的改動，用 `git diff $(git merge-base A B) B`。

### 7.6 待辦與未決

- [x] reviewer 回覆：approve。
- [x] 是否補 stub：不需要。
- [x] 誤按 resolve 的 thread：不用 reopen。
- [ ] 等 `waiting-for-merge` → merged。沒有新要求就不再推 commit。
- [ ] CI / workflows 狀態：截圖沒顯示，之後到 Checks 頁確認。
- [ ] `__replace__`：`get_dataclass_replace` 也讀 `converter_param`，這次沒動；之後另開 follow-up，並把 maintainer 的留言連結補進去。
- [x] 合併後清理本機 `stash@{0}`、`backup-2586`、`pr-3798`。

### 7.7 AI 揭露

PR 描述與 commit message 有 Claude 協助潤飾，已在描述中註明；對 reviewer 的回覆用自己的話寫。

---

## Q1: `model_config = ConfigDict(strict=True)` 這一行作用是什麼

這一行把這個 model 的**驗證模式設成 strict（嚴格模式）**。

**pydantic 執行時的意思**

`model_config` 是 pydantic 保留的類別屬性，用來放 model 的設定；`ConfigDict` 是給它做型別檢查的 TypedDict。`strict=True` 的效果是**不做型別轉換**：

```python
from pydantic import BaseModel, ConfigDict

class Lax(BaseModel):
    n: int

class Strict(BaseModel):
    model_config = ConfigDict(strict=True)
    n: int

Lax(n="1")     # 通過，"1" 被轉成 1
Strict(n="1")  # ValidationError，str 不能當 int
```

pydantic 另外也允許寫成 `class Model(BaseModel, strict=True)`。

- 已確認：pyrefly 在 `binding/pydantic.rs` 的 `PydanticConfigDict` 處理 `model_config = ConfigDict(...)` 或 dict 字面值。
- 未驗證：pyrefly 有沒有讀 class keyword 這種寫法。查法：`git grep -n 'BaseModel, strict=True' -- pyrefly/lib/test/pydantic`。

**pyrefly 怎麼決定 `__init__` 的參數型別（讀程式碼確認）**

`as_param`（`class_field.rs`）依序判斷：

1. 有 `converter_param`：用它，不管 strict。
2. 非 strict：`Any`。
3. 有 descriptor setter：用 setter 型別。
4. 其餘：欄位宣告型別。

另外 `get_dataclass_init` 只在非 strict 時才查 lax 轉換表（例如 `int` 展開成 `LaxInt`）。我們的旗標是在這裡把區域變數 `converter_param` 設成 `Some(Any)`，所以排在第 1 順位，連 lax 轉換表也會被它蓋掉。

**對測試與 PR 範圍的意義**

- strict：參數型別是宣告型別，所以誤報看得到，測試的負面斷言（`other: DType` 傳 `">i4"` 要報錯）才有意義。大部分測試因此用 `strict=True`，等於把 lax 的放寬關掉，單獨觀察「放寬是不是 custom schema 造成的」。
- lax、不在轉換表的型別（`np.dtype`、`DType`）：參數本來就是 `Any`。這就是 issue 原始範例在 main 的 lax 模式已經不報錯的原因（你 10/24 在 main 上確認過，見 Q2）。
- lax、在轉換表的型別（`int` → `LaxInt`）：沒有旗標時仍會誤報，有旗標則變 `Any`。這是唯一刻意不用 strict 的測試 `..._widens_lax_conversion_table_type`，它的預期輸出是 `n: Any, m: LaxInt`。