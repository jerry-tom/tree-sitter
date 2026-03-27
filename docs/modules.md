# Tree-sitter 核心模块深度解析

> 基于 Tree-sitter v0.27.0 源码分析
> 依赖文档：[architecture.md](./architecture.md)

---

## 目录

1. [Parser 模块](#1-parser-模块)
2. [Lexer 模块](#2-lexer-模块)
3. [Stack 模块](#3-stack-模块)
4. [Subtree 模块](#4-subtree-模块)
5. [Tree 模块](#5-tree-模块)
6. [Node 模块](#6-node-模块)
7. [TreeCursor 模块](#7-treecursor-模块)
8. [Language 模块](#8-language-模块)
9. [Query 模块](#9-query-模块)
10. [Generate 模块](#10-generate-模块)
11. [Highlight 模块](#11-highlight-模块)
12. [Tags 模块](#12-tags-模块)
13. [Loader 模块](#13-loader-模块)
14. [CLI 模块](#14-cli-模块)

---

## 1. Parser 模块

**文件**：`lib/src/parser.c`、`lib/src/parser.h`

### 1.1 边界

| 维度 | 说明 |
|------|------|
| **输入** | `TSInput`（读取回调）、`TSLanguage`（语法定义）、`TSTree`（旧树，增量解析用）、`TSParseOptions`（进度回调） |
| **输出** | `TSTree *`（语法树）；被取消时返回 NULL |
| **对外接口** | `ts_parser_new/delete`、`ts_parser_set_language`、`ts_parser_parse`、`ts_parser_parse_with_options`、`ts_parser_parse_string`、`ts_parser_reset`、`ts_parser_set_logger`、`ts_parser_set_included_ranges`、`ts_parser_print_dot_graphs` |

### 1.2 内聚性

Parser 模块职责集中于"驱动 GLR 解析过程"，约 2260 行代码，承担以下子职责：
- GLR 解析调度（主循环）
- 词法分析协调（调用 Lexer/外部扫描器）
- Shift/Reduce/Accept 操作
- 错误恢复（回溯 + 跳过）
- 增量解析（节点复用）
- 多版本栈管理（condense/merge/compare）
- 树平衡（重复节点的 AVL 式压缩）

**内聚性评价**：整体较好。底层细节良好地委托给了 Lexer、Stack、Subtree 等模块。`ts_parser__lex` 函数约 200 行，混合了外部扫描器调用、内部词法分析、关键字识别等逻辑，是模块内最复杂的函数。

### 1.3 关键数据结构

#### TSParser 内部字段

```c
struct TSParser {
  Lexer lexer;                      // 词法分析器实例（嵌入，非指针）
  Stack *stack;                     // GLR 多版本解析栈
  SubtreePool tree_pool;            // 子树内存池
  const TSLanguage *language;       // 当前语法定义
  ReduceActionSet reduce_actions;   // 临时归约动作集合
  Subtree finished_tree;            // 已完成的语法树根
  SubtreeArray trailing_extras;     // Reduce 时暂存的尾部 extra 节点
  SubtreeArray trailing_extras2;    // 多路径选择的备用 extra 暂存
  SubtreeArray scratch_trees;       // select_children 时的临时数组
  TokenCache token_cache;           // 最近一次词法分析缓存
  ReusableNode reusable_node;       // 旧树遍历器（增量解析）
  void *external_scanner_payload;   // 外部扫描器实例
  FILE *dot_graph_file;             // DOT 图调试输出
  unsigned accept_count;            // 已接受的解析结果数量
  unsigned operation_count;         // 操作计数器（progress_callback 节流）
  Subtree old_tree;                 // 旧语法树根
  TSRangeArray included_range_differences; // 新旧 included_ranges 差异
  TSParseOptions parse_options;     // 解析选项
  TSParseState parse_state;         // 当前解析状态（传给 progress_callback）
  bool has_scanner_error;           // 外部扫描器错误标志
  bool canceled_balancing;          // 树平衡是否被中断
  bool has_error;                   // 所有栈版本是否都处于错误状态
};
```

#### TSParseAction（联合体）

```c
typedef union {
  struct { uint8_t type; TSStateId state; bool extra; bool repetition; } shift;
  struct { uint8_t type; uint8_t child_count; TSSymbol symbol;
           int16_t dynamic_precedence; uint16_t production_id; } reduce;
  uint8_t type;  // Accept / Recover
} TSParseAction;
```

四种动作类型：**Shift**（压栈转状态）、**Reduce**（弹出子节点创建父节点）、**Accept**（解析完成）、**Recover**（进入错误恢复）。

#### TokenCache

```c
typedef struct {
  Subtree token;                // 缓存的 token
  Subtree last_external_token;  // 缓存时的外部 token 状态
  uint32_t byte_index;          // 缓存位置
} TokenCache;
```

### 1.4 关键算法

#### 主循环 (`ts_parser_parse`)

```
初始化 → 创建外部扫描器 → 设置 reusable_node
do:
  for each active stack version:
    while version is active:
      ts_parser__advance(version, allow_node_reuse)
      if position advanced: break  // 让其他版本也推进
  min_error_cost = ts_parser__condense_stack()  // 收缩栈版本
  if finished_tree.error_cost < min_error_cost: break  // 已有最优解
while (有活跃栈版本)
ts_parser__balance_subtree()  // 平衡重复节点
→ 返回 TSTree
```

#### Shift 操作

将 lookahead token 压入指定版本的栈顶，更新解析状态。若 token 是来自旧树复用的非叶节点，标记为 pending。

#### Reduce 操作

1. 弹出 `child_count` 个节点（GLR 场景可能有多条路径）
2. 分离尾部 extra 节点
3. 创建新的父节点（`ts_subtree_new_node`）
4. 多路径时通过 `select_children` 选择最优子节点集合
5. 查 GOTO 表确定下一状态
6. 尝试与已有版本合并

#### 错误恢复

**两阶段策略**：

1. **错误检测** (`ts_parser__handle_error`)：
   - 尝试无视 lookahead 做所有可能的 reduce
   - 尝试插入 MISSING token
   - 所有版本推入 ERROR_STATE 并合并
   - 记录栈摘要（用于回溯恢复）

2. **错误恢复** (`ts_parser__recover`)：
   - **策略 1 — 回溯**：搜索栈摘要，找到对当前 lookahead 有效的历史状态，弹出中间层包装为 ERROR 节点
   - **策略 2 — 跳过**：将当前 token 包装为 error_repeat 节点，推入 ERROR_STATE

#### 增量解析 — 节点复用 (`ts_parser__reuse_node`)

```
while reusable_node 有效:
  if byte_offset != position: descend/advance, continue
  不可复用的情况: has_changes / is_error / is_missing / is_fragile / 包含 range 变化
  可复用: leaf_symbol 在当前状态有效 → retain 并返回
```

#### 多版本栈收缩 (`ts_parser__condense_stack`)

对所有版本两两比较 ErrorStatus（错误代价、动态优先级、是否在错误中），剪枝劣势版本。硬上限：最多 `MAX_VERSION_COUNT = 6` 个版本。

### 1.5 设计模式

| 模式 | 体现 |
|------|------|
| **策略模式** | `TSInput.read`、`TSLanguage.lex_fn`、`external_scanner.scan` |
| **观察者模式** | `TSLogger`、`progress_callback` |
| **对象池模式** | `SubtreePool` |
| **迭代器模式** | `ReusableNode`（旧树遍历器） |
| **模板方法模式** | `ts_parser_parse` 定义解析骨架，具体步骤委托私有方法 |

### 1.6 交互

```
Parser ──→ Lexer:    ts_lexer_reset → ts_lexer_start → lex_fn → ts_lexer_finish
Parser ──→ Stack:    push / pop_count / pop_all / merge / copy_version / halt / pause
Parser ──→ Subtree:  new_leaf / new_node / new_error / retain / release / make_mut / compare
Parser ──→ Language: table_entry / next_state / lex_mode_for_state / symbol_name
```

### 1.7 扩展点

- **TSLogger**：日志回调，支持 `TSLogTypeParse` 和 `TSLogTypeLex`
- **progress_callback**：每 100 次操作检查一次，返回 true 可取消解析，支持中断后恢复
- **External Scanner**：语法作者自定义词法扫描器（create/destroy/scan/serialize/deserialize）
- **自定义内存分配器**：`ts_set_allocator` 全局替换 malloc/free
- **DOT 图输出**：`ts_parser_print_dot_graphs` 设置调试输出文件

### 关键常量

| 常量 | 值 | 用途 |
|------|------|------|
| `MAX_VERSION_COUNT` | 6 | GLR 栈最大版本数 |
| `MAX_SUMMARY_DEPTH` | 16 | 错误恢复栈摘要最大深度 |
| `MAX_COST_DIFFERENCE` | 1800 | 版本强制删除的代价差阈值 |
| `ERROR_COST_PER_RECOVERY` | 500 | 每次错误恢复的代价 |
| `ERROR_COST_PER_MISSING_TREE` | 110 | 每个 missing 节点的代价 |
| `ERROR_COST_PER_SKIPPED_TREE` | 100 | 每个跳过的树的代价 |

---

## 2. Lexer 模块

**文件**：`lib/src/lexer.c`、`lib/src/lexer.h`

### 2.1 边界

| 维度 | 说明 |
|------|------|
| **输入** | `TSInput`（读取回调）、`TSLanguage.lex_fn`（词法状态机）、`TSRange[]`（included_ranges） |
| **输出** | Token 描述信息（`result_symbol`、起止位置），由 Parser 读取后创建 Subtree |
| **对外接口** | `ts_lexer_init/delete`、`ts_lexer_set_input`、`ts_lexer_reset`、`ts_lexer_start`、`ts_lexer_finish`、`ts_lexer_mark_end`、`ts_lexer_set_included_ranges` |

### 2.2 内聚性

**高**。职责单一：从字节流中读取解码 Unicode 字符，为 lex_fn 提供逐字符的前瞻能力。

### 2.3 关键数据结构

#### Lexer 结构体

```c
typedef struct {
  TSLexer data;                     // 回调接口（lookahead, result_symbol, advance, mark_end 等）
  Length current_position;          // 当前读取位置
  Length token_start_position;      // token 起始位置
  Length token_end_position;        // token 结束位置（由 mark_end 设定）
  TSRange *included_ranges;         // 有效源码范围数组
  const char *chunk;                // 当前源码片段指针
  TSInput input;                    // 用户输入回调
  TSLogger logger;                  // 日志回调
  uint32_t included_range_count;
  uint32_t current_included_range_index;
  uint32_t chunk_start;             // chunk 在源码中的起始偏移
  uint32_t chunk_size;
  uint32_t lookahead_size;          // 当前前瞻字符的字节长度
  bool did_get_column;
  ColumnData column_data;           // 列号缓存（避免重复计算）
  char debug_buffer[1024];
} Lexer;
```

#### TSLexer 回调接口（虚函数表）

```c
struct TSLexer {
  int32_t lookahead;        // 当前前瞻字符（Unicode 码点）
  TSSymbol result_symbol;   // lex_fn 设置的匹配 token 符号
  void (*advance)(TSLexer *, bool skip);  // 前进一个字符
  void (*mark_end)(TSLexer *);            // 标记 token 结束位置
  uint32_t (*get_column)(TSLexer *);      // 获取 Unicode 列号
  bool (*is_at_included_range_start)(const TSLexer *);
  bool (*eof)(const TSLexer *);
  void (*log)(const TSLexer *, const char *, ...);
};
```

**设计意图**：生成的 parser 代码只依赖 `TSLexer` 结构体定义（在 `parser.h` 中），不需要链接 tree-sitter 库符号。lex_fn 通过 `lexer->advance(lexer, skip)` 调用，类似 C++ 虚函数。`TSLexer` 是 `Lexer` 的第一个字段，所以 `(Lexer *)lexer` 的类型转换是安全的（C 标准保证首成员地址对齐）。

### 2.4 关键算法

- **advance 操作**：更新位置（处理换行/列号），检查 included_range 边界（超出当前 range 时自动跳转到下一个 range），重新加载 chunk 并解码下一个前瞻字符
- **UTF 解码**：支持 UTF-8/UTF-16LE/UTF-16BE/自定义解码。跨 chunk 边界的多字节字符特殊处理
- **mark_end**：标记当前位置为 token 结束位置，lex_fn 可多次调用（每次找到更长匹配时更新）
- **included_ranges 处理**：ranges 间隙对 lex_fn 透明——advance 时自动跳过间隙

### 2.5 设计模式

**回调模式（虚函数表）**：TSLexer 的函数指针在 `ts_lexer_init` 中设置，生成的 parser 通过函数指针调用。

### 2.6 交互

- **与 Parser**：Parser 持有 Lexer 实例（嵌入），控制 lex 生命周期（reset → start → lex_fn → finish）
- **与 Language**：Language 提供 lex_fn / keyword_lex_fn
- **与 Unicode**：依赖 `ts_decode_utf8/utf16_le/utf16_be` 解码函数

---

## 3. Stack 模块

**文件**：`lib/src/stack.c`、`lib/src/stack.h`

### 3.1 边界

| 维度 | 说明 |
|------|------|
| **输入** | `Subtree`（语法树节点）、`TSStateId`（解析状态）、`SubtreePool` |
| **输出** | `StackSliceArray`（pop 返回的切片数组）、版本状态查询 |
| **对外接口** | `ts_stack_new/delete`、`ts_stack_push`、`ts_stack_pop_count/pop_all/pop_pending/pop_error`、`ts_stack_merge/copy_version/remove_version`、`ts_stack_halt/pause/resume`、`ts_stack_state/position/error_cost/dynamic_precedence`、`ts_stack_record_summary/get_summary` |

### 3.2 关键数据结构

#### StackNode（链表节点）

```c
struct StackNode {
  TSStateId state;            // 解析状态
  Length position;            // 文档位置
  StackLink links[8];         // 前驱链接（最多 8 条，支持 GLR 分叉）
  short unsigned int link_count;
  uint32_t ref_count;         // 引用计数
  unsigned error_cost;        // 累计错误代价
  unsigned node_count;        // 累计节点数
  int dynamic_precedence;     // 累计动态优先级
};
```

#### StackLink（边）

```c
typedef struct {
  StackNode *node;      // 前驱节点
  Subtree subtree;      // 边上的语法节点
  bool is_pending;      // 是否为 pending 状态
} StackLink;
```

#### StackHead（版本头）

```c
typedef struct {
  StackNode *node;                       // 栈顶节点
  StackSummary *summary;                 // 状态摘要（错误恢复用）
  unsigned node_count_at_last_error;
  Subtree last_external_token;
  Subtree lookahead_when_paused;
  StackStatus status;                    // Active / Paused / Halted
} StackHead;
```

#### Stack 结构

```c
struct Stack {
  Array(StackHead) heads;       // 版本头数组（索引即版本号）
  StackSliceArray slices;       // pop 结果缓冲
  Array(StackIterator) iterators; // 遍历迭代器缓冲
  StackNodeArray node_pool;     // StackNode 对象池（最多 50 个）
  StackNode *base_node;         // 所有版本共享的栈底
  SubtreePool *subtree_pool;    // Subtree 内存池引用
};
```

### 3.3 关键算法

- **Push**：创建新 StackNode，前驱指向当前 head->node，继承并累加前驱的 position/error_cost/node_count/dynamic_precedence
- **Pop（基于 `stack__iter`）**：通用遍历器，沿链接回溯收集 Subtree。在 GLR 分叉点分裂迭代器，返回多个 StackSlice
- **Split (`copy_version`)**：浅拷贝 StackHead，新旧版本共享底层 StackNode 链（引用计数管理）
- **Merge**：合并条件 = 同状态 + 同位置 + 同错误代价 + 同外部 token 状态。将 v2 的 links 添加到 v1 的栈顶节点，删除 v2
- **智能链接合并 (`stack_node_add_link`)**：等价 subtree 检测 → 保留动态优先级更高的；同状态前驱 → 递归合并

### 3.4 设计模式

**Graph-Structured Stack (GSS)**：经典 GLR 数据结构。StackNode 通过多条 link 形成有向无环图，多个 StackHead 共享底层子图。

```
Head0 → [NodeA] ─link0─→ [NodeC] ──→ [base]
Head1 → [NodeB] ─link0─→ [NodeC]    (NodeC 被共享)
                 ╲link1─→ [NodeD] ──→ [base]  (NodeB 有两条 link = 歧义)
```

### 3.5 版本状态管理

| 状态 | 含义 | 转换 |
|------|------|------|
| **Active** | 正在活跃解析 | 初始状态；由 resume 恢复 |
| **Paused** | 暂停等待恢复（保存 lookahead） | `ts_stack_pause` 设置 |
| **Halted** | 永久停止（完成或被淘汰） | `ts_stack_halt` 设置，不可恢复 |

---

## 4. Subtree 模块

**文件**：`lib/src/subtree.c`、`lib/src/subtree.h`

### 4.1 边界

| 维度 | 说明 |
|------|------|
| **输入** | TSSymbol、Length、TSStateId、TSInputEdit、子节点数组 |
| **输出** | Subtree/MutableSubtree 节点、S-expression 字符串 |
| **对外接口** | `ts_subtree_new_leaf/new_node/new_error/new_error_node/new_missing_leaf`、`ts_subtree_retain/release/make_mut`、`ts_subtree_edit/compress/summarize_children`、`ts_subtree_compare/string/print_dot_graph` |

### 4.2 关键数据结构（重点）

#### Subtree — Tagged Union

```c
typedef union {
  SubtreeInlineData data;       // inline 路径（8 字节，值类型）
  const SubtreeHeapData *ptr;   // heap 路径（指针）
} Subtree;
```

**判别方式**：检查 `data.is_inline`。利用指针对齐特性——合法堆指针的 LSB 始终为 0，所以 `is_inline = 1` 不会与指针冲突。

#### SubtreeInlineData — 位域布局（小端序 64 位）

```
字节  位          字段              宽度   含义
───────────────────────────────────────────────────
0     bit 0       is_inline         1 bit  =1 inline / =0 指针
      bit 1       visible           1 bit  CST 中是否可见
      bit 2       named             1 bit  是否命名节点
      bit 3       extra             1 bit  是否 extra（注释等）
      bit 4       has_changes       1 bit  是否被编辑影响
      bit 5       is_missing        1 bit  是否 missing 节点
      bit 6       is_keyword        1 bit  是否关键字
1     bits 0-7    symbol            8 bit  符号 ID（≤ 255）
2-3   bits 0-15   parse_state       16 bit 解析状态 ID
4     bits 0-7    padding_columns   8 bit  前导空白列数
5     bits 0-3    padding_rows      4 bit  前导空白行数（≤ 15）
      bits 4-7    lookahead_bytes   4 bit  前瞻字节数（≤ 15）
6     bits 0-7    padding_bytes     8 bit  前导空白字节数
7     bits 0-7    size_bytes        8 bit  内容字节数
───────────────────────────────────────────────────
总计：8 字节 = sizeof(void*)
```

**Inline 条件**：symbol ≤ 255 且无外部 token 且 padding/size 各字段不超出位域范围且不跨行。

#### SubtreeHeapData

```c
typedef struct {
  volatile uint32_t ref_count;     // 引用计数（原子操作）
  Length padding;                  // 前导空白
  Length size;                     // 内容大小
  uint32_t lookahead_bytes;
  uint32_t error_cost;            // 错误代价（聚合值）
  uint32_t child_count;
  TSSymbol symbol;
  TSStateId parse_state;

  bool visible : 1;  bool named : 1;  bool extra : 1;
  bool fragile_left : 1;  bool fragile_right : 1;
  bool has_changes : 1;  bool has_external_tokens : 1;
  bool has_external_scanner_state_change : 1;
  bool depends_on_column : 1;  bool is_missing : 1;  bool is_keyword : 1;

  union {
    // 变体1: 非终结符（child_count > 0）
    struct {
      uint32_t visible_child_count, named_child_count, visible_descendant_count;
      int32_t dynamic_precedence;
      uint16_t repeat_depth, production_id;
      struct { TSSymbol symbol; TSStateId parse_state; } first_leaf;
    };
    // 变体2: 外部 token 叶子
    ExternalScannerState external_scanner_state;
    // 变体3: 错误叶子
    int32_t lookahead_char;
  };
} SubtreeHeapData;
```

#### 父节点内存布局

子节点数组和父节点 HeapData 在**同一块连续内存**中分配：

```
┌──────────┬──────────┬─────┬──────────┬─────────────────┐
│ child[0] │ child[1] │ ... │ child[n] │ SubtreeHeapData │
│ (8 bytes)│ (8 bytes)│     │ (8 bytes)│   (父节点数据)   │
└──────────┴──────────┴─────┴──────────┴─────────────────┘
                                        ↑ ptr 指向这里
```

子节点通过指针回退访问：`(Subtree *)(ptr) - child_count`。

#### SubtreePool

```c
typedef struct {
  MutableSubtreeArray free_trees;   // 空闲 HeapData 缓存（最多 32 个）
  MutableSubtreeArray tree_stack;   // release/compare 的工作栈（复用）
} SubtreePool;
```

只池化叶子节点的 HeapData（大小固定）。父节点的变长内存（含子节点数组）不池化。

#### ExternalScannerState（SSO 模式）

```c
typedef struct {
  union { char *long_data; char short_data[24]; };
  uint32_t length;
} ExternalScannerState;
```

长度 ≤ 24 时 inline 存储，否则堆分配。

### 4.3 关键算法

- **引用计数**：`retain` 原子递增；`release` 原子递减，降为 0 时用显式栈（非递归）级联释放整棵子树
- **COW (`make_mut`)**：ref_count == 1 直接转换为可变；否则 clone（深拷贝 HeapData + 子节点数组，子节点 retain）
- **树平衡 (`compress`)**：类似 AVL 左旋，将右递归重复规则的 O(n) 深度链压缩为 O(log n) 的平衡树
- **子节点聚合 (`summarize_children`)**：从子节点重算父节点所有聚合属性（padding/size/error_cost/dynamic_precedence/visible_descendant_count 等）
- **编辑 (`edit`)**：使用显式栈迭代处理，调整 padding/size 并标记 `has_changes`。编辑不改变树结构，只调整位置信息

### 4.4 设计模式

| 模式 | 体现 |
|------|------|
| **Tagged Union** | inline/heap 通过 `is_inline` 位区分，零额外开销 |
| **对象池** | SubtreePool 缓存固定大小的 HeapData |
| **引用计数** | `volatile uint32_t ref_count` + 原子操作 |
| **COW** | `make_mut` 实现写时复制 |

### 4.5 设计亮点

| 亮点 | 说明 |
|------|------|
| 指针标签位 | 利用指针对齐将 8 字节 inline 数据和指针叠加在同一 union |
| 子节点内联存储 | 子节点数组和父节点 HeapData 在同一次分配中 |
| 平台适配位域 | 针对 LE/BE × 32/64 四种组合精确调整 `is_inline` 位的位置 |
| 无递归释放 | 显式栈防止深树栈溢出 |

---

## 5. Tree 模块

**文件**：`lib/src/tree.c`、`lib/src/tree.h`

### 5.1 边界

| 维度 | 说明 |
|------|------|
| **输入** | Subtree（根）、TSLanguage、TSRange[]、TSInputEdit |
| **输出** | TSTree、TSNode（根节点）、TSRange[]（变化范围） |
| **对外接口** | `ts_tree_new/copy/delete`、`ts_tree_root_node`、`ts_tree_edit`、`ts_tree_get_changed_ranges`、`ts_tree_language`、`ts_tree_included_ranges`、`ts_tree_print_dot_graph` |

### 5.2 关键数据结构

```c
struct TSTree {
  Subtree root;                  // 根子树
  const TSLanguage *language;    // 语言定义
  TSRange *included_ranges;      // 解析范围
  unsigned included_range_count;
};
```

注意：`tree.h` 中还定义了 `ParentCacheEntry`，但当前代码中未使用【待确认】，推测为历史遗留或未来优化保留。

### 5.3 关键算法

- **ts_tree_edit**：(1) 更新 included_ranges 的位置 (2) 调用 `ts_subtree_edit` 递归更新根子树。不重新解析，只修改位置信息并标记 `has_changes`
- **ts_tree_get_changed_ranges**：先计算 included_ranges 的差异（扫描线算法），再用两个 TreeCursor 同步遍历新旧子树对比差异

### 5.4 ChangedRanges 子模块 (`get_changed_ranges.c`)

**三种对比结果**：

| 结果 | 含义 | 处理 |
|------|------|------|
| IteratorMatches | 子树相同 | 跳过 |
| IteratorMayDiffer | 外层类似但内部可能不同 | 双方下降继续比较 |
| IteratorDiffers | 确定不同 | 记录变化范围 |

---

## 6. Node 模块

**文件**：`lib/src/node.c`

### 6.1 边界

| 维度 | 说明 |
|------|------|
| **输入** | TSTree（通过 TSNode 关联） |
| **输出** | TSNode（子节点/兄弟/祖先查询结果） |
| **对外接口** | `ts_node_type/symbol/start_byte/end_byte/start_point/end_point`、`ts_node_child/named_child/child_by_field_id/child_by_field_name`、`ts_node_next_sibling/prev_sibling/parent`、`ts_node_descendant_for_byte_range/point_range`、`ts_node_edit`、`ts_node_eq` |

### 6.2 关键数据结构

```c
typedef struct TSNode {
  uint32_t context[4];   // [start_byte, start_row, start_col, alias_symbol]
  const void *id;        // 指向 Subtree 的指针
  const TSTree *tree;    // 关联的 TSTree
} TSNode;
```

**值类型设计**：24 字节，栈分配。TSNode 是一个"视图"——不拥有数据，只引用 TSTree 中的 Subtree。`context[0..2]` 缓存起始位置，`context[3]` 存储 alias 符号。

### 6.3 关键算法

#### 隐藏节点展平

子节点遍历时自动跳过隐藏（不可见且无 alias）节点。如果目标索引落在隐藏节点的可见后代范围内，递归下降进去继续查找。效果：CST 内部的包装节点对 API 用户透明。

#### field 查询

通过 `production_id` 从语言定义中查找字段映射表（`field_map_slices` + `field_map_entries`），二分定位目标 field_id 对应的映射范围，然后用 `structural_child_index` 匹配。支持 `inherited` 标记（字段继承自隐藏子节点）。

#### 后代查找

自顶向下贪心搜索：从当前节点反复下降，找到完全包含目标范围的最深可见祖先。

---

## 7. TreeCursor 模块

**文件**：`lib/src/tree_cursor.c`、`lib/src/tree_cursor.h`

### 7.1 关键数据结构

```c
typedef struct {
  const Subtree *subtree;
  Length position;
  uint32_t child_index;
  uint32_t structural_child_index;
  uint32_t descendant_index;       // 可见后代中的全局索引
} TreeCursorEntry;

typedef struct {
  const TSTree *tree;
  Array(TreeCursorEntry) stack;    // 从根到当前位置的路径栈
  TSSymbol root_alias_symbol;
} TreeCursor;
```

### 7.2 TreeCursorStep

```c
enum { TreeCursorStepNone, TreeCursorStepHidden, TreeCursorStepVisible };
```

- **Visible**：API 可见节点，用户最终看到的
- **Hidden**：CST 内部包装节点，cursor 必须继续穿透
- **None**：无法继续移动

### 7.3 关键算法

- **goto_first_child**：遍历子节点，找到第一个 visible 或有可见后代的节点。Hidden 节点会继续自动下降
- **goto_next_sibling**：Pop 当前节点，在父节点子数组中搜索下一个 visible/有可见后代的兄弟。找不到则回溯到祖先继续搜索
- **goto_parent**：栈底向上扫描，截断到第一个可见祖先
- **goto_first_child_for_byte**：线性扫描子节点，找到 `end_byte > goal_byte` 的第一个节点

---

## 8. Language 模块

**文件**：`lib/src/language.c`、`lib/src/language.h`、`lib/src/parser.h`（TSLanguage 定义）

### 8.1 TSLanguage 结构概要

TSLanguage 是代码生成器产出的静态数据结构，包含：

| 字段组 | 关键字段 | 说明 |
|--------|----------|------|
| **计数** | `symbol_count`, `token_count`, `state_count`, `field_count` | 各种元素数量 |
| **解析表** | `parse_table`（大表，二维数组）、`small_parse_table`（小表，压缩格式）、`parse_actions` | 表驱动解析核心 |
| **符号/别名** | `symbol_names`, `symbol_metadata`, `public_symbol_map`, `alias_sequences` | 符号元信息 |
| **字段** | `field_names`, `field_map_slices`, `field_map_entries` | 字段映射 |
| **词法** | `lex_modes`, `lex_fn`, `keyword_lex_fn`, `keyword_capture_token` | 词法状态机 |
| **外部扫描器** | `external_scanner.{create,destroy,scan,serialize,deserialize}` | 自定义扫描器 |
| **v15+** | `name`, `reserved_words`, `supertype_*`, `metadata` | 新增元数据 |

### 8.2 双格式解析表查找

```c
static inline uint16_t ts_language_lookup(const TSLanguage *self, TSStateId state, TSSymbol symbol) {
  if (state >= self->large_state_count) {
    // 小表：通过 small_parse_table_map 定位，线性搜索分组
  } else {
    // 大表：parse_table[state * symbol_count + symbol] 直接索引
  }
}
```

- **大表**：O(1) 查找，空间 O(state × symbol)，适用于转移密集状态
- **小表**：相同动作的符号分组存储，空间节省但查找 O(n)

### 8.3 ABI 兼容

- v < 15：`TSLexMode` 无 `reserved_word_set_id`，需转换
- v < 14：`primary_state_ids` 不存在，总返回 true
- v < 15：`name/metadata/supertypes` 不存在，返回 NULL/空

---

## 9. Query 模块

**文件**：`lib/src/query.c`

### 9.1 边界

| 维度 | 说明 |
|------|------|
| **输入** | S-表达式查询字符串 + TSLanguage |
| **输出** | TSQuery（编译后的不可变查询） |
| **对外接口** | `ts_query_new/delete`、`ts_query_pattern/capture/string_count`、`ts_query_disable_pattern/capture`、`ts_query_cursor_new/delete/exec/next_match/next_capture`、`ts_query_cursor_set_byte_range/point_range/match_limit/max_start_depth` |

### 9.2 关键数据结构

#### TSQuery 内部

```c
struct TSQuery {
  SymbolTable captures;              // 捕获名双向映射
  SymbolTable predicate_values;      // 谓词字符串值映射
  Array(QueryStep) steps;            // 所有 pattern 的步骤展平数组
  Array(PatternEntry) pattern_map;   // 按首步 symbol 排序的索引
  Array(TSQueryPredicateStep) predicate_steps;
  Array(QueryPattern) patterns;
  const TSLanguage *language;
  uint16_t wildcard_root_pattern_count;
};
```

**核心设计**：所有 pattern 的 step 展平在一个连续数组中，每个 pattern 通过 `Slice(offset, length)` 引用。

#### QueryStep

```c
typedef struct {
  TSSymbol symbol;                    // 要匹配的符号（0 = 通配符）
  TSFieldId field;                    // 要求的字段
  uint16_t capture_ids[4];            // 捕获 ID 列表
  uint16_t depth;                     // 模式树中的深度
  uint16_t alternative_index;         // 分支/可选/重复的备选步骤
  bool is_pass_through : 1;           // 循环回跳点
  bool is_dead_end : 1;               // 跳转步骤
  bool root_pattern_guaranteed : 1;   // 静态分析：匹配是否确定
  // ... 其他标志
} QueryStep;
```

**分支/重复/可选**通过 `alternative_index` 形成**步骤图**（类似 NFA 的 ε-转移）。

### 9.3 关键算法

- **S-表达式解析**：手写递归下降解析器（`ts_query__parse_pattern`）
- **静态分析**（`ts_query__analyze_patterns`）：模拟解析表状态转移，标记 `root_pattern_guaranteed`。允许 `next_capture` 在匹配未完成时提前返回确定的捕获（流式高亮的关键）
- **匹配执行**：深度优先遍历 + 多状态并行推进（NFA 模拟器），每个 `QueryState` 代表一个进行中的匹配尝试
- **match vs capture**：`next_match` 等待完整匹配；`next_capture` 按文档顺序返回，可提前返回确定的捕获

### 9.4 性能优化

| 优化 | 机制 |
|------|------|
| pattern_map 二分查找 | O(log n) 查找匹配 pattern |
| root_pattern_guaranteed | `next_capture` 提前返回 |
| CaptureListPool | 对象池 |
| 步骤展平存储 | 缓存友好 |

### 9.5 设计模式

- **NFA 模拟器**：步骤图中的 `alternative_index` 相当于 ε-转移
- **迭代器模式**：`next_match` / `next_capture` 流式返回结果
- **编译时/运行时分离**：TSQuery 编译后不可变且线程安全，TSQueryCursor 有状态单线程使用

---

## 10. Generate 模块

**文件**：`crates/generate/src/`

### 10.1 边界

| 维度 | 说明 |
|------|------|
| **输入** | `grammar.js` 或 `grammar.json` |
| **输出** | `parser.c`（完整 C 解析器）、`node-types.json`、头文件 |
| **对外接口** | `generate_parser_in_directory`、`generate_parser_for_grammar_with_opts` |

### 10.2 数据结构演变

```
InputGrammar (rules: Vec<Variable<Rule>>)
    ↓ intern_symbols
InternedGrammar (rules: Vec<Variable<Rule>>, symbols: Symbol)
    ↓ extract_tokens
(ExtractedSyntaxGrammar, ExtractedLexicalGrammar)
    ↓ expand_repeats + flatten_grammar + expand_tokens
(SyntaxGrammar, LexicalGrammar)
    ↓ build_tables
Tables = ParseTable + LexTable(main) + LexTable(keyword)
    ↓ render_c_code
parser.c (String)
```

**关键类型**：

| 类型 | 说明 |
|------|------|
| `Rule` | 语法规则 AST：`Blank/String/Pattern/Symbol/Choice/Seq/Repeat/Metadata/Reserved` |
| `Production` | 压平后的产生式：`Vec<ProductionStep>` + `dynamic_precedence` |
| `Nfa/NfaState` | 词法 NFA：`Advance(chars→state)` / `Split(s1,s2)` / `Accept(var_idx)` |
| `ParseTable` | 解析表：`Vec<ParseState>`，每状态含终结符动作表 + 非终结符 GOTO 表 |
| `LexTable` | 词法表：`Vec<LexState>`，每状态含字符集转移 + 接受动作 |
| `TokenSet` | 位向量：高效表示 token 集合 |
| `CharacterSet` | Unicode 码点范围列表 |

### 10.3 关键算法

#### prepare_grammar 管道（6 步）

| 步骤 | 函数 | 变换 |
|------|------|------|
| 1 | `intern_symbols` | 字符串名 → 符号 ID |
| 2 | `extract_tokens` | 分离句法规则和词法规则 |
| 3 | `expand_repeats` | `Repeat(x)` → 辅助二叉树非终结符 |
| 4 | `flatten_grammar` | Rule AST → `Vec<Production>`（Choice 展开、Seq 展平、Metadata 下压到 step） |
| 5 | `expand_tokens` | 正则/字符串 → NFA（通过 regex-syntax 解析 HIR） |
| 6 | `extract_default_aliases` + `process_inlines` | 提取默认别名 + 构建 inline 展开映射 |

#### LR(1) 项目集构造 (`build_parse_table`)

1. 创建起始项目 `START → • S, {$}`
2. BFS 循环：计算闭包（FIRST 集合）→ 计算后继 → 填充 Shift/Reduce 动作
3. 冲突消解：优先级 + 结合性 + 用户声明的 expected_conflicts
4. 错误状态填充：state 0 的 Recover 动作

#### 词法 NFA → DFA (`build_lex_table`)

按需子集构造法：根据解析状态有效 token 集合收集 NFA 起始状态 → epsilon 闭包 → 字符转移 → 去重。不同解析状态可合并共享 lex 入口。

#### C 代码渲染 (`render.rs`)

按固定顺序生成：常量宏 → 符号枚举 → 符号名/元数据表 → 字段映射 → 词法函数（`ts_lex`/`ts_lex_keywords`）→ 解析表（大表+小表）→ 导出函数。大字符集（≥ 8 范围）提取为常量数组用 `set_contains` 查表。

### 10.4 设计模式

- **管道模式**：每步接收前步输出，产生新数据结构
- **构建者模式**：NfaBuilder、LexTableBuilder、Generator
- **游标模式**：NfaCursor 模拟 NFA 状态

---

## 11. Highlight 模块

**文件**：`crates/highlight/src/highlight.rs`

### 11.1 边界

| 维度 | 说明 |
|------|------|
| **输入** | `source: &[u8]`、`HighlightConfiguration`、`injection_callback` |
| **输出** | `Iterator<Item = HighlightEvent>`（`Source`/`HighlightStart`/`HighlightEnd`） |
| **对外接口** | `Highlighter::new/highlight`、`HighlightConfiguration::new/configure`、`HtmlRenderer` |

### 11.2 核心类型

- **HighlightConfiguration**：三个查询（injections + locals + highlights）拼接为一个 Query，通过 pattern index 分界
- **Highlighter**：持有 Parser + QueryCursor 对象池
- **HighlightIter**：多层迭代器（主语言 + 注入语言），每层有 cursor、captures 迭代器、highlight_end_stack、scope_stack

### 11.3 关键机制

- **注入**：普通注入（per-match，通过 `injection_for_match` 提取语言名和内容节点）+ 合并注入（`#set! injection.combined`）
- **locals 处理**：`@local.scope/definition/reference` 实现作用域感知高亮
- **模糊匹配**：捕获名 `function.method.builtin` 可匹配 recognized name `function.builtin`
- **层排序**：按下一事件字节偏移排序，结束事件优先于开始事件

### 11.4 设计模式

- **迭代器模式**：流式返回 HighlightEvent
- **分层拼接查询**：一次树遍历处理三种模式
- **对象池**：Highlighter 复用 Parser 和 QueryCursor

---

## 12. Tags 模块

**文件**：`crates/tags/src/tags.rs`

### 12.1 边界

| 维度 | 说明 |
|------|------|
| **输入** | `source: &[u8]`、`TagsConfiguration` |
| **输出** | `Iterator<Item = Tag>` |
| **对外接口** | `TagsContext::new/generate_tags`、`TagsConfiguration::new` |

### 12.2 核心类型

```rust
pub struct Tag {
  pub range: Range<usize>,       // 符号字节范围
  pub name_range: Range<usize>,  // 名字字节范围
  pub line_range: Range<usize>,  // 行范围
  pub span: Range<Point>,        // 行列范围
  pub docs: Option<String>,      // 文档注释
  pub is_definition: bool,       // 定义/引用
  pub syntax_type_id: u32,       // 类型 ID（function, class 等）
}
```

- **TagsConfiguration**：同样拼接 locals + tags 查询
- 捕获名约定：`@definition.function` → 函数定义，`@reference.call` → 调用引用
- 使用 `matches()`（非 `captures()`），因为需要完整 match 来获取所有相关捕获
- **tag_queue 去重**：同一节点只保留 pattern_index 最小的 tag

---

## 13. Loader 模块

**文件**：`crates/loader/src/loader.rs`

### 13.1 边界

| 维度 | 说明 |
|------|------|
| **输入** | parser 目录路径、`tree-sitter.json` / `package.json`、查询文件 |
| **输出** | `Language` 对象、`LanguageConfiguration`（含懒加载的 HighlightConfig/TagsConfig） |
| **对外接口** | `Loader::new`、`find_language_configurations_at_path`、`language_configuration_for_scope/file_name/injection_string/first_line_regex` |

### 13.2 核心设计

- **懒加载**：`OnceCell<Language>` 和 `OnceCell<Option<HighlightConfiguration>>` 实现首次请求时编译/加载
- **语言查找策略**：按 scope 名、文件名/后缀、注入字符串（正则）、首行内容等多种方式查找
- **编译流程**：检查动态库新旧 → C 编译 `parser.c` + `scanner.c` → `libloading` 加载 → 调用 `tree_sitter_{name}` 符号获取 TSLanguage
- **WASM 路径**：使用 wasi-sdk 编译为 `.wasm`，通过 WasmStore 加载

---

## 14. CLI 模块

**文件**：`crates/cli/src/`

### 14.1 命令列表

| 命令 | 输入 | 输出 |
|------|------|------|
| `init` | 交互式 | grammar.js、tree-sitter.json、绑定文件 |
| `generate` | grammar.js/json | parser.c、node-types.json |
| `build` | 语法目录 | .so/.dll/.wasm 动态库 |
| `parse` | 源文件 + parser | S-expression / DOT / XML |
| `test` | corpus 测试文件 | 测试结果 |
| `query` | 查询文件 + 源文件 | 匹配结果 |
| `highlight` | 源文件 + query | 终端/HTML 高亮 |
| `tags` | 源文件 + query | 标签列表 |
| `fuzz` | 语料库 + parser | 模糊测试结果 |
| `playground` | grammar 目录 | Web UI |

### 14.2 集成方式

CLI 是工具链的**胶合层**：

| 依赖 | 用途 |
|------|------|
| `tree_sitter` | 解析文件、运行查询 |
| `tree_sitter_generate` | grammar → parser.c |
| `tree_sitter_loader` | 编译/加载 parser 动态库 |
| `tree_sitter_highlight` | 语法高亮 |
| `tree_sitter_tags` | 标签提取 |
| `tree_sitter_config` | 用户配置 |

`tree_sitter_loader` 是关键中间层，所有需要 parser 的命令都通过它获取 Language。

---

## 模块间完整交互图

```
┌─────────────────────────────────────────────────────────────┐
│                         CLI                                  │
│  generate│parse│test│query│highlight│tags│fuzz│playground    │
└──────┬─────┬──────┬───────┬──────────┬────────┬─────────────┘
       │     │      │       │          │        │
       ▼     │      │       │          │        │
  ┌────────┐ │      │       │          │        │
  │Generate│ │      │       │          │        │
  └────────┘ │      │  ┌────▼─────┐    │        │
             │      │  │  Loader  │    │        │
             │      │  │ 编译/加载 │    │        │
             │      │  └──┬───┬───┘    │        │
             │      │     │   │        │        │
             ▼      ▼     ▼   ▼        ▼        ▼
      ┌────────────────────────┐  ┌─────────┐ ┌──────┐
      │   tree-sitter (lib)    │  │Highlight│ │ Tags │
      │  ┌───────┐  ┌───────┐ │  └────┬────┘ └──┬───┘
      │  │Parser │  │ Query │ │       │          │
      │  │ Lexer │  └───────┘ │       │          │
      │  │ Stack │             │  ┌────▼──────────▼────┐
      │  └───┬───┘             │  │  Query + TreeCursor │
      │      │                 │  │  (树遍历 + 匹配)    │
      │  ┌───▼──────────────┐  │  └────────────────────┘
      │  │ Subtree / Tree   │  │
      │  │ Node / TreeCursor│  │
      │  │ Language          │  │
      │  └──────────────────┘  │
      │  ┌──────────────────┐  │
      │  │ Alloc/Array/     │  │
      │  │ Length/Unicode    │  │
      │  └──────────────────┘  │
      └────────────────────────┘
```

---

*文档生成时间：2026-03-27*
*基于 Tree-sitter v0.27.0 源码分析*
*对不确定的地方已标注【待确认】*
