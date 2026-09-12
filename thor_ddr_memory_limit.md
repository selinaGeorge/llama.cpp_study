# 🎯 Thor 开发板 DDR 内存限制指南

## 📊 内存限制方案概述

在 Thor 开发板上限制 DDR 使用，将物理 128GB 内存分配为：
- **系统内存**: 8GB
- **推理内存**: 8GB  
- **未分配**: 112GB

---

## 🎯 限制 DDR 使用的方法

### 方法 1：Linux 内核启动参数（最简单、推荐）

修改启动参数限制可用内存，适合所有应用全局限制。

#### 步骤：

1. **编辑 GRUB 配置文件**
```bash
sudo nano /etc/default/grub
```

2. **找到 `GRUB_CMDLINE_LINUX_DEFAULT` 行，添加 `mem=16G` 参数**
```bash
# 原始
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"

# 修改后（限制到 16GB）
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash mem=16G"
```

3. **更新 GRUB 并重启**
```bash
sudo update-grub
sudo reboot
```

4. **验证**
```bash
free -h
# 输出应该显示 ~16GB 可用内存
```

**优点**：
- ✅ 简单易行
- ✅ 全局生效
- ✅ 持久化配置

**缺点**：
- ❌ 影响整个系统
- ❌ 系统应用也会受限

---

### 方法 2：Cgroup 控制（推荐用于容器/进程）

针对特定进程或容器限制内存，更灵活。

#### 2.1 使用 Cgroup v2（现代 Linux）

```bash
# 创建 cgroup
sudo mkdir -p /sys/fs/cgroup/user.slice/llama_inference.slice

# 限制内存到 8GB
echo "8589934592" | sudo tee /sys/fs/cgroup/user.slice/llama_inference.slice/memory.max

# 启动你的推理进程（假设是 ./main）
sudo systemd-run --scope -p MemoryMax=8G ./main -m model.gguf
```

#### 2.2 使用 systemd service（推荐）

创建 systemd 服务文件限制内存：

```bash
# 创建服务文件
sudo nano /etc/systemd/system/llama-inference.service
```

```ini
[Unit]
Description=Llama Inference Service
After=network.target

[Service]
Type=simple
User=your_user
WorkingDirectory=/home/your_user/llama.cpp
ExecStart=/home/your_user/llama.cpp/main -m model.gguf
MemoryMax=8G
MemoryLimit=8G

# 可选：保留 8GB 给系统，其余 8GB 给推理
# MemoryMax=8589934592

[Install]
WantedBy=multi-user.target
```

```bash
# 启用并启动服务
sudo systemctl daemon-reload
sudo systemctl enable llama-inference
sudo systemctl start llama-inference

# 查看状态
systemctl status llama-inference
```

**优点**：
- ✅ 针对性强，只限制推理程序
- ✅ 系统资源独立
- ✅ 容易调整

**缺点**：
- ❌ 配置稍复杂
- ❌ 需要 root 权限

---

### 方法 3：Docker 容器隔离（如果你用容器）

```bash
docker run -it \
  -m 8g \
  --memory-swap 8g \
  -v /path/to/models:/models \
  my-llama-image \
  ./main -m /models/model.gguf
```

---

### 方法 4：NUMA 内存绑定（高级，适合 Thor）

Thor 上可能有 NUMA 架构，可以绑定到特定 NUMA 节点：

```bash
# 查看 NUMA 配置
numactl --hardware

# 绑定进程到特定 NUMA 节点，限制内存
numactl --cpunodebind=0 --membind=0 \
  -l ./main -m model.gguf --n-gpu-layers 32
```

---

## 🧪 验证和监控

### 实时监控内存使用

```bash
# 方式 1：watch 命令实时监控
watch -n 1 'free -h && echo "---" && ps aux | grep main'

# 方式 2：使用 vmstat
vmstat 1

# 方式 3：Cgroup 监控（如果用 cgroup）
cat /sys/fs/cgroup/user.slice/llama_inference.slice/memory.current
cat /sys/fs/cgroup/user.slice/llama_inference.slice/memory.peak
```

### 测试推理时的内存占用

```bash
# 启动推理，同时监控内存
timeout 60 /usr/bin/time -v ./main -m model.gguf -p "Hello" -n 128

# 输出会显示：
# Maximum resident set size (kbytes): XXXXX
```

---

## 🎯 针对你的方案：16GB 总限制 + 8GB 系统 + 8GB 推理

**推荐组合方案**：

```bash
# 1. 系统级别限制到 16GB（可选）
# 编辑 /etc/default/grub，添加 mem=16G

# 2. 启动推理进程时限制到 8GB
sudo systemd-run --scope -p MemoryMax=8G \
  ./main -m model.gguf \
  --n-gpu-layers 32 \
  --batch-size 1 \
  --ctx-size 2048
```

或者创建专用服务：

```ini
# /etc/systemd/system/llama-moe-inference.service

[Unit]
Description=Llama MoE Inference with SSD Offloading
After=network.target

[Service]
Type=simple
User=your_user
WorkingDirectory=/path/to/llama.cpp
ExecStart=/path/to/llama.cpp/main \
  -m /path/to/moe-model.gguf \
  --weight-cache-dir /mnt/ssd/llama_weights \
  --max-ram-cache 8G \
  --n-gpu-layers 32

# 内存限制
MemoryMax=8G
MemoryLimit=8G

# CPU 亲和性（可选）
CPUAffinity=0-15

# 日志
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

---

## ⚠️ 重要注意事项

| 考虑因素 | 说明 |
|--------|------|
| **OOM Killer** | 如果超过限制，Linux 会杀死进程。调整 `memory.min` 保证最小内存 |
| **交换分区** | 如果有交换分区，可能会减缓性能。禁用：`--memory-swap` 设置相同值 |
| **GPU 显存** | 这是 GPU 内存，不受系统 RAM 限制。可用 `--n-gpu-layers` 独立控制 |
| **SSD 缓存** | 你的权重卸载方案需要 SSD 有足够空间，与 RAM 限制无关 |

---

## 💡 对你的 MoE 推理计划的影响

```
方案组合：
┌─ 物理内存: 128 GB
├─ 系统分配: 8 GB (系统运行)
├─ 推理限制: 8 GB (llama.cpp)
└─ 未使用: 112 GB (保留/未分配)

好处：
✅ 推理进程可预测的内存占用
✅ 系统稳定性有保障
✅ 便于性能对标
✅ SSD 缓存成为主要权重存储媒介
```

---

## 🚀 建议实现顺序

1. **第一步**：使用 `systemd-run` 临时测试
```bash
sudo systemd-run --scope -p MemoryMax=8G ./main -m test.gguf
```

2. **第二步**：验证 8GB 限制下的推理是否正常（需要 SSD 缓存）

3. **第三步**：创建 systemd service 持久化配置

4. **第四步**：集成到你的权重卸载实现中

---

## 📝 配置检查清单

```
□ 确认 Thor 的当前 BIOS/UEFI 固件支持内存限制
□ 了解 Thor 是否为 NUMA 架构
□ 测试不同内存限制下的系统稳定性
□ 设置监控工具（sysstat、iotop 等）
□ 创建 systemd service 配置
□ 测试在 8GB 限制下的 MoE 推理
□ 与 SSD 缓存方案集成
□ 性能基准测试
```

---

## 🔗 相关资源

- [Linux Cgroup 官方文档](https://man7.org/linux/man-pages/man7/cgroups.7.html)
- [systemd 内存限制文档](https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html)
- [NUMA 架构优化](https://www.kernel.org/doc/html/latest/vm/numa.html)
- [NVIDIA Jetson Thor 文档](https://developer.nvidia.com/jetson-thor)
