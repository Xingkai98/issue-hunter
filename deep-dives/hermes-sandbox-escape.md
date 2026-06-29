# 深度分析：Hermes Agent Docker 沙箱逃逸漏洞

> 适合面试讨论的安全案例分析
> Issue: https://github.com/NousResearch/hermes-agent/issues/54354

---

## 一、什么是这个漏洞

**一句话：** Hermes Agent 在 Docker 沙箱模式下，首次工具调用会在容器镜像拉取完成之前直接在宿主机上执行。

**后果：** 用户的文件操作、终端命令、代码执行在宿主机上运行而非隔离的 Docker 容器内——这本质上是一次**沙箱逃逸**。

```
用户发送 "显示当前文件路径"
  → Hermes 开始拉取 Docker 镜像（异步，最长 120s）
  → 工具调用到达，容器还没就绪
  → 回退到宿主机执行 → 返回 ~/username/...（宿主机路径！）
  
用户再次发送相同命令
  → 容器已就绪 → 返回 /root/...（容器内路径）✓
```

---

## 二、源码级根因分析

### 2.1 核心文件

| 文件 | 作用 |
|------|------|
| `tools/environments/docker.py` | Docker 沙箱环境实现 |
| `tools/terminal_tool.py` | 终端工具调度 + 环境生命周期管理 |
| `agent/tool_executor.py` | 工具调用执行入口 |

### 2.2 环境创建——不做容器启动

`tools/terminal_tool.py` 中的 `_create_environment()` 函数：

```python
def _create_environment(env_type, image, cwd, timeout, ...):
    if env_type == "local":
        return _LocalEnvironment(cwd=cwd, timeout=timeout)
    elif env_type == "docker":
        # 创建 DockerEnvironment 实例
        # 注意：这里只做参数校验，不启动容器！
        return DockerEnvironment(image=image, ...)
```

`DockerEnvironment.__init__()` 做了什么：
- 验证 Docker 可执行文件存在（`_ensure_docker_available()`）
- 构建资源限制参数（`--cpus`, `--memory`, `--pids-limit`）
- 设置卷挂载和安全选项
- **不启动容器**——`self._container_id = None`

### 2.3 容器启动——懒加载，120 秒超时

容器在第一次需要时才启动（`docker.py` 第 940-968 行）：

```python
# 关键代码段（简化）
if not reused:
    run_cmd = [
        self._docker_exe, "run", "-d",
        "--name", container_name,
        *label_args,
        "-w", cwd,
        *all_run_args,
        image,
        "sleep", "infinity",
    ]
    result = subprocess.run(
        run_cmd,
        capture_output=True, text=True,
        timeout=120,  # ← image pull may take a while
        check=True,
    )
    self._container_id = result.stdout.strip()
```

**设计决策：** `timeout=120` 意味着镜像拉取最多等待 2 分钟。在这 120 秒内，`self._container_id` 仍然是 `None`。

### 2.4 命令执行——断言容器已存在

`_run_bash()` 方法（第 1011 行）：

```python
def _run_bash(self, cmd_string, *, login=False, timeout=120):
    assert self._container_id, "Container not started"  # ← 硬断言
    cmd = [self._docker_exe, "exec"]
    cmd.extend([self._container_id])
    ...
```

如果容器未启动，这里直接 `AssertionError` 崩溃。

### 2.5 真正的问题——环境就绪前工具已开始执行

问题出在**工具执行调度层**和**环境懒加载**之间的竞态：

1. Agent 收到用户消息，决定调用工具（如 `read_file`）
2. 工具执行器调用 `get_active_env(task_id)` 获取当前环境
3. 如果是首次调用，环境要么不存在（返回 None），要么正在初始化中
4. 环境不存在时，系统**回退到本地环境**执行工具
5. 与此同时，Docker 容器的 `docker run` 正在后台拉取镜像

**关键缺陷：** 没有"等待容器就绪"的同步机制。工具调用和环境就绪是两个独立的时间线，它们在首次调用时产生竞态。

### 2.6 相关修复尝试——PR #1276 做了什么（以及没做什么）

PR #1276（"clearer docker backend preflight errors"）：
- ✅ 添加了 Docker 可执行文件存在性检查
- ❌ **没有预拉取镜像**
- ❌ **没有阻塞工具调用直到容器就绪**
- ❌ **没有添加"容器未就绪"的明确错误提示**

所以 PR #1276 修复了"Docker 没装时给出更好的错误提示"，但没有解决"Docker 装了但镜像没拉完时的行为"。

---

## 三、为什么这是面试好素材

### 3.1 涉及的计算机科学基础概念

| 概念 | 在这个 bug 中的体现 |
|------|-------------------|
| **竞态条件** | 工具调用和容器就绪是两个异步时间线，首次调用时产生竞态 |
| **懒加载 vs 预加载** | 镜像拉取的懒加载策略节省了启动时间，但引入了冷启动窗口 |
| **故障开放 vs 故障关闭** | 当 Docker 不可用时，系统选择了"回退到宿主机"（故障开放）而非"拒绝执行"（故障关闭）——安全上的错误选择 |
| **纵深防御** | 单一安全边界（Docker 容器）被绕过时，没有第二道防线 |
| **幂等性** | 同一个命令第一次和第二次执行产生不同结果（宿主机 vs 容器内） |

### 3.2 面试中可以讨论的角度

**角度 1：如果让你修，你怎么设计？**

- 方案 A：添加就绪守卫——工具执行前检查 `container_id is not None`，未就绪则阻塞等待或返回明确错误
- 方案 B：预拉取——Gateway 启动时拉取镜像，消除冷启动窗口
- 方案 C：故障关闭——当 Docker 配置了但不可用时，拒绝执行而非回退到宿主机
- **最佳实践：** B + C 组合——启动时预热 + 运行时绝不回退到不安全路径

**角度 2：这个 bug 反映的系统设计原则**

- "Never fall back to a less secure path" — 安全系统不应有"降级到不安全"的选项
- "Fail closed, not fail open" — 安全组件故障时应拒绝访问，而非放行
- 懒加载的代价——性能优化（延迟启动）引入了安全窗口

**角度 3：如何从代码层面发现这类问题**

- 搜索 `fallback`、`default`、`local` 等关键词在安全敏感路径中的使用
- 检查所有 `get_active_env()` 的调用点，看返回值 None 时如何处理
- 审计容器创建和命令执行之间的时间窗口

---

## 四、修复方案设计

### 推荐方案：三管齐下

```
第一层（预防）：Gateway 启动时预热
  → hermes gateway start 时执行 docker pull <image>
  → 如果拉取失败，启动时就报错（而非等到首次工具调用）

第二层（保护）：工具执行入口加守卫
  → 在 _create_environment 和 get_active_env 之间插入同步点
  → while container_not_ready: wait_or_error()
  → 绝不回退到不太安全的路径

第三层（监控）：添加可观测性
  → 记录"容器就绪等待时间"指标
  → 当发生环境回退时发送告警
```

### 具体代码改动

**`tools/terminal_tool.py`** — 在 `get_active_env()` 返回 None 时：
```python
env = get_active_env(task_id)
if env is None:
    # 不应该回退到本地——配置了 Docker 就应该用 Docker
    raise EnvironmentNotReadyError(
        "Docker container is still starting. Please retry in a moment."
    )
```

**`tools/environments/docker.py`** — 添加就绪等待：
```python
def wait_until_ready(self, timeout=120):
    """阻塞直到容器就绪或超时"""
    deadline = time.time() + timeout
    while not self._container_id:
        if time.time() > deadline:
            raise TimeoutError("Docker container failed to start")
        time.sleep(0.5)
```

---

## 五、复盘总结

| 维度 | 评价 |
|------|------|
| **严重程度** | 中高——沙箱隔离被绕过，但仅影响冷启动首次调用 |
| **发现难度** | 中——需要理解 Docker 懒加载机制 + 工具调度时序 |
| **修复难度** | 低——添加守卫条件即可，不涉及架构变更 |
| **根因类型** | 设计缺陷——懒加载 + 故障开放 + 缺少同步机制 |
| **教训** | 安全边界不能有"临时回退"路径；性能优化的懒加载在安全路径上需要配套的同步机制 |
