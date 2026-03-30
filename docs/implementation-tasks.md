# Ascend NPU CI 体系实施任务拆解

## 文档说明

本文档基于《Ascend NPU CI 集成验证体系设计方案》，将完整方案拆解为可执行的任务清单。

**任务组织原则**：
- 按照依赖关系组织，前置任务完成后才能开始后续任务
- 每个任务包含：目标、前置依赖、具体步骤、验收标准、预估工时
- 任务优先级：P0（必须）、P1（重要）、P2（可选）

**实施周期**：预计 6-8 周完成核心功能（Phase 1-3）

---

## Phase 1: 镜像基础设施（Week 1-2）

### 任务 1.1: 配置镜像仓库

**优先级**: P0
**预估工时**: 0.5 天
**前置依赖**: 无

**目标**：
配置华为云 SWR 镜像仓库，用于存储和分发测试镜像。

**具体步骤**：
1. 注册华为云账号（如未有）
2. 开通 SWR 服务（区域：cn-east-3）
3. 创建组织：`ascend-ci`
4. 创建镜像仓库：
   - `ascend-ci/base-npu`（公开）
   - `ascend-ci/cann-npu`（公开）
   - `ascend-ci/test-npu`（公开）
5. 生成访问凭证（AK/SK）
6. 在 GitHub 仓库配置 Secrets：
   - `SWR_USERNAME`: `cn-east-3@AK<your-ak>`
   - `SWR_PASSWORD`: `<your-sk>`

**验收标准**：
- ✅ SWR 仓库创建成功
- ✅ 本地可以 docker login 到 SWR
- ✅ GitHub Secrets 配置正确

**参考文档**：
- 华为云 SWR 文档：https://support.huaweicloud.com/swr/index.html

---

### 任务 1.2: 创建 Base Image

**优先级**: P0
**预估工时**: 1 天
**前置依赖**: 任务 1.1

**目标**：
构建基础镜像，包含 Ubuntu 22.04 + Python 3.11 + 编译工具。

**具体步骤**：
1. 创建目录结构：
   ```
   dockerfiles/
   └── base-npu/
       └── Dockerfile
   ```

2. 编写 `Dockerfile`（参考设计方案第二章 2.4.1）：
   - 基于 `ubuntu:22.04`
   - 安装 Python 3.11（从 deadsnakes PPA）
   - 安装编译工具：git, curl, wget, build-essential, cmake, ninja-build
   - 安装 ccache
   - 配置环境变量

3. 本地构建测试：
   ```bash
   cd dockerfiles/base-npu
   docker build -t base-npu:test .
   docker run --rm base-npu:test python --version
   ```

4. 创建构建 workflow：`.github/workflows/build-base-image.yml`
   - 触发方式：workflow_dispatch（手动触发）
   - 构建并推送到 SWR
   - Tag 格式：`ubuntu22.04-py311-YYYYMMDD`

5. 手动触发 workflow，验证构建成功

**验收标准**：
- ✅ Dockerfile 构建成功
- ✅ 镜像中 Python 版本为 3.11.x
- ✅ 镜像推送到 SWR 成功
- ✅ 镜像大小 < 1GB

**输出产物**：
- `dockerfiles/base-npu/Dockerfile`
- `.github/workflows/build-base-image.yml`
- 镜像：`swr.cn-east-3.myhuaweicloud.com/ascend-ci/base-npu:ubuntu22.04-py311-<date>`

---

### 任务 1.3: 创建 CANN Image

**优先级**: P0
**预估工时**: 2 天
**前置依赖**: 任务 1.2

**目标**：
构建 CANN 镜像，包含 CANN Toolkit 和 HCCL。

**具体步骤**：
1. 创建目录：
   ```
   dockerfiles/
   └── cann-npu/
       ├── Dockerfile
       └── README.md
   ```

2. 编写 `Dockerfile`（参考设计方案第二章 2.4.2）：
   - 基于 Base Image
   - 下载 CANN Toolkit（版本：8.0.RC1）
   - 安装 CANN Toolkit 到 `/usr/local/Ascend`
   - 下载并安装 HCCL
   - 设置环境变量：`ASCEND_HOME`, `PATH`, `LD_LIBRARY_PATH`

3. 本地构建测试（需要 NPU 环境）：
   ```bash
   docker build -t cann-npu:test --build-arg BASE_IMAGE=base-npu:ubuntu22.04-py311-<date> .
   docker run --rm cann-npu:test bash -c "source /usr/local/Ascend/bin/setenv.bash && which ascend-toolkit"
   ```

4. 创建构建 workflow：`.github/workflows/build-cann-image.yml`
   - 触发方式：workflow_dispatch
   - 输入参数：`cann_version`, `base_image`
   - 构建并推送到 SWR
   - Tag 格式：`<cann-version>`

5. 手动触发 workflow，验证构建成功

**验收标准**：
- ✅ CANN Toolkit 安装成功
- ✅ 环境变量配置正确
- ✅ 镜像推送到 SWR 成功
- ✅ 镜像大小 < 5GB

**注意事项**：
- CANN 安装包较大（~2GB），构建时间较长（~20 分钟）
- 需要确认 CANN 下载链接的有效性

**输出产物**：
- `dockerfiles/cann-npu/Dockerfile`
- `.github/workflows/build-cann-image.yml`
- 镜像：`swr.cn-east-3.myhuaweicloud.com/ascend-ci/cann-npu:8.0.RC1`

---

### 任务 1.4: 创建 Test Image

**优先级**: P0
**预估工时**: 1.5 天
**前置依赖**: 任务 1.3

**目标**：
构建测试镜像，包含 PyTorch nightly 和测试依赖。

**具体步骤**：
1. 创建目录：
   ```
   dockerfiles/
   └── test-npu/
       ├── Dockerfile
       └── README.md
   ```

2. 编写 `Dockerfile`（参考设计方案第二章 2.4.3）：
   - 基于 CANN Image
   - 安装 PyTorch nightly（CPU 版本）
   - 安装测试依赖：pytest, pytest-xdist, numpy, pyyaml, expecttest, hypothesis
   - 预留 torch-npu 安装位置（CI 时动态安装）

3. 本地构建测试：
   ```bash
   docker build -t test-npu:test --build-arg CANN_IMAGE=cann-npu:8.0.RC1 --build-arg PYTORCH_VERSION=2.7.0.dev20250330 .
   docker run --rm test-npu:test python -c "import torch; print(torch.__version__)"
   ```

4. 创建构建 workflow：`.github/workflows/build-test-image.yml`
   - 触发方式：schedule（每日 UTC 06:00）+ workflow_dispatch
   - 自动获取最新 PyTorch nightly 版本
   - 构建并推送到 SWR
   - Tag 格式：`py<pytorch-ver>-cann<cann-ver>-<date>` + `nightly`

5. 手动触发 workflow，验证构建成功

**验收标准**：
- ✅ PyTorch nightly 安装成功
- ✅ 测试依赖安装完整
- ✅ 镜像推送到 SWR 成功
- ✅ 每日自动构建运行正常

**输出产物**：
- `dockerfiles/test-npu/Dockerfile`
- `.github/workflows/build-test-image.yml`
- 镜像：`swr.cn-east-3.myhuaweicloud.com/ascend-ci/test-npu:nightly`

---

### 任务 1.5: 镜像验证和文档

**优先级**: P1
**预估工时**: 0.5 天
**前置依赖**: 任务 1.4

**目标**：
验证镜像可用性，编写镜像使用文档。

**具体步骤**：
1. 编写镜像验证脚本：`scripts/verify-images.sh`
   ```bash
   #!/bin/bash
   # 验证 Base Image
   docker run --rm swr.cn-east-3.myhuaweicloud.com/ascend-ci/base-npu:latest python --version

   # 验证 CANN Image
   docker run --rm swr.cn-east-3.myhuaweicloud.com/ascend-ci/cann-npu:8.0.RC1 bash -c "source /usr/local/Ascend/bin/setenv.bash && echo OK"

   # 验证 Test Image
   docker run --rm swr.cn-east-3.myhuaweicloud.com/ascend-ci/test-npu:nightly python -c "import torch; print(torch.__version__)"
   ```

2. 编写镜像使用文档：`docs/04-image-usage.md`
   - 镜像列表和用途
   - 如何拉取镜像
   - 如何在本地使用镜像
   - 如何更新镜像

3. 更新主 README.md，添加镜像相关说明

**验收标准**：
- ✅ 所有镜像验证通过
- ✅ 文档完整清晰

**输出产物**：
- `scripts/verify-images.sh`
- `docs/04-image-usage.md`

---

## Phase 2: PR 门禁（Week 3-4）

### 任务 2.1: 配置 NPU Runner 环境

**优先级**: P0
**预估工时**: 2 天
**前置依赖**: 任务 1.4

**目标**：
配置至少 2 台 NPU Runner，用于运行 PR 门禁测试。

**具体步骤**：
1. 准备硬件资源：
   - 2-4 台服务器，每台配置：
     - NPU: 910B 单卡
     - 内存: 32GB+
     - 存储: 200GB+
     - 网络: 可访问 GitHub 和华为云

2. 安装 CANN 驱动和 Toolkit（参考设计方案第六章）：
   ```bash
   # 下载并安装 CANN
   wget https://ascend-repo.obs.cn-east-2.myhuaweicloud.com/CANN/8.0.RC1/Ascend-cann-toolkit_8.0.RC1_linux-x86_64.run
   chmod +x Ascend-cann-toolkit*.run
   ./Ascend-cann-toolkit*.run --install
   ```

3. 配置 NPU 设备权限：
   ```bash
   cat > /etc/udev/rules.d/90-npu.rules << 'EOF'
   KERNEL=="davinci[0-9]*", MODE="0666"
   KERNEL=="davinci_manager", MODE="0666"
   EOF
   udevadm control --reload-rules
   ```

4. 安装 Docker：
   ```bash
   apt-get update && apt-get install -y docker.io
   usermod -aG docker $USER
   ```

5. 安装 GitHub Actions Runner：
   ```bash
   mkdir -p /actions-runner && cd /actions-runner
   curl -o actions-runner-linux-x64-2.311.0.tar.gz -L \
     https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
   tar xzf actions-runner-linux-x64-*.tar.gz
   ```

6. 配置 Runner：
   ```bash
   ./config.sh \
     --url https://github.com/computing-infra/pytorch-infra \
     --token <RUNNER_TOKEN> \
     --labels self-hosted,npu-910b \
     --name npu-runner-01
   ```

7. 启动 Runner 服务：
   ```bash
   sudo ./svc.sh install
   sudo ./svc.sh start
   ```

8. 验证 Runner 状态：
   - 在 GitHub 仓库 Settings → Actions → Runners 中查看 Runner 状态
   - 应显示为 "Idle" 状态

**验收标准**：
- ✅ 至少 2 台 Runner 配置成功
- ✅ Runner 在 GitHub 中显示为 "Idle"
- ✅ NPU 设备可用（`npu-smi info` 正常）
- ✅ Docker 可以访问 NPU 设备

**注意事项**：
- Runner Token 有效期为 1 小时，需要及时配置
- 确保 Runner 服务器可以访问 GitHub 和华为云 SWR

**输出产物**：
- 配置好的 NPU Runner（2-4 台）
- Runner 配置文档：`docs/05-runner-setup.md`

---

### 任务 2.2: 编写 Smoke Test

**优先级**: P0
**预估工时**: 1 天
**前置依赖**: 任务 2.1

**目标**：
编写 Smoke Test 脚本，快速验证 torch-npu 与 PyTorch nightly 的基本集成。

**具体步骤**：
1. 创建测试目录：
   ```
   test/
   └── smoke_test.py
   ```

2. 编写 `smoke_test.py`（参考设计方案第三章 3.2）：
   - 测试 PyTorch 版本
   - 测试 torch_npu 导入
   - 测试 NPU 设备可用性
   - 测试基本张量操作
   - 测试 PrivateUse1 后端注册

3. 本地测试（需要 NPU 环境）：
   ```bash
   # 在 NPU 机器上
   docker run --rm --device=/dev/davinci0 --device=/dev/davinci_manager \
     -v $(pwd)/test:/workspace/test \
     swr.cn-east-3.myhuaweicloud.com/ascend-ci/test-npu:nightly \
     python /workspace/test/smoke_test.py --verbose
   ```

4. 添加命令行参数支持：
   - `--verbose`: 详细输出
   - `--device`: 指定 NPU 设备编号

**验收标准**：
- ✅ 所有测试用例通过
- ✅ 运行时间 < 5 分钟
- ✅ 输出清晰易读
- ✅ 失败时返回非零退出码

**输出产物**：
- `test/smoke_test.py`

---

### 任务 2.3: 实现 PR Gate Workflow（Smoke Test）

**优先级**: P0
**预估工时**: 2 天
**前置依赖**: 任务 2.2

**目标**：
实现 PR 门禁的第一层测试（Smoke Test），验证基本流程。

**具体步骤**：
1. 创建 workflow 文件：`.github/workflows/pr-gate.yml`

2. 实现 Smoke Test job（参考设计方案第四章 4.2）：
   ```yaml
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
   ```

3. 配置触发方式：
   - `repository_dispatch`: 接收来自 torch-npu 的触发
   - `workflow_dispatch`: 手动触发（用于测试）

4. 添加输入参数：
   - `torch_npu_sha`: torch-npu commit SHA
   - `pr_number`: PR 编号（可选）

5. 本地测试 workflow：
   - 手动触发 workflow
   - 验证 Smoke Test 运行成功

**验收标准**：
- ✅ Workflow 可以手动触发
- ✅ Smoke Test 在 NPU Runner 上运行成功
- ✅ 失败时 workflow 状态为 failed
- ✅ 运行时间 < 20 分钟

**输出产物**：
- `.github/workflows/pr-gate.yml`（初版，仅包含 Smoke Test）

---

### 任务 2.4: 扩展 PR Gate Workflow（完整测试）

**优先级**: P0
**预估工时**: 3 天
**前置依赖**: 任务 2.3

**目标**：
扩展 PR 门禁，添加 Device-Agnostic Test 和 NPU-Specific Test。

**具体步骤**：
1. 添加 Device-Agnostic Test job：
   - 使用 matrix 策略实现 6 分片并行
   - 克隆 PyTorch 主仓库
   - 运行 `test/run_test.py --include test_torch test_ops test_nn --shard X 6`
   - 环境变量：`PYTORCH_TESTING_DEVICE_ONLY_FOR=npu`

2. 添加 NPU-Specific Test job：
   - 依赖 Device-Agnostic Test 完成
   - 运行 `pytest test/ -v --ignore=test/distributed/ -n auto`

3. 添加 Aggregate Results job：
   - 汇总所有测试结果
   - 生成测试报告
   - 上传测试日志为 artifact

4. 配置 job 依赖关系：
   ```yaml
   jobs:
     smoke-test: ...
     device-agnostic-test:
       needs: smoke-test
       if: success()
     npu-specific-test:
       needs: device-agnostic-test
       if: success() || failure()  # 即使 DA 失败也运行
     aggregate-results:
       needs: [smoke-test, device-agnostic-test, npu-specific-test]
       if: always()
   ```

5. 测试完整流程：
   - 手动触发 workflow
   - 验证所有 job 按顺序运行
   - 验证分片并行执行

**验收标准**：
- ✅ 所有 3 层测试运行成功
- ✅ Device-Agnostic Test 6 分片并行执行
- ✅ 总运行时间 < 60 分钟
- ✅ 测试日志完整上传

**输出产物**：
- `.github/workflows/pr-gate.yml`（完整版）

---

### 任务 2.5: 实现跨仓库触发和结果反馈

**优先级**: P0
**预估工时**: 2 天
**前置依赖**: 任务 2.4

**目标**：
实现 torch-npu PR 自动触发 pytorch-infra 测试，并将结果反馈到 PR。

**具体步骤**：
1. 在 torch-npu 仓库创建触发 workflow：
   - 文件：`Ascend/pytorch/.github/workflows/trigger-ci.yml`
   - 触发时机：`pull_request` (opened, synchronize, reopened)
   - 使用 `repository_dispatch` 触发 pytorch-infra

2. 配置 GitHub Token：
   - 在 pytorch-infra 创建 PAT（Personal Access Token）
   - 权限：`repo`, `workflow`
   - 在 torch-npu 仓库配置 Secret：`PYTORCH_INFRA_TOKEN`

3. 在 pytorch-infra 添加结果反馈逻辑：
   - PR Comment：测试摘要和结果
   - GitHub Check：commit status
   - 使用 `actions/github-script@v7`

4. 实现反馈模板：
   ```markdown
   ## 🔥 torch-npu PR Gate Results

   **Status**: ✅ All tests passed

   | Test Layer | Status | Duration |
   |------------|--------|----------|
   | Smoke Test | ✅ | 3m 24s |
   | Device-Agnostic | ✅ | 8m 42s |
   | NPU-Specific | ✅ | 24m 18s |
   ```

5. 端到端测试：
   - 在 torch-npu 创建测试 PR
   - 验证自动触发 pytorch-infra workflow
   - 验证结果正确反馈到 PR

**验收标准**：
- ✅ torch-npu PR 自动触发测试
- ✅ 测试结果反馈到 PR Comment
- ✅ GitHub Check 状态正确设置
- ✅ 端到端流程运行正常

**注意事项**：
- 需要 torch-npu 仓库的写权限来配置 workflow
- Token 权限需要仔细配置，避免安全问题

**输出产物**：
- `Ascend/pytorch/.github/workflows/trigger-ci.yml`
- `.github/workflows/pr-gate.yml`（添加反馈逻辑）

---

## Phase 3: 每日集成验证（Week 5-6）

### 任务 3.1: 实现每日集成 Workflow

**优先级**: P0
**预估工时**: 1.5 天
**前置依赖**: 任务 2.5

**目标**：
实现每日定时运行完整测试套件，验证 torch-npu master 与 PyTorch nightly 的兼容性。

**具体步骤**：
1. 创建 workflow：`.github/workflows/nightly-integration.yml`

2. 配置触发方式：
   ```yaml
   on:
     schedule:
       - cron: '0 21 * * *'  # UTC 21:00 = 北京时间 05:00
     workflow_dispatch:
   ```

3. 获取最新版本：
   - torch-npu master 最新 commit
   - PyTorch nightly 最新版本

4. 复用 PR Gate workflow：
   ```yaml
   jobs:
     full-test:
       uses: ./.github/workflows/pr-gate.yml
       with:
         torch_npu_sha: ${{ needs.get-latest.outputs.sha }}
         pr_number: ''
   ```

5. 添加扩展测试（Layer 4）：
   - 分布式测试（需要多卡 Runner）
   - Inductor 测试（可选）

6. 测试定时触发：
   - 手动触发验证流程
   - 等待定时触发验证

**验收标准**：
- ✅ 每日自动运行
- ✅ 包含完整测试套件
- ✅ 运行时间 < 120 分钟

**输出产物**：
- `.github/workflows/nightly-integration.yml`

---

### 任务 3.2: 实现 Issue 自动追踪

**优先级**: P1
**预估工时**: 2 天
**前置依赖**: 任务 3.1

**目标**：
测试失败时自动创建 issue，修复后自动关闭 issue。

**具体步骤**：
1. 创建 workflow：`.github/workflows/issue-tracker.yml`

2. 配置触发方式：
   ```yaml
   on:
     workflow_run:
       workflows: ["nightly-integration"]
       types: [completed]
   ```

3. 实现失败追踪逻辑：
   - 检查 workflow 运行结果
   - 如果失败，创建 issue 到 torch-npu 仓库
   - Issue 标签：`compatibility`, `ci-failure`
   - Issue 内容：失败信息、PyTorch 版本、torch-npu commit、日志链接

4. 实现成功追踪逻辑：
   - 如果成功，查找相关的 open issue
   - 自动关闭已修复的 issue
   - 添加关闭评论

5. 避免重复创建 issue：
   - 检查是否已存在相同的 issue
   - 使用 issue title 或 label 去重

6. 测试 issue 追踪：
   - 模拟失败场景，验证 issue 创建
   - 模拟成功场景，验证 issue 关闭

**验收标准**：
- ✅ 失败时自动创建 issue
- ✅ 成功时自动关闭 issue
- ✅ 不会重复创建 issue
- ✅ Issue 内容完整清晰

**输出产物**：
- `.github/workflows/issue-tracker.yml`

---

### 任务 3.3: 生成兼容性报告

**优先级**: P2
**预估工时**: 1 天
**前置依赖**: 任务 3.1

**目标**：
生成每日兼容性报告，记录测试历史。

**具体步骤**：
1. 在 nightly-integration workflow 中添加报告生成步骤

2. 报告内容：
   - 日期
   - PyTorch nightly 版本
   - torch-npu commit
   - 测试结果（通过/失败）
   - 测试时长
   - 失败原因（如有）

3. 报告格式：Markdown

4. 报告存储：
   - 上传为 workflow artifact
   - 可选：提交到仓库 `reports/` 目录

5. 实现报告汇总页面：
   - 列出最近 30 天的测试结果
   - 可视化兼容性趋势

**验收标准**：
- ✅ 每日生成报告
- ✅ 报告内容完整
- ✅ 报告易于查看

**输出产物**：
- 兼容性报告生成脚本
- 报告汇总页面（可选）

---

## Phase 4: 扩展功能（Week 7+）

### 任务 4.1: 配置多卡 Runner（分布式测试）

**优先级**: P1
**预估工时**: 1.5 天
**前置依赖**: 任务 3.1

**目标**：
配置多卡 NPU Runner，用于分布式测试。

**具体步骤**：
1. 准备硬件资源：
   - 1 台服务器，配置：
     - NPU: 910B 4 卡
     - 内存: 64GB+
     - 存储: 300GB+

2. 安装 CANN 和配置环境（参考任务 2.1）

3. 配置 Runner：
   ```bash
   ./config.sh \
     --url https://github.com/computing-infra/pytorch-infra \
     --token <RUNNER_TOKEN> \
     --labels self-hosted,npu-910b-4 \
     --name npu-runner-multi-01
   ```

4. 验证多卡可用：
   ```bash
   npu-smi info
   # 应显示 4 张卡
   ```

5. 测试分布式环境：
   ```bash
   docker run --rm \
     --device=/dev/davinci0 \
     --device=/dev/davinci1 \
     --device=/dev/davinci2 \
     --device=/dev/davinci3 \
     --device=/dev/davinci_manager \
     swr.cn-east-3.myhuaweicloud.com/ascend-ci/test-npu:nightly \
     python -c "import torch; import torch_npu; print(torch.npu.device_count())"
   ```

**验收标准**：
- ✅ Runner 配置成功
- ✅ 4 张 NPU 卡可用
- ✅ Docker 可以访问所有 NPU 设备

**输出产物**：
- 配置好的多卡 NPU Runner（1 台）

---

### 任务 4.2: 实现分布式测试

**优先级**: P1
**预估工时**: 2 天
**前置依赖**: 任务 4.1

**目标**：
在每日集成中添加分布式测试。

**具体步骤**：
1. 在 nightly-integration.yml 中添加 distributed-test job：
   ```yaml
   distributed-test:
     runs-on: [self-hosted, npu-910b-4]
     timeout-minutes: 90
     container:
       image: swr.cn-east-3.myhuaweicloud.com/ascend-ci/test-npu:nightly
       options: --device=/dev/davinci0 --device=/dev/davinci1 --device=/dev/davinci2 --device=/dev/davinci3 --device=/dev/davinci_manager
   ```

2. 运行分布式测试：
   ```bash
   pytest test/distributed/ -v
   ```

3. 配置 HCCL 环境变量：
   - `RANK`
   - `WORLD_SIZE`
   - `MASTER_ADDR`
   - `MASTER_PORT`

4. 测试分布式功能：
   - 手动触发 nightly-integration
   - 验证分布式测试运行成功

**验收标准**：
- ✅ 分布式测试运行成功
- ✅ 4 卡并行训练正常
- ✅ HCCL 通信正常

**输出产物**：
- `.github/workflows/nightly-integration.yml`（添加分布式测试）

---

### 任务 4.3: 多架构支持（可选）

**优先级**: P2
**预估工时**: 3 天
**前置依赖**: 任务 2.5

**目标**：
支持多种 NPU 架构（910A, 310P）的测试。

**具体步骤**：
1. 配置不同架构的 Runner：
   - `self-hosted, npu-910a`
   - `self-hosted, npu-310p`

2. 构建不同架构的 CANN 镜像：
   - `cann-npu:8.0.RC1-910A`
   - `cann-npu:8.0.RC1-310P`

3. 在 PR Gate 中添加架构矩阵：
   ```yaml
   strategy:
     matrix:
       arch: [910b, 910a, 310p]
   ```

4. 测试多架构流程

**验收标准**：
- ✅ 支持至少 2 种架构
- ✅ 不同架构测试并行运行

**输出产物**：
- 多架构 CANN 镜像
- 多架构 Runner

---

### 任务 4.4: Inductor 测试集成（可选）

**优先级**: P2
**预估工时**: 3 天
**前置依赖**: 任务 3.1

**目标**：
集成 PyTorch Inductor 编译器测试。

**具体步骤**：
1. 确认 torch-npu 支持 Inductor

2. 在 nightly-integration 中添加 inductor-test job：
   ```bash
   pytest inductor/test_torchinductor.py -v
   ```

3. 配置 Inductor 环境变量：
   - `TORCHINDUCTOR_COMPILE_THREADS`
   - `TORCHINDUCTOR_CACHE_DIR`

4. 测试 Inductor 功能

**验收标准**：
- ✅ Inductor 测试运行成功
- ✅ 编译缓存正常工作

**输出产物**：
- `.github/workflows/nightly-integration.yml`（添加 Inductor 测试）

---

## 任务总结

### 按优先级分类

**P0（必须完成）**：
- Phase 1: 任务 1.1-1.4（镜像基础设施）
- Phase 2: 任务 2.1-2.5（PR 门禁）
- Phase 3: 任务 3.1（每日集成）

**P1（重要）**：
- Phase 1: 任务 1.5（镜像验证）
- Phase 3: 任务 3.2（Issue 追踪）
- Phase 4: 任务 4.1-4.2（分布式测试）

**P2（可选）**：
- Phase 3: 任务 3.3（兼容性报告）
- Phase 4: 任务 4.3-4.4（多架构、Inductor）

### 关键里程碑

**Milestone 1（Week 2）**：镜像基础设施完成
- ✅ 3 层镜像构建成功
- ✅ 镜像每日自动构建

**Milestone 2（Week 4）**：PR 门禁上线
- ✅ torch-npu PR 自动触发测试
- ✅ 测试结果反馈到 PR
- ✅ 60 分钟内完成测试

**Milestone 3（Week 6）**：每日集成完善
- ✅ 每日自动运行完整测试
- ✅ Issue 自动追踪
- ✅ 兼容性报告生成

**Milestone 4（Week 8+）**：扩展功能
- ✅ 分布式测试
- ✅ 多架构支持（可选）
- ✅ Inductor 测试（可选）

### 资源需求汇总

**硬件资源**：
- NPU 单卡 Runner: 2-4 台（910B）
- NPU 多卡 Runner: 1 台（910B 4 卡）
- 可选：其他架构 Runner（910A, 310P）

**云服务**：
- 华为云 SWR 镜像仓库
- 存储空间: ~100GB（镜像）

**人力投入**：
- 核心功能（Phase 1-3）: 1-2 人 × 6 周
- 扩展功能（Phase 4）: 1 人 × 2-4 周

### 风险和应对

**风险 1：NPU Runner 资源不足**
- 影响：PR 门禁排队，反馈延迟
- 应对：优先配置 2 台 Runner，后续根据负载扩容

**风险 2：CANN 版本兼容性问题**
- 影响：镜像构建失败或测试失败
- 应对：提前验证 CANN 版本，保持与 torch-npu 同步

**风险 3：PyTorch nightly API 变更**
- 影响：测试失败，需要适配
- 应对：Issue 自动追踪，及时通知 torch-npu 团队

**风险 4：跨仓库权限配置**
- 影响：无法触发 workflow 或反馈结果
- 应对：提前申请必要权限，测试端到端流程

### 后续优化方向

1. **性能优化**：
   - 使用 ccache 加速 torch-npu 构建
   - 优化测试分片策略
   - 并行运行更多测试

2. **测试覆盖**：
   - 增加更多 PyTorch 核心测试
   - 添加性能基准测试
   - 添加内存泄漏检测

3. **用户体验**：
   - 优化 PR Comment 格式
   - 添加测试结果可视化
   - 提供测试失败的快速诊断

4. **基础设施**：
   - 镜像构建缓存优化
   - Runner 自动扩缩容
   - 测试环境隔离增强

---

## 附录

### A. 相关文档

- 设计方案：`docs/02-ascend-npu-ci-image-gate-design.md`
- 镜像使用：`docs/04-image-usage.md`（待创建）
- Runner 配置：`docs/05-runner-setup.md`（待创建）

### B. 参考资料

- PyTorch CI 配置：https://github.com/pytorch/pytorch/tree/main/.github/workflows
- PyTorch Docker 镜像：https://github.com/pytorch/pytorch/tree/main/docker
- CANN 文档：https://www.hiascend.com/document
- torch-npu 仓库：https://gitcode.com/Ascend/pytorch
- GitHub Actions 文档：https://docs.github.com/en/actions

### C. 联系方式

如有问题，请联系：
- CI 基础设施：[团队联系方式]
- torch-npu 维护：[团队联系方式]

---

**文档版本**: v1.0
**最后更新**: 2026-03-30
**维护者**: CI Team
