# Tree-sitter 数据流追踪分析

> 基于 Tree-sitter v0.27.0 源码分析
> 依赖文档：[architecture.md](./architecture.md)、[modules.md](./modules.md)、[businessflow.md](./businessflow.md)

---

## 目录

1. [主要数据结构的生命周期](#1-主要数据结构的生命周期)
2. [数据在模块间的流转](#2-数据在模块间的流转)
3. [状态数据](#3-状态数据)
4. [跨语言边界的数据流](#4-跨语言边界的数据流)

---

## 1. 主要数据结构的生命周期

### 1.1 输入文本到语法树的数据变换链

```
源码字节流(bytes)
    │
    ▼  TSInput.read 回调
Lexer chunk 缓存(const char*)
    │
    ▼  UTF 解码 + lex_fn 状态机匹配
Subtree (叶子节点: inline 8字节 或 HeapData)
    │
    ▼  Parser Shift 压入 Stack
StackNode.links[].subtree (边上的语法节点)
    │
    ▼  Parser Reduce 弹出子节点
Subtree (内部节点: 子数组 + HeapData 连续内存)
    │
    ▼  Parser Accept + Balance
TSTree { root: Subtree, language, included_ranges }
    │
    ▼  ts_tree_root_node
TSNode { context[4], id→Subtree*, tree→TSTree* }  (只读视图)
```

每一步的数据形态说明：

| 阶段 | 数据形态 | 大小 | 生命周期 |
|------|----------|------|----------|
| 源码字节流 | 用户提供的缓冲区，通过 `TSInput.read` 回调按需读取 | 按 chunk 读取 | 用户管理 |
| Lexer chunk | `const char*` 指向用户缓冲区，不拥有 | 不定长 | 随 `read` 回调返回值有效 |
| 前瞻字符 | `TSLexer.lookahead` (int32 Unicode 码点) | 4 字节 | 瞬态，每次 advance 更新 |
| Token (叶子 Subtree) | inline 8 字节值类型 或 堆上 `SubtreeHeapData` | 8B / ~80B | 引用计数管理 |
| StackNode 边上的 Subtree | `StackLink.subtree`，指向同一个 Subtree | 8 字节句柄 | 随 StackNode 引用计数 |
| 内部节点 Subtree | 子数组 + HeapData 连续分配 | `n×8 + sizeof(HeapData)` | 引用计数管理 |
| TSTree | 堆分配的结构体，持有 root Subtree | ~32 字节 | 用户调用 `ts_tree_delete` |
| TSNode | 栈分配的值类型视图 | 24/32 字节 | 不拥有数据，需 TSTree 存活 |

### 1.2 Subtree — 核心节点数据结构

#### 1.2.1 Tagged Union 设计

```c
// lib/src/subtree.h
typedef union {
  SubtreeInlineData data;       // inline 路径（8 字节，值类型）
  const SubtreeHeapData *ptr;   // heap 路径（指针）
} Subtree;
```

**判别机制**：利用指针对齐特性。合法堆指针的最低位始终为 0，因此 `data.is_inline = 1` 不会与指针冲突。

#### 1.2.2 SubtreeInlineData 位域布局（小端序 x86/x64，最常见）

```
字节  位          字段              宽度    含义
───────────────────────────────────────────────────────
0     bit 0       is_inline         1 bit   =1 inline / =0 指针
      bit 1       visible           1 bit   CST 中是否可见
      bit 2       named             1 bit   是否命名节点
      bit 3       extra             1 bit   是否 extra（注释等）
      bit 4       has_changes       1 bit   是否被编辑影响
      bit 5       is_missing        1 bit   是否 missing 节点
      bit 6       is_keyword        1 bit   是否关键字
1     bits 0-7    symbol            8 bit   符号 ID（≤ 255）
2-3   bits 0-15   parse_state       16 bit  解析状态 ID
4     bits 0-7    padding_columns   8 bit   前导空白列数
5     bits 0-3    padding_rows      4 bit   前导空白行数（≤ 15）
      bits 4-7    lookahead_bytes   4 bit   前瞻字节数（≤ 15）
6     bits 0-7    padding_bytes     8 bit   前导空白字节数
7     bits 0-7    size_bytes        8 bit   内容字节数
───────────────────────────────────────────────────────
总计：8 字节 = sizeof(void*)
```

**平台适配**：`subtree.h` 针对 大端+32位 / 大端+64位 / 小端 三种组合分别调整 `is_inline` 位在联合体中的位置，保证 `is_inline` 与指针最低有效位对齐。

**Inline 条件** (`subtree.c:155-164, 175-179`)：

```
ts_subtree_can_inline(padding, size, lookahead_bytes):
  padding.bytes < 255         &&
  padding.extent.row < 16     &&   // 4 bit
  padding.extent.column < 255 &&
  size.bytes < 255            &&
  size.extent.row == 0        &&   // 不跨行
  size.extent.column < 255    &&
  lookahead_bytes < 16             // 4 bit

额外条件:
  symbol <= 255               &&   // 8 bit
  !has_external_tokens              // 无外部 token 状态
```

#### 1.2.3 SubtreeHeapData 完整字段

```c
// lib/src/subtree.h:111-154
typedef struct {
  volatile uint32_t ref_count;     // 引用计数（原子操作）
  Length padding;                  // 前导空白 {bytes, extent{row, col}}
  Length size;                     // 内容大小 {bytes, extent{row, col}}
  uint32_t lookahead_bytes;        // 前瞻字节数
  uint32_t error_cost;            // 错误代价（聚合值）
  uint32_t child_count;           // 子节点数（0 = 叶子）
  TSSymbol symbol;                // 符号 ID
  TSStateId parse_state;          // 产出此节点时的解析状态

  // 布尔标志位域
  bool visible : 1;               // CST 可见
  bool named : 1;                 // 命名节点
  bool extra : 1;                 // extra（注释等）
  bool fragile_left : 1;          // 左侧脆弱（增量解析不可复用边界）
  bool fragile_right : 1;         // 右侧脆弱
  bool has_changes : 1;           // 被编辑影响
  bool has_external_tokens : 1;   // 含外部扫描器 token
  bool has_external_scanner_state_change : 1;
  bool depends_on_column : 1;     // 依赖列号（缩进敏感语言）
  bool is_missing : 1;            // 错误恢复插入的缺失节点
  bool is_keyword : 1;            // 关键字

  union {
    // 变体1: 非终结符（child_count > 0）
    struct {
      uint32_t visible_child_count;
      uint32_t named_child_count;
      uint32_t visible_descendant_count;
      int32_t dynamic_precedence;
      uint16_t repeat_depth;
      uint16_t production_id;
      struct { TSSymbol symbol; TSStateId parse_state; } first_leaf;
    };
    // 变体2: 外部 token 叶子（child_count == 0 && has_external_tokens）
    ExternalScannerState external_scanner_state;
    // 变体3: 错误叶子（child_count == 0 && symbol == ts_builtin_sym_error）
    int32_t lookahead_char;
  };
} SubtreeHeapData;
```

#### 1.2.4 父节点内存布局 — 连续分配

子节点数组和父节点 HeapData 在**同一块连续内存**中分配 (`subtree.c:477-516`)：

```
┌──────────┬──────────┬─────┬──────────┬─────────────────┐
│ child[0] │ child[1] │ ... │ child[n] │ SubtreeHeapData │
│ (8 bytes)│ (8 bytes)│     │ (8 bytes)│   (父节点数据)   │
└──────────┴──────────┴─────┴──────────┴─────────────────┘
                                        ↑ ptr 指向这里
```

- **分配大小**：`child_count × sizeof(Subtree) + sizeof(SubtreeHeapData)` (`subtree.h:246-248`)
- **子节点访问**：`(Subtree *)(ptr) - child_count`，向前回退指针 (`subtree.h:251-252`)
- **释放**：一次 `ts_free(children)` 释放整块 (`subtree.c:574-586`)

**设计优势**：
- 只需一次 malloc/free 完成分配/释放
- 子节点数组与父节点数据在同一缓存行附近，提升缓存局部性
- 遍历子节点时内存访问连续

#### 1.2.5 ExternalScannerState — SSO 优化

```c
// lib/src/subtree.h
typedef struct {
  union { char *long_data; char short_data[24]; };
  uint32_t length;
} ExternalScannerState;
```

长度 ≤ 24 字节时 inline 存储（Small String Optimization），超过则堆分配。

### 1.3 SubtreePool — 对象池

```c
// lib/src/subtree.h
typedef struct {
  MutableSubtreeArray free_trees;   // 空闲 HeapData 缓存
  MutableSubtreeArray tree_stack;   // release/compare 的工作栈（复用）
} SubtreePool;
```

#### 工作机制

| 维度 | 说明 |
|------|------|
| **池化对象** | 仅池化**叶子节点**的 `SubtreeHeapData`（大小固定）；父节点的变长连续内存不池化 |
| **容量上限** | `TS_MAX_TREE_POOL_SIZE = 32` (`subtree.c:23`) |
| **获取** | `ts_subtree_pool_allocate`：池非空则 pop，否则 `ts_malloc(sizeof(SubtreeHeapData))` (`subtree.c:137-142`) |
| **归还** | `ts_subtree_pool_free`：`capacity > 0` 且 `size+1 ≤ 32` 时 push，否则直接 `ts_free` (`subtree.c:145-150`) |
| **容量为 0 的池** | `ts_tree_delete`/`ts_tree_edit` 中创建 `ts_subtree_pool_new(0)`，此时 `capacity=0` → 释放时永不进池，直接 free |
| **池销毁** | 遍历 `free_trees` 中每一项调用 `ts_free` (`subtree.c:127-134`) |

```
Pool 获取/归还流程:

  ts_subtree_new_leaf:
    需要 HeapData → ts_subtree_pool_allocate
                       │
                       ├── free_trees.size > 0 → array_pop → 返回
                       └── 否则 → ts_malloc(sizeof(SubtreeHeapData))

  ts_subtree_release → ref_count 降为 0:
    叶子节点 → ts_subtree_pool_free
                       │
                       ├── capacity > 0 且 size < 32 → array_push (回收)
                       └── 否则 → ts_free
    父节点   → ts_free(children 整块)
```

### 1.4 引用计数与所有权模型

#### 1.4.1 retain — 原子递增

```c
// subtree.c:558-563
void ts_subtree_retain(Subtree self) {
  if (self.data.is_inline) return;      // inline 无引用计数
  atomic_inc(&self.ptr->ref_count);     // __atomic_add_fetch SEQ_CST
}
```

`atomic_inc` 使用 `__atomic_add_fetch`（GCC/Clang）或 `InterlockedIncrement`（MSVC） (`atomic.h:50-55`)。

#### 1.4.2 release — 显式栈非递归级联释放

```c
// subtree.c:565-593（简化）
void ts_subtree_release(SubtreePool *pool, Subtree self) {
  if (self.data.is_inline) return;

  if (atomic_dec(&self.ptr->ref_count) == 0)
    array_push(&pool->tree_stack, self);

  while (pool->tree_stack.size > 0) {
    tree = array_pop(&pool->tree_stack);
    if (tree.ptr->child_count > 0) {
      // 父节点: 对每个子节点 atomic_dec，为 0 则入栈
      for each child: if atomic_dec == 0 → push child
      ts_free(children);        // 释放整块连续内存
    } else {
      // 叶子: 清理外部状态，归还池
      if has_external_tokens: delete external_scanner_state
      ts_subtree_pool_free(pool, tree.ptr);
    }
  }
}
```

**设计要点**：
- 使用显式栈而非递归，防止深树导致栈溢出
- 原子操作保证多线程安全
- inline 节点完全跳过（无堆分配、无引用计数）

#### 1.4.3 COW — 写时复制 (make_mut)

```c
// subtree.c:279-289
MutableSubtree ts_subtree_make_mut(SubtreePool *pool, Subtree self) {
  if (self.data.is_inline) return (MutableSubtree){self.data};
  if (self.ptr->ref_count == 1) return ts_subtree_to_mut_unsafe(self);  // 唯一所有者
  // ref_count > 1: 深拷贝
  MutableSubtree result = ts_subtree_clone(self);  // 新分配 + memcpy + 子节点 retain
  ts_subtree_release(pool, self);                  // 减少原引用
  return result;
}
```

`ts_subtree_clone` (`subtree.c:259-277`)：分配同样大小的连续内存，`memcpy` 整块，对每个子节点调用 `ts_subtree_retain`，设置 `ref_count = 1`。

#### 1.4.4 所有权/生命周期总表

```
┌─────────────────────────────────────────────────────────────────┐
│                     所有权层次结构                                │
│                                                                  │
│  用户代码                                                        │
│    │                                                             │
│    ├── TSParser*  ──────────────────────┐                        │
│    │   (ts_parser_new / delete)        │                        │
│    │   ├── Lexer (嵌入)                 │                        │
│    │   ├── Stack* (拥有)                │                        │
│    │   │   └── StackNode* (引用计数)    │                        │
│    │   │       └── StackLink.subtree   │  ← retain             │
│    │   ├── SubtreePool (嵌入)           │                        │
│    │   ├── ReusableNode (嵌入)          │  → 指向旧树 (不拥有)   │
│    │   └── finished_tree: Subtree      │  ← retain             │
│    │                                    │                        │
│    └── TSTree*  ────────────────────────┘                        │
│        (ts_tree_new / copy / delete)                             │
│        ├── root: Subtree (拥有, retain/release)                  │
│        │   └── 子节点 (引用计数共享)                              │
│        ├── language: TSLanguage* (引用计数拷贝)                   │
│        └── included_ranges: TSRange* (独立拷贝)                  │
│                                                                  │
│    TSNode (值类型, 不拥有)                                        │
│        ├── id → Subtree* (借用, 需 TSTree 存活)                  │
│        ├── tree → TSTree* (借用)                                 │
│        └── context[4] (缓存的位置和 alias)                       │
│                                                                  │
│    TSTreeCursor (值类型, 但内含动态栈)                            │
│        ├── tree → TSTree* (借用)                                 │
│        └── stack: Array(TreeCursorEntry) (拥有)                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| 对象 | 所有权模式 | 创建 | 销毁 | 共享方式 |
|------|-----------|------|------|----------|
| **Subtree (inline)** | 值类型，无堆分配 | 栈上构造 | 无需释放 | 可任意复制 |
| **Subtree (heap)** | 引用计数 | `ts_subtree_new_leaf/new_node` | `ts_subtree_release` 级联 | `ts_subtree_retain` |
| **TSParser** | 唯一所有权 | `ts_parser_new` | `ts_parser_delete` | 不可共享 |
| **TSTree** | 唯一所有权 + 可共享根 | `ts_tree_new` | `ts_tree_delete` | `ts_tree_copy`（浅拷贝, root retain） |
| **TSNode** | 借用视图 | `ts_tree_root_node` 等 | 无需释放 | 需 TSTree 存活 |
| **Stack** | TSParser 独占 | `ts_stack_new` | `ts_stack_delete` | 不可共享 |
| **StackNode** | 引用计数 | `stack_node_new` | `stack_node_release` | 多 StackHead 共享 |
| **TSLanguage** | 引用计数 | 生成的静态/动态加载 | `ts_language_delete` | `ts_language_copy` |
| **SubtreePool** | 嵌入在 TSParser | 随 TSParser 创建 | 随 TSParser 销毁 | 不可共享 |

### 1.5 TSTree 生命周期

```
创建:
  ts_tree_new(root, language, included_ranges, count)    // tree.c:9-19
  ├── ts_malloc(sizeof(TSTree))
  ├── result->root = root                                // Subtree 已 retain
  ├── result->language = ts_language_copy(language)       // 引用计数 +1
  └── result->included_ranges = ts_calloc + memcpy       // 独立拷贝

拷贝 (浅共享):
  ts_tree_copy(self)                                      // tree.c:22-25
  ├── ts_subtree_retain(self->root)                       // 共享同一子树图
  └── ts_tree_new(self->root, ...)                        // 新建外壳

编辑:
  ts_tree_edit(self, edit)                                // tree.c:55-62
  ├── 更新 included_ranges 位置
  └── ts_subtree_edit(self->root, edit, &pool)
      ├── COW: 被影响的节点 make_mut (ref_count>1 时 clone)
      ├── 调整 padding/size
      └── 设置 has_changes 标记

销毁:
  ts_tree_delete(self)                                    // tree.c:27-35
  ├── ts_subtree_release(&pool, self->root)               // 级联释放子树
  ├── ts_language_delete(self->language)                  // 引用计数 -1
  ├── ts_free(self->included_ranges)
  └── ts_free(self)
```

### 1.6 TSNode 值类型设计

```c
// api.h:133-137
typedef struct TSNode {
  uint32_t context[4];   // [start_byte, start_row, start_col, alias_symbol]
  const void *id;        // 指向 Subtree* (树内某处子数组中的元素)
  const TSTree *tree;    // 关联的 TSTree
} TSNode;
```

| 字段 | 大小 | 含义 |
|------|------|------|
| `context[0]` | 4B | 起始字节偏移 |
| `context[1]` | 4B | 起始行号 |
| `context[2]` | 4B | 起始列号 |
| `context[3]` | 4B | alias 符号 ID |
| `id` | 4/8B | 指向 `const Subtree*`（树内位置） |
| `tree` | 4/8B | 关联的 TSTree 指针 |

**总大小**：32 位系统 24 字节，64 位系统 32 字节。

**关键设计**：TSNode 是纯**只读视图**——不拥有数据，不参与引用计数。`context` 中缓存位置信息避免每次遍历时重新计算。`id` 指向树中某个 Subtree 的地址，通过 `*(const Subtree *)self.id` 解引用获取子树。

### 1.7 解析表的数据结构与运行时查询

#### 1.7.1 TSLanguage 中的解析表字段

```c
// parser.h:107-152（关键字段摘录）
struct TSLanguage {
  // 计数
  uint32_t symbol_count;
  uint32_t token_count;          // 终结符数量
  uint32_t state_count;
  uint32_t large_state_count;    // 大表状态数量分界

  // 解析表
  const uint16_t *parse_table;        // 大表: state × symbol → action_index
  const uint16_t *small_parse_table;  // 小表: 分组压缩格式
  const uint32_t *small_parse_table_map; // 小表状态 → 小表偏移
  const TSParseActionEntry *parse_actions; // 动作数组

  // 词法
  const TSLexMode *lex_modes;     // parse_state → lex_state 映射
  bool (*lex_fn)(TSLexer *, TSStateId);           // 主词法函数
  bool (*keyword_lex_fn)(TSLexer *, TSStateId);   // 关键词词法函数
  TSSymbol keyword_capture_token;

  // 符号/别名/字段
  const char *const *symbol_names;
  const TSSymbolMetadata *symbol_metadata;
  const TSSymbol *public_symbol_map;
  const uint16_t *alias_sequences;
  const TSFieldMapSlice *field_map_slices;
  const TSFieldMapEntry *field_map_entries;

  // 外部扫描器
  struct {
    const bool *states;
    void *(*create)(void);
    void (*destroy)(void *);
    bool (*scan)(void *, TSLexer *, const bool *);
    unsigned (*serialize)(void *, char *);
    void (*deserialize)(void *, const char *, unsigned);
  } external_scanner;
};
```

#### 1.7.2 双格式解析表查询

```
ts_language_lookup(language, state, symbol)
│
├── state < large_state_count:                    // 大表（密集状态）
│   index = state × symbol_count + symbol
│   return parse_table[index]                     // O(1) 直接索引
│
└── state >= large_state_count:                   // 小表（稀疏状态）
    offset = small_parse_table_map[state - large_state_count]
    遍历 small_parse_table[offset..]:
      每组: (section_value, symbol_count, symbol[0..n])
      线性搜索匹配 symbol                          // O(k) k=每组符号数
    未找到 → 0
```

```
ts_language_table_entry(language, state, symbol, &table_entry)
│
├── symbol == ts_builtin_sym_error → 空动作
│
├── action_index = ts_language_lookup(state, symbol)
│
└── entry = &parse_actions[action_index]
    table_entry.action_count = entry->entry.count
    table_entry.is_reusable = entry->entry.reusable
    table_entry.actions = (TSParseAction *)(entry + 1)  // 紧随其后的动作数组
```

#### 1.7.3 TSParseAction 联合体

```c
// parser.h:66-81
typedef union {
  struct {                           // Shift 动作
    uint8_t type;                    // TSParseActionTypeShift
    TSStateId state;                 // 目标状态
    bool extra;                      // 是否标记为 extra
    bool repetition;                 // 是否重复规则
  } shift;
  struct {                           // Reduce 动作
    uint8_t type;                    // TSParseActionTypeReduce
    uint8_t child_count;             // 弹出的子节点数
    TSSymbol symbol;                 // 归约为的符号
    int16_t dynamic_precedence;      // 动态优先级
    uint16_t production_id;          // 产生式 ID
  } reduce;
  uint8_t type;                      // Accept / Recover
} TSParseAction;
```

#### 1.7.4 TSParseActionEntry（查表返回的头部）

```c
// parser.h:94-100
typedef union {
  TSParseAction action;
  struct {
    uint8_t count;       // 后续动作数量
    bool reusable;       // 增量解析时此状态的 token 是否可复用
  } entry;
} TSParseActionEntry;
```

内存布局：`[TSParseActionEntry(header)] [TSParseAction × count]`，连续存储。

### 1.8 Subtree 缓存/共享机制

#### 1.8.1 SubtreePool — 叶子节点对象池

见 1.3 节。只池化固定大小的叶子 HeapData，最多 32 个。

#### 1.8.2 TokenCache — 同次解析的词法缓存

```c
// parser.c:83-87
typedef struct {
  Subtree token;                // 缓存的 token (Subtree)
  Subtree last_external_token;  // 缓存时的外部扫描器状态
  uint32_t byte_index;          // 缓存位置
} TokenCache;
```

**用途**：同一次解析中，多个 GLR 栈版本在相同位置需要 lex 时，避免重复词法分析。

**缓存命中条件** (`parser.c:710-721`)：
1. `cache->token` 有效
2. `cache->byte_index == position`（位置相同）
3. `external_scanner_state` 等价
4. `ts_parser__can_reuse_first_leaf` 确认在当前 state 下 token 合法

**写入时机** (`parser.c:724-737`)：每次成功 lex 后写入。解析器 new/reset/delete 时清空。

#### 1.8.3 增量解析节点复用 — ReusableNode

```c
// reusable_node.h
typedef struct {
  Array(StackEntry) stack;           // DFS 路径栈
  Subtree last_external_token;       // 最后一个含 external token 的子树
} ReusableNode;
```

**复用条件** (`parser.c:753-830`)：
- 位置对齐 (`byte_offset == position`)
- 无 `has_changes` 标记
- 非 error / missing / fragile 节点
- 不与 `included_range_differences` 相交
- `external_scanner_state` 一致
- 当前解析状态的 lex_mode 与节点产出时一致
- 解析表标记 `is_reusable`

#### 1.8.4 树共享 — ts_tree_copy

`ts_tree_copy` 创建新的 TSTree 外壳，但共享同一个 `root` Subtree 图（引用计数 +1）。多个 TSTree 可以指向同一棵子树，COW 保证修改安全。

### 1.9 内存分配策略

#### 1.9.1 全局函数指针抽象

```c
// alloc.h:18-21
TS_PUBLIC extern void *(*ts_current_malloc)(size_t size);
TS_PUBLIC extern void *(*ts_current_calloc)(size_t count, size_t size);
TS_PUBLIC extern void *(*ts_current_realloc)(void *ptr, size_t size);
TS_PUBLIC extern void (*ts_current_free)(void *ptr);
```

宏封装 (`alloc.h:24-34`)：

```c
#define ts_malloc   ts_current_malloc
#define ts_calloc   ts_current_calloc
#define ts_realloc  ts_current_realloc
#define ts_free     ts_current_free
```

#### 1.9.2 默认实现与 OOM 策略

```c
// alloc.c:5-30
static void *ts_malloc_default(size_t size) {
  void *result = malloc(size);
  if (size > 0 && !result) {
    fprintf(stderr, "tree-sitter failed to allocate %zu bytes", size);
    abort();
  }
  return result;
}
// calloc/realloc 类似: 失败时 abort
```

#### 1.9.3 可替换分配器

```c
// alloc.c:38-48
void ts_set_allocator(
  void *(*new_malloc)(size_t),
  void *(*new_calloc)(size_t, size_t),
  void *(*new_realloc)(void *, size_t),
  void (*new_free)(void *)
) {
  ts_current_malloc  = new_malloc  ? new_malloc  : ts_malloc_default;
  ts_current_calloc  = new_calloc  ? new_calloc  : ts_calloc_default;
  ts_current_realloc = new_realloc ? new_realloc : ts_realloc_default;
  ts_current_free    = new_free    ? new_free    : free;
}
```

传入 NULL 时回退到默认实现。**全局替换**，影响所有 Tree-sitter 对象的分配。

#### 1.9.4 分配策略汇总

| 对象 | 分配方式 | 释放方式 |
|------|----------|----------|
| SubtreeHeapData (叶子) | SubtreePool 获取 或 `ts_malloc` | `ts_subtree_pool_free` 或 `ts_free` |
| SubtreeHeapData (父节点+子数组) | `ts_malloc/ts_realloc` 连续分配 | `ts_free` 一次释放整块 |
| TSTree | `ts_malloc(sizeof(TSTree))` | `ts_free` |
| TSTree.included_ranges | `ts_calloc + memcpy` | `ts_free` |
| Stack | `ts_malloc(sizeof(Stack))` | `ts_free` |
| StackNode | StackNode 池 (最多 50 个) 或 `ts_malloc` | 归还池 或 `ts_free` |
| Array 动态数组 | `ts_realloc` 倍增 | `array_delete → ts_free` |

---

## 2. 数据在模块间的流转

### 2.1 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         用户代码                                        │
│  TSInput.read(payload, byte_index, position, &bytes_read)               │
│  → const char* (文本块)                                                 │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ chunk 指针 (不拷贝)
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Lexer 模块 (lexer.c)                                                   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │ Lexer 结构体                                                │        │
│  │ ├── chunk: const char*         ← TSInput.read 返回          │        │
│  │ ├── chunk_start/chunk_size     ← 跟踪 chunk 在文档中位置    │        │
│  │ ├── current_position: Length   ← 逐字符前进                 │        │
│  │ └── lookahead: int32           ← UTF 解码后的 Unicode 码点  │        │
│  └─────────────────────────────────────────────────────────────┘        │
│                                                                          │
│  TSLexer 回调接口 (advance/mark_end/get_column/eof)                     │
│           ↓                                                              │
│  lex_fn(lexer, lex_state) → result_symbol                               │
│  或 external_scanner.scan(payload, lexer, valid_tokens)                  │
│                                                                          │
│  输出: result_symbol + token_start_position + token_end_position         │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ (symbol, padding, size)
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Parser 模块 (parser.c)                                                  │
│                                                                          │
│  ┌──────────── Token 创建 ────────────┐                                 │
│  │ ts_subtree_new_leaf(symbol, ...)    │                                 │
│  │ → Subtree (inline 或 heap)         │                                 │
│  └────────────────┬───────────────────┘                                 │
│                   │ lookahead: Subtree                                   │
│                   ▼                                                      │
│  ┌──────────── 查解析表 ──────────────┐                                 │
│  │ ts_language_table_entry(           │                                 │
│  │   state, symbol) → TableEntry      │                                 │
│  │   {actions[], count, reusable}     │                                 │
│  └────────────────┬───────────────────┘                                 │
│                   │                                                      │
│      ┌────────────┼────────────┬──────────────┐                         │
│      ▼            ▼            ▼              ▼                         │
│   Shift        Reduce       Accept        Recover                       │
│      │            │            │              │                         │
│      ▼            ▼            ▼              ▼                         │
│  ┌────────────────────────────────────────────────────────────┐         │
│  │                    Stack 模块                               │         │
│  │                                                             │         │
│  │  Shift:                                                    │         │
│  │    创建 StackNode {state, position}                         │         │
│  │    StackLink {node→前驱, subtree=lookahead, is_pending}     │         │
│  │    → push 到 StackHead                                     │         │
│  │                                                             │         │
│  │  Reduce:                                                   │         │
│  │    pop_count(child_count) → StackSlice[] (多路径)           │         │
│  │    每条路径: SubtreeArray (弹出的子节点)                    │         │
│  │    → ts_subtree_new_node(symbol, &children)                │         │
│  │    → 创建父 Subtree (连续内存)                              │         │
│  │    → GOTO 查表 → push 新状态                               │         │
│  │                                                             │         │
│  │  Accept:                                                   │         │
│  │    pop_all → 根节点 Subtree                                 │         │
│  │    → self->finished_tree                                   │         │
│  │                                                             │         │
│  └────────────────────────────────────────────────────────────┘         │
│                                                                          │
│  后处理:                                                                 │
│  ts_parser__balance_subtree → 压缩高重复深度的链                         │
│  ts_tree_new(finished_tree, language, ranges) → TSTree*                  │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ TSTree*
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Tree/Node 模块 (tree.c, node.c)                                        │
│                                                                          │
│  ts_tree_root_node(tree) → TSNode {context, id→Subtree*, tree}          │
│  ts_node_child(node, i) → TSNode (遍历子数组, 跳过隐藏节点)             │
│  ts_node_string(node) → S-expression 字符串                             │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 每一步数据的形态与传递方式

#### 步骤 1：字节流 → Lexer chunk

```
用户缓冲区                     Lexer
  "let x = 1;\n..."    ─read→  chunk = "let x = 1;\n..."
                                chunk_start = 0
                                chunk_size = 12
```

- **传递方式**：零拷贝。`TSInput.read` 返回 `const char*`，Lexer 直接引用，不拥有
- **生命周期**：chunk 指针的有效性由 `TSInput.read` 回调保证（下一次 `read` 前有效）

#### 步骤 2：chunk → Unicode 码点

```
chunk: "let"                     Lexer
  字节 0x6C,0x65,0x74   ─解码→  lookahead = 'l' (0x6C)
                                 lookahead_size = 1
                         ─advance→ lookahead = 'e' (0x65)
                         ─advance→ lookahead = 't' (0x74)
```

- **传递方式**：`ts_lexer__get_lookahead` 根据 encoding 调用 `ts_decode_utf8/utf16_le/utf16_be`
- **跨 chunk 处理**：多字节字符可能跨 chunk 边界，此时触发重新 `get_chunk`

#### 步骤 3：Unicode 码点 → Token (Subtree)

```
lex_fn(lexer, lex_state):
  逐字符匹配 switch-case 状态机
  ├── 匹配到: mark_end, result_symbol = sym_identifier
  └── 返回 true

Parser:
  ts_subtree_new_leaf(sym_identifier, padding={0,0}, size={3,{0,3}}, ...)
  → Subtree (inline: 8 字节, 因为 sym ≤ 255 且 size ≤ 255)
```

- **传递方式**：lex_fn 通过设置 `TSLexer.result_symbol` 传递匹配结果；Parser 读取 `lexer.data.result_symbol` 和位置信息创建 Subtree

#### 步骤 4：Token → Stack 上的边

```
Shift:
  StackNode_new {state=5, position={3,{0,3}}}
  StackLink {
    node → 前驱 StackNode,
    subtree = Subtree(sym_identifier),    ← retain
    is_pending = false
  }
  StackHead.node = new_node
```

- **传递方式**：`ts_stack_push` 接收 Subtree 并 retain，存入 StackLink.subtree

#### 步骤 5：Stack 弹出 → 父节点

```
Reduce (sym_variable_declaration, child_count=5):
  ts_stack_pop_count(version, 5) → StackSlice {
    subtrees: ["let", identifier, "=", number, ";"]
  }
  ts_subtree_new_node(sym_variable_declaration, &subtrees)
  → SubtreeHeapData (连续内存: 5×8 + sizeof(HeapData))
  → summarize_children: 汇总 padding/size/error_cost/...
```

- **传递方式**：`ts_stack_pop_count` 沿 GSS 图回溯收集子树，返回 `StackSlice`（含 `SubtreeArray`）

#### 步骤 6：完成树 → TSTree

```
Accept:
  ts_stack_pop_all → root Subtree
  ts_parser__balance_subtree → 平衡重复节点
  ts_tree_new(root, language, ranges) → TSTree*
```

### 2.3 主要数据对象流转图

```
┌─────────┐  read   ┌────────┐  UTF解码  ┌──────────┐  lex_fn   ┌──────────┐
│ 用户    │ ──────→ │ Lexer  │ ───────→  │ 码点流   │ ───────→  │ Token    │
│ 缓冲区  │ chunk   │ 模块   │ lookahead │(瞬态)    │ symbol   │ Subtree  │
└─────────┘         └────────┘           └──────────┘           └────┬─────┘
                                                                     │
                                                          Shift      │
                                                   ┌─────────────────┘
                                                   │
                                                   ▼
                           ┌──────────────────────────────────┐
                           │  Stack (GSS)                      │
                           │  ┌──────┐  ┌──────┐  ┌──────┐   │
                           │  │Node_n│→ │Node_2│→ │Node_1│   │
                           │  │  ↑   │  │  ↑   │  │  ↑   │   │
                           │  │ sub  │  │ sub  │  │ sub  │   │
                           │  └──┬───┘  └──┬───┘  └──┬───┘   │
                           │     │         │         │        │
                           │  ┌──▼───┐  ┌──▼───┐  ┌──▼───┐   │
                           │  │Token │  │Token │  │Token │   │
                           │  │Subtree│ │Subtree│ │Subtree│   │
                           │  └──────┘  └──────┘  └──────┘   │
                           └──────────────┬───────────────────┘
                                          │ Reduce: pop → children
                                          ▼
                           ┌───────────────────────────────────┐
                           │  Parent Subtree (连续内存)          │
                           │  ┌───┬───┬───┬───┬───┬──────────┐ │
                           │  │c0 │c1 │c2 │c3 │c4 │HeapData  │ │
                           │  └───┴───┴───┴───┴───┴──────────┘ │
                           └──────────────┬────────────────────┘
                                          │ push 回 Stack
                                          │ ...循环...
                                          ▼
                           ┌───────────────────────────────────┐
                           │  TSTree                            │
                           │  ├── root: Subtree (整棵树)       │
                           │  ├── language: TSLanguage*         │
                           │  └── included_ranges              │
                           └──────────────┬────────────────────┘
                                          │ ts_tree_root_node
                                          ▼
                           ┌───────────────────────────────────┐
                           │  TSNode (只读视图)                 │
                           │  ├── context: [byte, row, col, alias] │
                           │  ├── id → Subtree*                │
                           │  └── tree → TSTree*               │
                           └───────────────────────────────────┘
```

---

## 3. 状态数据

### 3.1 Parser 内部状态

#### 3.1.1 TSParser 完整结构

```c
// parser.c:83-114
struct TSParser {
  Lexer lexer;                      // 嵌入: 词法分析器实例
  Stack *stack;                     // 指针: GLR 多版本解析栈
  SubtreePool tree_pool;            // 嵌入: 子树内存池
  const TSLanguage *language;       // 指针: 当前语法定义
  TSWasmStore *wasm_store;          // 指针: WASM 语言存储（可选）
  ReduceActionSet reduce_actions;   // 嵌入: 临时归约动作集合
  Subtree finished_tree;            // 嵌入: 已完成的语法树根
  SubtreeArray trailing_extras;     // 嵌入: Reduce 时暂存的尾部 extra 节点
  SubtreeArray trailing_extras2;    // 嵌入: 多路径选择的备用 extra 暂存
  SubtreeArray scratch_trees;       // 嵌入: select_children 时的临时数组
  TokenCache token_cache;           // 嵌入: 最近一次词法分析缓存
  ReusableNode reusable_node;       // 嵌入: 旧树遍历器（增量解析用）
  void *external_scanner_payload;   // 指针: 外部扫描器实例
  FILE *dot_graph_file;             // 指针: DOT 图调试输出
  unsigned accept_count;            // 已接受的解析结果数量
  unsigned operation_count;         // 操作计数器（进度回调节流）
  Subtree old_tree;                 // 嵌入: 旧语法树根（增量解析）
  TSRangeArray included_range_differences; // 嵌入: 新旧范围差异
  TSParseOptions parse_options;     // 嵌入: 解析选项（含 progress_callback）
  TSParseState parse_state;         // 嵌入: 当前解析状态（传给回调）
  unsigned included_range_difference_index; // 范围差异遍历索引
  bool has_scanner_error;           // 外部扫描器错误标志
  bool canceled_balancing;          // 树平衡是否被中断
  bool has_error;                   // 所有栈版本是否都处于错误状态
};
```

#### 3.1.2 operation_count 与 progress_callback 的关系

```
operation_count 节流机制 (parser.c:1536-1555):
  OP_COUNT_PER_PARSER_CALLBACK_CHECK = 100

  ts_parser__check_progress(self, position, lookahead, operations):
    self->operation_count += operations
    if operation_count >= 100:
      operation_count = 0               ← 归零（每 100 次清一次）
      更新 parse_state.current_byte_offset
      更新 parse_state.has_error
      if progress_callback(&parse_state):
        释放 lookahead
        return false                    ← 取消解析
    return true
```

### 3.2 Stack (GSS) 结构与工作过程

#### 3.2.1 完整数据结构

```
Stack 结构体 (stack.c:64-71)
├── heads: Array(StackHead)           版本头数组（索引 = 版本号）
│   └── StackHead {
│         node: StackNode*            栈顶节点
│         summary: StackSummary*      状态摘要（错误恢复用）
│         node_count_at_last_error    上次错误时的节点计数
│         last_external_token         上一个外部 token（Subtree）
│         lookahead_when_paused       暂停时保存的 lookahead
│         status: StackStatus         Active / Paused / Halted
│       }
│
├── slices: StackSliceArray           pop 结果缓冲
├── iterators: Array(StackIterator)   遍历迭代器缓冲
├── node_pool: StackNodeArray         StackNode 对象池（最多 50 个）
├── base_node: StackNode*             所有版本共享的栈底
└── subtree_pool: SubtreePool*        Subtree 内存池引用
```

```
StackNode 结构体 (stack.c:29-38)
├── state: TSStateId                  解析状态
├── position: Length                  文档位置 {bytes, {row, col}}
├── links: StackLink[8]              前驱链接（最多 8 条，支持 GLR 分叉）
│   └── StackLink {
│         node: StackNode*            前驱节点
│         subtree: Subtree            边上的语法节点
│         is_pending: bool            是否为 pending 状态
│       }
├── link_count: uint16               实际链接数
├── ref_count: uint32                引用计数
├── error_cost: uint32               累计错误代价
├── node_count: uint32               累计节点数
└── dynamic_precedence: int32        累计动态优先级
```

#### 3.2.2 GLR 栈的工作过程

```
初始状态:
  heads: [Head0]
  Head0.node → [base_node: state=1]

═══ 正常解析（单版本） ═══

Shift token_A (state 1 → 5):
  创建 StackNode_1 {state=5, pos=3}
  StackLink {node→base_node, subtree=token_A, pending=false}
  Head0.node = StackNode_1

  heads: [Head0]
  Head0.node → [Node_1:s5] ─token_A─→ [base:s1]

Shift token_B (state 5 → 8):
  创建 StackNode_2 {state=8, pos=6}
  Head0.node = StackNode_2

  Head0.node → [Node_2:s8] ─token_B─→ [Node_1:s5] ─token_A─→ [base:s1]

Reduce (sym_X, child_count=2):
  pop 2 → subtrees = [token_A, token_B]
  parent = ts_subtree_new_node(sym_X, [token_A, token_B])
  next_state = GOTO(base.state=1, sym_X) = 10
  创建 StackNode_3 {state=10, pos=6}
  StackLink {node→base_node, subtree=parent, pending=false}
  Head0.node = StackNode_3

  Head0.node → [Node_3:s10] ─parent(sym_X)─→ [base:s1]

═══ GLR 分叉（冲突产生多版本） ═══

Shift/Reduce 冲突:
  先 Reduce → 创建新 version:
    ts_stack__add_version → Head1
    pop + new_node + push

  再 Shift 原 version:
    Head0 → shift

  heads: [Head0, Head1]
  Head0.node → [Node_A] ─...─→ [shared]
  Head1.node → [Node_B] ─...─→ [shared]   ← 共享底层子图

═══ 版本合并 ═══

ts_stack_merge(v0, v1):
  条件: 同 state + 同 position + 同 error_cost + 同 external_token
  → Head0.node 添加 Head1 的 links
  → 删除 Head1

  Head0.node → [Node_merged]
               ├── link0 → [path_A]
               └── link1 → [path_B]    ← 歧义保留在 links 中

═══ 版本收缩 (ts_parser__condense_stack) ═══

  两两比较 ErrorStatus:
  ├── 明显更差 → remove_version
  ├── 可合并 → ts_stack_merge
  └── 需交换 → swap (保持优先顺序)

  硬上限: MAX_VERSION_COUNT = 6
  超出 → 删除多余版本
```

#### 3.2.3 StackNode 对象池

```
常量 (stack.c:11-13):
  MAX_LINK_COUNT = 8           每个 StackNode 最多 8 条前驱链接
  MAX_NODE_POOL_SIZE = 50      StackNode 池最大缓存数
  MAX_ITERATOR_COUNT = 64      pop 遍历时迭代器分裂上限

分配: stack_node_new (stack.c:83-102)
  池非空 → array_pop
  否则   → ts_malloc(sizeof(StackNode))

释放: stack_node_release (stack.c:107-117)
  ref_count 原子递减至 0 时:
    递归释放前驱 links
    释放 links 上的 subtree
    池未满(size < 50) → array_push (回收)
    否则 → ts_free
```

### 3.3 Parser 状态机状态的存储与转移

#### 3.3.1 状态存储

```
parse state 分布在三个地方:

1. 解析表 (编译时静态):
   TSLanguage.parse_table / small_parse_table
   ┌─────────┬─────────┬─────────┬─────┐
   │         │ token_A │ token_B │ ... │
   ├─────────┼─────────┼─────────┼─────┤
   │ state_0 │ S(5)    │ R(X,2)  │     │ ← S=Shift, R=Reduce
   │ state_1 │ R(Y,3)  │ S(8)    │     │
   │ ...     │         │         │     │
   └─────────┴─────────┴─────────┴─────┘

2. 栈节点 (运行时动态):
   StackNode.state = 当前解析状态 ID (TSStateId)

3. Subtree (运行时记录):
   SubtreeHeapData.parse_state = 产出此节点时的解析状态
   （用于增量解析时判断是否可复用）
```

#### 3.3.2 状态转移过程

```
状态转移发生在三个操作中:

Shift (token):
  current_state ──查表──→ action = Shift(next_state)
  创建 StackNode {state = next_state}
  push 到栈顶

Reduce (symbol, child_count):
  pop child_count 个节点
  回到底部 StackNode 的 state
  查 GOTO 表: GOTO(bottom_state, symbol) = next_state
  创建 StackNode {state = next_state}
  push 到栈顶

Error Recovery:
  state → ERROR_STATE (特殊状态 0)
  恢复时:
    搜索栈摘要找到对 lookahead 有效的历史 state
    弹出到该层, 恢复到 goal_state
```

### 3.4 错误恢复时状态变化

```
正常状态:
  Head → [s8] → [s5] → [s1]      所有状态有效

发现错误 (无有效动作):
  ts_stack_pause(version, lookahead)
  Head.status = Paused
  Head.lookahead_when_paused = lookahead

所有版本都 Paused 时:
  ts_stack_resume → Head.status = Active
  ts_parser__handle_error:

  ┌─────────────────────────────────────────────────────────┐
  │ 阶段 1: ts_parser__handle_error                          │
  │                                                          │
  │ 1. 不看 lookahead, 尝试所有可能的 reduce                 │
  │    → 可能产生新版本                                       │
  │                                                          │
  │ 2. 对每个版本尝试插入 MISSING token                      │
  │    → ts_subtree_new_missing_leaf(symbol)                 │
  │    → push + reduce                                       │
  │                                                          │
  │ 3. push NULL_SUBTREE 到 ERROR_STATE (state=0)            │
  │    Head → [s0] → [s8] → [s5] → [s1]                     │
  │                                                          │
  │ 4. 记录栈摘要 (最多 16 层):                               │
  │    summary = [(s8,depth=1,pos=6), (s5,depth=2,pos=3)]   │
  └─────────────────────┬───────────────────────────────────┘
                        │
                        ▼
  ┌─────────────────────────────────────────────────────────┐
  │ 阶段 2: ts_parser__recover                               │
  │                                                          │
  │ 策略 1 — 回溯到历史状态:                                  │
  │   for each summary_entry:                                │
  │     if ts_language_has_actions(entry.state, lookahead):  │
  │       ts_parser__recover_to_state(version, depth, state) │
  │       ├── pop depth 层                                   │
  │       ├── 合并弹出节点为 ERROR 节点                       │
  │       └── push 到 goal_state                             │
  │       Head → [goal_state] → ... (跳过了错误区域)         │
  │       break                                              │
  │                                                          │
  │ 策略 2 — 跳过当前 token:                                  │
  │   包装 lookahead 为 error_repeat 节点                    │
  │   push error_repeat 到 ERROR_STATE                       │
  │   Head → [s0: error_repeat] → [s0] → [s8] → ...         │
  │   继续消耗 token...                                       │
  └─────────────────────────────────────────────────────────┘

错误恢复代价计算:
  error_cost = ERROR_COST_PER_RECOVERY(500)
             + depth × ERROR_COST_PER_SKIPPED_TREE(100)
             + bytes × ERROR_COST_PER_SKIPPED_CHAR(1)
             + lines × ERROR_COST_PER_SKIPPED_LINE(30)
             + missing_count × ERROR_COST_PER_MISSING_TREE(110)
```

### 3.5 Parse Stack 在解析中的完整状态变化示例

```
输入: "let x = 1 + ;"  ('+' 后缺少操作数)

[位置0] 初始:  Head → [s1]

[位置0] Shift 'let':
  Head → [s5] ─"let"─→ [s1]

[位置4] Shift 'x':
  Head → [s8] ─"x"─→ [s5] ─"let"─→ [s1]

[位置6] Shift '=':
  Head → [s12] ─"="─→ [s8] ─"x"─→ [s5] ─"let"─→ [s1]

[位置8] Shift '1':
  Head → [s15] ─"1"─→ [s12] ─"="─→ [s8] ─"x"─→ ...

[位置10] Shift '+':
  Head → [s20] ─"+"─→ [s15] ─"1"─→ [s12] ─"="─→ ...

[位置12] lookahead = ';'
  查表: state=20, symbol=';' → 无动作!
  ts_stack_pause → status = Paused

  handle_error:
    尝试 reduce: 无法
    尝试 MISSING: 插入 MISSING(expression)
      → Shift MISSING → Reduce binary_expression → Reduce expression_statement
      → 恢复正常

  或: push ERROR_STATE, 跳过 ';', 回溯恢复
```

---

## 4. 跨语言边界的数据流

### 4.1 C API 的类型设计

```
┌─────────────────────────────────────────────────────────┐
│ api.h 类型分类                                           │
│                                                          │
│ 不透明类型 (仅前置声明, 通过指针使用):                    │
│   TSParser*     → struct TSParser (parser.c 内部)         │
│   TSTree*       → struct TSTree (tree.h 内部)             │
│   TSQuery*      → struct TSQuery (query.c 内部)           │
│   TSQueryCursor* → struct TSQueryCursor (query.c 内部)    │
│   TSLanguage*   → struct TSLanguage (parser.h 内部)       │
│                                                          │
│ 值类型 (完整定义, 按值传递):                              │
│   TSNode        → 32 字节 {context[4], id, tree}          │
│   TSTreeCursor  → {tree, id, context[3]}                  │
│   TSInput       → {payload, read, encoding, decode}       │
│   TSInputEdit   → {start/old_end/new_end × byte+point}   │
│   TSPoint       → {row, column}                           │
│   TSRange       → {start_point/end_point, start/end_byte} │
│                                                          │
│ 枚举:                                                    │
│   TSInputEncoding → UTF8 / UTF16LE / UTF16BE / Custom     │
│   TSSymbolType    → Regular / Anonymous / Supertype / Aux  │
│   TSLogType       → Parse / Lex                           │
└─────────────────────────────────────────────────────────┘
```

### 4.2 C → Rust 绑定 (binding_rust/lib.rs)

#### 4.2.1 类型包装映射

```
C 类型              Rust 包装                     所有权模型
─────────────────────────────────────────────────────────
TSParser*     →  Parser(NonNull<ffi::TSParser>)     唯一所有权, Drop 调用 ts_parser_delete
TSTree*       →  Tree(NonNull<ffi::TSTree>)         唯一所有权, Clone=ts_tree_copy, Drop=ts_tree_delete
TSNode        →  Node<'tree>(ffi::TSNode, PhantomData)  借用视图, 生命周期绑定 Tree
TSLanguage*   →  Language(*const ffi::TSLanguage)    引用计数, Clone=ts_language_copy, Drop=ts_language_delete
TSTreeCursor  →  TreeCursor<'tree>(ffi::TSTreeCursor, PhantomData, &'tree Tree)
TSQuery*      →  Query { ptr: NonNull, ..., language }
TSQueryCursor*→  QueryCursor { ptr: NonNull }
```

#### 4.2.2 内存安全边界

```
┌─────────────────────────────────────────────────────────┐
│ Rust 绑定的安全保证                                       │
│                                                          │
│ 1. Parser 唯一所有权:                                    │
│    pub struct Parser(NonNull<ffi::TSParser>);             │
│    impl Drop: ts_parser_delete                           │
│    不实现 Clone → 不可复制                                │
│                                                          │
│ 2. Tree 的 Clone 是浅拷贝:                               │
│    impl Clone: ts_tree_copy (共享子树图, root retain)    │
│    impl Drop: ts_tree_delete (release root)              │
│                                                          │
│ 3. Node 借用 Tree 的生命周期:                            │
│    pub struct Node<'tree>(ffi::TSNode, PhantomData<&'tree ()>)
│    → Rust 编译器保证 Node 不会超过 Tree 的生命周期       │
│    → 内部 TSNode.tree 指针自然有效                       │
│                                                          │
│ 4. Language 引用计数:                                    │
│    Clone → ts_language_copy (C 侧引用计数 +1)            │
│    Drop  → ts_language_delete (C 侧引用计数 -1)          │
│                                                          │
│ 5. Send + Sync:                                          │
│    unsafe impl Send/Sync for:                            │
│      Language, Node<'_>, Parser, Tree,                   │
│      TreeCursor<'_>, Query, QueryCursor,                 │
│      LookaheadIterator                                   │
│    → 信任 C 库的线程安全契约                              │
│    → 不同线程使用不同 Tree 拷贝是安全的                   │
│                                                          │
│ 6. unsafe 使用模式:                                      │
│    所有 ts_* FFI 调用包在 unsafe 块中                    │
│    slice::from_raw_parts, CStr::from_ptr 按 C 契约使用   │
│    ts_tree_included_ranges 返回的内存用匹配的 free 释放   │
└─────────────────────────────────────────────────────────┘
```

#### 4.2.3 Rust 绑定数据流图

```
Rust 用户代码                        Rust 绑定层                    C 运行时
─────────────                     ─────────────                 ──────────
let mut parser = Parser::new()  →  ts_parser_new()          →  malloc TSParser
parser.set_language(&lang)      →  ts_parser_set_language()  →  设置 language 指针

let tree = parser.parse(src, None)
  │
  ├── 构造 TSInput { payload, read }     ← Rust 闭包包装为 C 回调
  ├── ts_parser_parse(parser, None, input)
  │     │
  │     └── C 运行时完整解析流程...
  │         └── ts_tree_new() → TSTree*
  │
  └── Tree(NonNull::new(raw_tree))       ← 包装为 Rust Tree

let root = tree.root_node()
  │
  ├── ts_tree_root_node(tree.0)          → ffi::TSNode
  └── Node(raw_node, PhantomData)        ← 生命周期 'tree 绑定

let child = root.child(0)
  │
  ├── ts_node_child(root.0, 0)           → ffi::TSNode
  └── Option<Node<'tree>>               ← null 检查 → None

drop(tree)
  │
  └── ts_tree_delete(tree.0)             → ts_subtree_release + ts_free
```

### 4.3 C → Web/WASM 绑定 (binding_web/src/)

#### 4.3.1 内存交互模型

```
┌──────────────────────────────────────────────────────────────────────┐
│  JavaScript 层                                                       │
│                                                                      │
│  Parser {                              Tree {                        │
│    [0]: number (WASM 指针)               [0]: number (WASM 指针)     │
│    [1]: number (需 free 的辅助数据)      language: Language           │
│    language: Language                    textCallback: Function       │
│  }                                    }                              │
│                                                                      │
│  Node {                                TreeCursor {                   │
│    tree: Tree                           tree: Tree                   │
│    id: number                           [0]: WASM 地址              │
│    startIndex, startPosition            [1]: WASM 地址              │
│    [0]: number (other 字段)              [2]: WASM 地址              │
│  }                                    }                              │
│                                                                      │
├──────────────────────── TRANSFER_BUFFER ─────────────────────────────┤
│                                                                      │
│  共享内存区域 (C._ts_init() 返回的地址)                               │
│  用于 JS ↔ WASM 之间的参数/返回值传递                                │
│                                                                      │
│  marshalNode(node):                                                  │
│    C.setValue(addr, node.id, 'i32')                                  │
│    C.setValue(addr+4, node.startIndex, 'i32')                        │
│    C.setValue(addr+8, node.startPosition.row, 'i32')                 │
│    C.setValue(addr+12, node.startPosition.column, 'i32')             │
│    C.setValue(addr+16, node[0], 'i32')                               │
│                                                                      │
│  unmarshalNode(tree, addr):                                          │
│    从 TRANSFER_BUFFER 读取字段                                       │
│    构造 Node { tree, id, ... }                                       │
│                                                                      │
├──────────────────────── Emscripten WASM ─────────────────────────────┤
│                                                                      │
│  _ts_parser_new() → WASM 指针                                       │
│  _ts_parser_parse_wasm() → 通过共享内存传参                          │
│  _ts_tree_copy() → WASM 指针                                        │
│  _ts_tree_delete(address) → 释放 C 侧内存                           │
│  _ts_node_*_wasm() → 通过 TRANSFER_BUFFER 编组                      │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

#### 4.3.2 内存管理方式

| 对象 | 创建 | 销毁 | GC 兜底 |
|------|------|------|---------|
| **Parser** | `Parser.init()` → `_ts_parser_new` | 显式 `parser.delete()` | `FinalizationRegistry` |
| **Tree** | `parser.parse()` / `tree.copy()` | 显式 `tree.delete()` | `FinalizationRegistry` |
| **TreeCursor** | `tree.walk()` | 显式 `cursor.delete()` | `FinalizationRegistry` |
| **Node** | 从 TRANSFER_BUFFER 反编组 | 无需释放（JS 对象） | GC 自然回收 |
| **Query** | `Language.query()` | 显式 `query.delete()` | `FinalizationRegistry` |

**`FinalizationRegistry` 的局限**：JavaScript 规范不保证 finalizer 的执行时机。因此 Web 绑定**强烈依赖用户调用 `delete()`**，`FinalizationRegistry` 仅作为兜底防止内存泄漏。

#### 4.3.3 与 C TSNode 的关键差异

Web 绑定的 `Node` 不是直接映射 C 的 `TSNode` 结构体。每次调用 `ts_node_*_wasm` 前，先通过 `marshalNode` 将 Node 的关键字段写入 `TRANSFER_BUFFER`，调用 WASM 函数，再从 `TRANSFER_BUFFER` 读取结果。这是因为 WASM 函数无法直接接收复合结构体参数。

### 4.4 跨语言内存安全边界总结

```
┌──────────────────────────────────────────────────────────────────┐
│                    内存安全边界图                                  │
│                                                                   │
│  C 运行时 (lib/src/)                                              │
│  ├── 手动内存管理: malloc/free + 引用计数 + 对象池                 │
│  ├── 线程安全: 原子操作 (atomic_inc/dec)                          │
│  ├── 安全契约: TSTree 存活期间 TSNode 有效                        │
│  └── 可替换分配器: ts_set_allocator                               │
│       │                                                           │
│       │ FFI 边界                                                  │
│       ├───────────────────────────────────────────┐               │
│       │                                           │               │
│       ▼                                           ▼               │
│  Rust 绑定                                   Web/WASM 绑定        │
│  ├── 编译时保证:                              ├── 运行时保证:      │
│  │   ├── 所有权 (Drop)                       │   ├── 显式 delete()│
│  │   ├── 生命周期 ('tree)                    │   ├── FinalizationRegistry │
│  │   └── Send/Sync 标记                      │   └── TRANSFER_BUFFER 编组 │
│  │                                            │                   │
│  ├── 不安全边界:                              ├── 不安全边界:      │
│  │   └── unsafe FFI 调用                     │   └── WASM 指针管理│
│  │       (信任 C 契约)                       │       (手动 free)  │
│  │                                            │                   │
│  └── 零开销:                                  └── 编组开销:        │
│      Node 直接映射 TSNode                        marshalNode 每次  │
│      无序列化/反序列化                            读写共享内存      │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 附录：关键数据结构字段速查

### Subtree 相关

| 结构 | 文件 | 关键字段 |
|------|------|----------|
| `Subtree` (union) | `subtree.h` | `data: SubtreeInlineData` / `ptr: *SubtreeHeapData` |
| `SubtreeInlineData` | `subtree.h:52-99` | `is_inline`, `visible`, `named`, `extra`, `symbol`, `parse_state`, `padding_*`, `size_bytes` |
| `SubtreeHeapData` | `subtree.h:111-154` | `ref_count`, `padding`, `size`, `child_count`, `symbol`, `error_cost`, 布尔标志, 变体联合 |
| `SubtreePool` | `subtree.h` | `free_trees` (最多 32), `tree_stack` |
| `ExternalScannerState` | `subtree.h` | `short_data[24]` / `long_data*`, `length` |

### Parser 相关

| 结构 | 文件 | 关键字段 |
|------|------|----------|
| `TSParser` | `parser.c:83-114` | `lexer`, `stack`, `tree_pool`, `language`, `token_cache`, `reusable_node`, `finished_tree` |
| `TokenCache` | `parser.c:83-87` | `token`, `last_external_token`, `byte_index` |
| `TSParseAction` | `parser.h:66-81` | `shift{type,state,extra}`, `reduce{type,child_count,symbol,dynamic_precedence}` |
| `TSParseActionEntry` | `parser.h:94-100` | `action` / `entry{count, reusable}` |

### Stack 相关

| 结构 | 文件 | 关键字段 |
|------|------|----------|
| `Stack` | `stack.c:64-71` | `heads`, `slices`, `iterators`, `node_pool` (50), `base_node` |
| `StackHead` | `stack.c:55-62` | `node`, `summary`, `last_external_token`, `status` |
| `StackNode` | `stack.c:29-38` | `state`, `position`, `links[8]`, `ref_count`, `error_cost` |
| `StackLink` | `stack.c:23-27` | `node`, `subtree`, `is_pending` |

### Tree/Node 相关

| 结构 | 文件 | 关键字段 |
|------|------|----------|
| `TSTree` | `tree.h:17-22` | `root`, `language`, `included_ranges`, `included_range_count` |
| `TSNode` | `api.h:133-137` | `context[4]`, `id` (→Subtree*), `tree` (→TSTree*) |
| `TSTreeCursor` (内部) | `tree_cursor.h` | `tree`, `stack: Array(TreeCursorEntry)` |

### Lexer 相关

| 结构 | 文件 | 关键字段 |
|------|------|----------|
| `Lexer` | `lexer.h:18-38` | `data: TSLexer`, `current_position`, `chunk`, `input`, `included_ranges` |
| `TSLexer` | `parser.h:48-57` | `lookahead`, `result_symbol`, `advance`, `mark_end`, `get_column` |

### 常量速查

| 常量 | 值 | 文件 | 用途 |
|------|------|------|------|
| `TS_MAX_INLINE_TREE_LENGTH` | 255 | `subtree.c:22` | inline 节点最大字节数 |
| `TS_MAX_TREE_POOL_SIZE` | 32 | `subtree.c:23` | SubtreePool 最大缓存数 |
| `MAX_LINK_COUNT` | 8 | `stack.c:11` | StackNode 最大前驱链接数 |
| `MAX_NODE_POOL_SIZE` | 50 | `stack.c:12` | StackNode 池最大缓存数 |
| `MAX_ITERATOR_COUNT` | 64 | `stack.c:13` | pop 遍历迭代器上限 |
| `MAX_VERSION_COUNT` | 6 | `parser.c:78` | GLR 最大栈版本数 |
| `OP_COUNT_PER_PARSER_CALLBACK_CHECK` | 100 | `parser.c:81` | 进度回调检查频率 |

---

*文档生成时间：2026-03-27*
*基于 Tree-sitter v0.27.0 源码分析*
*对不确定的地方已标注【待确认】*
