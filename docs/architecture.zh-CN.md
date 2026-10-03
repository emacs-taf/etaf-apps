# ETAF 应用集合架构

本文件是应用集合与框架之间已接受的契约。它描述当前实现的系统，不记录任务过程：
已实现的行为以各应用仓库的入口 Commentary 与 `docs/` 为准，本文件只规定跨仓库职责、
依赖方向与可观察的验收标准。

## 定位

本仓库是 ETAF 文本应用的集成根：维护应用清单、共享检查工具、集成检查点与共同约定。
每个应用仍是独立仓库，拥有自己的历史、公共接口、手册和检查，本仓库不复制它们的实现。

## 职责与依赖方向

| 层级 | 仓库 | 职责 |
| --- | --- | --- |
| 应用 | etaf-ncm 及后续应用 | 具体产品的界面、交互与业务模型 |
| 组件 | etaf-ui | 可复用界面组件与主题 token |
| 运行时 | etaf | 组件、状态与挂载运行环境 |
| 布局与渲染 | ebox | 盒模型布局、绘制与增量发布 |
| 选择器语义 | ecss | 选择器与层叠语义 |

依赖方向只允许自上而下：应用依赖 etaf-ui、etaf、ebox 与 ecss；框架侧不得反向依赖
任何应用，也不得因某个应用的需要改变框架的公共接口。应用只通过提供者根入口声明的
公共接口使用它们，未在 Commentary 中列出的符号一律视为内部实现。

## 应用契约

每个应用以本目录下的同名目录独立检出，并满足：

- 根入口 `<名称>.el` 是唯一的包入口，实现放在 `lisp/`，入口 Commentary 是该应用公共
  API 的唯一清单，并在同一处声明 `Function`、`Macro`、`Variable`、`Hook`、`Error`、
  `Component` 记录与 autoload 声明。
- `Makefile` 提供 `compile`、`test`、`check`、`clean`、`structure-check`、`setup-hooks`
  与 `api-check`；`check` 包含结构检查、字节编译、公共 API 边界检查和验收场景。
- 正确性场景集中在 `tests/` 与 `tests/acceptance.json`，只通过公共接口观察行为；
  数据样本放 `tests/fixtures/`，示例与非交互用法放 `examples/`。
- 与外部服务的交互由应用自己的服务层负责，界面与模型只依赖该层的公共接口，因此
  模型可以在无网络、无外部进程的条件下用注入的假实现驱动。
- 应用自有、会增长的持久数据默认经 `etaf-db` 存取：元数据与正文分表，批量写入用
  `etaf-data-mutate-many` 只对账一次；后端权威数据只落本地派生值，远程列表用
  `etaf-list-cache`。存储访问留在应用自己的存储层，界面与模型只依赖其公共接口。

## 提供者解析

框架提供者由各应用在检出时解析，不进入本仓库的版本控制：

- 嵌套检出（本仓库位于 `emacs-taf/etaf-apps` 时）位于上一层：`../../etaf`、
  `../../etaf-ui`、`../../ebox`、`../../ecss`。
- 独立克隆时提供者与应用的顶层目录同级：`../etaf` 等。
  应用 Makefile 因此按 `../etaf/etaf.el` 是否存在来决定使用 `../` 还是 `../../`。

`scripts/api-workspace.json` 记录边界检查所需的模块名、入口与仓库地址；
`python3 scripts/check-api.py --workspace . --fetch-providers` 会补齐缺失的检出。

## 集成检查点

`workspace.json` 记录每个应用的地址、分支与已验证修订。应用的提交先在其仓库内完成并
通过检查，再由本仓库刷新修订；本仓库的 `make check` 在自身结构检查之外逐个运行应用的
`check`，因此它同时覆盖应用的结构、编译、公共 API 边界与验收场景。

## 可观察的验收标准

- `python3 scripts/check-repository.py` 与 `python3 -m unittest discover -s tests/tools -p 'test_*.py'` 通过。
- 每个应用 `make check` 通过，且 `make api-check` 能在批处理 Emacs 中加载其公共入口、验证
  绑定与签名。
- 应用在真实图形 Emacs 中可完成其主要工作流，并由人工确认渲染与交互。
- 应用的服务层在外部服务或外部进程不可用时给出可读的失败结果，而不是无声失败或破坏界面状态。

## 当前状态与缺口

已实现的应用是 etaf-ncm（网易云音乐客户端）、etaf-weread（微信读书客户端）、
etaf-md-preview（Markdown 阅读视图）、etaf-reader（本地电子书阅读器）与 etaf-rss（RSS 阅读器）。etaf-weread 覆盖书架、
笔记本、书籍详情与目录、划线与想法、书城搜索、阅读统计、笔记导出，并通过 etaf-db 建立
本地 SQLite 划线索引供离线检索；etaf-md-preview 把本地 Markdown 渲染为居中阅读栏，
覆盖表格、提示块、代码、脚注与媒体卡片；etaf-reader 扫描本地 EPUB/TXT/Markdown/HTML/
FB2/PDF 文件，经 ebook-parse 技能导入 etaf-db，提供翻页与滚动阅读、鼠标或键盘选中标注与
想法、目录、笔记、书内搜索与按书记录的阅读进度；etaf-rss 订阅 RSS 2.0/Atom/JSON Feed，
本地 JSON 存储加保留策略，提供全部/未读/收藏与按订阅、文件夹的列表、正文阅读、搜索、
OPML 导入导出与后台条件刷新。尚未实现的部分：后续应用清单未确定；应用级性能基准
已在 etaf-rss 的 benchmarks/performance-scenarios.el 登记交互场景，但尚无跨应用门槛。
