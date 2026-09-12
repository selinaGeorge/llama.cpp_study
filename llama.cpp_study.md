# 🎯 超大 MoE 权重卸载到 SSD 实现计划

## 📊 背景分析

### Thor 开发板规格

- **系统 RAM**: 128 GB LPDDR5X
- **存储**: 256GB-1TB NVMe SSD（PCIe Gen5）
- **GPU**: NVIDIA Blackwell 架构
- **优势**: 内存充足，SSD 速度快（PCIe Gen5）

### 你的挑战

- MoE 模型虽然只激活少数专家，但**权重存储量巨大**
- 即使 128GB RAM，超大模型仍可能超出限制
- 需要**动态权重加载**机制

---

## 📋 五阶段实现计划

### 第一阶段：基础学习（2-3 周）

**目标**：深入理解 llama.cpp 的架构和权重管理

#### 1.1 学习 llama.cpp 核心概念

```
□ 克隆 llama.cpp 仓库并编译
□ 理解 GGML 模型格式（权重序列化格式）
□ 分析主要模块结构：
  ├── llama.cpp - 核心逻辑
  ├── ggml.c/h - 张量计算框架
  ├── ggml-backend.c/h - 后端管理
  ├── ggml-cuda.cu - CUDA 实现
  └── ggml-metal.m - Metal 实现
```

#### 1.2 学习现有权重卸载机制

```
□ 研究 `--n-gpu-layers` 参数的实现
□ 分析层级分割逻辑
□ 理解 CPU/GPU 张量分配策略
□ 查看 ggml-backend buffer 管理
```

#### 1.3 理解 MoE 架构

```
□ 学习 Mixtral、GLaM 等 MoE 模型结构
□ 理解专家路由机制
□ 分析稀疏激活特性
□ 对比 MoE vs Dense 模型的内存需求
```

**输出物**：

- 一份 llama.cpp 架构文档
- MoE 权重存储分析（哪些权重被频繁访问）
- Thor 板上的内存容量规划

---

### 第二阶段：SSD 权重缓存系统设计（2-3 周）

**目标**：设计权重卸载到 SSD 的完整方案

#### 2.1 权重分类和分析

```
□ 分类权重：
  ├── 频繁访问（Embedding、Router、Output Layer）
  ├── 中等访问（激活的专家权重）
  └── 低频访问（未激活的专家权重）

□ 实现权重访问频率统计工具
□ 估算每层权重大小和加载时间
```

#### 2.2 缓存策略设计

```
选择缓存策略（从简到复杂）：

方案 A - 简单预加载（推荐起点）：
  • 模型启动时：高频权重 → RAM，专家权重 → SSD
  • 推理时：仅加载当前批次激活的专家
  • 缺点：批次间等待时间

方案 B - 智能预取（中级）：
  • 基于路由信息预测下一个 token 的专家
  • 后台异步加载下一批权重
  • 优点：减少阻塞时间

方案 C - 分层多级缓存（高级）：
  • SSD → 系统 RAM → GPU显存 三级缓存
  • LRU 淘汰策略
  • 动态调整各级容量
```

#### 2.3 存储格式和元数据

```
□ 设计权重文件布局：
  ├── Header: 模型配置 + 权重位置表
  ├── Embedding/Router层 - 预加载区
  └── 专家权重 - 动态加载区

□ 创建权重索引文件：
  {
    "layer_id": {
      "offset": 12345,
      "size": 67890,
      "access_pattern": "sparse",
      "experts": [0, 5, 12, ...]
    }
  }
```

**输出物**：

- 详细的缓存策略文档（带性能模型）
- 权重布局设计
- 系统架构图
- 性能预估（吞吐量、延迟、I/O 开销）

---

### 第三阶段：核心模块实现（3-4 周）

**目标**：实现权重卸载引擎的核心组件

#### 3.1 权重管理层（llama-weight-manager）

```cpp
class LlamaWeightManager {
  // 权重元数据
  std::map<WeightID, WeightMetadata> weights_;
  
  // 内存管理
  RAMPool ram_pool_;      // 系统 RAM 缓冲
  SSDCache ssd_cache_;    // SSD 缓存
  
  // 核心接口
  Tensor* getWeight(WeightID id);           // 获取权重（自动加载）
  void preloadWeights(std::vector<WeightID>);
  void evictWeights(std::vector<WeightID>);
  
  // 异步 I/O
  Future<Tensor*> getWeightAsync(WeightID id);
};
```

#### 3.2 SSD 缓存模块（ssd-cache）

```cpp
class SSDCache {
  // 配置
  std::string cache_dir_;
  size_t max_ram_size_;
  
  // 核心方法
  void init(const std::string& model_path);
  Tensor* load(WeightID id);           // 从 SSD 加载到 RAM
  void save(WeightID id, const Tensor&);
  bool exists(WeightID id);
  
  // LRU 淘汰
  void evictLRU(size_t target_size);
};
```

#### 3.3 路由预测器（router-prefetcher）

```cpp
class RouterPrefetcher {
  // 分析路由历史
  void recordRouting(const std::vector<int>& expert_ids);
  
  // 预测下一个专家
  std::vector<int> predictNextExperts();
  
  // 触发异步预加载
  void prefetchExperts(const std::vector<int>& ids);
};
```

#### 3.4 集成到 llama.cpp

```
修改点：
□ llama.cpp 中的权重加载逻辑
□ ggml-backend：替换为��持 SSD 的后端
□ 推理循环：插入权重管理钩子
□ 参数新增：--weight-cache-dir, --max-ram-cache, --ssd-prefetch
```

**输出物**：

- 完整的权重管理模块（生产级代码）
- 单元测试和性能测试
- 集成到 llama.cpp 的补丁

---

### 第四阶段：MoE 特化优化（2-3 周）

**目标**：针对 MoE 模型的特定优化

#### 4.1 MoE 感知的权重布局

```
优化 SSD 文件组织：
□ 按专家分组存储（而非按层）
□ 热专家聚集在一起（减少 SSD 碎片化）
□ 路由权重独立存储（常驻 RAM）
```

#### 4.2 专家动态加载策略

```cpp
class MoEWeightManager : public LlamaWeightManager {
  // 专家级管理
  void loadExpert(int expert_id);
  void unloadExpert(int expert_id);
  
  // 批次处理
  void processBatchExperts(
    const std::vector<std::vector<int>>& routing,  // 每个 token 的专家 ID
    const std::vector<Tensor>& tokens
  );
  
  // 统计信息
  void updateExpertStats(const std::vector<int>& used_experts);
  std::vector<int> getFrequentExperts(int topK);
};
```

#### 4.3 性能优化

```
□ 批处理：合并相同专家的计算
□ 并行化：利用 PCIe Gen5 带宽充分读取
□ 压缩：对低精度专家（int8/fp16）进行压缩存储
□ 工作流优化：
  1. 路由 → 2. 预加载下批专家 → 3. 计算当前专家 → 4. 聚合
```

**输出物**：

- MoE 感知的权重管理器
- 性能基准测试工具
- 优化调试指南

---

### 第五阶段：测试、优化和部署（2-3 周）

**目标**：验证系统性能并优化生产就绪

#### 5.1 功能测试

```
□ 单元测试：
  ├── 权重加载/卸载正确性
  ├── 缓存淘汰策略
  ├── 异步 I/O
  └── 错误恢复

□ 集成测试：
  ├── 在 Thor 上运行 Mixtral、GLaM 等模型
  ├── 对比标准 llama.cpp 的输出一致性
  └── 验证内存使用不超限
```

#### 5.2 性能基准测试

```
测试场景：
1. 不同模型大小（7B, 13B, 30B, 70B）
2. 不同批大小（1, 4, 8, 16）
3. 不同缓存策略（A、B、C）

关键指标：
• 首 token 延迟 (TTFT)
• 单 token 延迟 (TPS)
• 内存峰值
• SSD I/O 吞吐 (GB/s)
• CPU/GPU 利用率

输出：性能对比表、瓶颈分析
```

#### 5.3 优化迭代

```
□ 性能分析（profiling）
  ├── 用 perf/nvprof 找瓶颈
  ├── I/O 优化：调整预取大小、并发度
  └── 计算优化：kernel fusion、量化

□ 内存优化
  ├── 减少 RAM 缓冲区大小
  ├── 压缩权重格式
  └── 共享缓冲池

□ 对用户友好性
  ├── 自动缓存调优
  ├── 清晰的性能报告
  └── 故障诊断工具
```

#### 5.4 文档和发布

```
□ API 文档
□ 使用指南
□ 性能调优手册
□ 贡献指南
□ 考虑向 llama.cpp 官方提交 PR
```

**输出物**：

- 完整的测试套件（测试覆盖率 > 80%）
- 性能报告和优化建议
- 用户文档
- 可部署的二进制和源码

---

## 📅 时间线总结

| 阶段 | 任务 | 周数 | 关键输出 |
|------|------|------|---------|
| 1 | 基础学习 | 2-3 | 架构文档、权重分析 |
| 2 | 方案设计 | 2-3 | 缓存策略、性能模型 |
| 3 | 核心实现 | 3-4 | 权重管理器、SSD 缓存 |
| 4 | MoE 优化 | 2-3 | MoE 感知管理器、基准 |
| 5 | 测试部署 | 2-3 | 完整系统、文档 |
| **总计** | | **11-16 周** | **可部署产品** |

---

## 🛠️ 工具和资源

### 开发环境

```
□ CUDA Toolkit (for Thor GPU)
□ CMake 3.20+
□ Git
□ Python 3.9+ (for scripts/analysis)
```

### 关键依赖

```
□ llama.cpp (fork & modify)
□ Boost (异步 I/O)
□ nlohmann/json (配置)
□ spdlog (日志)
```

### 性能分析工具

```
□ NVIDIA Nsys/Nsight Compute (GPU profiling)
□ perf/flamegraph (CPU profiling)
□ iotop (I/O monitoring)
□ numactl (内存绑定)
```

---

## ✅ 里程碑检查清单

```
□ 第 1-3 周末：完成架构文档，提交设计审查
□ 第 7 周末：完成权重管理器核心实现（单元测试通过）
□ 第 11 周末：在 Thor 上跑通 Mixtral 7B，内存占用 < 80GB
□ 第 14 周末：性能达标（>10 tokens/sec），所有测试通过
□ 第 16 周末：完整文档，发布 v1.0
```

---

## 🚀 后续方向（可选）

```
□ 支持分布式部署（多块 Thor 板）
□ 动态量化优化（运行时精度调整）
□ 训练时权重优化（合作学习友好的存储格式）
□ Web API 服务部署
```

---

## 💡 我能帮你的具体事项

1. **代码审查**：你实现的每个模块
2. **架构设计**：讨论权重加载策略
3. **问题排查**：性能瓶颈分析、bug 修复
4. **文档编写**：API 文档、使用指南
5. **性能优化**：Profiling 结果分析、优化建议

---

**下一步**：你想怎么推进？

1. 🔍 查看和分析 llama.cpp 源代码
2. 📐 深入讨论某个阶段���设计细节
3. 💻 开始实现第一个模块的代码框架
