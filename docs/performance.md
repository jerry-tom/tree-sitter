# Tree-sitter 性能设计分析

> 基于 Tree-sitter v0.27.0 源码分析
> 依赖文档：[architecture.md](./architecture.md)、[modules.md](./modules.md)、[businessflow.md](./businessflow.md)、[dataflow.md](./dataflow.md)

---

## 目录

1. [词法/语法分析效率](#1-词法语法分析效率)
2. [AST 构建与内存布局](#2-ast-构建与内存布局)
3. [符号表与作用域](#3-符号表与作用域)
4. [类型检查与语义分析](#4-类型检查与语义分析)
5. [算法复杂度](#5-算法复杂度)
6. [增量解析](#6-增量解析)
7. [工程实现](#7-工程实现)
8. [性能亮点总结表](#8-性能亮点总结表)

---

## 1. 词法/语法分析效率

### 1.1 Lexer 实现方式：生成的表驱动 switch-case 状态机

**文件**：`crates/generate/src/render.rs`（生成端）、`lib/src/lexer.c`（运行时）

Tree-sitter 的词法器既非手写递归下降，也非传统的 DFA 表驱动。它在**编译时**将词法 NFA 通过子集构造法转为 DFA（`build_lex_table.rs`），然后渲染为 C 语言的 **switch-case 嵌套**函数 `ts_lex()`。

```c
// 生成的 parser.c（示意）
static bool ts_lex(TSLexer *lexer, TSStateId state) {
  switch (state) {
    case 0:
      if (lookahead == 'i') ADVANCE(5);
      if (lookahead == 'f') ADVANCE(8);
      // ...
    case 5:
      if (lookahead == 'f') ADVANCE(12);
      ACCEPT_TOKEN(sym_identifier);
      // ...
  }
}
```

**性能收益**：

| 对比维度 | Tree-sitter (switch-case) | 传统 DFA 表驱动 | 手写递归下降 |
|----------|--------------------------|-----------------|-------------|
| 分支预测 | 编译器可优化为跳转表，CPU 分支预测友好 | 间接索引，每步需查表 | 最灵活但无法自动优化 |
| 指令缓存 | 热路径的 case 分支集中在 `.text` 段 | 表数据在 `.data` 段，需额外 cache line | 取决于代码组织 |
| 编译器优化 | 可内联、消除死分支、常量折叠 | 表结构限制了编译器优化空间 | 完全可优化 |
| 空间效率 | 只生成有效转移的代码 | 需要完整状态×字符表 | 代码量大 |

**代码位置**：
- 渲染 switch-case：`crates/generate/src/render.rs` → `Generator::add_lex_function()`
- 运行时调用入口：`lib/src/parser.c:341` → `ts_parser__call_main_lex_fn()`

### 1.2 上下文感知词法 — parse state 驱动 lex state

**文件**：`lib/src/language.h:73-93`、`crates/generate/src/build_tables/build_lex_table.rs`

传统工具（Flex/Bison）将词法和句法完全分离。Tree-sitter 在生成阶段利用每个 parse state 允许的 token 集合构建**不同的 lex state**：

```c
// language.h:73-93
static inline uint16_t ts_language_lookup(
  const TSLanguage *self, TSStateId state, TSSymbol symbol
) {
  if (state >= self->large_state_count) {
    // 小表: 分组压缩，线性搜索
    ...
  } else {
    // 大表: O(1) 直接索引
    return self->parse_table[state * self->symbol_count + symbol];
  }
}
```

每个 parse state 通过 `lex_modes[parse_state]` 映射到特定的 `lex_state`。不同的 parse state 可能共享同一个 `lex_state`（无词法冲突时合并），减少生成代码量。

**性能收益**：
- 词法器只匹配当前解析上下文中**有效的 token 子集**，减少无效匹配分支
- 例如 JSX 中 `<` 根据上下文可以是比较运算符或标签开始，无需回溯尝试
- 相比"词法先全量扫描再解析"的方案，减少了不必要的 token 化工作

### 1.3 双格式解析表 — 大表 O(1) + 小表压缩

**文件**：`lib/src/language.h:73-93`、`crates/generate/src/render.rs`

Tree-sitter 的解析表采用**双格式**设计：

```
parse_table 结构:
├── 大表 (state < large_state_count):
│   二维数组 parse_table[state × symbol_count + symbol]
│   O(1) 直接索引，适用于转移密集的状态
│
└── 小表 (state >= large_state_count):
    small_parse_table 压缩格式
    相同动作的符号分组存储
    通过 small_parse_table_map[state] 定位
    O(k) 线性搜索 (k=每组符号数，通常很小)
```

**性能收益**：
- **大表**（转移密集状态）：O(1) 查表，最大化运行时速度
- **小表**（转移稀疏状态）：分组压缩，显著减少内存占用
- 实际语法中，大部分状态是稀疏的（只有少量有效 token），小表压缩率很高
- 内存节省意味着更好的缓存命中率

**对比常见方案**：
- LALR/LR 工具（Bison）通常使用全密集表或基于偏移+检查（offset+check）的压缩，Tree-sitter 的分组方案在稀疏状态上更紧凑
- ANTLR (LL) 使用预测表或自适应预测，查询开销更高

### 1.4 Token 化策略 — 按需词法 + 缓存

**文件**：`lib/src/parser.c:505-701`、`lib/src/parser.c:703-737`

Tree-sitter 采用**按需词法分析**（lazy lexing）而非预先全量扫描：

```
ts_parser__advance():
  优先级 1: reuse_node     → 从旧树复用（增量解析，跳过 lex）
  优先级 2: cached_token   → 从缓存获取（同次解析的 GLR 分支共享）
  优先级 3: ts_parser__lex → 实际执行词法分析
```

**TokenCache** (`parser.c:83-87`)：

```c
typedef struct {
  Subtree token;
  Subtree last_external_token;
  uint32_t byte_index;
} TokenCache;
```

多个 GLR 栈版本在相同位置需要 lex 时，第一个版本的 lex 结果被缓存，后续版本直接复用。

**性能收益**：
- 按需 lex 避免了对未使用 token 的无效词法分析
- GLR 多版本并行时，TokenCache 避免重复 lex（N 个版本只 lex 一次）
- 增量解析时，`reuse_node` 优先于 lex，大量跳过词法分析

### 1.5 大字符集 set_contains 优化

**文件**：`crates/generate/src/render.rs`

生成的 `ts_lex()` 中，当字符集范围 ≥ 8 时，提取为常量数组，用 `set_contains()` 二分查找替代逐一比较：

```c
// 生成的 parser.c（示意）
static bool set_contains(TSCharacterRange *ranges, uint32_t n, int32_t c) {
  // 二分查找 O(log n)
  ...
}
// 使用
if (set_contains(ts_character_range_0, 42, lookahead)) ADVANCE(5);
```

**性能收益**：将 O(k) 的逐范围比较优化为 O(log k)。对于 Unicode 分类（如"所有字母字符"包含数十个范围）效果显著。

### 1.6 关键词消歧 — 两遍 lex 最小化代价

**文件**：`lib/src/parser.c:653-670`

Tree-sitter 通过 `keyword_capture_token` 机制处理关键字与标识符的歧义：

```
1. 主 lex_fn 匹配到 identifier
2. 若 symbol == keyword_capture_token:
   重置 lexer → 用 keyword_lex_fn 再 lex 一次
   若匹配到关键字且在当前 parse state 有效 → 替换 symbol
```

**性能收益**：
- 避免在主词法函数中为每个标识符检查所有关键字
- keyword_lex_fn 是独立的小状态机，只包含关键字规则，状态数少
- 绝大多数标识符不需要进入第二遍 lex（只有匹配到 `keyword_capture_token` 时才触发）

### 1.7 错误恢复对性能的影响

**文件**：`lib/src/parser.c:1250-1437`、`lib/src/error_costs.h`

Tree-sitter 的错误恢复使用**启发式代价模型**控制性能开销：

| 控制机制 | 值 | 说明 |
|----------|------|------|
| `MAX_VERSION_COUNT` | 6 | 最大 GLR 栈版本数硬上限 |
| `MAX_COST_DIFFERENCE` | 1800 | 版本间代价差超过此值强制淘汰 |
| `MAX_SUMMARY_DEPTH` | 16 | 错误恢复栈摘要最大深度 |
| 回溯搜索提前终止 | 代价超过最优版本时 break | 避免穷举搜索 |

**性能特征**：
- 正常解析（无错误）：零开销，错误恢复逻辑完全不触发
- 少量错误：受控开销，版本数量有严格上限（最多 6 个并行）
- 大量错误：启发式代价裁剪 + 贪心跳过策略，避免指数级搜索
- 代价比较是 O(N²) 其中 N ≤ 6，每次 condense_stack 最多 15 次比较

---

## 2. AST 构建与内存布局

### 2.1 Subtree Inline/Heap 双模式 — 指针标签位

**文件**：`lib/src/subtree.h:52-101, 156-160`

Tree-sitter 的核心节点数据结构使用 **tagged union**，将小节点（叶子）内联在 8 字节中：

```c
// subtree.h:156-160
typedef union {
  SubtreeInlineData data;       // inline 路径（8 字节，值类型）
  const SubtreeHeapData *ptr;   // heap 路径（指针）
} Subtree;
```

**判别机制**：利用指针对齐特性——合法堆指针的最低位始终为 0，因此 `is_inline = 1` 时不会与指针冲突。

**Inline 条件** (`subtree.c:155-164`)：
- `symbol ≤ 255`（8 bit）
- 无 external token
- `padding.bytes < 255`、`padding.extent.row < 16`
- `size.bytes < 255`、不跨行
- `lookahead_bytes < 16`

```
SubtreeInlineData 位域布局 (8 字节):
┌─────────┬────────┬────────────┬─────────────────────────────┐
│ 1b      │ 6b     │ 8b         │ 16b                         │
│is_inline│flags   │symbol      │parse_state                  │
├─────────┴────────┴────────────┴─────────────────────────────┤
│ 8b pad_cols│4b pad_rows│4b lookahead│8b pad_bytes│8b size   │
└─────────────────────────────────────────────────────────────┘
```

**性能收益**：

| 维度 | Inline (8 字节) | Heap (~80+ 字节) |
|------|-----------------|------------------|
| 分配 | 零分配（值类型，栈上构造） | malloc 或 pool 获取 |
| 释放 | 零释放 | 引用计数 + pool 归还 |
| 缓存 | 完美——与父结构体连续 | 需要解引用一次指针 |
| 引用计数 | 无 | 原子操作 |

**实际比例**：在典型源码中，大量 token（标识符、关键字、运算符、分号等）满足 inline 条件。据估计 **60-80% 的叶子节点可以 inline**，显著减少堆分配和引用计数操作。

**平台适配** (`subtree.h:67-101`)：针对 大端+32位 / 大端+64位 / 小端 三种组合分别调整 `is_inline` 位在联合体中的位置，确保跨平台正确性。

### 2.2 父节点连续内存分配 — 子节点 + HeapData 一次 malloc

**文件**：`lib/src/subtree.c:480-517`、`lib/src/subtree.h:248-255`

父节点的子节点数组和自身 HeapData 在**同一块连续内存**中分配：

```
内存布局:
┌──────────┬──────────┬─────┬──────────┬─────────────────┐
│ child[0] │ child[1] │ ... │ child[n] │ SubtreeHeapData │
│ (8 bytes)│ (8 bytes)│     │ (8 bytes)│   (父节点数据)   │
└──────────┴──────────┴─────┴──────────┴─────────────────┘
                                        ↑ ptr 指向这里
子节点访问: (Subtree *)(ptr) - child_count
```

```c
// subtree.h:248-249
static inline size_t ts_subtree_alloc_size(uint32_t child_count) {
  return child_count * sizeof(Subtree) + sizeof(SubtreeHeapData);
}
```

**性能收益**：
- **一次 malloc/free**：创建/销毁父节点只需一次分配操作，而非 1+N 次
- **缓存友好**：子节点数组与父节点元数据在同一缓存行附近，遍历子节点时内存访问连续
- **空间紧凑**：无需额外的指针/容量/大小字段来管理子数组

**对比常见方案**：
- 许多 AST 实现使用 `Vec<Node>` 或 `Node**` 作为子节点容器，需要独立分配和管理
- GCC/Clang 的 AST 节点使用单独的 bump allocator + 尾部柔性数组，方向类似但 Tree-sitter 的 union 设计更紧凑

### 2.3 SubtreePool — 叶子节点对象池

**文件**：`lib/src/subtree.c:121-151`

```c
#define TS_MAX_TREE_POOL_SIZE 32

static SubtreeHeapData *ts_subtree_pool_allocate(SubtreePool *self) {
  if (self->free_trees.size > 0) {
    return array_pop(&self->free_trees).ptr;  // O(1) 池取
  } else {
    return ts_malloc(sizeof(SubtreeHeapData));
  }
}

static void ts_subtree_pool_free(SubtreePool *self, SubtreeHeapData *tree) {
  if (self->free_trees.capacity > 0 && self->free_trees.size + 1 <= TS_MAX_TREE_POOL_SIZE) {
    array_push(&self->free_trees, (MutableSubtree){.ptr = tree});  // 回收
  } else {
    ts_free(tree);
  }
}
```

**设计要点**：
- 只池化**叶子节点**的 HeapData（固定大小 `sizeof(SubtreeHeapData)`）
- 父节点的变长连续内存不池化（大小不固定）
- 容量上限 32 个，防止池占用过多内存
- `ts_subtree_pool_new(0)` 创建容量为 0 的池——`ts_tree_delete`/`ts_tree_edit` 中使用，释放时直接 free 不回收

**性能收益**：
- 解析过程中叶子节点频繁创建/销毁（lex → shift → reduce 后被合并到父节点）
- 对象池将反复的 malloc/free 降低为 O(1) 的 push/pop 操作
- 32 个缓存项在典型解析中足以覆盖热路径的叶子节点分配

### 2.4 StackNode 对象池

**文件**：`lib/src/stack.c:11-13, 90-121`

```c
#define MAX_NODE_POOL_SIZE 50

// 分配
static StackNode *stack_node_new(...) {
  if (pool->size > 0) return array_pop(pool);
  return ts_malloc(sizeof(StackNode));
}

// 释放
if (pool->size < MAX_NODE_POOL_SIZE) {
  array_push(pool, self);
} else {
  ts_free(self);
}
```

解析栈节点同样采用对象池，最多缓存 50 个 StackNode。由于 GLR 解析中 StackNode 的创建/销毁更频繁（每次 shift/reduce 都会操作），池的效果更显著。

### 2.5 Tagged Union / Compact Representation

Tree-sitter 在多处使用了紧凑表示：

| 数据结构 | 紧凑设计 | 文件 |
|----------|----------|------|
| **Subtree** | inline 8 字节 / heap 指针 tagged union | `subtree.h:156-160` |
| **TSParseAction** | 4 种动作类型共用 union | `parser.h:66-81` |
| **TSParseActionEntry** | 头部+动作数组连续存储 | `parser.h:94-100` |
| **ExternalScannerState** | SSO 优化（≤24 字节 inline，否则堆分配） | `subtree.h:31-37` |
| **SubtreeHeapData** | 布尔标志用位域（11 个 bool 共用约 2 字节） | `subtree.h:121-131` |
| **SubtreeHeapData.union** | 非终结符/外部token/错误叶子三变体共用 union | `subtree.h:133-153` |

**性能收益**：紧凑表示减少内存占用，提高缓存行利用率。位域标志将 11 个 bool 压缩到约 2 字节，相比独立 bool 数组节省 9 字节。

### 2.6 非递归释放 — 显式栈防止深树栈溢出

**文件**：`lib/src/subtree.c:565-594`

```c
void ts_subtree_release(SubtreePool *pool, Subtree self) {
  if (self.data.is_inline) return;
  array_clear(&pool->tree_stack);

  if (atomic_dec(&self.ptr->ref_count) == 0) {
    array_push(&pool->tree_stack, ts_subtree_to_mut_unsafe(self));
  }

  while (pool->tree_stack.size > 0) {
    MutableSubtree tree = array_pop(&pool->tree_stack);
    if (tree.ptr->child_count > 0) {
      // 父节点: 对每个子节点 atomic_dec，为 0 则入栈
      for (uint32_t i = 0; i < tree.ptr->child_count; i++) {
        Subtree child = children[i];
        if (child.data.is_inline) continue;
        if (atomic_dec(&child.ptr->ref_count) == 0) {
          array_push(&pool->tree_stack, ts_subtree_to_mut_unsafe(child));
        }
      }
      ts_free(children);
    } else {
      ts_subtree_pool_free(pool, tree.ptr);
    }
  }
}
```

**性能收益**：
- 使用 `pool->tree_stack`（复用的显式栈）替代递归调用栈
- 避免深树（如高度数千的嵌套表达式）导致栈溢出
- `tree_stack` 在 SubtreePool 中复用，避免每次释放都分配新栈
- inline 节点 `continue` 跳过，减少原子操作次数

---

## 3. 符号表与作用域

Tree-sitter 作为语法解析器，**不构建传统意义的符号表**。符号查找和作用域管理通过 **Query 系统** 在解析后进行，属于上层能力。

### 3.1 Query 模式的编译时优化

**文件**：`lib/src/query.c`

TSQuery 编译后的数据结构针对匹配性能做了多处优化：

| 优化 | 机制 | 性能收益 |
|------|------|----------|
| **pattern_map 排序** | 按首步 symbol 排序，二分查找 O(log n) | 快速定位匹配的 pattern |
| **步骤展平存储** | 所有 pattern 的 step 展平在连续数组中 | 缓存友好的顺序访问 |
| **CaptureListPool** | 捕获列表的对象池 | 减少频繁的动态分配 |
| **root_pattern_guaranteed** | 静态分析标记，`next_capture` 提前返回确定的捕获 | 流式高亮的关键优化 |

### 3.2 SymbolTable — 双向映射

Query 内部的 `SymbolTable` 使用简单的 `Array(char)` 存储字符串，通过 `Slice(offset, length)` 引用。查找是线性的，但 pattern 数量通常很小（数十到数百），性能足够。

### 3.3 Highlight locals 作用域查找

`crates/highlight/src/highlight.rs` 中的 `@local.scope/definition/reference` 机制实现了简单的作用域感知：

- 维护 `scope_stack`：记录当前作用域链
- 定义时将符号加入当前作用域的 `local_defs`
- 引用时在 `scope_stack` 中自下而上查找

这是 O(S × D) 的查找（S = 作用域深度，D = 每个作用域的定义数），但由于 Tree-sitter 的 locals 查询通常只涉及局部变量，S 和 D 都很小。

---

## 4. 类型检查与语义分析

Tree-sitter 不执行类型检查或语义分析。它专注于**语法层面**的解析，产出 CST（具体语法树），将语义分析留给上层工具。

但 Tree-sitter 为上层的增量语义分析提供了关键基础设施：

### 4.1 增量树差异计算

**文件**：`lib/src/get_changed_ranges.c`、`lib/src/tree.c:72-94`

`ts_tree_get_changed_ranges(old_tree, new_tree)` 精确计算新旧树之间的差异范围，使上层工具（如 LSP 服务器）可以：
- 只对变化范围重新做语义分析
- 只重新渲染变化范围的高亮

### 4.2 Query 系统作为语义查询接口

Query 的 `next_capture` 流式 API 配合 `root_pattern_guaranteed` 静态分析，使得编辑器可以在树未完全遍历时就返回部分高亮结果，实现**流水线式语义标注**。

---

## 5. 算法复杂度

### 5.1 关键路径复杂度分析

| 操作 | 时间复杂度 | 空间复杂度 | 文件 |
|------|-----------|-----------|------|
| **完整解析** | O(n) 无歧义 / O(n · k) GLR（k = 最大版本数 ≤ 6） | O(n) 语法树 + O(d) 栈深度 | `parser.c` |
| **单步 lex** | O(L) L = token 长度 | O(1) | `lexer.c` |
| **解析表查询 (大表)** | O(1) | O(S × T) S=状态数 T=符号数 | `language.h:91` |
| **解析表查询 (小表)** | O(k) k=每组符号数 | O(紧凑) | `language.h:78-89` |
| **增量解析** | O(Δ + log n) Δ=变化区域大小 | O(1) 额外（复用旧树） | `parser.c` |
| **树编辑** | O(Δ · d) Δ=影响节点数 d=树深度 | O(d) 显式栈 | `subtree.c:633-786` |
| **变化范围计算** | O(n) 最坏 / O(Δ) 实际 | O(Δ) 结果范围数 | `get_changed_ranges.c` |
| **引用计数释放** | O(n) n=子树节点数 | O(d) 显式栈 | `subtree.c:565-594` |
| **树平衡** | O(r) r=重复深度差 | O(r) 栈 | `subtree.c:292-336` |
| **栈版本收缩** | O(k²) k ≤ 6 | O(1) | `parser.c:1766-1864` |
| **Query 匹配** | O(n × P) P=pattern 数 | O(P × M) M=最大匹配数 | `query.c` |

### 5.2 GLR 复杂度的实际约束

GLR 解析的理论复杂度为 O(n³) 在最坏情况下（高度歧义语法），但 Tree-sitter 通过以下机制将实际复杂度控制在 O(n · k)：

```
约束机制:
├── MAX_VERSION_COUNT = 6          ← 硬上限，O(k) = O(6) = O(1)
├── MAX_VERSION_COUNT_OVERFLOW = 4 ← Reduce 期间临时允许 10 个版本
├── MAX_COST_DIFFERENCE = 1800     ← 代价差过大直接淘汰
├── ErrorStatus 比较              ← 两两比较 O(k²) = O(36) = O(1)
└── ts_stack_merge                ← 同状态+同位置+同 error_cost 的版本合并
```

**对比**：
- ANTLR ALL(*) 在最坏情况下每次预测可能回溯整个输入，Tree-sitter 的 GLR 不需要回溯
- Bison LALR(1) 不支持歧义文法，遇到冲突需要用户手动消解

### 5.3 O(log n) 的重复节点平衡

**文件**：`lib/src/subtree.c:292-336`、`lib/src/parser.c:1867-1912`

对于 `a + b + c + d + ...` 这样的左递归重复模式，解析器天然产生 O(n) 深度的右偏链：

```
before:  _binary_expression
         ├── _binary_expression
         │   ├── _binary_expression
         │   │   ├── a
         │   │   └── b
         │   └── c
         └── d

after:   _binary_expression
         ├── _binary_expression
         │   ├── a
         │   └── b
         └── _binary_expression
             ├── c
             └── d
```

`ts_subtree_compress()` 执行类似 AVL 左旋的操作，将 O(n) 深度链压缩为 O(log n) 的平衡树。这对 TreeCursor 遍历和增量解析时的 descend 操作至关重要——防止 O(n) 深度导致 O(n) 的遍历时间。

---

## 6. 增量解析

### 6.1 增量解析的核心机制

**文件**：`lib/src/parser.c:753-830`、`lib/src/reusable_node.h`

增量解析是 Tree-sitter 最核心的性能特性。其设计思路：

```
旧树节点:  [   A   ] [   B   ] [ C (changed) ] [   D   ] [   E   ]
           ═══复用══  ═══复用══  ──重新解析──    ═══复用══  ═══复用══
```

1. **`ts_tree_edit()`** 标记被编辑影响的节点（`has_changes`），但不重新解析
2. **`ts_parser_parse(old_tree, new_input)`** 在主循环中通过 `ReusableNode` 沿旧树遍历：
   - 位置对齐且可复用 → 整块复用，跳过 lex
   - 不可复用 → 退回正常 lex → shift/reduce 流程

### 6.2 复用条件与保障

**文件**：`lib/src/parser.c:753-830`

节点复用需要通过以下全部检查：

```
ts_parser__reuse_node 复用决策树:
│
├── byte_offset > position → break (旧树节点在当前位置之后)
├── byte_offset < position → advance/descend (对齐位置)
├── byte_offset == position:
│   │
│   ├── external_scanner_state 不一致 → 拒绝
│   │
│   ├── has_changes         → 拒绝 (编辑影响了此节点)
│   ├── is_error            → 拒绝 (错误节点不稳定)
│   ├── is_missing          → 拒绝 (缺失节点不稳定)
│   ├── is_fragile          → 拒绝 (脆弱节点边界不确定)
│   ├── 与 included_range_differences 相交 → 拒绝
│   │
│   ├── lex_mode 不匹配     → 拒绝 (上下文不同)
│   ├── table_entry.is_reusable == false → 拒绝
│   ├── 空 token (非 EOF)    → 拒绝
│   │
│   └── 全部通过 → 复用! retain 并返回
```

### 6.3 复用率的保证策略

| 策略 | 机制 | 文件 |
|------|------|------|
| **has_changes 最小传播** | 只标记编辑影响的节点及其祖先，不扩散到兄弟 | `subtree.c:633-786` |
| **fragile 标记精确性** | 只有 GLR 多版本/错误节点的边界标记为 fragile | `subtree.c:459-460` |
| **descend 优先** | 不可复用时先下探子节点，尝试复用更小的子树 | `parser.c:810-815` |
| **lex_mode 检查** | 通过 `memcmp(&leaf_lex_mode, &current_lex_mode)` 确保上下文一致 | `parser.c:488-495` |
| **编辑只调整位置** | `ts_tree_edit` 不改变树结构，只修改 padding/size | `subtree.c:633-786` |
| **COW 语义** | `make_mut` 保证旧树引用不被破坏 | `subtree.c:284-290` |

### 6.4 避免重复计算

增量解析通过以下机制避免重复工作：

1. **节点复用 = 跳过 lex + shift**：复用的子树直接作为 lookahead 进入 Shift，完全跳过词法分析
2. **`first_leaf` 缓存**：每个父节点缓存其最左叶子的 `(symbol, parse_state)`，复用验证时无需遍历子树
3. **TreeCursor 遍历复用**：`ReusableNode` 使用轻量级 DFS 栈，复用旧树的子节点指针
4. **included_range_differences 预计算**：新旧 `included_ranges` 的差异在解析前一次性计算，解析中只做交叉检查

### 6.5 增量解析性能特征

```
典型编辑场景的性能模型:

修改一个 token (如重命名变量):
  重新 lex:    O(token_length)
  重新 reduce: O(tree_depth) ← 沿归约链到根
  复用:        O(n - Δ) 个节点直接复用
  总计:        O(Δ + d) ≈ O(d)，d=树深度

插入一行代码:
  重新 lex:    O(line_length)
  重新 reduce: O(受影响的产生式数量)
  复用:        编辑点前后的子树全部复用
  总计:        O(Δ · d)

删除一大块代码:
  ts_tree_edit: O(受影响节点数 × 深度)
  重新解析:     O(编辑区域两端的收敛距离)
  总计:         O(Δ · d)

所有场景共同特征:
  与文件总大小 n 基本无关，只与变化量 Δ 和树深度 d 相关
```

**对比**：ANTLR/Bison 不支持增量解析，任何修改都需要 O(n) 的全量重新解析。

---

## 7. 工程实现

### 7.1 单编译单元聚合 (Amalgamation)

**文件**：`lib/src/lib.c`

```c
#include "./alloc.c"
#include "./get_changed_ranges.c"
#include "./language.c"
#include "./lexer.c"
#include "./node.c"
#include "./parser.c"
#include "./point.c"
#include "./query.c"
#include "./stack.c"
#include "./subtree.c"
#include "./tree_cursor.c"
#include "./tree.c"
#include "./wasm_store.c"
```

将所有 `.c` 文件聚合为单个编译单元。

**性能收益**：
- **LTO 级优化无需 LTO**：编译器可在所有模块间做内联、常量传播、死代码消除
- 关键热路径函数（如 `ts_subtree_symbol()`、`ts_language_lookup()`）声明为 `static inline`，聚合编译确保可以跨文件内联
- **减少链接开销**：单个 `.o` 文件，无需符号解析

### 7.2 static inline 热路径函数

**文件**：`lib/src/subtree.h:232-384`、`lib/src/language.h:73-178`

大量高频访问的属性查询函数使用 `static inline`：

```c
// subtree.h — 所有 Subtree 属性访问函数
#define SUBTREE_GET(self, name) ((self).data.is_inline ? (self).data.name : (self).ptr->name)

static inline TSSymbol ts_subtree_symbol(Subtree self) { return SUBTREE_GET(self, symbol); }
static inline bool ts_subtree_visible(Subtree self) { return SUBTREE_GET(self, visible); }
static inline bool ts_subtree_named(Subtree self) { return SUBTREE_GET(self, named); }
// ... 约 20 个 inline 访问器
```

```c
// language.h — 解析表查询
static inline uint16_t ts_language_lookup(...) { ... }          // 每次 shift/reduce 都调用
static inline bool ts_lookahead_iterator__next(...) { ... }     // 迭代器热路径
```

**性能收益**：消除函数调用开销，对于每个 token 都要调用数十次的属性访问函数，内联带来的累积收益显著。

### 7.3 forceinline 关键路径

**文件**：`lib/src/stack.c:15-19`

```c
#if defined _WIN32 && !defined __GNUC__
#define forceinline __forceinline
#else
#define forceinline static inline __attribute__((always_inline))
#endif
```

Stack 模块中的关键迭代器操作使用 `forceinline`，强制编译器内联，即使在 `-O1` 优化级别。

### 7.4 原子操作 — 多线程安全的引用计数

**文件**：`lib/src/atomic.h`

```c
// GCC/Clang
static inline uint32_t atomic_inc(volatile uint32_t *p) {
  return __atomic_add_fetch(p, 1U, __ATOMIC_SEQ_CST);
}
static inline uint32_t atomic_dec(volatile uint32_t *p) {
  return __atomic_sub_fetch(p, 1U, __ATOMIC_SEQ_CST);
}

// MSVC
static inline uint32_t atomic_inc(volatile uint32_t *p) {
  return InterlockedIncrement((long volatile *)p);
}
```

- 使用 `__ATOMIC_SEQ_CST` 语义保证最强顺序一致性
- 跨平台抽象：GCC/Clang 使用 `__atomic_*` 内建，MSVC 使用 `Interlocked*`，TinyC 回退到非原子操作
- 只用于 `SubtreeHeapData.ref_count`，inline 节点完全跳过原子操作

**性能影响**：`SEQ_CST` 在 x86 上通常编译为 `lock xadd`，开销约 20-80 周期。但由于 inline 节点跳过、对象池减少分配/释放次数，实际的原子操作调用频率被大幅降低。

### 7.5 并发/并行设计

Tree-sitter **不使用多线程并行解析**。这是有意的设计选择：

| 方面 | 设计 |
|------|------|
| TSParser | 单线程使用，不可共享 |
| TSTree | 通过 `ts_tree_copy` 浅拷贝后可跨线程使用 |
| TSLanguage | 引用计数，可跨线程共享 |
| TSQuery | 编译后不可变，线程安全 |
| TSQueryCursor | 单线程使用 |

**设计理由**：
- 解析是 I/O bound（等待用户输入）而非 CPU bound
- 增量解析使得每次解析时间极短（亚毫秒级），不需要并行
- 单线程模型避免了锁竞争、数据竞争等复杂性
- `Send + Sync` 标记（Rust 绑定中）允许不同线程持有不同的 Tree 拷贝

### 7.6 输入读取与缓冲策略

**文件**：`lib/src/lexer.c:89-200`

```
TSInput 回调式读取:
  用户 → TSInput.read(byte_index, point) → const char* chunk
                                            ↑ 零拷贝，lexer 直接引用

Lexer 缓存:
  chunk: const char*        ← 当前文本块指针
  chunk_start: uint32_t     ← chunk 在文档中的起始偏移
  chunk_size: uint32_t      ← chunk 的字节大小
  current_position: Length   ← 当前读取位置
```

**性能收益**：
- **零拷贝**：`TSInput.read` 返回用户缓冲区的指针，Lexer 不拷贝数据
- **支持 rope 等编辑器数据结构**：回调式接口允许编辑器直接返回 rope 内部的 chunk 指针
- **按需读取**：只有 advance 超过当前 chunk 时才触发新的 `read` 调用
- **列号惰性计算**：`ColumnData` 缓存列号，只在 `get_column` 被调用时计算，避免每次 advance 都做行扫描

### 7.7 可替换内存分配器

**文件**：`lib/src/alloc.h`、`lib/src/alloc.c`

```c
TS_PUBLIC extern void *(*ts_current_malloc)(size_t size);
TS_PUBLIC extern void *(*ts_current_calloc)(size_t count, size_t size);
TS_PUBLIC extern void *(*ts_current_realloc)(void *ptr, size_t size);
TS_PUBLIC extern void (*ts_current_free)(void *ptr);

void ts_set_allocator(
  void *(*new_malloc)(size_t),
  void *(*new_calloc)(size_t, size_t),
  void *(*new_realloc)(void *, size_t),
  void (*new_free)(void *)
);
```

通过全局函数指针抽象内存分配，允许嵌入者替换为自定义分配器（如 arena allocator、jemalloc、mimalloc 等）。

**性能收益**：
- 允许编辑器将 Tree-sitter 的分配纳入全局内存池，减少碎片
- 默认分配器在 OOM 时直接 `abort()`，避免了到处检查 NULL 的开销

### 7.8 Array 动态数组 — 倍增策略

**文件**：`lib/src/array.h:242-253`

```c
static inline void *_array__grow(void *contents, uint32_t size, uint32_t *capacity,
                               uint32_t count, size_t element_size) {
  uint32_t new_size = size + count;
  if (new_size > *capacity) {
    uint32_t new_capacity = *capacity * 2;
    if (new_capacity < 8) new_capacity = 8;
    if (new_capacity < new_size) new_capacity = new_size;
    new_contents = _array__reserve(contents, capacity, element_size, new_capacity);
  }
  return new_contents;
}
```

- 最小容量 8，倍增扩容
- 包含 `array_search_sorted_by` 宏——内联的二分查找，用于排序数组
- 所有数组操作通过宏实现，避免泛型函数调用开销

### 7.9 进度回调节流

**文件**：`lib/src/parser.c:1536-1555`

```c
#define OP_COUNT_PER_PARSER_CALLBACK_CHECK 100

// 每 100 次操作检查一次 progress_callback
self->operation_count += operations;
if (self->operation_count >= OP_COUNT_PER_PARSER_CALLBACK_CHECK) {
  self->operation_count = 0;
  // 更新 parse_state，调用 progress_callback
}
```

**性能收益**：避免每次操作都调用回调函数。100 次操作批量检查一次，将回调开销降低了 ~100 倍。树平衡阶段的检查频率还会根据压缩大小动态调整（`i >> 4`）。

---

## 8. 性能亮点总结表

| # | 亮点 | 类别 | 文件 | 性能收益 | 对比常见方案 |
|---|------|------|------|----------|-------------|
| 1 | **Subtree Inline/Heap 双模式** | 内存 | `subtree.h:156-160` | 60-80% 叶子节点零分配、零引用计数 | 常规 AST 每个节点都需堆分配 |
| 2 | **父节点连续内存分配** | 内存 | `subtree.c:480-517` | 一次 malloc/free，缓存行连续访问 | 常规需要分别分配节点和子数组 |
| 3 | **增量解析 — 旧树节点复用** | 算法 | `parser.c:753-830` | 编辑后只重解析变化区域 O(Δ+d) | ANTLR/Bison 每次 O(n) 全量解析 |
| 4 | **双格式解析表** | 空间/时间 | `language.h:73-93` | 大表 O(1) 查表，小表压缩节省空间 | 传统工具用单一格式，空间或时间有妥协 |
| 5 | **上下文感知词法** | 算法 | `build_lex_table.rs` | 减少无效 token 匹配，消除词法歧义 | Flex 独立扫描，不知道解析上下文 |
| 6 | **生成的 switch-case 词法** | 指令缓存 | `render.rs` | 编译器优化为跳转表，分支预测友好 | DFA 表驱动需间接索引 |
| 7 | **SubtreePool 叶子对象池** | 内存 | `subtree.c:121-151` | 热路径 malloc/free 降为 O(1) push/pop | 无池化时每个 token 一次 malloc |
| 8 | **StackNode 对象池** | 内存 | `stack.c:82-122` | shift/reduce 的频繁分配降为池操作 | 每次 shift 一次 malloc |
| 9 | **TokenCache** | 时间 | `parser.c:83-87,703-737` | GLR 多版本同位置避免重复 lex | 无缓存时每个版本独立 lex |
| 10 | **重复节点树平衡** | 算法 | `subtree.c:292-336` | O(n) 深度链压为 O(log n)，加速遍历 | 不平衡的 CST 遍历性能退化 |
| 11 | **非递归释放** | 鲁棒性 | `subtree.c:565-594` | 显式栈防止深树栈溢出 | 递归释放在深树上可能 crash |
| 12 | **指针标签位** | 空间 | `subtree.h:44-101` | 零额外开销区分 inline/heap | 需要额外标志位或 nullable 指针 |
| 13 | **单编译单元聚合** | 编译优化 | `lib.c` | 跨模块内联，无需 LTO | 多编译单元需 LTO 才能跨模块优化 |
| 14 | **TSInput 零拷贝读取** | I/O | `lexer.c:89-200` | 直接引用用户缓冲区，无拷贝 | 许多解析器要求先拷贝到内部缓冲区 |
| 15 | **GLR 版本数硬上限** | 算法 | `parser.c:77` | O(n³) 理论复杂度降为 O(n·6) | 无限制 GLR 可能指数级分支 |
| 16 | **列号惰性计算** | 时间 | `lexer.c:47-68` | 只在需要时计算，避免每次 advance 行扫描 | 每次 advance 都更新列号 |
| 17 | **ExternalScannerState SSO** | 内存 | `subtree.h:31-37` | ≤24 字节 inline 存储，避免堆分配 | 每次都堆分配序列化状态 |
| 18 | **关键词两遍 lex** | 时间 | `parser.c:653-670` | 只在匹配到捕获 token 时做第二遍 | 主词法中逐一检查所有关键字 |
| 19 | **set_contains 二分查找** | 时间 | `render.rs` | 大字符集 O(log k) 替代 O(k) 逐一比较 | 逐范围线性比较 |
| 20 | **进度回调节流** | 时间 | `parser.c:1536-1555` | 每 100 次操作检查一次，降低回调开销 ~100× | 每次操作都检查 |
| 21 | **可替换内存分配器** | 灵活性 | `alloc.h` | 允许嵌入者使用 arena/pool allocator | 硬编码 malloc/free |
| 22 | **COW 写时复制** | 内存 | `subtree.c:284-290` | ref_count==1 时零拷贝就地修改 | 每次修改都深拷贝 |
| 23 | **first_leaf 缓存** | 时间 | `subtree.h:142-145,265-268` | 复用验证无需遍历子树找最左叶子 | 每次复用都要 DFS 到叶子 |
| 24 | **位域布尔标志** | 空间 | `subtree.h:121-131` | 11 个 bool 压缩到 ~2 字节 | 独立 bool 需 11 字节 |

---

## 附录：性能设计模式索引

### A. 内存优化模式

| 模式 | 应用 | 适用场景 |
|------|------|----------|
| Tagged Union (指针标签) | Subtree inline/heap | 小对象高频创建，多数可放入机器字 |
| 对象池 | SubtreePool, StackNode pool | 固定大小对象的频繁创建/销毁 |
| 连续分配 | 父节点 + 子数组 | 关联数据需要同时访问 |
| SSO (Small String Optimization) | ExternalScannerState | 短数据高频出现 |
| COW (Copy on Write) | ts_subtree_make_mut | 多引用共享，少量修改 |
| 引用计数 + 显式栈释放 | Subtree ref_count | 共享数据图的生命周期管理 |

### B. 时间优化模式

| 模式 | 应用 | 适用场景 |
|------|------|----------|
| 按需计算 (Lazy) | 列号计算、按需 lex | 不一定需要的计算 |
| 缓存 (Memoization) | TokenCache、first_leaf | 重复计算同一结果 |
| 增量计算 | 增量解析、summarize_children | 局部变化不需全量重算 |
| 批量检查 (Throttle) | 进度回调每 100 次 | 高频低价值检查 |
| 二分查找 | set_contains、pattern_map | 有序集合的成员检查 |
| 编译时特化 | parse state → lex state | 运行时不变的分派 |

### C. 架构优化模式

| 模式 | 应用 | 适用场景 |
|------|------|----------|
| 单编译单元 | lib.c amalgamation | 需要跨模块优化 |
| 零拷贝 I/O | TSInput 回调 | 避免不必要的数据复制 |
| 表驱动 + 代码生成 | lex_fn switch-case | 利用编译器优化 |
| 有界并行 | GLR MAX_VERSION_COUNT=6 | 控制分支爆炸 |
| 分层压缩 | 大表/小表双格式 | 空间与时间的平衡 |

---

*文档生成时间：2026-03-27*
*基于 Tree-sitter v0.27.0 源码分析*
*对不确定的地方已标注【待确认】*
