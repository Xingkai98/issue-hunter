# 深度分析：Hermes Agent Docker 沙箱逃逸漏洞

> Issue: https://github.com/NousResearch/hermes-agent/issues/54354
> 状态：无人认领，无 PR，0 条评论

---

## 一、漏洞描述

**一句话：** Hermes Agent 配置 Docker 后端后，首次工具调用在容器镜像拉取完成之前直接在宿主机上执行。

**严重程度：** 中高——沙箱隔离在冷启动窗口内被绕过，属于实质上的沙箱逃逸。

---

## 二、源码级根因分析

### 2.1 受影响文件

| 文件 | 作用 |
|------|------|
| `tools/environments/docker.py` | Docker 沙箱环境实现 |
| `tools/terminal_tool.py` | 终端工具调度 + 环境生命周期 |

### 2.2 环境配置解析——默认值是 local

`tools/terminal_tool.py` 第 1244 行：

```python
env_type = os.getenv("TERMINAL_ENV", "local")
```

如果 `TERMINAL_ENV` 环境变量未正确传播，`env_type` 回退为 `"local"`——工具直接在宿主机执行。

### 2.3 Docker 容器创建——懒加载，但同步阻塞

`DockerEnvironment.__init__()` 只做参数校验，**不启动容器**（docker.py 第 606 行）：
```python
self._container_id: Optional[str] = None
```

容器在 `run()` 方法中首次使用时才启动（docker.py 第 925-968 行）：
```python
result = subprocess.run(
    run_cmd,
    capture_output=True, text=True,
    timeout=120,  # image pull may take a while
    check=True,
)
```

这是**同步阻塞**的——`subprocess.run(timeout=120)` 等待 `docker run -d` 完成（包括镜像拉取）。所以理论上首次工具调用**应该**等待容器就绪。

### 2.4 真正的根因——配置传播竞态

问题不在 Docker 代码本身，而在**配置传播层**：

1. 用户在 `config.yaml` 中设置 `terminal.backend: docker`
2. Gateway 启动时需将配置转为环境变量 `TERMINAL_ENV=docker`
3. 首次工具调用到达时，配置可能尚未完全传播——`TERMINAL_ENV` 尚未设置
4. `_get_env_config()` 读取 `os.getenv("TERMINAL_ENV", "local")` → 返回 `"local"`
5. 工具在**本地**执行（宿主机），同时配置传播在后台完成
6. 第二次调用时 `TERMINAL_ENV=docker` 已就绪 → 正常在容器内执行

**证据链：**
- 关联 issue #25402："terminal.backend=local can still use Docker when stale Docker config keys remain"——配置传播/残留问题
- Reporter 原话："first tool call runs on local before docker instance is ready"
- Reporter 提到 PR #1276 添加了 Docker preflight 但**没有预拉取镜像或阻塞工具调用**

### 2.5 竞态时序图

```
Gateway 启动
  ├─ 读取 config.yaml (terminal.backend: docker)
  ├─ 设置 TERMINAL_ENV=docker ───────┐ (异步传播)
  │                                   │
首次工具调用 ←────────────────────────┘ (可能还未传播完成)
  ├─ os.getenv("TERMINAL_ENV", "local") → "local"
  ├─ _create_environment("local", ...) → LocalEnvironment
  ├─ 命令在宿主机上执行！← 沙箱逃逸
  └─ 返回宿主机路径 (如 ~/username/...)

TERMINAL_ENV=docker 传播完成

第二次工具调用
  ├─ os.getenv("TERMINAL_ENV", "local") → "docker"
  ├─ _create_environment("docker", ...) → DockerEnvironment
  ├─ docker run -d ... (阻塞拉取镜像)
  └─ 命令在容器内执行 ✓
```

---

## 三、修复方案设计

### 方案概述

修改 `tools/terminal_tool.py` 中的环境配置解析逻辑，确保 Docker 配置的读取不依赖环境变量传播时序。

### 改动 1：`_get_env_config()` —— 直接从配置文件读取后端类型

**文件：** `tools/terminal_tool.py`，约第 1244 行

**现状：**
```python
env_type = os.getenv("TERMINAL_ENV", "local")
```

**改为：**
```python
# 优先从 config.yaml 读取，避免环境变量传播竞态
env_type = _resolve_backend_type()

def _resolve_backend_type() -> str:
    """从配置源读取 terminal backend 类型，不依赖环境变量传播时序。
    
    优先级：TERMINAL_ENV 环境变量 > config.yaml > 默认 local
    """
    env_val = os.getenv("TERMINAL_ENV")
    if env_val:
        return env_val
    
    # 直接从 config.yaml 读取，绕过环境变量传播延迟
    try:
        from hermes_cli.config import load_config
        cfg = load_config()
        backend = cfg.get("terminal", {}).get("backend", "local")
        if backend in ("docker", "singularity", "modal", "daytona", "ssh", "local"):
            return backend
    except Exception:
        pass
    
    return "local"
```

### 改动 2：`_get_env_config()` —— 首次调用时硬等待配置就绪

**文件：** `tools/terminal_tool.py`

如果 `_resolve_backend_type()` 返回 `"local"` 但后续发现 Docker 相关配置项非空，说明配置传播可能尚未完成——应等待片刻后重试：

```python
# 在 _get_env_config() 中，解析完 env_type 后追加
docker_image = os.getenv("TERMINAL_DOCKER_IMAGE")
docker_volumes_raw = os.getenv("TERMINAL_DOCKER_VOLUMES")

# 检测配置不一致：env_type=local 但 Docker 配置已设置
if env_type == "local" and (docker_image or docker_volumes_raw):
    logger.warning(
        "Backend is 'local' but Docker config keys are present. "
        "This may indicate a config propagation race. "
        "Waiting 2s and re-checking..."
    )
    time.sleep(2)
    env_type = os.getenv("TERMINAL_ENV", env_type)
    if env_type == "local":
        logger.warning(
            "Backend still 'local' after wait. Docker config keys exist "
            "but TERMINAL_ENV is not set. Falling back to local — if you "
            "intended Docker, check your config propagation."
        )
```

### 改动 3：`terminal_tool.py` —— 首次创建环境时预拉取镜像

**文件：** `tools/terminal_tool.py`，约第 2205 行（`_create_environment` 调用后）

在 Docker 环境创建后，立即触发镜像预拉取（在 `__init__` 阶段做，可配置超时）：

```python
# _create_environment 返回后追加
if env_type == "docker" and hasattr(new_env, 'prewarm'):
    try:
        new_env.prewarm(timeout=30)  # 预拉取，短超时
    except Exception as e:
        logger.warning("Docker prewarm failed: %s", e)
```

**同时在 `docker.py` 的 `DockerEnvironment` 中添加 `prewarm()` 方法：**

```python
def prewarm(self, timeout=30):
    """预拉取镜像但不执行命令。在环境创建时调用，缩短首次工具调用延迟。"""
    if self._container_id:
        return  # 已就绪
    # 触发容器创建（会拉取镜像）
    self.run("true", timeout=timeout)
```

### 改动 4（保险措施）：`docker.py` —— 绝不容忍不安全回退

**文件：** `tools/environments/docker.py`

在 `DockerEnvironment.__init__()` 末尾添加：

```python
# 确保不会回退到不安全的执行路径
self._started = False

def _run_bash(self, cmd_string, ...):
    assert self._container_id, (
        "Container not started — Docker backend must have a running container. "
        "This is a bug: the environment should be started before command execution."
    )
```

### 改动优先级

| 优先级 | 改动 | 说明 |
|--------|------|------|
| **P0（必须）** | 改动 1 | 从配置文件直接读取后端类型，绕过环境变量传播竞态 |
| **P1（强烈建议）** | 改动 2 | 检测配置不一致并告警，帮助发现传播问题 |
| **P2（改善体验）** | 改动 3 | 预拉取镜像缩短冷启动延迟 |
| **P3（纵深防御）** | 改动 4 | 代码层面确保不会静默回退 |

---

## 四、为什么这是面试好素材

| 概念 | 在漏洞中的体现 |
|------|--------------|
| **竞态条件** | 配置传播与首次工具调用之间的时序竞争 |
| **故障开放 vs 故障关闭** | `env_type` 回退到 `"local"` 而非报错——安全上的错误选择 |
| **默认值安全** | `os.getenv("TERMINAL_ENV", "local")` 的默认值 `"local"` 是不安全的默认值 |
| **纵深防御** | 单层防御（环境变量）被突破后无第二道防线 |
| **幂等性** | 同一命令第一次（宿主机）和第二次（容器）结果不同 |
| **可观测性** | 缺少告警机制——用户可能长期未发现工具在宿主机执行 |

---

## 五、关联资源

- Issue: https://github.com/NousResearch/hermes-agent/issues/54354
- 相关 Issue: #25402（配置残留导致后端选择不一致）
- 相关 PR: #1276（Docker preflight 检查，但未修复此问题）
- 提交修复前需签署 CLA：https://www.nousresearch.com/cla
- 贡献指南：https://github.com/NousResearch/hermes-agent/blob/main/CONTRIBUTING.md
