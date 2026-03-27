# Tree-sitter 架构文档

> 基于 Tree-sitter v0.27.0 源码分析

---

## 1. 项目概述

Tree-sitter 既是**解析器生成工具**，也是**增量解析库**。它能为源文件构建具体语法树（CST），
并在编辑源文件时高效地增量更新语法树。

**设计目标**：
- **通用**：适用于多种编程语言
- **快速**：编辑器逐键解析级别的性能
- **容错**：语法错误下仍能产出有用的语法树
- **轻量**：C 运行时库无重依赖，可嵌入任何应用

---

## 2. 完整目录树与职责标注

```
tree-sitter/
├── lib/                            # 【核心】C 运行时库 + 语言绑定
│   ├── include/
│   │   └── tree_sitter/
│   │       └── api.h               # 公开 C API（TSParser, TSTree, TSNode 等全部对外类型与函数）
│   ├── src/                        # C 运行时实现
│   │   ├── parser.c                # 解析器核心：GLR 状态机驱动、shift/reduce、错误恢复
│   │   ├── lexer.c                 # 词法器：字符读取、UTF 解码、included ranges、token 识别
│   │   ├── subtree.c               # 语法树节点：inline/堆分配、引用计数、合并统计
│   │   ├── tree.c                  # TSTree 生命周期、edit、get_changed_ranges
│   │   ├── node.c                  # TSNode 遍历：子节点、field、兄弟、祖先查询
│   │   ├── stack.c                 # GLR 多版本解析栈：StackNode 链表/池、push/pop/merge
│   │   ├── tree_cursor.c           # TSTreeCursor 栈式深度遍历
│   │   ├── language.c              # TSLanguage 查询接口与表查找
│   │   ├── query.c                 # S-表达式查询编译与树匹配执行
│   │   ├── get_changed_ranges.c    # 新旧树对比，计算变化范围
│   │   ├── alloc.c                 # 内存分配器抽象（可替换 malloc/free）
│   │   ├── wasm_store.c            # WASM 语言支持（通过 wasmtime）
│   │   ├── lib.c                   # 单编译单元聚合（#include 所有 .c）
│   │   ├── parser.h                # TSLanguage 完整定义、解析动作、词法模式等内部类型
│   │   ├── lexer.h                 # Lexer 内部结构
│   │   ├── subtree.h               # Subtree tagged union（inline/heap）+ SubtreePool
│   │   ├── tree.h                  # TSTree 内部结构（root Subtree + language + ranges）
│   │   ├── tree_cursor.h           # TreeCursor 内部结构（entry 栈）
│   │   ├── stack.h                 # Stack 不透明接口 + StackSlice/StackSummary
│   │   ├── language.h              # TableEntry + LookaheadIterator 内部实现
│   │   ├── array.h                 # 通用动态数组宏（Array(T)）
│   │   ├── alloc.h                 # 全局分配函数指针 + ts_malloc 等宏
│   │   ├── length.h                # Length 类型（字节 + 二维 Point）
│   │   ├── reusable_node.h         # 增量解析时沿旧树 walking 的复用节点
│   │   ├── reduce_action.h         # ReduceAction/ReduceActionSet
│   │   ├── error_costs.h           # 错误恢复启发式代价常数
│   │   ├── get_changed_ranges.h    # TSRangeArray 类型
│   │   ├── point.h / point.c       # TSPoint 运算
│   │   ├── atomic.h                # 跨平台原子操作抽象
│   │   ├── host.h                  # 平台差异抽象
│   │   ├── unicode.h               # Unicode 分类/解码
│   │   ├── unicode/                # ICU Unicode 表（UTF-8/16 解码）
│   │   ├── wasm/                   # WASM stdlib 符号表
│   │   └── portable/endian.h       # 字节序抽象
│   ├── binding_rust/               # Rust 绑定（安全封装 C FFI）
│   │   ├── lib.rs                  # Parser, Tree, Node, Language 等 Rust 类型
│   │   ├── ffi.rs                  # bindgen 生成的 C FFI 声明
│   │   ├── bindings.rs             # 自动生成的原始绑定
│   │   ├── wasm_language.rs        # WASM 语言支持
│   │   ├── util.rs                 # 工具函数
│   │   └── build.rs               # 构建脚本（编译 C 源码 + 可选 bindgen）
│   ├── binding_web/                # Web/WASM 绑定（TypeScript + Emscripten）
│   │   ├── src/                    # TypeScript 封装类（parser.ts, tree.ts, node.ts 等）
│   │   ├── lib/                    # WASM 模块加载与导出
│   │   └── test/                   # Web binding 测试
│   ├── Cargo.toml                  # Rust crate 配置（tree-sitter）
│   └── tree-sitter.pc.in           # pkg-config 模板
│
├── crates/                         # 【工具链】Rust crate 集合
│   ├── generate/                   # 语法生成器：grammar.js → C parser
│   │   └── src/
│   │       ├── generate.rs         # 生成流程主入口
│   │       ├── parse_grammar.rs    # JSON → InputGrammar
│   │       ├── grammars.rs         # 语法数据结构定义
│   │       ├── rules.rs            # Rule AST 定义
│   │       ├── tables.rs           # ParseTable/LexTable 数据结构
│   │       ├── nfa.rs              # NFA 实现（词法状态机构建）
│   │       ├── render.rs           # C 代码渲染（生成 parser.c）
│   │       ├── quickjs.rs          # 内嵌 QuickJS 运行 grammar.js
│   │       ├── prepare_grammar/    # 语法预处理管道
│   │       │   ├── intern_symbols.rs       # 符号名 → 符号 ID
│   │       │   ├── extract_tokens.rs       # 分离句法规则与词法规则
│   │       │   ├── expand_repeats.rs       # 展开 repeat 为辅助非终结符
│   │       │   ├── flatten_grammar.rs      # 压平为 Production 列表
│   │       │   ├── expand_tokens.rs        # 正则 → NFA 构建
│   │       │   ├── extract_default_aliases.rs  # 提取全局默认 alias
│   │       │   └── process_inlines.rs      # 构建 inline 展开映射
│   │       └── build_tables/       # 解析表构建
│   │           ├── build_parse_table.rs    # LR(1) 项目集构造 + 冲突消解
│   │           ├── build_lex_table.rs      # NFA → 词法状态表
│   │           ├── item.rs                 # LR Item 定义
│   │           ├── item_set_builder.rs     # FIRST/LAST 集 + 闭包
│   │           ├── minimize_parse_table.rs # 状态合并优化
│   │           ├── token_conflicts.rs      # Token 冲突矩阵
│   │           └── coincident_tokens.rs    # 同状态共现 token 索引
│   ├── cli/                        # 命令行工具（tree-sitter CLI）
│   │   └── src/
│   │       ├── main.rs             # CLI 入口（clap 子命令）
│   │       ├── parse.rs            # parse 命令：解析文件输出 CST
│   │       ├── highlight.rs        # highlight 命令：语法高亮
│   │       ├── query.rs            # query 命令：S-表达式查询
│   │       ├── tags.rs             # tags 命令：符号标签提取
│   │       ├── test.rs             # test 命令：运行语法测试
│   │       ├── init.rs             # init 命令：初始化语法项目
│   │       ├── wasm.rs             # WASM 编译支持
│   │       ├── fuzz/               # 模糊测试
│   │       ├── tests/              # 集成测试
│   │       └── templates/          # 项目模板
│   ├── highlight/                  # 语法高亮库
│   │   └── src/highlight.rs        # Highlighter + HighlightConfiguration
│   ├── tags/                       # 代码标签提取库
│   │   └── src/tags.rs             # TagsContext + TagsConfiguration
│   ├── loader/                     # 语言动态加载器
│   │   └── src/loader.rs           # 编译/加载 .so + 懒加载配置
│   ├── language/                   # 语言函数指针抽象（no_std）
│   │   └── src/language.rs         # LanguageFn 类型
│   ├── config/                     # 用户配置管理
│   │   └── src/tree_sitter_config.rs  # Config 读写（JSON + XDG）
│   └── xtask/                      # 开发任务（benchmark, fetch 等）
│
├── docs/                           # 项目文档（mdBook）
│   └── src/
│       ├── index.md                # 项目介绍
│       ├── creating-parsers/       # 创建解析器指南
│       ├── using-parsers/          # 使用解析器指南
│       ├── cli/                    # CLI 命令文档
│       └── 5-implementation.md     # 实现细节
│
├── test/                           # 测试数据
│   └── fixtures/
│       ├── grammars/               # 真实语言语法（集成测试用）
│       ├── test_grammars/          # 小型专用测试语法
│       ├── error_corpus/           # 错误语料
│       └── template_corpus/        # 模板语料
│
├── Cargo.toml                      # Workspace 根配置
├── CMakeLists.txt                  # C 库 CMake 构建
├── Makefile                        # C 库直编 + 开发目标
├── Package.swift                   # Swift 包集成
├── build.zig / build.zig.zon       # Zig 构建
├── flake.nix / flake.lock          # Nix flake
└── Dockerfile                      # 容器构建
```

---

## 3. 核心子系统识别与层次结构

### 架构层次图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        应用层 (Application)                         │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  CLI (crates/cli)                                            │   │
│  │  ┌──────┐ ┌──────┐ ┌───────┐ ┌──────┐ ┌──────┐ ┌────────┐  │   │
│  │  │parse │ │test  │ │generate│ │query │ │fuzz  │ │  ...   │  │   │
│  │  └──────┘ └──────┘ └───────┘ └──────┘ └──────┘ └────────┘  │   │
│  └──────────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│                       能力层 (Capabilities)                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │  Highlight   │  │    Tags      │  │       Loader             │  │
│  │ (高亮引擎)   │  │ (标签提取)   │  │ (语言编译/加载/配置)     │  │
│  └──────┬───────┘  └──────┬───────┘  └────────────┬─────────────┘  │
│         │                 │                       │                 │
├─────────┼─────────────────┼───────────────────────┼─────────────────┤
│         │        核心层 (Core Runtime)            │                 │
│  ┌──────┴─────────────────┴───────────────────────┴──────────────┐  │
│  │               tree-sitter (lib/)  — C 实现                    │  │
│  │  ┌─────────┐ ┌────────┐ ┌───────┐ ┌───────┐ ┌─────────────┐ │  │
│  │  │ Parser  │ │ Lexer  │ │ Stack │ │Subtree│ │   Query     │ │  │
│  │  │(parser.c)│(lexer.c)│(stack.c)│(subtree.c)│ (query.c)   │ │  │
│  │  └────┬────┘ └───┬────┘ └───┬───┘ └───┬───┘ └──────┬──────┘ │  │
│  │       │          │          │         │             │        │  │
│  │  ┌────┴──────────┴──────────┴─────────┴─────────────┘        │  │
│  │  │  Tree / Node / TreeCursor / Language / ChangedRanges      │  │
│  │  └──────────────────────────────────────────────────────────  │  │
│  │  ┌───────────────────────────────────────────────────────┐   │  │
│  │  │  基础设施: Array / Alloc / Length / Atomic / Unicode   │   │  │
│  │  └───────────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────────┤
│                    生成层 (Generator — 编译时)                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Generate (crates/generate)                                  │  │
│  │  grammar.js ──→ parse_grammar ──→ prepare_grammar            │  │
│  │                                     │                        │  │
│  │                            ┌────────┴────────┐               │  │
│  │                            ▼                 ▼               │  │
│  │                     SyntaxGrammar      LexicalGrammar        │  │
│  │                            │                 │               │  │
│  │                            ▼                 ▼               │  │
│  │                     build_parse_table   build_lex_table       │  │
│  │                            │                 │               │  │
│  │                            └────────┬────────┘               │  │
│  │                                     ▼                        │  │
│  │                               render (C code)                │  │
│  │                                     │                        │  │
│  │                                     ▼                        │  │
│  │                              parser.c (生成)                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────────┤
│                      绑定层 (Bindings)                             │
│  ┌────────────┐  ┌─────────────────┐  ┌───────────────────────┐   │
│  │ Rust FFI   │  │ Web/WASM (TS)   │  │  LanguageFn (no_std)  │   │
│  │(binding_rust)│ │ (binding_web)   │  │  (crates/language)    │   │
│  └────────────┘  └─────────────────┘  └───────────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│                      横切关注点 (Cross-cutting)                    │
│  ┌────────────────┐  ┌──────────────────────────────────────────┐  │
│  │  Config        │  │  xtask (开发任务: bench, fetch, bump)    │  │
│  │ (用户配置)     │  │                                          │  │
│  └────────────────┘  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 各子系统一览

| 子系统 | 职责 | 对应文件 | 对外接口 |
|--------|------|----------|----------|
| **Parser** | GLR 状态机驱动解析，shift/reduce 决策，错误恢复 | `lib/src/parser.c` | `ts_parser_new/delete/parse/reset` |
| **Lexer** | 字符读取、UTF 解码、token 识别、included ranges | `lib/src/lexer.c` | 内部 `Lexer` 结构，通过 `TSLexer` 回调接口 |
| **Stack** | GLR 多版本解析栈管理、分支/合并 | `lib/src/stack.c` | 内部 `Stack` 不透明类型 |
| **Subtree** | 语法树节点存储（inline/堆）、引用计数、pool | `lib/src/subtree.c` | 内部 `Subtree/MutableSubtree` |
| **Tree** | TSTree 生命周期、edit 操作 | `lib/src/tree.c` | `ts_tree_copy/delete/edit/root_node` |
| **Node** | 节点遍历与属性查询 | `lib/src/node.c` | `ts_node_*` 系列函数 |
| **TreeCursor** | 栈式高效树遍历 | `lib/src/tree_cursor.c` | `ts_tree_cursor_*` 系列函数 |
| **Query** | S-表达式模式匹配 | `lib/src/query.c` | `ts_query_new/delete`, `ts_query_cursor_*` |
| **Language** | 语言元数据与解析表查询 | `lib/src/language.c` | `ts_language_*` 系列函数 |
| **ChangedRanges** | 新旧树差异计算 | `lib/src/get_changed_ranges.c` | `ts_tree_get_changed_ranges` |
| **Generate** | grammar.js → C parser 完整管道 | `crates/generate/` | `generate_parser_in_directory` |
| **Highlight** | 基于查询的语法高亮 | `crates/highlight/` | `Highlighter`, `HighlightConfiguration` |
| **Tags** | 代码符号标签提取 | `crates/tags/` | `TagsContext`, `TagsConfiguration` |
| **Loader** | 语言动态编译与加载 | `crates/loader/` | `Loader`, `LanguageConfiguration` |
| **CLI** | 命令行开发工具 | `crates/cli/` | `tree-sitter` 二进制 |
| **Config** | 用户配置管理 | `crates/config/` | `Config` |
| **LanguageFn** | 语言入口函数指针抽象 | `crates/language/` | `LanguageFn` |

---

## 4. 子系统依赖关系与边界

### Crate 级依赖图

```
                         ┌──────────┐
                         │   CLI    │
                         └────┬─────┘
              ┌───────┬───────┼───────┬──────────┐
              ▼       ▼       ▼       ▼          ▼
         ┌────────┐┌──────┐┌──────┐┌──────┐┌─────────┐
         │Generate││Loader││ High ││ Tags ││ Config  │
         └────────┘└──┬───┘│ light│└──┬───┘└─────────┘
                      │    └──┬───┘   │
                      │       │       │
                      ▼       ▼       ▼
                   ┌──────────────────────┐
                   │  tree-sitter (lib)   │
                   │    C core + Rust FFI │
                   └──────────┬───────────┘
                              │
                              ▼
                   ┌──────────────────────┐
                   │  tree-sitter-language │
                   │    (LanguageFn)       │
                   └──────────────────────┘
```

### C 运行时内部依赖图

```
                    api.h（对外声明层）
                         ↑
    ┌────────────────────┼────────────────────┐
    │                    │                    │
 parser.c            tree.c / node.c     query.c
    │                    │                    │
    ├── lexer.c          ├── subtree.c        ├── tree_cursor.c
    ├── stack.c          ├── tree_cursor.c    ├── language.c
    ├── subtree.c        └── get_changed      └── array / alloc
    ├── language.c           _ranges.c
    ├── reusable_node.h
    └── reduce_action.h

get_changed_ranges.c ──→ subtree + tree_cursor + language

所有模块 ──→ array.h / alloc.h / length.h（基础设施）
```

### 边界分析

| 边界 | 描述 |
|------|------|
| **C API 边界** (`api.h`) | 所有对外交互必须通过 `api.h` 定义的类型和函数，内部头文件不对外暴露 |
| **Runtime / Generator 分离** | C 运行时与 Rust 生成器完全解耦：生成器产出标准 C 文件，运行时只需 `TSLanguage` 结构 |
| **Rust FFI 边界** | `binding_rust/ffi.rs` 定义了 C→Rust 的类型映射，`lib.rs` 提供安全封装 |
| **WASM 边界** | `binding_web` 通过 Emscripten 编译的 WASM 模块交互，用内存缓冲区传递数据 |
| **Query DSL 边界** | `query.c` 接受 S-表达式字符串，独立于解析过程 |
| **Highlight/Tags 边界** | 上层能力通过 `Query` + `Tree` API 构建，不直接访问解析器内部 |

---

## 5. 架构风格分析

Tree-sitter 采用的是 **混合架构风格**，综合了三种模式：

### 5.1 分层架构（Layered Architecture）— 主体

整体呈现清晰的分层：

```
应用层:     CLI
能力层:     Highlight / Tags / Loader
核心层:     Parser / Lexer / Tree / Query (C runtime)
基础设施层:  Array / Alloc / Length / Unicode
```

上层只依赖下层的接口，不跨层直接访问。`api.h` 是核心层的唯一对外边界。

### 5.2 管道架构（Pipeline Architecture）— 生成流程

Grammar 生成器是典型的**编译器管道**：

```
grammar.js → JSON → InputGrammar → [intern → extract → expand → flatten]
           → SyntaxGrammar + LexicalGrammar → [build_parse_table + build_lex_table]
           → Tables → render → parser.c
```

每个阶段有明确的输入/输出数据结构，数据单向流动。

### 5.3 表驱动架构（Table-Driven Architecture）— 运行时

运行时解析器本质上是**表驱动状态机**：
- `TSLanguage` 包含所有静态表（parse_table, lex_modes, symbol_names 等）
- Parser 运行时只做查表 + 状态转移，不含语言特定逻辑
- 这是 Runtime 与 Grammar 解耦的关键

### 结论

> Tree-sitter 的架构核心是 **"分层 + 管道 + 表驱动"** 的混合体。
> 分层保证了关注点分离，管道实现了生成流程的清晰阶段划分，
> 表驱动实现了运行时与语法定义的完全解耦。

---

## 6. 关键架构设计决策

### 决策 1：Runtime 与 Grammar 完全分离

**描述**：C 运行时库与语法定义完全解耦。生成器产出包含 `TSLanguage` 结构的独立 C 文件，
运行时通过查表驱动解析，不包含任何语言特定代码。

**代码位置**：
- `TSLanguage` 结构定义：`lib/src/parser.h`（含 parse_table, lex_fn, symbol_names 等全部静态数据）
- 运行时查表：`lib/src/language.c` → `ts_language_table_entry()`
- 生成器产出：`crates/generate/src/render.rs` → 渲染为 C 代码的 `TSLanguage` 实例

**价值**：一个运行时库支持所有语言；语法可独立发布为库；更新运行时不需要重新生成语法。

### 决策 2：GLR 解析 + 动态优先级消歧

**描述**：采用 GLR（Generalized LR）解析而非传统 LR(1)，遇到冲突时并行维护多个解析分支，
通过动态优先级在解析过程中消歧。

**代码位置**：
- 多版本栈：`lib/src/stack.c`（`StackVersion`、`ts_stack_split`）
- 分支合并：`lib/src/parser.c`（`ts_parser__condense_stack`）
- 动态优先级：`lib/src/parser.c` + `crates/generate/src/build_tables/build_parse_table.rs`

**价值**：能处理真实编程语言中的歧义和上下文相关语法，同时保持 LR 级性能。

### 决策 3：增量解析

**描述**：编辑操作后不重新解析整个文件，而是复用旧语法树中未受影响的子树，
只重新解析发生变化的区域。

**代码位置**：
- 树编辑：`lib/src/tree.c` → `ts_tree_edit()`
- 节点复用：`lib/src/reusable_node.h`（`ReusableNode` 沿旧树 walking）
- 变化范围计算：`lib/src/get_changed_ranges.c`
- 复用判断：`lib/src/parser.c`（`ts_parser__reuse_node` 相关逻辑）

**价值**：编辑器场景下实现亚毫秒级增量更新，核心竞争力。

### 决策 4：Subtree 的 Inline/Heap 双模式存储

**描述**：语法树节点使用 tagged union，小节点（叶子）内联存储在指针大小的空间内，
大节点（含子节点的）在堆上分配并引用计数。

**代码位置**：
- `lib/src/subtree.h`：`SubtreeInlineData`（位域打包）+ `SubtreeHeapData`（堆分配）
- `Subtree` union：`data`（inline）或 `ptr`（堆指针），由 `is_inline` 位区分
- `SubtreePool`：`lib/src/subtree.c`（可复用节点池）

**价值**：减少堆分配次数，提升缓存局部性，降低内存占用。

### 决策 5：词法与句法的编译时强耦合

**描述**：在生成阶段，词法和句法紧密关联——根据每个 parse state 允许的 token 集合
构建不同的 lex state，而非像 Flex/Bison 那样独立生成。

**代码位置**：
- Token 冲突矩阵：`crates/generate/src/build_tables/token_conflicts.rs`
- 同状态共现索引：`crates/generate/src/build_tables/coincident_tokens.rs`
- 词法表构建：`crates/generate/src/build_tables/build_lex_table.rs`（按 parse state 合并 lex state）

**价值**：利用解析上下文减少词法歧义，支持上下文相关的 token 识别（如 JSX 中 `<` 的歧义）。

### 决策 6：External Scanner 扩展机制

**描述**：生成的解析器通过 `external_scanner` 回调机制支持手写 C/C++ 扫描器，
处理缩进敏感、字符串插值等正则/CFG 无法表达的词法规则。

**代码位置**：
- `TSLanguage.external_scanner`：`lib/src/parser.h`（含 create/destroy/serialize/deserialize/scan 函数指针）
- 运行时调用：`lib/src/parser.c`（external token 处理）
- DSL 声明：`crates/generate/src/grammars.rs`（`externals` 字段）

**价值**：在表驱动的通用架构下保留了处理特殊词法需求的灵活性。

### 决策 7：单编译单元聚合（Amalgamation）

**描述**：`lib/src/lib.c` 通过 `#include` 将所有 `.c` 文件聚合为单个编译单元，
方便嵌入式使用和跨模块优化。

**代码位置**：`lib/src/lib.c`

**价值**：简化集成（只需包含一个 .c 文件）；编译器可做全局优化（内联、消除冗余等）。

---

## 7. 与同类项目对比

### 对比维度表

| 维度 | Tree-sitter | ANTLR | Flex/Bison |
|------|-------------|-------|------------|
| **解析算法** | GLR（表驱动 + 多分支并行） | LL(*), ALL(*) | LALR(1) |
| **词法方案** | 与句法耦合的表驱动词法 | 内建词法（也可 LL 模式） | Flex 独立 DFA 词法 |
| **语法定义语言** | JavaScript DSL (grammar.js) | ANTLR 语法文件 (.g4) | BNF 变体 (.y/.l) |
| **运行时语言** | C（极小依赖） | Java/C#/Python/... | C |
| **增量解析** | ✅ 核心特性 | ❌ | ❌ |
| **错误恢复** | 内建启发式恢复 | 内建 | 需手写规则 |
| **CST vs AST** | 具体语法树（CST） | 解析树 + 访问者模式 | 用户自定义语义动作 |
| **编辑器集成** | 核心设计目标 | 非主要目标 | 非目标 |
| **容错解析** | ✅（ERROR 节点保留上下文） | ✅（但非核心优化方向） | ❌ |
| **语言绑定** | C/Rust/WASM/Node.js/Swift/Zig | Java 为主 | 仅 C |

### Tree-sitter 的独特性

**1. 增量解析是一等公民**

这是 Tree-sitter 与其他工具最本质的区别。ANTLR 和 Bison 面向"整文件解析"场景设计，
而 Tree-sitter 从底层数据结构（Subtree 复用、ReusableNode walking、ChangedRanges 计算）
到 API（`ts_tree_edit` + 重新 `parse`）都围绕增量场景优化。

**2. 编辑器原生设计**

- 输入通过回调函数提供（`TSInput`），支持 rope 等编辑器数据结构
- 解析可随时取消（`TSParseOptions.progress_callback`）
- 包含范围过滤（`included_ranges`）支持嵌入语言
- 高亮/标签/查询作为一等能力提供

**3. 运行时与语法的彻底解耦**

ANTLR 生成的解析器是独立程序（Java/C#类），需要对应语言的 ANTLR 运行时。
Bison 生成的解析器通常与用户的语义动作代码混合编译。
Tree-sitter 生成纯数据（`TSLanguage` 表结构），加载进通用运行时即可工作。

**4. GLR + 上下文感知词法**

传统工具通常将词法和句法完全分离（Flex 产出 token 流，Bison 消费）。
Tree-sitter 在生成阶段利用 parse state 约束 lex state，
使词法器能根据解析上下文做出不同的 token 化决策。

**5. 具体语法树（CST）而非抽象语法树（AST）**

Tree-sitter 保留源码中的所有信息（包括空白、注释位置），
产出的是 CST 而非 AST。上层工具通过 Query 按需提取所需信息。
ANTLR 也产出解析树但通常通过 Visitor/Listener 转化为 AST；Bison 直接在动作中构建 AST。

---

## 8. 架构亮点总结

| # | 亮点 | 核心价值 |
|---|------|----------|
| 1 | Runtime/Grammar 分离（表驱动） | 一个运行时支持所有语言 |
| 2 | 增量解析（Subtree 复用） | 编辑器级实时性能 |
| 3 | GLR 多版本并行解析 | 处理真实语言的歧义 |
| 4 | Inline/Heap 双模式 Subtree | 内存效率与缓存友好 |
| 5 | 上下文感知词法生成 | 减少词法歧义 |
| 6 | External Scanner 扩展 | 处理非正则词法需求 |
| 7 | S-表达式 Query 系统 | 声明式树匹配能力 |
| 8 | 单文件聚合 + 最小依赖 | 极易嵌入 |
| 9 | 多语言绑定（C/Rust/WASM） | 广泛生态兼容 |
| 10 | JavaScript DSL 语法定义 | 开发者友好 |

---

*文档生成时间：2026-03-27*
*基于 Tree-sitter v0.27.0（Cargo workspace）源码分析*
