# Tree-sitter 核心业务流程分析

> 基于 Tree-sitter v0.27.0 源码分析
> 依赖文档：[architecture.md](./architecture.md)、[modules.md](./modules.md)

---

## 目录

1. [流程 1：Grammar 编译流程](#流程-1grammar-编译流程)
2. [流程 2：解析流程（Runtime）](#流程-2解析流程runtime)
3. [流程 3：增量解析流程](#流程-3增量解析流程)

---

## 流程 1：Grammar 编译流程

**主线**：`grammar.js` → JavaScript 运行时执行 → JSON → 语法预处理管道 → 构建解析表/词法表 → 渲染为 C 代码 → 编译为 `.so`

### 1.1 全流程步骤图

```
grammar.js
    │
    ▼
┌─────────────────────────────────┐
│  1. JS 运行时执行 grammar.js    │  crates/generate/src/generate.rs  load_js_grammar_file()
│     QuickJS (native) 或         │  crates/generate/src/quickjs.rs   execute_native_runtime()
│     Node/Bun/Deno (子进程)      │
│     DSL: grammar() → JSON       │  内嵌 dsl.js 提供 grammar/rule/... 函数
└────────────┬────────────────────┘
             │  JSON 字符串
             ▼
┌─────────────────────────────────┐
│  2. parse_grammar               │  crates/generate/src/parse_grammar.rs  parse_grammar()
│     JSON → InputGrammar         │
│     解析 rules/extras/conflicts │
│     /externals/precedences 等   │
└────────────┬────────────────────┘
             │  InputGrammar
             ▼
┌─────────────────────────────────┐
│  3. prepare_grammar (管道)      │  crates/generate/src/prepare_grammar.rs
│                                 │
│  3a. validate_precedences       │  校验优先级声明
│  3b. validate_indirect_recursion│  检测间接递归环
│  3c. intern_symbols             │  字符串名 → 符号 ID
│  3d. extract_tokens             │  分离句法规则与词法规则
│  3e. expand_repeats             │  Repeat → 辅助二叉树非终结符
│  3f. flatten_grammar            │  Rule AST → Production 列表
│  3g. expand_tokens              │  正则/字符串 → NFA
│  3h. extract_default_aliases    │  提取全局默认 alias
│  3i. process_inlines            │  构建 inline 展开映射
│                                 │
│  输出: SyntaxGrammar            │
│      + LexicalGrammar           │
│      + InlinedProductionMap     │
│      + AliasMap                 │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  4. build_tables                │  crates/generate/src/build_tables/
│                                 │
│  4a. build_parse_table          │  LR(1) 项目集构造 + 冲突消解
│  4b. populate_error_state       │  填充错误状态动作
│  4c. minimize_parse_table       │  状态合并优化
│  4d. build_lex_table            │  NFA → DFA 子集构造
│  4e. minimize_lex_table         │  词法状态去重
│  4f. populate_external_lex      │  外部扫描器状态
│  4g. mark_fragile_tokens        │  标记脆弱 token
│                                 │
│  输出: ParseTable + LexTable    │
│      + keyword_lex_table        │
└────────────┬────────────────────┘
             │  Tables
             ▼
┌─────────────────────────────────┐
│  5. render_c_code               │  crates/generate/src/render.rs  Generator::generate()
│     Tables → parser.c 源码     │
│     含: ts_lex / ts_lex_keywords│
│         parse_table / small_    │
│         parse_table             │
│         symbol_names/metadata   │
│         field_map / alias_seq   │
│         TSLanguage 导出函数     │
└────────────┬────────────────────┘
             │  parser.c (字符串)
             ▼
┌─────────────────────────────────┐
│  6. 写入文件 + 编译             │  generate_parser_in_directory()
│     parser.c + parser.h +       │  crates/loader → cc 编译 → .so/.dylib
│     alloc.h + array.h           │
│     → C 编译器 → .so/.dylib    │
└─────────────────────────────────┘
```

### 1.2 步骤详解

#### 步骤 1：grammar.js 的加载与执行

**入口**：`generate_parser_in_directory()` → `load_grammar_file()` → `load_js_grammar_file()`

**文件**：`crates/generate/src/generate.rs`、`crates/generate/src/quickjs.rs`

Tree-sitter 使用 JavaScript DSL 定义语法。`grammar.js` 文件调用 `grammar()` 函数，该函数由内嵌的 `dsl.js` 提供，将规则声明序列化为 JSON。

**两种执行路径**：

| 路径 | 条件 | 机制 |
|------|------|------|
| **QuickJS (native)** | `js_runtime == "native"` 且编译了 `qjs-rt` feature | 进程内嵌入 QuickJS 引擎，`Module::evaluate("dsl", DSL)` 执行 dsl.js，读取 `globalThis.output` |
| **Node/Bun/Deno** | 默认路径 | 子进程启动 JS 运行时，stdin 写入 CLI 版本 + `dsl.js`，`TREE_SITTER_GRAMMAR_PATH` 环境变量指向 grammar 路径，stdout 最后一行为 JSON |

**`dsl.js` 的核心职责**：
- 提供 `grammar()`、`rule()`、`choice()`、`seq()`、`repeat()`、`token()` 等 DSL 函数
- 执行 `grammar.js`，将声明式规则转换为嵌套 JSON 结构
- 输出到 `globalThis.output`（QuickJS）或 stdout（Node 路径）

**QuickJS 运行时环境设置** (`quickjs.rs: execute_native_runtime`)：
- `require` 函数：从 grammar 文件目录解析模块路径，支持 `.js`（CommonJS 包装）和 `.json`（`JSON.parse`）
- `module.exports`、`console`、`process.env` 等 Node 兼容全局对象

#### 步骤 2：JSON → InputGrammar

**入口**：`parse_grammar(input: &str)`

**文件**：`crates/generate/src/parse_grammar.rs`

将 DSL 产出的 JSON 反序列化为 `InputGrammar`：

```
GrammarJSON                          InputGrammar
├── name: String          ──→        ├── name: String
├── rules: Map<name, RuleJSON>       ├── variables: Vec<Variable<Rule>>
├── extras: Vec<RuleJSON>            ├── extra_symbols: Vec<Rule>
├── conflicts: Vec<Vec<name>>        ├── expected_conflicts: Vec<Vec<Symbol>>
├── externals: Vec<RuleJSON>         ├── external_tokens: Vec<ExternalToken>
├── precedences: Vec<Vec<RuleJSON>>  ├── precedence_orderings: Vec<Vec<Rule>>
├── inline: Vec<name>                ├── variables_to_inline: Vec<Symbol>
├── supertypes: Vec<name>            ├── supertype_symbols: Vec<Symbol>
├── word: name                       ├── word_token: Option<Symbol>
└── reserved: Map<name, Vec>         └── reserved_words: Vec<(name, Vec<Rule>)>
```

**关键逻辑**：
- `parse_rule()` 递归将 `RuleJSON` 变为 `rules::Rule`（CHOICE/SEQ/REPEAT/TOKEN 等）
- 可达性剪枝：从第一条规则做 BFS，未引用的变量从 conflicts/supertypes/inline 等中剔除

#### 步骤 3：prepare_grammar 管道

**入口**：`prepare_grammar(&InputGrammar)`

**文件**：`crates/generate/src/prepare_grammar.rs` 及 `prepare_grammar/` 子目录

```
InputGrammar
    │
    ▼ validate_precedences()           校验优先级规则引用的符号存在
    ▼ validate_indirect_recursion()    检测 A→B→A 式间接递归环
    │
    ▼ intern_symbols()                 ── intern_symbols.rs
    │  字符串符号名 → 带类型的 Symbol(index, kind)
    │  校验 start rule 非 hidden、supertypes/word 存在性
    │  输出: InternedGrammar
    │
    ▼ extract_tokens()                 ── extract_tokens.rs
    │  从句法规则中抽出字符串/正则 → 独立的词法变量
    │  SymbolReplacer 重连索引
    │  输出: (ExtractedSyntaxGrammar, ExtractedLexicalGrammar)
    │
    ▼ expand_repeats()                 ── expand_repeats.rs
    │  Rule::Repeat(x) → 辅助非终结符 + choice(seq(aux, x), x) 二叉树结构
    │  相同内容复用同一 repeat 符号
    │  输出: SyntaxGrammar (含辅助变量)
    │
    ▼ flatten_grammar()                ── flatten_grammar.rs
    │  Rule AST → Vec<Production>
    │  Choice 展开为多个 Production
    │  Seq 展平为 ProductionStep 序列
    │  Metadata（prec/assoc/alias/field）下压到 step
    │  校验空产生式
    │  输出: SyntaxGrammar (含 productions + reserved_word_sets)
    │
    ▼ expand_tokens()                  ── expand_tokens.rs
    │  为每个词法变量在共享 NFA 上构建状态转移
    │  正则通过 regex-syntax → HIR → NFA
    │  非 immediate token 后接 separator（extras 合成的 repeat choice）
    │  输出: LexicalGrammar { nfa: Nfa, variables: Vec<LexicalVariable> }
    │
    ▼ extract_default_aliases()        ── extract_default_aliases.rs
    │  若某符号在所有产生式中都带相同 alias，升为默认 alias
    │  减小表大小，统一 ERROR 子节点名
    │  输出: AliasMap (BTreeMap<Symbol, Alias>)
    │
    ▼ process_inlines()                ── process_inlines.rs
      预计算 inline 展开后的产生式列表
      不修改 SyntaxGrammar 结构，仅生成 InlinedProductionMap
      供 LR closure 使用
      输出: InlinedProductionMap
```

#### 步骤 4：构建解析表和词法表

**入口**：`build_tables()`

**文件**：`crates/generate/src/build_tables/`

##### 4a. 构建解析表 (`build_parse_table.rs`)

**算法**：LR(1) 项目集构造

```
1. 初始化:
   ├── 状态 0: 错误状态（保留）
   └── 状态 1: 起始项集 { START → • S, {$} }

2. BFS 队列循环:
   for each 新状态:
     │
     ├── transitive_closure(item_set)     ── item_set_builder.rs
     │   ├── 计算全局 FIRST/LAST 集合
     │   ├── 展开 inline 产生式
     │   └── 对点前非终结符: 加入所有产生式,
     │       传播终结符 lookahead 和 reserved_word_set_id
     │
     └── add_actions(state, item_set)
         ├── 未完成项(点未到末尾):
         │   按下一符号分组 → terminal_successors / non_terminal_successors
         │   合并 lookahead → add_parse_state (按项集去重)
         │
         ├── 完成项(点到末尾):
         │   → Reduce 动作 (或增广规则的 Accept)
         │   优先级抢先: 低优先级 reduce 不插入
         │
         ├── terminal 边 → Shift
         ├── non_terminal 边 → Goto
         │
         └── 冲突处理 (handle_conflict):
             ├── Shift/Reduce 冲突: 比较优先级和结合性
             ├── Reduce/Reduce 冲突: 比较优先级
             ├── 检查 grammar conflicts 白名单
             └── 无法消解 → ConflictError

3. 后处理:
   ├── populate_error_state: 状态 0 的 Recover 动作
   └── minimize_parse_table: 迭代合并等价状态
```

##### 4b. 构建词法表 (`build_lex_table.rs`)

**算法**：NFA → DFA 子集构造

```
1. 按 parse state 收集可用终结符集合
   ├── merge_token_set: 无词法冲突的状态合并为同一 lex 入口
   └── 每个等价类 → 一个 lex_state_id，写回 parse_table.states[*].lex_state_id

2. 对每个 lex 入口:
   │
   ├── 初始 NFA 状态集 = 所选 token 的 start_state
   │
   └── add_state (经典子集构造):
       ├── (排序后的 NFA id 向量, eof_valid) → 去重 key
       │
       └── populate_state:
           ├── cursor.completions(): Accept 候选 → TokenConflictMap.prefer_token 择一
           ├── cursor.transitions(): NFA 边的字符集细分(group_transitions)
           │   → 互斥边 → (CharacterSet, AdvanceAction{next_state})
           └── eof → eof_action

3. minimize_lex_table: 迭代细分等价类, 合并同结构 lex 状态
```

**关键词表**：若存在 `word_token`，对 keywords 子集单独建一张小词法表 (`keyword_lex_table`)，用于运行时的关键词消歧。

#### 步骤 5：渲染 C 代码

**入口**：`render_c_code()`

**文件**：`crates/generate/src/render.rs`

`Generator::generate()` 按固定顺序输出一个完整的 C 文件：

```
parser.c 结构
├── 头文件 include 和 pragma
├── 常量宏 (LANGUAGE_VERSION, STATE_COUNT, SYMBOL_COUNT, ...)
├── 符号枚举 (enum { sym_identifier = 1, sym_number = 2, ... })
├── 符号名表 (ts_symbol_names[])
├── 符号元数据表 (ts_symbol_metadata[])
├── 字段枚举和映射 (field_names, field_map_slices, field_map_entries)
├── alias 序列 (ts_alias_sequences)
├── primary_state_ids
├── supertype 映射 (ABI ≥ 15)
├── 大字符集常量 (set_contains 查表)
├── 词法函数 ts_lex() / ts_lex_keywords()
│   └── switch-case 状态机, 每状态按字符分支
├── lex_modes 数组 (parse_state → lex_state/external_lex_state/reserved_word_set_id)
├── reserved_word_sets
├── 解析表:
│   ├── parse_table (大表: 二维数组, O(1) 查找)
│   └── small_parse_table (小表: 分组压缩, 节省空间)
├── 外部扫描器枚举和状态
└── TSLanguage 导出函数: tree_sitter_{name}()
```

**大字符集优化**：字符集范围 ≥ 8 时提取为常量数组，用 `set_contains()` 二分查找替代逐一比较。

#### 步骤 6：编译为共享库

**文件**：`crates/loader/src/loader.rs`

Loader 模块负责编译和加载：
1. 检查已有 `.so`/`.dylib` 是否过期
2. 用 `cc` crate 编译 `parser.c` + 可选的 `scanner.c`（外部扫描器）
3. `libloading` 动态加载库，调用 `tree_sitter_{name}()` 获取 `TSLanguage*`

### 1.3 数据结构演变总结

```
grammar.js  ──JS执行──→  JSON string
    ──parse_grammar──→   InputGrammar { variables: Vec<Variable<Rule>> }
    ──intern_symbols──→  InternedGrammar { variables 中的 Rule 引用 Symbol(id) }
    ──extract_tokens──→  (ExtractedSyntaxGrammar, ExtractedLexicalGrammar)
    ──expand_repeats──→  SyntaxGrammar (含辅助 repeat 变量)
    ──flatten_grammar──→ SyntaxGrammar { variables[i].productions: Vec<Production> }
    ──expand_tokens──→   LexicalGrammar { nfa: Nfa, variables: Vec<LexicalVariable> }
    ──build_tables──→    ParseTable { states: Vec<ParseState> }
                       + LexTable { states: Vec<LexState> }
    ──render_c_code──→   String (parser.c 源码)
    ──cc 编译──→         parser.so / parser.dylib
```

---

## 流程 2：解析流程（Runtime）

**主线**：输入文本 → Lexer tokenize → Parser 状态机驱动 → 构建 CST

### 2.1 全流程步骤图

```
              TSInput.read (用户提供的读取回调)
                    │
                    ▼
┌──────────────────────────────────────────────────────────┐
│  1. 初始化                                                │
│     ts_parser_parse()                 parser.c:2074       │
│     ├── 绑定 TSInput                                      │
│     ├── 创建 external_scanner                             │
│     ├── 初始化栈 (初始状态=1)                              │
│     └── 若有 old_tree: 初始化 ReusableNode                │
└───────────────────┬──────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────────────┐
│  2. 主解析循环 (do-while)             parser.c:2122       │
│                                                           │
│  do {                                                     │
│    for each active stack version:                         │
│      while version is active:                             │
│        ┌─────────────────────────────────────────────┐    │
│        │  3. ts_parser__advance()   parser.c:1557    │    │
│        │                                             │    │
│        │  ┌──── 尝试复用旧树节点 ◄── (增量解析时)    │    │
│        │  │     ts_parser__reuse_node()               │    │
│        │  │                                           │    │
│        │  ├──── 尝试 Token 缓存                      │    │
│        │  │     ts_parser__get_cached_token()         │    │
│        │  │                                           │    │
│        │  ├──── 词法分析                              │    │
│        │  │     ts_parser__lex()    parser.c:505      │    │
│        │  │     ├── external scanner (若有)           │    │
│        │  │     ├── 内部 lex_fn                       │    │
│        │  │     ├── keyword 消歧                      │    │
│        │  │     └── 错误字符跳过                      │    │
│        │  │                                           │    │
│        │  └──── 处理解析动作                          │    │
│        │        ts_language_table_entry() 查表        │    │
│        │        ├── Shift  → 压栈, 返回              │    │
│        │        ├── Reduce → 弹出子节点, 建父节点    │    │
│        │        ├── Accept → 解析完成                 │    │
│        │        └── Recover → 错误恢复               │    │
│        └─────────────────────────────────────────────┘    │
│                                                           │
│    ts_parser__condense_stack()  // 裁剪/合并栈版本        │
│    if finished_tree.error_cost < min_error_cost: break    │
│  } while (有活跃栈版本)                                    │
└───────────────────┬──────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────────────┐
│  4. 后处理                                                │
│     ts_parser__balance_subtree()  // 平衡重复节点         │
│     ts_tree_new(finished_tree)    // 封装为 TSTree        │
│     ts_parser_reset()             // 清理解析器状态       │
└──────────────────────────────────────────────────────────┘
```

### 2.2 步骤详解

#### 步骤 1：字符读取与缓冲

**TSInput 回调机制**

```c
// lib/include/tree_sitter/api.h
typedef struct {
  void *payload;
  const char *(*read)(         // 读取回调
    void *payload,
    uint32_t byte_index,       // 请求的起始字节偏移
    TSPoint position,          // 行列位置
    uint32_t *bytes_read       // 输出: 实际读取字节数
  );
  TSInputEncoding encoding;    // UTF8 / UTF16LE / UTF16BE / 自定义
} TSInput;
```

**Chunk 管理** (`lexer.c`)

Lexer 内部维护一个 chunk 缓存：

```
Lexer
├── chunk: const char*         当前文本块指针 (指向 TSInput.read 返回的缓冲区)
├── chunk_start: uint32_t      chunk 在文档中的起始偏移
├── chunk_size: uint32_t       chunk 的字节大小
├── current_position: Length    当前读取位置 (bytes + row/col)
└── lookahead_size: uint32_t   当前前瞻字符的字节长度
```

**读取流程**：
1. `ts_lexer__get_chunk()` (`lexer.c:89`)：调用 `input.read(payload, position.bytes, position.extent, &chunk_size)` 获取文本块
2. `ts_lexer__get_lookahead()` (`lexer.c:106`)：解码下一个 Unicode 码点，根据 encoding 选择 `ts_decode_utf8` / `ts_decode_utf16_le` / `ts_decode_utf16_be`；跨 chunk 的多字节字符会触发重新 `get_chunk`
3. `ts_lexer__do_advance()` (`lexer.c:200`)：前进一个字符，更新位置信息，处理换行/列号，检查 `included_ranges` 边界（超出当前 range 时跳转到下一个 range），需要时重新加载 chunk

**TSLexer 虚函数表**

生成的 `lex_fn` 通过 `TSLexer` 结构体的函数指针与 Lexer 交互，类似 C++ 虚函数表：

```c
struct TSLexer {
  int32_t lookahead;              // 当前前瞻字符 (Unicode 码点)
  TSSymbol result_symbol;         // lex_fn 设置匹配到的 token 符号
  void (*advance)(TSLexer *, bool skip);   // 前进一个字符
  void (*mark_end)(TSLexer *);             // 标记 token 结束位置
  uint32_t (*get_column)(TSLexer *);       // 获取列号
  bool (*is_at_included_range_start)(const TSLexer *);
  bool (*eof)(const TSLexer *);
};
```

**设计要点**：`TSLexer` 是 `Lexer` 的第一个字段，`(Lexer *)lexer` 的类型转换安全（C 标准保证首成员地址对齐）。生成的 parser 只依赖 `TSLexer` 结构定义（在 `parser.h` 中），不需要链接 tree-sitter 库符号。

#### 步骤 2：词法分析 (`ts_parser__lex`)

**文件**：`lib/src/parser.c:505-699`

```
ts_parser__lex(self, version, parse_state)
│
├── lex_mode = ts_language_lex_mode_for_state(parse_state)
│   从 TSLanguage.lex_modes[parse_state] 获取:
│   ├── lex_state:              内部词法状态编号
│   ├── external_lex_state:     外部扫描器状态编号 (0=无)
│   └── reserved_word_set_id:   保留字集合 ID
│
├── ts_lexer_reset(start_position)     重置到栈顶位置
│
└── for (;;) {  // 词法循环
    │
    ├── 阶段 1: 外部扫描器 (若 external_lex_state ≠ 0)
    │   ├── ts_lexer_start()
    │   ├── ts_parser__external_scanner_deserialize(external_token)
    │   │   从上一个 external token 的序列化状态恢复扫描器
    │   ├── ts_parser__external_scanner_scan(lex_mode.external_lex_state)
    │   │   调用 language->external_scanner.scan(payload, lexer, valid_external_tokens)
    │   ├── ts_lexer_finish()
    │   ├── 成功: serialize 状态, break
    │   └── 失败: ts_lexer_reset, 继续内部词法
    │
    ├── 阶段 2: 内部词法 (生成的 lex_fn)
    │   ├── ts_lexer_start()
    │   ├── ts_parser__call_main_lex_fn(lex_mode)
    │   │   language->lex_fn(&lexer.data, lex_mode.lex_state)
    │   │   生成的 switch-case 状态机, 逐字符匹配
    │   ├── ts_lexer_finish()
    │   └── 成功: break
    │
    ├── 阶段 3: 错误模式 (若内部词法失败)
    │   ├── 首次失败: 切换到 ERROR_STATE 的 lex_mode
    │   │   ts_lexer_reset(start_position), continue
    │   └── 仍失败: 逐字符跳过, 构造 error leaf
    │
    └── 阶段 4: 关键词消歧 (lex 成功后)
        若 symbol == keyword_capture_token:
        ├── ts_lexer_reset(token_start)
        ├── ts_parser__call_keyword_lex_fn()
        └── 若匹配关键词且在当前状态有效: 替换 symbol
    }

    → 构造 Subtree:
      ├── 正常 token: ts_subtree_new_leaf(symbol, padding, size, ...)
      ├── 跳过的错误字符: ts_subtree_new_error(first_error_char, ...)
      └── 外部 token: 额外存储 ExternalScannerState
```

**关键细节 — 外部扫描器与 lex_fn 的关系**：
- 外部扫描器**优先于**内部 lex_fn
- 外部扫描器使用与 lex_fn 相同的 `TSLexer` 接口
- 若外部扫描器匹配了空 token 且无状态变化，在 error_mode 或 extra 时会被忽略（防止无限循环）

**关键细节 — parse state 与 lex state 的关系**：
- 每个 parse state 通过 `lex_modes[]` 映射到特定的 `lex_state`
- 不同的 parse state 可能共享同一个 `lex_state`（在 `build_lex_table` 阶段合并）
- 这种耦合使词法器能根据解析上下文做出不同决策（如 JSX 中 `<` 可以是比较运算符或标签开始）

#### 步骤 3：Parser 主循环与 `ts_parser__advance`

**文件**：`lib/src/parser.c:1557-1763`

**主循环控制** (`ts_parser_parse:2122-2176`)：

```
do {
  for each version (0..version_count):
    allow_node_reuse = (version_count == 1)   // 多版本时关闭复用
    while version is active:
      ts_parser__advance(version, allow_node_reuse)
      position = ts_stack_position(version).bytes
      if position > last_position:            // 有前进
        last_position = position
        break                                  // 让其他版本也推进
  min_error_cost = ts_parser__condense_stack() // 裁剪劣势版本
  if finished_tree.error_cost < min_error_cost:
    break                                      // 已有最优解
} while (version_count != 0)
```

**`ts_parser__advance` 控制流**：

```
ts_parser__advance(version, allow_node_reuse)
│
├── 获取上下文: state, position, last_external_token
│
├── 获取 lookahead token:
│   ├── 优先级 1: reuse_node (增量解析, 仅 version_count==1)
│   ├── 优先级 2: cached_token (同次解析的缓存)
│   └── 优先级 3: ts_parser__lex (新词法分析)
│       → ts_parser__set_cached_token  (缓存供其他版本使用)
│
├── ts_language_table_entry(state, symbol) → table_entry
│   查解析表获取该状态下该符号的动作列表
│
├── 进度回调检查: ts_parser__check_progress()
│   每 100 次操作检查一次, 可取消解析
│
└── 动作处理循环:
    for each action in table_entry.actions:
    │
    ├── Shift:
    │   ├── 若 lookahead 是非叶节点(复用的子树):
    │   │   ts_parser__breakdown_lookahead → 分解为第一个叶子
    │   ├── ts_parser__shift(version, next_state, lookahead, extra)
    │   ├── 若是复用: reusable_node_advance()
    │   └── return true
    │
    ├── Reduce:
    │   ├── ts_parser__reduce(version, symbol, child_count, ...)
    │   ├── 记录 last_reduction_version
    │   └── continue (同一 lookahead 可能有多个 reduce)
    │
    ├── Accept:
    │   ├── ts_parser__accept(version, lookahead)
    │   └── return true
    │
    └── Recover:
        ├── ts_parser__recover(version, lookahead)
        └── return true

    若只有 reduce (无 shift):
    ├── renumber_version, 更新 state
    ├── 重新查表, continue
    └── 若 reduce 合并入已有版本: halt 当前版本

    若无有效动作:
    ├── keyword 回退: 尝试将关键词改回 word_token
    ├── breakdown_top_of_stack: 拆开栈顶复用的节点
    └── ts_stack_pause(version, lookahead)  → 标记为暂停
```

#### 步骤 4：Shift 操作详解

**文件**：`lib/src/parser.c:908-929`

```
ts_parser__shift(version, state, lookahead, extra)
│
├── 若 extra 标记与 subtree 不一致(且是叶子):
│   ts_subtree_make_mut → ts_subtree_set_extra
│
├── ts_stack_push(stack, version, subtree, pending=!is_leaf, state)
│   ├── 创建新 StackNode, 前驱指向当前 head
│   └── 继承并累加: position, error_cost, node_count, dynamic_precedence
│
└── 若 subtree 含 external tokens:
    ts_stack_set_last_external_token(subtree 中最后一个 external token)
```

**`pending` 标志**：非叶节点（从旧树复用的整个子树）标记为 `pending=true`，表示尚未归约确认。后续若发现不合法，`ts_parser__breakdown_top_of_stack` 会将其拆开。

#### 步骤 5：Reduce 操作详解

**文件**：`lib/src/parser.c:931-1046`

```
ts_parser__reduce(version, symbol, child_count, dynamic_precedence, production_id, ...)
│
├── 1. ts_stack_pop_count(version, child_count)
│      沿 GLR 图枚举路径, 每条路径得到 StackSlice(version, SubtreeArray)
│      GLR 分叉点: 单次 pop 可能产生多个 slice
│
├── 2. 对每条 slice:
│   ├── 版本数溢出检查:
│   │   if slice_version > MAX_VERSION_COUNT + OVERFLOW + halted:
│   │     remove_version, continue
│   │
│   ├── ts_subtree_array_remove_trailing_extras → trailing_extras
│   │   额外的 extra 节点(注释等)不包含在父节点中
│   │
│   ├── ts_subtree_new_node(symbol, &children, production_id, language)
│   │   分配 SubtreeHeapData + children 连续内存
│   │   ts_subtree_summarize_children: 汇总聚合属性
│   │
│   ├── 多路径择优:
│   │   while 下一个 slice 属于同一 version:
│   │     ts_parser__select_children(parent, &next_children)
│   │     选择 error_cost 更低 / dynamic_precedence 更高的子节点集合
│   │
│   ├── GOTO: state = ts_stack_state(slice_version)
│   │        next_state = ts_language_next_state(language, state, symbol)
│   │   ├── 非终结符: ts_language_lookup(state, symbol) 直接索引
│   │   └── 终结符: 扫描动作表找最后一个 shift 的目标状态
│   │
│   ├── 设置属性:
│   │   fragile = (is_fragile || pop.size > 1 || version_count > 1)
│   │   parse_state = fragile ? NONE : state
│   │   dynamic_precedence += dynamic_precedence
│   │
│   ├── ts_stack_push(slice_version, parent, false, next_state)
│   │   trailing_extras 也逐个 push
│   │
│   └── 尝试 merge 到已有版本:
│       for j < slice_version:
│         ts_stack_merge(j, slice_version)
│         条件: 同状态 + 同位置 + 同 error_cost + 同 external_token_state
│
└── 3. 返回第一个新创建的版本号 (或 STACK_VERSION_NONE)
```

#### 步骤 6：GLR 多分支并行与冲突解决

**文件**：`lib/src/parser.c`（多处）、`lib/src/stack.c`

##### 分支的产生

GLR 分支来自以下场景：

| 场景 | 机制 |
|------|------|
| Shift/Reduce 冲突 | 同一 lookahead 同时有 Shift 和 Reduce 动作 → Reduce 创建新 version，Shift 更新原 version |
| Reduce/Reduce 冲突 | 同一 lookahead 有多个 Reduce 动作 → 每个 Reduce 创建新 version |
| Pop 分叉 | `ts_stack_pop_count` 在 GLR 图的合并点分裂 → 多条 StackSlice |
| 错误恢复 | `ts_stack_copy_version` 创建新 version 尝试不同恢复策略 |

##### Graph-Structured Stack (GSS)

```
Head0 → [NodeA] ─link0─→ [NodeC] ──→ [base]
Head1 → [NodeB] ─link0─→ [NodeC]    (NodeC 被共享, ref_count > 1)
                 ╲link1─→ [NodeD] ──→ [base]  (NodeB 有两条 link = 歧义)
```

每个 `StackNode` 最多 8 条前驱链接，支持 GLR 分叉。多个 `StackHead` 共享底层子图。

##### 栈收缩 (`ts_parser__condense_stack`)

**文件**：`lib/src/parser.c:1766-1864`

```
ts_parser__condense_stack:
│
├── 1. 移除已 halt 的版本
│
├── 2. 两两比较 (ErrorStatus):
│      ErrorStatus = { error_cost, is_in_error, dynamic_precedence }
│      ├── 明显更差: remove_version
│      ├── 可合并 (同状态+同位置+同error_cost): ts_stack_merge
│      └── 需交换: swap (保持优先顺序)
│
├── 3. 硬上限裁剪:
│      while version_count > MAX_VERSION_COUNT(6):
│        ts_stack_remove_version(MAX_VERSION_COUNT)
│
└── 4. 暂停版本恢复:
       若所有活跃版本都 paused:
       ├── ts_stack_resume (恢复保存的 lookahead)
       └── ts_parser__handle_error → 进入错误恢复
```

**关键常量**：

| 常量 | 值 | 含义 |
|------|------|------|
| `MAX_VERSION_COUNT` | 6 | GLR 栈最大版本数硬上限 |
| `MAX_VERSION_COUNT_OVERFLOW` | 4 | Reduce 期间临时允许超出的量 |
| `MAX_COST_DIFFERENCE` | 1800 | 版本间代价差超过此值强制删除 |

#### 步骤 7：错误恢复

**文件**：`lib/src/parser.c:1439-1437`

**两阶段策略**：

```
阶段 1: ts_parser__handle_error (parser.c:1439)
│
├── ts_parser__do_all_potential_reductions(version, 0)
│   不看 lookahead，尝试所有可能的 reduce
│   目的: 在跳过坏 token 后，某些 reduce 可能本来是合法的
│
├── 对每个产生的 version:
│   尝试插入 MISSING token:
│   ├── ts_subtree_new_missing_leaf(symbol)
│   ├── ts_stack_push(missing_leaf)
│   └── ts_parser__do_all_potential_reductions(version, lookahead_symbol)
│
├── ts_stack_push(NULL_SUBTREE, false, ERROR_STATE)
│   在栈上标记错误断裂点
│
├── merge 多余版本
│
├── ts_stack_record_summary(depth ≤ MAX_SUMMARY_DEPTH=16)
│   记录栈历史 (state, depth, position) 供回溯恢复使用
│
├── 若 lookahead 是非叶: breakdown_lookahead
│
└── ts_parser__recover(version, lookahead)  → 进入阶段 2


阶段 2: ts_parser__recover (parser.c:1250)
│
├── 策略 1 — 回溯到历史状态:
│   遍历 stack summary:
│   for each entry(state, depth, position):
│   │
│   ├── 跳过 ERROR_STATE 和同位置
│   ├── 估算代价: depth * COST_PER_SKIPPED_TREE + bytes * COST_PER_SKIPPED_CHAR
│   ├── if 代价已超过最优版本: break
│   │
│   └── if ts_language_has_actions(entry.state, lookahead_symbol):
│       ts_parser__recover_to_state(version, depth, entry.state)
│       ├── pop depth 层
│       ├── 合并栈上节点为 ERROR 节点
│       ├── push 到 goal_state
│       └── did_recover = true, break
│
├── EOF 特殊处理:
│   全部包装为 ERROR 节点 → ts_parser__accept
│
├── 策略 2 — 跳过当前 token:
│   (与策略 1 并行: 策略 1 成功后仍可尝试策略 2)
│   │
│   ├── 代价检查: 若跳过代价超过最优版本 → halt
│   │
│   ├── 包装 lookahead 为 error_repeat 节点
│   │   children = [lookahead]
│   │   error_repeat = ts_subtree_new_node(ts_builtin_sym_error_repeat, ...)
│   │
│   ├── 若栈顶已有 error (node_count_since_error > 0):
│   │   pop 栈顶 error_repeat
│   │   合并: new_error_repeat = [old_error_repeat, current_error_repeat]
│   │
│   └── ts_stack_push(error_repeat, false, ERROR_STATE)
│       继续在 ERROR_STATE 中消耗 token
│
└── 更新 self->has_error (所有版本都在错误中?)
```

**错误恢复代价常量**：

| 常量 | 值 | 含义 |
|------|------|------|
| `ERROR_COST_PER_RECOVERY` | 500 | 每次错误恢复的基础代价 |
| `ERROR_COST_PER_MISSING_TREE` | 110 | 每个 MISSING 节点的代价 |
| `ERROR_COST_PER_SKIPPED_TREE` | 100 | 每个跳过的树的代价 |
| `ERROR_COST_PER_SKIPPED_CHAR` | 1 | 每个跳过字符的代价 |
| `ERROR_COST_PER_SKIPPED_LINE` | 30 | 每个跳过行的代价 |

#### 步骤 8：树的构建过程

**文件**：`lib/src/subtree.c`

##### 叶子节点创建 (`ts_subtree_new_leaf`)

```
ts_subtree_new_leaf(pool, symbol, padding, size, lookahead_bytes, parse_state, ...)
│
├── 检查 ts_subtree_can_inline:
│   symbol ≤ 255 且无外部 token 且 padding/size 各字段不超出位域范围
│   且不跨行
│
├── 可 inline:
│   构造 SubtreeInlineData (8 字节, 值类型)
│   ┌─────────────────────────────────────────────┐
│   │ is_inline=1 | visible | named | extra | ... │
│   │ symbol(8b) | parse_state(16b)               │
│   │ padding_cols | padding_rows | lookahead      │
│   │ padding_bytes | size_bytes                    │
│   └─────────────────────────────────────────────┘
│
└── 不可 inline:
    从 SubtreePool 获取或 malloc SubtreeHeapData
    设置 ref_count=1, padding, size, symbol, parse_state, ...
```

##### 内部节点创建 (`ts_subtree_new_node`)

```
ts_subtree_new_node(symbol, &children, production_id, language)
│
├── 分配连续内存:
│   ┌──────────┬──────────┬─────┬──────────┬─────────────────┐
│   │ child[0] │ child[1] │ ... │ child[n] │ SubtreeHeapData │
│   │ (8 bytes)│ (8 bytes)│     │ (8 bytes)│   (父节点数据)   │
│   └──────────┴──────────┴─────┴──────────┴─────────────────┘
│                                            ↑ ptr 指向这里
│   子节点通过 (Subtree *)(ptr) - child_count 回退访问
│
├── ts_subtree_summarize_children(parent, language):
│   从子节点汇总计算:
│   ├── padding = 第一个子节点的 total_size (前导空白)
│   ├── size = 所有子节点 total_size 之和 - padding
│   ├── error_cost = Σ子节点 error_cost
│   ├── dynamic_precedence = Σ子节点 dynamic_precedence
│   ├── visible_child_count, named_child_count
│   ├── visible_descendant_count
│   ├── has_external_tokens (任意子节点有)
│   ├── depends_on_column (任意子节点有)
│   └── first_leaf = 最左叶子的 (symbol, parse_state)
│
└── 设置 ref_count=1, symbol, production_id, visible, named, ...
```

##### 最终树封装

```
ts_parser__accept(version, lookahead)     parser.c:1048
│
├── push EOF lookahead
├── ts_stack_pop_all(version) → SubtreeArray
├── 找到第一个非 extra 父节点
├── 展开其 children 并重建根节点
├── ts_parser__select_tree: 多个 accept 路径择优
└── 写入 self->finished_tree

ts_parser_parse 返回:
├── ts_parser__balance_subtree: 对高 repeat_depth 的链做平衡
│   ts_subtree_compress: 类似 AVL 左旋, O(n) 深度 → O(log n)
├── ts_tree_new(finished_tree, language, included_ranges)
│   retain 根 + copy language + copy ranges → TSTree
└── ts_parser_reset: 清理状态
```

#### 步骤 9：Token Cache

**文件**：`lib/src/parser.c:83-87, 703-737`

```c
typedef struct {
  Subtree token;                // 缓存的 token
  Subtree last_external_token;  // 缓存时的外部 token 状态
  uint32_t byte_index;          // 缓存位置
} TokenCache;
```

**用途**：在同一次解析中，当多个 GLR 栈版本在相同位置需要 lex 时，避免重复词法分析。

**命中条件** (`ts_parser__get_cached_token`)：
1. 缓存 token 存在
2. `byte_index == position`（位置相同）
3. external scanner 状态等价
4. 在当前 `state` 查表后，`ts_parser__can_reuse_first_leaf` 确认 token 仍合法

**与增量复用的区别**：Token Cache 是**同一次 parse** 的短路优化；`ReusableNode` 是从**上一次 parse 的旧树**中整块复用子树。

---

## 流程 3：增量解析流程

**主线**：旧语法树 + 编辑操作 → 标记变化 → 最小范围重解析 → 新语法树

### 3.1 全流程步骤图

```
              旧 TSTree  +  TSInputEdit  +  新文本
                │               │
                ▼               ▼
┌──────────────────────────────────────────────────────────┐
│  1. 应用编辑: ts_tree_edit()          tree.c:55          │
│     ├── 更新 included_ranges 位置                        │
│     └── ts_subtree_edit(): 递归更新子树                  │
│         ├── 调整 padding/size                            │
│         └── 标记 has_changes                             │
└───────────────────┬──────────────────────────────────────┘
                    │  编辑后的旧树 (位置已调整, has_changes 已标记)
                    ▼
┌──────────────────────────────────────────────────────────┐
│  2. 重新解析: ts_parser_parse(old_tree, new_input)       │
│                                                           │
│  初始化:                                                  │
│  ├── included_range_differences = diff(old_ranges, new)  │
│  ├── reusable_node_reset(old_tree.root)                  │
│  └── 开始主解析循环                                       │
│                                                           │
│  ts_parser__advance 中:                                   │
│  ├── ts_parser__reuse_node():                            │
│  │   沿旧树 DFS, 位置对齐时尝试复用                      │
│  │   ├── 可复用: 整个子树作为 lookahead, skip lex        │
│  │   └── 不可复用: descend/advance, 走 lexer             │
│  └── 正常 lex + shift/reduce                             │
└───────────────────┬──────────────────────────────────────┘
                    │  新 TSTree
                    ▼
┌──────────────────────────────────────────────────────────┐
│  3. 计算变化范围 (可选):                                  │
│     ts_tree_get_changed_ranges(old_tree, new_tree)        │
│     ├── included_ranges 差异                              │
│     └── 双 TreeCursor 并行遍历, 对比子树差异             │
└──────────────────────────────────────────────────────────┘
```

### 3.2 步骤详解

#### 步骤 1：编辑应用 (`ts_tree_edit`)

**文件**：`lib/src/tree.c:55-63`

```c
void ts_tree_edit(TSTree *self, const TSInputEdit *edit) {
  // 更新所有 included_ranges
  for (unsigned i = 0; i < self->included_range_count; i++) {
    ts_range_edit(&self->included_ranges[i], edit);
  }
  // 更新语法树节点
  SubtreePool pool = ts_subtree_pool_new(0);
  self->root = ts_subtree_edit(self->root, edit, &pool);
  ts_subtree_pool_delete(&pool);
}
```

**TSInputEdit 结构**：

```c
typedef struct TSInputEdit {
  uint32_t start_byte;      // 编辑起始位置
  uint32_t old_end_byte;    // 旧文本中编辑结束位置
  uint32_t new_end_byte;    // 新文本中编辑结束位置
  TSPoint start_point;      // 行列形式的起始位置
  TSPoint old_end_point;
  TSPoint new_end_point;
} TSInputEdit;
```

**`ts_range_edit`** (`get_changed_ranges.c:106-138`)：

更新 `included_ranges` 中每个 `TSRange` 的位置：
- 若 range 在 `old_end` 之后：整体平移 `new_end - old_end`
- 若与编辑区间相交：裁剪到 `start_*`

**`ts_subtree_edit`** (`subtree.c:633-786`)：

使用**显式栈**（非递归）迭代处理子树：

```
ts_subtree_edit(tree, edit, pool):
│
├── 栈元素: (subtree, edit{start, old_end, new_end})
│
├── 对每个节点 (自顶向下):
│   │
│   ├── 跳过: 编辑在子树 (padding+size+lookahead) 之后
│   │
│   ├── padding/size 调整:
│   │   ├── 编辑完全在 padding 内: 只平移 padding
│   │   ├── 编辑从 padding 跨进 content: 缩小 size, padding=new_end
│   │   └── 编辑在 content 内: 按差值调整 size
│   │
│   ├── ts_subtree_make_mut: COW (ref_count>1 时 clone)
│   │   inline 节点若调整后超出位域 → 升级为 heap 节点
│   │
│   ├── 写回 padding/size → 设置 has_changes
│   │
│   └── 处理子节点:
│       按 child_left/child_right 累加 ts_subtree_total_size
│       仅对与编辑相交的子节点继续下推
│       坐标变换: child_edit = edit - child_left
│
└── has_changes 传播: 被修改的每个节点都设置 has_changes
    祖先链上的所有节点都会被标记
```

**关键特性**：
- **不改变树结构**：只调整位置信息，不重新解析
- **显式栈防止溢出**：避免深树递归栈溢出
- **COW 语义**：`make_mut` 保证不修改共享的子树（旧树可能被其他引用持有）

#### 步骤 2：ReusableNode — 旧树遍历器

**文件**：`lib/src/reusable_node.h`

```c
typedef struct {
  const Subtree *subtree;    // 当前子树指针
  uint32_t byte_offset;      // 在文档中的起始字节
  uint32_t child_index;      // 在父节点中的索引
} StackEntry;

typedef struct {
  Array(StackEntry) stack;           // DFS 路径栈
  Subtree last_external_token;       // 最后一个含 external token 的子树
} ReusableNode;
```

**初始化** (`reusable_node_reset`)：
- 压入旧树根节点，但**不使用根节点本身**（根有 EOF/额外子节点等非标准结构）
- 下探到根的第一个子节点开始

**遍历操作**：

| 操作 | 行为 |
|------|------|
| `descend` | 有子节点则进入第一个子节点，`byte_offset` 不变 |
| `advance` | 跳过当前子树（`byte_offset += total_size`），沿栈找下一个兄弟；无兄弟则上溯 |
| `advance_past_leaf` | 反复 `descend` 到叶子再 `advance`（无法整块复用但要跳过第一片叶子时） |

#### 步骤 3：节点复用 (`ts_parser__reuse_node`)

**文件**：`lib/src/parser.c:753-830`

```
ts_parser__reuse_node(self, version, &state, position, last_external_token, &table_entry)
│
└── while reusable_node 有效:
    │
    ├── byte_offset > position:
    │   → break, 返回 NULL (旧树下一节点在当前位置之后, 走 lexer)
    │
    ├── byte_offset < position:
    │   → advance 或 descend (对齐位置)
    │
    ├── byte_offset == position:
    │   │
    │   ├── 检查 external_scanner_state 一致性:
    │   │   不一致 → advance, continue
    │   │
    │   ├── 不可复用检查 (任一命中则拒绝):
    │   │   ├── has_changes         编辑影响了此节点
    │   │   ├── is_error            错误节点
    │   │   ├── is_missing          缺失节点
    │   │   ├── is_fragile          脆弱节点 (fragile_left || fragile_right)
    │   │   └── 与 included_range_differences 相交
    │   │
    │   │   不可复用时:
    │   │   ├── descend (进入子节点继续找)
    │   │   └── descend 失败: advance + breakdown_top_of_stack
    │   │
    │   ├── 解析表验证:
    │   │   leaf_symbol = ts_subtree_leaf_symbol(result)
    │   │   ts_language_table_entry(state, leaf_symbol, &table_entry)
    │   │   ts_parser__can_reuse_first_leaf(state, result, table_entry):
    │   │   ├── 当前状态的 lex_mode 与节点产生时一致
    │   │   ├── table_entry.is_reusable == true
    │   │   ├── external_lex_state == 0 (无外部扫描器上下文依赖)
    │   │   └── 非空 token (避免无效叶子)
    │   │
    │   │   验证失败: advance_past_leaf, break
    │   │
    │   └── 复用成功:
    │       ts_subtree_retain(result)
    │       return result  (整个子树作为 lookahead token!)
    │
    └── 返回 NULL_SUBTREE (无法复用, 走 lexer)
```

**复用后的处理**：
- 复用的子树直接作为 `lookahead` 进入 Shift 操作
- 若是非叶节点：`ts_parser__breakdown_lookahead` 将其分解，只取第一个叶子做 lookahead，其余子节点放回 `ReusableNode`
- Shift 成功后：`reusable_node_advance()` 前进到下一个旧树节点

**"只读游标"模式**：
- 旧树仅作为只读参考
- 解析器在能复用的区域直接跳过 lexer
- 在不能复用的区域（`has_changes` 等）退回到正常 lex → shift/reduce 流程
- 没有显式的"开始/停止重解析"标志——**复用失败即重解析，复用成功即跳过**

**`allow_node_reuse` 的限制**：
- 仅在 `version_count == 1` 时启用
- 多 GLR 版本并行时关闭复用，避免因复用导致版本间语义不一致

#### 步骤 4：重解析边界的确定

Tree-sitter 增量解析没有显式的"重解析区域"计算，而是通过以下机制**隐式**确定边界：

```
编辑影响链:
                    编辑点
                      ↓
    ┌──────────────────────────────────────┐
    │            旧语法树                   │
    │  ┌───────┐  ┌───────┐  ┌──────────┐ │
    │  │ 未变  │  │has_chg│  │  未变    │ │
    │  │ 可复用 │  │不可复用│  │  可复用  │ │
    │  └───────┘  └───────┘  └──────────┘ │
    └──────────────────────────────────────┘

解析进度:
    ═══复用═══  ──重新解析──  ═══复用═══
```

**"起点"**：逻辑上从文档开头开始解析。实际上，`reusable_node` 在编辑点之前的节点都能被复用（`byte_offset == position` 且无 `has_changes`），解析器直接跳过这些区域。真正的重新 lex 从第一个不能复用的节点开始。

**"终点"**：当 `reusable_node` 在编辑点之后重新对齐位置（`byte_offset == position`），且该子树通过所有复用检查，解析器恢复复用模式。这相当于增量解析的"收敛"——新解析的结果与旧树在此处汇合。

**`has_changes` 的传播**：
- `ts_subtree_edit` 沿编辑影响链向下传播 `has_changes`
- 被标记的节点及其所有祖先都会被标记
- 复用时 `has_changes` 一票否决，迫使解析器在该区域下探或前进

#### 步骤 5：变化范围计算 (`ts_tree_get_changed_ranges`)

**文件**：`lib/src/tree.c:72-94`、`lib/src/get_changed_ranges.c`

解析完成后，用户可调用此函数获取新旧树之间的差异范围（用于编辑器的增量重渲染）。

```
ts_tree_get_changed_ranges(old_tree, new_tree)
│
├── 1. included_ranges 差异:
│      ts_range_array_get_changed_ranges(old_ranges, new_ranges)
│      merge-style 扫描两个排序的 range 数组
│
└── 2. 子树差异 (双 Iterator 并行遍历):
       ts_subtree_get_changed_ranges(old_root, new_root, ...)
       │
       └── 主循环:
           初始化两个 Iterator (基于 TreeCursor)
           │
           └── while 两个 Iterator 都未结束:
               │
               ├── iterator_compare(old, new) → 比较结果:
               │
               ├── IteratorMatches:
               │   子树相同 (符号/大小/parse_state/error_cost 等一致)
               │   → 跳过, next_position = 可见段结束
               │   特殊: 若与 included_range_differences 相交
               │         → 升级为 IteratorMayDiffer
               │
               ├── IteratorMayDiffer:
               │   外层相似但内部可能不同
               │   → 双方 iterator_descend (进入子节点继续比较)
               │   一侧无法下探 → 标记 changed
               │
               └── IteratorDiffers:
                   确定不同 (符号/alias/缺一不同)
                   → 标记 changed
                   next_position = min(old_end, new_end)
               │
               advance 两个 Iterator 到 next_position
               拉齐 visible_depth
               若 changed: ts_range_array_add 合并连续区间
```

**`iterator_compare` 比较维度**：

| 维度 | 说明 |
|------|------|
| 符号 (symbol) | 不同 → `Differs` |
| alias | 不同 → `Differs` |
| 一侧为空 | → `Differs` |
| 起始位置 | 不同 → `MayDiffer` |
| size | 不同 → `MayDiffer` |
| parse_state | 不同 → `MayDiffer` |
| error_cost | 不同 → `MayDiffer` |
| has_external_tokens | 不同 → `MayDiffer` |
| has_changes | 为 true → `MayDiffer` |
| external_scanner_state | 不同 → `MayDiffer` |
| 全部相同 | → `Matches` |

### 3.3 增量解析完整示例

假设源码 `let x = 1;` 修改为 `let xy = 1;`（在 `x` 后插入 `y`）：

```
TSInputEdit = {
  start_byte: 5,       // 'x' 后面
  old_end_byte: 5,     // 插入操作, old_end == start
  new_end_byte: 6,     // 插入了 'y'
  start_point: (0,5),
  old_end_point: (0,5),
  new_end_point: (0,6)
}

步骤 1: ts_tree_edit
  旧树:  program
         └── variable_declaration
             ├── "let"              padding:0 size:3  → 未变
             ├── identifier "x"    padding:1 size:1  → has_changes! (size: 1→2)
             ├── "="               padding:1 size:1  → padding 平移
             ├── number "1"        padding:1 size:1  → padding 平移
             └── ";"               padding:0 size:1  → padding 平移

步骤 2: ts_parser_parse(old_tree, new_input)
  reusable_node 遍历:
  ├── "let"     byte_offset=0, position=0  → 无 has_changes → 复用! skip lex
  ├── "x"→"xy"  byte_offset=4, position=4  → has_changes → 不可复用!
  │             → descend 失败(叶子) → advance → 走 lexer
  │             → lex "xy" → new identifier subtree
  ├── "="       byte_offset=5→6, position=6 → 位置对齐, 无 has_changes → 复用!
  ├── "1"       byte_offset=7→8, position=8 → 复用!
  └── ";"       byte_offset=8→9, position=9 → 复用!

  结果: 只有 identifier 节点被重新 lex + 构建, 其余 4 个节点直接复用

步骤 3: ts_tree_get_changed_ranges
  输出: [{start_byte:4, end_byte:6}]  // 只有 identifier 区域变化
  编辑器只需重新渲染这一小段
```

---

## 附录：核心函数速查表

### Grammar 编译流程

| 步骤 | 函数 | 文件 |
|------|------|------|
| JS 加载 | `load_grammar_file` / `load_js_grammar_file` | `crates/generate/src/generate.rs` |
| QuickJS 执行 | `execute_native_runtime` | `crates/generate/src/quickjs.rs` |
| JSON 解析 | `parse_grammar` | `crates/generate/src/parse_grammar.rs` |
| 语法预处理 | `prepare_grammar` | `crates/generate/src/prepare_grammar.rs` |
| 符号内化 | `intern_symbols` | `prepare_grammar/intern_symbols.rs` |
| Token 提取 | `extract_tokens` | `prepare_grammar/extract_tokens.rs` |
| Repeat 展开 | `expand_repeats` | `prepare_grammar/expand_repeats.rs` |
| 产生式展平 | `flatten_grammar` | `prepare_grammar/flatten_grammar.rs` |
| NFA 构建 | `expand_tokens` | `prepare_grammar/expand_tokens.rs` |
| 默认别名 | `extract_default_aliases` | `prepare_grammar/extract_default_aliases.rs` |
| Inline 处理 | `process_inlines` | `prepare_grammar/process_inlines.rs` |
| 解析表构建 | `build_parse_table` | `build_tables/build_parse_table.rs` |
| LR 闭包 | `transitive_closure` | `build_tables/item_set_builder.rs` |
| 冲突处理 | `handle_conflict` | `build_tables/build_parse_table.rs` |
| 词法表构建 | `build_lex_table` | `build_tables/build_lex_table.rs` |
| 状态最小化 | `minimize_parse_table` | `build_tables/minimize_parse_table.rs` |
| C 代码渲染 | `render_c_code` / `Generator::generate` | `crates/generate/src/render.rs` |
| 动态库加载 | `Loader` | `crates/loader/src/loader.rs` |

### 运行时解析流程

| 步骤 | 函数 | 文件 |
|------|------|------|
| 解析入口 | `ts_parser_parse` | `lib/src/parser.c:2074` |
| 单步推进 | `ts_parser__advance` | `lib/src/parser.c:1557` |
| 词法分析 | `ts_parser__lex` | `lib/src/parser.c:505` |
| 主词法函数 | `ts_parser__call_main_lex_fn` | `lib/src/parser.c:341` |
| 关键词词法 | `ts_parser__call_keyword_lex_fn` | `lib/src/parser.c:349` |
| 外部扫描器 | `ts_parser__external_scanner_scan` | `lib/src/parser.c:443` |
| Shift | `ts_parser__shift` | `lib/src/parser.c:908` |
| Reduce | `ts_parser__reduce` | `lib/src/parser.c:931` |
| Accept | `ts_parser__accept` | `lib/src/parser.c:1048` |
| 错误处理入口 | `ts_parser__handle_error` | `lib/src/parser.c:1439` |
| 错误恢复 | `ts_parser__recover` | `lib/src/parser.c:1250` |
| 回溯恢复 | `ts_parser__recover_to_state` | `lib/src/parser.c:1191` |
| 栈收缩 | `ts_parser__condense_stack` | `lib/src/parser.c:1766` |
| 版本比较 | `ts_parser__compare_versions` | `lib/src/parser.c` |
| 择优选择 | `ts_parser__select_tree` | `lib/src/parser.c:836` |
| 树平衡 | `ts_parser__balance_subtree` | `lib/src/parser.c:1867` |
| 解析表查询 | `ts_language_table_entry` | `lib/src/language.c` |
| GOTO 查询 | `ts_language_next_state` | `lib/src/language.c:142` |
| 叶子创建 | `ts_subtree_new_leaf` | `lib/src/subtree.c:166` |
| 节点创建 | `ts_subtree_new_node` | `lib/src/subtree.c:480` |
| 缓存查询 | `ts_parser__get_cached_token` | `lib/src/parser.c:703` |
| 缓存写入 | `ts_parser__set_cached_token` | `lib/src/parser.c:724` |
| 栈压入 | `ts_stack_push` | `lib/src/stack.c:512` |
| 栈弹出 | `ts_stack_pop_count` | `lib/src/stack.c` |
| 栈合并 | `ts_stack_merge` | `lib/src/stack.c` |

### 增量解析流程

| 步骤 | 函数 | 文件 |
|------|------|------|
| 树编辑 | `ts_tree_edit` | `lib/src/tree.c:55` |
| 子树编辑 | `ts_subtree_edit` | `lib/src/subtree.c:633` |
| Range 编辑 | `ts_range_edit` | `lib/src/get_changed_ranges.c:106` |
| 复用节点遍历 | `reusable_node_reset/descend/advance` | `lib/src/reusable_node.h` |
| 节点复用判断 | `ts_parser__reuse_node` | `lib/src/parser.c:753` |
| 首叶可复用 | `ts_parser__can_reuse_first_leaf` | `lib/src/parser.c:470` |
| Range 差异 | `ts_parser__has_included_range_difference` | `lib/src/parser.c` |
| 变化范围计算 | `ts_tree_get_changed_ranges` | `lib/src/tree.c:72` |
| 子树差异比较 | `ts_subtree_get_changed_ranges` | `lib/src/get_changed_ranges.c:413` |
| 迭代器比较 | `iterator_compare` | `lib/src/get_changed_ranges.c:342` |

---

*文档生成时间：2026-03-27*
*基于 Tree-sitter v0.27.0 源码分析*
*对不确定的地方已标注【待确认】*
