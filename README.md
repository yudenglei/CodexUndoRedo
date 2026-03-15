# CodexUndoRedo

## 面向现代 C++ CAE/PCB 的 Undo/Redo：Immer 实战与事务机制对比

本文聚焦你给定的 PCB 场景：`Trace` 包含头尾与中间连接类型、网络、线宽和一系列 `Segment`（直线段/弧线段），并讨论在 10w / 100w / 800w 量级下，Immer、OCAF、KLayout 事务机制的性能与内存特性。

---

## 1) 如何用 Immer 实现 Trace/Via 的新建、删除、修改

> 核心思路：把 PCB 设计主数据建成 **不可变状态树**（persistent data structure），每次编辑返回一个新版本，Undo/Redo 只是在版本指针间移动。

### 1.1 数据模型（示例）

```cpp
#include <immer/map.hpp>
#include <immer/vector.hpp>
#include <immer/box.hpp>
#include <string>
#include <variant>
#include <cstdint>

using Id = std::uint64_t;

struct Point {
    double x{};
    double y{};
};

enum class ConnectKind {
    HeadPad,     // trace 头部连接 pad/pin
    TailPad,     // trace 尾部连接 pad/pin
    MidTee       // 中间 T 连接/分叉连接
};

struct StraightSeg {
    Point p0;
    Point p1;
};

struct ArcSeg {
    Point center;
    double radius{};
    double a0{}; // start angle
    double a1{}; // end angle
    bool cw{};
};

using Segment = std::variant<StraightSeg, ArcSeg>;

struct Trace {
    Id id{};
    Id net_id{};
    double width{};
    ConnectKind head_conn{ConnectKind::HeadPad};
    ConnectKind tail_conn{ConnectKind::TailPad};
    immer::vector<Segment> segments;
};

struct Via {
    Id id{};
    Id net_id{};
    Point pos;
    double drill{};
    double diameter{};
};

struct PcbState {
    // 主数据：trace/via/net 关系（这里只简化）
    immer::map<Id, Trace> traces;
    immer::map<Id, Via> vias;
};
```

### 1.2 事务与历史栈

```cpp
#include <vector>
#include <stdexcept>

struct History {
    std::vector<PcbState> undo_stack;
    std::vector<PcbState> redo_stack;
    PcbState current;

    explicit History(PcbState init) : current(std::move(init)) {}

    template <typename F>
    void commit(F&& mutator) {
        auto next = mutator(current); // 返回新状态
        undo_stack.push_back(current);
        current = std::move(next);
        redo_stack.clear();
    }

    bool can_undo() const { return !undo_stack.empty(); }
    bool can_redo() const { return !redo_stack.empty(); }

    void undo() {
        if (!can_undo()) throw std::runtime_error("nothing to undo");
        redo_stack.push_back(current);
        current = undo_stack.back();
        undo_stack.pop_back();
    }

    void redo() {
        if (!can_redo()) throw std::runtime_error("nothing to redo");
        undo_stack.push_back(current);
        current = redo_stack.back();
        redo_stack.pop_back();
    }
};
```

### 1.3 新建 Trace

```cpp
auto add_trace(const PcbState& s, Trace t) {
    if (s.traces.find(t.id) != nullptr) {
        throw std::runtime_error("trace id exists");
    }
    auto next = s;
    next.traces = next.traces.set(t.id, std::move(t));
    return next;
}
```

### 1.4 删除 Trace

```cpp
auto remove_trace(const PcbState& s, Id trace_id) {
    if (s.traces.find(trace_id) == nullptr) {
        return s; // 无操作
    }
    auto next = s;
    next.traces = next.traces.erase(trace_id);
    return next;
}
```

### 1.5 修改 Trace（线宽、网络、segment 编辑）

```cpp
auto update_trace_width(const PcbState& s, Id trace_id, double new_width) {
    auto p = s.traces.find(trace_id);
    if (!p) throw std::runtime_error("trace not found");

    auto tr = *p;
    tr.width = new_width;

    auto next = s;
    next.traces = next.traces.set(trace_id, std::move(tr));
    return next;
}

auto append_straight_seg(const PcbState& s, Id trace_id, StraightSeg seg) {
    auto p = s.traces.find(trace_id);
    if (!p) throw std::runtime_error("trace not found");

    auto tr = *p;
    tr.segments = tr.segments.push_back(Segment{seg});

    auto next = s;
    next.traces = next.traces.set(trace_id, std::move(tr));
    return next;
}

auto replace_segment(const PcbState& s, Id trace_id, std::size_t idx, Segment seg) {
    auto p = s.traces.find(trace_id);
    if (!p) throw std::runtime_error("trace not found");

    auto tr = *p;
    if (idx >= tr.segments.size()) throw std::runtime_error("segment index out of range");

    tr.segments = tr.segments.set(idx, std::move(seg));

    auto next = s;
    next.traces = next.traces.set(trace_id, std::move(tr));
    return next;
}
```

### 1.6 使用方式（对应新建/删除/修改）

```cpp
History hist{PcbState{}};

// 1) 新建 trace
hist.commit([&](const PcbState& s) {
    Trace t;
    t.id = 1001;
    t.net_id = 88;
    t.width = 0.15;
    t.head_conn = ConnectKind::HeadPad;
    t.tail_conn = ConnectKind::TailPad;
    t.segments = t.segments.push_back(Segment{StraightSeg{{0,0},{10,0}}});
    t.segments = t.segments.push_back(Segment{ArcSeg{{10,5},5,270,180,true}});
    return add_trace(s, std::move(t));
});

// 2) 修改 width
hist.commit([&](const PcbState& s) {
    return update_trace_width(s, 1001, 0.20);
});

// 3) 删除 trace
hist.commit([&](const PcbState& s) {
    return remove_trace(s, 1001);
});

// Undo / Redo
hist.undo();
hist.redo();
```

### 1.7 面向百万级规模的关键落地点

1. **大对象不要整体复制**：Immer 的结构共享能避免深拷贝，但你的 `Trace` 结构体仍要保持“小而稳定”，超大几何缓存应外置（ID 引用）。
2. **分区化状态**：按板、层、区域切片成多个 map，降低单次写放大。
3. **批量编辑走单事务**：例如推挤布线一次操作里改变几百条 trace，应聚合成一次 `commit`。
4. **派生数据不入主历史**：DRC 缓存、三角网格、渲染 BVH 只记录 `dirty`，后台重建。
5. **快照节流**：每 N 次 commit 做 checkpoint，历史栈其余版本可压缩/分级缓存。

---

## 2) Immer 在哪些软件中使用过？

结论要务实：**Immer（arximboldi/immer）在工业 EDA/CAE 的公开案例不算多**，它更常见于追求函数式不可变数据结构的 C++ 项目、研究原型、编辑器内核或游戏工具链。原因是很多大型 CAD/EDA 核心历史包袱深，更多采用“可变对象 + 命令日志/增量记录”。

因此建议你的决策方式是：
- 把 Immer 当做 **核心状态层技术选型**（若团队接受函数式风格）；
- 先在 trace/via/net 子域试点，再决定是否全面推广到 geometry/kernel 层。

---

## 3) Immer vs OCAF vs KLayout（PCB trace/via 场景）

> 这里给的是工程上常见趋势对比，具体数值会受数据分布、分区策略、内存分配器、事务粒度影响。

### 3.1 机制差异

1. **Immer**
   - 本质：不可变持久化容器 + 结构共享
   - Undo：版本指针回退
   - 优点：并发读天然友好、回滚简单
   - 风险：写路径常数开销较高；对象设计不当会有额外分配成本

2. **OCAF（Open CASCADE）事务/Delta**
   - 本质：文档对象可变 + Delta 记录
   - Undo：按 Delta 回滚
   - 优点：CAD 几何语义强、属性系统成熟
   - 风险：跨域（PCB 网络关系/仿真）需较多桥接层；调优复杂

3. **KLayout 事务（编辑器命令/数据库变更）**
   - 本质：偏编辑器工作流的变更记录/撤销
   - 优点：版图编辑实战成熟、操作语义直接
   - 风险：若扩展到“多物理场仿真 + 复杂对象关系”，需要额外事务编排

### 3.2 在 10w / 100w / 800w 规模的趋势对比（定性）

| 规模 | Immer | OCAF | KLayout式事务 |
|---|---|---|---|
| 10w 对象 | 开发体验好，Undo/Redo 清晰；内存可控 | 稳定，集成几何友好 | 编辑操作响应通常很好 |
| 100w 对象 | 需分区+对象瘦身；写放大要重点优化 | Delta 管理复杂度上升，但成熟度高 | 取决于你对数据库与命令层改造程度 |
| 800w 对象 | 若不分层分片会吃紧；需冷热分层/外置大块数据 | 可做但工程成本高；事务链调优难 | 作为基础编辑事务可用，但跨域统一事务需重构 |

### 3.3 性能与内存核心结论（PCB trace/via）

1. **若你追求多线程读取一致性 + 简洁 undo 语义**：Immer 很有吸引力。  
2. **若你已有重 OCCT/OCAF 体系且几何核心占主导**：延续 OCAF 成本更低。  
3. **若你已有类似 KLayout 的编辑器内核**：可保留命令事务层，但建议在底层引入“分区快照+结构共享”思想。  
4. 对 800w 级别，**没有任何机制可以“裸跑”**：必须做分区、延迟加载、派生缓存外置、事务批处理、内存池。  

---

## 4) 结合你的场景的推荐落地路线

1. **第一阶段（低风险）**：
   - 保留现有命令式事务接口；
   - 在 trace/via/net 子域引入 Immer 状态层；
   - 先打通 Undo/Redo + DRC dirty 传播。

2. **第二阶段（扩展）**：
   - 把 board 按区块分片（空间分桶/层分桶）；
   - 引入 checkpoint + redo branch 管理；
   - 建立大事务（自动布线/批量规则修改）压缩策略。

3. **第三阶段（百万级优化）**：
   - 热/冷数据拆分，几何细节页式加载；
   - 仿真/网格派生缓存异步重建；
   - 事务日志持久化（崩溃恢复 + 回放定位）。

---

## 5) 一个 Trace + Via 混合事务例子（语义级）

一次用户动作：
- 把 `Trace#1001` 的尾部改接 `Via#5002`；
- 新增一段弧线规避障碍；
- 线宽从 0.15 改为 0.18；
- 标记局部 DRC 和阻抗仿真缓存失效。

在 Immer 模式中就是一次 `commit`：
1. 修改 trace 连接类型（tail 从 `TailPad` -> `MidTee`/via 连接语义）；
2. `segments.push_back(ArcSeg{...})`；
3. `width=0.18`；
4. 更新 `via.net_id` 一致性；
5. 仅记录 `dirty_region_ids`（派生缓存不入主状态）。

Undo 即回退到前一版本指针，所有上述变化原子撤销。
