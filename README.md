# CodexUndoRedo

## C++17 + Qt5.13.1 + OCAF 双系统：面向百万级高速 PCB 多板的 Undo/Redo 事务架构

> 目标：保留现有 OCAF 生产体系，同时新增一套高性能低内存的新事务内核（建议基于 Immer 持久化容器），形成 **OCAF + NewCore 双系统架构**。新内核中的参数化变量必须可回映到 OCAF 的总 Label 体系。

---

## 1. 约束前提与总体方案

你给出的约束非常明确：

1. 技术栈固定：**C++17、Qt 5.13.1、OCAF**。
2. 现状：OCAF 在百万级器件下内存压力大。
3. 要求：新增新框架，但参数变量依旧与 OCAF 总 Label 对齐。
4. 场景：高速 PCB 多板，含分层对象（Trace/Surface）和跨层对象（Via/BondWire 等）。

### 1.1 推荐架构：双写一致 + 渐进切换

- **LegacyCore（OCAF）**：保留现有业务逻辑与参数化语义。
- **NewCore（Immer State）**：承载高频编辑、Undo/Redo、并发读。
- **Bridge（Label 参数桥）**：维护 OCAF Label 与 NewCore ParamId 的双向映射。
- **SyncPolicy**：按场景选择同步策略：
  - 交互编辑：NewCore 主写，异步回写 OCAF；
  - 关键出图/持久化：强一致 flush 到 OCAF。

---

## 2. 参数化变量必须兼容 OCAF Label：如何设计

核心原则：**NewCore 不复制 OCAF 的参数语义，而是引用 OCAF Label 的参数路径作为“参数主键”**。

### 2.1 统一参数主键

```cpp
// C++17
using LabelPath = std::string;   // 例如 "0:1:5:18"
using ParamName = std::string;   // 例如 "x", "y", "width", "drill"

struct ParamKey {
    LabelPath root_label;   // OCAF 总 Label 或其子树路径
    ParamName name;         // 参数名

    bool operator==(const ParamKey& r) const {
        return root_label == r.root_label && name == r.name;
    }
};
```

> 生产实现中建议把 `LabelPath` 编码为压缩整数路径，减少字符串开销。

### 2.2 参数值表达式（支持参数化）

```cpp
#include <variant>
#include <vector>
#include <cstdint>

struct LiteralDouble { double v{}; };
struct RefParam { ParamKey key; }; // 引用 OCAF 参数

enum class BinOp : std::uint8_t { Add, Sub, Mul, Div };

struct Expr;
struct BinaryExpr {
    BinOp op;
    std::shared_ptr<Expr> lhs;
    std::shared_ptr<Expr> rhs;
};

struct Expr {
    std::variant<LiteralDouble, RefParam, BinaryExpr> node;
};
```

这样 `Trace/Via/BondWire` 的坐标、尺寸都可写成表达式：
- 绝对值：`LiteralDouble{10.0}`
- 参数引用：`RefParam{ {"0:1:5", "board_thickness"} }`
- 参数计算：`drill = ref("via_base") * 0.8`

### 2.3 OCAF/NewCore 双向同步

- **OCAF -> NewCore**：监听 Label 参数变更，增量更新 NewCore 参数缓存。
- **NewCore -> OCAF**：事务提交时输出 ParamDelta（变更键值对），批量回写 Label。
- **冲突规则**：按版本戳（`ocaf_rev`, `newcore_rev`）做乐观冲突检测。

---

## 3. 数据分层：哪些对象可以“像 KLayout 一样按层管理”？

结论：**可分层，但要分“几何层”和“逻辑层”**。

### 3.1 适合强分层的数据（几何主导）

1. **Trace（走线）**：天然属于某导体层。
2. **Surface（铜皮）**：层内区域对象。
3. **Keepout / Text / Region**：通常层内独立。

这部分可参考 KLayout 的层组织方式：
- 容器按 `LayerId -> ObjectMap` 管理；
- 层可独立加载、独立重建索引、独立渲染。

### 3.2 不应只按层管理的数据（跨层/复合语义）

1. **Via（pad + drill）**：几何跨层，且和网络、电气规则强耦合。
2. **BondWire**：连接跨器件，可能跨层/跨封装语义。
3. **DiffPair / NetClass 约束对象**：逻辑关系优先。
4. **器件实例（含 3D 封装）**：通常需要板级/装配级层次管理。

### 3.3 推荐：二维索引模型（LayerIndex + RelationIndex）

- **LayerIndex（按层）**：服务几何查询、渲染、局部编辑。
- **RelationIndex（按关系图）**：服务网络、约束、器件连接。

任何对象至少落一个主索引：
- Trace/Surface：LayerIndex 为主，RelationIndex 为辅。
- Via/BondWire：RelationIndex 为主，LayerIndex 为辅（仅存可见几何切片）。

---

## 4. C++17 + Immer 的事务实现示例（Trace/Via）

> 说明：以下代码偏架构示意，重点体现可落地的数据组织方式和 Undo/Redo 语义。

### 4.1 数据结构

```cpp
#include <immer/map.hpp>
#include <immer/vector.hpp>
#include <variant>
#include <cstdint>

using Id = std::uint64_t;
using LayerId = std::uint16_t;

struct P2Expr { Expr x; Expr y; }; // 坐标参数表达式

struct StraightSeg { P2Expr p0; P2Expr p1; };
struct ArcSeg {
    P2Expr center;
    Expr radius;
    Expr a0;
    Expr a1;
    bool cw{};
};
using Segment = std::variant<StraightSeg, ArcSeg>;

struct Trace {
    Id id{};
    Id net_id{};
    LayerId layer{};
    Expr width;                          // 宽度支持参数表达式
    immer::vector<Segment> segments;
    ParamKey owner_label;                // 对应 OCAF Label 根
};

struct Via {
    Id id{};
    Id net_id{};
    P2Expr pos;
    Expr drill;
    Expr diameter;
    ParamKey owner_label;
};

struct BoardState {
    // 几何层索引（示意：layer->traceIds）
    immer::map<LayerId, immer::vector<Id>> layer_traces;

    // 主对象表
    immer::map<Id, Trace> traces;
    immer::map<Id, Via> vias;

    // 参数缓存（key -> 当前求值结果）
    immer::map<std::string, double> param_cache;
};
```

### 4.2 Undo/Redo 历史

```cpp
#include <vector>

struct History {
    std::vector<BoardState> undo_stack;
    std::vector<BoardState> redo_stack;
    BoardState current;

    template <class F>
    void commit(F&& f) {
        BoardState next = f(current);
        undo_stack.push_back(current);
        current = std::move(next);
        redo_stack.clear();
    }

    void undo() {
        if (undo_stack.empty()) return;
        redo_stack.push_back(current);
        current = undo_stack.back();
        undo_stack.pop_back();
    }

    void redo() {
        if (redo_stack.empty()) return;
        undo_stack.push_back(current);
        current = redo_stack.back();
        redo_stack.pop_back();
    }
};
```

### 4.3 Trace 新建/删除/修改（含层索引维护）

```cpp
BoardState add_trace(const BoardState& s, Trace t) {
    BoardState n = s;
    n.traces = n.traces.set(t.id, t);

    auto ids = n.layer_traces.find(t.layer)
        ? *n.layer_traces.find(t.layer)
        : immer::vector<Id>{};
    ids = ids.push_back(t.id);
    n.layer_traces = n.layer_traces.set(t.layer, ids);
    return n;
}

BoardState remove_trace(const BoardState& s, Id trace_id) {
    auto p = s.traces.find(trace_id);
    if (!p) return s;

    BoardState n = s;
    const auto& tr = *p;

    // 从主表删除
    n.traces = n.traces.erase(trace_id);

    // 从层索引删除（线性示例，生产建议二级索引）
    auto ids_ptr = n.layer_traces.find(tr.layer);
    if (ids_ptr) {
        auto ids = *ids_ptr;
        immer::vector<Id> out;
        for (auto id : ids) if (id != trace_id) out = out.push_back(id);
        n.layer_traces = n.layer_traces.set(tr.layer, out);
    }
    return n;
}

BoardState update_trace_width(const BoardState& s, Id trace_id, Expr new_width) {
    auto p = s.traces.find(trace_id);
    if (!p) return s;

    auto tr = *p;
    tr.width = std::move(new_width);

    BoardState n = s;
    n.traces = n.traces.set(trace_id, std::move(tr));
    return n;
}
```

### 4.4 OCAF 回写（事务提交后）

```cpp
struct ParamDelta { std::string key; double value; };
using ParamDeltaList = std::vector<ParamDelta>;

// 伪代码：把 NewCore 已提交状态中变更参数批量写回 OCAF Label
void flush_to_ocaf(const ParamDeltaList& deltas) {
    // Qt5.13.1 工程中可放在专用工作线程，最终 UI 通知走 signal/slot
    // for (const auto& d : deltas) {
    //    TDF_Label label = resolve_label(d.key);
    //    set_ocaf_real_attribute(label, d.value);
    // }
}
```

---

## 5. Immer 使用注意事项（性能/内存）

下面按“必须做 / 不建议做”给出。

### 5.1 必须做（高收益）

1. **对象瘦身**：`Trace` 里只放必要字段；大缓存（网格、三角化、仿真矩阵）外置到缓存系统。
2. **分片状态**：按 board/layer/region 分片，不要全板单棵大 map。
3. **批量事务**：一次交互动作中合并多个小修改，减少版本数量。
4. **稳定 ID**：64-bit 全局 ID，避免容器搬移导致关联失效。
5. **派生数据 dirty 化**：Undo/Redo 只回滚主数据，派生缓存重建。
6. **自定义分配器策略**：结合 arena/池化，减少小对象分配抖动。

### 5.2 不建议做（常见踩坑）

1. 在事务中频繁“读-改-写同对象”数百次（应先局部聚合再一次 set）。
2. 把超长字符串直接作为热路径键（改为整数键或 intern）。
3. 把 UI 临时状态塞进主历史（会爆栈并污染撤销语义）。
4. 不做 checkpoint，导致长历史链回收慢、内存波动大。

### 5.3 Immer 常用容器与 PCB 适配建议

1. `immer::map<K,V>`
   - 用途：对象主表（trace/via/component）。
   - 建议：`K` 用整数 ID；`V` 保持轻量。

2. `immer::vector<T>`
   - 用途：segment 列表、layer 内对象列表。
   - 建议：对超长 segment 序列做分块（例如 path-chunk）。

3. `immer::set<T>`
   - 用途：选择集、dirty 对象集合。
   - 建议：短生命周期集合可和事务外临时容器搭配。

4. `immer::table<K,V>`（哈希表语义）
   - 用途：高频 key/value 查询。
   - 建议：若键分布均匀，查询性能更稳。

5. `immer::box<T>`
   - 用途：共享大对象引用（谨慎使用）。
   - 建议：用于少量大对象元数据，避免过深复制。

---

## 6. 与 OCAF / KLayout 方式在 10w、100w、800w 规模的对比（PCB Trace/Via 场景）

> 说明：以下为工程实践上的趋势对比，不是绝对 benchmark 数值。

### 6.1 10w 级

- **OCAF**：成熟稳定，集成成本低。
- **NewCore(Immer)**：编辑撤销语义清晰，响应通常更稳。
- **KLayout 风格分层**：几何编辑高效，易实现层级可见性管理。

### 6.2 100w 级

- **OCAF**：单文档属性/Label 膨胀明显，需强优化。
- **NewCore(Immer)**：若已分片 + 对象瘦身，内存/撤销体验通常优于“全量深拷贝”模型。
- **KLayout 风格**：层内仍高效，但 Via/BondWire/约束等跨层语义需要额外关系层。

### 6.3 800w 级

- **仅靠 OCAF**：内存与事务成本高，风险大。
- **仅靠分层几何**：无法覆盖复杂关系一致性。
- **推荐**：
  - OCAF 作为参数与持久化语义权威源；
  - NewCore 作为编辑事务与高频读取内核；
  - 双索引（Layer + Relation）+ 增量同步 + checkpoint。

---

## 7. 针对你的场景的最终落地建议

1. **先做双系统而非一次性替换**：风险最小。
2. **参数统一到 OCAF Label Key**：保证历史兼容与工程可迁移。
3. **Trace/Surface 先分层迁移，Via/BondWire 同步建设关系索引**。
4. **Undo/Redo 以 NewCore 为主，OCAF 做最终一致回写**。
5. **上线前必须做三档压测**：10w、100w、800w；指标至少包含：
   - 单次事务延时（P50/P95）
   - Undo/Redo 延时
   - 常驻内存与峰值内存
   - OCAF 回写时延

---

## 8. Qt 5.13.1 集成建议（简要）

1. UI 层（Qt）只发事务请求，不直接写核心状态。
2. NewCore 在工作线程提交，提交完成发 `signal` 通知视图刷新。
3. OCAF 回写可异步批处理，关键节点（保存/导出）执行强一致 flush。
4. 大事务（自动布线）采用进度可取消任务，避免 UI 卡死。

---

如果你愿意，下一步我可以继续给你一份：
- `C++17` 头文件级接口草图（`TransactionCoordinator`、`ParamBridge`、`LayerRelationIndex`）；
- 一套可直接用于压测 10w/100w/800w 的基准测试模板（含指标采集点）。
