# Ascend NPU CI 集成验证体系设计方案

## 文档说明

**文档定位**：完整 CI 体系方案设计，涵盖镜像基础设施、门禁验证、测试策略的整体架构。

**设计原则**：参考 PyTorch 社区的 CI 实践，识别必要任务，构建适合 torch-npu 的集成验证体系。

**目标读者**：CI 基础设施开发者、torch-npu 维护者、测试工程师。

---

## 一、项目背景与定位

### 1.1 为什么需要中间 CI 仓库

**核心问题**：torch-npu 作为 PyTorch 的 NPU 设备扩展，代码存储在独立仓库 `Ascend/pytorch`，而非 PyTorch 主社区。这导致：

1. **缺少主仓门禁保护**：CUDA/ROCm/XPU 等设备的代码在 PyTorch 主仓，每次 PR 都会触发设备测试，但 torch-npu 没有这种保护
2. **集成验证滞后**：torch-npu 开发者无法及时发现与 PyTorch nightly 的兼容性问题
3. **测试环境不一致**：缺少标准化的测试镜像和环境配置

**解决方案**：建设中间 CI 仓库（pytorch-infra），提供：
- **门禁验证**：torch-npu PR 自动触发与 PyTorch nightly 的集成测试
- **每日集成**：定期验证 torch-npu master 与 PyTorch nightly 的兼容性
- **标准化环境**：构建和维护稳定的测试镜像

### 1.2 与 PyTorch 社区 CI 的对比

| 维度 | PyTorch 主仓（CUDA/ROCm/XPU） | torch-npu（中间 CI 仓库） |
|------|------------------------------|--------------------------|
| **代码位置** | PyTorch 主仓 `torch/cuda/` | 独立仓库 `Ascend/pytorch` |
| **门禁触发** | PR 提交自动触发 | 需要跨仓库触发机制 |
| **PyTorch 版本** | 主仓源码（最新） | PyTorch nightly wheel |
| **测试环境** | GitHub Actions + 自托管 Runner | 需要自建 NPU Runner |
| **镜像管理** | PyTorch 官方镜像 | 需要自建 CANN + PyTorch 镜像 |
| **测试策略** | Smoke + 分片测试 + 分布式 | 参考主仓，适配 NPU |

**关键差异**：
- PyTorch 主仓使用**源码构建**，中间仓库使用 **nightly wheel**（因为 torch-npu 依赖已发布的 PyTorch）
- PyTorch 主仓的镜像由官方维护，中间仓库需要**自建镜像体系**（CANN + PyTorch + torch-npu）

### 1.3 参考 PyTorch 社区的关键实践

通过分析 PyTorch 主仓的 CI 配置（`.github/workflows/`），识别以下必要任务：

#### 1.3.1 镜像基础设施（参考 `docker/` 目录）

PyTorch 社区维护分层镜像：
```
pytorch/pytorch:latest
  ├── Base: Ubuntu + Python + 编译工具
  ├── CUDA: CUDA Toolkit + cuDNN
  └── PyTorch: 预装 PyTorch + 测试依赖
```

**torch-npu 需要类似架构**：
```
ascend-ci/test-npu:latest
  ├── Base: Ubuntu + Python + 编译工具
  ├── CANN: CANN Toolkit + HCCL
  ├── PyTorch: PyTorch nightly
  └── torch-npu: 构建的 torch-npu wheel
```

#### 1.3.2 门禁测试策略（参考 `_linux-test.yml`）

PyTorch 主仓的测试分层：
1. **Smoke Test**（5 分钟）：快速验证基本功能
2. **Default Test**（30-60 分钟）：核心测试套件，6-12 分片并行
3. **Distributed Test**（60+ 分钟）：多卡分布式测试
4. **Inductor Test**（60+ 分钟）：编译器测试

**torch-npu 采用相同策略**：
- Layer 1: Smoke（必须，阻塞 PR）
- Layer 2: Device-Agnostic（必须，阻塞 PR）
- Layer 3: NPU-Specific（必须，阻塞 PR）
- Layer 4: Distributed + Inductor（可选，每日运行）

#### 1.3.3 Runner 配置（参考 `_linux-test.yml` 的 `runs-on`）

PyTorch 使用：
- `ubuntu-latest`：CPU 测试
- `linux.g5.4xlarge.nvidia.gpu`：CUDA 测试（AWS）
- `linux.rocm.gpu`：ROCm 测试（自托管）

**torch-npu 需要**：
- `self-hosted, npu-910b`：NPU 单卡测试
- `self-hosted, npu-910b-4`：NPU 多卡测试（分布式）

### 1.4 中间 CI 仓库的架构定位

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     torch-npu ↔ PyTorch 集成验证架构                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────┐         ┌──────────────────┐         ┌─────────────┐ │
│  │  Ascend/pytorch  │         │  pytorch-infra   │         │pytorch/main │ │
│  │  (torch-npu)     │────────▶│  (中间CI仓库)     │◀────────│(PyTorch官方)│ │
│  │                  │         │                  │         │             │ │
│  │  - NPU适配代码   │  触发   │  - 镜像构建      │  拉取   │ - nightly   │ │
│  │  - NPU算子实现   │  门禁   │  - PR门禁        │  wheel  │ - 源码      │ │
│  │  - NPU测试用例   │         │  - 每日集成      │         │             │ │
│  └──────────────────┘         │  - Issue追踪     │         └─────────────┘ │
│                                └──────────────────┘                         │
│                                                                             │
│  核心功能：                                                                  │
│  1. 镜像基础设施：构建和维护 CANN + PyTorch + torch-npu 测试镜像             │
│  2. PR 门禁：torch-npu PR 自动触发集成测试，结果反馈到 PR                    │
│  3. 每日集成：定期验证 torch-npu master + PyTorch nightly 兼容性            │
│  4. Issue 追踪：失败时自动创建 issue，修复后自动关闭                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**与现有每日验证的关系**：
- **现有**：每日构建 torch-npu，发现编译失败时创建 issue（已实现）
- **新增**：PR 门禁 + 完整测试套件 + 镜像基础设施（本方案）
- **协同**：每日验证作为 PR 门禁的补充，覆盖更长时间的测试

---

## 二、镜像体系设计

### 2.1 为什么需要镜像体系

**参考 PyTorch 社区实践**：PyTorch 维护官方 Docker 镜像（`pytorch/pytorch`），原因：

1. **环境一致性**：开发、测试、生产使用相同镜像，避免"在我机器上能跑"问题
2. **依赖管理**：预装所有依赖（CUDA、cuDNN、测试工具），加速 CI 运行
3. **版本控制**：镜像 tag 对应特定版本组合，可复现历史测试
4. **缓存加速**：避免每次 CI 都重新安装依赖

**torch-npu 的特殊需求**：
- CANN 安装包较大（~2GB），每次下载耗时 10+ 分钟
- PyTorch nightly 每日更新，需要缓存机制
- torch-npu 构建耗时 20-30 分钟，需要 ccache

### 2.2 镜像分层架构

参考 PyTorch 社区的分层设计，torch-npu 采用 **3 层架构**（简化版）：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         镜像分层架构（3 层）                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Layer 1: Base Image (基础层)                                                │
│  ─────────────────────────────────                                         │
│  内容: Ubuntu 22.04 + Python 3.11 + 编译工具 + ccache                        │
│  更新频率: 每 6 个月或安全更新                                                 │
│  镜像名: base-npu:ubuntu22.04-py311                                         │
│  参考: pytorch/pytorch-base                                                 │
│                                                                             │
│  Layer 2: CANN Image (驱动层)                                                │
│  ─────────────────────────────────                                         │
│  内容: Base + CANN Toolkit + HCCL + ATB                                     │
│  更新频率: CANN 版本发布时（~3 个月）                                          │
│  镜像名: cann-npu:8.0.RC1                                                   │
│  参考: nvidia/cuda:12.1-devel                                               │
│                                                                             │
│  Layer 3: Test Image (测试层)                                                │
│  ─────────────────────────────────                                         │
│  内容: CANN + PyTorch nightly + torch-npu + 测试依赖                          │
│  更新频率: 每日构建（跟随 PyTorch nightly）                                    │
│  镜像名: test-npu:py2.7.0.dev20250330-cann8.0                               │
│  参考: pytorch/pytorch:nightly                                              │
│                                                                             │
│  继承关系: Base → CANN → Test                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**为什么是 3 层而不是 4 层**：
- PyTorch 社区有独立的 Runner 镜像（包含 GitHub Actions Runner）
- torch-npu 使用自托管 Runner，不需要将 Runner 打包到镜像中
- 简化架构，降低维护成本

### 2.3 镜像构建策略

#### 2.3.1 构建触发机制

| 触发源 | 触发条件 | 构建目标 | 说明 |
|--------|---------|---------|------|
| **手动触发** | workflow_dispatch | 所有层级 | 用于测试和紧急修复 |
| **CANN 发布** | 新版本发布 | CANN Image → Test Image | 华为发布新版 CANN |
| **PyTorch nightly** | 每日更新 | Test Image | 每日 UTC 06:00 检查 |
| **torch-npu 更新** | master 分支提交 | Test Image | torch-npu 有新提交 |

#### 2.3.2 版本命名规范

```
镜像命名格式: [registry]/[namespace]/[name]:[tag]

Tag 规则:
├── Base Image: ubuntu22.04-py311-[date]
│   例: base-npu:ubuntu22.04-py311-20250301
│
├── CANN Image: [cann-version]
│   例: cann-npu:8.0.RC1
│   例: cann-npu:8.1.beta
│
└── Test Image: py[pytorch-ver]-cann[cann-ver]-[date]
    例: test-npu:py2.7.0.dev20250330-cann8.0-20250330
    例: test-npu:nightly  (latest nightly)
```

#### 2.3.3 镜像存储策略

**参考 PyTorch 社区**：PyTorch 使用 Docker Hub 作为主要镜像仓库。

**torch-npu 推荐**：华为云 SWR（Software Repository for Container）

原因：
- 国内访问速度快（NPU Runner 通常在国内）
- 与华为云 NPU 生态集成
- 支持私有镜像（如需要）

```yaml
镜像仓库配置:
  主仓库: swr.cn-east-3.myhuaweicloud.com/ascend-ci/
  备份: ghcr.io/computing-infra/ascend-ci/  (可选，用于公开分发)

保留策略:
  - Base Image: 保留所有版本
  - CANN Image: 保留最近 3 个版本
  - Test Image: 保留最近 30 天 + 所有 stable 标签
```

### 2.4 关键 Dockerfile 设计

#### 2.4.1 Base Image

```dockerfile
# dockerfiles/base-npu/Dockerfile
FROM ubuntu:22.04

# 安装基础工具和 Python 3.11
RUN apt-get update && apt-get install -y \
    git curl wget build-essential cmake ninja-build \
    python3.11 python3.11-dev python3.11-venv \
    ccache && \
    ln -sf /usr/bin/python3.11 /usr/bin/python3 && \
    curl -sS https://bootstrap.pypa.io/get-pip.py | python3.11

# 配置 ccache
ENV CCACHE_DIR=/cache/ccache CCACHE_MAXSIZE=5G

WORKDIR /workspace
```

#### 2.4.2 CANN Image

```dockerfile
# dockerfiles/cann-npu/Dockerfile
ARG BASE_IMAGE=base-npu:ubuntu22.04-py311
FROM ${BASE_IMAGE}

ARG CANN_VERSION=8.0.RC1

# 安装 CANN Toolkit（关键步骤）
RUN wget -q https://ascend-repo.obs.cn-east-2.myhuaweicloud.com/CANN/${CANN_VERSION}/Ascend-cann-toolkit_${CANN_VERSION}_linux-x86_64.run && \
    chmod +x Ascend-cann-toolkit*.run && \
    ./Ascend-cann-toolkit*.run --install --install-path=/usr/local/Ascend && \
    rm -f Ascend-cann-toolkit*.run

# 设置 CANN 环境变量
ENV ASCEND_HOME=/usr/local/Ascend \
    PATH="${ASCEND_HOME}/bin:${PATH}" \
    LD_LIBRARY_PATH="${ASCEND_HOME}/lib64:${LD_LIBRARY_PATH}"
```

#### 2.4.3 Test Image

```dockerfile
# dockerfiles/test-npu/Dockerfile
ARG CANN_IMAGE=cann-npu:8.0.RC1
FROM ${CANN_IMAGE}

ARG PYTORCH_VERSION=2.7.0.dev20250330

# 安装 PyTorch nightly
RUN pip install --no-cache-dir --pre torch==${PYTORCH_VERSION} \
    --index-url https://download.pytorch.org/whl/nightly/cpu

# 安装测试依赖
RUN pip install --no-cache-dir pytest pytest-xdist numpy pyyaml

# 预留 torch-npu 安装（CI 时动态安装）
# 原因：torch-npu 每次 PR 都可能变化，不固化到镜像中
```

### 2.5 镜像构建 Workflow（关键逻辑）

```yaml
# .github/workflows/build-test-image.yml
name: build-test-image

on:
  schedule:
    - cron: '0 6 * * *'  # 每日 UTC 06:00
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Get PyTorch nightly version
        id: pytorch
        run: |
          VERSION=$(pip index versions torch --pre 2>/dev/null | grep -oP '2\.\d+\.\d+\.dev\d+' | head -1)
          echo "version=${VERSION}" >> $GITHUB_OUTPUT

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: ./dockerfiles/test-npu
          push: true
          tags: |
            swr.cn-east-3.myhuaweicloud.com/ascend-ci/test-npu:py${{ steps.pytorch.outputs.version }}-cann8.0-$(date +%Y%m%d)
            swr.cn-east-3.myhuaweicloud.com/ascend-ci/test-npu:nightly
          build-args: |
            PYTORCH_VERSION=${{ steps.pytorch.outputs.version }}
          cache-from: type=registry,ref=swr.cn-east-3.myhuaweicloud.com/ascend-ci/test-npu:nightly
```

**关键设计点**：
1. 使用 Docker BuildKit 缓存加速构建
2. 同时推送带日期的 tag 和 `nightly` tag
3. PyTorch 版本动态获取，确保使用最新 nightly

---

## 三、门禁测试策略

### 3.1 测试分层设计（参考 PyTorch 社区）

#### 3.1.1 PyTorch 主仓的测试策略

分析 PyTorch 主仓 `.github/workflows/` 中的 CUDA/ROCm 测试：

**CUDA 测试**（`_linux-test.yml`）：
```yaml
# 1. Smoke Test (linux-focal-cuda12.4-py3.11-gcc9-sm86-build)
- 运行时间: ~5 分钟
- 测试内容: test_cuda_primary_ctx, test_cuda_nvml_based_avail
- 目的: 快速验证 CUDA 基本功能

# 2. Default Test (linux-focal-cuda12.4-py3.11-gcc9-sm86-test)
- 运行时间: ~60 分钟
- 分片: 6 个 shard
- 测试内容: test_torch, test_ops, test_nn, test_autograd 等
- 环境变量: PYTORCH_TESTING_DEVICE_ONLY_FOR=cuda

# 3. Distributed Test (linux-focal-cuda12.4-py3.11-gcc9-sm86-distributed)
- 运行时间: ~90 分钟
- 测试内容: distributed/test_c10d, distributed/test_distributed_spawn
```

**ROCm 测试**（`_rocm-test.yml`）：
```yaml
# 类似 CUDA，但使用 ROCm 设备
- Smoke: test_transformers (快速验证)
- Default: 6 shard 并行
- 环境变量: PYTORCH_TESTING_DEVICE_ONLY_FOR=cuda (ROCm 复用 CUDA 测试)
```

**关键发现**：
1. **Smoke 必须通过**才能继续后续测试（fail-fast）
2. **Default 测试使用分片**并行执行，加速反馈
3. **设备无关测试**通过环境变量控制（`PYTORCH_TESTING_DEVICE_ONLY_FOR`）
4. **分布式测试**独立运行，不阻塞基础测试

#### 3.1.2 torch-npu 测试分层策略

参考 PyTorch 社区，设计 **3 层测试**（简化为必要任务）：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          torch-npu 测试分层                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Layer 1: Smoke Test (5 分钟)                                               │
│  ─────────────────────────────────                                         │
│  目的: 快速验证基本集成，发现严重问题                                          │
│  触发: 每个 PR 必须运行                                                       │
│  阻塞: 失败则阻塞 PR，不运行后续测试                                           │
│  内容:                                                                       │
│    - torch_npu 导入验证                                                     │
│    - NPU 设备检测 (torch.npu.is_available())                                │
│    - 基本张量操作 (to('npu'), add, matmul)                                   │
│    - PrivateUse1 后端注册验证                                                │
│  参考: PyTorch 的 test_cuda_primary_ctx                                     │
│                                                                             │
│  Layer 2: Device-Agnostic Test (30-45 分钟)                                 │
│  ─────────────────────────────────                                         │
│  目的: 验证 PyTorch 通用功能在 NPU 上的正确性                                  │
│  触发: Smoke 通过后                                                          │
│  阻塞: 建议通过才能合入 PR（可配置）                                           │
│  内容:                                                                       │
│    - test_torch.py (核心张量操作)                                           │
│    - test_ops.py (算子测试，OpInfo 框架)                                     │
│    - test_nn.py (神经网络模块)                                              │
│  分片: 6 个 shard 并行执行                                                   │
│  环境变量: PYTORCH_TESTING_DEVICE_ONLY_FOR=npu                              │
│  参考: PyTorch 的 linux-focal-cuda12.4-py3.11-gcc9-sm86-test               │
│                                                                             │
│  Layer 3: NPU-Specific Test (30-60 分钟)                                    │
│  ─────────────────────────────────                                         │
│  目的: 验证 torch-npu 专用功能                                               │
│  触发: Device-Agnostic 通过后                                                │
│  阻塞: 必须通过                                                              │
│  内容:                                                                       │
│    - test/test_npu.py (NPU 核心功能)                                        │
│    - test/test_custom_op.py (自定义算子)                                    │
│    - test/test_aclnn.py (ACL NN 算子)                                       │
│  并行: pytest -n auto                                                       │
│                                                                             │
│  Layer 4: Extended Test (每日运行，不阻塞 PR)                                │
│  ─────────────────────────────────                                         │
│  目的: 深度验证复杂场景                                                       │
│  触发: 每日定时或 PR 合入后                                                   │
│  阻塞: 不阻塞 PR 合入                                                         │
│  内容:                                                                       │
│    - distributed/test_c10d* (分布式训练)                                    │
│    - test/test_hccl.py (HCCL 通信)                                          │
│    - inductor/test_torchinductor.py (Inductor 编译)                        │
│  参考: PyTorch 的 linux-focal-cuda12.4-py3.11-gcc9-sm86-distributed        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Smoke Test 设计

#### 3.2.1 测试内容

```python
# test/smoke_test.py
"""
torch-npu Smoke 测试
参考: PyTorch test_cuda_primary_ctx, test_cuda_nvml_based_avail
"""

def test_pytorch_version():
    """验证 PyTorch 版本"""
    assert torch.__version__.startswith("2.")

def test_torch_npu_import():
    """验证 torch_npu 导入"""
    import torch_npu

def test_npu_available():
    """验证 NPU 设备可用"""
    assert torch.npu.is_available()
    assert torch.npu.device_count() > 0

def test_basic_tensor_ops():
    """验证基本张量操作"""
    x = torch.randn(2, 3).to('npu')
    y = x + x
    z = torch.matmul(x, x.T)
    assert z.device.type == 'npu'

def test_privateuse1_backend():
    """验证 PrivateUse1 后端注册"""
    backend = torch.utils.backend_registration._get_privateuse1_backend_name()
    assert backend == 'npu'
```

#### 3.2.2 运行方式

```bash
# 在 CI 中运行
python test/smoke_test.py --verbose

# 预期输出
# ✓ PyTorch version: 2.7.0.dev20250330
# ✓ torch_npu imported
# ✓ NPU available: 1 device
# ✓ Basic tensor operations passed
# ✓ PrivateUse1 backend: npu
```

### 3.3 Device-Agnostic Test 设计

#### 3.3.1 测试选择（参考 PyTorch 主仓）

PyTorch 主仓使用 `instantiate_device_type_tests` 机制，让同一测试在不同设备上运行：

```python
# PyTorch: torch/testing/_internal/common_device_type.py
class TestTorchDeviceType(TestCase):
    @dtypes(torch.float32, torch.float64)
    def test_add(self, device, dtype):
        x = torch.randn(2, 3, device=device, dtype=dtype)
        y = x + x
        # 测试逻辑...

# 实例化为不同设备的测试
instantiate_device_type_tests(TestTorchDeviceType, globals(), only_for='cuda')
# 生成: test_add_cuda_float32, test_add_cuda_float64, ...
```

**torch-npu 复用这套机制**：

```python
# torch-npu 需要注册 NPU 设备类型
# 通过环境变量控制: PYTORCH_TESTING_DEVICE_ONLY_FOR=npu

# CI 中运行
export PYTORCH_TESTING_DEVICE_ONLY_FOR=npu
python test/run_test.py --include test_torch test_ops test_nn --shard 1 6
```

#### 3.3.2 测试分片策略

参考 PyTorch 主仓的分片配置：

| Shard | 测试文件 | 预估时间 |
|-------|---------|---------|
| 1 | test_torch (part 1) | ~8 分钟 |
| 2 | test_torch (part 2) | ~8 分钟 |
| 3 | test_ops (part 1) | ~10 分钟 |
| 4 | test_ops (part 2) | ~10 分钟 |
| 5 | test_nn | ~8 分钟 |
| 6 | test_autograd | ~6 分钟 |

**总时间**：~10 分钟（并行执行）

### 3.4 NPU-Specific Test 设计

torch-npu 仓库中的专用测试：

```
test/
├── test_npu.py              # NPU 核心功能
├── test_custom_op.py        # 自定义算子
├── test_aclnn.py            # ACL NN 算子
├── test_npu_format.py       # NPU 特殊格式 (NZ, FRACTAL_NZ)
└── test_npu_profiler.py     # NPU profiler
```

运行方式：
```bash
pytest test/ -v --ignore=test/distributed/ -n auto
```

---

## 四、PR 门禁 Workflow 设计

### 4.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PR 门禁 Workflow 架构                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  torch-npu PR 提交                                                           │
│         ↓                                                                   │
│  触发 pytorch-infra workflow (repository_dispatch)                          │
│         ↓                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Job 1: smoke-test (必须通过)                                          │   │
│  │   - 拉取 PyTorch nightly                                              │   │
│  │   - 构建 torch-npu                                                    │   │
│  │   - 运行 smoke_test.py                                                │   │
│  │   - 失败 → 阻塞，不运行后续测试                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         ↓ (成功)                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Job 2: device-agnostic-test (6 shard 并行)                           │   │
│  │   - Shard 1-6 并行运行                                                │   │
│  │   - 环境变量: PYTORCH_TESTING_DEVICE_ONLY_FOR=npu                     │   │
│  │   - 失败 → 警告，但继续运行 NPU-Specific                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         ↓                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Job 3: npu-specific-test                                              │   │
│  │   - pytest -n auto                                                    │   │
│  │   - 失败 → 阻塞 PR                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         ↓                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Job 4: aggregate-results                                              │   │
│  │   - 汇总所有测试结果                                                   │   │
│  │   - 发送 PR Comment                                                   │   │
│  │   - 设置 GitHub Check 状态                                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 关键 Workflow 配置

#### 4.2.1 PR Gate Workflow（pytorch-infra 仓库）

```yaml
# .github/workflows/pr-gate.yml
name: torch-npu-pr-gate

on:
  repository_dispatch:
    types: [torch-npu-pr]
  workflow_dispatch:
    inputs:
      torch_npu_sha:
        required: true
      pr_number:
        required: false

jobs:
  smoke-test:
    runs-on: [self-hosted, npu-910b]
    timeout-minutes: 20
    container:
      image: swr.cn-east-3.myhuaweicloud.com/ascend-ci/test-npu:nightly
      options: --device=/dev/davinci0 --device=/dev/davinci_manager

    steps:
      - name: Checkout torch-npu
        uses: actions/checkout@v4
        with:
          repository: Ascend/pytorch
          ref: ${{ inputs.torch_npu_sha }}
          submodules: recursive

      - name: Build torch-npu
        run: |
          python setup.py bdist_wheel
          pip install dist/torch_npu*.whl

      - name: Run Smoke Test
        run: python test/smoke_test.py --verbose

  device-agnostic-test:
    needs: smoke-test
    strategy:
      matrix:
        shard: [1, 2, 3, 4, 5, 6]
      fail-fast: false
    runs-on: [self-hosted, npu-910b]
    timeout-minutes: 45
    env:
      PYTORCH_TESTING_DEVICE_ONLY_FOR: npu
      SHARD_NUMBER: ${{ matrix.shard }}
      NUM_TEST_SHARDS: 6

    steps:
      # ... (类似 smoke-test)
      - name: Run Device-Agnostic Tests
        run: |
          git clone --depth=1 https://github.com/pytorch/pytorch.git /tmp/pytorch
          cd /tmp/pytorch
          python test/run_test.py --include test_torch test_ops test_nn --shard ${{ matrix.shard }} 6

  npu-specific-test:
    needs: device-agnostic-test
    runs-on: [self-hosted, npu-910b]
    timeout-minutes: 60

    steps:
      # ... (类似 smoke-test)
      - name: Run NPU-Specific Tests
        run: pytest test/ -v --ignore=test/distributed/ -n auto
```

#### 4.2.2 触发 Workflow（torch-npu 仓库）

```yaml
# Ascend/pytorch/.github/workflows/trigger-ci.yml
name: trigger-pytorch-infra

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.PYTORCH_INFRA_TOKEN }}
          script: |
            github.rest.repos.createDispatchEvent({
              owner: 'computing-infra',
              repo: 'pytorch-infra',
              event_type: 'torch-npu-pr',
              client_payload: {
                torch_npu_sha: '${{ github.event.pull_request.head.sha }}',
                pr_number: '${{ github.event.pull_request.number }}'
              }
            });
```

### 4.3 结果反馈机制

#### 4.3.1 PR Comment

```markdown
## 🔥 torch-npu PR Gate Results

**Status**: ✅ All tests passed

| Test Layer | Status | Duration |
|------------|--------|----------|
| Smoke Test | ✅ Passed | 3m 24s |
| Device-Agnostic (6 shards) | ✅ Passed | 8m 42s |
| NPU-Specific | ✅ Passed | 24m 18s |

**Environment**:
- PyTorch nightly: `2.7.0.dev20250330`
- torch-npu commit: `abc123def`
- CANN version: `8.0.RC1`

[View full logs](https://github.com/computing-infra/pytorch-infra/actions/runs/12345)
```

#### 4.3.2 GitHub Check

```yaml
# 设置 commit status
- uses: actions/github-script@v7
  script: |
    github.rest.repos.createCommitStatus({
      owner: 'Ascend',
      repo: 'pytorch',
      sha: '${{ inputs.torch_npu_sha }}',
      state: 'success',  # or 'failure'
      context: 'pytorch-infra/pr-gate',
      description: 'All tests passed',
      target_url: '${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}'
    });
```

---

## 五、每日集成验证

### 5.1 与 PR 门禁的区别

| 维度 | PR 门禁 | 每日集成验证 |
|------|---------|-------------|
| **触发时机** | torch-npu PR 提交 | 每日定时（UTC 21:00） |
| **测试范围** | Layer 1-3 | Layer 1-4（包含分布式） |
| **运行时间** | ~60 分钟 | ~120 分钟 |
| **失败处理** | 阻塞 PR 合入 | 创建 issue 追踪 |
| **目标** | 保证 PR 质量 | 发现兼容性问题 |

### 5.2 每日集成 Workflow

```yaml
# .github/workflows/nightly-integration.yml
name: nightly-integration

on:
  schedule:
    - cron: '0 21 * * *'  # UTC 21:00 = 北京时间 05:00

jobs:
  full-test:
    uses: ./.github/workflows/pr-gate.yml
    with:
      torch_npu_sha: ${{ needs.get-latest.outputs.sha }}
      pr_number: ''  # 不关联 PR

  distributed-test:
    runs-on: [self-hosted, npu-910b-4]  # 4 卡 Runner
    steps:
      - name: Run Distributed Tests
        run: pytest test/distributed/ -v

  create-issue-on-failure:
    needs: [full-test, distributed-test]
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        script: |
          github.rest.issues.create({
            owner: 'Ascend',
            repo: 'pytorch',
            title: '[Compatibility] Nightly integration failed',
            labels: ['compatibility', 'ci-failure']
          });
```

---

## 六、Runner 环境配置

### 6.1 Runner 需求

参考 PyTorch 社区的 Runner 配置（`linux.g5.4xlarge.nvidia.gpu`），torch-npu 需要：

| Runner 类型 | 硬件配置 | 用途 | 数量 |
|------------|---------|------|------|
| `self-hosted, npu-910b` | 910B 单卡 + 32GB 内存 | PR 门禁 | 2-4 台（并发） |
| `self-hosted, npu-910b-4` | 910B 4 卡 + 64GB 内存 | 分布式测试 | 1 台 |

### 6.2 Runner 配置脚本

```bash
#!/bin/bash
# 配置 NPU Runner

# 1. 安装 CANN（参考华为文档）
wget https://ascend-repo.obs.cn-east-2.myhuaweicloud.com/CANN/8.0.RC1/Ascend-cann-toolkit_8.0.RC1_linux-x86_64.run
chmod +x Ascend-cann-toolkit*.run
./Ascend-cann-toolkit*.run --install

# 2. 配置 NPU 设备权限
cat > /etc/udev/rules.d/90-npu.rules << 'EOF'
KERNEL=="davinci[0-9]*", MODE="0666"
KERNEL=="davinci_manager", MODE="0666"
EOF
udevadm control --reload-rules

# 3. 安装 Docker
apt-get update && apt-get install -y docker.io
usermod -aG docker $USER

# 4. 安装 GitHub Actions Runner
mkdir -p /actions-runner && cd /actions-runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf actions-runner-linux-x64-*.tar.gz

# 5. 配置 Runner
./config.sh \
  --url https://github.com/computing-infra/pytorch-infra \
  --token <RUNNER_TOKEN> \
  --labels self-hosted,npu-910b \
  --name npu-runner-01

# 6. 启动 Runner（作为服务）
sudo ./svc.sh install
sudo ./svc.sh start
```

---

## 七、实施路线图

### 7.1 Phase 1: 镜像基础设施（Week 1-2）

**目标**：构建稳定的测试镜像

**任务**：
- [ ] 创建 Base Image Dockerfile
- [ ] 创建 CANN Image Dockerfile
- [ ] 创建 Test Image Dockerfile
- [ ] 配置华为云 SWR 镜像仓库
- [ ] 实现镜像构建 Workflow
- [ ] 验证镜像可用性

**验收标准**：
- ✅ 镜像构建成功并推送到 SWR
- ✅ 镜像中 PyTorch nightly 可用
- ✅ 镜像中 CANN 环境正确配置

### 7.2 Phase 2: PR 门禁（Week 3-4）

**目标**：实现 torch-npu PR 自动触发测试

**任务**：
- [ ] 配置 NPU Runner 环境（至少 2 台）
- [ ] 实现 pr-gate.yml（Smoke + Device-Agnostic + NPU-Specific）
- [ ] 实现 torch-npu 触发机制（repository_dispatch）
- [ ] 实现结果反馈（PR Comment + GitHub Check）
- [ ] 编写 smoke_test.py

**验收标准**：
- ✅ torch-npu PR 自动触发测试
- ✅ 60 分钟内完成全部测试
- ✅ 结果正确反馈到 PR

### 7.3 Phase 3: 每日集成（Week 5-6）

**目标**：定期验证兼容性

**任务**：
- [ ] 实现 nightly-integration.yml
- [ ] 实现 Issue 自动追踪机制
- [ ] 配置分布式测试 Runner（4 卡）
- [ ] 实现兼容性报告生成

**验收标准**：
- ✅ 每日自动运行集成测试
- ✅ 失败时自动创建 issue
- ✅ 修复后自动关闭 issue

### 7.4 Phase 4: 扩展功能（Week 7+）

**目标**：完善测试覆盖

**任务**：
- [ ] Inductor 测试集成
- [ ] 多架构支持（910A, 310P）
- [ ] 性能基准测试
- [ ] 测试结果可视化

---

## 八、总结

### 8.1 核心设计要点

1. **参考 PyTorch 社区**：镜像分层、测试分层、分片并行
2. **3 层镜像架构**：Base → CANN → Test，简化维护
3. **3 层测试策略**：Smoke（必须）→ Device-Agnostic（建议）→ NPU-Specific（必须）
4. **跨仓库触发**：repository_dispatch 实现 torch-npu → pytorch-infra
5. **快速反馈**：Smoke 5 分钟，全部测试 60 分钟

### 8.2 必要任务清单

**镜像基础设施**：
- ✅ Base Image（Ubuntu + Python + 编译工具）
- ✅ CANN Image（CANN Toolkit + HCCL）
- ✅ Test Image（PyTorch nightly + 测试依赖）
- ✅ 每日构建 Workflow

**PR 门禁**：
- ✅ Smoke Test（5 分钟）
- ✅ Device-Agnostic Test（6 shard，30-45 分钟）
- ✅ NPU-Specific Test（30-60 分钟）
- ✅ 结果反馈（PR Comment + GitHub Check）

**每日集成**：
- ✅ 完整测试套件（包含分布式）
- ✅ Issue 自动追踪

**Runner 环境**：
- ✅ NPU 单卡 Runner（2-4 台）
- ✅ NPU 多卡 Runner（1 台，用于分布式）

### 8.3 参考资料

- PyTorch CI 配置：https://github.com/pytorch/pytorch/tree/main/.github/workflows
- PyTorch Docker 镜像：https://github.com/pytorch/pytorch/tree/main/docker
- CANN 文档：https://www.hiascend.com/document
- torch-npu 仓库：https://gitcode.com/Ascend/pytorch

