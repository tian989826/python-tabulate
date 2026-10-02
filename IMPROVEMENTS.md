# python-tabulate 改进报告

## 项目简介与选择理由

**tabulate**（https://github.com/astanin/python-tabulate）是一个把列表、字典、
NumPy 数组、pandas DataFrame 等二维数据美化打印为纯文本表格的 Python 库，
支持 30 种输出格式（grid、pipe、GitHub Markdown、HTML、LaTeX 等），并提供
命令行工具。

选择理由：

- **规模适中**：核心为单模块约 2900 行，加 CLI 共约 3100 行，可完整通读；
- **结构清晰**：格式定义（`TableFormat` 声明表）与渲染管线（类型推断 →
  格式化 → 对齐 → 拼表）分层明确，易于安全地修改；
- **质量基线好**：自带 365 个 pytest 用例 + 16 个 doctest + benchmark 脚本，
  每处改进都能验证「不破坏原有行为」。

基线：`365 passed, 1 skipped`（Python 3.13）。

## 改进清单

### 1. Bug 修复：`_expand_iterable()` 遇到 tuple 输入直接崩溃

- **现象**：`tabulate(rows, headers=hs, maxheadercolwidths=(5, 6))` 或
  `rowalign=("top", "bottom")` 抛出
  `TypeError: can only concatenate tuple (not "list") to tuple`。
  而同类参数 `maxcolwidths` 在调用处单独做了 tuple 兼容——属于漏修，
  根因在 `_expand_iterable()` 内部直接对 `original + [default] * n` 做拼接。
- **修改**：`tabulate/__init__.py` 的 `_expand_iterable()` 中先
  `list(original)` 再拼接，并补注释说明原因。
- **预期效果**：所有经过该函数的参数（`maxcolwidths`、`maxheadercolwidths`、
  `rowalign`、内部 `numparses`）统一接受任意非字符串序列，行为与 list 一致。
- **验证**：新增 `test_internal.py::test_expand_iterable_with_tuple` 及
  `test_regression.py` 中 3 个回归用例（tuple 与 list 输出逐一相等）。

### 2. 性能：`_more_generic()` 的类型字典提升为模块级常量

- **问题**：列类型推断用 `reduce(_more_generic, types)`，即每个单元格调用
  一次；而原实现每次调用都重建 `types`、`invtypes` 两个字典。
- **修改**：两个字典提升为模块级 `_type_ranks` / `_rank_types`，逻辑不变。
- **预期效果**：消除每单元格两次字典构造，大表类型推断更快。

### 3. 性能：padding 函数改用 `str.rjust` / `str.ljust`

- **问题**：`_padleft` / `_padright` 每个单元格都临时拼
  `f"{{0:>{width}s}}"` 再走 `.format()` 机制。
- **修改**：改为等价的 `s.rjust(width)` / `s.ljust(width)`。
  **注意**：`_padboth`（居中）不能换成 `str.center`——奇数留白时
  `str.center` 把多余空格放左侧而 `format` 的 `^` 放右侧，行为不等价
  （曾被 6 个 pretty 格式测试当场抓住），故保留原实现并加注释说明。
- **预期效果**：左右对齐路径省去 format 字符串构造与解析开销。

### 4. 性能/内存：ANSI/多行检测改为分批扫描 + 提前退出

- **问题**：`tabulate()` 为检测 ANSI 转义码和多行单元格，先把整张表
  拼成一个巨大的 `\t` 分隔字符串（等于把全部文本复制一份），再整体扫两遍。
  大表下既费内存又是全量扫描。
- **修改**：改为逐批取 256 个单元格、批内 `\t` 拼接后扫描（与整表拼接的
  语义完全一致），一旦 `has_invisible` 与 `is_multiline` 两个属性都确定
  立即提前退出；不需要多行检测的格式只需确定 ANSI 一项即可退出。
- **预期效果**：内存占用从「整表文本的一份额外拷贝」降为「有界的 256 单元格
  批次」；含 ANSI 码的表在首个批次后即可停止扫描。

### 5. 文档：`tabulate()` docstring 修正与补全

- 格式列表过时：原文只列 11 种且把 `tsv` 写成无引号的 tsv，实际支持
  30 种，已补全为完整列表；
- 修复笔误：`` `tabulate_formats`contains ``（缺空格）、注释中的
  `conversino`（随第 4 项重写）、`charcter's`；
- `rowalign` 参数此前完全没有文档，补充说明其取值
  （`"top"`/`"center"`/`"bottom"` 或逐行列表）及仅对多行单元格生效。

### 6. 工程：CHANGELOG 与回归测试

- 按项目惯例在 `CHANGELOG` 顶部新增 Unreleased 条目，记录本次改动；
- 新增 4 个测试（1 个内部单元测试 + 3 个回归测试），全部通过。

## 验证结果

| 项目 | 改前 | 改后 |
|---|---|---|
| pytest 套件 | 365 passed, 1 skipped | **369 passed, 1 skipped** |
| doctest（`--doctest-modules`） | 16 passed | 16 passed |
| flake8（max-line-length=100） | 7 条既有告警 | 7 条（**零新增**） |
| 大表渲染（10000 行 × 10 列） | 0.233 s | **0.198 s（约 -15%）** |
| 首行含 ANSI 码大表（grid） | 0.294 s | **0.255 s（约 -13%）** |
| 新旧版本输出逐字节对比 | — | **完全一致** |

测试环境：Windows 11 / Python 3.13.14 / wcwidth 已安装。
