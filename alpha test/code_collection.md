# Directory: backend

### __init__.py
```

```

### container.py
```
# backend/container.py

import logging
from typing import Dict, Any, Callable, Set
# 1. 将 threading.Lock 替换为 threading.RLock
import threading

from backend.core.contracts import Container as ContainerInterface

logger = logging.getLogger(__name__)


class Container(ContainerInterface):
    """一个简单的、通用的、线程安全的依赖注入容器，带循环依赖检测。"""
    def __init__(self):
        self._factories: Dict[str, Callable] = {}
        self._singletons: Dict[str, bool] = {}
        self._instances: Dict[str, Any] = {}
        # 2. 使用 RLock (Re-entrant Lock) 替代 Lock
        self._lock = threading.RLock()
        
        # 使用 threading.local() 来创建线程本地的解析栈
        # 每个线程都会有自己独立的 `_resolution_stack` 副本
        self._local = threading.local()

    def _get_resolution_stack(self) -> Set[str]:
        """安全地获取或初始化当前线程的解析栈。"""
        if not hasattr(self._local, 'resolution_stack'):
            self._local.resolution_stack = set()
        return self._local.resolution_stack

    def register(self, name: str, factory: Callable, singleton: bool = True) -> None:
        """注册一个服务工厂。"""
        if name in self._factories:
            logger.warning(f"Overwriting service registration for '{name}'")
        self._factories[name] = factory
        self._singletons[name] = singleton

    def resolve(self, name: str) -> Any:
        """
        解析（获取）一个服务实例。
        此方法是线程安全的，并能检测循环依赖。
        """
        # 循环依赖检测
        resolution_stack = self._get_resolution_stack()
        if name in resolution_stack:
            path = " -> ".join(list(resolution_stack) + [name])
            raise RuntimeError(f"Circular dependency detected: {path}")

        resolution_stack.add(name)

        try:
            # --- 原有的双重检查锁定逻辑 ---
            is_singleton = self._singletons.get(name, True)
            if is_singleton and name in self._instances:
                return self._instances[name]

            if name not in self._factories:
                raise ValueError(f"Service '{name}' not found in container.")

            if not is_singleton:
                factory = self._factories[name]
                try:
                    return factory(self)
                except TypeError:
                    return factory()

            with self._lock:
                if name in self._instances:
                    return self._instances[name]

                factory = self._factories[name]
                try:
                    instance = factory(self)
                except TypeError:
                    instance = factory()

                logger.debug(f"Resolved service '{name}'. Singleton: True")
                self._instances[name] = instance
                return instance
        finally:
            resolution_stack.remove(name)
```

### README.md
```
# Hevno Engine

**一个为构建复杂、持久、可交互的 AI 世界而生的状态图执行引擎。**

---

## 1. 我们的愿景与核心设计哲学

### 1.1 我们的愿景：从“聊天机器人”到“世界模拟器”

当前的语言模型（LLM）应用，大多停留在“一问一答”的聊天机器人模式，这极大地限制了 LLM 的潜能。我们相信，LLM 的未来在于构建**复杂、持久、可交互的动态世界**。

想象一下：

*   一个能与你玩《是，大臣》策略游戏的 AI，其中每个角色（哈克、汉弗莱、伯纳）都是一个独立的、有自己动机和知识库的 LLM 实例。
*   一个能让你沉浸式体验的互动小说，你的每一个决定都会被记录，并动态地解锁、修改甚至创造新的故事线和世界规则。

这些不再是简单的“提示工程”，而是**状态管理**、**并发控制**和**动态逻辑编排**的复杂工程问题。

**Hevno Engine 的诞生，正是为了解决这个问题。** 它的核心使命是提供一个强大的后端框架，让开发者能够轻松地将离散的 LLM 调用，编织成有记忆、能演化、可回溯的智能代理和交互式世界。我们不是在构建另一个聊天应用，我们是在构建一个**创造世界的引擎**。

### 1.2 核心设计哲学：您如何构建世界

这四条哲学是您作为“世界创造者”与 Hevno 引擎交互的基础。它们定义了您如何通过图（Graph）来描述和构建动态世界的行为逻辑。

> **注意**: 这些哲学描述的是由 `core_engine` 插件实现的**图执行模型**，与后端代码的组织方式（插件架构）是两个不同但互补的概念。

#### 1.2.1 哲学一：以运行时为中心，指令式地构建行为

> **"Behavior is a sequence of instructions."**

我们摒弃了将所有配置混杂在一起的模式，转而采用一种更清晰、更强大的**指令式**设计。在 Hevno 中：

*   **极简的节点 (`GenericNode`)**: 节点本身只是一个容器。
*   **行为由指令驱动**: 节点的具体行为由其 `run` 字段中一个**有序的指令列表**所定义。
*   **原子指令 (`RuntimeInstruction`)**: 每个指令都包含一个 `runtime`（一个可执行的功能单元）和它自己独立的 `config`（配置）。这确保了逻辑的清晰和数据的隔离。
*   **强大的节点内管道**: 引擎会严格按照指令列表的顺序执行。后一个指令的宏，可以访问到前一个指令执行后产生的最新状态。

**Hevno 的指令式设计:**
```json
// 一个先通过宏设置世界状态，再调用 LLM 的复杂节点
{
    "id": "advanced_llm",
    "run": [
    {
        "runtime": "system.execute",
        "config": {
        "code": "{{ moment.character_mood = 'happy' }}"
        }
    },
    {
        "runtime": "llm.default",
        "config": {
        "prompt": "{{ f'根据角色愉悦的心情 ({moment.character_mood})，生成一句问候。' }}"
        }
    }
    ]
}
```
这种设计将节点的行为分解为一系列原子的、可预测的步骤，提供了无与伦比的控制力和可读性。

#### 1.2.2 哲学二：状态先行，计算短暂 (State is permanent, execution is ephemeral.)

> 一个交互式模拟世界的核心恰恰是“状态”。我们构建了一个以**三层作用域**为核心的、清晰而强大的状态架构，以解决不同类型数据的生命周期问题。
 
 *   **沙盒 (`Sandbox`)**: 代表一个完整的、隔离的交互环境（例如，一局游戏、一个项目）。它现在是所有状态的根容器。
 *   **三层状态作用域**:
     1.  **`definition` (静态定义层)**: 沙盒的“设计蓝图”。它在沙盒创建时被定义，并且在整个生命周期中是**只读**的。它约定了世界的初始样貌。
     2.  **`lore` (演化知识层)**: 沙盒的“世界法典”或“历史书”。这部分状态会随着时间演化，但 crucial，它在**读档（Revert）时不会被回滚**。它代表了世界不可逆的成长。
     3.  **`moment` (瞬时状态层)**: 沙盒的“即时快照”，包含了所有**可回滚**的状态，例如玩家的生命值 (`moment.player.hp`)、当前位置、以及短期记忆 (`moment.memoria`)。
 
 #### 两种修改模式：引擎的“演进” vs. 创作者的“编辑”
 
 Hevno 引擎深刻理解“世界演进”和“创作者编辑”是两种完全不同的操作，并为之提供了不同的状态修改机制：
 
 1.  **引擎驱动的演进 (不可变)**: 当**引擎通过 `step()` 方法执行图逻辑时**，它遵循严格的不可变原则。它读取一个状态快照，经过计算，然后**总是创建一个新的历史快照**。这保证了所有由AI或游戏规则驱动的世界演进都是可追溯、可回滚的，确保了模拟的确定性。
 
 2.  **创作者驱动的编辑 (可变)**: 当**您（创作者）通过统一资源API进行修改时**，系统服务于您的直觉和效率。
     *   对 `definition` 或 `lore` 的修改会**直接更新**沙盒的蓝图或法典。
     *   对 `moment` 的修改，您可以选择**直接更新**当前快照的内容（默认行为），而**不创建新的历史**。这允许您像在画布上作画一样自由地调试和设定场景，而不会产生冗余的快照。

**不可变演进执行流程示意：**
```
+-------------------------------------------------+
|                    Sandbox (Old)                |
|  - definition (只读)                             |
|  - lore (演化知识)                               |
|  - head_snapshot_id -> (Old Moment Snapshot)    |
+-------------------------------------------------+
                          |
                          v
+-------------------------------------------------+
|                Execution Engine                 |
|               (State Transition)                |
|                   step(sandbox)                 |
+-------------------------------------------------+
                          |
                          v
+-------------------------------------------------+
|                    Sandbox (New)                |
|  - definition (不变)                             |
|  - lore (可能已更新)                              |
|  - head_snapshot_id -> (New Moment Snapshot)    |
+-------------------------------------------------+
```

这种架构天然地带来了巨大的好处：
1.  **完美的回溯能力**: “读档”操作变成了简单地将沙盒的 `head_snapshot_id` 指针指向一个历史快照。`lore` 中演化的世界规则和历史保持不变，只有 `moment` 中的角色状态等被回滚。
2.  **健壮的并发与调试**: 不可变性消除了大量的并发问题，并使得追踪状态变化变得异常简单。
3.  **动态的逻辑演化**: 因为图和知识库的定义本身也是 `lore` 状态的一部分，所以它们可以被自己执行的逻辑所修改，实现世界的“自我进化”。

而对于创作者通过 API 进行的编辑操作，一切操作以**修改就是修改**为原则，最大化对直觉的尊重和对创作者意图的服务。

#### 1.2.3 哲学三：约定与配置相结合，智能推断与明确声明并存

> **"Be smart, but provide an escape hatch."**

我们力求为图的创建者提供最流畅的体验，在大多数情况下，你无需关心节点间的连接。但我们也承认，在复杂场景下，明确性优于魔法。

*   **智能的依赖推断 (约定)**: 在我们的图定义中，你通常找不到 `edges` 字段。当一个节点的指令在其 `config` 中通过宏 `{{ nodes.A.output }}` 引用了另一个节点 `A` 时，引擎会自动建立一条从 `A` 到当前节点的执行依赖。这是我们的主要约定，能处理 90% 的场景。

*   **明确的依赖声明 (配置)**: 对于那些无法通过宏自动推断的**隐式依赖**（例如，一个节点通过副作用修改了 `moment` 状态，而另一个节点依赖这个状态），我们提供了一个明确的 `depends_on` 字段。这让你可以在需要时精确控制执行顺序，消除竞态条件。

**依赖推断示例:**
```json
// 节点 B 自动依赖节点 A，因为它的宏引用了 nodes.A
{
  "id": "B",
  "run": [{
    "runtime": "llm.default",
    "config": { "prompt": "{{ f'基于A的结果: {nodes.A.output}' }}" }
  }]
}
```

**明确声明依赖示例:**
```json
// B 节点读取由 A 节点设置的状态，这是一种隐式依赖。
// 我们使用 depends_on 来确保 A 在 B 之前执行。
{
    "id": "A_set_state",
    "run": [{ "runtime": "system.execute", "config": { "code": "{{ moment.theme = 'fantasy' }}" }}]
},
{
    "id": "B_read_state",
    "depends_on": ["A_set_state"],
    "run": [{
    "runtime": "llm.default",
    "config": { "prompt": "{{ f'生成一个关于 {moment.theme} 世界的故事。' }}" }
    }]
}
```
这种“约定为主，配置为辅”的设计，将开发者的精力从繁琐的工程细节中解放出来，同时在关键时刻给予他们完全的控制权。

#### 1.2.4 哲学四：默认安全，并发无忧 (Safe by Default, Concurrently Sound)

> **"Write natural code, get parallel safety for free."**

在 Hevno 中，图的节点可以并行执行，这极大地提升了效率。但并发也带来了风险：如果两个并行节点同时修改同一个状态（如 `moment.counter`），就会产生不可预测的结果（竞态条件）。

我们坚信，开发者不应该为了利用并发而成为并发控制专家。因此，Hevno Engine 内置了**宏级原子锁**机制：

*   **默认开启，完全透明**: 您无需编写任何特殊代码。引擎会自动确保每一个宏脚本的执行都是一个**原子操作**。
*   **无竞态条件之忧**: 当您在宏中编写 `moment.counter += 1` 时，即使有十个节点同时执行这段代码，引擎也能保证其过程不会被打断，最终结果绝对正确。
*   **专注业务逻辑**: 您可以像在单线程环境中一样自然地编写代码，将全部精力投入到构建世界逻辑中，而引擎在幕后处理了所有复杂的并发安全问题。

**并发写入示例:**
```json
// 节点 A 和 B 会被并行执行
// 但由于宏级原子锁，对 moment.gold 的修改是安全的
{
  "id": "A_earn_gold",
  "run": [{ "runtime": "system.execute", "config": { "code": "moment.gold += 10" } }]
},
{
  "id": "B_spend_gold",
  "run": [{ "runtime": "system.execute", "config": { "code": "moment.gold -= 5" } }]
}
```
这种设计让您能够无缝地从简单的线性逻辑扩展到复杂的高性能并行图，而无需修改一行状态操作代码。

---

## 2. 软件架构：一个可插拔的插件化系统

Hevno 的后端架构遵循“微内核 + 插件”的设计模式，其灵感来自于 VS Code 或 OBS 等高度可扩展的软件。这种设计旨在实现最大限度的**模块化**、**解耦**和**可扩展性**。

### 2.1 宏大构想：从单体应用到微内核平台

旧的架构是一个紧耦合的单体，`container.py` 和 `app.py` 知道所有服务的创建细节。新的架构则完全不同：

*   **平台内核 (`backend/`)**: `backend` 目录被精简为一个与业务无关的“微内核”。它不包含任何具体的业务逻辑（如图执行、LLM 调用），只提供所有插件赖以生存的基础设施。
*   **插件 (`plugins/`)**: 所有的核心功能，如执行引擎、API 接口、LLM 网关、持久化服务等，都被重构成独立的、可互换的插件包。每个插件都是一个自包含的功能单元。

您可以将 Hevno 平台想象成一个**操作系统**：
*   **内核 (`backend/`)** 提供了进程管理（`Container`）和系统调用（`HookManager`）等底层能力。
*   **驱动和程序 (`plugins/`)** 是在操作系统上运行的独立软件，它们遵循操作系统的规则（`Contracts`），共同构建出完整的用户体验。

### 2.2 架构的三大支柱

插件化系统建立在三个核心概念之上，它们共同构成了插件之间协作的基石。

#### 支柱一：共享契约 (`backend/core/contracts.py`) - 通用语言

这是整个系统中最关键的文件，是所有插件必须遵守的“法律”和“API 规范”。它定义了：

1.  **核心数据模型**: 如 `StateSnapshot`, `Sandbox`, `GenericNode` 等。确保了所有插件对“状态”和“图”有统一的理解。
2.  **核心服务接口**: 如 `ExecutionEngineInterface`, `SnapshotStoreInterface`。插件不应依赖于其他插件的具体实现类，而应依赖这些抽象接口。这使得任何插件（例如一个第三方的持久化插件）都可以替代核心插件，只要它实现了相同的接口。
3.  **系统事件模型**: 为所有通过钩子系统传递的事件定义了标准的 Pydantic 模型，如 `NodeExecutionStartContext`。

> **黄金法则**: 一个插件只能导入 `backend.core.contracts` 和它自身包内的模块。它**永远不应**直接从另一个插件（如 `from plugins.core_engine.engine import ExecutionEngine`）导入具体实现。

#### 支柱二：依赖注入容器 (`backend/container.py`) - 服务总管

容器是一个通用的服务定位器，它解耦了“服务的使用”和“服务的创建”。

*   **工作模式**:
    1.  **注册 (Registration)**: 在启动时，每个插件的 `register_plugin` 函数会向容器注册一个或多个**工厂函数**。这个工厂函数知道如何创建该插件提供的服务。
    2.  **解析 (Resolution)**: 当系统中任何部分（另一个服务或一个 API 端点）需要一个服务时，它不会自己去 `new` 一个实例，而是向容器请求 `container.resolve("service_name")`。容器会查找对应的工厂，创建实例（如果是第一次请求且是单例），并返回它。

*   **带来的好处**:
    *   **解耦**: `core_engine` 插件不需要知道 `LLMService` 是如何被创建的，它只需要知道去容器里找一个名为 `"llm_service"` 的服务即可。
    *   **可配置性**: 我们可以轻松地替换实现。例如，在测试环境中，可以注册一个 `MockLLMService` 工厂来代替真实的 `LLMService` 工厂，而使用服务的代码无需任何改动。
    *   **懒加载**: 服务只在第一次被请求时才会被创建，避免了不必要的启动开销。

**示例：`LLMService` 的生命周期**
```python
# 1. 在 core_llm/__init__.py 中，插件注册了一个工厂
def _create_llm_service(container: Container):
    # ...复杂的服务创建逻辑，可能依赖其他服务...
    return LLMService(...)

def register_plugin(container: Container, hook_manager: HookManager):
    container.register("llm_service", _create_llm_service)
```
```python
# 2. 在 core_engine/evaluation.py 中，宏上下文被构建
#    注意 'services' 是一个特殊的代理对象
services = DotAccessibleDict(ServiceResolverProxy(container))
```
```python
# 3. 在图的宏中，用户懒加载并使用服务
#    这是第一次访问 services.llm_service，
#    它会触发 ServiceResolverProxy -> container.resolve("llm_service")
#    -> _create_llm_service()，最终返回实例。
"{{ services.llm_service.request(...) }}"
```

#### 支柱三：钩子系统 (`backend/core/hooks.py`) - 协作总线

如果说 DI 容器解决了“谁来创建服务”的问题，那么钩子系统就解决了“插件如何安全地对话和扩展彼此”的问题。它是一个发布-订阅（Pub/Sub）事件总线。

*   **工作模式**:
    1.  **触发 (Trigger)**: 一个插件（发布者）在某个关键执行点，会通过 `hook_manager` 触发一个命名的事件（钩子），并传递相关数据。例如，`core_engine` 在需要所有运行时的时候，会触发 `"collect_runtimes"` 钩子。
    2.  **实现 (Implementation)**: 其他插件（订阅者）可以向 `hook_manager` 注册一个异步函数，以响应该钩子。
    3.  **执行 (Execution)**: `hook_manager` 负责调用所有已注册的实现函数，并根据钩子类型（如 `filter`）聚合它们的返回值。

*   **钩子类型**:
    *   **通知 (`trigger`)**: “我刚刚做完了这件事，通知大家一下。” 所有实现并发执行，返回值被忽略。
    *   **过滤 (`filter`)**: “我有一份数据，谁想在上面添加或修改一些东西？” 所有实现按优先级顺序链式执行，后一个实现会接收前一个修改过的数据。非常适合用于收集信息。

**示例：`core_engine` 如何从所有插件收集运行时和 API 路由**
```python
# 在 core_engine/__init__.py 中...
async def populate_runtime_registry(container: Container):
    # 引擎广播：“我需要运行时，请把你们的运行时给我。”
    all_runtimes = await hook_manager.filter("collect_runtimes", {})
    # ...然后注册收集到的所有运行时...

# 在 app.py 中...
async def lifespan(app: FastAPI):
    # 应用启动时广播：“我需要 API 路由，请把你们的路由给我。”
    all_routers = await hook_manager.filter("collect_api_routers", [])
    for router in all_routers:
        app.include_router(router)
```
```python
# 在 core_llm/__init__.py 中...
async def provide_runtime(runtimes: dict) -> dict:
    runtimes["llm.default"] = LLMRuntime # LLM 插件响应
    return runtimes

# 在 core_api/__init__.py 中...
async def provide_own_routers(routers: list) -> list:
    routers.append(sandbox_router) # API 插件响应
    return routers

# 注册钩子实现
def register_plugin(container: Container, hook_manager: HookManager):
    hook_manager.add_implementation("collect_runtimes", provide_runtime)
    hook_manager.add_implementation("collect_api_routers", provide_own_routers)
```
通过这种方式，`core_engine` 和 `app.py` 根本不需要知道 `core_llm` 或 `core_api` 插件的存在，但依然能获取它们提供的功能，实现了彻底的解耦。

### 2.3 插件剖析：如何构建一个插件

每个位于 `plugins/` 目录下的子目录都是一个独立的插件。一个标准插件的结构如下：

```
plugins/
└── my_awesome_plugin/
    ├── __init__.py         # 插件的入口和注册点
    ├── manifest.json       # 插件的元数据
    ├── service.py          # 插件提供的核心服务
    ├── router.py           # 插件提供的 API 路由
    └── models.py           # 插件特有的数据模型
```

*   **`manifest.json`**: 定义了插件的元数据，如名称、版本、描述，以及最重要的**加载优先级 (`priority`)**。优先级越低的插件越先被加载。
*   **`__init__.py`**: 这是插件的入口文件，必须包含一个名为 `register_plugin(container: Container, hook_manager: HookManager)` 的函数。平台加载器会调用此函数，将内核服务注入进来，插件在此函数内完成所有服务和钩子的注册。

### 2.4 应用启动生命周期

整个应用的启动过程被清晰地定义在 `app.py` 的 `lifespan` 函数中，它展示了上述机制如何协同工作：

1.  **内核初始化**: 创建 `Container` 和 `HookManager` 实例。
2.  **同步注册阶段**: `PluginLoader` 被调用。它会：
    a. 扫描 `plugins` 目录并根据 `manifest.json` 的 `priority` 排序。
    b. 按顺序调用每个插件的 `register_plugin` 函数。在此阶段，插件只向容器注册**服务工厂**，和向钩子管理器注册**钩子实现**。服务实例**尚未被创建**。
3.  **异步初始化阶段**: `lifespan` 触发 `"services_post_register"` 钩子。
    *   监听此钩子的插件（如 `core_engine` 和 `core_api`）现在可以安全地从容器中解析依赖，并执行需要 `async` 的初始化任务，例如从其他插件收集并填充自己的注册表（如 `RuntimeRegistry`）。
4.  **API 路由收集**: `lifespan` 触发 `"collect_api_routers"` 钩子，收集所有插件提供的 FastAPI `APIRouter` 实例，并挂载到主 `app` 上。
5.  **启动完成**: 触发 `"app_startup_complete"` 钩子，应用正式就绪，开始接受请求。


## 3. 图与宏系统定义

本章节将深入探讨您作为“世界创造者”与 Hevno 引擎交互的核心——**图定义**。这部分内容主要由 `core_engine` 插件实现，但其使用方式对所有插件都是通用的。

### 3.1 图定义格式与核心概念

#### 3.1.1 顶层结构：图集合 (Graph Collection)

一个完整的工作流定义是一个 JSON 对象，其 `key` 为图的名称，`value` 为图的定义。

*   **约定入口图的名称必须为 `"main"`**。
*   这允许多个可复用的图存在于同一个配置文件中。

**示例:**
```json
{
  "main": {
    "nodes": [ /* ... 主图的节点 ... */ ]
  },
  "process_character_arc": {
    "nodes": [ /* ... 一个可复用子图的节点 ... */ ]
  }
}
```

#### 3.1.2 节点 (Node) 与指令 (Instruction)

节点是图的基本执行单元，其行为由一个或多个有序的指令定义。

```json
{
  "id": "unique_node_id_within_graph",
  "depends_on": ["another_node_id"],
  "run": [
    {
      "runtime": "runtime_name_A",
      "config": {
        "param1": "value1",
        "param2": "{{ nodes.data_provider.output }}" 
      }
    },
    {
      "runtime": "runtime_name_B",
      "config": {
        // 这个指令的配置
      }
    }
  ]
}
```
*   `id`: 节点在图内的唯一标识符。
*   `run`: 一个**有序的**指令列表，定义节点的行为。
*   `depends_on` (可选): 一个节点 ID 的列表。用于明确声明当前节点必须在列表中的所有节点成功执行后才能开始，解决了无法自动推断的隐式依赖问题。

### 3.2 Hevno 宏系统：可编程的配置

欢迎来到 Hevno 宏系统，这是让您的静态图定义变得鲜活、动态和智能的核心引擎。我们摒弃了复杂的模板语言，转而拥抱一种更强大、更直观的理念：

> **在配置中，像写 Python 一样思考。**

宏系统允许您在图定义（JSON 文件）的字符串值中直接嵌入可执行的 Python 代码。它不仅能用于简单的变量替换，更是实现动态逻辑、状态操作和世界演化的瑞士军刀。

#### 3.2.1 核心理念：逐步求值，精确控制

##### 3.2.1.1 唯一的语法：`{{ ... }}`

您只需要记住一种语法。任何被双大括号 `{{ ... }}` 包裹起来的内容，都会被 Hevno 引擎视为一段可执行的 Python 代码。

```json
// 简单求值
{ "config": { "value": "{{ 1 + 1 }}" } }

// 访问状态
{ "config": { "prompt": "{{ f'你好，{moment.player_name}！' }}" } }

// 执行复杂逻辑并修改状态
{
  "config": {
    "script": "{{
      if moment.player.is_tired:
          moment.player.energy -= 10
      else:
          moment.player.energy += 5
    }}"
  }
}
```

##### 3.2.1.2 智能的执行模型：指令前的即时求值 (Just-in-Time Evaluation)

这是理解宏系统强大能力的关键。宏的求值不是在节点开始时一次性完成的，而是与节点的指令执行紧密相连。

在一个节点内，引擎会严格按照 `run` 列表中的指令顺序执行。在**每一个**指令即将执行其 `runtime` **之前**，引擎会自动**遍历该指令的 `config`**。当它遇到一个值为 `{{...}}` 宏格式的字符串时，它会**执行一次**该宏，并用其返回结果**替换**掉原有的宏字符串。

这意味着：
1.  **所见即所得**：当您的运行时（如 `llm.default`）拿到 `prompt` 参数时，它**永远**是最终的、计算好的字符串。
2.  **节点内状态流动**：一个指令的宏，可以**立即访问**到该节点内**上一个指令**执行后产生的任何状态变化。这是实现复杂节点内逻辑链的关键。
3.  **隐式返回值**: 如果您的代码块最后一行是一个表达式（例如一个数字、一个字符串、一个函数调用），它的结果将成为这个宏的值。否则，其值为 `None`。

#### 3.2.2 入门指南 (为所有用户)

##### 3.2.2.1 【已重构】访问核心数据：您的世界交互窗口

宏最强大的能力，在于它能访问和操纵 Hevno 引擎在执行图过程中的所有内部状态。您可以把宏想象成一个开在指令配置上的“开发者控制台”，能让您直接与引擎的“记忆”互动。

在 `{{ ... }}` 内部，您可以访问一个包含了**所有可用上下文信息**的全局命名空间。这些上下文对象都支持便捷的**点符号访问**。

*   **`moment` (瞬时状态)**: 这是您沙盒的“即时快照”，是您最常交互的对象。所有需要跨越多个执行步骤、但在读档时**应该被回滚**的数据都应存放在这里。您可以读取它，也可以向其中写入新数据或修改现有数据，这些改动将被永久记录在下一个状态快照中。
    *   **用途**: 存储玩家属性（如 `moment.player.hp`）、任务进度、当前位置、短期记忆 (`moment.memoria`) 等。
    *   **示例 (读取)**: `"{{ f'玩家当前生命值：{moment.player.hp}' }}"`
    *   **示例 (写入)**: `"{{ moment.quest_log.append('新任务：击败恶龙') }}"`

*   **`lore` (演化知识)**: 这是您沙盒的“世界法典”。所有需要长期存在、并且在读档时**不应被回滚**的数据存放在这里。它代表了世界的永久性成长和演化。
    *   **用途**: 存储可演化的图定义 (`lore.graphs`)、可进化的知识库 (`lore.codices`)、已解锁的世界规则等。
    *   **示例 (读取)**: `"{{ f'当前世界规则版本：{lore.rules.version}' }}"`
    *   **示例 (写入)**: `"{{ lore.graphs['new_magic_spell'] = {...} }}"`

*   **`definition` (静态定义)**: 这是您沙盒的“设计蓝图”，在运行时是**只读**的。它定义了世界的初始状态。
    *   **用途**: 读取沙盒的初始配置，例如初始的 lore 和 moment。
    *   **示例**: `"{{ definition.initial_lore.description }}"`

*   **`nodes` (已完成节点的结果)**: 这是一个对象，其属性是所有在当前节点执行之前，已经成功完成的节点的 `id`。您可以访问这些节点的最终输出结果。这是实现**节点间**数据流动的关键。
    *   **用途**: 将一个节点的输出作为另一个节点的输入。
    *   **示例**: `"{{ nodes.get_character_name.output.upper() }}"`

*   **`pipe` (节点内管道状态)**: 这是一个特殊的对象，它包含了**本节点内**所有**上一个指令**执行完成后的输出结果。它允许您在一个节点内部构建强大的数据处理管道，实现**指令间**的数据流动。
    *   **用途**: 将 `system.io.input` 的结果传给 `llm.default`。
    *   **示例**:
        ```json
        "run": [
            { "runtime": "system.io.input", "config": { "value": "a cat" } },
            { "runtime": "llm.default", "config": { "prompt": "{{ f'Tell me a story about {pipe.output}' }}" } }
        ]
        ```

*   **`run` (本次运行的临时数据)**: 这是一个临时存储区域，其生命周期仅限于**单次**图的执行。执行结束后，其中的所有数据都会被丢弃。
    *   **用途**: 存储触发本次运行的外部输入（如用户的聊天消息）、本次运行中途的临时计算结果等。
    *   **示例**: `"{{ run.triggering_input.user_message }}"`

*   **`services` (可用的服务)**: 这是一个特殊的代理对象，是您访问所有已注册插件服务的入口。服务是**懒加载**的：只有当您在宏中第一次访问某个服务（如 `services.llm_service`）时，DI 容器才会创建或获取该服务的实例。
    *   **用途**: 从宏中直接调用任何插件提供的功能，例如发起 LLM 请求或访问持久化存储。
    *   **示例**:
        ```json
        {
          "runtime": "system.execute",
          "config": {
            "code": "{{ services.llm_service.request(model_name='gemini/gemini-pro', messages=[{'role': 'user', 'content': '你好！'}]) }}"
          }
        }
        ```

*   **`session` (会话元信息)**: 包含了关于整个交互会话的全局信息，例如会话开始的时间、总共执行的回合数等。
    *   **用途**: 用于记录、调试或实现与时间相关的逻辑。
    *   **示例**: `"{{ f'当前是第 {session.turn_count} 回合' }}"`

##### 3.2.2.2 “开箱即用”的工具箱

我们预置了一些标准 Python 模块，您无需 `import` 即可直接使用：`random`, `math`, `datetime`, `json`, `re`。

*   掷一个20面的骰子: `"{{ random.randint(1, 20) }}"`
*   从列表中随机选一个: `"{{ random.choice(['红色', '蓝色', '绿色']) }}"`

#### 3.2.3 实用示例

##### 示例1：动态生成 NPC 对话

根据玩家的声望 (`moment.player_reputation`)，NPC 会有不同的反应。这个例子展示了如何使用宏动态构建发送给 LLM 的指令，并利用 `contents` 列表来组织对话。

```json
{
  "id": "npc_greeting",
  "run": [{
    "runtime": "llm.default",
    "config": {
      "model": "gemini/gemini-1.5-pro",
      "contents": [
        {
          "role": "system",
          "content": "你是一名角色扮演游戏中的NPC。根据我提供的上下文，生成一句符合角色身份和当前情境的对话。"
        },
        {
          "role": "user",
          "content": "{{
            rep = moment.player_reputation
            if rep > 50:
                f'上下文：你看到了一位声望显赫的英雄，玩家 {moment.player_name}。请生成一句充满敬意的欢迎语。'
            elif rep < -50:
                f'上下文：你看到了一个臭名昭著的恶棍，玩家 {moment.player_name}。请生成一句充满警惕和蔑视的话语。'
            else:
                f'上下文：你看到了一个你不怎么认识的普通冒险者，玩家 {moment.player_name}。请生成一句平淡而中立的问候。'
          }}"
        }
      ]
    }
  }]
}
```

##### 示例2：在一个节点内完成“计算伤害并更新状态”

这个例子完美地展示了指令式执行和 `pipe` 对象的能力。

```json
{
    "id": "take_damage",
    "run": [
    {
        "runtime": "system.io.input",
        "config": {
        "value": "{{ run.triggering_input.damage }}"
        }
    },
    {
        "runtime": "system.execute",
        "config": {
        "code": "{{
            damage_amount = pipe.output
            moment.player_hp -= damage_amount
            moment.battle_log.append(f'玩家受到了 {damage_amount} 点伤害。')
        }}"
        }
    }
    ]
}
```
**执行流程**:
1.  第一个指令执行，它的输出 (`damage` 值) 被放入 `pipe` 对象。
2.  第二个指令开始执行，它的 `config` 中的宏被求值。
3.  `pipe.output` 成功地获取了上一步的伤害值。
4.  `moment` 状态被成功修改。

#### 3.2.4 并发安全：引擎的承诺与您的责任

Hevno 引擎天生支持并行节点执行，这意味着没有依赖关系的节点会被同时运行以提升性能。为了让您在享受并行优势的同时，不必担心复杂的数据竞争问题，我们内置了强大的**宏级原子锁 (Macro-level Atomic Lock)** 机制。

##### 3.2.4.1 引擎的承诺：透明的并发安全

Hevno 引擎的核心承诺是：**为所有基于 Python 基础数据类型（字典、列表、数字、字符串等）的状态操作，提供完全透明的、默认开启的并发安全保护。**

这意味着，当您在宏中编写以下代码时，我们保证其结果在任何并行执行下都是正确和可预测的：
*   `moment.counter += 1`
*   `moment.player['stats']['strength'] -= 5`
*   `lore.events.append("New event")`

**工作原理：** 在执行任何一个宏脚本（即 `{{ ... }}` 中的全部内容）之前，引擎会自动获取一个全局写入锁。在宏脚本执行完毕后，锁会自动释放。这保证了**每一个宏脚本的执行都是一个不可分割的原子操作**。您无需做任何事情，即可免费获得这份安全保障。

##### 3.2.4.2 问题的“完美风暴”：何时会超出引擎的保护范围？

我们的自动化保护机制是有边界的。一个操作**必定会**产生不可预测的并发问题（竞态条件），当且仅当它**同时满足以下所有条件**：

1.  **使用自定义类管理可变状态：** 您在宏中定义了 `class MyObject:` 并在其实例中直接存储可变数据（如 `self.hp = 100`），并将其存入 `moment` 或 `lore`。
2.  **使用非纯方法修改状态：** 您调用了该实例的一个方法来直接修改其内部状态（如 `my_obj.take_damage(10)`，其内部实现是 `self.hp -= 10`）。
3.  **真正的并行执行：** 您将这个修改操作放在了两个或多个**无依赖关系**的并行节点中。
4.  **操作同一数据实例：** 这些并行节点操作的是**同一个对象实例**（e.g., `moment.player_character`）。

这个场景的本质是，您创建了一个引擎无法自动理解其内部工作原理的“黑盒”（您的自定义类），并要求引擎在并行环境下保证其内部操作的原子性。这是一个理论上无法被通用引擎自动解决的问题。

##### 3.2.4.3 解决方案路径：从推荐模式到自定义运行时

如果您发现自己确实需要实现上述的复杂场景，我们提供了从易到难、从推荐到专业的解决方案路径：

**第一层（强烈推荐）：遵循“数据-逻辑分离”设计模式**

这是解决此问题的**首选方案**。它不需要您理解并发的复杂性，只需稍微调整代码组织方式：

*   **状态用字典：** `moment.player = {'hp': 100}`
*   **逻辑用函数：** `def take_damage(p, amount): p['hp'] -= amount`
*   **在宏中调用：** `{{ take_damage(moment.player, 10) }}`

这种模式能完美地被我们的自动化安全机制所覆盖，是 99% 的用户的最佳选择。

**第二层（最终选择）：编写自定义运行时（插件）**

如果您是高级开发者，并且有强烈的理由必须使用自定义类和方法来处理并发状态（例如，与外部系统集成或实现极其复杂的领域模型），那么正确的做法是**将这个责任从宏中移出，封装到一个自定义的运行时中**。

**为什么应该使用自定义运行时？**

*   **明确的控制权：** 在您自己的运行时 `execute` 方法中，您可以直接访问 `ExecutionContext` 并获取全局的 **`asyncio.Lock`**。这让您可以**精确地、手动地**控制加锁的范围，以保护您的自定义对象操作。
*   **清晰的职责划分：**
    *   **宏（Macro）** 的设计目标是**快速、便捷的逻辑编排和数据转换**，它不应该承载复杂的、需要手动管理的并发控制逻辑。
    *   **运行时（Runtime）** 的设计目标是**执行封装好的、具有明确输入和输出的、可重用的功能单元**。处理与特定自定义对象相关的、复杂的原子操作，正是运行时的用武之地。
*   **可测试与可复用：** 将复杂逻辑封装在运行时中，使得该逻辑单元可以被独立测试，并在图定义中被多次复用。

#### 3.2.5 高级指南 (为开发者)

##### 3.2.5.1 宏的转义与 `system.execute`

这是宏系统最强大的功能之一，允许您创建能与 LLM 进行深度交互的代理。

**场景**: 您想让 LLM 能够通过返回特定指令来操纵世界状态。

**步骤 1: 发送“转义”的指令给 LLM**

您需要向 LLM 发送包含 `{{...}}` 语法的提示，但又不希望它在发送前被引擎执行。诀窍是**在宏内部使用字符串字面量**。

```json
// 指令 1: 指导 LLM
{
  "runtime": "llm.default",
  "config": {
    "model": "gemini/gemini-1.5-pro",
    "contents": [{
      "role": "user",
      "content": "{{
        # 使用 f-string 和单引号来构建最终的 content
        # 这样 '{{...}}' 部分就会被当作普通文本
        f'''
        你是一个游戏助手。要将玩家的生命值设置为100，你应该返回：'{{ moment.player_hp = 100 }}'
        
        现在，请为我恢复玩家的能量。
        '''
      }}"
    }]
  }
}
```
引擎在预处理此指令时，会执行 f-string，生成一个包含 `{{ moment.player_hp = 100 }}` **文本**的字符串，然后将其作为 `content` 组装进消息列表，最后发送给 LLM。
**步骤 2: 执行 LLM 返回的指令**

假设上一步的 LLM 返回了字符串 `"{{ moment.player_energy = 100 }}"`。这个字符串现在存储在 `pipe.output` 中。它只是一个普通的字符串，不会自动执行。

要执行它，我们需要在下一个指令使用 `system.execute` 运行时。

```json
// 指令 2: 执行 LLM 的返回
{
  "runtime": "system.execute",
  "config": {
    // 1. 在预处理阶段，这个宏被执行，
    //    'code' 的值变成了字符串 "{{ moment.player_energy = 100 }}"
    "code": "{{ pipe.output }}",
  }
}
```
`system.execute` 运行时会接收到 `code` 字段的值，并对其进行**二次求值**，从而真正执行 `moment.player_energy = 100`，修改状态。

##### 3.2.5.2 动态定义函数与 `import`

您可以 `import` 任何库，或在 `lore` 或 `moment` 对象上动态创建函数，以构建可复用的逻辑或实现世界的自我演化。

```json
{
  "id": "teach_world_new_trick",
  "run": [{
    "runtime": "system.execute",
    "config": { "code": "{{
      import numpy as np # 导入外部库

      def calculate_weighted_average(values, weights):
          return np.average(values, weights=weights)

      if not hasattr(lore, 'utils'): lore.utils = {}
      lore.utils['weighted_avg'] = calculate_weighted_average
    }}"
  }}]
}
```
在后续节点中，您就可以直接调用 `{{ lore.utils.weighted_avg(...) }}`。

## 4. `system` 运行时参考

Hevno 引擎通过 `core_engine` 插件提供了一套功能强大、职责清晰的内置运行时，它们都位于 `system` 命名空间下。这套运行时是构建复杂图逻辑的基础工具。

### 4.1. 核心设计原则

*   **职责明确**: 每个运行时都封装一个有意义的、可复用的逻辑单元。
*   **LLM的纯粹性**: 保持 `llm.default` 的核心职责——与LLM API交互。所有的数据预处理（如格式化）和后处理（如解析）都应由独立的 `system` 运行时完成。
*   **通用性与可组合性**: 运行时被设计为通用的“瑞士军刀”，能够通过组合解决广泛的问题。
*   **优先使用宏**: 对于简单的状态读写操作（如 `moment.player.hp` 的读取和修改），应鼓励直接使用宏。

### 4.2. `system.io` (输入/输出)

#### 4.2.1. `system.io.input`
*   **摘要**: 将一个静态或动态生成的值注入到节点管道中。这是节点内部数据处理流程最基础的“数据源”。
*   **配置**:
    *   `value` (any, 必需): 要注入的值，支持宏。
*   **输出**: `{"output": <value 求值后的结果>}`.
*   **示例**:
    ```json
    {
      "runtime": "system.io.input",
      "config": {
        "value": "{{ f'当前玩家是 {moment.player_name}' }}"
      }
    }
    ```

#### 4.2.2. `system.io.log`
*   **摘要**: 向后端日志系统输出一条消息，用于调试图执行流程。
*   **配置**:
    *   `message` (string, 必需): 要记录的日志消息，支持宏。
    *   `level` (string, 可选): 日志级别，可选 `"debug"`, `"info"`, `"warning"`, `"error"`, `"critical"`。默认为 `"info"`。
*   **输出**: `{}` (纯副作用)。
*   **示例**:
    ```json
    {
      "runtime": "system.io.log",
      "config": {
        "message": "Debug: Player HP is {{ moment.player_hp }} before combat.",
        "level": "debug"
      }
    }
    ```

### 4.3. `system.data` (数据处理)

#### 4.3.1. `system.data.format`
*   **摘要**: 将列表或字典数据源格式化为单一字符串。
*   **配置**:
    *   `items` (list | dict, 必需): 要格式化的数据源。
    *   `template` (string, 必需): 格式化模板。`items` 为列表时，可使用 `{item}`；为字典时，可使用 `{key}` 和 `{value}`。
    *   `joiner` (string, 可选): 连接符，默认为 `"\n"`。
*   **输出**: `{"output": <格式化后的最终字符串>}`.
*   **示例**:
    ```json
    {
      "runtime": "system.data.format",
      "config": {
        "items": "{{ nodes.get_recent_events.output }}",
        "template": "- {item.content} (Source: {item.source})",
        "joiner": "\\n"
      }
    }
    ```

#### 4.3.2. `system.data.parse`
*   **摘要**: 将字符串（如LLM的输出）解析为结构化的数据对象（JSON/XML）。
*   **配置**:
    *   `text` (string, 必需): 待解析的字符串。
    *   `format` (string, 必需): 解析格式，支持 `"json"` 和 `"xml"`。
    *   `selector` (string, 可选): 仅 `format` 为 `"xml"` 时使用，一个简化的类XPath选择器。
    *   `strict` (boolean, 可选): 是否启用严格模式，默认为 `false`。
*   **输出**: 成功时为 `{"output": <解析后的对象>}`，失败（非严格模式）时为 `{"output": {"error": "..."}}`。
*   **示例**:
    ```json
    {
      "runtime": "system.data.parse",
      "config": {
        "text": "{{ pipe.output }}",
        "format": "json"
      }
    }
    ```

#### 4.3.3. `system.data.regex`
*   **摘要**: 对输入文本执行正则表达式匹配，并提取内容。
*   **配置**:
    *   `text` (string, 必需): 源文本。
    *   `pattern` (string, 必需): 正则表达式模式，支持命名捕获组。
    *   `mode` (string, 可选): 匹配模式，可选 `"search"` (默认) 或 `"find_all"`。
*   **输出**: `{"output": <匹配结果>}` (字典、字符串、列表或 `null`)。
*   **示例**:
    ```json
    {
      "runtime": "system.data.regex",
      "config": {
        "text": "{{ pipe.output }}",
        "pattern": "<thinking>(?P<thought>.+?)</thinking>"
      }
    }
    ```

### 4.4. `system.flow` (流程控制)

#### 4.4.1. `system.flow.call`
*   **摘要**: 调用并执行一个可复用的子图。
*   **配置**:
    *   `graph` (string, 必需): 要调用的子图名称。
    *   `using` (dict, 可选): 传递给子图的输入。
*   **输出**: `{"output": <子图所有最终节点状态的字典>}`.
*   **示例**:
    ```json
    {
      "runtime": "system.flow.call",
      "config": {
        "graph": "process_damage",
        "using": {
          "damage_input": "{{ run.triggering_input.damage_amount }}"
        }
      }
    }
    ```

#### 4.4.2. `system.flow.map`
*   **摘要**: 对一个列表进行并行迭代，为每个元素执行一次子图。
*   **配置**:
    *   `list` (list, 必需): 要迭代的列表。
    *   `graph` (string, 必需): 为每个列表项执行的子图名称。
    *   `using` (dict, 可选): 传递给子图的输入，上下文中额外包含 `source.item` 和 `source.index`。
    *   `collect` (any, 可选): 一个宏，用于从每次子图运行结果中聚合数据。
*   **输出**: `{"output": <包含所有子图运行结果的列表>}`.
*   **示例**:
    ```json
    {
      "runtime": "system.flow.map",
      "config": {
        "list": "{{ moment.enemies_in_area }}",
        "graph": "calculate_threat",
        "using": { "enemy_data": "{{ source.item }}" },
        "collect": "{{ nodes.threat_assessment.output }}"
      }
    }
    ```

### 4.5. `system.execute` (高级)

*   **摘要**: 对一个字符串形式的宏代码进行二次求值和执行。
*   **目的**: 作为宏系统的终极“逃生舱口”，用于实现标准运行时无法覆盖的动态逻辑（如执行由LLM生成的代码）。
*   **配置**:
    *   `code` (string, 必需): 包含宏代码的字符串。
*   **输出**: `{"output": <code字段执行后的结果>}`.

## 5. 插件生态系统：动态扩展世界

Hevno Engine 的核心力量来自于其动态、可插拔的插件系统。我们相信，最好的功能来自于社区，因此整个架构被设计为允许任何人轻松地创建、分享和使用插件来扩展引擎的能力。您甚至可以替换掉核心插件，以完全定制您的 Hevno 体验。

### 5.1 声明式插件管理 (`hevno.json`)

我们摒弃了手动管理 `plugins` 文件夹的传统方式，转而采用一种更现代、更健壮的**声明式管理**。您项目所需的所有插件都定义在一个位于项目根目录的 `hevno.json` 文件中。这个文件是您项目插件依赖的**唯一事实来源**。

一个专门的命令行工具 `hevno` 会读取这个文件，并自动完成插件的下载、安装和更新。

**`hevno.json` 格式约定:**

```json
{
  "plugins": {
    "core_engine": {
      "source": "local"
    },
    "stable-dice-roller": {
      "source": "git",
      "url": "https://github.com/Community/hevno-dice-roller",
      "ref": "v1.2.0"
    },
    "my-dev-plugin": {
      "source": "git",
      "url": "https://github.com/YourName/my-plugin-dev",
      "ref": "main",
      "strategy": "latest"
    }
  }
}
```

*   **`plugins`**: 插件字典。
    *   **键 (`core_engine`, `stable-dice-roller`)**: 插件的唯一标识符。这将是它在 `plugins/` 目录下的文件夹名称。
    *   **值 (插件配置对象)**:
        *   `source`: 定义插件来源。
            *   `"local"`: 表示这是一个项目本地的核心插件，由主仓库管理，插件管理器会跳过它。
            *   `"git"`: 表示这是一个需要从远程 Git 仓库下载的第三方插件。
        *   `url` (当 `source` 为 `git` 时): Git 仓库的 HTTPS URL。
        *   `ref` (当 `source` 为 `git` 时): 指定要使用的 Git 版本。可以是**分支名** (如 `main`)、**标签** (如 `v1.2.0`) 或精确的**提交哈希**。
        *   `subdirectory` (可选): 如果插件的代码位于仓库的某个子目录中，在此处指定其路径。
        *   `strategy` (可选): 定义插件的更新策略。
            *   `"pin"` (默认): **固定版本**。插件会被固定在 `ref` 指定的提交上。`sync` 命令如果发现插件已存在，会跳过它。这是用于生产和稳定依赖的最佳实践。
            *   `"latest"`: **追踪最新**。`sync` 命令会总是删除本地的旧版本，并从 `ref` 指定的分支（如 `main`）重新下载最新版本。这非常适合在开发阶段追踪一个活跃开发的插件。

### 5.2 插件管理工作流 (`hevno` CLI)

Hevno Engine 自带一个命令行工具，让插件管理变得轻而易举。

#### 首次设置项目

新开发者克隆您的项目后，只需几个简单步骤即可启动：

```bash
# 1. 克隆仓库
git clone https://github.com/Niurouxing/TheWanderingHevno.git
cd TheWanderingHevno

# 2. 安装项目依赖（包括命令行工具）
pip install -e ".[dev]"

# 3. 同步插件
#    这个命令会读取 hevno.json 并下载所有 Git 插件
hevno plugins sync

# 4. 启动应用
uvicorn backend.main:app --reload
```

#### 管理插件

*   **添加一个稳定的、固定版本的插件:**
    ```bash
    # `ref` 可以是标签或提交哈希
    hevno plugins add https://github.com/some-dev/hevno-weather-plugin --ref v1.0.0
    ```

*   **添加一个开发中的、需要追踪最新版本的插件:**
    ```bash
    # 使用 --track-latest 标志（注意：此功能在示例 cli.py 中未实现，但文档可以先行）
    hevno plugins add https://github.com/YourName/my-plugin-dev --ref main --track-latest
    ```

*   **移除插件:**
    ```bash
    hevno plugins remove hevno-weather-plugin
    ```

*   **更新插件:**
    *   对于**固定版本 (`pin`)**的插件: 在 `hevno.json` 中，手动将 `ref` 修改为新的版本号或提交哈希，然后运行 `hevno plugins sync`。
    *   对于**追踪最新 (`latest`)**的插件: 只需运行 `hevno plugins sync`，管理器就会自动获取指定分支的最新代码。

### 5.3 插件开发约定

想要为您自己的 Hevno 项目或为社区贡献一个插件吗？遵循以下简单的约定即可：

1.  **目录结构**: 每个插件都是一个独立的 Python 包，位于 `plugins/` 目录下。一个标准的插件包含：
    ```
    plugins/
    └── my-awesome-plugin/
        ├── __init__.py         # 插件入口点，必须包含 register_plugin 函数
        ├── manifest.json       # 插件元数据（名称、优先级等）
        ├── ... (service.py, router.py, etc.)
    ```

2.  **`manifest.json`**: 这是插件的“身份证”，定义了其元数据。
    ```json
    {
        "name": "my-awesome-plugin",
        "version": "1.0.0",
        "description": "A brief description of what this plugin does.",
        "author": "Your Name",
        "priority": 50,
        "dependencies": ["core_engine"] 
    }
    ```
    *   `priority`: 加载优先级，数字越小越先加载。核心插件有预设的优先级（如 `core_logging` 是 -100，`core_engine` 是 50）。

3.  **入口点 `__init__.py`**: 每个插件必须提供一个 `register_plugin` 函数，这是引擎与插件通信的桥梁。
    ```python
    # plugins/my-awesome-plugin/__init__.py
    from backend.core.contracts import Container, HookManager

    def register_plugin(container: Container, hook_manager: HookManager):
        # 在这里...
        # 1. 向容器注册你的服务工厂
        # container.register("my_service", _create_my_service)
        
        # 2. 向钩子管理器注册你的钩子实现
        # hook_manager.add_implementation("collect_runtimes", provide_my_runtime)
    ```

通过这个系统，Hevno 的世界可以像乐高积木一样被无限组合和扩展。我们期待看到您创造的精彩插件！



## 6. 统一资源 API：读写世界状态的唯一入口

> **"One API to rule them all."**

为了实现终极的简洁性和可扩展性，Hevno 摒弃了为每个数据类型（图、Codex、Memoria）设计专用API的传统模式。取而代之的是，我们提供了一套统一的、路径驱动的、功能强大的API端点。这套API是您作为创作者与沙盒内任何数据进行交互的**唯一**方式。

无论未来增加多少种新的数据型插件，您都将使用相同的API来读写它们的数据。

### 6.1 核心理念

*   **路径驱动**: 任何沙盒内的数据都可以通过一个类似文件路径的字符串（如 `lore/codices/main/entries/0`）来唯一定位。
*   **批量操作**: 所有的读写操作都被设计为批处理模式，允许您在一次网络请求中执行多个操作，保证了原子性和效率。
*   **低级工具**: 此API被设计为一个精确、强大的“开发者工具”。它不包含复杂的业务逻辑（例如自动生成ID），而是忠实地执行您下达的数据操作指令。需要业务逻辑的创建操作（如添加一条新记忆），应通过执行包含特定运行时的图来完成。

### 6.2 批量读取数据: `resource:query`

*   **端点**: `POST /api/sandboxes/{sandbox_id}/resource:query`
*   **功能**: 在一次请求中，根据一个或多个路径，查询沙盒内的任意数据。

**请求体:**
```json
{
  "paths": [
    "moment/player/hp", 
    "lore/graphs/main",
    "definition/initial_lore/codices/world_rules"
  ]
}
```

**响应体:**
```json
{
  "results": {
    "moment/player/hp": 100,
    "lore/graphs/main": { "nodes": [ /*...*/ ] },
    "definition/initial_lore/codices/world_rules": { "entries": [ /*...*/ ] }
  }
}
```
*如果某个路径不存在，其在`results`中的值将为`null`。*

### 6.3 批量修改数据: `resource:mutate`

*   **端点**: `POST /api/sandboxes/{sandbox_id}/resource:mutate`
*   **功能**: 在一次请求中，原子性地执行一个或多个数据修改操作。如果其中任何一个操作失败，所有操作都将回滚。

**请求体:**
请求体包含一个`mutations`列表，每个对象描述一个独立的修改操作。
```json
{
  "mutations": [
    {
      "type": "UPSERT",
      "path": "lore/codices/character_background/description",
      "value": "一个新的背景描述。"
    },
    {
      "type": "LIST_APPEND",
      "path": "lore/graphs/main/nodes",
      "value": { "id": "new_node", "run": [] }
    },
    {
      "type": "DELETE",
      "path": "moment/active_effects/0"
    }
  ]
}
```

**Mutation 对象详解**:
*   `path` (`string`): 必需。从根（`definition`, `lore`, `moment`）开始的完整资源路径。
*   `type` (`string`): 必需。操作类型：
    *   `UPSERT`: 更新或插入。如果路径不存在，会尝试创建。
    *   `DELETE`: 删除路径指向的资源。
    *   `LIST_APPEND`: 向路径指向的列表末尾追加`value`。
*   `value` (`any`): `UPSERT` 和 `LIST_APPEND` 时必需。
*   `mutation_mode` (`string`): 可选，仅当`path`以`moment/`开头时有效。
    *   `"DIRECT"` (默认): **直接修改**当前快照，不创建新历史。用于编辑器调试。
    *   `"SNAPSHOT"`: **创建新快照**来记录本次修改。用于需要被记录的用户行为（如GM操作）。


## `core_codex` 插件文档

### 1. 简介：构建动态的、分层的知识库

`core_codex` 是 Hevno 引擎中的一个关键插件，它提供了一个强大、声明式、基于优先级的知识库系统。与简单的文本拼接不同，Codex 允许您将知识分解为独立的、可被条件性激活的“条目”（Entries），并根据上下文智能地将它们组合成连贯的文本。

`core_codex` 的核心能力是响应动态变化。它通过**`lore` 和 `moment` 的分层设计**，让知识既有稳定的基石，又能灵活应对瞬息万变的状况：

*   **基础知识 (`lore.codices`)**: 定义世界观、角色背景、核心规则等不常变化的基础信息。
*   **情境知识 (`moment.codices`)**: 动态注入临时的、与当前情境相关的知识，例如“角色当前正处于中毒状态”、“场景中刚刚下过雨”等。这些临时知识可以在逻辑上**覆盖或增强**基础知识。
*   **递归激活**: 一个知识条目被激活后，其产生的内容可以**再次触发**其他相关条目的激活，形成逻辑链，构建出富有深度和细节的最终文本。

Codex 是构建能感知环境、拥有深度背景的 AI 代理和动态世界描述的利器。

### 2. 核心概念：Lore 与 Moment 中的知识分层

遵循 Hevno“状态先行”的哲学，Codex 的所有定义都驻留在 `lore` 和 `moment` 作用域下一个名为 `codices` 的结构里。当 `codex.invoke` 运行时被调用时，它会智能地将这两个来源的知识合并。

#### 2.1 知识库集合 (`Codex Collection`)

`lore.codices` 和 `moment.codices` 都是字典，其 `key` 为知识库的名称，`value` 为一个 `Codex` 对象。

```json
// 在 lore 中定义基础知识
"lore": {
  "codices": {
    "character_background_gandalf": {
      "entries": [
        {"id": "base_identity", "priority": 100, "content": "甘道夫是一位埃斯塔力，是派来中土帮助对抗索伦的迈雅。"},
        {"id": "appearance", "priority": 90, "content": "他通常以一个戴着尖顶蓝帽、身穿灰色斗篷、手持法杖的老人形象出现。"}
      ]
    }
  }
},
// 在 moment 中定义临时状态
"moment": {
  "codices": {
    "character_background_gandalf": {
      "entries": [
        // 这个条目会覆盖 lore 中的同名条目
        {"id": "appearance", "priority": 95, "content": "他看起来筋疲力尽，灰色的斗篷上沾满了泥土。"},
        // 这是一个全新的、只在当前 moment 存在的条目
        {"id": "current_action", "priority": 120, "content": "他正在低声吟唱咒语，法杖顶端发出微光。"}
      ]
    }
  }
}
```
**合并规则**:
1.  **Codex 级别**: `moment` 中的 `Codex` 会与 `lore` 中同名的 `Codex` 合并。如果 `lore` 中没有，则直接添加。
2.  **Entry 级别**: 在合并的 `Codex` 内部，`moment` 中的条目（`Entry`）会根据其 `id` **覆盖** `lore` 中的同名条目。
3.  **最终结果**: `codex.invoke` 总是工作在合并后的知识库上。

#### 2.2 知识条目 (`Codex Entry`)

每个知识条目都是一个独立的、可被逻辑控制的知识片段。

```json
{
  "id": "unique_entry_id",
  "content": "这是知识的具体内容，支持宏。{{ moment.player_name }}",
  "priority": 100,
  "trigger_mode": "on_keyword",
  "keywords": ["战斗", "危险"],
  "is_enabled": "{{ moment.player.in_combat }}"
}
```
*   `content`: 知识的文本内容，支持宏。
*   `priority`: 整数，决定了在最终输出时，这个条目的排序位置。数字越大，越靠前。
*   `trigger_mode`: 激活模式。`always_on`（总是激活）或 `on_keyword`（当 `source` 文本包含 `keywords` 之一时激活）。
*   `keywords`: 一个字符串列表，用于 `on_keyword` 模式。
*   `is_enabled`: 一个布尔值或返回布尔值的宏，用于在最开始就决定该条目是否参与激活判断。

### 3. 使用指南

`core_codex` 插件主要通过两种方式与系统交互：

#### 3.1 `codex.invoke` 运行时

这是在**图执行期间**使用Codex的核心方式。它负责在运行时动态地收集、合并和渲染知识条目，生成最终的文本。

*   **功能**: 从 `lore` 和 `moment` 中收集并合并知识库，根据配置激活相关条目，最后将它们按优先级排序组合成一段单一的文本。
*   **配置参数**:
    *   `from` (必需): `List[Dict]` - 一个列表，定义了要从哪些知识库中、使用什么源文本来激活条目。
        *   `codex`: `string` - 要扫描的知识库名称。
        *   `source`: `string` (可选) - 一个包含宏的字符串，其求值结果将作为触发 `on_keyword` 模式的源文本。
    *   `recursion_enabled` (可选): `bool` - 是否允许已激活条目的内容再次触发新的条目。默认为 `false`。
    *   `debug` (可选): `bool` - 是否返回详细的调试信息。

*   **示例**:
    ```json
    {
      "id": "generate_npc_description",
      "run": [{
        "runtime": "codex.invoke",
        "config": {
          "from": [
            {
              "codex": "world_lore",
              "source": "{{ run.trigger_input.user_query }}"
            },
            {
              "codex": "npc_status_gandalf"
            }
          ],
          "recursion_enabled": true
        }
      }]
    }
    ```
    **执行流程**:
    1.  从 `lore` 和 `moment` 中合并所有 `codex` 定义。
    2.  扫描 `world_lore`，使用 `run.trigger_input.user_query` 的内容来匹配 `on_keyword` 条目。
    3.  扫描 `npc_status_gandalf`，由于没有 `source`，只会激活其中的 `always_on` 条目。
    4.  所有被激活的条目放入一个池中。
    5.  由于 `recursion_enabled` 为 `true`，引擎会按优先级渲染条目，并用其输出内容递归地去激活池中其他尚未渲染的条目。
    6.  所有最终被渲染的条目内容，按优先级从高到低拼接成最终的 `output`。

 #### 3.2 通过统一资源API进行编辑
 
 作为创作者，您可以使用标准的**统一资源API**来读取和写入Codex的数据结构。由于Codex条目不包含复杂的、需要后端生成的字段，您可以非常直观地进行CRUD操作。
 
 **示例：通过`:mutate` API添加一个新的Codex条目**
 ```http
 POST /api/sandboxes/{sandbox_id}/resource:mutate
 
 {
   "mutations": [
     {
       "type": "LIST_APPEND",
       "path": "lore/codices/world_history/entries",
       "value": {
         "id": "age_of_dragons",
         "content": "在远古时期，巨龙统治着天空。",
         "priority": 90
       }
     }
   ]
 }
 ```
 *关于统一资源API的详细信息，请参阅本文档的第6章。*

## `core_memoria` 插件文档

### 1. 简介：赋予世界记忆与反思的能力

`core_memoria` 是 Hevno 引擎中的一个核心插件，其使命是为您的 AI 世界提供一个强大、持久且可查询的记忆系统。它不仅仅是一个日志记录器，更是一个能让 AI 代理和世界本身从历史中学习、演化和反思的动态知识库。

通过 `core_memoria`，您可以轻松实现：
*   **短期记忆**：为 LLM 提供最近发生的事件，确保对话和行为的连贯性。
*   **长期反思**：自动将一系列零散事件**综合**成更高层次的“里程碑”或“章节总结”。
*   **情境感知**：根据标签或层级检索相关的历史记忆，让 AI 能够将新旧信息联系起来。
*   **角色心路历程**：为每个角色或主题维护独立的“记忆回廊”，追踪他们的成长与变化。

`core_memoria` 将离散的事件流，编织成一张富有深度和因果关系的智慧之网，是构建真正动态、可交互 AI 世界的关键。

### 2. 核心概念：`moment` 状态中的“记忆宫殿”

遵循 Hevno“状态先行”的哲学，所有记忆的最终真相都驻留在**瞬时状态层 (`moment`)** 中一个名为 `memoria` 的结构里。这确保了记忆是与特定时间点绑定的，当游戏读档时，记忆也会回滚到当时的状态。

#### 2.1 记忆回廊 (Memory Stream)

“记忆宫殿” (`moment.memoria`) 是一个字典，它的每一个键都代表一个独立的“记忆回廊”（Stream）。这允许您为不同的实体或主题（如主线故事、特定角色的思想、某个组织的历史）分别记录记忆。

```json
// moment.memoria 的结构
"moment": {
  "memoria": {
    "__global_sequence__": 12, // 内部使用的全局序列号
    "main_story": { /* ... 主线故事的 MemoryStream 对象 ... */ },
    "character_thoughts_humphrey": { /* ... 汉弗莱爵士心路历程的 MemoryStream 对象 ... */ }
  }
}
```

#### 2.2 记忆条目 (Memory Entry)

每个“记忆回廊”中都包含一个按时间顺序排列的 `entries` 列表，列表中的每一项都是一个结构化的“记忆条目”。

```json
// 一个 MemoryEntry 对象的示例
{
  "id": "a9b1c2d3-...",
  "sequence_id": 5, // 全局唯一的、严格递增的因果序列号
  "level": "event",
  "tags": ["combat", "goblin"],
  "content": "玩家在森林入口遭遇并击败了三只哥布林。"
}
```

*   `sequence_id`：**这至关重要**。它是一个在**所有**记忆流中都单调递增的整数，精确地记录了事件在游戏世界内的**因果顺序**。它不受现实时间影响，确保了读档和回滚的确定性。
*   `level`: 记忆的层级，如 `"event"`, `"dialogue"`, `"milestone"`, `"thought"`。用于分层检索。 **当记录对话时，约定将角色（如 `"user"` 或 `"model"`）作为此字段的值**。
*   `tags`: 一个关键词列表，用于实现强大的相关性检索。
*   `content`: 记忆的文本内容。

### 3. 使用指南：运行时 (Runtimes)

`core_memoria` 插件提供了一套简洁、强大的运行时（积木块），让您可以轻松地与记忆系统交互，而无需编写复杂的代码。

#### 3.1 `memoria.add`：烙印新记忆

这是将一个新事件转化为持久记忆的核心指令。

*   **功能**：向指定的记忆流中添加一条新的条目。
*   **配置参数**：
    *   `stream` (必需): `string` - 要添加到的记忆流的名称。
    *   `content` (必需): `any` - 记忆的内容。宏求值后会被转换为字符串。
    *   `level` (可选): `string` - 该条目的层级，默认为 `"event"`。**约定**: 当记录对话时，推荐将角色（如 `"user"`, `"model"`）作为此字段的值，以便于后续的格式化查询。
    *   `tags` (可选): `List[string]` - 与该条目关联的标签列表。

*   **示例**：
    ```json
    {
      "id": "record_user_chat",
      "run": [{
        "runtime": "memoria.add",
        "config": {
          "stream": "chat_history",
          "level": "user",
          "tags": ["dialogue"],
          "content": "{{ run.triggering_input.user_message }}"
        }
      }]
    }
    ```

#### 3.2 `memoria.query`：检索历史片段

这个运行时允许您根据多种条件，以声明式的方式从记忆流中检索信息。

*   **功能**：查询一个记忆流并返回一个符合条件的记忆条目列表。
*   **配置参数**：
    *   `stream` (必需): `string` - 要查询的记忆流的名称。
    *   `latest` (可选): `int` - 只返回最新的 N 条记忆。
    *   `levels` (可选): `List[string]` - 只返回 `level` 在此列表中的条目。
    *   `tags` (可选): `List[string]` - 返回至少包含**一个**指定标签的条目。
    *   `order` (可选): `"ascending"` 或 `"descending"` - 返回结果的排序顺序，基于 `sequence_id`。默认为 `"ascending"`。
    *   `format` (可选): `"raw_entries"` 或 `"message_list"` - 定义输出格式。
        *   `"raw_entries"` (默认): 返回完整的 `MemoryEntry` 对象列表。
        *   `"message_list"`: **(新)** 将 `level` 为 `"user"` 或 `"model"` 的条目转换为 `{"role": ..., "content": ...}` 格式的消息列表，专为 `llm.default` 设计。

#### 3.3 为 LLM 提供对话历史 (协同示例)

通过 `memoria.query` 新增的 `format: "message_list"` 模式，结合重构后的 `llm.default` 运行时，可以极其优雅地构建包含对话历史的 LLM 请求。这避免了在宏中手动拼接字符串的复杂操作。

以下是一个标准的“聊天机器人”图，展示了这两个插件如何协同工作：

```json
[
    {
      "id": "get_chat_history",
      "run": [{
        "runtime": "memoria.query",
        "config": {
          "stream": "chat_history",
          "latest": 10,
          "format": "message_list" 
        }
      }]
    },
    {
      "id": "generate_response",
      "depends_on": ["get_chat_history"],
      "run": [{
        "runtime": "llm.default",
        "config": {
          "model": "gemini/gemini-1.5-pro",
          "contents": [
            {
              "type": "MESSAGE_PART",
              "role": "system",
              "content": "你是一个乐于助人的AI助手。"
            },
            {
              "type": "INJECT_MESSAGES",
              "source": "{{ nodes.get_chat_history.output }}"
            },
            {
              "type": "MESSAGE_PART",
              "role": "user",
              "content": "{{ run.triggering_input.user_message }}"
            }
          ]
        }
      }]
    }
]
```
**执行流程**:
1.  `get_chat_history` 节点运行，`memoria.query` 检索最新的10条对话记忆，并直接将它们格式化为 `[{"role": "user", "content": "..."}, {"role": "model", "content": "..."}]` 形式的列表。
2.  `generate_response` 节点运行，`llm.default` 运行时处理其 `contents` 列表。
3.  `INJECT_MESSAGES` 指令将上一个节点的结果无缝地展开并注入到消息列表中。
4.  最终，一个结构完整、包含系统提示、历史记录和当前用户消息的请求被发送给 LLM。

### 4. 自动化功能：记忆的自动综合

`core_memoria` 最强大的功能之一是它能够自动进行“记忆综合”——即在后台调用 LLM，将一系列零散的事件总结成一个更高层次的“里程碑”或“章节概要”。

这个过程是**完全异步和非阻塞的**，不会影响主图的执行性能。

#### 4.1 如何配置

您可以通过修改 `moment.memoria` 中特定流的 `config` 对象来启用和配置此功能。这可以在图的任何地方，使用 `system.execute` 来完成。

```json
// 使用 system.execute 在游戏开始时配置 main_story 流
{
  "id": "setup_memory_synthesis",
  "run": [{
    "runtime": "system.execute",
    "config": {
      "code": "{{
        # 确保 memoria 和 main_story 流存在
        if 'memoria' not in moment: moment.memoria = {}
        if 'main_story' not in moment.memoria: moment.memoria.main_story = {}
        if 'config' not in moment.memoria.main_story: moment.memoria.main_story.config = {}

        # 定义自动综合的行为
        moment.memoria.main_story.config.auto_synthesis = {
            'enabled': True,
            'trigger_count': 10,  // 每发生 10 次事件
            'level': 'milestone', // 就生成一个“里程碑”级别的总结
            'model': 'gemini/gemini-1.5-flash',
            'prompt': '''
As a master storyteller, synthesize the following sequence of events into a single, cohesive paragraph that captures the key developments and overall tone.

Events:
{events_text}
'''
        }
      }}"
    }
  }]
}
```

#### 4.2 工作流程

1.  当 `memoria.add` 被调用时，它会为该流的内部计数器加一。
2.  如果计数器达到了 `trigger_count`，`memoria.add` 会将最近的 N 条事件连同您的配置（`model`, `prompt`）一起，打包成一个后台任务。
3.  主图执行继续，**不受任何影响**。
4.  在后台，任务管理器会执行该任务：调用 LLM，生成总结。
5.  任务完成后，它会将生成的总结作为一个**待处理事件**放入一个特殊的队列中。
6.  在**下一次**图执行开始之前，引擎会自动检查该队列，并将所有已完成的总结作为新的记忆条目，原子性地添加到对应的记忆流中。

这种设计确保了系统的响应速度和状态的一致性。

### 5. 高级用法：通过宏直接访问

对于需要高度定制化逻辑的开发者，`moment.memoria` 的完整数据结构对宏是完全透明的。您可以随时使用 `system.execute` 来编写任意 Python 代码，进行标准运行时无法完成的复杂查询和分析。

```json
// 示例：计算“汉弗莱”的总结中，“背叛”一词在过去24小时内的出现频率
{
  "runtime": "system.execute",
  "config": {
    "code": "{{
      # ... 复杂的、定制化的 Python 逻辑，操作 moment.memoria ...
    }}"
  }
}
```

这种灵活性确保了 `core_memoria` 既是一个易于上手的工具，也是一个功能没有上限的强大平台。

### 6. 高级用法：通过API直接操作记忆数据

> **警告**: 本章节内容面向高级用户和开发者。直接通过API操作`moment.memoria`结构是一个强大的“紧急出口”，但如果使用不当，可能会破坏记忆的因果链，导致不可预期的行为。请在充分理解其底层机制和风险后再使用。

#### 6.1 何时应该使用API直接编辑？

在99%的情况下，您应该使用`memoria.add`, `memoria.query`等运行时来与记忆系统交互。但是，在以下特定场景中，直接API编辑可能是必要的：

*   **数据修复**: 当一个记忆条目的内容出现错误，需要手动修正时。
*   **调试**: 在调试复杂的图时，需要临时修改或插入一个记忆条目来模拟特定状态。
*   **数据迁移**: 从外部系统导入历史数据，需要手动指定`id`和`sequence_id`。
*   **创作者工具**: 构建一个允许创作者完全控制记忆时间线的外部编辑器或GM工具。

#### 6.2 底层风险：`sequence_id` 的重要性

`memoria`系统的核心是`moment.memoria.__global_sequence__`计数器和每个记忆条目（`MemoryEntry`）的`sequence_id`字段。这个`sequence_id`是一个在**整个沙盒的所有记忆流中都严格单调递增**的整数。它不是时间戳，而是事件**因果关系**的绝对记录。

**风险在于**：
*   **手动创建条目**: 如果您通过API的`LIST_APPEND`操作添加一个新条目，您**必须**手动提供一个唯一的`id`和一个**正确**的`sequence_id`。如果您提供的`sequence_id`重复，或者破坏了递增顺序，可能会导致`memoria.query`等运行时的排序和筛选功能出现严重错误。
*   **忘记更新全局计数器**: 当您手动创建一个`sequence_id`为`101`的条目后，您还**必须**确保`moment.memoria.__global_sequence__`的值至少为`101`。否则，下一次`memoria.add`运行时可能会再次生成一个`sequence_id`为`101`的条目，造成冲突。

**黄金法则**：当通过API进行写操作时，您是在扮演“系统”的角色，您有责任维护数据结构的完整性和一致性。

#### 6.3 API用法示例

所有操作都通过项目的**统一资源API**完成。关于此API的完整文档，请参阅主文档的第6章。

##### 示例1：修正一个已存在条目的内容（安全操作）

这是最常见的用例。假设您发现第一条记忆的内容有错别字。

```http
POST /api/sandboxes/{sandbox_id}/resource:mutate

{
  "mutations": [
    {
      "type": "UPSERT",
      "path": "moment/memoria/main_story/entries/0/content",
      "value": "玩家找到的其实是一把[修正后]的钥匙。",
      "mutation_mode": "DIRECT"
    }
  ]
}
```
*   **风险**: 低。`UPSERT`一个叶子节点的值是相对安全的操作。

##### 示例2：手动添加一个完整的记忆条目（高风险操作）

假设您需要修复一段丢失的历史，需要在`sequence_id`为5和6之间插入一个事件。

1.  **第一步: 查询当前状态**
    首先，查询当前的全局序列号，以决定新条目的`sequence_id`应该是什么。
    `POST /api/sandboxes/{id}/resource:query`
    Body: `{"paths": ["moment/memoria/__global_sequence__"]}`
    假设返回结果是`100`。

2.  **第二步: 执行批量修改**
    您决定将新条目的`sequence_id`设为`101`，并同时更新全局计数器。这必须在**一次**`:mutate`调用中完成以保证原子性。

```http
POST /api/sandboxes/{sandbox_id}/resource:mutate

{
  "mutations": [
    {
      "type": "LIST_APPEND",
      "path": "moment/memoria/main_story/entries",
      "value": {
        "id": "manual-fix-a9b1c2d3",
        "sequence_id": 101, 
        "level": "event",
        "tags": ["manual_edit"],
        "content": "一个被手动插入的修复性事件。",
        "created_at": "2025-08-12T05:10:07Z" 
      },
      "mutation_mode": "DIRECT"
    },
    {
      "type": "UPSERT",
      "path": "moment/memoria/__global_sequence__",
      "value": 101,
      "mutation_mode": "DIRECT"
    }
  ]
}```
*   **风险**: 高。您必须确保`sequence_id`的唯一性和正确性，并同步更新全局计数器。

##### 示例3：删除一个记忆条目（中风险操作）

删除一个条目可能会在记忆序列中造成一个“空缺”，但通常不会破坏后续条目的因果关系。

```http
POST /api/sandboxes/{sandbox_id}/resource:mutate

{
  "mutations": [
    {
      "type": "DELETE",
      "path": "moment/memoria/main_story/entries/5",
      "mutation_mode": "DIRECT"
    }
  ]
}
```
*   **风险**: 中。您需要精确知道要删除的条目的索引（在这个例子中是第6个条目，索引为5）。
```
```

### app.py
```
# backend/app.py
import logging
from contextlib import asynccontextmanager
from fastapi import FastAPI, APIRouter
from fastapi.middleware.cors import CORSMiddleware

from backend.container import Container
from backend.core.hooks import HookManager
from backend.core.loader import PluginLoader
from backend.core.tasks import BackgroundTaskManager


from plugins.core_remote_hooks.contracts import GlobalHookRegistryInterface

@asynccontextmanager
async def lifespan(app: FastAPI):
    # --- 启动阶段 ---
    container = Container()
    hook_manager = HookManager(container)

    # 1. 注册平台核心服务
    container.register("container", lambda: container)
    container.register("hook_manager", lambda: hook_manager)
    
    task_manager = BackgroundTaskManager(container)
    container.register("task_manager", lambda: task_manager, singleton=True)
    hook_manager.add_shared_context("task_manager", task_manager)

    # 2. 加载插件（同步注册）
    loader = PluginLoader(container, hook_manager)
    loader.load_plugins()
    
    logger = logging.getLogger(__name__)
    logger.info("--- FastAPI 应用组装 ---")

    # 3. 将核心服务附加到 app.state
    app.state.container = container
    hook_manager.add_shared_context("app", app)

    # --- 【核心修复】 ---
    # 4. 填充后端钩子到全局注册表 (在触发任何钩子之前)
    try:
        global_registry: GlobalHookRegistryInterface = container.resolve("global_hook_registry")
        backend_hooks = list(hook_manager._hooks.keys())
        global_registry.register_backend_hooks(backend_hooks)
        logger.info("Global hook registry populated with all backend hooks.")
    except ValueError:
        logger.warning(
            "Service 'global_hook_registry' not found. "
            "Assuming 'core_remote_hooks' is not installed. Full-stack hooks will be disabled."
        )

    # 5. 触发异步服务初始化钩子
    logger.info("正在为异步初始化触发 'services_post_register' 钩子...")
    await hook_manager.trigger('services_post_register')
    logger.info("异步服务初始化完成。")

    # 启动后台工作者
    task_manager.start()

    # 6. 平台核心负责收集并装配 API 路由
    logger.info("正在从所有插件收集 API 路由...")
    routers_to_add: list[APIRouter] = await hook_manager.filter("collect_api_routers", [])
    
    if routers_to_add:
        logger.info(f"已收集到 {len(routers_to_add)} 个路由。正在添加到应用中...")
        for router in routers_to_add:
            app.include_router(router)
            logger.debug(f"已添加路由: prefix='{router.prefix}', tags={router.tags}")
    else:
        logger.warning("未从插件中收集到任何 API 路由。")
    
    # 7. 触发最终启动完成钩子
    await hook_manager.trigger('app_startup_complete')
    
    logger.info("--- Hevno 引擎已就绪 ---")
    yield
    # --- 关闭阶段 ---
    logger.info("--- Hevno 引擎正在关闭 ---")
    
    await task_manager.stop()
    await hook_manager.trigger('app_shutdown')


def create_app() -> FastAPI:
    """应用工厂函数"""
    app = FastAPI(
        title="Hevno Engine (Plugin Architecture)",
        version="1.2.0",
        lifespan=lifespan
    )

    app.add_middleware(
        CORSMiddleware,
        allow_origins=["*"], 
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
    
    return app
```

### main.py
```
# backend/main.py

import uvicorn
import os
from dotenv import load_dotenv
load_dotenv()

from backend.app import create_app

# 调用工厂函数来获取完全配置好的应用实例
app = create_app()

@app.get("/")
def read_root():
    return {"message": "Hevno Engine (Plugin Architecture) is running!"}


if __name__ == "__main__":
    uvicorn.run(
        "backend.main:app",
        host=os.getenv("HOST", "0.0.0.0"),
        port=int(os.getenv("PORT", 8000)),
        reload=True,
        reload_dirs=["backend", "plugins"]
    )
```

### core/hooks.py
```
# backend/core/hooks.py
import asyncio
import logging
import inspect
from collections import defaultdict
from dataclasses import dataclass, field
from typing import Any, Callable, Dict, List, Awaitable, TypeVar, Optional

# 从平台核心导入最基础的接口
from backend.core.contracts import HookManager as HookManagerInterface, Container

# 从新插件的契约中导入核心模型和接口
from plugins.core_remote_hooks.contracts import (
    HookLocation,
    GlobalHookRegistryInterface,
    RemoteHookEmitterInterface
)


logger = logging.getLogger(__name__)

# 定义可被过滤的数据类型变量
T = TypeVar('T')

# 定义钩子函数的通用签名
HookCallable = Callable[..., Awaitable[Any]]

@dataclass(order=True)
class HookImplementation:
    """封装一个钩子实现及其元数据。"""
    priority: int
    func: HookCallable = field(compare=False)
    plugin_name: str = field(compare=False, default="<unknown>")


class HookManager(HookManagerInterface):
    """
    一个中心化的、上下文感知的服务，负责发现、注册和调度所有钩子实现。
    它能自动将上下文注入到钩子函数中，并支持向远端的智能路由。
    """
    def __init__(self, container: Container):
        self._hooks: Dict[str, List[HookImplementation]] = defaultdict(list)
        self._container = container
        self._shared_context: Dict[str, Any] = {
            "container": container,
            "hook_manager": self
        }
        
        # 【优化】为远程服务添加缓存属性
        self._global_registry: Optional[GlobalHookRegistryInterface] = None
        self._remote_emitter: Optional[RemoteHookEmitterInterface] = None
        self._remote_services_resolved: bool = False # 标志位，确保只解析一次

        logger.info("HookManager initialized and context-aware.")

    def add_shared_context(self, name: str, service: Any) -> None:
        """允许在启动过程中向钩子系统添加更多的共享服务。"""
        if name in self._shared_context:
            logger.warning(f"Overwriting shared context for hooks: '{name}'")
        self._shared_context[name] = service

    def _prepare_hook_args(
        self, 
        func: HookCallable, 
        call_context: Dict[str, Any],
        positional_data: Optional[Any] = None
    ) -> tuple[list, dict]:
        """智能地准备传递给钩子函数的参数。"""
        sig = inspect.signature(func)
        params = sig.parameters
        
        hook_kwargs = {}
        hook_args = []
        
        # 处理位置参数 (主要用于 filter 钩子)
        if positional_data is not None:
            # 假设 filter 的数据是第一个参数
            hook_args.append(positional_data)
        
        # 处理关键字参数
        for name, param in params.items():
            # 跳过已处理的位置参数
            if param.kind == param.POSITIONAL_ONLY or (param.kind == param.POSITIONAL_OR_KEYWORD and name in [p.name for p in params.values() if p.kind != param.KEYWORD_ONLY][:len(hook_args)]):
                continue
            
            if name in call_context:
                hook_kwargs[name] = call_context[name]

        return hook_args, hook_kwargs

    def add_implementation(
        self,
        hook_name: str,
        implementation: HookCallable,
        priority: int = 10,
        plugin_name: str = "<core>"
    ):
        """向管理器注册一个钩子实现。"""
        if not asyncio.iscoroutinefunction(implementation):
            raise TypeError(f"Hook implementation for '{hook_name}' must be an async function.")

        hook_impl = HookImplementation(priority=priority, func=implementation, plugin_name=plugin_name)
        self._hooks[hook_name].append(hook_impl)
        self._hooks[hook_name].sort() # 保持列表按优先级排序（从小到大）
        logger.debug(f"Registered hook '{hook_name}' from plugin '{plugin_name}' with priority {priority}.")


    async def trigger(self, hook_name: str, **kwargs: Any) -> None:
        """
        【重构 & 优化】触发一个“通知型”钩子。
        现在会查询全域路由表，并智能地决定是在本地执行、发送到远端，还是两者都做。
        远程服务采用懒加载模式，只在第一次调用时解析。
        """
        # --- 优化的智能路由逻辑 ---
        if not self._remote_services_resolved:
            try:
                # 尝试解析一次并缓存
                self._global_registry = self._container.resolve("global_hook_registry")
                self._remote_emitter = self._container.resolve("remote_hook_emitter")
                logger.info("Successfully resolved and cached remote hook services.")
            except ValueError:
                # 无法解析服务，说明插件未安装。我们将它们保持为 None。
                logger.info("Remote hook services not found. HookManager will operate in local-only mode.")
            finally:
                # 无论成功与否，都标记为已解析，不再重复尝试
                self._remote_services_resolved = True

        location = HookLocation.LOCAL # 默认是本地
        if self._global_registry:
            # 如果注册表已缓存，则查询位置
            location = self._global_registry.get_hook_location(hook_name)

        executed_locally = False

        # --- 本地执行逻辑 ---
        if location in [HookLocation.LOCAL, HookLocation.BOTH]:
            if hook_name in self._hooks:
                call_context = {**self._shared_context, **kwargs}
                implementations = self._hooks[hook_name]
                tasks = []
                for impl in implementations:
                    try:
                        _, prepared_kwargs = self._prepare_hook_args(impl.func, call_context)
                        tasks.append(impl.func(**prepared_kwargs))
                    except Exception as e:
                        logger.error(
                            f"Error preparing args for NOTIFICATION hook '{hook_name}' from plugin '{impl.plugin_name}': {e}",
                            exc_info=e
                        )

                results = await asyncio.gather(*tasks, return_exceptions=True)

                for i, result in enumerate(results):
                    if isinstance(result, Exception):
                        impl = implementations[i]
                        logger.error(
                            f"Error in NOTIFICATION hook '{hook_name}' from plugin '{impl.plugin_name}': {result}",
                            exc_info=result
                        )
                executed_locally = True

        # --- 远程执行逻辑 ---
        if self._remote_emitter and location in [HookLocation.REMOTE, HookLocation.BOTH]:
            await self._remote_emitter.emit(hook_name, kwargs)

        # --- 未找到处理程序的警告 ---
        if not executed_locally and location == HookLocation.UNKNOWN:
            logger.warning(
                f"Triggered hook '{hook_name}' has no known implementation in backend or frontend."
            )

    async def filter(self, hook_name: str, data: T, **kwargs: Any) -> T:
        """
        触发一个“过滤型”钩子，形成处理链。
        此类型钩子仅在本地执行。
        """
        if hook_name not in self._hooks:
            return data

        call_context = {**self._shared_context, **kwargs}
        current_data = data
        
        for impl in self._hooks[hook_name]:
            try:
                prepared_args, prepared_kwargs = self._prepare_hook_args(impl.func, call_context, positional_data=current_data)
                current_data = await impl.func(*prepared_args, **prepared_kwargs)
            except Exception as e:
                logger.error(
                    f"Error in FILTER hook '{hook_name}' from plugin '{impl.plugin_name}'. Skipping. Error: {e}",
                    exc_info=e
                )
        
        return current_data

    async def decide(self, hook_name: str, **kwargs: Any) -> Optional[Any]:
        """
        触发一个“决策型”钩子。按优先级从高到低执行，并返回第一个非 None 的结果。
        此类型钩子仅在本地执行。
        """
        if hook_name not in self._hooks:
            return None

        call_context = {**self._shared_context, **kwargs}

        # 注意：决策型钩子应该从高优先级（数字小）到低优先级（数字大）执行，所以顺序是正确的
        for impl in self._hooks[hook_name]:
            try:
                _, prepared_kwargs = self._prepare_hook_args(impl.func, call_context)
                result = await impl.func(**prepared_kwargs)
                if result is not None:
                    logger.debug(
                        f"DECIDE hook '{hook_name}' was resolved by plugin "
                        f"'{impl.plugin_name}' with priority {impl.priority}."
                    )
                    return result
            except Exception as e:
                logger.error(
                    f"Error in DECIDE hook '{hook_name}' from plugin '{impl.plugin_name}'. Skipping. Error: {e}",
                    exc_info=e
                )
        return None
```

### core/tasks.py
```
# backend/core/tasks.py

import asyncio
import logging
from typing import Callable, Coroutine, Any, List

# 从核心契约中导入 Container 接口
from backend.core.contracts import Container, BackgroundTaskManager as BackgroundTaskManagerInterface

logger = logging.getLogger(__name__)

class BackgroundTaskManager(BackgroundTaskManagerInterface):
    """一个简单的、通用的后台任务管理器。"""
    def __init__(self, container: Container, max_workers: int = 3):
        self._container = container
        self._queue: asyncio.Queue = asyncio.Queue()
        self._workers: List[asyncio.Task] = []
        self._max_workers = max_workers
        self._is_running = False

    def start(self):
        """启动工作者协程。"""
        if self._is_running:
            logger.warning("BackgroundTaskManager is already running.")
            return
            
        logger.info(f"正在启动 {self._max_workers} 个后台工作者...")
        for i in range(self._max_workers):
            worker_task = asyncio.create_task(self._worker(f"后台工作者-{i}"))
            self._workers.append(worker_task)
        self._is_running = True

    async def stop(self):
        """优雅地停止所有工作者。"""
        if not self._is_running:
            return
            
        logger.info("正在停止后台工作者...")
        # 等待队列中的所有任务被处理完毕
        await self._queue.join()
        
        # 取消所有工作者协程
        for worker in self._workers:
            worker.cancel()
            
        # 等待所有工作者协程完全停止
        await asyncio.gather(*self._workers, return_exceptions=True)
        self._is_running = False
        logger.info("所有后台工作者已安全停止。")

    def submit_task(self, coro_func: Callable[..., Coroutine], *args: Any, **kwargs: Any):
        """
        向队列提交一个任务。
        
        :param coro_func: 要在后台执行的协程函数。
        :param args, kwargs: 传递给协程函数的参数。
        """
        if not self._is_running:
            logger.error("无法提交任务：后台任务管理器尚未启动。")
            return
            
        # 我们将协程函数本身和它的参数一起放入队列
        self._queue.put_nowait((coro_func, args, kwargs))
        logger.debug(f"任务 '{coro_func.__name__}' 已提交到后台队列。")

    async def _worker(self, name: str):
        """
        工作者协程，它会持续从队列中获取并执行任务。
        """
        logger.info(f"后台工作者 '{name}' 已启动。")
        while True:
            try:
                # 从队列中阻塞式地获取任务
                coro_func, args, kwargs = await self._queue.get()
                
                logger.debug(f"工作者 '{name}' 获取到任务: {coro_func.__name__}")
                try:
                    # 【关键】执行协程函数。
                    # 我们将容器实例作为第一个参数注入，以便后台任务能解析它需要的任何服务。
                    await coro_func(self._container, *args, **kwargs)
                except Exception:
                    logger.exception(f"工作者 '{name}' 在执行任务 '{coro_func.__name__}' 时遇到错误。")
                finally:
                    # 标记任务完成，以便 queue.join() 可以正确工作
                    self._queue.task_done()
            
            except asyncio.CancelledError:
                logger.info(f"后台工作者 '{name}' 正在关闭。")
                break
```

### core/__init__.py
```

```

### core/utils.py
```
# backend/core/utils.py

from typing import Any, Dict, List, Tuple, Union
from pathlib import Path

def _navigate_to_sub_path(
    root_obj: Dict[str, Any], 
    sub_path: str, 
    create_if_missing: bool = False
) -> Tuple[Union[Dict, List], Union[str, int]]:
    """
    Navigates a nested structure and returns the PARENT object and the FINAL key/index.
    
    Example: for path "a/b/0", it returns the list at root['a']['b'] and the index 0.
    This allows the caller to perform GET, SET, or DELETE operations.
    
    Args:
        root_obj: The dictionary to start from.
        sub_path: A slash-separated path string (e.g., "data/users/0/name").
        create_if_missing: If True, creates intermediate dictionaries for PUT operations.
        
    Returns:
        A tuple of (parent_object, final_key_or_index).
        
    Raises:
        HTTPException if the path is invalid or not found.
    """
    parts = [p for p in sub_path.split('/') if p]
    if not parts:
        raise HTTPException(status_code=400, detail="Sub-path cannot be empty.")

    current_obj = root_obj
    for i, part in enumerate(parts[:-1]): # Iterate to the second-to-last part
        try:
            # Try to convert to int for list access
            try:
                key = int(part)
                current_obj = current_obj[key]
                continue
            except (ValueError, TypeError):
                # It's a dictionary key
                if part not in current_obj:
                    if create_if_missing and isinstance(current_obj, dict):
                        current_obj[part] = {}
                    else:
                        raise HTTPException(status_code=404, detail=f"Path segment '{part}' not found.")
                current_obj = current_obj[part]

        except (KeyError, IndexError, TypeError):
            raise HTTPException(status_code=404, detail=f"Path not found at segment '{part}'.")

    # Now handle the final part
    final_key = parts[-1]
    try: # Try to convert final key to int
        final_key = int(final_key)
    except ValueError:
        pass # It's a string key
        
    return current_obj, final_key

def unwrap_dot_accessible_dicts(data: Any) -> Any:
    """
    递归地将 DotAccessibleDict 实例转换回普通的 Python 字典。
    这对于将包含这些对象的数据结构序列化为 JSON 至关重要。
    """
    if isinstance(data, DotAccessibleDict):
        # 基础情况：解包 DotAccessibleDict 并对其内容进行递归调用
        return unwrap_dot_accessible_dicts(data._data)
    elif isinstance(data, dict):
        # 递归情况：遍历字典的值
        return {key: unwrap_dot_accessible_dicts(value) for key, value in data.items()}
    elif isinstance(data, list):
        # 递归情况：遍历列表的项
        return [unwrap_dot_accessible_dicts(item) for item in data]
    else:
        # 基本类型：直接返回
        return data

class DotAccessibleDict:
    """
    一个【递归】代理类，它包装一个字典，并允许通过点符号进行属性访问。
    所有读取和写入操作都会直接作用于原始的底层字典。
    """
    def __init__(self, data: Dict[str, Any]):
        # 使用 object.__setattr__ 来避免触发我们自己的 __setattr__
        object.__setattr__(self, '_data', data)

    @classmethod
    def _wrap(cls, value: Any) -> Any:
        if isinstance(value, dict):
            # 避免重复包装
            if not isinstance(value, cls):
                return cls(value)
            return value
        if isinstance(value, list):
            return [cls._wrap(item) for item in value]
        return value

    def __contains__(self, key: str) -> bool:
        return key in self._data

    def __getattr__(self, name: str) -> Any:
        if name.startswith('__'):
            raise AttributeError(f"'{type(self).__name__}' object has no attribute '{name}'")
            
        try:
            value = self._data[name]
            return self._wrap(value)
        except KeyError:
            # 允许调用底层字典的方法，如 .keys(), .items()
            underlying_attr = getattr(self._data, name, None)
            if callable(underlying_attr):
                return underlying_attr
            raise AttributeError(f"'{type(self).__name__}' object has no attribute or method '{name}'")

    def __setattr__(self, name: str, value: Any):
        if name == '_data':
            object.__setattr__(self, name, value)
        else:
            self._data[name] = value

    def __delattr__(self, name: str):
        try:
            del self._data[name]
        except KeyError:
            raise AttributeError(f"'{type(self).__name__}' object has no attribute '{name}'")

    def __repr__(self) -> str:
        return f"DotAccessibleDict({self._data})"
    
    def __getitem__(self, key):
        return self._wrap(self._data[key])
    
    def __setitem__(self, key, value):
        self._data[key] = value
```

### core/loader.py
```
# backend/core/loader.py

import json
from pathlib import Path
import logging
import importlib
import traceback
from typing import List, Dict

from backend.core.contracts import Container, HookManager, PluginRegisterFunc

logger = logging.getLogger(__name__)

class PluginLoader:
    def __init__(self, container: Container, hook_manager: HookManager):
        self._container = container
        self._hook_manager = hook_manager
        self._loaded_plugins_metadata: List[Dict] = [] 

    def load_plugins(self):
        """执行插件加载的全过程：发现、排序、注册。"""
        print("\n--- Hevno 插件系统：开始加载 ---")
        
        all_plugins = self._discover_plugins()
        if not all_plugins:
            print("警告：在 'plugins' 目录中未发现任何插件。")
            print("--- Hevno 插件系统：加载完成 ---\n")
            return

        sorted_plugins = sorted(
            all_plugins, 
            key=lambda p: (p['manifest']['backend'].get('priority', 100), p['name'])
        )
        
        self._loaded_plugins_metadata = [p['manifest'] for p in sorted_plugins]
        self._container.register(
            "loaded_plugins_manifests", 
            lambda: self._loaded_plugins_metadata, 
            singleton=True
        )
        
        print("插件加载顺序已确定：")
        for i, p_info in enumerate(sorted_plugins):
            print(f"  {i+1}. {p_info['name']} (优先级: {p_info['manifest']['backend'].get('priority', 100)})")

        self._register_plugins(sorted_plugins)
        
        logger.info("所有已发现的插件均已加载并注册完毕。")
        print("--- Hevno 插件系统：加载完成 ---\n")
    
    def _discover_plugins(self) -> List[Dict]:
        discovered = []
        plugins_root_dir = Path(__file__).parent.parent.parent / 'plugins'

        if not plugins_root_dir.is_dir():
            print("信息：根目录下的 'plugins' 目录不存在，跳过插件加载。")
            return []

        for plugin_path in plugins_root_dir.iterdir():
            if not plugin_path.is_dir() or plugin_path.name.startswith(('__', '.')):
                continue

            manifest_path = plugin_path / "manifest.json"
            if not manifest_path.is_file():
                continue
            
            try:
                with open(manifest_path, 'r', encoding='utf-8') as f:
                    manifest = json.load(f)
                
                if "backend" in manifest:
                    import_path = f"plugins.{plugin_path.name}"
                    
                    plugin_info = {
                        "name": manifest.get('id', plugin_path.name),
                        "manifest": manifest,
                        "import_path": import_path
                    }
                    discovered.append(plugin_info)
            except Exception as e:
                print(f"警告：无法解析插件 '{plugin_path.name}' 的 manifest.json: {e}")
                pass
            
        return discovered
    
    def _register_plugins(self, plugins: List[Dict]):
        """按顺序导入并调用每个插件的注册函数。"""
        for plugin_info in plugins:
            plugin_name = plugin_info['name']
            import_path = plugin_info['import_path']
            
            try:
                plugin_module = importlib.import_module(import_path)
                
                if not hasattr(plugin_module, "register_plugin"):
                    print(f"警告：插件 '{plugin_name}' 未定义 'register_plugin' 函数，已跳过。")
                    continue
                
                register_func: PluginRegisterFunc = getattr(plugin_module, "register_plugin")
                register_func(self._container, self._hook_manager)

            except Exception as e:
                print("\n" + "="*80)
                print(f"!!! 致命错误：加载插件 '{plugin_name}' ({import_path}) 失败 !!!")
                print("="*80)
                traceback.print_exc()
                print("="*80)
                raise RuntimeError(f"无法加载插件 {plugin_name}，应用启动中止。") from e
```

### core/contracts.py
```
# backend/core/contracts.py

from __future__ import annotations
from typing import Any, Callable, Coroutine, Optional, TypeVar
from abc import ABC, abstractmethod

# --- 1. 核心服务接口与类型别名 (由平台内核提供) ---

# 定义一个泛型，常用于 filter 钩子
T = TypeVar('T')

# 插件注册函数的标准签名
PluginRegisterFunc = Callable[['Container', 'HookManager'], None]

class Container(ABC):
    @abstractmethod
    def register(self, name: str, factory: Callable, singleton: bool = True) -> None: raise NotImplementedError
    @abstractmethod
    def resolve(self, name: str) -> Any: raise NotImplementedError

class HookManager(ABC):
    @abstractmethod
    def add_implementation(self, hook_name: str, implementation: Callable, priority: int = 10, plugin_name: str = "<unknown>"): raise NotImplementedError
    @abstractmethod
    async def trigger(self, hook_name: str, **kwargs: Any) -> None: raise NotImplementedError
    @abstractmethod
    async def filter(self, hook_name: str, data: T, **kwargs: Any) -> T: raise NotImplementedError
    @abstractmethod
    async def decide(self, hook_name: str, **kwargs: Any) -> Optional[Any]: raise NotImplementedError

# 后台任务管理器是由 app.py 直接创建的平台级服务，所以其接口也属于核心契约
class BackgroundTaskManager(ABC):
    @abstractmethod
    def start(self) -> None: raise NotImplementedError
    @abstractmethod
    async def stop(self) -> None: raise NotImplementedError
    @abstractmethod
    def submit_task(self, coro_func: Callable[..., Coroutine], *args: Any, **kwargs: Any) -> None: raise NotImplementedError


```

### core/serialization.py
```
# backend/core/serialization.py

import cloudpickle
import base64
import json
from typing import Any

def pickle_fallback_encoder(obj: Any) -> Any:
    """
    一个专门用作 Pydantic `model_dump` fallback 的编码器。
    当 Pydantic 遇到无法序列化的对象时，此函数会被调用。
    """
    try:
        pickled_bytes = cloudpickle.dumps(obj)
        b64_string = base64.b64encode(pickled_bytes).decode('ascii')
        return {
            "__hevno_pickle__": True,
            "data": b64_string
        }
    except Exception as e:
        # Pydantic 的 fallback 机制期望在失败时能得到一个可序列化的错误表示
        # 或者直接抛出错误。这里我们选择抛出，让调用者知道问题。
        raise TypeError(f"Object of type {type(obj).__name__} could not be pickled by cloudpickle: {e}") from e

def custom_json_decoder_object_hook(obj: dict) -> Any:
    """
    这个解码器保持不变，因为它需要处理 `__hevno_pickle__` 结构。
    """
    if "__hevno_pickle__" in obj:
        b64_string = obj['data']
        pickled_bytes = base64.b64decode(b64_string)
        try:
            return cloudpickle.loads(pickled_bytes)
        except Exception as e:
            return {"__unpickling_error__": f"Failed to unpickle object with cloudpickle: {e}"}
    return obj
```

### core/plugin_manager.py
```
# backend/core/plugin_manager.py
import httpx
import zipfile
import io
import shutil
from pathlib import Path
from typing import Dict, Any

class PluginManager:
    def __init__(self, plugins_dir: Path, manifest: Dict[str, Any]):
        self.plugins_dir = plugins_dir
        self.manifest = manifest
        self.plugins_dir.mkdir(exist_ok=True)

    def sync(self):
        """
        Main sync method. Ensures the plugins directory matches the manifest.
        """
        print("--- Plugin Sync ---")
        # 1. 移除不再需要的插件
        self._clean_removed_plugins()
        # 2. 安装或更新声明的 Git 插件
        for name, config in self.manifest.items():
            if config.get("source") == "git":
                self._install_git_plugin(name, config)
            elif config.get("source") == "local":
                # 确认本地插件存在，如果不存在则警告
                if not (self.plugins_dir / name).exists():
                     print(f"  - ⚠️ Warning: Local plugin '{name}' declared in manifest but not found in filesystem.")
                else:
                    print(f"  - ✅ Keeping local plugin: {name}")

        print("-------------------")

    def _clean_removed_plugins(self):
        """
        Removes any directories in 'plugins/' that are not declared in the manifest.
        This correctly handles plugins that were removed from hevno.json.
        """
        print("🧹 Checking for stale plugins to remove...")
        if not self.plugins_dir.exists():
            return

        declared_plugins = set(self.manifest.keys())

        for item in self.plugins_dir.iterdir():
            # 我们只关心目录，并且忽略像 __pycache__ 这样的特殊目录
            if item.is_dir() and not item.name.startswith(('__', '.')):
                if item.name not in declared_plugins:
                    print(f"  - Removing stale plugin: {item.name}")
                    shutil.rmtree(item)

    def _install_git_plugin(self, name: str, config: Dict[str, Any]):
        """
        Fetches a plugin from a Git repository zip archive and installs it.
        Behavior depends on the 'strategy' config.
        """
        url = config["url"]
        ref = config["ref"]
        subdir = config.get("subdirectory") 
        strategy = config.get("strategy", "pin") # 默认是 "pin"
        target_dir = self.plugins_dir / name

        # 核心逻辑变更：
        # 1. 如果是 'latest' 策略，并且目录存在，则强制删除以进行更新。
        if strategy == "latest" and target_dir.exists():
            print(f"  - 🔄 Updating '{name}' to latest from branch '{ref}'. Removing old version.")
            shutil.rmtree(target_dir)
        # 2. 如果是 'pin' 策略，并且目录已存在，则跳过（保持现有行为）。
        elif strategy == "pin" and target_dir.exists():
            print(f"  - ✅ Plugin '{name}' is pinned and already exists. Skipping.")
            return

        # 如果目录不存在（或已被删除），则执行安装流程
        print(f"  - 📥 Installing '{name}' from {url} @ {ref}")
        
        repo_path = url.replace("https://github.com/", "")
        zip_url = f"https://github.com/{repo_path}/archive/{ref}.zip"

        try:
            with httpx.Client() as client:
                response = client.get(zip_url, follow_redirects=True, timeout=60)
                response.raise_for_status()
            
            zip_bytes = io.BytesIO(response.content)

            with zipfile.ZipFile(zip_bytes) as zf:
                root_folder_name = zf.namelist()[0]
                source_path_in_zip = Path(root_folder_name) / (subdir or "")

                for member in zf.infolist():
                    try:
                        member_path = Path(member.filename)
                        # 检查成员是否在我们的目标子目录内
                        relative_path = member_path.relative_to(source_path_in_zip)
                    except ValueError:
                        continue # 不是我们关心的文件，跳过

                    # 目标路径现在是 `plugins/{plugin_name}/{relative_path}`
                    final_target_path = target_dir / relative_path

                    if member.is_dir():
                        final_target_path.mkdir(parents=True, exist_ok=True)
                    else:
                        final_target_path.parent.mkdir(parents=True, exist_ok=True)
                        with final_target_path.open("wb") as f_out:
                            f_out.write(zf.read(member.filename))

        except httpx.HTTPStatusError as e:
            raise IOError(f"Failed to download plugin '{name}' from {zip_url}. Status: {e.response.status_code}") from e
        except Exception as e:
            if target_dir.exists():
                shutil.rmtree(target_dir)
            raise IOError(f"An error occurred while installing plugin '{name}': {e}") from e
```

### core/dependencies.py
```
# backend/core/dependencies.py
from typing import Any
from fastapi import Request

class Service:
    """
    一个可调用的依赖注入类，用于 FastAPI 的 Depends。
    它使得我们可以按名称从容器中解析任何服务。
    """
    def __init__(self, service_name: str):
        """
        初始化时，只记录需要解析的服务名称。
        :param service_name: 在 DI 容器中注册的服务名称。
        """
        self.service_name = service_name

    def __call__(self, request: Request) -> Any:
        """
        当 FastAPI 处理 Depends(Service(...)) 时，它会调用这个方法。
        我们从请求中获取容器，并解析所需的服务。
        """
        try:
            return request.app.state.container.resolve(self.service_name)
        except ValueError as e:
            # 如果服务未找到，这将自动导致一个清晰的服务器错误
            raise ValueError(f"Could not resolve service '{self.service_name}' from container. "
                             f"Ensure the plugin providing this service is installed and registered correctly.") from e
```

# Directory: plugins/core_engine

### editor_api.py
```
import logging
from uuid import UUID

from fastapi import HTTPException, Depends, Body, APIRouter

from backend.core.dependencies import Service
from backend.core.utils import _navigate_to_sub_path, unwrap_dot_accessible_dicts

from .contracts import (
    Sandbox,
    EditorUtilsServiceInterface,
    SnapshotStoreInterface,
    StateSnapshot,
    SandboxStoreInterface,
    Mutation,
    MutateResourceRequest,
    ResourceQueryRequest,
    ResourceQueryResponse,
    MutateResourceResponse,
)

logger = logging.getLogger(__name__)

#  路由器和标签
editor_router = APIRouter(
    prefix="/api/sandboxes/{sandbox_id}",
    tags=["Editor API - Resource Mutation"],
)

# 辅助函数 get_sandbox 保持不变
def get_sandbox(
    sandbox_id: UUID,
    sandbox_store: dict = Depends(Service("sandbox_store"))
) -> Sandbox:
    sandbox = sandbox_store.get(sandbox_id)
    if not sandbox:
        raise HTTPException(status_code=404, detail=f"Sandbox with ID '{sandbox_id}' not found.")
    return sandbox


# --- 统一资源查询API ---
@editor_router.post(
    "/resource:query",
    response_model=ResourceQueryResponse,
    summary="Atomically query any resource within a sandbox"
)
async def query_resource(
    sandbox: Sandbox = Depends(get_sandbox),
    request: ResourceQueryRequest = Body(...),
    editor_utils: EditorUtilsServiceInterface = Depends(Service("editor_utils_service"))
):
    """
    The single entry point for all data queries in the editor.
    Executes a list of path queries atomically and returns the results.
    """
    try:
        query_results = await editor_utils.execute_queries(sandbox, request.paths)
        return ResourceQueryResponse(results=query_results)
    except Exception as e:
        logger.error(f"Unexpected error during query for sandbox {sandbox.id}: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail="An internal server error occurred during query.")


# --- [核心] 统一资源修改API ---

@editor_router.post(
    "/resource:mutate",
    response_model=MutateResourceResponse,
    summary="Atomically mutate any resource within a sandbox"
)
async def mutate_resource(
    sandbox: Sandbox = Depends(get_sandbox),
    request: MutateResourceRequest = Body(...),
    editor_utils: EditorUtilsServiceInterface = Depends(Service("editor_utils_service"))
):
    """
    The single entry point for all data modifications in the editor.
    Executes a list of mutations atomically.
    - Path format: 'scope/path/to/resource' (e.g., 'lore/graphs/main').
    - For 'moment' mutations, 'mutation_mode' determines if a new snapshot is created.
    """
    if not request.mutations:
        # 如果没有修改操作，直接返回当前状态
        return MutateResourceResponse(
            sandbox_id=sandbox.id,
            head_snapshot_id=sandbox.head_snapshot_id
        )

    try:
        updated_sandbox = await editor_utils.execute_mutations(sandbox, request.mutations)
        
        return MutateResourceResponse(
            sandbox_id=updated_sandbox.id,
            head_snapshot_id=updated_sandbox.head_snapshot_id
        )
    except (ValueError, TypeError, KeyError) as e:
        # 捕获服务层可能抛出的已知错误
        logger.warning(f"Mutation request failed for sandbox {sandbox.id}: {e}")
        raise HTTPException(status_code=422, detail=str(e))
    except Exception as e:
        # 捕获未知错误
        logger.error(f"Unexpected error during mutation for sandbox {sandbox.id}: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail="An internal server error occurred during mutation.")


```

### graph_resolver.py
```
# plugins/core_engine/graph_resolver.py

import logging
from typing import Dict, Any
from copy import deepcopy

from .contracts import ExecutionContext, GraphCollection

logger = logging.getLogger(__name__)

class GraphResolver:
    """
    负责在每次图执行前，根据上下文动态地“组装”出最终要执行的图集合。
    """
    def resolve(self, context: ExecutionContext) -> GraphCollection:
        """
        根据上下文中的 Lore 和 Moment 作用域，合并和验证图定义。
        Moment 中的图会覆盖 Lore 中的同名图。
        """
        logger.debug("Resolving graph collection from Lore and Moment states...")

        # 1. 从 lore_state 获取基础图集合，如果没有则为空字典
        # 使用 deepcopy 以避免意外修改原始上下文
        base_graphs = deepcopy(context.shared.lore_state.get('graphs', {}))
        
        # 2. 从 moment_state 获取覆盖图集合
        override_graphs = deepcopy(context.shared.moment_state.get('graphs', {}))

        # 3. 将两者合并。字典的 update 方法天然地实现了覆盖逻辑。
        # 如果 override_graphs 中有与 base_graphs 同名的键，其值将覆盖 base_graphs 中的值。
        final_graph_data = {**base_graphs, **override_graphs}
        
        if not final_graph_data:
            raise ValueError("No graph definitions found in 'lore.graphs' or 'moment.graphs'.")

        logger.debug(f"Final merged graph keys: {list(final_graph_data.keys())}")
        
        # 4. 使用 Pydantic 模型进行验证
        try:
            validated_collection = GraphCollection.model_validate(final_graph_data)
        except Exception as e:
            logger.error(f"Failed to validate the final merged graph collection: {e}")
            raise ValueError(f"Invalid graph structure after merging Lore and Moment. Error: {e}")

        return validated_collection
```

### editor_utils.py
```
# plugins/core_engine/editor_utils.py
import logging
from copy import deepcopy
from uuid import UUID
from typing import Callable, Dict, Any, List

from fastapi import HTTPException

from .contracts import (
    Sandbox, EditorUtilsServiceInterface, SnapshotStoreInterface, StateSnapshot,
    SandboxStoreInterface, Mutation
)
# 确保从 contracts 导入 Mutation
from .contracts import Mutation
from backend.core.utils import _navigate_to_sub_path, unwrap_dot_accessible_dicts

logger = logging.getLogger(__name__)

class EditorUtilsService(EditorUtilsServiceInterface):
    def __init__(self, sandbox_store: SandboxStoreInterface, snapshot_store: SnapshotStoreInterface):
        self._sandbox_store = sandbox_store
        self._snapshot_store = snapshot_store

    async def execute_mutations(self, sandbox: Sandbox, mutations: List[Mutation]) -> Sandbox:
        """
        【已重构】
        以原子方式执行一批修改操作。此版本简化了逻辑，确保缓存一致性。
        """
        # 1. 在操作开始时，立即创建一份沙盒的深拷贝。所有修改都将应用于这份拷贝。
        #    这可以防止直接修改缓存中的原始对象，保证了操作的原子性。
        sandbox_to_modify = sandbox.model_copy(deep=True)
        
        # 标志位，用于判断是否需要保存沙盒对象本身
        sandbox_state_changed = False

        # 2. 将修改按其目标作用域进行分组
        moment_mutations = []
        other_mutations = []
        for m in mutations:
            if m.path.startswith("moment/"):
                moment_mutations.append(m)
            elif m.path.startswith("definition/") or m.path.startswith("lore/"):
                other_mutations.append(m)
            else:
                raise ValueError(f"Invalid mutation path root: '{m.path}'. Must start with 'moment/', 'lore/', or 'definition/'.")

        # --- 阶段 1: 处理对 Sandbox (definition/lore) 的直接修改 ---
        if other_mutations:
            try:
                for m in other_mutations:
                    # 直接在可变的拷贝上应用修改
                    self._apply_single_mutation(sandbox_to_modify, m)
                sandbox_state_changed = True
            except (KeyError, IndexError, TypeError, ValueError) as e:
                logger.error(f"Failed to apply mutation to sandbox {sandbox.id}: {e}", exc_info=True)
                raise ValueError(f"Mutation failed: {e}")
        
        # --- 阶段 2: 处理对 Moment 的修改 ---
        if moment_mutations:
            # 检查所有 moment 修改是否使用相同的 mutation_mode
            first_mode = moment_mutations[0].mutation_mode
            if not all(m.mutation_mode == first_mode for m in moment_mutations):
                raise ValueError("All mutations targeting 'moment' in a single request must use the same 'mutation_mode'.")

            if not sandbox_to_modify.head_snapshot_id:
                raise ValueError("Cannot mutate moment: Sandbox has no head snapshot.")
            
            head_snapshot = self._snapshot_store.get(sandbox_to_modify.head_snapshot_id)
            if not head_snapshot:
                 raise ValueError(f"Head snapshot {sandbox_to_modify.head_snapshot_id} not found.")

            # 深拷贝 moment 数据以进行安全修改
            mutable_moment = deepcopy(dict(head_snapshot.moment))
            try:
                 for m in moment_mutations:
                     self._apply_single_mutation(mutable_moment, m)
            except (KeyError, IndexError, TypeError, ValueError) as e:
                logger.error(f"Failed to apply mutation to moment for sandbox {sandbox.id}: {e}", exc_info=True)
                raise ValueError(f"Mutation failed: {e}")

            # 根据模式决定是覆盖保存还是创建新快照
            if first_mode == "DIRECT":
                mutated_snapshot = head_snapshot.model_copy(update={"moment": mutable_moment})
                await self._snapshot_store.save(mutated_snapshot)
                logger.info(f"Directly mutated snapshot {head_snapshot.id}.")
            else: # mode == "SNAPSHOT"
                new_snapshot = StateSnapshot(
                    sandbox_id=sandbox.id,
                    moment=mutable_moment,
                    parent_snapshot_id=sandbox_to_modify.head_snapshot_id
                )
                await self._snapshot_store.save(new_snapshot)
                # 修改沙盒拷贝的头指针
                sandbox_to_modify.head_snapshot_id = new_snapshot.id
                sandbox_state_changed = True # 头指针已改变，需要保存沙盒
                logger.info(f"Created new snapshot {new_snapshot.id} for sandbox {sandbox.id}.")

        # --- 阶段 3: 如果有任何对沙盒状态的修改，则进行持久化 ---
        if sandbox_state_changed:
            await self._sandbox_store.save(sandbox_to_modify)

        # 4. 返回经过所有修改的、最新的沙盒对象
        return sandbox_to_modify

    def _apply_single_mutation(self, root_obj: Any, mutation: Mutation):
        path_parts = mutation.path.split('/')
        scope = path_parts[0]
        sub_path = '/'.join(path_parts[1:]) if len(path_parts) > 1 else ""

        current_level = root_obj
        if isinstance(root_obj, Sandbox):
             current_level = getattr(root_obj, scope)
        
        if not sub_path:
            if mutation.type == 'UPSERT':
                if not isinstance(mutation.value, dict):
                    raise TypeError(f"Cannot replace a scope root with a non-dictionary value for path '{mutation.path}'.")
                current_level.clear()
                current_level.update(mutation.value)
                return
            else:
                raise ValueError(f"Operation '{mutation.type}' on a scope root ('{mutation.path}') is not supported.")

        parent, key = _navigate_to_sub_path(current_level, sub_path, create_if_missing=(mutation.type == 'UPSERT'))
        
        if mutation.type == 'UPSERT':
            parent[key] = mutation.value
        elif mutation.type == 'DELETE':
            try:
                if isinstance(parent, list) and isinstance(key, int):
                    del parent[key]
                elif isinstance(parent, dict):
                    del parent[key]
                else:
                    # _navigate_to_sub_path 应该已经处理了路径不存在的错误，但这里再加一层保险
                    raise KeyError
            except (KeyError, IndexError):
                 raise KeyError(f"Resource at path '{mutation.path}' not found for deletion.")
        elif mutation.type == 'LIST_APPEND':
            target_list = parent.get(key) if isinstance(parent, dict) else parent[key]
            if not isinstance(target_list, list):
                raise TypeError(f"Cannot use LIST_APPEND on a non-list object at path '{mutation.path}'.")
            target_list.append(mutation.value)

    async def execute_queries(self, sandbox: Sandbox, paths: List[str]) -> Dict[str, Any]:
        results = {}
        moment_data = None
        
        if any(p == 'moment' or p.startswith("moment/") for p in paths):
            if sandbox.head_snapshot_id:
                head_snapshot = self._snapshot_store.get(sandbox.head_snapshot_id)
                if head_snapshot:
                    moment_data = head_snapshot.moment
            if moment_data is None:
                moment_data = {}

        for path in paths:
            path_parts = path.split('/')
            scope = path_parts[0]
            sub_path = '/'.join(path_parts[1:])

            root_obj = None
            if scope == 'moment':
                root_obj = moment_data
            elif scope == 'lore':
                root_obj = sandbox.lore
            elif scope == 'definition':
                root_obj = sandbox.definition
            else:
                results[path] = None
                continue
            
            if root_obj is None:
                results[path] = None
                continue

            if not sub_path:
                results[path] = unwrap_dot_accessible_dicts(root_obj)
                continue

            try:
                parent, key = _navigate_to_sub_path(root_obj, sub_path)
                value = parent[key]
                results[path] = unwrap_dot_accessible_dicts(value)
            except (HTTPException, KeyError, IndexError, TypeError):
                results[path] = None
        
        return results
```

### models.py
```
# plugins/core_engine/models.py
from pydantic import BaseModel, Field, RootModel, field_validator
from typing import List, Dict, Any, Optional # <-- 导入 Optional

class RuntimeInstruction(BaseModel):
    """
    一个运行时指令，封装了运行时名称及其隔离的配置。
    这是节点执行逻辑的基本单元。
    """
    runtime: str
    config: Dict[str, Any] = Field(
        default_factory=dict,
        description="该运行时专属的、隔离的配置字典。"
    )

class GenericNode(BaseModel):
    """
    节点模型，现在以一个有序的运行时指令列表为核心。
    """
    id: str
    run: List[RuntimeInstruction] = Field(
        ...,
        description="定义节点执行逻辑的有序指令列表。"
    )
    depends_on: Optional[List[str]] = Field(
        default=None,
        description="一个可选的列表，用于明确声明此节点在执行前必须等待的其他节点的ID。用于处理无法通过宏自动推断的隐式依赖。"
    )
    metadata: Dict[str, Any] = Field(default_factory=dict)

class GraphDefinition(RootModel[Dict[str, Any]]):
    """
    图定义，本质上是一个字典。
    它必须包含一个 'nodes' 键，其值为一个节点列表。
    """
    @field_validator('root')
    @classmethod
    def check_nodes_exist_and_are_valid(cls, v: Dict[str, Any]) -> Dict[str, Any]:
        """验证器，确保 'nodes' 键存在且其内容是有效的节点列表。"""
        if "nodes" not in v:
            raise ValueError("A 'nodes' list must be defined in the graph definition.")
        
        # Pydantic v2 会自动验证内部列表的类型，如果 GenericNode 被正确定义。
        # 这里我们手动触发一次验证以确保健壮性。
        validated_nodes = [GenericNode.model_validate(node) for node in v["nodes"]]
        v["nodes"] = validated_nodes # 可以选择用验证过的模型替换原始数据
        return v

    @property
    def nodes(self) -> List[GenericNode]:
        """提供便捷的属性访问器，用法与旧模型保持一致。"""
        return self.root['nodes']
    
    @property
    def metadata(self) -> Dict[str, Any]:
        """提供便捷的属性访问器。"""
        return self.root.get('metadata', {})

class GraphCollection(RootModel[Dict[str, GraphDefinition]]):
    """
    整个配置文件的顶层模型。
    使用 RootModel，模型本身就是一个 `Dict[str, GraphDefinition]`。
    """
    
    @field_validator('root')
    @classmethod
    def check_main_graph_exists(cls, v: Dict[str, GraphDefinition]) -> Dict[str, GraphDefinition]:
        """验证器，确保存在一个 'main' 图作为入口点。"""
        if "main" not in v:
            raise ValueError("A 'main' graph must be defined as the entry point.")
        return v
```

### registry.py
```
# plugins/core_engine/registry.py

from typing import Dict, Type, Callable
import logging

from .contracts import RuntimeInterface

logger = logging.getLogger(__name__)

class RuntimeRegistry:
    def __init__(self):
        self._registry: Dict[str, Type[RuntimeInterface]] = {}

    def register(self, name: str, runtime_class: Type[RuntimeInterface]):
        """
        向注册表注册一个运行时类。
        """
        if name in self._registry:
            logger.warning(f"Overwriting runtime registration for '{name}'.")
        self._registry[name] = runtime_class
        logger.debug(f"Runtime '{name}' registered to the registry.")

    def get_runtime(self, name: str) -> RuntimeInterface:
        """
        获取一个运行时的【新实例】。
        """
        runtime_class = self._registry.get(name)
        if runtime_class is None:
            raise ValueError(f"Runtime '{name}' not found in registry.")
        return runtime_class()

    def get_runtime_class(self, name: str) -> Type[RuntimeInterface]:
        """
        获取运行时类本身，而不是一个实例。
        这对于访问类方法 (如 get_dependency_config) 非常有用。
        """
        runtime_class = self._registry.get(name)
        if runtime_class is None:
            raise ValueError(f"Runtime class for '{name}' not found in registry.")
        return runtime_class


```

### evaluation.py
```
# plugins/core_engine/evaluation.py 

import ast
import asyncio
import re
from typing import Any, Dict, List, Optional   
from functools import partial
import random
import math
import datetime
import json
import re as re_module

from backend.core.utils import DotAccessibleDict 
from .contracts import ExecutionContext

INLINE_MACRO_REGEX = re.compile(r"{{\s*(.+?)\s*}}", re.DOTALL)
MACRO_REGEX = re.compile(r"^{{\s*(.+)\s*}}$", re.DOTALL)
import random
import math
import datetime
import json
import re as re_module

PRE_IMPORTED_MODULES = {
    "random": random,
    "math": math,
    "datetime": datetime,
    "json": json,
    "re": re_module,
}

def build_evaluation_context(
    exec_context: ExecutionContext,
    pipe_vars: Optional[Dict[str, Any]] = None
) -> Dict[str, Any]:
    """
    从 ExecutionContext 构建宏的执行环境，注入新的三层作用域。
    """
    context = {
        **PRE_IMPORTED_MODULES,
        # 【新】注入三层作用域，并用 DotAccessibleDict 包装以支持点符号访问
        "definition": DotAccessibleDict(exec_context.shared.definition_state),
        "lore": DotAccessibleDict(exec_context.shared.lore_state),
        "moment": DotAccessibleDict(exec_context.shared.moment_state),
        
        # 共享服务和会话信息保持不变
        "services": exec_context.shared.services, # services 已经是 DotAccessibleDict
        "session": DotAccessibleDict(exec_context.shared.session_info),
        
        # run 和 nodes 是当前图执行所私有的
        "run": DotAccessibleDict(exec_context.run_vars),
        "nodes": DotAccessibleDict(exec_context.node_states),
    }
    if pipe_vars is not None:
        context['pipe'] = DotAccessibleDict(pipe_vars)
        
    return context

# ... (evaluate_expression 和 evaluate_data 函数保持不变) ...
async def evaluate_expression(code_str: str, context: Dict[str, Any], lock: asyncio.Lock) -> Any:
    """..."""
    # ast.parse 可能会失败，需要 try...except
    try:
        tree = ast.parse(code_str, mode='exec')
    except SyntaxError as e:
        raise ValueError(f"Macro syntax error: {e}\nCode: {code_str}")

    # 如果代码块为空，直接返回 None
    if not tree.body:
        return None

    # 如果最后一行是表达式，我们将其转换为一个赋值语句，以便捕获结果
    result_var = "_macro_result"
    if isinstance(tree.body[-1], ast.Expr):
        # 包装最后一条表达式
        assign_node = ast.Assign(
            targets=[ast.Name(id=result_var, ctx=ast.Store())],
            value=tree.body[-1].value
        )
        tree.body[-1] = ast.fix_missing_locations(assign_node)
    
    # 将 AST 编译为代码对象
    code_obj = compile(tree, filename="<macro>", mode="exec")
    
    # 在锁的保护下运行
    async with lock:
        # 在另一个线程中运行，以避免阻塞事件循环
        # 注意：这里我们直接修改传入的 context 字典来捕获结果
        await asyncio.get_running_loop().run_in_executor(
            None, exec, code_obj, context
        )
    
    # 从被修改的上下文字典中获取结果
    return context.get(result_var)

async def evaluate_data(data: Any, eval_context: Dict[str, Any], lock: asyncio.Lock) -> Any:
    if isinstance(data, str):
        # 模式1: 检查是否为“全宏替换”
        # 这种模式很重要，因为它允许宏返回非字符串类型（如列表、布尔值）
        full_match = MACRO_REGEX.match(data)
        if full_match:
            code_to_run = full_match.group(1)
            # 这里返回的结果可以是任何类型
            return await evaluate_expression(code_to_run, eval_context, lock)

        # 模式2: 如果不是全宏，检查是否包含“内联模板”
        # 这种模式的结果总是字符串
        if '{{' in data and '}}' in data:
            matches = list(INLINE_MACRO_REGEX.finditer(data))
            if not matches:
                # 包含 {{ 和 }} 但格式不正确，按原样返回
                return data

            # 并发执行所有宏的求值
            codes_to_run = [m.group(1) for m in matches]
            tasks = [evaluate_expression(code, eval_context, lock) for code in codes_to_run]
            evaluated_results = await asyncio.gather(*tasks)

            # 将求值结果替换回原字符串
            # 使用一个迭代器来确保替换顺序正确
            results_iter = iter(evaluated_results)
            # re.sub 的 lambda 每次调用时，都会从迭代器中取下一个结果
            # 这比多次调用 str.replace() 更安全、更高效
            final_string = INLINE_MACRO_REGEX.sub(lambda m: str(next(results_iter)), data)
            
            return final_string

        # 如果两种模式都不匹配，说明是普通字符串
        return data

    if isinstance(data, dict):
        keys = list(data.keys())
        # 创建异步任务列表
        value_tasks = [evaluate_data(data[k], eval_context, lock) for k in keys]
        # 并发执行所有值的求值
        evaluated_values = await asyncio.gather(*value_tasks)
        # 重新组装字典
        return dict(zip(keys, evaluated_values))

    if isinstance(data, list):
        # 并发执行列表中所有项的求值
        item_tasks = [evaluate_data(item, eval_context, lock) for item in data]
        return await asyncio.gather(*item_tasks)

    return data
```

### __init__.py
```
# plugins/core_engine/__init__.py

import logging
from typing import Dict, Type, List
from fastapi import APIRouter

from backend.core.contracts import Container, HookManager
from .engine import ExecutionEngine
from .registry import RuntimeRegistry
from .state import SnapshotStore
from .contracts import RuntimeInterface
from .evaluation_service import MacroEvaluationService


from .runtimes.io_runtimes import InputRuntime, LogRuntime
from .runtimes.data_runtimes import FormatRuntime, ParseRuntime, RegexRuntime
from .runtimes.flow_runtimes import ExecuteRuntime, CallRuntime, MapRuntime
from .api import router as sandbox_router
from .editor_api import editor_router
from .editor_utils import EditorUtilsService

logger = logging.getLogger(__name__)

# --- 服务工厂  ---
def _create_runtime_registry() -> RuntimeRegistry:
    registry = RuntimeRegistry()
    base_runtimes: Dict[str, Type[RuntimeInterface]] = {
        "system.io.input": InputRuntime, "system.io.log": LogRuntime,
        "system.data.format": FormatRuntime, "system.data.parse": ParseRuntime, "system.data.regex": RegexRuntime,
        "system.flow.call": CallRuntime, "system.flow.map": MapRuntime, "system.execute": ExecuteRuntime,
    }
    for name, runtime_class in base_runtimes.items():
        registry.register(name, runtime_class)
    logger.info(f"Registered {len(base_runtimes)} built-in system runtimes.")
    return registry

def _create_execution_engine(container: Container) -> ExecutionEngine:
    return ExecutionEngine(
        registry=container.resolve("runtime_registry"),
        container=container,
        hook_manager=container.resolve("hook_manager")
    )

def _create_editor_utils_service(container: Container) -> EditorUtilsService:
    sandbox_store = container.resolve("sandbox_store")
    snapshot_store = container.resolve("snapshot_store")
    return EditorUtilsService(sandbox_store=sandbox_store, snapshot_store=snapshot_store)

# --- 钩子实现 ---
async def populate_runtime_registry(container: Container):
    logger.debug("Async task: Populating runtime registry from other plugins...")
    hook_manager = container.resolve("hook_manager")
    registry = container.resolve("runtime_registry")
    external_runtimes: Dict[str, Type[RuntimeInterface]] = await hook_manager.filter("collect_runtimes", {})
    if external_runtimes:
        logger.info(f"Discovered {len(external_runtimes)} external runtime(s): {list(external_runtimes.keys())}")
        for name, runtime_class in external_runtimes.items():
            registry.register(name, runtime_class)

# --- (新) 钩子实现：提供API路由 ---
async def provide_api_router(routers: List[APIRouter]) -> List[APIRouter]:
    """钩子实现：将本插件的路由添加到收集中。"""
    routers.append(sandbox_router)
    routers.append(editor_router)
    logger.debug("Provided sandbox runner and editor API routers to the application.")
    return routers

# --- 主注册函数 ---
def register_plugin(container: Container, hook_manager: HookManager):
    logger.info("--> 正在注册 [core_engine] 插件...")

    container.register("runtime_registry", _create_runtime_registry, singleton=True)
    container.register("execution_engine", _create_execution_engine, singleton=True)
    container.register("macro_evaluation_service", lambda: MacroEvaluationService(), singleton=True)
    container.register("editor_utils_service", _create_editor_utils_service, singleton=True)
    
    hook_manager.add_implementation(
        "services_post_register", 
        populate_runtime_registry, 
        plugin_name="core_engine"
    )
    
    # --- 注册提供API路由的钩子 ---
    hook_manager.add_implementation(
        "collect_api_routers",
        provide_api_router,
        plugin_name="core_engine"
    )

    logger.info("插件 [core_engine] 注册成功。")
```

### api.py
```
# plugins/core_engine/api.py

import io
import logging
import json
import time
import uuid
import copy
import aiofiles
from typing import Dict, Any, List, Optional
from uuid import UUID
from pydantic import BaseModel, Field, ValidationError, field_validator
from datetime import datetime, timezone
from PIL import Image

from fastapi import APIRouter, Body, Depends, HTTPException, UploadFile, File
from fastapi.responses import Response, FileResponse, JSONResponse

# 导入核心依赖解析器和所有必要的接口与数据模型（契约）
from backend.core.dependencies import Service
from backend.core.serialization import pickle_fallback_encoder, custom_json_decoder_object_hook
from plugins.core_engine.contracts import (
    Sandbox, 
    StateSnapshot, 
    ExecutionEngineInterface,
    SnapshotStoreInterface,
    StepDiagnostics,
    StepResponse
)
from plugins.core_persistence.contracts import (
    PersistenceServiceInterface, 
    PackageManifest, 
    PackageType
)
# 导入新的持久化存储类以进行类型提示，增强代码可读性
from plugins.core_engine.contracts import SandboxStoreInterface

logger = logging.getLogger(__name__)

router = APIRouter(
    prefix="/api/sandboxes", 
    tags=["Sandboxes"]
)

# --- Request/Response Models ---

class CreateSandboxRequest(BaseModel):
    name: str = Field(..., description="沙盒的人类可读名称。")
    definition: Optional[Dict[str, Any]] = Field(
        None, 
        description="沙盒的'设计蓝图'，如果未提供，将使用默认模板。"
    )

class SandboxListItem(BaseModel):
    id: UUID
    name: str
    created_at: datetime
    icon_url: str
    has_custom_icon: bool

class UpdateSandboxRequest(BaseModel):
    name: str = Field(..., min_length=1, description="沙盒的新名称。")

class SandboxArchiveJSON(BaseModel):
    sandbox: Sandbox
    snapshots: List[StateSnapshot]

# --- Sandbox Lifecycle API ---

@router.get("/{sandbox_id}", response_model=Sandbox, summary="Get a single Sandbox by ID")
async def get_sandbox_by_id(
    sandbox_id: UUID,
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store"))
):
    """
    通过其 ID 检索单个沙盒的完整对象。
    """
    sandbox = sandbox_store.get(sandbox_id)
    if not sandbox:
        raise HTTPException(status_code=404, detail="Sandbox not found.")
    return sandbox
@router.post("", response_model=Sandbox, status_code=201, summary="Create a new Sandbox")
async def create_sandbox(
    request_body: CreateSandboxRequest, 
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store")),
    snapshot_store: SnapshotStoreInterface = Depends(Service("snapshot_store"))
):
    """
    创建一个新的沙盒，并将其立即持久化到本地文件系统。
    如果未提供定义，则使用默认的聊天机器人模板。
    """
    # --- [MODIFIED] 扩展默认模板以包含 codex 定义 ---
    DEFAULT_LORE = {
        "graphs": {
            "main": {
                "__hevno_type__": "hevno/graph",
                "nodes": [
                    {
                        "id": "record_user_input", 
                        "run": [{
                            "runtime": "memoria.add", 
                            "config": {
                                "stream": "chat_history", 
                                "level": "user",
                                "content": "{{ moment._user_input }}"
                            }
                        }]
                    },
                    {
                        "id": "get_chat_history",
                        "run": [{
                            "runtime": "memoria.query",
                            "config": {
                                "stream": "chat_history",
                                "latest": 10,
                                "format": "message_list"
                            }
                        }]
                    },
                    {
                        "id": "get_system_prompt",
                        "run": [{
                            "runtime": "codex.invoke",
                            "config": {
                                "from": [{"codex": "ai_persona"}]
                            }
                        }]
                    },
                    {
                        "id": "generate_response", 
                        "depends_on": ["record_user_input", "get_chat_history", "get_system_prompt"],
                        "run": [{
                            "runtime": "llm.default", 
                            "config": {
                                "model": "gemini/gemini-1.5-flash",
                                "contents": [
                                    {
                                        "name": "系统提示",
                                        "type": "MESSAGE_PART",
                                        "role": "system",
                                        "content": "{{ nodes.get_system_prompt.output }}"
                                    },
                                    {
                                        "name": "注入聊天记录",
                                        "type": "INJECT_MESSAGES",
                                        "source": "{{ nodes.get_chat_history.output }}",
                                        "is_enabled": "{{  len(nodes.get_chat_history.output) > 0 }}"
                                    },
                                    {
                                        "name": "用户当前输入",
                                        "type": "MESSAGE_PART",
                                        "role": "user",
                                        "content": "{{ moment._user_input }}"
                                    }
                                ]
                            }
                        }]
                    },
                    {
                        "id": "set_output", 
                        "depends_on": ["generate_response"], 
                        "run": [{
                            "runtime": "system.execute", 
                            "config": {
                                "code": "{{ moment._user_output = nodes.generate_response.output }}"
                            }
                        }]
                    },
                    {
                        "id": "record_ai_response", 
                        "depends_on": ["set_output"], 
                        "run": [{
                            "runtime": "memoria.add", 
                            "config": {
                                "stream": "chat_history", 
                                "level": "model",
                                "content": "{{ moment._user_output }}"
                            }
                        }]
                    }
                ]
            }
        },
        "codices": {
            "ai_persona": {
                "__hevno_type__": "hevno/codex",
                "description": "Defines the core personality and instructions for the AI.",
                "entries": [
                    {
                        "id": "core_identity",
                        "priority": 100,
                        "content": "You are Hevno, a friendly and helpful AI assistant designed to demonstrate the capabilities of the Hevno Engine. You are currently running inside a default sandbox template."
                    },
                    {
                        "id": "personality_quirk",
                        "priority": 50,
                        "content": "You should be concise but not robotic. Feel free to use emojis where appropriate. 😊 Your goal is to be helpful and showcase the system's features."
                    }
                ]
            }
        }
    }
    DEFAULT_MOMENT = {
        "_user_input": "",
        "_user_output": "",
        "memoria": {
            "__hevno_type__": "hevno/memoria",
            "__global_sequence__": 0,
            "chat_history": {"config": {}, "entries": []}
        }
    }
    DEFAULT_DEFINITION = {
        "name": "Default Chat Sandbox",
        "description": "A default sandbox configured for conversational chat with a persona defined by a Codex.",
        "initial_lore": DEFAULT_LORE,
        "initial_moment": DEFAULT_MOMENT
    }

    # (函数其余部分保持不变)
    if request_body.definition:
        if "initial_lore" not in request_body.definition or "initial_moment" not in request_body.definition:
            raise HTTPException(status_code=422, detail="Custom definition must contain 'initial_lore' and 'initial_moment' keys.")
        initial_lore = request_body.definition.get("initial_lore", {})
        initial_moment = request_body.definition.get("initial_moment", {})
        definition = request_body.definition
    else:
        initial_lore = DEFAULT_LORE
        initial_moment = DEFAULT_MOMENT
        # 将默认定义与用户提供的名称合并
        definition = {**DEFAULT_DEFINITION, "name": request_body.name}


    sandbox = Sandbox(
        name=request_body.name,
        definition=definition,
        lore=initial_lore
    )
    if sandbox.id in sandbox_store:
        raise HTTPException(status_code=409, detail=f"Sandbox with ID {sandbox.id} already exists.")
    
    genesis_snapshot = StateSnapshot(
        sandbox_id=sandbox.id,
        moment=initial_moment
    )
    await snapshot_store.save(genesis_snapshot)
    
    sandbox.head_snapshot_id = genesis_snapshot.id
    
    await sandbox_store.save(sandbox)
    
    logger.info(f"Created new sandbox '{sandbox.name}' ({sandbox.id}) and saved to disk.")
    return sandbox

# ... (文件的其余部分保持不变) ...
@router.post("/{sandbox_id}/step", response_model=StepResponse, summary="Execute a step")
async def execute_sandbox_step(
    sandbox_id: UUID, 
    user_input: Dict[str, Any] = Body(...),
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store")),
    engine: ExecutionEngineInterface = Depends(Service("execution_engine"))
):
    """
    执行一步计算。持久化逻辑已封装在 engine.step 方法内部。
    返回一个包含执行元数据和更新后沙盒的信封。
    """
    sandbox = sandbox_store.get(sandbox_id)
    if not sandbox:
        raise HTTPException(status_code=404, detail="Sandbox not found.")
    
    start_time = time.monotonic()
    
    try:
        updated_sandbox = await engine.step(sandbox, user_input)
        execution_time_ms = (time.monotonic() - start_time) * 1000
        
        # 从临时属性中获取诊断日志
        diagnostics_log = getattr(updated_sandbox, '_temp_diagnostics_log', None)
        if hasattr(updated_sandbox, '_temp_diagnostics_log'):
            delattr(updated_sandbox, '_temp_diagnostics_log') # 清理临时属性

        return StepResponse(
            status="COMPLETED",
            sandbox=updated_sandbox,
            diagnostics=StepDiagnostics(
                execution_time_ms=execution_time_ms,
                detailed_log=diagnostics_log # 将日志放入响应
            )
        )
    except Exception as e:
        logger.error(f"Error during engine step for sandbox {sandbox_id}: {e}", exc_info=True)
        # 失败时，返回执行前的沙盒状态
        return JSONResponse(
            status_code=500,
            content=StepResponse(
                status="ERROR",
                sandbox=sandbox, # 返回原始沙盒
                error_message=str(e)
            ).model_dump(mode="json")
        )


@router.put("/{sandbox_id}/revert", status_code=200, summary="Revert to a snapshot")
async def revert_sandbox_to_snapshot(
    sandbox_id: UUID, 
    snapshot_id: UUID = Body(..., embed=True),
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store")),
    snapshot_store: SnapshotStoreInterface = Depends(Service("snapshot_store"))
):
    """
    将沙盒的状态回滚到指定的历史快照。
    """
    sandbox = sandbox_store.get(sandbox_id)
    if not sandbox:
        raise HTTPException(status_code=404, detail="Sandbox not found.")

    target_snapshot = snapshot_store.get(snapshot_id)
    # 如果缓存里没有，就从磁盘加载所有快照来确认它是否存在
    if not target_snapshot or target_snapshot.sandbox_id != sandbox.id:
        all_snapshots = await snapshot_store.find_by_sandbox(sandbox_id)
        if not any(s.id == snapshot_id for s in all_snapshots):
            raise HTTPException(status_code=404, detail="Target snapshot not found or does not belong to this sandbox.")

    sandbox.head_snapshot_id = snapshot_id
    
    await sandbox_store.save(sandbox)
    
    logger.info(f"Reverted sandbox '{sandbox.name}' ({sandbox.id}) to snapshot {snapshot_id} and saved.")
    return {"message": f"Sandbox '{sandbox.name}' successfully reverted to snapshot {snapshot_id}"}

@router.delete("/{sandbox_id}/snapshots/{snapshot_id}", status_code=204, summary="Delete a Snapshot")
async def delete_snapshot(
    sandbox_id: UUID,
    snapshot_id: UUID,
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store")),
    snapshot_store: SnapshotStoreInterface = Depends(Service("snapshot_store")),
):
    """
    删除一个指定的历史快照。

    **注意**: 为了保证沙盒的完整性，不允许删除当前作为 `head` 的快照。
    如果需要删除 `head` 快照，请先将沙盒 `revert` 到另一个快照。
    """
    sandbox = sandbox_store.get(sandbox_id)
    if not sandbox:
        raise HTTPException(status_code=404, detail="Sandbox not found.")

    if sandbox.head_snapshot_id == snapshot_id:
        raise HTTPException(
            status_code=409,  # 409 Conflict 是一个合适的代码
            detail="Cannot delete the head snapshot. Please revert to another snapshot first."
        )

    # 检查快照是否存在（可选，但更健壮）
    snapshot = snapshot_store.get(snapshot_id)
    if not snapshot or snapshot.sandbox_id != sandbox_id:
        # 即使找不到，也返回成功，因为最终状态是“不存在”
        return Response(status_code=204)

    await snapshot_store.delete(snapshot_id)
    
    logger.info(f"Deleted snapshot '{snapshot_id}' for sandbox '{sandbox.name}' ({sandbox.id}).")
    return Response(status_code=204)

@router.post("/{sandbox_id}/history:reset", response_model=Sandbox, summary="Reset Sandbox History")
async def reset_sandbox_history(
    sandbox_id: UUID,
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store")),
    snapshot_store: SnapshotStoreInterface = Depends(Service("snapshot_store")),
):
    """
    通过创建一个新的“创世”快照来重置沙盒的会话历史。

    此操作会：
    1. 读取沙盒 `definition` 中的 `initial_moment`。
    2. 基于此 `initial_moment` 创建一个全新的 `StateSnapshot`。
    3. 将沙盒的 `head_snapshot_id` 指向这个新快照。
    4. 新快照没有父快照，有效开启一个全新的、干净的对话分支。
    """
    sandbox = sandbox_store.get(sandbox_id)
    if not sandbox:
        raise HTTPException(status_code=404, detail="Sandbox not found.")

    initial_moment = sandbox.definition.get("initial_moment")
    if not isinstance(initial_moment, dict):
        raise HTTPException(
            status_code=422,
            detail="Cannot reset history: Sandbox 'definition' is missing a valid 'initial_moment' dictionary."
        )

    # 创建一个新的创世快照
    new_genesis_snapshot = StateSnapshot(
        sandbox_id=sandbox.id,
        moment=initial_moment,
        parent_snapshot_id=None # 关键：这开启了一个新分支
    )
    
    # 保存新快照
    await snapshot_store.save(new_genesis_snapshot)
    
    # 更新沙盒的头指针
    sandbox.head_snapshot_id = new_genesis_snapshot.id
    
    # 保存更新后的沙盒
    await sandbox_store.save(sandbox)
    
    logger.info(f"Reset history for sandbox '{sandbox.name}' ({sandbox.id}). New head snapshot is {new_genesis_snapshot.id}.")
    
    # 返回更新后的沙盒，让前端立即知道新状态
    return sandbox

@router.post(
    "/{sandbox_id}/apply_definition", 
    response_model=Sandbox, 
    summary="Apply Definition to State"
)
async def apply_definition_to_sandbox(
    sandbox_id: UUID,
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store")),
    snapshot_store: SnapshotStoreInterface = Depends(Service("snapshot_store")),
):
    """
    使用沙盒 `definition` 中的 `initial_lore` 和 `initial_moment` 
    来完全重置当前的 lore 和 moment 状态。
    这是一个破坏性操作，会开启一个全新的历史分支。
    """
    sandbox = sandbox_store.get(sandbox_id)
    if not sandbox:
        raise HTTPException(status_code=404, detail="Sandbox not found.")

    # 1. 验证并获取定义
    initial_lore = sandbox.definition.get("initial_lore")
    initial_moment = sandbox.definition.get("initial_moment")

    if not isinstance(initial_lore, dict) or not isinstance(initial_moment, dict):
        raise HTTPException(
            status_code=422,
            detail="Cannot apply definition: 'initial_lore' or 'initial_moment' is missing or not a dictionary in the definition."
        )

    # 2. 重置 Lore
    # 使用深拷贝以避免任何意外的引用问题
    sandbox.lore = copy.deepcopy(initial_lore)
    logger.info(f"Applied initial_lore to sandbox '{sandbox.name}' ({sandbox.id}).")

    # 3. 重置 Moment (通过创建一个新的创世快照)
    new_genesis_snapshot = StateSnapshot(
        sandbox_id=sandbox.id,
        moment=copy.deepcopy(initial_moment),
        parent_snapshot_id=None # 关键: 这开启了一个新的、干净的历史分支
    )
    await snapshot_store.save(new_genesis_snapshot)
    
    # 4. 更新沙盒的头指针
    sandbox.head_snapshot_id = new_genesis_snapshot.id
    
    # 5. 持久化更新后的沙盒 (包含新的 lore 和 head_snapshot_id)
    await sandbox_store.save(sandbox)
    
    logger.info(f"Reset history for sandbox '{sandbox.name}' based on definition. New head is {new_genesis_snapshot.id}.")
    
    # 6. 返回更新后的沙盒对象
    return sandbox


@router.get("/{sandbox_id}/history", response_model=List[StateSnapshot], summary="Get history")
async def get_sandbox_history(
    sandbox_id: UUID,
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store")),
    snapshot_store: SnapshotStoreInterface = Depends(Service("snapshot_store"))
):
    if sandbox_id not in sandbox_store:
        raise HTTPException(status_code=404, detail="Sandbox not found.")
    return await snapshot_store.find_by_sandbox(sandbox_id)


@router.patch("/{sandbox_id}", response_model=Sandbox, summary="Update Sandbox Details")
async def update_sandbox_details(
    sandbox_id: UUID,
    request_body: UpdateSandboxRequest,
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store"))
):
    sandbox = sandbox_store.get(sandbox_id)
    if not sandbox:
        raise HTTPException(status_code=404, detail="Sandbox not found.")
    sandbox.name = request_body.name
    
    await sandbox_store.save(sandbox)
    
    logger.info(f"Updated name for sandbox '{sandbox.id}' to '{sandbox.name}' and saved.")
    return sandbox


@router.delete("/{sandbox_id}", status_code=204, summary="Delete a Sandbox")
async def delete_sandbox(
    sandbox_id: UUID,
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store")),
):
    """
    从文件系统和缓存中完全删除一个沙盒及其所有数据。
    """
    if sandbox_id not in sandbox_store:
        raise HTTPException(status_code=404, detail="Sandbox not found.")
    
    await sandbox_store.delete(sandbox_id)
    
    logger.info(f"Deleted sandbox '{sandbox_id}' and all associated data from disk.")
    return Response(status_code=204)


@router.get("", response_model=List[SandboxListItem], summary="List all Sandboxes")
async def list_sandboxes(
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store")),
    persistence_service: PersistenceServiceInterface = Depends(Service("persistence_service"))
):
    # sandbox_store.values() 从缓存读取，是同步的
    all_sandboxes = sandbox_store.values()
    response_items = []
    
    for sandbox in all_sandboxes:
        icon_path = persistence_service.get_sandbox_icon_path(str(sandbox.id))
        has_custom_icon = icon_path is not None
        
        icon_url = f"/api/sandboxes/{sandbox.id}/icon"
        if sandbox.icon_updated_at:
            icon_url += f"?v={int(sandbox.icon_updated_at.timestamp())}"
        
        response_items.append(
            SandboxListItem(
                id=sandbox.id,
                name=sandbox.name,
                created_at=sandbox.created_at,
                icon_url=icon_url,
                has_custom_icon=has_custom_icon
            )
        )
        
    return sorted(response_items, key=lambda s: s.created_at, reverse=True)

# --- Icon, Export, Import 端点 ---

@router.get("/{sandbox_id}/icon", response_class=FileResponse, summary="Get Sandbox Icon")
async def get_sandbox_icon(
    sandbox_id: UUID,
    persistence_service: PersistenceServiceInterface = Depends(Service("persistence_service"))
):
    icon_path = persistence_service.get_sandbox_icon_path(str(sandbox_id))
    if icon_path:
        return FileResponse(icon_path)
    
    default_icon_path = persistence_service.get_default_icon_path()
    if not default_icon_path.is_file():
        raise HTTPException(status_code=404, detail="Default icon not found on server.")
    return FileResponse(default_icon_path)


@router.post("/{sandbox_id}/icon", status_code=200, summary="Upload/Update Sandbox Icon")
async def upload_sandbox_icon(
    sandbox_id: UUID,
    file: UploadFile = File(...),
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store")),
    persistence_service: PersistenceServiceInterface = Depends(Service("persistence_service")),
):
    sandbox = sandbox_store.get(sandbox_id)
    if not sandbox:
        raise HTTPException(status_code=404, detail="Sandbox not found.")
    
    if not file.content_type == "image/png":
        raise HTTPException(status_code=400, detail="Invalid file type. Only PNG is allowed.")

    icon_bytes = await file.read()
    
    try:
        img = Image.open(io.BytesIO(icon_bytes))
        img.verify() 
        img = Image.open(io.BytesIO(icon_bytes))
        if img.format != 'PNG':
            raise ValueError("Image format is not PNG.")
        if max(img.size) > 2048:
            raise ValueError("Image dimensions are too large (max 2048x2048).")
    except Exception as e:
        raise HTTPException(status_code=422, detail=f"Invalid PNG file: {e}")

    # 添加 await 调用异步方法
    await persistence_service.save_sandbox_icon(str(sandbox.id), icon_bytes)
    sandbox.icon_updated_at = datetime.now(timezone.utc)
    
    await sandbox_store.save(sandbox)
    
    return {"message": "Icon updated successfully."}


@router.get(
    "/{sandbox_id}/export/json", 
    response_class=JSONResponse, 
    summary="Export a Sandbox as JSON"
)
async def export_sandbox_json(
    sandbox_id: UUID,
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store")),
    snapshot_store: SnapshotStoreInterface = Depends(Service("snapshot_store")),
):
    sandbox = sandbox_store.get(sandbox_id)
    if not sandbox:
        raise HTTPException(status_code=404, detail="Sandbox not found")

    # 添加 await 调用异步方法
    snapshots = await snapshot_store.find_by_sandbox(sandbox_id)
    if not snapshots:
        raise HTTPException(status_code=404, detail="No snapshots found for this sandbox to export.")

    archive = SandboxArchiveJSON(sandbox=sandbox, snapshots=snapshots)
    filename = f"hevno_sandbox_{sandbox.name.replace(' ', '_')}_{sandbox_id}.json"
    headers = {"Content-Disposition": f"attachment; filename={filename}"}
    return JSONResponse(content=archive.model_dump(mode="json"), headers=headers)


@router.post(
    "/import/json", 
    response_model=Sandbox, 
    status_code=201, 
    summary="Import a Sandbox from JSON"
)
async def import_sandbox_json(
    file: UploadFile = File(...),
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store")),
    snapshot_store: SnapshotStoreInterface = Depends(Service("snapshot_store")),
) -> Sandbox:
    if not file.content_type == "application/json":
        raise HTTPException(status_code=400, detail="Invalid file type. Please upload a .json file.")

    try:
        content = await file.read()
        data = json.loads(content, object_hook=custom_json_decoder_object_hook)
        archive = SandboxArchiveJSON.model_validate(data)
    except (json.JSONDecodeError, ValidationError) as e:
        raise HTTPException(status_code=422, detail=f"Invalid sandbox archive format: {e}")

    old_sandbox_id = archive.sandbox.id
    new_sandbox_id = uuid.uuid4()
    snapshot_id_map = {snap.id: uuid.uuid4() for snap in archive.snapshots}
    
    for old_snapshot in archive.snapshots:
        new_snapshot = old_snapshot.model_copy(update={
            'id': snapshot_id_map[old_snapshot.id],
            'sandbox_id': new_sandbox_id,
            'parent_snapshot_id': snapshot_id_map.get(old_snapshot.parent_snapshot_id)
        })
        await snapshot_store.save(new_snapshot)
    
    new_head_id = snapshot_id_map.get(archive.sandbox.head_snapshot_id)
    new_sandbox = archive.sandbox.model_copy(update={
        'id': new_sandbox_id,
        'head_snapshot_id': new_head_id,
        'name': f"{archive.sandbox.name} (Imported)",
        'created_at': datetime.now(timezone.utc),
        'icon_updated_at': None
    })

    if new_sandbox.id in sandbox_store:
         raise HTTPException(status_code=409, detail=f"A sandbox with the newly generated ID '{new_sandbox.id}' already exists. This is highly unlikely, please try again.")
    
    await sandbox_store.save(new_sandbox)

    logger.info(f"Successfully imported sandbox from JSON, new ID is '{new_sandbox.id}'. Original ID was '{old_sandbox_id}'.")
    return new_sandbox


# PNG 导入/导出端点
@router.get("/{sandbox_id}/export", response_class=Response, summary="Export a Sandbox as PNG")
async def export_sandbox(
    sandbox_id: UUID,
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store")),
    snapshot_store: SnapshotStoreInterface = Depends(Service("snapshot_store")),
    persistence_service: PersistenceServiceInterface = Depends(Service("persistence_service"))
):
    sandbox = sandbox_store.get(sandbox_id)
    if not sandbox:
        raise HTTPException(status_code=404, detail="Sandbox not found")

    # 添加 await 调用异步方法
    snapshots = await snapshot_store.find_by_sandbox(sandbox_id)
    if not snapshots:
        raise HTTPException(status_code=404, detail="No snapshots found for this sandbox to export.")

    manifest = PackageManifest(
        package_type=PackageType.SANDBOX_ARCHIVE,
        entry_point="sandbox.json",
        metadata={"sandbox_name": sandbox.name}
    )
    
    # 使用与持久化存储相同的序列化机制，确保UUID等类型被正确处理
    export_sandbox_data = sandbox.model_dump(mode='json', fallback=pickle_fallback_encoder)
    
    # 为了兼容旧版，手动添加 graph_collection (如果 lore.graphs 存在)
    if sandbox.lore and 'graphs' in sandbox.lore:
        export_sandbox_data['graph_collection'] = sandbox.lore['graphs']

    data_files: Dict[str, Any] = {"sandbox.json": export_sandbox_data}
    for snap in snapshots:
        # 对每个快照也使用相同的序列化机制
        data_files[f"snapshots/{snap.id}.json"] = snap.model_dump(mode='json', fallback=pickle_fallback_encoder)

    base_image_bytes = None
    icon_path = persistence_service.get_sandbox_icon_path(str(sandbox.id))
    if icon_path and icon_path.is_file():
        # 读取图标文件是I/O操作
        async with aiofiles.open(icon_path, 'rb') as f:
            base_image_bytes = await f.read()
    
    if not base_image_bytes:
        default_icon_path = persistence_service.get_default_icon_path()
        if default_icon_path.is_file():
             async with aiofiles.open(default_icon_path, 'rb') as f:
                base_image_bytes = await f.read()

    try:
        # 添加 await 调用异步方法
        png_bytes = await persistence_service.export_package(manifest, data_files, base_image_bytes)
    except Exception as e:
        logger.error(f"Failed to create package for sandbox {sandbox_id}: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail=f"Failed to create package: {e}")

    filename = f"hevno_sandbox_{sandbox.name.replace(' ', '_')}_{sandbox_id}.png"
    return Response(content=png_bytes, media_type="image/png", headers={"Content-Disposition": f"attachment; filename={filename}"})

@router.post(":import", response_model=Sandbox, summary="Import a Sandbox from PNG")
async def import_sandbox(
    file: UploadFile = File(...),
    sandbox_store: SandboxStoreInterface = Depends(Service("sandbox_store")),
    snapshot_store: SnapshotStoreInterface = Depends(Service("snapshot_store")),
    persistence_service: PersistenceServiceInterface = Depends(Service("persistence_service"))
) -> Sandbox:
    if not file.filename or not file.filename.endswith(".png"):
        raise HTTPException(status_code=400, detail="Invalid file type. Please upload a .png file.")

    package_bytes = await file.read()
    
    try:
        manifest, data_files, png_bytes = await persistence_service.import_package(package_bytes)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=f"Invalid package: {e}")

    if manifest.package_type != PackageType.SANDBOX_ARCHIVE:
        raise HTTPException(status_code=400, detail=f"Invalid package type. Expected '{PackageType.SANDBOX_ARCHIVE.value}'.")

    try:
        sandbox_data_str = data_files.get(manifest.entry_point)
        if not sandbox_data_str:
            raise ValueError(f"Entry point file '{manifest.entry_point}' not found in package.")
        
        old_sandbox_data = json.loads(sandbox_data_str, object_hook=custom_json_decoder_object_hook)
        
        # --- [修复开始] ---
        # 1. 直接从导入的数据中获取完整的 `lore` 和 `definition`。
        imported_lore = old_sandbox_data.get('lore')
        imported_definition = old_sandbox_data.get('definition')

        # 2. 添加对旧格式的向后兼容支持。
        #    如果 `lore` 不存在，但旧的 `graph_collection` 存在，则用它来构建 lore。
        if not imported_lore and 'graph_collection' in old_sandbox_data:
            imported_lore = {"graphs": old_sandbox_data.get("graph_collection", {})}
        
        # 3. 如果 `definition` 不存在（非常旧的格式），则根据已有的 lore 和空的 moment 构建一个。
        if not imported_definition:
             imported_definition = {
                 "initial_lore": imported_lore or {},
                 "initial_moment": {}
             }

        if not imported_lore or not imported_definition:
            raise ValueError("Imported sandbox data is missing 'lore' or 'definition' fields.")
        # --- [修复结束] ---

        new_id = uuid.uuid4()
        # 使用修复后的、完整的数据来创建新的 Sandbox 对象
        new_sandbox = Sandbox(
            id=new_id,
            name=old_sandbox_data.get('name', 'Imported Sandbox'),
            definition=imported_definition, # 使用完整的 definition
            lore=imported_lore,             # 使用完整的 lore
            created_at=datetime.now(timezone.utc)
        )
        
        if new_sandbox.id in sandbox_store:
            raise HTTPException(status_code=409, detail=f"Conflict: A sandbox with the newly generated ID '{new_sandbox.id}' already exists.")

        recovered_snapshots = []
        old_to_new_snap_id = {}
        for filename in data_files:
            if filename.startswith("snapshots/"):
                old_id_str = filename.split('/')[1].split('.')[0]
                old_to_new_snap_id[UUID(old_id_str)] = uuid.uuid4()

        for filename, content_str in data_files.items():
            if filename.startswith("snapshots/"):
                old_snapshot_data = json.loads(content_str, object_hook=custom_json_decoder_object_hook)
                old_id = UUID(old_snapshot_data.get('id'))
                old_parent_id = old_snapshot_data.get('parent_snapshot_id')
                old_parent_id_uuid = UUID(old_parent_id) if old_parent_id else None
                new_parent_id = old_to_new_snap_id.get(old_parent_id_uuid) if old_parent_id_uuid else None
                
                new_snapshot = StateSnapshot(
                    id=old_to_new_snap_id[old_id],
                    sandbox_id=new_sandbox.id,
                    moment=old_snapshot_data.get('moment', {}),
                    parent_snapshot_id=new_parent_id,
                    triggering_input=old_snapshot_data.get('triggering_input', {}),
                    run_output=old_snapshot_data.get('run_output'),
                    created_at=datetime.fromisoformat(old_snapshot_data.get('created_at')) if old_snapshot_data.get('created_at') else datetime.now(timezone.utc)
                )
                recovered_snapshots.append(new_snapshot)
        
        if not recovered_snapshots:
            # [修改] 如果没有快照，我们应该基于 definition 创建一个创世快照，而不是报错。
            initial_moment = new_sandbox.definition.get("initial_moment", {})
            genesis_snapshot = StateSnapshot(
                id=uuid.uuid4(),
                sandbox_id=new_sandbox.id,
                moment=initial_moment
            )
            recovered_snapshots.append(genesis_snapshot)
            # 将 head 指向这个唯一的快照
            new_sandbox.head_snapshot_id = genesis_snapshot.id


        for snapshot in recovered_snapshots:
            await snapshot_store.save(snapshot)
        
        # [修改] 如果 head_snapshot_id 还没有被设置（例如上面创建了创世快照的情况），
        # 则需要从导入的数据中设置它。
        if not new_sandbox.head_snapshot_id:
            old_head_id = old_sandbox_data.get('head_snapshot_id')
            if old_head_id:
                new_sandbox.head_snapshot_id = old_to_new_snap_id.get(UUID(old_head_id))

        try:
            await persistence_service.save_sandbox_icon(str(new_sandbox.id), png_bytes)
            new_sandbox.icon_updated_at = datetime.now(timezone.utc)
        except Exception as e:
            logger.warning(f"Failed to set icon for newly imported sandbox {new_sandbox.id}: {e}")

        await sandbox_store.save(new_sandbox)
        
        logger.info(f"Successfully imported sandbox '{new_sandbox.name}' ({new_sandbox.id}) from PNG.")
        return new_sandbox
    except (ValidationError, ValueError, json.JSONDecodeError) as e:
        logger.warning(f"Failed to process package data for file {file.filename}: {e}", exc_info=True)
        raise HTTPException(status_code=422, detail=f"Failed to process package data: {str(e)}")
```

### engine.py
```
# plugins/core_engine/engine.py

import asyncio
import logging
from enum import Enum, auto
from typing import Dict, Any, Set, List, Optional
from collections import defaultdict
import traceback

from backend.core.contracts import Container, HookManager
from .contracts import (
    GraphDefinition, GenericNode, ExecutionContext, Sandbox,
    EngineStepStartContext, EngineStepEndContext,
    NodeExecutionStartContext, NodeExecutionSuccessContext, NodeExecutionErrorContext,
    BeforeConfigEvaluationContext, AfterMacroEvaluationContext,
    SnapshotStoreInterface,
    SandboxStoreInterface,
    RuntimeInterface, SubGraphRunner
)
from .dependency_parser import build_dependency_graph_async
from .registry import RuntimeRegistry
from .evaluation import build_evaluation_context, evaluate_data
from .state import (
    create_main_execution_context, 
    create_sub_execution_context, 
    create_next_snapshot
)
from .graph_resolver import GraphResolver
from .contracts import RuntimeInterface, SubGraphRunner

logger = logging.getLogger(__name__)

class NodeState(Enum):
    PENDING = auto()
    READY = auto()
    RUNNING = auto()
    SUCCEEDED = auto()
    FAILED = auto()
    SKIPPED = auto()

class GraphRun:
    def __init__(self, context: ExecutionContext, graph_def: GraphDefinition, dependencies: Dict[str, Set[str]]):
        self.context = context
        self.graph_def = graph_def
        if not self.graph_def:
            raise ValueError("GraphRun must be initialized with a valid GraphDefinition.")
        
        self.dependencies = dependencies
        
        self.node_map: Dict[str, GenericNode] = {n.id: n for n in self.graph_def.nodes}
        self.node_states: Dict[str, NodeState] = {}
        self.subscribers: Dict[str, Set[str]] = self._build_subscribers()
        self._detect_cycles()
        self._initialize_node_states()

    def _build_subscribers(self) -> Dict[str, Set[str]]:
        subscribers = defaultdict(set)
        for node_id, deps in self.dependencies.items():
            for dep_id in deps:
                subscribers[dep_id].add(node_id)
        return subscribers

    def _detect_cycles(self):
        path = set()
        visited = set()
        def visit(node_id):
            path.add(node_id)
            visited.add(node_id)
            for neighbour in self.dependencies.get(node_id, set()):
                if neighbour in path:
                    raise ValueError(f"Cycle detected involving node {neighbour}")
                if neighbour not in visited:
                    visit(neighbour)
            path.remove(node_id)
        for node_id in self.node_map:
            if node_id not in visited:
                visit(node_id)

    def _initialize_node_states(self):
        for node_id in self.node_map:
            if not self.dependencies.get(node_id):
                self.node_states[node_id] = NodeState.READY
            else:
                self.node_states[node_id] = NodeState.PENDING

    def get_node(self, node_id: str) -> GenericNode:
        return self.node_map[node_id]
    def get_node_state(self, node_id: str) -> NodeState:
        return self.node_states.get(node_id)
    def set_node_state(self, node_id: str, state: NodeState):
        self.node_states[node_id] = state
    def get_node_result(self, node_id: str) -> Dict[str, Any]:
        return self.context.node_states.get(node_id)
    def set_node_result(self, node_id: str, result: Dict[str, Any]):
        self.context.node_states[node_id] = result
    def get_nodes_in_state(self, state: NodeState) -> List[str]:
        return [nid for nid, s in self.node_states.items() if s == state]
    def get_dependencies(self, node_id: str) -> Set[str]:
        return self.dependencies.get(node_id, set())
    def get_subscribers(self, node_id: str) -> Set[str]:
        return self.subscribers.get(node_id, set())
    def get_execution_context(self) -> ExecutionContext:
        return self.context
    def get_final_node_states(self) -> Dict[str, Any]:
        return self.context.node_states

class ExecutionEngine(SubGraphRunner):
    def __init__(
        self,
        registry: RuntimeRegistry,
        container: Container,
        hook_manager: HookManager,
        num_workers: int = 5
    ):
        self.registry = registry
        self.container = container
        self.hook_manager = hook_manager
        self.num_workers = num_workers
        self.graph_resolver = GraphResolver()
        
    async def step(
        self, 
        sandbox: Sandbox,
        triggering_input: Dict[str, Any] = None
    ) -> Sandbox: 
        """
        在沙盒的最新状态上执行一步计算。
        """
        if triggering_input is None: triggering_input = {}
        
        # 1. 获取最新的快照
        if not sandbox.head_snapshot_id:
            raise ValueError(f"Sandbox '{sandbox.name}' has no head snapshot to step from.")
        snapshot_store: SnapshotStoreInterface = self.container.resolve("snapshot_store")
        initial_snapshot = snapshot_store.get(sandbox.head_snapshot_id)
        if not initial_snapshot:
            raise ValueError(f"Head snapshot '{sandbox.head_snapshot_id}' not found for sandbox '{sandbox.name}'.")

        await self.hook_manager.trigger(
            "engine_step_start",
            context=EngineStepStartContext(
                initial_snapshot=initial_snapshot,
                triggering_input=triggering_input
            )
        )
        
        # 2. 调用 state.py 中的工厂函数创建运行时上下文
        context = create_main_execution_context(
            snapshot=initial_snapshot,
            sandbox=sandbox,
            container=self.container,
            hook_manager=self.hook_manager,
            run_vars={"triggering_input": triggering_input}
        )

        await self.hook_manager.filter("before_graph_execution", context)
        
        # 3. 调用 GraphResolver 动态解析出本次要执行的图
        graph_collection_to_run = self.graph_resolver.resolve(context)
        main_graph_def = graph_collection_to_run.root.get("main")
        if not main_graph_def: raise ValueError("'main' graph not found in resolved collection.")
        
        # 4. 执行图
        final_node_states = await self._internal_execute_graph(main_graph_def, context)
        diagnostics_log = context.run_vars.get("diagnostics_log", [])
        
        # 5. 创建新快照和更新后的 Lore
        new_snapshot, updated_lore = await create_next_snapshot(
            context=context, 
            final_node_states=final_node_states, 
            triggering_input=triggering_input
        )
        
        # 6. 原子性地更新和保存状态
        # a. 保存新快照
        snapshot_store: SnapshotStoreInterface = self.container.resolve("snapshot_store")
        await snapshot_store.save(new_snapshot) 

        # b. 更新 Sandbox 对象的 Lore 和头指针
        sandbox.lore = updated_lore
        sandbox.head_snapshot_id = new_snapshot.id
        
        # c. 保存更新后的 Sandbox 对象
        # 使用通用接口进行类型提示
        sandbox_store: SandboxStoreInterface = self.container.resolve("sandbox_store")
        await sandbox_store.save(sandbox)
        
        await self.hook_manager.trigger(
            "snapshot_committed", 
            snapshot=new_snapshot,
            container=self.container
        )

        await self.hook_manager.trigger(
            "engine_step_end",
            context=EngineStepEndContext(final_snapshot=new_snapshot)
        )
        
        # 7. 返回更新后的 Sandbox 对象
        setattr(sandbox, '_temp_diagnostics_log', diagnostics_log)
        
        return sandbox

    async def execute_graph(
        self,
        graph_name: str,
        parent_context: ExecutionContext,
        inherited_inputs: Optional[Dict[str, Any]] = None
    ) -> Dict[str, Any]:
        graph_collection = self.graph_resolver.resolve(parent_context).root
        graph_def = graph_collection.get(graph_name)
        if not graph_def:
            raise ValueError(f"Graph '{graph_name}' not found.")
        
        sub_run_context = create_sub_execution_context(parent_context)

        return await self._internal_execute_graph(
            graph_def=graph_def,
            context=sub_run_context,
            inherited_inputs=inherited_inputs
        )

    async def _internal_execute_graph(self, graph_def: GraphDefinition, context: ExecutionContext, inherited_inputs: Optional[Dict[str, Any]] = None) -> Dict[str, Any]:
        
        dependencies = await build_dependency_graph_async(
            nodes=[node.model_dump() for node in graph_def.nodes],
            runtime_registry=self.registry
        )

        run = GraphRun(context=context, graph_def=graph_def, dependencies=dependencies)

        task_queue = asyncio.Queue()
        
        if inherited_inputs:
            for node_id, result in inherited_inputs.items():
                run.set_node_state(node_id, NodeState.SUCCEEDED)
                run.set_node_result(node_id, result)
        
        for node_id in run.get_nodes_in_state(NodeState.PENDING):
            dependencies = run.get_dependencies(node_id)
            if all(run.get_node_state(dep_id) == NodeState.SUCCEEDED for dep_id in dependencies):
                run.set_node_state(node_id, NodeState.READY)
        
        for node_id in run.get_nodes_in_state(NodeState.READY):
            task_queue.put_nowait(node_id)

        if task_queue.empty() and not any(s == NodeState.SUCCEEDED for s in run.node_states.values()):
            return {}

        workers = [
            asyncio.create_task(self._worker(f"worker-{i}", run, task_queue))
            for i in range(self.num_workers)
        ]
        
        await task_queue.join()

        for w in workers:
            w.cancel()
        
        await asyncio.gather(*workers, return_exceptions=True)
        
        final_states = {
            nid: run.get_node_result(nid)
            for nid, n in run.node_map.items()
            if run.get_node_result(nid) is not None
        }
        return final_states

    async def _worker(self, name: str, run: 'GraphRun', queue: asyncio.Queue):
        while True:
            try:
                node_id = await queue.get()
            except asyncio.CancelledError:
                break

            run.set_node_state(node_id, NodeState.RUNNING)
            try:
                node = run.get_node(node_id)
                context = run.get_execution_context()

                await self.hook_manager.trigger(
                    "node_execution_start",
                    context=NodeExecutionStartContext(node=node, execution_context=context)
                )
                output = await self._execute_node(node, context)
                if isinstance(output, dict) and "error" in output:
                    run.set_node_state(node_id, NodeState.FAILED)

                    await self.hook_manager.trigger(
                        "node_execution_error",
                        context=NodeExecutionErrorContext(
                            node=node,
                            execution_context=context,
                            exception=ValueError(output["error"])
                        )
                    )
                else:
                    run.set_node_state(node_id, NodeState.SUCCEEDED)

                    await self.hook_manager.trigger(
                        "node_execution_success",
                        context=NodeExecutionSuccessContext(
                            node=node,
                            execution_context=context,
                            result=output
                        )
                    )
                run.set_node_result(node_id, output)

            except Exception as e:
                error_message = f"Worker-level error for node {node_id}: {type(e).__name__}: {e}"
                import traceback
                traceback.print_exc()
                run.set_node_state(node_id, NodeState.FAILED)
                run.set_node_result(node_id, {"error": error_message})

                await self.hook_manager.trigger(
                    "node_execution_error",
                    context=NodeExecutionErrorContext(
                        node=node,
                        execution_context=context,
                        exception=e
                    )
                )
            self._process_subscribers(node_id, run, queue)
            queue.task_done()

    def _process_subscribers(self, completed_node_id: str, run: 'GraphRun', queue: asyncio.Queue):
        completed_node_state = run.get_node_state(completed_node_id)
        for sub_id in run.get_subscribers(completed_node_id):
            if run.get_node_state(sub_id) != NodeState.PENDING:
                continue
            if completed_node_state == NodeState.FAILED:
                run.set_node_state(sub_id, NodeState.SKIPPED)
                run.set_node_result(sub_id, {"status": "skipped", "reason": f"Upstream failure of node {completed_node_id}."})
                self._process_subscribers(sub_id, run, queue)
                continue
            dependencies = run.get_dependencies(sub_id)
            is_ready = all(
                (dep_id not in run.node_map) or (run.get_node_state(dep_id) == NodeState.SUCCEEDED)
                for dep_id in dependencies
            )
            if is_ready:
                run.set_node_state(sub_id, NodeState.READY)
                queue.put_nowait(sub_id)

    async def _execute_node(self, node: GenericNode, context: ExecutionContext) -> Dict[str, Any]:
        pipeline_state: Dict[str, Any] = {}
        if not node.run: return {}
        
        lock = context.shared.global_write_lock

        for i, instruction in enumerate(node.run):
            runtime_name = instruction.runtime
            try:
                eval_context = build_evaluation_context(context, pipe_vars=pipeline_state)
                

                config_to_process = instruction.config.copy()
                as_key = config_to_process.pop("as", None)

                config_to_process = await self.hook_manager.filter(
                    "before_config_evaluation",
                    config_to_process,
                    context=BeforeConfigEvaluationContext(
                        node=node,
                        execution_context=context,
                        instruction_config=config_to_process
                    )
                )

                runtime_instance: RuntimeInterface = self.registry.get_runtime(runtime_name)
                
                templates = {}
                template_fields = getattr(runtime_instance, 'template_fields', [])
                for field in template_fields:
                    if field in config_to_process:
                        templates[field] = config_to_process.pop(field)

                processed_config = await evaluate_data(config_to_process, eval_context, lock)

                processed_config = await self.hook_manager.filter(
                    "after_macro_evaluation",
                    processed_config,
                    context=AfterMacroEvaluationContext(
                        node=node,
                        execution_context=context,
                        evaluated_config=processed_config
                    )
                )

                if templates:
                    processed_config.update(templates)

                output = await runtime_instance.execute(
                    config=processed_config,
                    context=context,
                    subgraph_runner=self,
                    pipeline_state=pipeline_state,
                    node=node
                )
                
                if not isinstance(output, dict):
                    error_message = f"Runtime '{runtime_name}' did not return a dictionary. Got: {type(output).__name__}"
                    return {"error": error_message, "failed_step": i, "runtime": runtime_name}
                
                if as_key and isinstance(as_key, str):
                    # 如果 as_key 存在，将整个输出字典嵌套在它下面
                    pipeline_state[as_key] = output
                else:
                    # 如果 as_key 不存在，行为和以前一样，直接合并
                    pipeline_state.update(output)
            except Exception as e:
                import traceback
                traceback.print_exc()
                error_message = f"Failed at step {i+1} ('{runtime_name}'): {type(e).__name__}: {e}"
                return {"error": error_message, "failed_step": i, "runtime": runtime_name}
        return pipeline_state
```

### utils.py
```
# plugins/core_engine/utils.py

from backend.core.contracts import Container



class ServiceResolverProxy:
    """
    一个代理类，它包装一个 DI 容器，使其表现得像一个字典。
    这使得宏系统可以通过 `services.service_name` 语法懒加载并访问容器中的服务。
    """
    def __init__(self, container: Container):
        """
        :param container: 要代理的 DI 容器实例。
        """
        self._container = container
        # 创建一个简单的缓存，避免对同一个单例服务重复调用 resolve
        self._cache: dict = {}

    def __getitem__(self, name: str):
        """
        这是核心魔法所在。当代码执行 `proxy['service_name']` 时，此方法被调用。
        """
        if name in self._cache:
            return self._cache[name]
        
        service_instance = self._container.resolve(name)
        
        self._cache[name] = service_instance
        
        return service_instance

    def get(self, key: str, default=None):
        try:
            return self.__getitem__(key)
        except (ValueError, KeyError):
            return default

    def keys(self):
        return self._container._factories.keys()
    
    def __contains__(self, key: str) -> bool:
        return key in self._container._factories
```

### manifest.json
```
{
    "id": "core_engine",
    "name": "core_engine",
    "version": "1.0.0",
    "description": "Provides the graph parsing, scheduling, and execution engine.",
    "author": "Hevno Team",
    "backend": {
        "priority": 50
    }
}
```

### contracts.py
```
# plugins/core_engine/contracts.py 

from __future__ import annotations
import asyncio
from datetime import datetime, timezone
from typing import Any, Dict, List, Optional, Set, Callable
from uuid import UUID, uuid4
from pydantic import BaseModel, Field, RootModel, ConfigDict, field_validator
from abc import ABC, abstractmethod
from typing_extensions import Literal

# 从平台核心导入最基础的接口
from backend.core.contracts import HookManager

# --- 1. 核心持久化状态模型 ---

class RuntimeInstruction(BaseModel):
    runtime: str
    config: Dict[str, Any] = Field(default_factory=dict)

class GenericNode(BaseModel):
    id: str
    run: List[RuntimeInstruction]
    depends_on: Optional[List[str]] = Field(default=None)
    metadata: Dict[str, Any] = Field(default_factory=dict)

class GraphDefinition(BaseModel):
    nodes: List[GenericNode]
    metadata: Dict[str, Any] = Field(default_factory=dict)

class GraphCollection(RootModel[Dict[str, GraphDefinition]]):
    @field_validator('root')
    @classmethod
    def check_main_graph_exists(cls, v: Dict[str, GraphDefinition]) -> Dict[str, GraphDefinition]:
        if "main" not in v:
            raise ValueError("A 'main' graph must be defined as the entry point.")
        return v
        
class StateSnapshot(BaseModel):
    id: UUID = Field(default_factory=uuid4)
    sandbox_id: UUID
    moment: Dict[str, Any] = Field(default_factory=dict)
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    parent_snapshot_id: Optional[UUID] = None
    triggering_input: Dict[str, Any] = Field(default_factory=dict)
    run_output: Optional[Dict[str, Any]] = None
    
    model_config = ConfigDict(
        frozen=True,
        arbitrary_types_allowed=True 
    )

class Sandbox(BaseModel):
    id: UUID = Field(default_factory=uuid4)
    name: str
    head_snapshot_id: Optional[UUID] = None
    definition: Dict[str, Any] = Field(...)
    lore: Dict[str, Any] = Field(default_factory=dict)
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    icon_updated_at: Optional[datetime] = None
    
    model_config = ConfigDict(
        arbitrary_types_allowed=True
    )

# --- 2. 核心运行时上下文模型 (确保有 arbitrary_types_allowed=True) ---
class SharedContext(BaseModel):
    definition_state: Dict[str, Any] = Field(default_factory=dict)
    lore_state: Dict[str, Any] = Field(default_factory=dict)
    moment_state: Dict[str, Any] = Field(default_factory=dict)
    session_info: Dict[str, Any]
    global_write_lock: asyncio.Lock
    services: Any
    model_config = {"arbitrary_types_allowed": True}

class ExecutionContext(BaseModel):
    node_states: Dict[str, Any] = Field(default_factory=dict)
    run_vars: Dict[str, Any] = Field(default_factory=dict)
    shared: SharedContext
    initial_snapshot: StateSnapshot
    hook_manager: HookManager
    model_config = {"arbitrary_types_allowed": True}


# --- 3. 系统事件契约 (用于钩子) ---
class NodeContext(BaseModel):
    node: GenericNode
    execution_context: ExecutionContext
    model_config = ConfigDict(arbitrary_types_allowed=True)

class EngineStepStartContext(BaseModel):
    initial_snapshot: StateSnapshot
    triggering_input: Dict[str, Any]
    model_config = ConfigDict(arbitrary_types_allowed=True)

class EngineStepEndContext(BaseModel):
    final_snapshot: StateSnapshot
    model_config = ConfigDict(arbitrary_types_allowed=True)

class NodeExecutionStartContext(NodeContext): pass
class NodeExecutionSuccessContext(NodeContext):
    result: Dict[str, Any]
class NodeExecutionErrorContext(NodeContext):
    exception: Exception

class BeforeConfigEvaluationContext(NodeContext):
    instruction_config: Dict[str, Any]
class AfterMacroEvaluationContext(NodeContext):
    evaluated_config: Dict[str, Any]

class BeforeSnapshotCreateContext(BaseModel):
    snapshot_data: Dict[str, Any]
    execution_context: ExecutionContext
    model_config = ConfigDict(arbitrary_types_allowed=True)

class ResolveNodeDependenciesContext(BaseModel):
    node: GenericNode
    auto_inferred_deps: Set[str]


# --- 4. 核心服务接口契约 (关键修改部分) ---

class SubGraphRunner(ABC):
    """定义执行子图能力的抽象接口。"""
    @abstractmethod
    async def execute_graph(
        self,
        graph_name: str,
        parent_context: ExecutionContext,
        inherited_inputs: Optional[Dict[str, Any]] = None
    ) -> Dict[str, Any]:
        pass

class RuntimeInterface(ABC):
    """定义所有运行时必须实现的接口。"""
    @abstractmethod
    async def execute(
        self,
        config: Dict[str, Any],
        context: ExecutionContext,
        subgraph_runner: Optional[SubGraphRunner] = None,
        pipeline_state: Optional[Dict[str, Any]] = None
    ) -> Dict[str, Any]:
        pass

    @classmethod
    def get_dependency_config(cls) -> Dict[str, Any]:
        """允许运行时声明其依赖解析行为。"""
        return {}



MutationType = Literal["UPSERT", "DELETE", "LIST_APPEND"]
MutationMode = Literal["DIRECT", "SNAPSHOT"]

class Mutation(BaseModel):
    """定义一个独立的、原子性的修改操作。"""
    type: MutationType
    path: str = Field(..., description="从沙盒根开始的完整资源路径，例如 'lore/graphs/main'。")
    value: Any = Field(None, description="对于 UPSERT 和 LIST_APPEND 操作是必需的。")
    mutation_mode: MutationMode = Field(
        default="DIRECT",
        description="仅当路径以'moment/'开头时有效。'DIRECT'直接修改，'SNAPSHOT'创建新历史。"
    )

class MutateResourceRequest(BaseModel):
    mutations: List[Mutation]

class MutateResourceResponse(BaseModel):
    sandbox_id: UUID
    head_snapshot_id: Optional[UUID]

class ResourceQueryRequest(BaseModel):
    paths: List[str]

class ResourceQueryResponse(BaseModel):
    results: Dict[str, Any]

class EditorUtilsServiceInterface(ABC):
    """
    定义了用于安全地、原子性地执行数据修改的核心工具接口。
    """
    @abstractmethod
    async def execute_mutations(self, sandbox: Sandbox, mutations: List[Mutation]) -> Sandbox:
        """
        原子性地执行一批修改操作，并根据操作类型智能地选择
        直接修改沙盒、直接修改快照或创建新快照。
        """
        raise NotImplementedError
    
    @abstractmethod
    async def execute_queries(self, sandbox: Sandbox, paths: List[str]) -> Dict[str, Any]:
        """
        根据提供的路径列表，批量从沙盒中获取数据。
        """
        raise NotImplementedError


class MacroEvaluationServiceInterface(ABC):
    """为宏求值逻辑定义服务接口。"""
    @abstractmethod
    def build_context(
        self,
        exec_context: ExecutionContext,
        pipe_vars: Optional[Dict[str, Any]] = None
    ) -> Dict[str, Any]:
        raise NotImplementedError

    @abstractmethod
    async def evaluate(
        self,
        data: Any,
        eval_context: Dict[str, Any],
        lock: asyncio.Lock
    ) -> Any:
        raise NotImplementedError

class ExecutionEngineInterface(ABC):
    """定义执行引擎的核心接口。"""
    @abstractmethod
    async def step(self, sandbox: 'Sandbox', triggering_input: Dict[str, Any] = None) -> 'Sandbox':
        """
        在沙盒的最新状态上执行一步计算，并返回更新后的、已持久化的沙盒对象。
        """
        raise NotImplementedError

class SnapshotStoreInterface(ABC):
    """定义快照存储的核心接口。"""
    @abstractmethod
    async def save(self, snapshot: 'StateSnapshot') -> None:
        """异步保存一个快照。"""
        raise NotImplementedError
    
    @abstractmethod
    def get(self, snapshot_id: UUID) -> Optional['StateSnapshot']:
        """同步从缓存获取一个快照。"""
        raise NotImplementedError
    
    @abstractmethod
    async def find_by_sandbox(self, sandbox_id: UUID) -> List['StateSnapshot']:
        """异步查找并加载属于特定沙盒的所有快照。"""
        raise NotImplementedError

    @abstractmethod
    async def delete(self, snapshot_id: UUID) -> None:
        """异步删除一个指定的快照。"""
        raise NotImplementedError

    @abstractmethod
    async def delete_all_for_sandbox(self, sandbox_id: UUID) -> None:
        """异步删除属于特定沙盒的所有快照。"""
        raise NotImplementedError

    @abstractmethod
    def clear(self) -> None:
        """清除存储（在持久化存储中可能为空操作）。"""
        raise NotImplementedError

class SandboxStoreInterface(ABC):
    """定义沙盒存储的核心接口。"""
    @abstractmethod
    async def initialize(self) -> None:
        """异步初始化存储，例如从磁盘预加载。"""
        raise NotImplementedError

    @abstractmethod
    async def save(self, sandbox: 'Sandbox') -> None:
        """异步保存一个沙盒。"""
        raise NotImplementedError

    @abstractmethod
    def get(self, key: UUID) -> Optional['Sandbox']:
        """同步从缓存中获取一个沙盒。"""
        raise NotImplementedError

    @abstractmethod
    async def delete(self, key: UUID) -> None:
        """异步删除一个沙盒及其所有相关数据。"""
        raise NotImplementedError

    @abstractmethod
    def values(self) -> List['Sandbox']:
        """同步获取所有缓存的沙盒。"""
        raise NotImplementedError

    @abstractmethod
    def __contains__(self, key: UUID) -> bool:
        """检查一个沙盒是否存在于缓存中。"""
        raise NotImplementedError

    def __getitem__(self, key: UUID) -> 'Sandbox':
        """允许通过字典语法访问。"""
        sandbox = self.get(key)
        if sandbox is None:
            raise KeyError(key)
        return sandbox

# --- 5. API 契约 ---

class StepDiagnostics(BaseModel):
    """用于承载本次执行的诊断信息。"""
    execution_time_ms: float
    detailed_log: Optional[List[Dict[str, Any]]] = Field(
        default=None,
        description="一个包含本次 step 执行期间所有详细诊断事件的列表。"
    )

class StepResponse(BaseModel):
    """/step 端点的标准响应信封。"""
    status: Literal["COMPLETED", "ERROR"]
    sandbox: Sandbox
    diagnostics: Optional[StepDiagnostics] = None
    error_message: Optional[str] = None
```

### evaluation_service.py
```
# plugins/core_engine/evaluation_service.py
import asyncio
from typing import Any, Dict, Optional

from .contracts import ExecutionContext, MacroEvaluationServiceInterface
from .evaluation import build_evaluation_context as build_context_impl
from .evaluation import evaluate_data as evaluate_data_impl

class MacroEvaluationService(MacroEvaluationServiceInterface):
    """
    实现了宏求值接口，作为对 evaluation.py 中复杂逻辑的封装。
    """
    def build_context(
        self,
        exec_context: ExecutionContext,
        pipe_vars: Optional[Dict[str, Any]] = None
    ) -> Dict[str, Any]:
        """
        代理调用原始的 build_evaluation_context 函数。
        """
        return build_context_impl(exec_context, pipe_vars)

    async def evaluate(
        self,
        data: Any,
        eval_context: Dict[str, Any],
        lock: asyncio.Lock
    ) -> Any:
        """
        代理调用原始的 evaluate_data 函数。
        """
        return await evaluate_data_impl(data, eval_context, lock)
```

### dependency_parser.py
```
# plugins/core_engine/dependency_parser.py
import re
from typing import Set, Dict, Any, List
import asyncio

from plugins.core_engine.registry import RuntimeRegistry

NODE_DEP_REGEX = re.compile(r'nodes\.([^.]+)')

def extract_dependencies_from_string(s: str) -> Set[str]:
    if not isinstance(s, str):
        return set()
    if '{{' in s and '}}' in s and 'nodes.' in s:
        return set(NODE_DEP_REGEX.findall(s))
    return set()

def extract_dependencies_from_value(value: Any) -> Set[str]:
    deps = set()
    if isinstance(value, str):
        deps.update(extract_dependencies_from_string(value))
    elif isinstance(value, list):
        for item in value:
            deps.update(extract_dependencies_from_value(item))
    elif isinstance(value, dict):
        for k, v in value.items():
            deps.update(extract_dependencies_from_value(k))
            deps.update(extract_dependencies_from_value(v))
    return deps

async def build_dependency_graph_async(
    nodes: List[Dict[str, Any]],
    runtime_registry: RuntimeRegistry 
) -> Dict[str, Set[str]]:
    dependency_map: Dict[str, Set[str]] = {}

    for node_dict in nodes:
        node_id = node_dict['id']
        auto_inferred_deps = set()

        for instruction in node_dict.get('run', []):
            runtime_name = instruction.get('runtime')
            if not runtime_name:
                continue

            # 1. 获取运行时类
            try:
                runtime_class = runtime_registry.get_runtime_class(runtime_name)
            except ValueError as e:
                # 如果运行时不存在，最好是抛出错误而不是静默失败
                raise ValueError(f"Error in node '{node_id}': {e}")

            # 2. 获取该运行时的依赖解析配置
            dep_config = runtime_class.get_dependency_config()
            ignore_fields = set(dep_config.get('ignore_fields', []))

            instruction_config = instruction.get('config', {})

            # 3. 根据配置进行智能扫描
            for field, value in instruction_config.items():
                if field in ignore_fields:
                    continue  # 跳过被忽略的字段
                
                # 对 value 进行递归扫描
                field_deps = extract_dependencies_from_value(value)
                auto_inferred_deps.update(field_deps)

        explicit_deps = set(node_dict.get('depends_on') or [])
        
        dependency_map[node_id] = auto_inferred_deps.union(explicit_deps)
    
    return dependency_map
```

### state.py
```
# plugins/core_engine/state.py 

from __future__ import annotations
import asyncio
import json
from copy import deepcopy
from uuid import UUID
from typing import Dict, Any, List, Optional, Tuple

from fastapi import Request
from pydantic import ValidationError

from backend.core.contracts import HookManager, Container
from .contracts import (
    Sandbox, 
    StateSnapshot, 
    ExecutionContext, 
    SharedContext,
    BeforeSnapshotCreateContext,
    GraphCollection
)
from backend.core.utils import DotAccessibleDict, unwrap_dot_accessible_dicts
from .utils import ServiceResolverProxy 

# --- Section 1: 状态存储类 ---

class SnapshotStore:
    """
    一个简单的内存快照存储。
    它操作从 contracts.py 导入的 StateSnapshot 模型。
    """
    def __init__(self):
        self._store: Dict[UUID, StateSnapshot] = {}

    def save(self, snapshot: StateSnapshot):
        if snapshot.id in self._store:
            pass
        self._store[snapshot.id] = snapshot

    def get(self, snapshot_id: UUID) -> Optional[StateSnapshot]:
        return self._store.get(snapshot_id)

    def find_by_sandbox(self, sandbox_id: UUID) -> List[StateSnapshot]:
        return sorted(
            [s for s in self._store.values() if s.sandbox_id == sandbox_id],
            key=lambda s: s.created_at
        )

    def clear(self):
        self._store.clear()


# --- Section 2: 核心上下文与快照的工厂/助手函数  ---

def create_main_execution_context(
    snapshot: StateSnapshot, 
    sandbox: Sandbox, 
    container: Container,
    hook_manager: HookManager, 
    run_vars: Dict[str, Any] = None
) -> ExecutionContext:
    """
    从持久化的 Snapshot 和 Sandbox 中，为一次安全的、隔离的执行准备运行时上下文。
    使用深拷贝来防止执行过程意外修改原始状态。
    """
    # 在 run_vars 中添加一个 diagnostics_log 列表
    initial_run_vars = {
        "triggering_input": {},
        "diagnostics_log": []
    }
    if run_vars:
        initial_run_vars.update(run_vars)
        
    shared_context = SharedContext(
        definition_state=deepcopy(sandbox.definition),
        lore_state=deepcopy(sandbox.lore),
        moment_state=deepcopy(snapshot.moment),
        
        session_info={
            "start_time": snapshot.created_at,
            "turn_count": 0
        },
        global_write_lock=asyncio.Lock(),
        services=DotAccessibleDict(ServiceResolverProxy(container))
    )
    return ExecutionContext(
        shared=shared_context,
        initial_snapshot=snapshot,
        run_vars=initial_run_vars,
        hook_manager=hook_manager
    )


def create_sub_execution_context(
    parent_context: ExecutionContext, 
    run_vars: Dict[str, Any] = None
) -> ExecutionContext:
    return ExecutionContext(
        shared=parent_context.shared,
        initial_snapshot=parent_context.initial_snapshot,
        run_vars=run_vars or {},
        hook_manager=parent_context.hook_manager
    )

async def create_next_snapshot(
    context: ExecutionContext,
    final_node_states: Dict[str, Any],
    triggering_input: Dict[str, Any]
) -> Tuple[StateSnapshot, Dict[str, Any]]: 
    """
    从执行完毕的上下文中，创建新的 StateSnapshot 并分离出更新后的 Lore。
    """
    final_moment_state = context.shared.moment_state
    final_lore_state = context.shared.lore_state
    
    # 调用从 backend.core.utils 导入的官方函数
    unwrapped_moment = unwrap_dot_accessible_dicts(final_moment_state)
    unwrapped_lore = unwrap_dot_accessible_dicts(final_lore_state)
    unwrapped_node_states = unwrap_dot_accessible_dicts(final_node_states)
    
    snapshot_data = {
        "sandbox_id": context.initial_snapshot.sandbox_id,
        "moment": unwrapped_moment, 
        "parent_snapshot_id": context.initial_snapshot.id,
        "run_output": unwrapped_node_states,
        "triggering_input": triggering_input,
    }

    filtered_snapshot_data = await context.hook_manager.filter(
        "before_snapshot_create",
        snapshot_data,
        context=BeforeSnapshotCreateContext(
            snapshot_data=snapshot_data,
            execution_context=context
        )
    )
    
    new_snapshot = StateSnapshot.model_validate(filtered_snapshot_data)
    
    return (new_snapshot, unwrapped_lore)



# --- Section 3: FastAPI 依赖注入函数  ---

def get_sandbox_store(request: Request) -> Dict[UUID, Sandbox]:
    return request.app.state.sandbox_store

def get_snapshot_store(request: Request) -> SnapshotStore:
    return request.app.state.snapshot_store
```

### runtimes/flow_runtimes.py
```
# plugins/core_engine/runtimes/flow_runtimes.py
import asyncio
from typing import Dict, Any, List, Optional

from backend.core.utils import DotAccessibleDict
from ..contracts import RuntimeInterface, SubGraphRunner, ExecutionContext
from ..evaluation import evaluate_data, evaluate_expression, build_evaluation_context


class ExecuteRuntime(RuntimeInterface):
    """
    system.execute: 对一个字符串形式的宏代码进行二次求值和执行。
    作为宏系统的终极“逃生舱口”。
    """
    # 1. 修改方法签名以接收 pipeline_state
    async def execute(
        self, 
        config: Dict[str, Any], 
        context: ExecutionContext, 
        pipeline_state: Optional[Dict[str, Any]] = None, 
        **kwargs
    ) -> Dict[str, Any]:
        code_to_execute = config.get("code")

        if not isinstance(code_to_execute, str):
            return {"output": code_to_execute}

        # 2. 将接收到的 pipeline_state 传递给 build_evaluation_context
        eval_context = build_evaluation_context(context, pipe_vars=pipeline_state)
        lock = context.shared.global_write_lock
        result = await evaluate_expression(code_to_execute, eval_context, lock)
        return {"output": result}

class CallRuntime(RuntimeInterface):
    """
    system.flow.call: 调用并执行一个可复用的子图。
    """
    async def execute(self, config: Dict[str, Any], context: ExecutionContext, subgraph_runner: Optional[SubGraphRunner] = None, **kwargs) -> Dict[str, Any]:
        if not subgraph_runner:
            raise ValueError("CallRuntime requires a SubGraphRunner.")
            
        graph_name = config.get("graph")
        if not graph_name:
            raise ValueError("CallRuntime requires a 'graph' name in its config.")
            
        using_inputs = config.get("using", {})
        
        inherited_inputs = {
            placeholder_name: {"output": value}
            for placeholder_name, value in using_inputs.items()
        }

        subgraph_results = await subgraph_runner.execute_graph(
            graph_name=graph_name,
            parent_context=context,
            inherited_inputs=inherited_inputs
        )
        
        return {"output": subgraph_results}


class MapRuntime(RuntimeInterface):
    """
    system.flow.map: 对一个列表进行并行迭代，为每个元素执行一次子图。
    """
    template_fields = ["using", "collect"]

    @classmethod
    def get_dependency_config(cls) -> Dict[str, Any]:
        """
        我们告诉解析器，'using' 和 'collect' 字段包含的 'nodes.xxx' 
        是在子图的上下文中，不应被视为主图的依赖。
        """
        return {
            "ignore_fields": ["using", "collect"]
        }

    async def execute(self, config: Dict[str, Any], context: ExecutionContext, subgraph_runner: Optional[SubGraphRunner] = None, **kwargs) -> Dict[str, Any]:
        if not subgraph_runner:
            raise ValueError("MapRuntime requires a SubGraphRunner.")

        list_to_iterate = config.get("list")
        graph_name = config.get("graph")
        using_template = config.get("using", {})
        collect_template = config.get("collect")

        if not isinstance(list_to_iterate, list):
            raise TypeError(f"MapRuntime 'list' field must be a list, got {type(list_to_iterate).__name__}.")
        if not graph_name:
            raise ValueError("MapRuntime requires a 'graph' name in its config.")

        tasks = []
        base_eval_context = build_evaluation_context(context)
        lock = context.shared.global_write_lock

        for index, item in enumerate(list_to_iterate):
            using_eval_context = {
                **base_eval_context,
                "source": DotAccessibleDict({"item": item, "index": index})
            }
            
            evaluated_using = await evaluate_data(using_template, using_eval_context, lock)
            inherited_inputs = {
                placeholder: {"output": value}
                for placeholder, value in evaluated_using.items()
            }
            
            task = asyncio.create_task(
                subgraph_runner.execute_graph(
                    graph_name=graph_name,
                    parent_context=context,
                    inherited_inputs=inherited_inputs
                )
            )
            tasks.append(task)
        
        subgraph_results: List[Dict[str, Any]] = await asyncio.gather(*tasks)
        
        if collect_template is not None:
            collected_outputs = []
            for result in subgraph_results:
                collect_eval_context = build_evaluation_context(context)
                collect_eval_context["nodes"] = DotAccessibleDict(result)
                
                collected_value = await evaluate_data(collect_template, collect_eval_context, lock)
                collected_outputs.append(collected_value)
            
            return {"output": collected_outputs}
        else:
            return {"output": subgraph_results}
```

### runtimes/__init__.py
```

```

### runtimes/io_runtimes.py
```
# plugins/core_engine/runtimes/io_runtimes.py
import logging
from typing import Dict, Any

from ..contracts import ExecutionContext, RuntimeInterface

logger = logging.getLogger(__name__)

class InputRuntime(RuntimeInterface):
    """
    system.io.input: 将一个静态或动态生成的值注入到节点管道中。
    作为节点内部数据处理流程最基础的“数据源”。
    """
    async def execute(self, config: Dict[str, Any], context: ExecutionContext, **kwargs) -> Dict[str, Any]:
        return {"output": config.get("value")}

class LogRuntime(RuntimeInterface):
    """
    system.io.log: 向后端日志系统输出一条消息。
    提供一个标准化的、用于调试图执行流程的工具。
    """
    LOG_LEVELS = {
        "debug": logging.DEBUG,
        "info": logging.INFO,
        "warning": logging.WARNING,
        "error": logging.ERROR,
        "critical": logging.CRITICAL,
    }

    async def execute(self, config: Dict[str, Any], context: ExecutionContext, **kwargs) -> Dict[str, Any]:
        message = config.get("message")
        if message is None:
            raise ValueError("LogRuntime requires a 'message' field in its config.")

        level_str = config.get("level", "info").lower()
        level = self.LOG_LEVELS.get(level_str, logging.INFO)
        
        # 使用一个专用的 logger，可以方便地在日志配置中控制其输出
        runtime_logger = logging.getLogger("hevno.runtime.log")
        runtime_logger.log(level, str(message))
        
        return {}
```

### runtimes/data_runtimes.py
```
# plugins/core_engine/runtimes/data_runtimes.py
import json
import re
import xml.etree.ElementTree as ET
from typing import Dict, Any, List

from ..contracts import ExecutionContext, RuntimeInterface

# --- XML to Dict Helper ---
def etree_to_dict(t):
    d = {t.tag: {} if t.attrib else None}
    children = list(t)
    if children:
        dd = {}
        for dc in map(etree_to_dict, children):
            for k, v in dc.items():
                if k in dd:
                    if not isinstance(dd[k], list):
                        dd[k] = [dd[k]]
                    dd[k].append(v)
                else:
                    dd[k] = v
        d = {t.tag: dd}
    if t.attrib:
        d[t.tag].update(('@' + k, v) for k, v in t.attrib.items())
    if t.text:
        text = t.text.strip()
        if children or t.attrib:
            if text:
                d[t.tag]['#text'] = text
        else:
            d[t.tag] = text
    return d

class FormatRuntime(RuntimeInterface):
    """
    system.data.format: 将列表或字典数据源格式化为单一字符串。
    """
    async def execute(self, config: Dict[str, Any], context: ExecutionContext, **kwargs) -> Dict[str, Any]:
        items = config.get("items")
        template = config.get("template")
        if items is None or template is None:
            raise ValueError("FormatRuntime requires 'items' and 'template' in its config.")
        
        formatted_parts = []
        if isinstance(items, list):
            joiner = config.get("joiner", "\n")
            for item in items:
                try:
                    # 支持 item 是字典的情况 (e.g., template="{item[name]}")
                    if isinstance(item, dict):
                         formatted_parts.append(template.format(item=item, **item))
                    else:
                         formatted_parts.append(template.format(item=item))
                except (KeyError, IndexError, TypeError) as e:
                    raise ValueError(f"Error formatting item with template: {e}. Item: {item}, Template: '{template}'")
            return {"output": joiner.join(formatted_parts)}
        
        elif isinstance(items, dict):
            for key, value in items.items():
                formatted_parts.append(template.format(key=key, value=value))
            # 对于字典，通常没有 joiner 的概念，但为了一致性，我们也允许它
            joiner = config.get("joiner", "\n")
            return {"output": joiner.join(formatted_parts)}
            
        else:
            raise TypeError(f"FormatRuntime 'items' field must be a list or a dict, not {type(items).__name__}.")

class ParseRuntime(RuntimeInterface):
    """
    system.data.parse: 将字符串（如LLM的输出）解析为结构化的数据对象。
    """
    async def execute(self, config: Dict[str, Any], context: ExecutionContext, **kwargs) -> Dict[str, Any]:
        text = config.get("text")
        fmt = config.get("format")
        strict = config.get("strict", False)

        if text is None or fmt is None:
            raise ValueError("ParseRuntime requires 'text' and 'format' in its config.")
        
        try:
            if fmt == "json":
                return {"output": json.loads(text)}
            
            elif fmt == "xml":
                root = ET.fromstring(text)
                selector = config.get("selector")
                if selector:
                    # 简化版 XPath, 只支持标签名
                    found_element = root.find(selector.strip('/'))
                    if found_element is not None:
                        return {"output": found_element.text}
                    else:
                        return {"output": None}
                else:
                    return {"output": etree_to_dict(root)}
            
            else:
                raise ValueError(f"Unsupported format '{fmt}'. Supported formats are 'json', 'xml'.")

        except Exception as e:
            if strict:
                raise e
            return {"output": {"error": str(e), "original_text": text}}

class RegexRuntime(RuntimeInterface):
    """
    system.data.regex: 对输入文本执行正则表达式匹配，并提取匹配内容。
    """
    async def execute(self, config: Dict[str, Any], context: ExecutionContext, **kwargs) -> Dict[str, Any]:
        text = str(config.get("text", ""))
        pattern = config.get("pattern")
        mode = config.get("mode", "search")

        if not pattern:
            raise ValueError("RegexRuntime requires a 'pattern' in its config.")
        
        if mode == "search":
            match = re.search(pattern, text)
            if not match:
                return {"output": None}
            if match.groupdict():
                return {"output": match.groupdict()}
            return {"output": match.group(0)}
        
        elif mode == "find_all":
            matches = re.findall(pattern, text)
            return {"output": matches}
        
        else:
            raise ValueError(f"Invalid mode '{mode}'. Supported modes are 'search', 'find_all'.")
```

# Directory: plugins/core_api

### system_router.py
```
# plugins/core_api/system_router.py

import json
import logging
from pathlib import Path
from typing import List, Dict, Any

from fastapi import APIRouter, HTTPException, Depends
from fastapi.responses import FileResponse

# 从后端核心导入依赖
from backend.core.dependencies import Service
from backend.core.contracts import HookManager

# 获取这个模块的 logger 实例
logger = logging.getLogger(__name__)

# --- 路径计算 (保持健壮) ---
# __file__ -> .../project_root/plugins/core_api/system_router.py
# .parent -> .../project_root/plugins/core_api
# .parent.parent -> .../project_root/plugins
PLUGINS_DIR = Path(__file__).resolve().parent.parent

# --- 路由器 1: 用于 /api/... (平台元信息API) ---
# 我们将所有相关的API都聚合到这个路由器下
system_api_router = APIRouter(
    prefix="/api",
    tags=["System Platform API"]
)

@system_api_router.get("/plugins/manifest", response_model=List[Dict[str, Any]], summary="Get All Plugin Manifests")
async def get_all_plugins_manifest():
    """
    Retrieves the manifest.json content for all discovered plugins.
    This provides a central way for the frontend to understand what capabilities
    are available on the backend.
    """
    if not PLUGINS_DIR.is_dir():
        return []

    manifests = []
    for plugin_path in PLUGINS_DIR.iterdir():
        if not plugin_path.is_dir() or plugin_path.name.startswith(('__', '.')):
            continue
        
        manifest_file = plugin_path / "manifest.json"
        if manifest_file.is_file():
            try:
                with open(manifest_file, 'r', encoding='utf-8') as f:
                    manifests.append(json.load(f))
            except json.JSONDecodeError:
                logger.warning(f"Could not parse manifest.json for plugin: {plugin_path.name}")
                pass
    return manifests

@system_api_router.get("/system/hooks/manifest", response_model=Dict[str, List[str]], summary="Get Backend Hooks Manifest")
async def get_backend_hooks_manifest(
    hook_manager: HookManager = Depends(Service("hook_manager"))
):
    """
    Retrieves a list of all hook names that have been registered on the backend.
    Useful for frontend diagnostics and understanding event flow.
    """
    # ._hooks is an implementation detail, but for a diagnostics endpoint, it's acceptable.
    return {"hooks": list(hook_manager._hooks.keys())}


# --- 路由器 2: 用于 /plugins/... (服务前端插件的静态资源) ---
# 这个路由器没有前缀，因为它需要匹配根URL路径
frontend_assets_router = APIRouter(
    tags=["System Frontend Assets"]
)

@frontend_assets_router.get("/plugins/{plugin_id}/{resource_path:path}")
async def serve_plugin_resource(plugin_id: str, resource_path: str):
    """
    Dynamically serves static assets (like JS, CSS, images) from any plugin's
    directory. This is crucial for enabling frontend components of plugins.
    """
    logger.info(f"[ASSET_SERVER] Request for: /plugins/{plugin_id}/{resource_path}")
    
    try:
        if ".." in plugin_id or "\\" in plugin_id:
            logger.warning(f"[ASSET_SERVER] Invalid plugin ID detected: {plugin_id}")
            raise HTTPException(status_code=400, detail="Invalid plugin ID.")
        
        plugin_base_path = (PLUGINS_DIR / plugin_id).resolve()
        target_file_path = (plugin_base_path / resource_path).resolve()

        # Security check: Ensure the resolved path is still within the plugin's directory
        is_safe = str(target_file_path).startswith(str(plugin_base_path))
        if not is_safe:
            logger.warning(f"[ASSET_SERVER] Forbidden access attempt: {plugin_id}/{resource_path}")
            raise HTTPException(status_code=403, detail="Forbidden: Access outside of plugin directory is not allowed.")

        if not target_file_path.is_file():
            raise HTTPException(status_code=404, detail=f"Resource '{resource_path}' not found in plugin '{plugin_id}'.")

        logger.info(f"[ASSET_SERVER] Success! Serving file: {target_file_path}")
        return FileResponse(target_file_path)

    except HTTPException as e:
        raise e
    except Exception as e:
        logger.error(f"[ASSET_SERVER] Error serving plugin resource '{plugin_id}/{resource_path}': {e}", exc_info=True)
        raise HTTPException(status_code=500, detail="Internal server error while serving plugin resource.")
```

### __init__.py
```
# plugins/core_api/__init__.py
import logging
from typing import List
from fastapi import APIRouter

from backend.core.contracts import Container, HookManager


logger = logging.getLogger(__name__)


async def provide_own_routers(routers: List[APIRouter]) -> List[APIRouter]:
    """
    Hook implementation: Adds this plugin's routers to the application's collection.
    By importing inside the function, we ensure the router modules are executed
    only when the application is ready to collect them.
    """
    logger.info("--> [core_api] 'collect_api_routers' hook triggered. Importing routers...")
    
    # 【重点】只从一个文件中导入，不再需要 base_router
    from .system_router import system_api_router, frontend_assets_router
    
    logger.debug(f"[core_api] Appending system_api_router (prefix='{system_api_router.prefix}', {len(system_api_router.routes)} routes)")
    logger.debug(f"[core_api] Appending frontend_assets_router (prefix='{frontend_assets_router.prefix}', {len(frontend_assets_router.routes)} routes)")
    
    routers.append(system_api_router)
    routers.append(frontend_assets_router)
    
    logger.info("--> [core_api] Routers have been provided.")
    return routers

# --- Main Registration Function ---
def register_plugin(container: Container, hook_manager: HookManager):
    """
    Registers the core_api plugin. Its sole backend purpose is to provide
    platform-level API endpoints for introspection and asset serving.
    """
    logger.info("--> 正在注册 [core_api] 插件...")
    
    hook_manager.add_implementation(
        "collect_api_routers", 
        provide_own_routers, 
        priority=100,  # High priority to ensure system routes are available
        plugin_name="core_api"
    )
    logger.info("插件 [core_api] 注册成功。'collect_api_routers' hook has been implemented.")
```

### manifest.json
```
{
    "id": "core_api",
    "name": "core_api",
    "version": "1.0.0",
    "description": "Provides the core RESTful API endpoints and the system reporting auditor.",
    "author": "Hevno Team",
    "backend": {
        "priority": 100
    }
}
```

# Directory: plugins/core_codex

### invoke_runtime.py
```
# plugins/core_codex/invoke_runtime.py

import asyncio
import logging
import re
from typing import Dict, Any, List, Optional, Set
from copy import deepcopy

from pydantic import ValidationError

from backend.core.utils import DotAccessibleDict
from plugins.core_engine.contracts import (
    RuntimeInterface, 
    ExecutionContext,
    MacroEvaluationServiceInterface
)
from .models import CodexCollection, ActivatedEntry, TriggerMode, Codex

logger = logging.getLogger("hevno.runtime.codex")


def _merge_codices(lore_codices: Dict[str, Any], moment_codices: Dict[str, Any]) -> Dict[str, Any]:
    """
    智能地合并 Lore 和 Moment 中的 Codex。
    - 它会合并同名 Codex 的条目列表。
    - 如果条目 ID 相同，Moment 中的条目会覆盖 Lore 中的。
    """
    # 先深拷贝 lore 的数据作为基础，避免修改原始状态
    merged_data = deepcopy(lore_codices)

    for name, moment_codex_data in moment_codices.items():
        if name not in merged_data:
            # 如果 Lore 中没有这个 Codex，直接添加
            merged_data[name] = deepcopy(moment_codex_data)
        else:
            # 如果 Lore 中有同名 Codex，进行条目级别的合并
            lore_codex_data = merged_data[name]
            
            # 合并 config, moment 优先
            lore_codex_data['config'] = {**lore_codex_data.get('config', {}), **moment_codex_data.get('config', {})}
            
            # 合并 entries, moment 优先
            lore_entries = lore_codex_data.get('entries', [])
            moment_entries = moment_codex_data.get('entries', [])
            
            # 使用字典来处理覆盖逻辑，以 entry id 为键
            entries_map: Dict[str, Any] = {entry['id']: entry for entry in lore_entries}
            for entry in moment_entries:
                entries_map[entry['id']] = entry
            
            # 将合并后的条目转换回列表
            lore_codex_data['entries'] = list(entries_map.values())
    
    return merged_data


class InvokeRuntime(RuntimeInterface):
    """
    【已重构】codex.invoke 运行时的实现。
    它现在能从 Lore 和 Moment 两个作用域中合并知识库。
    """
    async def execute(
        self,
        config: Dict[str, Any],
        context: ExecutionContext,
        **kwargs
    ) -> Dict[str, Any]:
        
        macro_service: MacroEvaluationServiceInterface = context.shared.services.macro_evaluation_service
        from_sources = config.get("from", [])
        if not from_sources:
            return {"output": ""}
        
        recursion_enabled = config.get("recursion_enabled", False)
        debug_mode = config.get("debug", False)
        lock = context.shared.global_write_lock
        

        lore_codices = context.shared.lore_state.get("codices", {})
        moment_codices = context.shared.moment_state.get("codices", {})
        
        unified_codex_data = _merge_codices(lore_codices, moment_codices)
        
        if not unified_codex_data:
            logger.warning("No codices found in lore_state or moment_state.")
            return {"output": ""}
            
        try:
            codex_collection = CodexCollection.model_validate(unified_codex_data).root
        except ValidationError as e:
            raise ValueError(f"Invalid codex structure after merging lore and moment: {e}")
            
        initial_pool: List[ActivatedEntry] = []
        structural_eval_context = macro_service.build_context(context)

        logger.debug("--- [CODEX.INVOKE START] ---")
        logger.debug(f"Recursion enabled: {recursion_enabled}")
        logger.debug(f"Available codices after merge: {list(codex_collection.keys())}")

        for source_config in from_sources:
            codex_name = source_config.get("codex")
            if not codex_name or not codex_collection.get(codex_name):
                logger.debug(f"Codex '{codex_name}' requested but not found in merged collection. Skipping.")
                continue
            
            codex_model = codex_collection.get(codex_name)
            source_text_macro = source_config.get("source", "")
            source_text = await macro_service.evaluate(source_text_macro, structural_eval_context, lock) if source_text_macro else ""
            
            logger.debug(f"Scanning codex '{codex_name}' with source text: '{str(source_text)[:100]}...'")

            for entry in codex_model.entries:
                is_enabled = await macro_service.evaluate(entry.is_enabled, structural_eval_context, lock)
                if not is_enabled:
                    continue
                
                keywords = await macro_service.evaluate(entry.keywords, structural_eval_context, lock)
                priority = await macro_service.evaluate(entry.priority, structural_eval_context, lock)

                is_activated, matched_keywords = False, []
                if entry.trigger_mode == TriggerMode.ALWAYS_ON:
                    is_activated = True
                elif entry.trigger_mode == TriggerMode.ON_KEYWORD and source_text and keywords:
                    for keyword in keywords:
                        if re.search(re.escape(str(keyword)), str(source_text), re.IGNORECASE):
                            matched_keywords.append(keyword)
                    if matched_keywords:
                        is_activated = True
                
                if is_activated:
                    activated = ActivatedEntry(
                        entry_model=entry, codex_name=codex_name, codex_config=codex_model.config,
                        priority_val=int(priority), keywords_val=keywords, is_enabled_val=bool(is_enabled),
                        source_text=str(source_text), matched_keywords=matched_keywords,
                        depth=0
                    )
                    initial_pool.append(activated)
                    logger.debug(f"  [+] Initial activation: '{entry.id}' (prio: {priority}, depth: 0)")
        
        rendered_entry_ids: Set[str] = set()
        rendered_parts_with_priority = []
        rendering_pool = sorted(initial_pool, key=lambda x: x.priority_val, reverse=True)
        
        logger.debug("--- [STARTING RENDER LOOP] ---")
        
        while rendering_pool:
            pool_state_log = ", ".join([f"{e.entry_model.id}({e.priority_val})" for e in rendering_pool])
            logger.debug(f"Loop Start | Pool: [{pool_state_log}]")

            entry_to_render = rendering_pool.pop(0)

            if entry_to_render.entry_model.id in rendered_entry_ids:
                logger.debug(f"  Skipping '{entry_to_render.entry_model.id}' as it's already rendered.")
                continue
            
            logger.debug(f"  -> Rendering '{entry_to_render.entry_model.id}' (prio: {entry_to_render.priority_val}, depth: {entry_to_render.depth})")
            
            content_eval_context = macro_service.build_context(context)
            content_eval_context['trigger'] = DotAccessibleDict({
                "source_text": entry_to_render.source_text,
                "matched_keywords": entry_to_render.matched_keywords
            })
            
            rendered_content = str(await macro_service.evaluate(entry_to_render.entry_model.content, content_eval_context, lock))
            
            rendered_parts_with_priority.append({
                "content": rendered_content, "priority": entry_to_render.priority_val, "id": entry_to_render.entry_model.id
            })
            rendered_entry_ids.add(entry_to_render.entry_model.id)
            
            max_depth = entry_to_render.codex_config.recursion_depth
            if recursion_enabled and entry_to_render.depth < max_depth:
                new_source_text = rendered_content
                logger.debug(f"     Recursion check (depth {entry_to_render.depth} < max_depth {max_depth}). Source: '{new_source_text[:50]}...'")
                
                newly_activated_this_pass = []
                for codex_name, codex_model in codex_collection.items():
                    for entry in codex_model.entries:
                        if entry.id in rendered_entry_ids or any(p.entry_model.id == entry.id for p in rendering_pool):
                            continue
                        
                        if entry.trigger_mode == TriggerMode.ON_KEYWORD:
                            is_enabled = await macro_service.evaluate(entry.is_enabled, structural_eval_context, lock)
                            if not is_enabled: continue
                            keywords = await macro_service.evaluate(entry.keywords, structural_eval_context, lock)
                            new_matched_keywords = [kw for kw in keywords if re.search(re.escape(str(kw)), new_source_text, re.IGNORECASE)]
                            
                            if new_matched_keywords:
                                priority = await macro_service.evaluate(entry.priority, structural_eval_context, lock)
                                activated = ActivatedEntry(
                                    entry_model=entry, codex_name=codex_name, codex_config=codex_model.config,
                                    priority_val=int(priority), keywords_val=keywords, is_enabled_val=is_enabled,
                                    source_text=new_source_text, matched_keywords=new_matched_keywords,
                                    depth=entry_to_render.depth + 1
                                )
                                newly_activated_this_pass.append(activated)
                                logger.debug(f"       [+] Recursive activation: '{entry.id}' (prio: {priority}, depth: {activated.depth})")
                
                if newly_activated_this_pass:
                    rendering_pool.extend(newly_activated_this_pass)
                    rendering_pool.sort(key=lambda x: x.priority_val, reverse=True)
        
        logger.debug("--- [FINALIZING OUTPUT] ---")
        pre_sort_log = ", ".join([f"{p['id']}({p['priority']})" for p in rendered_parts_with_priority])
        logger.debug(f"Rendered parts (in render order): [{pre_sort_log}]")
        
        final_sorted_parts = sorted(rendered_parts_with_priority, key=lambda p: p['priority'], reverse=True)
        
        post_sort_log = ", ".join([f"{p['id']}({p['priority']})" for p in final_sorted_parts])
        logger.debug(f"Final parts (sorted by priority): [{post_sort_log}]")

        final_text = "\n\n".join([p['content'] for p in final_sorted_parts])
        
        logger.debug(f"Final output text:\n---\n{final_text}\n---")
        logger.debug("--- [CODEX.INVOKE END] ---")

        if debug_mode:
            # 调试输出可以更丰富，但目前保持简单
            return {"output": final_text, "debug_info": {"rendered_ids": list(rendered_entry_ids)}}
        
        return {"output": final_text}
```

### models.py
```
# plugins/core_codex/models.py
from enum import Enum
from typing import List, Dict, Any, Optional
from pydantic import BaseModel, Field, RootModel, ConfigDict, field_validator

class TriggerMode(str, Enum):
    ALWAYS_ON = "always_on"
    ON_KEYWORD = "on_keyword"

class CodexEntry(BaseModel):
    id: str
    content: str
    is_enabled: Any = Field(default=True)
    trigger_mode: TriggerMode = Field(default=TriggerMode.ALWAYS_ON)
    keywords: Any = Field(default_factory=list)
    priority: Any = Field(default=0)
    model_config = ConfigDict(extra='forbid')

class CodexConfig(BaseModel):
    recursion_depth: int = Field(default=3, ge=0)
    model_config = ConfigDict(extra='forbid')

class Codex(RootModel[Dict[str, Any]]):
    """
    Codex 定义，本质上是一个字典。
    它必须包含一个 'entries' 键，其值为一个条目列表。
    """
    @field_validator('root')
    @classmethod
    def check_entries_exist_and_are_valid(cls, v: Dict[str, Any]) -> Dict[str, Any]:
        if 'entries' not in v or not isinstance(v['entries'], list):
            raise ValueError("A codex must contain an 'entries' list.")
        
        # 手动验证每个条目
        validated_entries = [CodexEntry.model_validate(entry) for entry in v['entries']]
        v['entries'] = validated_entries
        return v

    @property
    def entries(self) -> List[CodexEntry]:
        """提供便捷的属性访问器。"""
        return self.root['entries']

    @property
    def config(self) -> CodexConfig:
        """提供便捷的属性访问器，并处理默认值。"""
        config_data = self.root.get('config', {})
        return CodexConfig.model_validate(config_data)

    @property
    def description(self) -> Optional[str]:
        return self.root.get('description')

    @property
    def metadata(self) -> Dict[str, Any]:
        return self.root.get('metadata', {})

class CodexCollection(RootModel[Dict[str, Codex]]):
    pass

# 用于运行时内部处理的数据结构
class ActivatedEntry(BaseModel):
    entry_model: CodexEntry
    codex_name: str
    codex_config: CodexConfig
    
    priority_val: int
    keywords_val: List[str]
    is_enabled_val: bool
    
    source_text: str
    matched_keywords: List[str] = Field(default_factory=list)
    
    # 【确认此字段存在】
    depth: int = Field(default=0)
    
    model_config = ConfigDict(arbitrary_types_allowed=True)
```

### __init__.py
```
# plugins/core_codex/__init__.py
import logging
from backend.core.contracts import Container, HookManager

from .invoke_runtime import InvokeRuntime
logger = logging.getLogger(__name__)

# --- 钩子实现 ---
async def register_codex_runtime(runtimes: dict) -> dict:
    """钩子实现：向引擎注册本插件提供的 'codex.invoke' 运行时。"""
    runtimes["codex.invoke"] = InvokeRuntime 
    logger.debug("Runtime 'codex.invoke' provided to runtime registry.")
    return runtimes

# --- 主注册函数 ---
def register_plugin(container: Container, hook_manager: HookManager):
    logger.info("--> 正在注册 [core_codex] 插件...")

    # 注册运行时
    hook_manager.add_implementation(
        "collect_runtimes", 
        register_codex_runtime, 
        plugin_name="core_codex"
    )
    logger.debug("钩子实现 'collect_runtimes' 已注册。")

    logger.info("插件 [core_codex] 注册成功。")
```

### manifest.json
```
{
    "id": "core_codex",
    "name": "core_codex",
    "version": "1.0.0",
    "description": "Provides the Codex knowledge base system and the 'codex.invoke' runtime.",
    "author": "Hevno Team",
    "backend": {
        "priority": 30,
        "dependencies": ["core_engine"]
    }
}
```

# Directory: plugins/core_memoria

### tasks.py
```
# plugins/core_memoria/tasks.py 
import logging
from typing import List, Dict, Any
from uuid import UUID

# 从平台核心导入接口和类型
from backend.core.contracts import Container
# 从本插件导入模型
from .models import AutoSynthesisConfig

from plugins.core_llm.contracts import (
    LLMServiceInterface, 
    LLMResponse, 
    LLMRequestFailedError,
    LLMResponseStatus
)

logger = logging.getLogger(__name__)

async def run_synthesis_task(
    container: Container,
    sandbox_id: UUID,
    stream_name: str,
    synthesis_config: Dict[str, Any],
    entries_to_summarize_dicts: List[Dict[str, Any]]
):
    """
    一个解耦的后台任务。
    """
    logger.info(f"后台任务启动：为沙盒 {sandbox_id} 的流 '{stream_name}' 生成总结。")
    
    try:
        # --- 1. 解析需要的服务和数据 ---
        llm_service: LLMServiceInterface = container.resolve("llm_service")
        event_queue: Dict[UUID, List[Dict[str, Any]]] = container.resolve("memoria_event_queue")
        config = AutoSynthesisConfig.model_validate(synthesis_config)

        # --- 2. 准备并调用 LLM ---
        events_text = "\n".join([f"- {entry['content']}" for entry in entries_to_summarize_dicts])
        prompt = config.prompt.format(events_text=events_text)

        response: LLMResponse = await llm_service.request(model_name=config.model, prompt=prompt)

        if response.status != LLMResponseStatus.SUCCESS or not response.content:
            error_msg = response.error_details.message if response.error_details else 'No content'
            logger.error(f"LLM 总结失败 for sandbox {sandbox_id}: {error_msg}")
            return

        summary_content = response.content.strip()
        logger.info(f"LLM 成功生成总结 for sandbox {sandbox_id} of stream '{stream_name}'.")

        event_payload = {
            "type": "memoria_synthesis_completed",
            "stream_name": stream_name,
            "content": summary_content,
            "level": config.level,
            "tags": ["synthesis", "auto-generated"],
        }
        
        if sandbox_id not in event_queue:
            event_queue[sandbox_id] = []
        event_queue[sandbox_id].append(event_payload)
        
        logger.info(f"已为沙盒 {sandbox_id} 成功提交 'memoria_synthesis_completed' 事件。")

    except LLMRequestFailedError as e:
        logger.error(f"后台 LLM 请求在多次重试后失败: {e}", exc_info=False)
    except Exception:
        logger.exception(f"在执行 memoria 综合任务时发生未预料的错误 for sandbox {sandbox_id}")
```

### models.py
```
# plugins/core_memoria/models.py
from __future__ import annotations
import logging
from uuid import UUID, uuid4
from datetime import datetime, timezone
from typing import List, Dict, Any, Optional

from pydantic import BaseModel, Field, RootModel, ConfigDict

logger = logging.getLogger(__name__)

# --- Core Data Models for Memoria Structure ---

class MemoryEntry(BaseModel):
    """一个单独的、结构化的记忆条目。"""
    # 这允许模型处理用户定义的字符串 ID（如 'initial-event'）和系统生成的 UUID。
    id: str = Field(default_factory=lambda: str(uuid4()))
    sequence_id: int = Field(..., description="在所有流中唯一的、单调递增的因果序列号。")
    level: str = Field(default="event", description="记忆的层级，如 'event', 'summary', 'milestone'。")
    tags: List[str] = Field(default_factory=list, description="用于快速过滤和检索的标签。")
    content: str = Field(..., description="记忆条目的文本内容。")
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    metadata: Dict[str, Any] = Field(default_factory=dict)

class AutoSynthesisConfig(BaseModel):
    """自动综合（大总结）的行为配置。"""
    enabled: bool = Field(default=False)
    trigger_count: int = Field(default=10, gt=0, description="触发综合所需的条目数量。")
    level: str = Field(default="summary", description="综合后产生的新条目的层级。")
    model: str = Field(default="gemini/gemini-2.5-flash", description="用于执行综合的 LLM 模型。")
    prompt: str = Field(
        default="The following is a series of events. Please provide a concise summary.\n\nEvents:\n{events_text}",
        description="用于综合的 LLM 提示模板。必须包含 '{events_text}' 占位符。"
    )


class MemoryStreamConfig(BaseModel):
    """每个记忆流的独立配置。"""
    auto_synthesis: AutoSynthesisConfig = Field(default_factory=AutoSynthesisConfig)
    metadata: Dict[str, Any] = Field(default_factory=dict)


class MemoryStream(BaseModel):
    """一个独立的记忆回廊，包含它自己的配置和条目列表。"""
    config: MemoryStreamConfig = Field(default_factory=MemoryStreamConfig)
    entries: List[MemoryEntry] = Field(default_factory=list)
    
    synthesis_trigger_counter: int = Field(
        default=0, 
        description="Internal counter for auto-synthesis trigger. This is part of the persisted state."
    )
class Memoria(RootModel[Dict[str, Any]]):
    """
    代表 world.memoria 的顶层结构。
    它是一个字典，键是流名称，值是 MemoryStream 对象。
    还包含一个全局序列号，以确保因果关系的唯一性。
    """
    root: Dict[str, Any] = Field(default_factory=lambda: {"__global_sequence__": 0})
    
    def get_stream(self, stream_name: str) -> Optional[MemoryStream]:
        """安全地获取一个 MemoryStream 的 Pydantic 模型实例。"""
        stream_data = self.root.get(stream_name)
        if isinstance(stream_data, dict):
            return MemoryStream.model_validate(stream_data)
        return None

    def set_stream(self, stream_name: str, stream_model: MemoryStream):
        """将一个 MemoryStream 模型实例写回到根字典中。"""
        self.root[stream_name] = stream_model.model_dump()

    def get_next_sequence_id(self) -> int:
        """获取并递增全局序列号，确保原子性。"""
        current_seq = self.root.get("__global_sequence__", 0)
        next_seq = current_seq + 1
        self.root["__global_sequence__"] = next_seq
        return next_seq
```

### runtimes.py
```
# plugins/core_memoria/runtimes.py

import logging
from typing import Dict, Any, List

from backend.core.contracts import BackgroundTaskManager 
from plugins.core_engine.contracts import ExecutionContext, RuntimeInterface

from .models import Memoria, MemoryEntry
from .tasks import run_synthesis_task

logger = logging.getLogger(__name__)


class MemoriaAddRuntime(RuntimeInterface):
    """
    向指定的记忆流中添加一条新的记忆条目。
    数据现在被写入 context.shared.moment_state。
    """
    async def execute(self, config: Dict[str, Any], context: ExecutionContext, **kwargs) -> Dict[str, Any]:
        stream_name = config.get("stream")
        content = config.get("content")
        if not stream_name or not content:
            raise ValueError("MemoriaAddRuntime requires 'stream' and 'content' in its config.")
        
        level = config.get("level", "event")
        tags = config.get("tags", [])
        
        # 从 moment_state 中获取或创建 memoria 数据
        memoria_data = context.shared.moment_state.setdefault("memoria", {"__global_sequence__": 0})
        
        if "__hevno_type__" not in memoria_data:
            memoria_data["__hevno_type__"] = "hevno/memoria"

        memoria = Memoria.model_validate(memoria_data)
        
        stream = memoria.get_stream(stream_name)
        if stream is None:
            from .models import MemoryStream
            stream = MemoryStream()

        new_entry = MemoryEntry(
            sequence_id=memoria.get_next_sequence_id(),
            level=level,
            tags=tags,
            content=str(content)
        )
        stream.entries.append(new_entry)
        stream.synthesis_trigger_counter += 1
        
        memoria.set_stream(stream_name, stream)
        
        context.shared.moment_state["memoria"] = memoria.model_dump()

        synth_config = stream.config.auto_synthesis
        if synth_config.enabled and stream.synthesis_trigger_counter >= synth_config.trigger_count:
            logger.info(f"流 '{stream_name}' 满足综合条件，正在提交后台任务。")
            
            task_manager: BackgroundTaskManager = context.shared.services.task_manager
            entries_to_summarize = stream.entries[-synth_config.trigger_count:]
            
            task_manager.submit_task(
                run_synthesis_task,
                sandbox_id=context.initial_snapshot.sandbox_id,
                stream_name=stream_name,
                synthesis_config=synth_config.model_dump(),
                entries_to_summarize_dicts=[e.model_dump() for e in entries_to_summarize]
            )
            # 注意：synthesis_trigger_counter 的重置现在在 `apply_pending_synthesis` 钩子中完成
            # 此处不再需要重置，以避免状态不一致

        return {"output": new_entry.model_dump()}


class MemoriaQueryRuntime(RuntimeInterface):
    """
    根据声明式条件从一个记忆流中检索条目。
    数据现在从 context.shared.moment_state 读取。
    """
    async def execute(self, config: Dict[str, Any], context: ExecutionContext, **kwargs) -> Dict[str, Any]:
        stream_name = config.get("stream")
        if not stream_name:
            raise ValueError("MemoriaQueryRuntime requires a 'stream' name in its config.")

        output_format = config.get("format", "raw_entries")
        if output_format not in ["raw_entries", "message_list"]:
             raise ValueError(f"Invalid 'format' value '{output_format}'. Must be 'raw_entries' or 'message_list'.")

        memoria_data = context.shared.moment_state.get("memoria", {})

        memoria = Memoria.model_validate(memoria_data)
        stream = memoria.get_stream(stream_name)
        
        if not stream:
            return {"output": []}

        # --- 过滤逻辑 ---
        results = stream.entries
        
        levels_to_get = config.get("levels")
        if isinstance(levels_to_get, list):
            results = [entry for entry in results if entry.level in levels_to_get]

        tags_to_get = config.get("tags")
        if isinstance(tags_to_get, list):
            tags_set = set(tags_to_get)
            results = [entry for entry in results if tags_set.intersection(entry.tags)]

        latest_n = config.get("latest")
        if isinstance(latest_n, int):
            results.sort(key=lambda e: e.sequence_id)
            results = results[-latest_n:]
            
        order = config.get("order", "ascending")
        reverse = (order == "descending")
        results.sort(key=lambda e: e.sequence_id, reverse=reverse)

        # --- 格式化逻辑 ---
        if output_format == "message_list":
            message_list = []
            for entry in results:
                if entry.level in ["user", "model"]:
                    message_list.append({
                        "role": entry.level,
                        "content": entry.content
                    })
            return {"output": message_list}
        else: # "raw_entries"
            return {"output": [entry.model_dump() for entry in results]}
```

### __init__.py
```
# plugins/core_memoria/__init__.py

import logging
from typing import Dict, List, Any
from uuid import UUID


from backend.core.contracts import Container, HookManager
from plugins.core_engine.contracts import ExecutionContext
from .runtimes import MemoriaAddRuntime, MemoriaQueryRuntime
from .models import Memoria, MemoryEntry

logger = logging.getLogger(__name__)

def _create_memoria_event_queue() -> Dict[UUID, List[Dict[str, Any]]]:
    """
    工厂函数：创建一个简单的、内存中的事件队列。
    这个队列是 core_memoria 插件私有的，用于暂存后台任务完成的事件。
    - 键: sandbox_id
    - 值: 一个包含事件负载字典的列表
    """
    logger.debug("创建 memoria_event_queue 单例。")
    return {}

async def provide_memoria_runtimes(runtimes: dict) -> dict:
    """钩子实现：向引擎注册本插件提供的所有运行时。"""
    memoria_runtimes = {
        "memoria.add": MemoriaAddRuntime,
        "memoria.query": MemoriaQueryRuntime,
    }
    
    for name, runtime_class in memoria_runtimes.items():
        if name not in runtimes:
            runtimes[name] = runtime_class
            logger.debug(f"Provided '{name}' runtime to the engine.")
            
    return runtimes

async def apply_pending_synthesis(context: ExecutionContext, container: Container) -> ExecutionContext:
    """
    钩子实现：在图执行前应用待处理的综合事件。
    操作 context.shared.moment_state。
    """
    event_queue: Dict[UUID, List[Dict[str, Any]]] = container.resolve("memoria_event_queue")
    sandbox_id = context.initial_snapshot.sandbox_id
    
    pending_events = event_queue.pop(sandbox_id, [])
    if not pending_events:
        return context

    logger.info(f"Memoria: 发现 {len(pending_events)} 个待处理的综合事件，正在应用到 moment_state...")
    
    moment_state = context.shared.moment_state
    memoria_data = moment_state.setdefault("memoria", {"__global_sequence__": 0})
    
    memoria = Memoria.model_validate(memoria_data)
    
    for event in pending_events:
        if event.get("type") == "memoria_synthesis_completed":
            stream_name = event.get("stream_name")
            if not stream_name:
                continue
                
            stream = memoria.get_stream(stream_name)
            if stream:
                # 重置触发器计数器
                stream.synthesis_trigger_counter = 0
                
                # 创建并添加新的总结条目
                summary_entry = MemoryEntry(
                    sequence_id=memoria.get_next_sequence_id(),
                    level=event.get("level", "summary"),
                    tags=event.get("tags", ["synthesis", "auto-generated"]),
                    content=str(event.get("content", ""))
                )
                stream.entries.append(summary_entry)
                memoria.set_stream(stream_name, stream)
                logger.debug(f"已将新总结应用到流 '{stream_name}'。")
    
    moment_state["memoria"] = memoria.model_dump()

    return context

# --- 主注册函数 ---
def register_plugin(container: Container, hook_manager: HookManager):
    logger.info("--> 正在注册 [core_memoria] 插件...")

    container.register("memoria_event_queue", _create_memoria_event_queue, singleton=True)
    logger.debug("服务 'memoria_event_queue' 已注册。")

    hook_manager.add_implementation(
        "collect_runtimes", 
        provide_memoria_runtimes, 
        plugin_name="core_memoria"
    )

    hook_manager.add_implementation(
        "before_graph_execution",
        apply_pending_synthesis,
        priority=50,
        plugin_name="core_memoria"
    )

    logger.debug("钩子实现 'collect_runtimes' 和 'before_graph_execution' 已注册。")

    logger.info("插件 [core_memoria] 注册成功。")
```

### manifest.json
```
{
    "id": "core_memoria",
    "name": "core_memoria",
    "version": "1.0.0",
    "description": "Provides a dynamic memory system for storing, synthesizing, and querying events, enabling short-term memory and long-term reflection for AI agents.",
    "author": "Hevno Team",
    "backend": {
        "priority": 40,
        "dependencies": ["core_engine", "core_llm"]
    }
}
```

# Directory: plugins/core_remote_hooks

### registry.py
```
# plugins/core_remote_hooks/registry.py

import logging
from typing import List, Set
from .contracts import GlobalHookRegistryInterface, HookLocation

logger = logging.getLogger(__name__)

class GlobalHookRegistry(GlobalHookRegistryInterface):
    """
    一个中心化的单例服务，用于存储和管理全域钩子路由表。
    """
    def __init__(self):
        self._backend_hooks: Set[str] = set()
        self._frontend_hooks: Set[str] = set()
        logger.info("GlobalHookRegistry initialized.")

    def register_backend_hooks(self, hooks: List[str]) -> None:
        """注册所有在后端发现的钩子。"""
        count_before = len(self._backend_hooks)
        self._backend_hooks.update(hooks)
        count_after = len(self._backend_hooks)
        logger.info(f"Registered {count_after - count_before} new backend hooks. Total: {count_after}.")

    def register_frontend_hooks(self, hooks: List[str]) -> None:
        """在收到前端同步消息后，注册所有前端钩子。"""
        count_before = len(self._frontend_hooks)
        self._frontend_hooks.update(hooks)
        count_after = len(self._frontend_hooks)
        logger.info(f"Registered {count_after - count_before} new frontend hooks from remote sync. Total: {count_after}.")

    def get_hook_location(self, hook_name: str) -> HookLocation:
        """根据钩子名称，查询其在全栈中的位置。"""
        is_local = hook_name in self._backend_hooks
        is_remote = hook_name in self._frontend_hooks

        if is_local and is_remote:
            return HookLocation.BOTH
        if is_local:
            return HookLocation.LOCAL
        if is_remote:
            return HookLocation.REMOTE
        
        return HookLocation.UNKNOWN
```

### __init__.py
```
# plugins/core_remote_hooks/__init__.py
import json
import logging

from fastapi import WebSocket

from backend.core.contracts import Container, HookManager
# 依赖 core_websocket 提供的服务
from plugins.core_websocket.connection_manager import ConnectionManager

# 从本插件导入组件
from .contracts import GlobalHookRegistryInterface
from .registry import GlobalHookRegistry
from .emitter import RemoteHookEmitter

logger = logging.getLogger(__name__)

# --- 服务工厂 ---

def _create_global_hook_registry() -> GlobalHookRegistry:
    return GlobalHookRegistry()

def _create_remote_hook_emitter(container: Container) -> RemoteHookEmitter:
    # 这个工厂依赖于另一个插件的服务
    connection_manager = container.resolve("connection_manager")
    return RemoteHookEmitter(connection_manager)


# --- 钩子实现 ---


async def handle_incoming_message(
    data: str,
    container: Container,
    hook_manager: HookManager
):
    """
    钩子实现: 监听 'websocket.message_received'。
    解析来自前端的消息，并根据类型进行分发。
    """
    try:
        payload = json.loads(data)
        message_type = payload.get("type")

        # Case 1: 这是前端发来的钩子清单同步消息
        if message_type == 'sync_hooks':
            registry: GlobalHookRegistryInterface = container.resolve("global_hook_registry")
            frontend_hooks = payload.get("hooks", [])
            registry.register_frontend_hooks(frontend_hooks)
            logger.info(f"Received and registered {len(frontend_hooks)} hooks from frontend.")
            return

        # Case 2: 这是普通的远程钩子调用
        hook_name = payload.get("hook_name")
        hook_data = payload.get("data", {})

        if not hook_name:
            logger.warning("Received a remote message without 'hook_name' or 'type'.")
            return

        logger.debug(f"Relaying remote hook from frontend: '{hook_name}'")
        # 在后端触发该钩子
        await hook_manager.trigger(hook_name, **hook_data)

    except json.JSONDecodeError:
        logger.warning(f"Failed to decode incoming WebSocket message: {data}")
    except Exception:
        logger.exception("Error handling incoming remote hook.")


# --- 主注册函数 ---
def register_plugin(container: Container, hook_manager: HookManager):
    logger.info("--> 正在注册 [core_remote_hooks] 插件...")

    # 1. 注册本插件提供的核心服务
    container.register("global_hook_registry", _create_global_hook_registry, singleton=True)
    container.register("remote_hook_emitter", _create_remote_hook_emitter, singleton=True)

    # 2. 注册钩子实现
    # 【移除】不再注册 services_post_register 钩子
    
    # 这个钩子处理所有来自前端的 WS 消息
    hook_manager.add_implementation(
        "websocket.message_received",
        handle_incoming_message,
        plugin_name="core_remote_hooks"
    )

    logger.info("插件 [core_remote_hooks] 注册成功。")
```

### manifest.json
```
{
    "id": "core_remote_hooks",
    "name": "Full-Stack Hook Bridge",
    "version": "1.0.0",
    "description": "Bridges the backend and frontend hook systems via WebSocket, enabling location-transparent event handling.",
    "author": "Hevno Team",
    "backend": {
        "priority": 15
    }
}
```

### contracts.py
```
# plugins/core_remote_hooks/contracts.py

from __future__ import annotations
from abc import ABC, abstractmethod
from enum import Enum
from typing import Dict, Any, List

class HookLocation(Enum):
    """
    定义一个钩子实现的位置，用于智能路由。
    """
    LOCAL = "local"    # 仅在当前环境（后端）中实现
    REMOTE = "remote"  # 仅在远端环境（前端）中实现
    BOTH = "both"      # 在两个环境中都有实现
    UNKNOWN = "unknown"  # 未在任何注册表中找到

class RemoteHookEmitterInterface(ABC):
    """
    定义了将钩子事件发送到远端（前端）的能力。
    """
    @abstractmethod
    async def emit(self, hook_name: str, data: Dict[str, Any]) -> None:
        raise NotImplementedError

class GlobalHookRegistryInterface(ABC):
    """
    定义了全域钩子路由表的接口。
    """
    @abstractmethod
    def register_backend_hooks(self, hooks: List[str]) -> None:
        raise NotImplementedError

    @abstractmethod
    def register_frontend_hooks(self, hooks: List[str]) -> None:
        raise NotImplementedError

    @abstractmethod
    def get_hook_location(self, hook_name: str) -> HookLocation:
        raise NotImplementedError
```

### emitter.py
```
# plugins/core_remote_hooks/emitter.py

import json
import logging
from typing import Dict, Any

from plugins.core_websocket.connection_manager import ConnectionManager
from .contracts import RemoteHookEmitterInterface

logger = logging.getLogger(__name__)

class RemoteHookEmitter(RemoteHookEmitterInterface):
    """
    将后端钩子事件打包并通过 WebSocket 广播到所有前端客户端。
    """
    def __init__(self, connection_manager: ConnectionManager):
        self._manager = connection_manager

    async def emit(self, hook_name: str, data: Dict[str, Any]) -> None:
        """
        构建 payload 并通过 WebSocket 连接管理器广播。
        """
        try:
            # 注意：kwargs 可能包含不可序列化为 JSON 的对象。
            # 这是一个简化的实现，一个更健壮的系统可能需要一个序列化层。
            payload = {
                "hook_name": hook_name,
                "data": data
            }
            message = json.dumps(payload, ensure_ascii=False)
            logger.debug(f"Emitting remote hook to frontend: '{hook_name}'")
            await self._manager.broadcast(message)
        except TypeError as e:
            logger.error(
                f"Could not serialize payload for remote hook '{hook_name}'. "
                f"Data may contain non-JSON-serializable objects. Error: {e}"
            )
        except Exception:
            logger.exception(f"Unexpected error while emitting remote hook '{hook_name}'.")
```

# Directory: plugins/core_websocket

### __init__.py
```
# plugins/core_websocket/__init__.py
import logging
from fastapi import APIRouter, WebSocket, WebSocketDisconnect

from backend.core.contracts import Container, HookManager
from .connection_manager import ConnectionManager

logger = logging.getLogger(__name__)

# --- 服务工厂 ---
def _create_connection_manager() -> ConnectionManager:
    return ConnectionManager()

# --- 钩子实现 (提供API路由) ---
async def provide_router(routers: list, container: Container, hook_manager: HookManager) -> list:
    ws_router = APIRouter()
    manager: ConnectionManager = container.resolve("connection_manager")

    @ws_router.websocket("/ws/hooks")
    async def websocket_endpoint(websocket: WebSocket):
        await manager.connect(websocket)
        logger.info("New WebSocket client connected.")
        try:
            while True:
                data = await websocket.receive_text()
                # 触发钩子时，传递 websocket 和 data 作为临时上下文
                await hook_manager.trigger(
                    "websocket.message_received",
                    websocket=websocket, 
                    data=data
                )
        except WebSocketDisconnect:
            manager.disconnect(websocket)
            logger.info("WebSocket client disconnected.")

    routers.append(ws_router)
    return routers

# --- 主注册函数 ---
def register_plugin(container: Container, hook_manager: HookManager):
    logger.info("--> 正在注册 [core_websocket] 插件...")
    
    container.register("connection_manager", _create_connection_manager, singleton=True)
    hook_manager.add_implementation("collect_api_routers", provide_router, plugin_name="core_websocket")
    
    logger.info("插件 [core_websocket] 注册成功。")
```

### connection_manager.py
```
# plugins/core_websocket/connection_manager.py
import asyncio
from typing import List, Dict
from fastapi import WebSocket

class ConnectionManager:
    def __init__(self):
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: str):
        """向所有连接的客户端广播消息"""
        if not self.active_connections:
            return
        
        # 使用 asyncio.gather 并发发送
        tasks = [conn.send_text(message) for conn in self.active_connections]
        await asyncio.gather(*tasks, return_exceptions=True)

    async def send_to(self, websocket: WebSocket, message: str):
        """向单个客户端发送消息"""
        await websocket.send_text(message)
```

### manifest.json
```
{
    "id": "core_websocket",
    "name": "core_websocket",
    "version": "1.0.0",
    "description": "Provides core WebSocket connection management and broadcast capabilities.",
    "author": "Hevno Team",
    "backend": {
        "priority": 10
    }
}
```

# Directory: plugins/core_llm

### service.py
```
# plugins/core_llm/service.py

from __future__ import annotations
import asyncio
import logging
from typing import Dict, Optional, Any, List

from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    RetryCallState,
)

from .manager import KeyPoolManager, KeyInfo
from .registry import ProviderRegistry
from .contracts import (
    LLMServiceInterface,
    LLMResponse,
    LLMError,
    LLMErrorType,
    LLMResponseStatus,
    LLMRequestFailedError,
)

logger = logging.getLogger(__name__)

def is_retryable_llm_error(retry_state: RetryCallState) -> bool:
    """Tenacity 重试条件：只在错误是可重试类型时才重试。"""
    exception = retry_state.outcome.exception()
    if not exception:
        return False
    return (
        isinstance(exception, LLMRequestFailedError) and
        exception.last_error is not None and
        exception.last_error.is_retryable
    )

class LLMService(LLMServiceInterface):
    def __init__(
        self,
        key_manager: KeyPoolManager,
        provider_registry: ProviderRegistry,
        max_retries: int = 3
    ):
        self.key_manager = key_manager
        self.provider_registry = provider_registry
        self.max_retries = max_retries
        self.last_known_error: Optional[LLMError] = None

    async def request(
        self,
        model_name: str,
        messages: List[Dict[str, Any]],
        **kwargs
    ) -> LLMResponse:
        """
        【已重构】
        向指定的 LLM 发起请求。
        此方法现在包含一个外层循环，用于在密钥认证失败时自动切换到下一个可用密钥。
        """
        self.last_known_error = None
        try:
            provider_name, actual_model_name = self._parse_model_name(model_name)
        except ValueError as e:
            return self._create_failure_response(model_name, LLMError(LLMErrorType.INVALID_REQUEST_ERROR, str(e), False))

        provider = self.provider_registry.get(provider_name)
        if not provider:
            raise ValueError(f"Provider '{provider_name}' not found.")
            
        # 如果提供商不需要密钥，直接调用并返回
        if not provider.requires_api_key():
            return await self._attempt_request_with_key(provider_name, actual_model_name, messages, None, **kwargs)

        key_pool = self.key_manager.get_pool(provider_name)
        if not key_pool:
             raise ValueError(f"No key pool registered for provider '{provider_name}'.")

        # 外层循环：遍历所有密钥，实现密钥切换
        # 我们使用密钥池中密钥的总数作为尝试上限
        num_keys = key_pool.get_key_count()
        for attempt in range(num_keys):
            try:
                # 每次循环都尝试获取一个可用的密钥
                async with self.key_manager.acquire_key(provider_name) as key_info:
                    # 使用此密钥进行请求（包含内部的 tenacity 重试）
                    return await self._attempt_request_with_key(
                        provider_name, actual_model_name, messages, key_info, **kwargs
                    )
            except LLMRequestFailedError as e:
                # 检查失败的根本原因
                if e.last_error and e.last_error.error_type == LLMErrorType.AUTHENTICATION_ERROR:
                    # 如果是认证失败，记录日志并继续外层循环以尝试下一个密钥
                    logger.warning(
                        f"Authentication failed for key ending in '...{key_info.key_string[-4:]}'. "
                        f"Trying next available key... ({attempt + 1}/{num_keys} attempts)"
                    )
                    continue # 继续 for 循环
                else:
                    # 如果是其他类型的永久性错误，则直接抛出
                    raise
        
        # 如果循环完成仍未成功，说明所有密钥都已尝试并失败
        final_message = f"All {num_keys} API keys for provider '{provider_name}' failed."
        raise LLMRequestFailedError(final_message, last_error=self.last_known_error)

    @retry(
        stop=stop_after_attempt(3), # 内层重试次数
        wait=wait_exponential(multiplier=1, min=2, max=10),
        retry=is_retryable_llm_error,
        reraise=True
    )
    async def _attempt_request_with_key(
        self,
        provider_name: str,
        model_name: str,
        messages: List[Dict[str, Any]],
        key_info: Optional[KeyInfo],
        **kwargs
    ) -> LLMResponse:
        """
        【新】使用一个【特定】的密钥执行单次 LLM 请求尝试。
        此方法被 tenacity 装饰器包裹，用于处理瞬时错误。
        """
        provider = self.provider_registry.get(provider_name)
        api_key_str = key_info.key_string if key_info else ""

        try:
            response = await provider.generate(
                messages=messages, model_name=model_name, api_key=api_key_str, **kwargs
            )
            
            if response.status in [LLMResponseStatus.ERROR, LLMResponseStatus.FILTERED] and response.error_details:
                self.last_known_error = response.error_details
                if key_info:
                    await self._handle_error(provider_name, key_info, response.error_details)
                
                # 如果错误是可重试的，抛出异常让 tenacity 捕获
                if response.error_details.is_retryable:
                    raise LLMRequestFailedError("Provider returned a retryable error.", last_error=response.error_details)
            
            return response
        
        except Exception as e:
            if isinstance(e, LLMRequestFailedError):
                raise

            llm_error = provider.translate_error(e)
            self.last_known_error = llm_error
            if key_info:
                await self._handle_error(provider_name, key_info, llm_error)

            raise LLMRequestFailedError(f"Request failed with key: {llm_error.message}", last_error=llm_error) from e

    async def _handle_error(self, provider_name: str, key_info: KeyInfo, error: LLMError):
        if error.error_type == LLMErrorType.AUTHENTICATION_ERROR:
            logger.warning(f"Banning key for '{provider_name}' due to authentication error.")
            await self.key_manager.mark_as_banned(provider_name, key_info.key_string)
        elif error.error_type == LLMErrorType.RATE_LIMIT_ERROR:
            cooldown = error.retry_after_seconds or 60
            logger.info(f"Cooling down key for '{provider_name}' for {cooldown}s due to rate limit.")
            self.key_manager.mark_as_rate_limited(provider_name, key_info.key_string, cooldown)

    def _parse_model_name(self, model_name: str) -> tuple[str, str]:
        parts = model_name.split('/', 1)
        if len(parts) != 2 or not all(parts):
            raise ValueError(f"Invalid model name format: '{model_name}'. Expected 'provider/model_id'.")
        return parts[0], parts[1]
    
    def _create_failure_response(self, model_name: str, error: LLMError) -> LLMResponse:
        return LLMResponse(status=LLMResponseStatus.ERROR, model_name=model_name, error_details=error)

```

### registry.py
```
# plugins/core_llm/registry.py

from typing import Dict, Type, Optional, Callable
from pydantic import BaseModel
from .providers.base import LLMProvider
import logging

logger = logging.getLogger(__name__)

class ProviderInfo(BaseModel):
    provider_class: Type[LLMProvider]
    key_env_var: str

# ProviderRegistry 现在是一个普通的类，不再有全局实例
class ProviderRegistry:
    """
    负责注册和查找 LLMProvider 实例及其元数据。
    它的实例由 DI 容器管理。
    """
    def __init__(self):
        self._providers: Dict[str, LLMProvider] = {}
        self._provider_info: Dict[str, ProviderInfo] = {}

    # register 不再是装饰器，而是一个普通的实例方法
    def register(self, name: str, provider_class: Type[LLMProvider], key_env_var: str):
        """向注册表注册一个 LLM 提供商。"""
        if name in self._provider_info:
            logger.warning(f"Overwriting LLM provider registration for '{name}'.")
        self._provider_info[name] = ProviderInfo(provider_class=provider_class, key_env_var=key_env_var)
        logger.info(f"LLM Provider '{name}' registered (keys from '{key_env_var}').")

    def get_provider_info(self, name: str) -> Optional[ProviderInfo]:
        return self._provider_info.get(name)

    def instantiate_all(self):
        """实例化所有已注册的 Provider。"""
        for name, info in self._provider_info.items():
            if name not in self._providers:
                self._providers[name] = info.provider_class()
    
    def get(self, name: str) -> Optional[LLMProvider]:
        return self._providers.get(name)
    
    def get_all_provider_info(self) -> Dict[str, ProviderInfo]:
        return self._provider_info
```

### __init__.py
```
# plugins/core_llm/__init__.py

import logging
import os
from typing import List, Dict, Type
from fastapi import APIRouter, Depends

# 从平台核心导入接口和类型
from backend.core.contracts import Container, HookManager

# 导入本插件内部的组件
from .service import LLMService
from .manager import KeyPoolManager, CredentialManager
from .registry import ProviderRegistry
from .runtime import LLMRuntime
from .reporters import LLMProviderReporter
from .providers.base import LLMProvider
from .providers.gemini import GeminiProvider
from .providers.mock import MockProvider
from .config_api import config_api_router

logger = logging.getLogger(__name__)

# --- 服务工厂 (Service Factories) ---

def _create_provider_registry() -> ProviderRegistry:
    """工厂：创建 ProviderRegistry 的【空】实例。"""
    return ProviderRegistry()

def _create_llm_service(container: Container) -> LLMService:
    """这个工厂函数现在只负责创建服务，不再负责填充注册表。"""
    # 依赖容器来获取已注册（但可能尚未填充）的服务
    provider_registry: ProviderRegistry = container.resolve("provider_registry")
    key_manager: KeyPoolManager = container.resolve("key_pool_manager")

    return LLMService(
        key_manager=key_manager,
        provider_registry=provider_registry,
        max_retries=3
    )

def _create_key_pool_manager() -> KeyPoolManager:
    """工厂：创建 KeyPoolManager。"""
    cred_manager = CredentialManager()
    return KeyPoolManager(credential_manager=cred_manager)


# --- 钩子实现 (Hook Implementations) ---

async def provide_llm_providers(providers: Dict[str, Dict[str, any]]) -> Dict[str, Dict[str, any]]:
    """钩子实现：向系统中提供本插件知道的所有 LLM Provider。"""
    if "gemini" not in providers:
        providers["gemini"] = {
            "class": GeminiProvider,
            "key_env_var": "GEMINI_API_KEYS"
        }
    
    # 无条件注册模拟提供商
    if "mock" not in providers:
        providers["mock"] = {
            "class": MockProvider,
            "key_env_var": "MOCK_API_KEYS_DUMMY" # 虚拟变量，不会被找到，因此不会创建密钥池
        }
    
    return providers

async def populate_llm_services(container: Container, hook_manager: HookManager):
    """
    钩子实现：监听 'services_post_register'。
    异步地收集所有 provider，填充注册表，并配置密钥管理器。
    """
    logger.debug("Async task: Populating LLM services...")
    provider_registry: ProviderRegistry = container.resolve("provider_registry")
    key_manager: KeyPoolManager = container.resolve("key_pool_manager")

    all_providers: Dict[str, Dict[str, any]] = await hook_manager.filter("collect_llm_providers", {})
    
    if not all_providers:
        logger.warning("No LLM providers were collected. LLM service will not be functional.")
        return

    # 2. 用收集到的信息填充注册表和密钥管理器
    for name, info in all_providers.items():
        provider_class = info.get("class")
        key_env_var = info.get("key_env_var")
        if provider_class and key_env_var:
            provider_registry.register(name, provider_class, key_env_var)
            key_manager.register_provider(name, key_env_var)

    # 3. 实例化所有 provider
    provider_registry.instantiate_all()
    logger.info(f"LLM Provider Registry populated with {len(all_providers)} provider(s).")


async def provide_runtime(runtimes: dict) -> dict:
    """钩子实现：向引擎注册 'llm.default' 运行时。"""
    if "llm.default" not in runtimes:
        runtimes["llm.default"] = LLMRuntime
        logger.debug("Provided 'llm.default' runtime to the engine.")
    return runtimes

async def provide_reporter(reporters: list, container: Container) -> list:
    """
    钩子实现：向审计员提供本插件的报告器。
    我们在这里从容器解析依赖，并实例化报告器。
    
    注意：container 被定义为关键字参数，以匹配 hook_manager.filter 的调用方式。
    """
    provider_registry = container.resolve("provider_registry")
    reporters.append(LLMProviderReporter(provider_registry))
    logger.debug("Provided 'LLMProviderReporter' to the auditor.")
    return reporters

async def provide_api_router(routers: List[APIRouter]) -> List[APIRouter]:
    """钩子实现：将本插件的配置API路由添加到收集中。"""
    routers.append(config_api_router)
    logger.debug("Provided LLM configuration API router to the application.")
    return routers



# --- 主注册函数 (Main Registration Function) ---
def register_plugin(container: Container, hook_manager: HookManager):
    logger.info("--> 正在注册 [core_llm] 插件...")

    container.register("provider_registry", _create_provider_registry)
    container.register("key_pool_manager", _create_key_pool_manager)
    container.register("llm_service", _create_llm_service)
    logger.debug("Services 'provider_registry', 'key_pool_manager', 'llm_service' registered.")

    hook_manager.add_implementation("services_post_register", populate_llm_services, plugin_name="core_llm")
    hook_manager.add_implementation("collect_llm_providers", provide_llm_providers, plugin_name="core_llm")
    hook_manager.add_implementation("collect_runtimes", provide_runtime, plugin_name="core_llm")
    hook_manager.add_implementation(
        "collect_api_routers",
        provide_api_router,
        plugin_name="core_llm"
    )
    
    # 移除 lambda，因为 HookManager 现在足够智能
    hook_manager.add_implementation("collect_reporters", provide_reporter, plugin_name="core_llm")

    logger.debug("Hook implementations registered.")
    logger.info("插件 [core_llm] 注册成功。")
```

### runtime.py
```
# plugins/core_llm/runtime.py

import logging
from datetime import datetime
from typing import Dict, Any, List

from plugins.core_engine.contracts import ExecutionContext, RuntimeInterface, MacroEvaluationServiceInterface
from .contracts import LLMResponse, LLMRequestFailedError

logger = logging.getLogger(__name__)

class LLMRuntime(RuntimeInterface):
    """
    一个强大的运行时，它通过“列表展开”机制编排一个结构化的消息列表，
    然后通过 Hevno LLM Gateway 发起调用。
    """
    async def execute(self, config: Dict[str, Any], context: ExecutionContext, **kwargs) -> Dict[str, Any]:
        model_name = config.get("model")
        if not model_name:
            raise ValueError("LLMRuntime requires a 'model' field in its config (e.g., 'gemini/gemini-2.5-flash').")

        if "prompt" in config:
            logger.warning("The 'prompt' field in 'llm.default' is deprecated and will be ignored. Please use the 'contents' list instead.")
        
        contents_config = config.get("contents")
        if not isinstance(contents_config, list):
            raise ValueError("LLMRuntime requires a 'contents' field in its config, which must be a list of message parts or injection directives.")
            
        macro_service: MacroEvaluationServiceInterface = context.shared.services.macro_evaluation_service
        lock = context.shared.global_write_lock
        
        final_messages: List[Dict[str, Any]] = []
        for item in contents_config:
            if not isinstance(item, dict):
                logger.warning(f"Skipping invalid item in 'contents' list: {item}. Must be a dictionary.")
                continue

            is_enabled_macro = item.get("is_enabled", True)
            eval_context = macro_service.build_context(context)
            if not await macro_service.evaluate(is_enabled_macro, eval_context, lock):
                continue
                
            item_type = item.get("type", "MESSAGE_PART")

            if item_type == "MESSAGE_PART":
                role = item.get("role")
                content_macro = item.get("content")
                if not role or content_macro is None:
                    logger.warning(f"Skipping MESSAGE_PART with missing 'role' or 'content': {item}")
                    continue
                
                evaluated_content = await macro_service.evaluate(content_macro, eval_context, lock)
                final_messages.append({"role": role, "content": str(evaluated_content)})

            elif item_type == "INJECT_MESSAGES":
                source_macro = item.get("source")
                if not source_macro:
                    logger.warning(f"Skipping INJECT_MESSAGES with missing 'source': {item}")
                    continue
                
                injected_messages = await macro_service.evaluate(source_macro, eval_context, lock)
                
                if isinstance(injected_messages, list):
                    for msg in injected_messages:
                        # --- FIX: Loosen validation and convert to plain dict ---
                        if msg and "role" in msg and "content" in msg:
                            # Append a new plain dict to ensure compatibility
                            final_messages.append({"role": msg["role"], "content": msg["content"]})
                        else:
                            logger.warning(f"Skipping invalid item in injected message list: {msg}")
                elif injected_messages is not None:
                     logger.warning(f"Macro for INJECT_MESSAGES 'source' did not evaluate to a list. Got {type(injected_messages).__name__}. Ignoring.")
            
            else:
                logger.warning(f"Unknown item type '{item_type}' in 'contents' list. Skipping.")

        llm_params = {k: v for k, v in config.items() if k not in ["model", "prompt", "contents"]}
        llm_service = context.shared.services.llm_service

        node = kwargs.get("node")
        
        # 准备要发送的请求体
        request_payload = {
            "model_name": model_name,
            "messages": final_messages,
            **llm_params
        }
        
        response: LLMResponse = None
        try:
            response = await llm_service.request(**request_payload)
            
            # --- 无论成功与否，都记录日志 ---
            if "diagnostics_log" in context.run_vars:
                diagnostic_entry = {
                    "timestamp": datetime.now().isoformat(),
                    "node_id": node.id if node else 'unknown',
                    "runtime": "llm.default",
                    "request": request_payload,
                    # 使用 model_dump 确保 Pydantic 模型被正确序列化
                    "response": response.model_dump(mode='json') if response else None 
                }
                context.run_vars["diagnostics_log"].append(diagnostic_entry)

            if response.error_details:
                return {"error": response.error_details.message, "error_type": response.error_details.error_type.value, "details": response.error_details.model_dump()}
            return {"output": response.content, "usage": response.usage, "model_name": response.model_name}
        
        except LLMRequestFailedError as e:
            # --- 在异常情况下也记录日志 ---
            if "diagnostics_log" in context.run_vars:
                diagnostic_entry = {
                    "timestamp": datetime.now().isoformat(),
                    "node_id": node.id if node else 'unknown',
                    "runtime": "llm.default",
                    "request": request_payload,
                    "response": {
                        "status": "ERROR",
                        "error_details": {
                            "message": str(e),
                            "last_known_provider_error": e.last_error.model_dump(mode='json') if e.last_error else None
                        }
                    }
                }
                context.run_vars["diagnostics_log"].append(diagnostic_entry)
            
            return {"error": str(e), "details": e.last_error.model_dump() if e.last_error else None}
```

### config_api.py
```
# plugins/core_llm/config_api.py

import logging
from typing import List, Dict, Any, Optional
from pydantic import BaseModel, Field

from fastapi import APIRouter, Depends, HTTPException

from backend.core.dependencies import Service
from .manager import KeyPoolManager, KeyInfo, ProviderKeyPool

logger = logging.getLogger(__name__)

config_api_router = APIRouter(
    prefix="/api/llm/config",
    tags=["LLM Configuration API"]
)

# --- Pydantic Models (保持不变) ---
class ApiKeyStatus(BaseModel):
    key_suffix: str
    status: str
    rate_limit_until: Optional[float] = None

class KeyConfigResponse(BaseModel):
    provider: str
    keys: List[ApiKeyStatus]

# --- [新] Pydantic Model for Add Key ---
class AddKeyRequest(BaseModel):
    key: str = Field(..., min_length=10, description="要添加的完整 API 密钥。")

# --- API Endpoints (重构后) ---

def get_key_pool(
    provider_name: str, 
    key_manager: KeyPoolManager = Depends(Service("key_pool_manager"))
) -> ProviderKeyPool:
    pool = key_manager.get_pool(provider_name)
    if not pool:
        raise HTTPException(
            status_code=404,
            detail=f"Provider '{provider_name}' not found or has no key pool registered."
        )
    return pool

@config_api_router.get("/{provider_name}", response_model=KeyConfigResponse)
async def get_key_configuration(
    provider_name: str,
    key_pool: ProviderKeyPool = Depends(get_key_pool)
):
    key_statuses = []
    for key_info in key_pool._keys:
        key_statuses.append(ApiKeyStatus(
            key_suffix=f"...{key_info.key_string[-4:]}",
            status=key_info.status.value,
            rate_limit_until=key_info.rate_limit_until if key_info.rate_limit_until > 0 else None
        ))
    return KeyConfigResponse(provider=provider_name, keys=key_statuses)

@config_api_router.post("/{provider_name}/keys", status_code=201)
async def add_provider_key(
    provider_name: str,
    request: AddKeyRequest,
    key_manager: KeyPoolManager = Depends(Service("key_pool_manager"))
):
    """向 .env 文件添加一个新的 API 密钥并重新加载。"""
    try:
        key_manager.add_key_to_provider(provider_name, request.key)
        return {"message": "Key added successfully and pool reloaded."}
    except (ValueError, RuntimeError) as e:
        raise HTTPException(status_code=400, detail=str(e))
    except Exception:
        logger.exception(f"Failed to add key for provider {provider_name}")
        raise HTTPException(status_code=500, detail="An internal server error occurred.")

@config_api_router.delete("/{provider_name}/keys/{key_suffix}", status_code=200)
async def remove_provider_key(
    provider_name: str,
    key_suffix: str,
    key_manager: KeyPoolManager = Depends(Service("key_pool_manager"))
):
    """从 .env 文件中删除一个 API 密钥并重新加载。"""
    if len(key_suffix) != 4:
        raise HTTPException(status_code=400, detail="Key suffix must be exactly 4 characters long.")
    try:
        key_manager.remove_key_from_provider(provider_name, key_suffix)
        return {"message": "Key removed successfully and pool reloaded."}
    except (ValueError, RuntimeError) as e:
        raise HTTPException(status_code=400, detail=str(e))
    except Exception:
        logger.exception(f"Failed to remove key for provider {provider_name}")
        raise HTTPException(status_code=500, detail="An internal server error occurred.")
```

### manifest.json
```
{
    "id": "core_llm",
    "name": "core_llm",
    "version": "1.0.0",
    "description": "Provides the LLM Gateway, including multi-provider support, key management, and retry logic.",
    "author": "Hevno Team",
    "backend": {
        "priority": 20,
        "dependencies": ["core_engine"] 
    }
}
```

### contracts.py
```
# plugins/core_llm/contracts.py

from __future__ import annotations
from abc import ABC, abstractmethod
from enum import Enum
from typing import Optional, Dict, Any, List

from pydantic import BaseModel, Field

# --- Enums for Status and Error Types (公共契约) ---

class LLMResponseStatus(str, Enum):
    """定义 LLM 响应的标准化状态。"""
    SUCCESS = "success"
    FILTERED = "filtered"
    ERROR = "error"


class LLMErrorType(str, Enum):
    """定义标准化的 LLM 错误类型，用于驱动重试和故障转移逻辑。"""
    AUTHENTICATION_ERROR = "authentication_error"
    RATE_LIMIT_ERROR = "rate_limit_error"
    PROVIDER_ERROR = "provider_error"
    NETWORK_ERROR = "network_error"
    INVALID_REQUEST_ERROR = "invalid_request_error"
    UNKNOWN_ERROR = "unknown_error"


# --- Core Data Models (公共契约) ---

class LLMError(BaseModel):
    """一个标准化的错误对象，用于封装来自任何提供商的错误信息。"""
    error_type: LLMErrorType
    message: str
    is_retryable: bool
    retry_after_seconds: Optional[int] = None
    provider_details: Optional[Dict[str, Any]] = Field(default_factory=dict)


class LLMResponse(BaseModel):
    """一个标准化的响应对象，用于封装来自任何提供商的成功、过滤或错误结果。"""
    status: LLMResponseStatus
    content: Optional[str] = None
    model_name: Optional[str] = None
    usage: Optional[Dict[str, int]] = None
    error_details: Optional[LLMError] = None


# --- Custom Exception (公共契约) ---

class LLMRequestFailedError(Exception):
    """在所有重试和故障转移策略都用尽后，由 LLMService 抛出的最终异常。"""
    def __init__(self, message: str, last_error: Optional[LLMError] = None):
        super().__init__(message)
        self.last_error = last_error

    def __str__(self):
        if self.last_error:
            return f"{super().__str__()}\nLast known error ({self.last_error.error_type.value}): {self.last_error.message}"
        return super().__str__()


# --- Service Interface (公共契约) ---

class LLMServiceInterface(ABC):
    """
    定义了 LLM 网关服务必须提供的核心能力的抽象接口。
    其他插件应该依赖于这个接口，而不是具体的 LLMService 类。
    """
    @abstractmethod
    async def request(
        self,
        model_name: str,
        messages: List[Dict[str, Any]],
        **kwargs: Any
    ) -> LLMResponse:
        """
        向指定的 LLM 发起请求，并处理重试逻辑。
        """
        raise NotImplementedError
```

### reporters.py
```
# plugins/core_llm/reporters.py
from typing import Any
from plugins.core_diagnostics.contracts import Reportable
from .registry import ProviderRegistry


class LLMProviderReporter(Reportable):
    
    def __init__(self, provider_registry: ProviderRegistry):
        self._provider_registry = provider_registry

    @property
    def report_key(self) -> str:
        return "llm_providers"
    
    async def generate_report(self) -> Any:
        manifest = []
        all_info = self._provider_registry.get_all_provider_info()
        for name, info in all_info.items():
            provider_class = info.provider_class
            manifest.append({
                "name": name,
                "supported_models": getattr(provider_class, 'supported_models', [])
            })
        return sorted(manifest, key=lambda x: x['name'])
```

### manager.py
```
# plugins/core_llm/manager.py

import asyncio
import os
import time
from contextlib import asynccontextmanager
from dataclasses import dataclass, field
from enum import Enum
from typing import List, Dict, Optional, AsyncIterator
from dotenv import find_dotenv, get_key, set_key, unset_key, load_dotenv
import logging

logger = logging.getLogger(__name__)


# --- Enums and Data Classes for Key State Management ---

class KeyStatus(str, Enum):
    """定义 API 密钥的健康状态。"""
    AVAILABLE = "available"
    RATE_LIMITED = "rate_limited"
    BANNED = "banned"


@dataclass
class KeyInfo:
    """存储单个 API 密钥及其状态信息。"""
    key_string: str
    status: KeyStatus = KeyStatus.AVAILABLE
    rate_limit_until: float = 0.0  # Unix timestamp until which the key is rate-limited

    def is_available(self) -> bool:
        """检查密钥当前是否可用。"""
        if self.status == KeyStatus.BANNED:
            return False
        if self.status == KeyStatus.RATE_LIMITED:
            if time.time() < self.rate_limit_until:
                return False
            self.status = KeyStatus.AVAILABLE
            self.rate_limit_until = 0.0
        return self.status == KeyStatus.AVAILABLE


# --- Core Manager Components ---

class CredentialManager:
    """负责从环境变量中安全地加载和解析密钥。"""

    def load_keys_from_env(self, env_variable: str) -> List[str]:
        """从指定的环境变量中加载 API 密钥。"""
        keys_str = os.getenv(env_variable)
        if not keys_str:
            return []
        
        keys = [key.strip() for key in keys_str.split(',') if key.strip()]
        return keys


class ProviderKeyPool:
    """管理特定提供商的一组 API 密钥。"""
    def __init__(self, provider_name: str, keys: List[str]):
        self.provider_name = provider_name
        self._keys: List[KeyInfo] = [KeyInfo(key_string=k) for k in keys]
        self._semaphore = asyncio.Semaphore(len(self._keys))

    def _get_next_available_key(self) -> Optional[KeyInfo]:
        for key_info in self._keys:
            if key_info.is_available():
                return key_info
        return None

    def get_key_by_string(self, key_string: str) -> Optional[KeyInfo]:
        for key in self._keys:
            if key.key_string == key_string:
                return key
        return None

    def get_key_count(self) -> int:
        return len(self._keys)
        
    @asynccontextmanager
    async def acquire_key(self) -> AsyncIterator[KeyInfo]:
        await self._semaphore.acquire()
        try:
            key_info = self._get_next_available_key()
            if not key_info:
                raise RuntimeError(f"No available keys in pool '{self.provider_name}' despite acquiring semaphore.")
            yield key_info
        finally:
            self._semaphore.release()

    def mark_as_rate_limited(self, key_string: str, duration_seconds: int = 60):
        for key in self._keys:
            if key.key_string == key_string:
                key.status = KeyStatus.RATE_LIMITED
                key.rate_limit_until = time.time() + duration_seconds
                logger.info(f"Key for '{self.provider_name}' ending with '...{key_string[-4:]}' marked as rate-limited for {duration_seconds}s.")
                break

    async def mark_as_banned(self, key_string: str):
        for key in self._keys:
            if key.key_string == key_string and key.status != KeyStatus.BANNED:
                key.status = KeyStatus.BANNED
                await self._semaphore.acquire()
                logger.warning(f"Key for '{self.provider_name}' ending with '...{key_string[-4:]}' permanently banned. Concurrency reduced.")
                break


class KeyPoolManager:
    """顶层管理器，负责协调对 .env 文件的读写和内存状态。"""
    def __init__(self, credential_manager: CredentialManager):
        self._pools: Dict[str, ProviderKeyPool] = {}
        self._cred_manager = credential_manager
        self._provider_env_vars: Dict[str, str] = {}
        self._dotenv_path = find_dotenv()
        if not self._dotenv_path:
            self._dotenv_path = os.path.join(os.getcwd(), '.env')
            logger.warning(f".env file not found. Will attempt to create it at: {self._dotenv_path}")

    def register_provider(self, provider_name: str, env_variable: str):
        self._provider_env_vars[provider_name] = env_variable
        keys = self._cred_manager.load_keys_from_env(env_variable)
        self._pools[provider_name] = ProviderKeyPool(provider_name, keys)
        if keys:
            logger.info(f"Registered provider '{provider_name}' with {len(keys)} keys from '{env_variable}'.")
        else:
            logger.info(f"Registered provider '{provider_name}' with 0 keys (env var '{env_variable}' is empty or not set). Pool is ready.")

    def reload_keys(self, provider_name: str):
        if provider_name not in self._provider_env_vars:
            raise ValueError(f"Provider '{provider_name}' is not registered.")
        
        env_variable = self._provider_env_vars[provider_name]
        
        # 即使 .env 文件现在为空，这个调用也会确保 os.environ 反映最新状态
        load_dotenv(dotenv_path=self._dotenv_path, override=True)
        
        keys = self._cred_manager.load_keys_from_env(env_variable)
        self._pools[provider_name] = ProviderKeyPool(provider_name, keys)
        logger.info(f"Reloaded provider '{provider_name}' with {len(keys)} keys from '{env_variable}'.")

    def add_key_to_provider(self, provider_name: str, new_key: str):
        if provider_name not in self._provider_env_vars:
            raise ValueError(f"Provider '{provider_name}' is not registered.")
        
        env_var = self._provider_env_vars[provider_name]
        current_keys_str = get_key(self._dotenv_path, env_var) or ""
        keys = [k.strip() for k in current_keys_str.split(',') if k.strip()]

        if new_key in keys:
            logger.warning(f"Key already exists for provider '{provider_name}'. Skipping.")
            return

        keys.append(new_key)
        set_key(self._dotenv_path, env_var, ",".join(keys))
        logger.info(f"Successfully wrote new key to .env for provider '{provider_name}'.")
        self.reload_keys(provider_name) 

    def remove_key_from_provider(self, provider_name: str, key_suffix_to_remove: str):
        if not os.path.exists(self._dotenv_path):
             logger.warning(f"Cannot remove key, .env file not found at {self._dotenv_path}.")
             return

        if provider_name not in self._provider_env_vars:
            raise ValueError(f"Provider '{provider_name}' is not registered.")

        env_var = self._provider_env_vars[provider_name]
        current_keys_str = get_key(self._dotenv_path, env_var) or ""
        keys = [k.strip() for k in current_keys_str.split(',') if k.strip()]

        key_found = False
        updated_keys = []
        for key in keys:
            if key.endswith(key_suffix_to_remove):
                key_found = True
            else:
                updated_keys.append(key)
        
        if not key_found:
            logger.warning(f"Key with suffix '...{key_suffix_to_remove}' not found for provider '{provider_name}'.")
            return

        # --- 核心修复开始 ---
        # 1. 在修改 .env 文件之前，从当前进程的 os.environ 中删除该变量
        #    这样可以确保后续的 load_dotenv 不会受到旧值的影响
        if env_var in os.environ:
            del os.environ[env_var]
            logger.debug(f"Temporarily removed '{env_var}' from os.environ to ensure clean reload.")
        # --- 核心修复结束 ---

        if not updated_keys:
            unset_key(self._dotenv_path, env_var)
            logger.info(f"Removed last key for '{env_var}' from .env file.")
        else:
            set_key(self._dotenv_path, env_var, ",".join(updated_keys))
            logger.info(f"Removed key ending in '...{key_suffix_to_remove}' from .env file.")
        
        self.reload_keys(provider_name)

    def get_pool(self, provider_name: str) -> Optional[ProviderKeyPool]:
        return self._pools.get(provider_name)

    @asynccontextmanager
    async def acquire_key(self, provider_name: str) -> AsyncIterator[KeyInfo]:
        pool = self.get_pool(provider_name)
        if not pool:
            raise ValueError(f"No key pool registered for provider '{provider_name}'.")
        
        async with pool.acquire_key() as key_info:
            yield key_info

    def mark_as_rate_limited(self, provider_name: str, key_string: str, duration_seconds: int = 60):
        pool = self.get_pool(provider_name)
        if pool:
            pool.mark_as_rate_limited(key_string, duration_seconds)

    async def mark_as_banned(self, provider_name: str, key_string: str):
        pool = self.get_pool(provider_name)
        if pool:
            await pool.mark_as_banned(key_string)
```

### providers/__init__.py
```

```

### providers/gemini.py
```
# plugins/core_llm/providers/gemini.py

from typing import Any, List, Dict
import google.generativeai as genai
from google.api_core import exceptions as google_exceptions
from google.generativeai import types as generation_types

from .base import LLMProvider
from ..contracts import (
    LLMResponse,
    LLMError,
    LLMResponseStatus,
    LLMErrorType,
)

class GeminiProvider(LLMProvider):
    """
    针对 Google Gemini API 的 LLMProvider 实现。
    """

    async def generate(
        self,
        *,
        messages: List[Dict[str, Any]],
        model_name: str,
        api_key: str,
        **kwargs: Any
    ) -> LLMResponse:
        try:
            genai.configure(api_key=api_key)
            
            # --- [核心修复开始] ---
            system_instruction = None
            provider_messages = []
            
            # 1. 遍历消息，分离出 system prompt
            for msg in messages:
                role = msg.get("role")
                content = msg.get("content", "")
                
                if role == "system":
                    # 如果有多条 system 消息，将它们合并
                    if system_instruction is None:
                        system_instruction = ""
                    system_instruction += str(content) + "\n"
                elif role in ["user", "model"]:
                    # The Gemini SDK expects {"role": "...", "parts": ["..."]}
                    provider_messages.append({"role": role, "parts": [str(content)]})
            
            # 2. 实例化模型时传入 system_instruction
            model = genai.GenerativeModel(
                model_name,
                system_instruction=system_instruction.strip() if system_instruction else None
            )

            generation_config = {
                "temperature": kwargs.get("temperature"),
                "top_p": kwargs.get("top_p"),
                "top_k": kwargs.get("top_k"),
                "max_output_tokens": kwargs.get("max_tokens"),
            }
            generation_config = {k: v for k, v in generation_config.items() if v is not None}
            
            # 3. generate_content_async 只接收 user/model 消息
            response: generation_types.GenerateContentResponse = await model.generate_content_async(
                contents=provider_messages,
                generation_config=generation_config
            )
            # --- [核心修复结束] ---

            if not response.parts:
                if response.prompt_feedback.block_reason:
                    error_message = f"Request blocked due to {response.prompt_feedback.block_reason.name}"
                    return LLMResponse(status=LLMResponseStatus.FILTERED, model_name=model_name, error_details=LLMError(error_type=LLMErrorType.INVALID_REQUEST_ERROR, message=error_message, is_retryable=False))
                # --- [新增健壮性] ---
                # 如果没有部分且没有明确的阻塞原因，返回一个通用错误
                else:
                     return LLMResponse(status=LLMResponseStatus.ERROR, model_name=model_name, error_details=LLMError(error_type=LLMErrorType.PROVIDER_ERROR, message="Provider returned an empty response without a clear reason.", is_retryable=True))


            usage = {"prompt_tokens": response.usage_metadata.prompt_token_count, "completion_tokens": response.usage_metadata.candidates_token_count, "total_tokens": response.usage_metadata.total_token_count}
            
            return LLMResponse(status=LLMResponseStatus.SUCCESS, content=response.text, model_name=model_name, usage=usage)

        except generation_types.StopCandidateException as e:
            return LLMResponse(status=LLMResponseStatus.FILTERED, model_name=model_name, error_details=LLMError(error_type=LLMErrorType.INVALID_REQUEST_ERROR, message=f"Generation stopped due to safety settings: {e}", is_retryable=False))

    def translate_error(self, ex: Exception) -> LLMError:
        # ... (此方法保持不变)
        error_details = {"provider": "gemini", "exception": type(ex).__name__, "message": str(ex)}
        if isinstance(ex, google_exceptions.PermissionDenied):
            return LLMError(error_type=LLMErrorType.AUTHENTICATION_ERROR, message="Invalid API key or insufficient permissions.", is_retryable=False, provider_details=error_details)
        if isinstance(ex, google_exceptions.ResourceExhausted):
            return LLMError(error_type=LLMErrorType.RATE_LIMIT_ERROR, message="Rate limit exceeded. Please try again later or use a different key.", is_retryable=False, provider_details=error_details)
        if isinstance(ex, google_exceptions.InvalidArgument):
            if "API key not valid" in str(ex):
                return LLMError(error_type=LLMErrorType.AUTHENTICATION_ERROR, message=f"The provided API key is invalid. Details: {ex}", is_retryable=False, provider_details=error_details)
            return LLMError(error_type=LLMErrorType.INVALID_REQUEST_ERROR, message=f"Invalid argument provided to the API. Check model name and parameters. Details: {ex}", is_retryable=False, provider_details=error_details)
        if isinstance(ex, (google_exceptions.ServiceUnavailable, google_exceptions.DeadlineExceeded)):
            return LLMError(error_type=LLMErrorType.PROVIDER_ERROR, message="The service is temporarily unavailable or the request timed out. Please try again.", is_retryable=True, provider_details=error_details)
        if isinstance(ex, google_exceptions.GoogleAPICallError):
            return LLMError(error_type=LLMErrorType.NETWORK_ERROR, message=f"A network-level error occurred while communicating with Google API: {ex}", is_retryable=True, provider_details=error_details)
        return LLMError(error_type=LLMErrorType.UNKNOWN_ERROR, message=f"An unknown error occurred with the Gemini provider: {ex}", is_retryable=False, provider_details=error_details)
```

### providers/base.py
```
# plugins/core_llm/providers/base.py

from abc import ABC, abstractmethod
from typing import Dict, Any, List

from ..contracts import LLMResponse, LLMError


class LLMProvider(ABC):
    """
    一个抽象基-类，定义了所有 LLM 提供商适配器的标准接口。
    """
    @classmethod
    def requires_api_key(cls) -> bool:
        """
        声明此提供商是否需要 API 密钥才能工作。
        如果此方法返回 False，LLM 服务将不会尝试为此提供商从池中获取密钥。
        """
        return True

    @abstractmethod
    async def generate(
        self,
        *,
        messages: List[Dict[str, Any]],
        model_name: str,
        api_key: str,
        **kwargs: Any
    ) -> LLMResponse:
        """
        与 LLM 提供商进行交互以生成内容。

        这个方法必须处理所有可能的成功和“软失败”（如内容过滤）场景，
        并将它们封装在标准的 LLMResponse 对象中。
        如果发生无法处理的硬性错误（如网络问题、认证失败），它应该抛出原始异常，
        以便上层服务可以捕获并使用 translate_error 进行处理。

        :param messages: 发送给模型的结构化消息列表。
        :param model_name: 要使用的具体模型名称 (e.g., 'gemini-1.5-pro-latest')。
        :param api_key: 用于本次请求的 API 密钥。
        :param kwargs: 其他特定于提供商的参数 (e.g., temperature, max_tokens)。
        :return: 一个标准的 LLMResponse 对象。
        :raises Exception: 任何未被处理的、需要由 translate_error 解析的硬性错误。
        """
        pass

    @abstractmethod
    def translate_error(self, ex: Exception) -> LLMError:
        """
        将特定于提供商的原始异常转换为我们标准化的 LLMError 对象。

        这个方法是解耦的关键，它将具体的 SDK 错误与我们系统的内部错误处理逻辑分离开。

        :param ex: 从 generate 方法捕获的原始异常。
        :return: 一个标准的 LLMError 对象。
        """
        pass
```

### providers/mock.py
```
# plugins/core_llm/providers/mock.py

import asyncio
from typing import Any, List, Dict

from .base import LLMProvider
from ..contracts import (
    LLMResponse,
    LLMError,
    LLMResponseStatus,
    LLMErrorType,
)

class MockProvider(LLMProvider):
    """
    一个用于测试和调试的模拟 LLM 提供商。
    它会返回一个预设的响应，而不会进行任何外部调用。
    """
    @classmethod
    def requires_api_key(cls) -> bool:
        """声明此提供商不需要 API 密钥。"""
        return False

    async def generate(
        self,
        *,
        messages: List[Dict[str, Any]],
        model_name: str,
        api_key: str, # 仍然接收此参数，但会忽略它
        **kwargs: Any
    ) -> LLMResponse:
        """
        生成一个模拟响应。
        """
        await asyncio.sleep(0.05) # 模拟网络延迟
        
        last_user_message = "No user message found."
        for msg in reversed(messages):
            if msg.get("role") == "user":
                last_user_message = msg.get("content", "")
                break

        mock_content = f"[MOCK RESPONSE for {model_name}] - Responding to: '{str(last_user_message)[:150]}...'"
        
        prompt_token_count = sum(len(str(msg.get("content", "")).split()) for msg in messages)

        return LLMResponse(
            status=LLMResponseStatus.SUCCESS,
            content=mock_content,
            model_name=model_name,
            usage={"prompt_tokens": prompt_token_count, "completion_tokens": 15, "total_tokens": prompt_token_count + 15}
        )

    def translate_error(self, ex: Exception) -> LLMError:
        """
        将异常转换为标准的 LLMError。
        对于模拟提供商，此方法不太可能被调用。
        """
        return LLMError(
            error_type=LLMErrorType.UNKNOWN_ERROR,
            message=f"An unexpected error occurred in MockProvider: {ex}",
            is_retryable=False
        )
```

# Directory: plugins/core_persistence

### service.py
```
# plugins/core_persistence/service.py

import os
import io
import json
import logging
import shutil
import asyncio
import base64
import zipfile
from pathlib import Path
from typing import Type, TypeVar, Tuple, Dict, Any, List, Optional
from uuid import UUID

import aiofiles
from pydantic import BaseModel, ValidationError
from PIL import Image, PngImagePlugin

# 导入位于后端内核的自定义序列化工具
from backend.core.serialization import custom_json_decoder_object_hook

from .contracts import PersistenceServiceInterface, PackageManifest
from plugins.core_engine.contracts import Sandbox, StateSnapshot # 仍然需要它们来做类型检查和序列化
from .models import AssetType, FILE_EXTENSIONS

T = TypeVar('T', bound=BaseModel)
logger = logging.getLogger(__name__)

class PersistenceService(PersistenceServiceInterface):
    def __init__(self, assets_base_dir: str):
        self.assets_base_dir = Path(assets_base_dir)
        self._sandboxes_root_dir = self.assets_base_dir / "sandboxes"
        self._sandboxes_root_dir.mkdir(parents=True, exist_ok=True)
        logger.info(f"PersistenceService initialized. Sandboxes directory: {self._sandboxes_root_dir.resolve()}")

    @property
    def sandboxes_root_dir(self) -> Path:
        return self._sandboxes_root_dir

    def _get_sandbox_dir(self, sandbox_id: UUID) -> Path:
        return self._sandboxes_root_dir / str(sandbox_id)

    async def save_sandbox(self, sandbox_id: UUID, data: Dict[str, Any]) -> None:
        sandbox_dir = self._get_sandbox_dir(sandbox_id)
        sandbox_dir.mkdir(parents=True, exist_ok=True)
        file_path = sandbox_dir / "sandbox.json"
        
        # 不再需要 `default` 参数，因为传入的 `data` 已经是完全 JSON 兼容的了
        json_string = json.dumps(data, indent=2, ensure_ascii=False)

        async with aiofiles.open(file_path, mode='w', encoding='utf-8') as f:
            await f.write(json_string)
        logger.debug(f"Persisted sandbox '{sandbox_id}' to {file_path}")

    async def load_sandbox(self, sandbox_id: UUID) -> Optional[Dict[str, Any]]:
        file_path = self._get_sandbox_dir(sandbox_id) / "sandbox.json"
        if not file_path.is_file(): return None
        async with aiofiles.open(file_path, mode='r', encoding='utf-8') as f:
            content = await f.read()
        return json.loads(content, object_hook=custom_json_decoder_object_hook)

    async def delete_sandbox(self, sandbox_id: UUID) -> None:
        sandbox_dir = self._get_sandbox_dir(sandbox_id)
        if sandbox_dir.exists():
            await asyncio.to_thread(shutil.rmtree, sandbox_dir)
            logger.info(f"Deleted sandbox directory: {sandbox_dir}")

    async def list_sandbox_ids(self) -> List[str]:
        if not self._sandboxes_root_dir.is_dir():
            return []
        def _sync_list_dirs():
            return [p.name for p in self._sandboxes_root_dir.iterdir() if p.is_dir()]
        return await asyncio.to_thread(_sync_list_dirs)

    async def save_snapshot(self, sandbox_id: UUID, snapshot_id: UUID, data: Dict[str, Any]) -> None:
        snapshot_dir = self._get_sandbox_dir(sandbox_id) / "snapshots"
        snapshot_dir.mkdir(parents=True, exist_ok=True)
        file_path = snapshot_dir / f"{snapshot_id}.json"
        
        json_string = json.dumps(data, indent=2, ensure_ascii=False)
        
        async with aiofiles.open(file_path, mode='w', encoding='utf-8') as f:
            await f.write(json_string)
        logger.debug(f"Persisted snapshot '{snapshot_id}' for sandbox '{sandbox_id}'")

    async def load_snapshot(self, sandbox_id: UUID, snapshot_id: UUID) -> Optional[Dict[str, Any]]:
        file_path = self._get_sandbox_dir(sandbox_id) / "snapshots" / f"{snapshot_id}.json"
        if not file_path.is_file(): return None
        async with aiofiles.open(file_path, mode='r', encoding='utf-8') as f: content = await f.read()
        return json.loads(content, object_hook=custom_json_decoder_object_hook)

    async def load_all_snapshots_for_sandbox(self, sandbox_id: UUID) -> List[Dict[str, Any]]:
        snapshot_dir = self._get_sandbox_dir(sandbox_id) / "snapshots"
        if not snapshot_dir.is_dir():
            return []

        def _sync_read_files() -> List[Dict[str, Any]]:
            snapshots_data = []
            for file_path in snapshot_dir.glob("*.json"):
                try:
                    content = file_path.read_text(encoding='utf-8')
                    snapshots_data.append(json.loads(content, object_hook=custom_json_decoder_object_hook))
                except (json.JSONDecodeError) as e:
                    logger.error(f"Skipping corrupt snapshot file {file_path}: {e}")
            return snapshots_data
            
        return await asyncio.to_thread(_sync_read_files)
        
    async def delete_all_for_sandbox(self, sandbox_id: UUID) -> None:
        """异步删除属于特定沙盒的所有快照文件。"""
        snapshot_dir = self._get_sandbox_dir(sandbox_id) / "snapshots"
        if snapshot_dir.is_dir():
            await asyncio.to_thread(shutil.rmtree, snapshot_dir)
            logger.debug(f"Deleted snapshot directory: {snapshot_dir}")

    async def delete_snapshot(self, sandbox_id: UUID, snapshot_id: UUID) -> None:
        """异步删除一个指定的快照文件。"""
        file_path = self._get_sandbox_dir(sandbox_id) / "snapshots" / f"{snapshot_id}.json"
        if file_path.is_file():
            try:
                # 使用 os.remove 比 shutil.rmtree 更适合删除文件
                await asyncio.to_thread(os.remove, file_path)
                logger.debug(f"Deleted snapshot file: {file_path}")
            except FileNotFoundError:
                # 如果在检查和删除之间文件消失了，这不是一个错误
                pass
            except Exception as e:
                logger.error(f"Error deleting snapshot file {file_path}: {e}")
                # 重新抛出，让上层知道操作失败
                raise
        
    async def list_assets(self, asset_type: AssetType) -> List[str]:
        """Lists all assets of a given type by scanning the assets directory."""
        if asset_type == AssetType.SANDBOX:
             # Sandboxes are directories, not files with extensions
            return await self.list_sandbox_ids()

        ext = FILE_EXTENSIONS.get(asset_type)
        if not ext:
            raise ValueError(f"Unknown asset type '{asset_type}' with no defined file extension.")

        # For other asset types, we'd define their storage location.
        # As of now, only sandboxes are fully implemented, so we return empty for others.
        # For example: search_dir = self.assets_base_dir / asset_type.value
        # This implementation assumes other assets aren't stored yet.
        return []

    async def _embed_zip_in_png(self, zip_bytes: bytes, base_image_bytes: Optional[bytes] = None) -> bytes:
        def _sync_embed():
            encoded_data = base64.b64encode(zip_bytes).decode('ascii')
            if base_image_bytes: image = Image.open(io.BytesIO(base_image_bytes))
            else: image = Image.new('RGBA', (1, 1), (0, 0, 0, 255))
            png_info_obj = PngImagePlugin.PngInfo()
            png_info_obj.add_text("hevno:data", encoded_data, zip=True)
            buffer = io.BytesIO()
            image.save(buffer, "PNG", pnginfo=png_info_obj)
            return buffer.getvalue()
        return await asyncio.to_thread(_sync_embed)

    async def _extract_zip_from_png(self, png_bytes: bytes) -> Tuple[bytes, bytes]:
        def _sync_extract():
            try:
                image = Image.open(io.BytesIO(png_bytes))
                image.load()
                encoded_data = image.text.get("hevno:data")
                if encoded_data is None: raise ValueError("Invalid Hevno package: 'hevno:data' chunk not found.")
                zip_data = base64.b64decode(encoded_data)
                return zip_data, png_bytes
            except Exception as e:
                logger.error(f"[EXTRACT] Exception while processing PNG: {e}", exc_info=True)
                raise ValueError(f"Failed to process PNG file: {e}") from e
        return await asyncio.to_thread(_sync_extract)

    async def save_sandbox_icon(self, sandbox_id: str, icon_bytes: bytes) -> Path:
        icon_path = self.assets_base_dir / "sandbox_icons" / f"{sandbox_id}.png"
        icon_path.parent.mkdir(parents=True, exist_ok=True)
        async with aiofiles.open(icon_path, 'wb') as f:
            await f.write(icon_bytes)
        logger.info(f"Saved icon for sandbox {sandbox_id} to {icon_path}")
        return icon_path

    async def export_package(self, manifest: PackageManifest, data_files: Dict[str, Any], base_image_bytes: Optional[bytes] = None) -> bytes:
        def _sync_zip():
            zip_buffer = io.BytesIO()
            with zipfile.ZipFile(zip_buffer, 'w', zipfile.ZIP_DEFLATED) as zf:
                zf.writestr('manifest.json', manifest.model_dump_json(indent=2))
                for filename, data_dict in data_files.items():
                    file_content = json.dumps(data_dict, indent=2, ensure_ascii=False)
                    zf.writestr(f'data/{filename}', file_content)
            return zip_buffer.getvalue()
        
        zip_bytes = await asyncio.to_thread(_sync_zip)
        return await self._embed_zip_in_png(zip_bytes, base_image_bytes)

    async def import_package(self, package_bytes: bytes) -> Tuple[PackageManifest, Dict[str, str], bytes]:
        zip_bytes, png_bytes = await self._extract_zip_from_png(package_bytes)
        def _sync_unzip():
            data_files: Dict[str, str] = {}
            with zipfile.ZipFile(io.BytesIO(zip_bytes), 'r') as zf:
                try:
                    manifest_content = zf.read('manifest.json').decode('utf-8')
                    manifest = PackageManifest.model_validate_json(manifest_content)
                except KeyError: raise ValueError("Package is missing 'manifest.json'.")
                except (ValidationError, json.JSONDecodeError) as e: raise ValueError(f"Invalid 'manifest.json': {e}") from e
                for item in zf.infolist():
                    if item.filename.startswith('data/') and not item.is_dir():
                        relative_path = item.filename.split('data/', 1)[1]
                        data_files[relative_path] = zf.read(item).decode('utf-8')
            return manifest, data_files
        manifest, data_files = await asyncio.to_thread(_sync_unzip)
        return manifest, data_files, png_bytes
        
    def get_sandbox_icon_path(self, sandbox_id: str) -> Optional[Path]:
        icon_path = self.assets_base_dir / "sandbox_icons" / f"{sandbox_id}.png"
        return icon_path if icon_path.is_file() else None

    def get_default_icon_path(self) -> Path:
        return self.assets_base_dir / "default_sandbox_icon.png"
```

### models.py
```
# plugins/core_persistence/models.py

from enum import Enum

# --- 文件约定 (插件内部实现细节) ---
class AssetType(str, Enum):
    GRAPH = "graph"
    CODEX = "codex"
    SANDBOX = "sandbox"

FILE_EXTENSIONS = {
    AssetType.GRAPH: ".graph.hevno.json",
    AssetType.CODEX: ".codex.hevno.json",
}

```

### __init__.py
```
# plugins/core_persistence/__init__.py
import os
import logging

from backend.core.contracts import Container, HookManager
from .service import PersistenceService
from .stores import PersistentSandboxStore, PersistentSnapshotStore
from .api import persistence_router

logger = logging.getLogger(__name__)

def _create_persistent_sandbox_store(container: Container) -> PersistentSandboxStore:
    return PersistentSandboxStore(container.resolve("persistence_service"))

def _create_persistent_snapshot_store(container: Container) -> PersistentSnapshotStore:
    return PersistentSnapshotStore(container.resolve("persistence_service"))

def _create_persistence_service() -> PersistenceService:
    assets_dir = os.getenv("HEVNO_ASSETS_DIR", "assets")
    return PersistenceService(assets_base_dir=assets_dir)

async def provide_router(routers: list) -> list:
    routers.append(persistence_router)
    logger.debug("Provided 'persistence_router' to the application.")
    return routers

async def initialize_stores(container: Container):
    """钩子实现: 在所有服务注册后，异步初始化持久化存储。"""
    logger.info("Initializing persistent stores...")
    sandbox_store: PersistentSandboxStore = container.resolve("sandbox_store")
    
    sandbox_store.set_container(container)
    
    await sandbox_store.initialize()

def register_plugin(container: Container, hook_manager: HookManager):
    logger.info("--> 正在注册 [core_persistence] 插件...")
    container.register(
        "persistence_service", _create_persistence_service, singleton=True
    )
    container.register(
        "sandbox_store", _create_persistent_sandbox_store, singleton=True
    )
    container.register(
        "snapshot_store", _create_persistent_snapshot_store, singleton=True
    )
    logger.debug(
        "Registered 'sandbox_store' and 'snapshot_store' with persistent implementations."
    )
    hook_manager.add_implementation(
        "collect_api_routers", provide_router, plugin_name="core_persistence"
    )
    hook_manager.add_implementation(
        "services_post_register",
        initialize_stores,
        priority=90, 
        plugin_name="core_persistence",
    )
    logger.info("插件 [core_persistence] 注册成功。")
```

### stores.py
```
# plugins/core_persistence/stores.py
import asyncio
import logging
from typing import Dict, List, Optional
from uuid import UUID

# 从 core_engine 导入接口定义
from plugins.core_engine.contracts import Sandbox, StateSnapshot, SnapshotStoreInterface, SandboxStoreInterface
from backend.core.serialization import pickle_fallback_encoder
from .contracts import PersistenceServiceInterface
from pydantic import ValidationError

logger = logging.getLogger(__name__)

# 继承自 SandboxStoreInterface
class PersistentSandboxStore(SandboxStoreInterface):
    def __init__(self, persistence_service: PersistenceServiceInterface):
        self._persistence = persistence_service
        self._cache: Dict[UUID, Sandbox] = {}
        self._locks: Dict[UUID, asyncio.Lock] = {}
        # 为 SnapshotStore 添加一个依赖，以便在删除时可以调用它
        self._snapshot_store: Optional[SnapshotStoreInterface] = None
        self._container: Optional['Container'] = None # type: ignore
        logger.info("PersistentSandboxStore initialized (cache is empty).")

    # 提供一种方式来注入容器，以便稍后解析 snapshot_store
    def set_container(self, container: 'Container'): # type: ignore
        self._container = container

    def _get_lock(self, sandbox_id: UUID) -> asyncio.Lock:
        if sandbox_id not in self._locks:
            self._locks.setdefault(sandbox_id, asyncio.Lock())
        return self._locks[sandbox_id]
        
    async def initialize(self):
        logger.info("Pre-loading all sandboxes and their snapshots from disk into cache...")
        count = 0
        
        # 1. 在初始化开始时，确保我们能访问到 snapshot_store
        if not self._container:
             logger.error("Container not set on PersistentSandboxStore. Cannot pre-load snapshots.")
             return
        
        # 解析一次并缓存，避免在循环中重复解析
        snapshot_store: SnapshotStoreInterface = self._container.resolve("snapshot_store")
        
        sandbox_ids = await self._persistence.list_sandbox_ids()
        for sid_str in sandbox_ids:
            try:
                sid = UUID(sid_str)
                data = await self._persistence.load_sandbox(sid)
                if data:
                    sandbox = Sandbox.model_validate(data)
                    self._cache[sid] = sandbox
                    count += 1
                    
                    # 2. 对于每一个加载的沙盒，立即让 snapshot_store 去加载它所有的快照
                    #    find_by_sandbox 会自动将加载的快照放入 snapshot_store 的缓存中
                    logger.debug(f"Pre-loading snapshots for sandbox {sid}...")
                    await snapshot_store.find_by_sandbox(sid)

            except (ValueError, FileNotFoundError, ValidationError) as e:
                logger.warning(f"Skipping invalid sandbox directory '{sid_str}': {e}")
        logger.info(f"Successfully pre-loaded {count} sandboxes and their associated snapshots into cache.")

    async def save(self, sandbox: Sandbox):
        lock = self._get_lock(sandbox.id)
        async with lock:
            # 使用 mode='json' 并提供 fallback 函数
            data = sandbox.model_dump(mode='json', fallback=pickle_fallback_encoder)

            await self._persistence.save_sandbox(sandbox.id, data)
            self._cache[sandbox.id] = sandbox
            
    def get(self, key: UUID) -> Optional[Sandbox]:
        return self._cache.get(key)

    async def delete(self, key: UUID):
        lock = self._get_lock(key)
        async with lock:
            # 正确地从容器中解析 snapshot_store 并调用其方法
            if self._container:
                if not self._snapshot_store:
                    self._snapshot_store = self._container.resolve("snapshot_store")
                await self._snapshot_store.delete_all_for_sandbox(key)
            else:
                 logger.warning("Container not set on PersistentSandboxStore, cannot delete snapshots automatically.")

            await self._persistence.delete_sandbox(key)
            self._cache.pop(key, None)
            self._locks.pop(key, None)

    def values(self) -> List[Sandbox]:
        return list(self._cache.values())
        
    def __contains__(self, key: UUID) -> bool:
        return key in self._cache
    
    def clear(self) -> None:
        logger.warning("`clear` called on PersistentSandboxStore, but it does nothing to disk state. Cache is NOT cleared.")
        pass


class PersistentSnapshotStore(SnapshotStoreInterface):
    """
    管理快照的持久化和缓存。
    - 快照按需从磁盘加载。
    - 所有对持久化层的调用现在都是非阻塞的。
    """
    def __init__(self, persistence_service: PersistenceServiceInterface):
        self._persistence = persistence_service
        self._cache: Dict[UUID, StateSnapshot] = {}
        # 为每个快照ID创建一个独立的锁，以实现原子性保存
        self._locks: Dict[UUID, asyncio.Lock] = {}
        logger.info("PersistentSnapshotStore initialized.")

    def _get_lock(self, snapshot_id: UUID) -> asyncio.Lock:
        return self._locks.setdefault(snapshot_id, asyncio.Lock())

    async def save(self, snapshot: StateSnapshot) -> None:
        """异步保存快照到磁盘并更新缓存。"""
        lock = self._get_lock(snapshot.id)
        async with lock:
            # 同样，使用 mode='json' 并提供 fallback 函数
            data = snapshot.model_dump(mode='json', fallback=pickle_fallback_encoder)
          
            await self._persistence.save_snapshot(snapshot.sandbox_id, snapshot.id, data)
            self._cache[snapshot.id] = snapshot

    def get(self, snapshot_id: UUID) -> Optional[StateSnapshot]:
        """
        从缓存中同步获取快照。
        注意：此方法不会从磁盘加载。它依赖于 find_by_sandbox 或 save 来填充缓存。
        这是一个设计权衡，以避免在get()中需要sandbox_id。
        """
        return self._cache.get(snapshot_id)

    async def find_by_sandbox(self, sandbox_id: UUID) -> List[StateSnapshot]:
        """异步加载属于特定沙盒的所有快照，并更新缓存。"""
        # 使用 await 调用异步的 load_all_snapshots_for_sandbox
        snapshots_data = await self._persistence.load_all_snapshots_for_sandbox(sandbox_id)
        for data in snapshots_data:
            try:
                s = StateSnapshot.model_validate(data)
                self._cache[s.id] = s
            except ValidationError as e:
                logger.warning(f"Skipping snapshot with invalid data for sandbox {sandbox_id}: {e}")
        
        # 即使磁盘上没有，也要确保返回缓存中可能存在的（例如，刚创建还未写入的）
        relevant_snapshots = [s for s in self._cache.values() if s.sandbox_id == sandbox_id]
        # 去重，以防万一
        unique_snapshots = {s.id: s for s in relevant_snapshots}.values()
        return sorted(list(unique_snapshots), key=lambda s: s.created_at)

    async def delete_all_for_sandbox(self, sandbox_id: UUID) -> None:
        """实现接口中新加的方法"""
        await self._persistence.delete_all_for_sandbox(sandbox_id)
        # 从缓存中也移除
        ids_to_remove = [sid for sid, s in self._cache.items() if s.sandbox_id == sandbox_id]
        for sid in ids_to_remove:
            self._cache.pop(sid, None)
            self._locks.pop(sid, None)

    async def delete(self, snapshot_id: UUID) -> None:
        """异步删除指定的快照，包括其持久化文件和缓存条目。"""
        snapshot = self.get(snapshot_id)
        if not snapshot:
            # 如果快照不存在，静默返回，因为目标已经达成
            return
            
        lock = self._get_lock(snapshot_id)
        async with lock:
            await self._persistence.delete_snapshot(snapshot.sandbox_id, snapshot.id)
            # 从缓存和锁字典中移除
            self._cache.pop(snapshot_id, None)
            self._locks.pop(snapshot_id, None)
            logger.info(f"Deleted snapshot {snapshot_id} from persistence and cache.")


    def clear(self) -> None:
        """此操作在持久化存储中无意义，记录警告并忽略。"""
        logger.warning("`clear` called on PersistentSnapshotStore, but it does nothing to disk state. Cache is NOT cleared.")
        pass
```

### api.py
```
# plugins/core_persistence/api.py

import logging
from typing import List
from fastapi import APIRouter, Depends, HTTPException

from backend.core.dependencies import Service
from .contracts import PersistenceServiceInterface 
from .models import AssetType

logger = logging.getLogger(__name__)

# --- Router for persistence API ---
persistence_router = APIRouter(
    prefix="/api/persistence", 
    tags=["core_Persistence"]
)

@persistence_router.get("/assets/{asset_type}", response_model=List[str])
async def list_assets_by_type(
    asset_type: AssetType,
    service: PersistenceServiceInterface = Depends(Service("persistence_service")) 
):
    """
    列出指定类型的所有已保存资产的名称。
    """
    try:
        return await service.list_assets(asset_type)
    except Exception as e:
        logger.error(f"Failed to list assets of type '{asset_type.value}': {e}", exc_info=True)
        raise HTTPException(status_code=500, detail="An error occurred while listing assets.")
```

### manifest.json
```
{
    "id": "core_persistence",
    "name": "core_persistence",
    "version": "1.0.0",
    "description": "Provides file system persistence, asset management, and package import/export.",
    "author": "Hevno Team",
    "backend": {
        "priority": 10
    }
}
```

### contracts.py
```
# plugins/core_persistence/contracts.py

from __future__ import annotations
from abc import ABC, abstractmethod
from typing import Dict, Tuple, TypeVar, List, Any, Optional
from uuid import UUID
from pathlib import Path
from pydantic import BaseModel, Field
from enum import Enum
from datetime import datetime, timezone

# 不再从 core_engine.contracts 导入 Sandbox 和 StateSnapshot
# from plugins.core_engine.contracts import Sandbox, StateSnapshot

T = TypeVar('T', bound=BaseModel)

# --- 共享数据模型 (Package/Manifest) ---

class PackageType(str, Enum):
    """定义了可以被导入/导出的不同类型的包。"""
    SANDBOX_ARCHIVE = "sandbox_archive"
    GRAPH_COLLECTION = "graph_collection"
    CODEX_COLLECTION = "codex_collection"

class PluginRequirement(BaseModel):
    """描述包所依赖的插件。"""
    name: str = Field(..., description="Plugin identifier, e.g., 'hevno-dice-roller'")
    source_url: str = Field(..., description="Plugin source, e.g., 'https://github.com/user/repo'")
    version: str = Field(..., description="Compatible version or Git ref")

class PackageManifest(BaseModel):
    """
    定义了 .hevno.zip 包内容的标准清单。
    这是核心的共享数据模型。
    """
    format_version: str = Field(default="1.0")
    package_type: PackageType
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    entry_point: str
    required_plugins: List[PluginRequirement] = Field(default_factory=list)
    metadata: Dict[str, Any] = Field(default_factory=dict)


# --- 服务接口 (合并并异步化) ---

class PersistenceServiceInterface(ABC):
    """
    定义了核心持久化服务的文件系统I/O能力。
    注意：它不再知道 Sandbox 或 StateSnapshot 的具体模型，
    而是处理通用的字典数据，使得依赖关系更清晰。
    """

    # --- 沙盒持久化方法 ---
    @abstractmethod
    async def save_sandbox(self, sandbox_id: UUID, data: Dict[str, Any]) -> None:
        raise NotImplementedError

    @abstractmethod
    async def load_sandbox(self, sandbox_id: UUID) -> Optional[Dict[str, Any]]:
        raise NotImplementedError

    @abstractmethod
    async def delete_sandbox(self, sandbox_id: UUID) -> None:
        raise NotImplementedError

    @abstractmethod
    async def list_sandbox_ids(self) -> List[str]:
        raise NotImplementedError

    # --- 快照持久化方法 ---
    @abstractmethod
    async def save_snapshot(self, sandbox_id: UUID, snapshot_id: UUID, data: Dict[str, Any]) -> None:
        raise NotImplementedError

    @abstractmethod
    async def load_snapshot(self, sandbox_id: UUID, snapshot_id: UUID) -> Optional[Dict[str, Any]]:
        raise NotImplementedError

    @abstractmethod
    async def load_all_snapshots_for_sandbox(self, sandbox_id: UUID) -> List[Dict[str, Any]]:
        raise NotImplementedError

    @abstractmethod
    async def delete_all_for_sandbox(self, sandbox_id: UUID) -> None:
        raise NotImplementedError
    
    # --- 包导入/导出方法 ---
    @abstractmethod
    async def export_package(self, manifest: PackageManifest, data_files: Dict[str, Any], base_image_bytes: Optional[bytes] = None) -> bytes:
        raise NotImplementedError

    @abstractmethod
    async def import_package(self, package_bytes: bytes) -> Tuple[PackageManifest, Dict[str, str], bytes]:
        raise NotImplementedError
    
    # --- 沙盒图标处理方法 ---
    @abstractmethod
    async def save_sandbox_icon(self, sandbox_id: str, icon_bytes: bytes) -> Path:
        raise NotImplementedError

    @abstractmethod
    def get_sandbox_icon_path(self, sandbox_id: str) -> Optional[Path]:
        raise NotImplementedError
    
    @abstractmethod
    def get_default_icon_path(self) -> Path:
        raise NotImplementedError

    @property
    @abstractmethod
    def sandboxes_root_dir(self) -> Path:
        raise NotImplementedError

    @abstractmethod
    async def delete_snapshot(self, sandbox_id: UUID, snapshot_id: UUID) -> None:
        """异步删除一个指定的快照文件。"""
        raise NotImplementedError
```

# Directory: frontend

### HookManager.js
```
// frontend/HookManager.js

/**
 * 一个支持在前后端之间进行智能路由的事件总线。
 * 它查询一个全局注册表来决定将事件发送到何处。
 */
export class HookManager {
  constructor() {
    /** @type {Map<string, Function[]>} */
    this.hooks = new Map();
    
    /** @type {import('./RemoteHookProxy.js').RemoteHookProxy | null} */
    this.remoteProxy = null;
    /** @type {import('./services/GlobalHookRegistry.js').GlobalHookRegistry | null} */
    this.globalRegistry = null;
  }

  /**
   * 在实例化后注入依赖，以解决循环依赖问题。
   * @param {import('./RemoteHookProxy.js').RemoteHookProxy} remoteProxy
   * @param {import('./services/GlobalHookRegistry.js').GlobalHookRegistry} globalRegistry
   */
  setDependencies(remoteProxy, globalRegistry) {
    this.remoteProxy = remoteProxy;
    this.globalRegistry = globalRegistry;
  }
  
  /**
   * 注册一个钩子实现，并通知全局注册表。
   * @param {string} hookName - 钩子的名称。
   * @param {Function} implementation - 要执行的函数。
   */
  addImplementation(hookName, implementation) {
    if (!this.hooks.has(hookName)) {
      this.hooks.set(hookName, []);
    }
    this.hooks.get(hookName).push(implementation);

    // 任务 4.3: 通知全局注册表这个新的本地钩子
    if (this.globalRegistry) {
        this.globalRegistry.addFrontendHook(hookName);
    }
    
    console.log(`[HookManager] ADDED listener for hook: '${hookName}'`);
  }

  /**
   * 使用智能路由触发一个钩子。
   * 它会判断钩子应该在本地执行、远程执行，还是两者都执行。
   * 注意: 'filter' 类型的钩子目前仍被视为仅本地操作。
   * @param {string} hookName - 钩子的名称。
   * @param {object} data - 钩子的数据负载。
   */
  async trigger(hookName, data = {}) {
    if (!this.globalRegistry || !this.remoteProxy) {
        console.error(`[HookManager] 无法触发 '${hookName}'。核心服务未注入。`);
        return;
    }
    
    // 任务 4.3: 查询注册表
    const isLocal = this.globalRegistry.isLocalHook(hookName);
    const isRemote = this.globalRegistry.isRemoteHook(hookName);

    console.log(`[HookManager] TRIGGERING '${hookName}'. Local: ${isLocal}, Remote: ${isRemote}`);

    let wasHandled = false;

    // 任务 4.3: 路由到本地实现
    if (isLocal) {
      const implementations = this.hooks.get(hookName) || [];
      const tasks = implementations.map(impl => Promise.resolve(impl(data)));
      await Promise.all(tasks);
      wasHandled = true;
    }

    // 任务 4.3: 路由到远程（后端）实现
    if (isRemote) {
      this.remoteProxy.trigger(hookName, data);
      wasHandled = true;
    }
    
    // 任务 4.3: 如果在任何地方都未找到处理程序，则发出警告
    if (!wasHandled) {
      console.warn(`[HookManager] 触发的钩子 '${hookName}' 在前端或后端都没有已知的实现。`);
    }
  }

  /**
   * 触发一个“过滤型”钩子 (保留用于本地功能)。
   * 此类型被假定为同步和本地的，不用于远程通信。
   * @param {string} hookName 
   * @param {*} initialData 
   * @param {object} extraData 
   * @returns {*}
   */
  async filter(hookName, initialData, extraData = {}) {
    const implementations = this.hooks.get(hookName) || [];
    let currentData = initialData;
    for (const impl of implementations) {
      currentData = await Promise.resolve(impl(currentData, extraData));
    }
    return currentData;
  }
  
  /**
   * 获取所有已注册的前端钩子名称。
   * @returns {string[]}
   */
  getAllHookNames() {
    return Array.from(this.hooks.keys());
  }

  /**
   * 移除一个已注册的钩子实现。
   * @param {string} hookName - 钩子的名称。
   * @param {Function} implementationToRemove - 要移除的函数实例。
   */
  removeImplementation(hookName, implementationToRemove) {
    const implementations = this.hooks.get(hookName);
    if (!implementations) {
        return;
    }
    const index = implementations.indexOf(implementationToRemove);
    if (index > -1) {
        implementations.splice(index, 1);
        console.log(`[HookManager] REMOVED listener for hook: '${hookName}'`);
    }
  }
}
```

### main.js
```
// frontend/main.js

import { ServiceContainer } from './ServiceContainer.js';
import { HookManager } from './HookManager.js';
import { RemoteHookProxy } from './RemoteHookProxy.js';
import { ManifestProvider } from './ManifestProvider.js';
import { GlobalHookRegistry } from './services/GlobalHookRegistry.js';

class FrontendLoader {
  constructor() {
    this.services = new ServiceContainer();
    window.Hevno = { services: this.services };

    const hookManager = new HookManager();
    const remoteProxy = new RemoteHookProxy();
    const globalHookRegistry = new GlobalHookRegistry();
    const manifestProvider = new ManifestProvider();
    
    hookManager.setDependencies(remoteProxy, globalHookRegistry);
    remoteProxy.setHookManager(hookManager);

    this.services.register('hookManager', hookManager, 'loader');
    this.services.register('remoteProxy', remoteProxy, 'loader');
    this.services.register('globalHookRegistry', globalHookRegistry, 'loader');
    this.services.register('manifestProvider', manifestProvider, 'loader');

    if (import.meta.env.DEV) {
      window.hevno = this.services;
    }
  }

  async load() {
    console.log("🚀 Hevno Frontend Loader starting...");
    const remoteProxy = this.services.get('remoteProxy');
    const globalHookRegistry = this.services.get('globalHookRegistry');
    const manifestProvider = this.services.get('manifestProvider');

    try {
      console.log("[Loader] 正在获取后端钩子清单...");
      const hooksResponse = await fetch('/api/system/hooks/manifest');
      if (!hooksResponse.ok) {
          throw new Error(`无法获取后端钩子清单: ${hooksResponse.statusText}`);
      }
      const backendHooksData = await hooksResponse.json();
      globalHookRegistry.setBackendHooks(backendHooksData.hooks);
      
      remoteProxy.connect();

      console.log("[Loader] 正在获取插件清单...");
      const manifestResponse = await fetch('/api/plugins/manifest');
      if (!manifestResponse.ok) {
        throw new Error(`无法获取插件清单: ${manifestResponse.statusText}`);
      }
      let allManifests = await manifestResponse.json();
      
      const frontendPlugins = allManifests
        .filter(m => m.frontend && m.frontend.entryPoint)
        .sort((a, b) => (a.frontend?.priority || 0) - (b.frontend?.priority || 0));

      console.log(`发现 ${frontendPlugins.length} 个前端插件待加载:`, frontendPlugins.map(p => p.id));


      for (const manifest of frontendPlugins) {
        manifestProvider.addManifest(manifest);
        try {
          let entryPointUrl = '';

          if (import.meta.env.DEV) {
            const srcEntryPoint = manifest.frontend.srcEntryPoint || `src/main.${manifest.id.includes('goliath') ? 'jsx' : 'js'}`;
            entryPointUrl = `/plugins/${manifest.id}/${srcEntryPoint}`;
            console.log(`[DEV MODE] Loading source for ${manifest.id}: ${entryPointUrl}`);
          } else {
            entryPointUrl = `/plugins/${manifest.id}/${manifest.frontend.entryPoint}`;
          }
          
          const pluginModule = await import(entryPointUrl);
          
          if (pluginModule.registerPlugin) {
            const pluginModule = await import(/* @vite-ignore */ entryPointUrl);
            console.log(`-> 正在注册插件: ${manifest.id} (priority: ${manifest.frontend?.priority || 0})`);
            await Promise.resolve(pluginModule.registerPlugin(this.services));
          }
        } catch (e) {
          console.error(`加载或注册插件 ${manifest.id} 失败:`, e);
        }
      }
      // ===============================================================

      console.log("[Loader] 所有插件已加载。正在与后端同步前端钩子...");
      remoteProxy.syncFrontendHooks();

    } catch (e) {
      console.error("致命错误: 无法初始化插件。", e);
      document.body.innerHTML = `<div style="color: red; padding: 2rem;">错误: 无法从后端加载插件清单。后端服务器是否正在运行？</div>`;
      return;
    }

    console.log("✅ 同步完成。正在将控制权移交给应用插件...");
    await this.services.get('hookManager').trigger('loader.ready');
  }
}

const loader = new FrontendLoader();
loader.load();
```

### RemoteHookProxy.js
```
// ./frontend/RemoteHookProxy.js

/**
 * 负责管理与后端 WebSocket 的连接，并作为前后端钩子系统的桥梁。
 * 它在连接建立后，向后端同步前端的钩子清单。
 */
export class RemoteHookProxy {
  constructor() {
    /** @type {import('./HookManager.js').HookManager | null} */
    this.localHookManager = null;
    this.ws = null;
    this.isConnected = false;
    console.log(`[RemoteProxy] CONSTRUCTED. Initial isConnected: ${this.isConnected}`);
  }

  /**
   * 注入 HookManager 依赖。
   * @param {import('./HookManager.js').HookManager} hookManager 
   */
  setHookManager(hookManager) {
    this.localHookManager = hookManager;
  }

  connect() {
    if (this.ws) {
        // 防止重复连接
        return;
    }
    const wsProtocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
    const wsUrl = `${wsProtocol}//${window.location.host}/ws/hooks`;
    
    this.ws = new WebSocket(wsUrl);

    this.ws.onopen = () => {
      console.log("🔗 WebSocket connection established.");
      this.isConnected = true;
      if (this.localHookManager) {
        // 在 `addImplementation` 调用时，`globalHookRegistry` 已经知道这个钩子了
        this.localHookManager.addImplementation('websocket.connected', () => {});
        this.localHookManager.trigger('websocket.connected');
      }

      // 【已移除】不再在此处同步钩子。
      // this.syncFrontendHooks(); 
    };
    
    this.ws.onmessage = (event) => this.handleIncoming(event);
    
    this.ws.onclose = () => {
      console.warn("🔌 WebSocket connection closed. Attempting to reconnect in 3 seconds...");
      if (this.isConnected) {
        this.isConnected = false;
        if (this.localHookManager) {
            // 确保钩子存在
            this.localHookManager.addImplementation('websocket.disconnected', () => {});
            this.localHookManager.trigger('websocket.disconnected');
        }
      }
      this.ws = null; // 清理实例以允许重新连接
      setTimeout(() => this.connect(), 3000);
    };

    this.ws.onerror = (error) => {
      console.error("WebSocket error:", error);
    };
  }
  
  /**
   * 将前端实现的钩子清单发送到后端。
   */
  syncFrontendHooks() {
    if (!this.localHookManager) {
        console.error("[RemoteProxy] 无法同步钩子，HookManager 未设置。");
        return;
    }
    // 添加一个延迟/重试机制，以防 `sync` 被调用时 WS 尚未完全打开
    const trySync = (retries = 5) => {
      if (this.ws && this.ws.readyState === WebSocket.OPEN) {
          const hookNamesArray = this.localHookManager.getAllHookNames();
          const payload = {
              type: 'sync_hooks', // 特殊类型
              hooks: hookNamesArray
          };
          const message = JSON.stringify(payload);
          console.log(`[ws >] 正在与后端同步 ${hookNamesArray.length} 个前端钩子。`);
          this.ws.send(message);
      } else if (retries > 0) {
          console.warn(`[RemoteProxy] WebSocket 未打开，将在 200ms 后重试同步 (剩余次数: ${retries - 1})`);
          setTimeout(() => trySync(retries - 1), 200);
      } else {
          console.error("[RemoteProxy] 无法同步钩子: WebSocket 未打开且已达到重试次数上限。");
      }
    }
    trySync();
  }

  handleIncoming(event) {
    try {
      const payload = JSON.parse(event.data);
      if (payload.hook_name) {
        console.log(`[ws <] 收到远程钩子: ${payload.hook_name}`, payload.data);
        if (this.localHookManager) {
          // 关键：直接执行本地实现，绕过智能路由，以防止无限循环。
          // 后端的 HookManager.trigger 已经确定这个钩子应该在前端本地运行。
          const implementations = this.localHookManager.hooks.get(payload.hook_name) || [];
          const tasks = implementations.map(impl => Promise.resolve(impl(payload.data || {})));
          Promise.all(tasks);
        }
      }
    } catch (e) {
      console.error("解析传入的 WebSocket 消息失败:", e);
    }
  }

  /**
   * 将一个钩子触发消息发送到后端。
   * 这个方法由本地 HookManager 的智能路由逻辑调用。
   * @param {string} hookName 
   * @param {object} data 
   */
  trigger(hookName, data = {}) {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      const payload = { hook_name: hookName, data };
      console.log(`[ws >] 正在触发远程钩子: ${hookName}`, data);
      this.ws.send(JSON.stringify(payload));
    } else {
      console.error("无法触发远程钩子: WebSocket 未打开。");
    }
  }
}
```

### ManifestProvider.js
```
// /frontend/ManifestProvider.js

/**
 * 在插件加载期间收集所有前端插件的清单文件内容。
 * 这是一个无逻辑的数据容器，由内核填充，由应用主控插件消费。
 */
export class ManifestProvider {
    constructor() {
        this.manifests = [];
    }

    /**
     * 由内核在加载每个插件时调用。
     * @param {object} manifest - 一个插件的 manifest.json 内容。
     */
    addManifest(manifest) {
        this.manifests.push(manifest);
    }

    /**
     * 由应用主控插件调用，以获取所有已加载插件的清单。
     * @returns {Array<object>}
     */
    getManifests() {
        return this.manifests;
    }
}
```

### ServiceContainer.js
```
// /frontend/ServiceContainer.js

/**
 * 一个简单的依赖注入(DI)容器，用于管理单例服务。
 * 确保服务的注册、获取和覆盖是明确且可追踪的。
 */
export class ServiceContainer {
    constructor() {
        this.serviceInstances = new Map();
        this.serviceProviders = new Map(); // 用于追踪哪个插件提供了服务
    }

    /**
     * 向容器注册一个服务实例。
     * @param {string} serviceName - 服务的唯一名称。
     * @param {*} serviceInstance - 服务的实例。
     * @param {string} pluginId - 提供此服务的插件ID。
     */
    register(serviceName, serviceInstance, pluginId) {
        if (this.serviceInstances.has(serviceName)) {
            const originalProvider = this.serviceProviders.get(serviceName);
            console.warn(`[Services] Service '${serviceName}' (provided by '${originalProvider}') is being overridden by plugin '${pluginId}'.`);
        }
        this.serviceInstances.set(serviceName, serviceInstance);
        this.serviceProviders.set(serviceName, pluginId);
        console.log(`[Services] Service '${serviceName}' registered by plugin '${pluginId}'.`);
    }

    /**
     * 从容器中获取一个服务实例。
     * @param {string} serviceName - 服务的名称。
     * @returns {*} 服务实例，如果不存在则返回 undefined。
     */
    get(serviceName) {
        const service = this.serviceInstances.get(serviceName);
        if (!service) {
            // 在开发中，这是一个有用的警告，可以帮助快速定位问题。
            console.warn(`[Services] Service '${serviceName}' was requested but has not been registered.`);
        }
        return service;
    }

    /**
     * 检查一个服务是否已被注册。
     * @param {string} serviceName - 服务的名称。
     * @returns {boolean}
     */
    has(serviceName) {
        return this.serviceInstances.has(serviceName);
    }
}
```

### services/GlobalHookRegistry.js
```
// frontend/services/GlobalHookRegistry.js

/**
 * 一个单例服务，用于存储和查询全域钩子路由表。
 * 它持有在前端和后端实现的所有钩子的完整清单。
 */
export class GlobalHookRegistry {
  constructor() {
    /** @type {Set<string>} */
    this.backendHooks = new Set();
    /** @type {Set<string>} */
    this.frontendHooks = new Set();
  }

  /**
   * 填充已知的后端钩子集合。在启动时调用。
   * @param {string[]} hooks - 来自后端的钩子名称数组。
   */
  setBackendHooks(hooks) {
    this.backendHooks = new Set(hooks);
    console.log(`[GlobalRegistry] 已注册 ${this.backendHooks.size} 个后端钩子。`);
  }

  /**
   * 将一个前端实现的钩子添加到注册表。
   * 由本地 HookManager 在每次添加实现时调用。
   * @param {string} hookName 
   */
  addFrontendHook(hookName) {
    if (!this.frontendHooks.has(hookName)) {
        this.frontendHooks.add(hookName);
    }
  }

  /**
   * 检查一个钩子是否有本地（前端）实现。
   * @param {string} hookName 
   * @returns {boolean}
   */
  isLocalHook(hookName) {
    return this.frontendHooks.has(hookName);
  }

  /**
   * 检查一个钩子是否有远程（后端）实现。
   * @param {string} hookName 
   * @returns {boolean}
   */
  isRemoteHook(hookName) {
    return this.backendHooks.has(hookName);
  }

  /**
   * 获取所有已知的前端钩子名称列表。
   * 用于与后端同步。
   * @returns {string[]}
   */
  getFrontendHooks() {
    return Array.from(this.frontendHooks);
  }
}
```

# Directory: plugins/core_layout

### vite.config.js
```
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
  plugins: [
    react({
      jsxRuntime: 'automatic'
    }),
  ],
  build: {
    lib: {
      entry: resolve(__dirname, 'src/main.jsx'),
      name: 'HevnoCoreLayout',
      fileName: 'main',
      formats: ['es'],
    },
    outDir: 'dist',
    emptyOutDir: true,
  },
});
```

### package.json
```
{
  "name": "hevno-plugin-core-layout",
  "version": "2.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "vite build"
  },
  "dependencies": {
    "@emotion/react": "latest",
    "@emotion/styled": "latest",
    "@mui/icons-material": "latest",
    "@mui/material": "latest",
    "react": "latest",
    "react-dom": "latest",
    "react-draggable": "^4.5.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "latest",
    "vite": "latest"
  }
}

```

### manifest.json
```
{
    "id": "core_layout",
    "name": "React Application Host",
    "version": "2.0.0",
    "description": "Provides the main application shell using React, including layout, theme, and a host for other page plugins.",
    "author": "Hevno Team",
    "frontend": {
        "type": "host",
        "priority": 100,
        "entryPoint": "dist/main.js",
        "srcEntryPoint": "src/main.jsx" 
    }
}
```

### src/main.jsx
```
// plugins/core_layout/src/main.jsx
import React from 'react';
import { LayoutProvider } from './context/LayoutContext';
import { createRoot } from 'react-dom/client';
import { App } from './App';

// 全局标志位，防止开发模式下的热重载重复执行
if (typeof window.hevnoCoreLayoutInitialized === 'undefined') {
  window.hevnoCoreLayoutInitialized = false;
}

/**
 * Hevno 插件系统的入口函数。
 * 由前端加载器在加载此插件时调用。
 * @param {import('../../../../frontend/ServiceContainer').ServiceContainer} context - 平台服务容器
 */
export function registerPlugin(context) {
  if (window.hevnoCoreLayoutInitialized) {
    console.warn('[core_layout] Attempted to re-register. Aborting.');
    return;
  }
  
  const hookManager = context.get('hookManager');
  if (!hookManager) {
    console.error('[core_layout] CRITICAL: HookManager service not found!');
    return;
  }

  // 监听内核加载器发出的“就绪”信号
  // 这是我们接管UI的最佳时机
  hookManager.addImplementation('loader.ready', () => {
    // 双重检查
    if (window.hevnoCoreLayoutInitialized) return;
    window.hevnoCoreLayoutInitialized = true;

    console.log('[core_layout] Received "loader.ready". Initializing React application host...');

    // 1. 找到根DOM容器
    const appContainer = document.getElementById('app');
    if (!appContainer) {
      console.error('[core_layout] CRITICAL: #app container not found in DOM!');
      return;
    }

    // 2. 清空容器，为React应用做准备
    appContainer.innerHTML = '';

    // 3. 创建并渲染React应用
    const root = createRoot(appContainer);
    root.render(
    <React.StrictMode>
        {/* 将平台服务注入到 React 世界 */}
        <LayoutProvider services={context}> 
        <App />
        </LayoutProvider>
    </React.StrictMode>
    );

    console.log('[core_layout] React host mounted successfully.');

    // 4. (未来) 在这里可以触发一个新的钩子，比如 'host.ready'
    // hookManager.trigger('host.ready');
  });
}
```

### src/App.jsx
```
// plugins/core_layout/src/main.jsx
import React from 'react';
import { createTheme, ThemeProvider } from '@mui/material/styles';
import CssBaseline from '@mui/material/CssBaseline';
import Box from '@mui/material/Box';
import { FloatingMenuButton } from './components/FloatingMenuButton';
import { PageContainer } from './components/PageContainer'; // 新组件

const darkTheme = createTheme({ palette: { mode: 'dark' } });

export function App() {
  return (
    <ThemeProvider theme={darkTheme}>
      <CssBaseline />
      <Box sx={{ width: '100vw', height: '100vh', display: 'flex' }}>
        <Box component="main" sx={{ flexGrow: 1, position: 'relative' }}>
        <PageContainer />
        <FloatingMenuButton />
        </Box>
      </Box>
    </ThemeProvider>
  );
}
```

### src/context/LayoutContext.jsx
```
// plugins/core_layout/src/context/LayoutContext.jsx
import React, { createContext, useState, useContext, useMemo } from 'react';
import { ContributionRegistry } from '../services/ContributionRegistry';

const LayoutContext = createContext(null);

export function LayoutProvider({ children, services }) {
  // 使用 useMemo 确保 registry 只被实例化一次
  const contributionRegistry = useMemo(() => {
    const manifestProvider = services.get('manifestProvider');
    return new ContributionRegistry(manifestProvider);
  }, [services]);

  const [pages] = useState(() => contributionRegistry.getPageComponents());
  const [activePageId, setActivePageId] = useState(null);
  const [currentSandboxId, setCurrentSandboxId] = useState(null); 

  const value = {
    pages,
    activePageId,
    setActivePageId,
    currentSandboxId,
    setCurrentSandboxId,
    services, // 将平台服务传递下去
  };

  return (
    <LayoutContext.Provider value={value}>
      {children}
    </LayoutContext.Provider>
  );
}

export const useLayout = () => {
  const context = useContext(LayoutContext);
  if (!context) {
    throw new Error('useLayout must be used within a LayoutProvider');
  }
  return context;
};
```

### src/components/FloatingMenuButton.jsx
```
// plugins/core_layout/src/components/FloatingMenuButton.jsx
import React, { useState, useRef, useEffect, useCallback } from 'react';
import Draggable from 'react-draggable';
import { useLayout } from '../context/LayoutContext';
import Box from '@mui/material/Box';
import SpeedDial from '@mui/material/SpeedDial';
import SpeedDialIcon from '@mui/material/SpeedDialIcon';
import SpeedDialAction from '@mui/material/SpeedDialAction';
import HomeRoundedIcon from '@mui/icons-material/HomeRounded';
import * as MuiIcons from '@mui/icons-material';

const DynamicIcon = ({ name }) => {
  const Icon = MuiIcons[name];
  return Icon ? <Icon /> : <div/>;
};

export function FloatingMenuButton() {
  const { pages, activePageId, setActivePageId } = useLayout();
  const draggableRef = useRef(null);

  const [direction, setDirection] = useState('up');
  const [tooltipPlacement, setTooltipPlacement] = useState('left');

  const updateDirections = useCallback(() => {
    if (!draggableRef.current) return;

    const rect = draggableRef.current.getBoundingClientRect();
    const viewHeight = window.innerHeight;
    const viewWidth = window.innerWidth;
    
    setDirection(rect.top > viewHeight / 2 ? 'up' : 'down');
    
    setTooltipPlacement(rect.left > viewWidth / 2 ? 'left' : 'right');
  }, []);

  useEffect(() => {
    updateDirections();
  }, [updateDirections]);


  const actions = [
    { 
      icon: <HomeRoundedIcon />, 
      name: 'Home',
      pageId: null,
    },
    ...pages
      .filter(page => page.menu) 
      .map(page => ({
        icon: <DynamicIcon name={page.menu.icon} />,
        name: page.menu.title,
        pageId: page.id
      }))
  ];

  const handleActionClick = (pageId) => {
    setActivePageId(pageId);
  };
  
  const handleDragStop = () => {
    updateDirections();
  };

  return (
    <Draggable 
      nodeRef={draggableRef}
      handle=".MuiFab-primary"
      bounds="parent"
      onStop={handleDragStop}
    >
      <Box 
        ref={draggableRef} 
        sx={{ 
          position: 'absolute', 
          bottom: 24, 
          left: 24, 
          zIndex: 1300 
        }}
      >
        <SpeedDial
          ariaLabel="Main navigation"
          icon={<SpeedDialIcon />}
          direction={direction}
        >
          {actions.map((action) => (
            <SpeedDialAction
              key={action.name}
              icon={action.icon}
              tooltipTitle={action.name}
              tooltipPlacement={tooltipPlacement}
              onClick={() => handleActionClick(action.pageId)}
              sx={
                activePageId === action.pageId 
                ? {
                    '& .MuiFab-root': {
                      bgcolor: 'primary.main',
                      color: 'common.white',
                      '&:hover': {
                        bgcolor: 'primary.dark',
                      },
                    },
                  }
                : {} 
              }
            />
          ))}
        </SpeedDial>
      </Box>
    </Draggable>
  );
}
```

### src/components/PageContainer.jsx
```
// plugins/core_layout/src/components/PageContainer.jsx
import React, { useState, useEffect, useMemo } from 'react';
import { useLayout } from '../context/LayoutContext';
import { Box, Typography, CircularProgress } from '@mui/material';

// 将组件的创建和缓存移到组件外部或使用 useMemo，以避免在每次渲染时重新创建 LazyComponent
const componentCache = new Map();

function getLazyComponent(pageInfo) {
  const { id, manifest, componentExportName } = pageInfo;

  if (componentCache.has(id)) {
    return componentCache.get(id);
  }

  const LazyComponent = React.lazy(async () => {
    // 动态 import 路径
    const modulePath = `/plugins/${manifest.id}/${manifest.frontend.srcEntryPoint || manifest.frontend.entryPoint}`;
    
    try {
      const module = await import(/* @vite-ignore */ modulePath);
      if (module[componentExportName]) {
        // React.lazy 期望一个包含 default 导出的模块
        return { default: module[componentExportName] };
      } else {
        // 如果找不到具名导出，尝试默认导出
        if (module.default) {
           return { default: module.default }
        }
        throw new Error(`Component export '${componentExportName}' not found in plugin '${manifest.id}'.`);
      }
    } catch (error) {
      console.error(`Failed to load component for plugin '${manifest.id}':`, error);
      // 返回一个显示错误的组件
      const ErrorComponent = () => (
        <Box sx={{ p: 2, color: 'error.main' }}>
          <Typography>Error loading page: {manifest.id}</Typography>
          <Typography variant="body2">{error.message}</Typography>
        </Box>
      );
      return { default: ErrorComponent };
    }
  });

  componentCache.set(id, LazyComponent);
  return LazyComponent;
}


export function PageContainer() {
  const { pages, activePageId, services } = useLayout();
  
  // 使用 useMemo 来根据 activePageId 查找页面信息并获取懒加载组件
  // 只有当 activePageId 变化时，才会重新计算
  const ActiveLazyComponent = useMemo(() => {
    if (!activePageId) return null;
    const pageInfo = pages.find(p => p.id === activePageId);
    return pageInfo ? getLazyComponent(pageInfo) : null;
  }, [activePageId, pages]);


  if (!ActiveLazyComponent) {
    return (
      <Box sx={{ p: 4, textAlign: 'center' }}>
        <Typography variant="h5">Hevno</Typography>
        <Typography color="text.secondary">戳一下那个按钮试试</Typography>
      </Box>
    );
  }

  return (
    // Suspense 包裹懒加载组件，提供 fallback UI
    <React.Suspense fallback={<Box sx={{ display: 'flex', justifyContent: 'center', p: 4 }}><CircularProgress /></Box>}>
      {/* 在这里将 props 传递给将要被渲染的组件 */}
      <ActiveLazyComponent services={services} />
    </React.Suspense>
  );
}
```

### src/services/ContributionRegistry.js
```
// plugins/core_layout/src/services/ContributionRegistry.js
export class ContributionRegistry {
    constructor(manifestProvider) {
        this.pageComponents = [];
        this.manifests = manifestProvider.getManifests();
        this.processContributions();
    }

    processContributions() {
        // 我们只关心 'page-component' 类型的插件
        const pagePlugins = this.manifests.filter(
            m => m.frontend?.type === 'page-component' && m.frontend?.contributions?.pageComponents
        );

        for (const manifest of pagePlugins) {
            for (const pageDef of manifest.frontend.contributions.pageComponents) {
                if (pageDef.id && pageDef.componentExportName) { // 修改: menu 变为可选
                    this.pageComponents.push({
                        ...pageDef,
                        pluginId: manifest.id,
                        manifest: manifest, // 保存整个 manifest 以便后续查找入口文件
                    });
                } else {
                    console.warn(`[core_layout] Invalid pageComponent contribution from plugin '${manifest.id}'.`, pageDef);
                }
            }
        }
        console.log(`[core_layout] Discovered ${this.pageComponents.length} page components.`, this.pageComponents);
    }

    getPageComponents() {
        return this.pageComponents;
    }
}
```

# Directory: plugins/core_llm_config

### vite.config.js
```
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
  plugins: [react()],
  build: {
    lib: {
      entry: resolve(__dirname, 'src/LLMConfigPage.jsx'),
      name: 'HevnoLLMConfig',
      fileName: 'main',
      formats: ['es'],
    },
    rollupOptions: {
      external: ['react', 'react-dom', '@mui/material', '@mui/icons-material'],
      output: {
        globals: {
          react: 'React',
          'react-dom': 'ReactDOM'
        }
      }
    },
    outDir: 'dist',
    emptyOutDir: true,
  },
});
```

### package.json
```
{
  "name": "hevno-plugin-llm-config",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "vite build"
  },
  "peerDependencies": {
    "react": ">=18.0.0",
    "@mui/material": ">=5.0.0",
    "@mui/icons-material": ">=5.0.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "latest",
    "vite": "latest"
  }
}
```

### manifest.json
```
{
    "id": "core_llm_config",
    "name": "LLM Configuration",
    "version": "1.0.0",
    "description": "Provides a UI to configure and monitor LLM provider API keys.",
    "author": "Hevno Team",
    "frontend": {
        "type": "page-component",
        "priority": 120,
        "entryPoint": "dist/main.js",
        "srcEntryPoint": "src/LLMConfigPage.jsx",
        "contributions": {
            "pageComponents": [
                {
                    "id": "llm_config.main_view",
                    "componentExportName": "LLMConfigPage",
                    "menu": {
                        "path": "/config/llm",
                        "title": "LLM 配置",
                        "icon": "KeyRounded"
                    }
                }
            ]
        }
    }
}
```

### src/LLMConfigPage.jsx
```
// plugins/core_llm_config/src/LLMConfigPage.jsx
import React, { useState, useEffect, useCallback } from 'react';
import {
    Box, Typography, Paper, CircularProgress, Button, Alert, TextField,
    Select, MenuItem, FormControl, InputLabel
} from '@mui/material';
import AddIcon from '@mui/icons-material/Add';
import RefreshIcon from '@mui/icons-material/Refresh';
import { KeyStatusTable } from './components/KeyStatusTable';
import { fetchKeyConfig, addKey, deleteKey } from './utils/api';

export function LLMConfigPage() {
    const [config, setConfig] = useState(null);
    const [loading, setLoading] = useState(true);
    const [actionInProgress, setActionInProgress] = useState(false); // 通用加载状态
    const [error, setError] = useState('');
    const [provider, setProvider] = useState('gemini');
    const [newKey, setNewKey] = useState('');

    const loadData = useCallback(async () => {
        setLoading(true);
        setError('');
        try {
            const data = await fetchKeyConfig(provider);
            setConfig(data);
        } catch (e) {
            setError(e.message);
            setConfig({ provider, keys: [] }); // 即使出错也显示空表
            console.error(e);
        } finally {
            setLoading(false);
        }
    }, [provider]);

    useEffect(() => {
        loadData();
    }, [loadData]);

    const handleAdd = async () => {
        if (!newKey.trim()) return;
        setActionInProgress(true);
        setError('');
        try {
            await addKey(provider, newKey.trim());
            setNewKey(''); // 清空输入框
            await loadData(); // 重新加载数据
        } catch (e) {
            setError(`添加失败: ${e.message}`);
        } finally {
            setActionInProgress(false);
        }
    };
    
    const handleDelete = async (keySuffix) => {
        if (!window.confirm(`确定要从 .env 文件中永久删除密钥 "${keySuffix}" 吗？`)) return;
        setActionInProgress(true);
        setError('');
        try {
            await deleteKey(provider, keySuffix);
            await loadData(); // 重新加载数据
        } catch (e) {
            setError(`删除失败: ${e.message}`);
        } finally {
            setActionInProgress(false);
        }
    };

    return (
        <Box sx={{ p: 3, height: '100%', overflowY: 'auto' }}>
            <Typography variant="h4" gutterBottom>LLM 提供商配置</Typography>

            <Alert severity="warning" sx={{ mb: 3 }}>
                <b>警告:</b> 此页面上的操作将直接修改您服务器上的 <code>.env</code> 文件。请谨慎操作。
            </Alert>

            {error && <Alert severity="error" sx={{ mb: 2 }} onClose={() => setError('')}>{error}</Alert>}

            <Paper sx={{ p: 3, display: 'flex', flexDirection: 'column', gap: 3 }}>
                <FormControl sx={{ minWidth: 200 }} size="small">
                    <InputLabel>提供商</InputLabel>
                    <Select value={provider} label="提供商" onChange={(e) => setProvider(e.target.value)}>
                        <MenuItem value="gemini">Gemini</MenuItem>
                    </Select>
                </FormControl>

                <Box>
                    <Typography variant="h6" gutterBottom>当前密钥状态</Typography>
                     {loading ? (
                        <Box sx={{ display: 'flex', justifyContent: 'center', my: 4 }}><CircularProgress /></Box>
                    ) : (
                        <KeyStatusTable keys={config?.keys || []} onDelete={handleDelete} isDeleting={actionInProgress} />
                    )}
                </Box>

                <Box>
                    <Typography variant="h6" gutterBottom>添加新密钥</Typography>
                    <Box sx={{ display: 'flex', gap: 2, alignItems: 'center' }}>
                         <TextField
                            fullWidth
                            label="新 API 密钥"
                            value={newKey}
                            onChange={(e) => setNewKey(e.target.value)}
                            placeholder="在此处粘贴完整的 API 密钥"
                            variant="outlined"
                            size="small"
                            onKeyPress={(e) => e.key === 'Enter' && handleAdd()}
                        />
                         <Button
                            variant="contained"
                            // startIcon={<AddIcon />}
                            onClick={handleAdd}
                            disabled={actionInProgress || loading || !newKey.trim()}
                        >
                            {actionInProgress && !loading ? <CircularProgress size={36} color="inherit" /> : '添加'}
                        </Button>
                    </Box>
                </Box>
                 <Button
                    variant="outlined"
                    startIcon={<RefreshIcon />}
                    onClick={loadData}
                    disabled={loading || actionInProgress}
                    sx={{ alignSelf: 'flex-start' }}
                >
                    刷新状态
                </Button>
            </Paper>
        </Box>
    );
}

export default LLMConfigPage;
export const registerPlugin = () => {};
```

### src/utils/api.js
```
// plugins/core_llm_config/src/utils/api.js

const BASE_URL = '/api/llm/config';

export async function fetchKeyConfig(providerName) {
    const response = await fetch(`${BASE_URL}/${providerName}`);
    if (!response.ok) {
        // 提供一个友好的默认值，以防提供商没有配置任何密钥
        if (response.status === 404) {
            return { provider: providerName, keys: [] };
        }
        const err = await response.json().catch(() => ({ detail: "API query failed." }));
        throw new Error(err.detail || `HTTP Error ${response.status}`);
    }
    return response.json();
}

// [新] 添加密钥的函数
export async function addKey(providerName, key) {
    const response = await fetch(`${BASE_URL}/${providerName}/keys`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ key }),
    });
    if (!response.ok) {
        const err = await response.json().catch(() => ({ detail: "Failed to add key." }));
        throw new Error(err.detail || `HTTP Error ${response.status}`);
    }
    return response.json();
}

// [新] 删除密钥的函数
export async function deleteKey(providerName, keySuffix) {
    // 从 "..." 中提取最后4位
    const suffix = keySuffix.slice(-4);
    const response = await fetch(`${BASE_URL}/${providerName}/keys/${suffix}`, {
        method: 'DELETE',
    });
    if (!response.ok) {
        const err = await response.json().catch(() => ({ detail: "Failed to delete key." }));
        throw new Error(err.detail || `HTTP Error ${response.status}`);
    }
    return response.json();
}
```

### src/components/KeyStatusTable.jsx
```
// plugins/core_llm_config/src/components/KeyStatusTable.jsx
import React from 'react';
import {
    Table, TableBody, TableCell, TableContainer, TableHead, TableRow, Paper, Chip,
    Typography, IconButton, Tooltip
} from '@mui/material';
import DeleteIcon from '@mui/icons-material/Delete';
import { Countdown } from './Countdown'; // 我们将倒计时逻辑移到自己的组件中

export function KeyStatusTable({ keys, onDelete, isDeleting }) {
    const getStatusChip = (key) => {
        switch (key.status) {
            case 'available':
                return <Chip label="可用" color="success" size="small" />;
            case 'rate_limited':
                return <Chip label="限速中" color="warning" size="small" />;
            case 'banned':
                return <Chip label="已禁用" color="error" size="small" />;
            default:
                return <Chip label={key.status} size="small" />;
        }
    };

    return (
        <TableContainer component={Paper} variant="outlined">
            <Table size="small">
                <TableHead>
                    <TableRow>
                        <TableCell>密钥 (后4位)</TableCell>
                        <TableCell>状态</TableCell>
                        <TableCell>可用时间</TableCell>
                        <TableCell align="right">操作</TableCell>
                    </TableRow>
                </TableHead>
                <TableBody>
                    {keys.length === 0 && (
                        <TableRow>
                            <TableCell colSpan={4} align="center">
                                <Typography color="text.secondary" sx={{ p: 2 }}>
                                    未找到为该提供商配置的密钥。
                                </Typography>
                            </TableCell>
                        </TableRow>
                    )}
                    {keys.map((key) => (
                        <TableRow key={key.key_suffix}>
                            <TableCell component="th" scope="row">
                                <Typography variant="body2" sx={{ fontFamily: 'monospace' }}>
                                    {key.key_suffix}
                                </Typography>
                            </TableCell>
                            <TableCell>{getStatusChip(key)}</TableCell>
                            <TableCell>
                                {key.status === 'rate_limited' && key.rate_limit_until ?
                                    <Countdown until={key.rate_limit_until} />
                                    : '—'}
                            </TableCell>
                            <TableCell align="right">
                                <Tooltip title="从 .env 文件中删除此密钥">
                                    <span>
                                        <IconButton
                                            size="small"
                                            color="error"
                                            onClick={() => onDelete(key.key_suffix)}
                                            disabled={isDeleting}
                                        >
                                            <DeleteIcon fontSize="small" />
                                        </IconButton>
                                    </span>
                                </Tooltip>
                            </TableCell>
                        </TableRow>
                    ))}
                </TableBody>
            </Table>
        </TableContainer>
    );
}
```

### src/components/Countdown.jsx
```
// plugins/core_llm_config/src/components/Countdown.jsx
import React, { useState, useEffect } from 'react';

export function Countdown({ until }) {
    const [timeLeft, setTimeLeft] = useState(Math.round(until - Date.now() / 1000));

    useEffect(() => {
        if (timeLeft <= 0) return;
        const timer = setInterval(() => {
            const newTimeLeft = Math.round(until - Date.now() / 1000);
            setTimeLeft(newTimeLeft > 0 ? newTimeLeft : 0);
        }, 1000);
        return () => clearInterval(timer);
    }, [timeLeft, until]);

    return timeLeft > 0 ? `~${timeLeft}s` : '可用';
}
```

# Directory: plugins/sandbox_editor

### vite.config.js
```
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
  plugins: [react()],
  build: {
    lib: {
      entry: resolve(__dirname, 'src/SandboxEditorPage.jsx'),
      name: 'HevnoSandboxEditor',
      fileName: 'main',
      formats: ['es'],
    },
    rollupOptions: {
      external: ['react', 'react-dom', '@mui/material', '@mui/icons-material'],
      output: {
        globals: {
          react: 'React',
          'react-dom': 'ReactDOM'
        }
      }
    },
    outDir: 'dist',
    emptyOutDir: true,
  },
});
```

### package.json
```
{
"name": "hevno-plugin-sandbox-editor",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "vite build"
  },
  "dependencies": {
    "@dnd-kit/core": "^6.1.0",
    "@dnd-kit/sortable": "^8.0.0",
    "@dnd-kit/utilities": "^3.2.2"
  },
  "peerDependencies": {
    "react": ">=18.0.0",
    "@mui/material": ">=5.0.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "latest",
    "vite": "latest"
  }
}
```

### manifest.json
```
{
    "id": "sandbox_editor",
    "name": "Sandbox Editor",
    "version": "1.0.0",
    "description": "Provides a UI to edit sandboxes, triggered from the explorer.",
    "author": "Hevno Team",
    "frontend": {
        "type": "page-component",
        "priority": 150,
        "entryPoint": "dist/main.js",
        "srcEntryPoint": "src/SandboxEditorPage.jsx",
        "contributions": {
            "pageComponents": [
                {
                    "id": "sandbox_editor.main_view",
                    "componentExportName": "SandboxEditorPage"
                }
            ]
        }
    }
}
```

### src/SandboxEditorPage.jsx
```
import React, { useState, useEffect, useCallback } from 'react';
import { Box, Typography, Tabs, Tab, CircularProgress, Button, Alert } from '@mui/material';
import ArrowBackIcon from '@mui/icons-material/ArrowBack';
import PublishedWithChangesIcon from '@mui/icons-material/PublishedWithChanges';
import { useLayout } from '../../core_layout/src/context/LayoutContext';
import { SCOPE_TABS } from './utils/constants';
import { DataTree } from './components/DataTree';
import { CodexEditor } from './editors/CodexEditor';
import { GraphEditor } from './editors/GraphEditor';
import { MemoriaEditor } from './editors/MemoriaEditor';
import { query, mutate, applyDefinition } from './utils/api';
import { GenericEditorDialog } from './editors/GenericEditorDialog';
import { AddItemDialog } from './editors/AddItemDialog';
import { isObject } from './utils/constants';

export function SandboxEditorPage({ services }) {
    const { currentSandboxId, setActivePageId, setCurrentSandboxId } = useLayout();
    const [sandboxData, setSandboxData] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState('');
    const [activeScope, setActiveScope] = useState(0);
    const [editingCodex, setEditingCodex] = useState(null);
    const [editingGraph, setEditingGraph] = useState(null);
    const [editingMemoria, setEditingMemoria] = useState(null);
    const [editingGeneric, setEditingGeneric] = useState(null);
    const [addItemTarget, setAddItemTarget] = useState(null);

    const loadSandboxData = useCallback(async () => {
        
        if (!currentSandboxId) return;
        setLoading(true);
        setError('');
        try {
            const results = await query(currentSandboxId, ['definition', 'lore', 'moment']);
            setSandboxData(results);
        } catch (e) {
            setError(e.message);
            console.error(e);
        } finally {
            setLoading(false);
        }
    }, [currentSandboxId]);

    useEffect(() => {
        if (currentSandboxId) {
            loadSandboxData();
        }
    }, [currentSandboxId, loadSandboxData]);

    const handleScopeChange = (event, newValue) => {
        
        setActiveScope(newValue);
        setEditingCodex(null);
        setEditingGraph(null);
    };

    const handleGoBackToExplorer = () => {
        
        setCurrentSandboxId(null);
        setActivePageId('sandbox_explorer.main_view');
    };
    
    const handleEdit = (path, value) => {
        
        const editorType = isObject(value) ? value.__hevno_type__ : undefined;
        if (editorType) {
            const pathParts = path.split('/');
            const name = pathParts[pathParts.length - 1];
            if (editorType === 'hevno/codex') setEditingCodex({ name, data: value, basePath: path });
            else if (editorType === 'hevno/graph') setEditingGraph({ name, data: value, basePath: path });
            else if (editorType === 'hevno/memoria') setEditingMemoria({ data: value, path: path });
            else setEditingGeneric({ path, value });
        } else {
            setEditingGeneric({ path, value });
        }
    };
    
    const handleBackToOverview = () => {
        
        setEditingCodex(null);
        setEditingGraph(null);
        setEditingMemoria(null);
        loadSandboxData();
    };

    const handleGenericSave = async (path, newValue) => {
        
        try {
            await mutate(currentSandboxId, [{ type: 'UPSERT', path, value: newValue }]);
            setEditingGeneric(null);
            await loadSandboxData();
        } catch (err) {
            console.error(`Failed to save value for path "${path}":`, err);
            throw err;
        }
    };

    const handleOpenAddDialog = (path, existingKeys) => {
        setAddItemTarget({ path, existingKeys });
    };

    const handleAddItem = async (parentPath, key, value) => {
        const fullPath = `${parentPath}/${key}`;
        try {
            await mutate(currentSandboxId, [{ type: 'UPSERT', path: fullPath, value }]);
            setAddItemTarget(null); // 关闭对话框
            await loadSandboxData(); // 重新加载数据
        } catch (err) {
            console.error(`Failed to add item at path "${fullPath}":`, err);
            throw err; // 将错误传递回对话框以显示
        }
    };

    const handleApplyDefinition = async () => {
        if (!window.confirm(
            "确定要应用这个蓝图吗？\n\n这将完全覆盖当前的 `lore` 和 `moment` 状态，并用 `definition` 中的初始值替换它们。\n\n当前的所有记忆和演化知识都将丢失，并开启一个全新的历史记录。此操作不可撤销。"
        )) {
            return;
        }

        setLoading(true);
        setError('');
        try {
            await applyDefinition(currentSandboxId);
            await loadSandboxData();
            alert("蓝图已成功应用！");
        } catch (e) {
            setError(`应用蓝图失败: ${e.message}`);
            console.error(e);
        } finally {
            setLoading(false);
        }
    };


    // ... (加载和错误状态的渲染保持不变) ...
    if (!currentSandboxId) return <Box sx={{ p: 4, textAlign: 'center' }}><Typography variant="h6" color="error">未选择要编辑的沙盒</Typography></Box>;
    if (loading) return <Box sx={{ display: 'flex', justifyContent: 'center', alignItems: 'center', height: '100%' }}><CircularProgress /></Box>;
    if (error) return <Box sx={{ p: 4, textAlign: 'center' }}><Alert severity="error" onClose={() => setError('')}>{error}</Alert><Button variant="outlined" sx={{ mt: 2 }} onClick={loadSandboxData}>重试</Button></Box>;
    const currentScopeData = sandboxData[SCOPE_TABS[activeScope]];

    if (editingCodex) return <CodexEditor sandboxId={currentSandboxId} basePath={editingCodex.basePath} codexName={editingCodex.name} codexData={editingCodex.data} onBack={handleBackToOverview} />;
    if (editingGraph) return <GraphEditor sandboxId={currentSandboxId} basePath={editingGraph.basePath} graphName={editingGraph.name} graphData={editingGraph.data} onBack={handleBackToOverview} />;
    if (editingMemoria) return <MemoriaEditor sandboxId={currentSandboxId} basePath={editingMemoria.path} memoriaData={editingMemoria.data} onBack={handleBackToOverview} />;

    const sandboxName = (sandboxData.definition?.name || sandboxData.lore?.name || 'Sandbox');

    return (
        <Box sx={{ p: 3, height: '100%', display: 'flex', flexDirection: 'column' }}>
            <Box sx={{ display: 'flex', alignItems: 'center', mb: 2, flexShrink: 0 }}>
                <Button variant="outlined" startIcon={<ArrowBackIcon />} onClick={handleGoBackToExplorer} sx={{ mr: 2 }}>
                    返回沙盒列表
                </Button>
                <Typography variant="h4" component="h1" noWrap sx={{ flexGrow: 1 }}>
                    正在编辑: {sandboxName}
                </Typography>
                {SCOPE_TABS[activeScope] === 'definition' && (
                    <Button 
                        variant="contained" 
                        color="secondary"
                        startIcon={<PublishedWithChangesIcon />} 
                        onClick={handleApplyDefinition}
                        disabled={loading}
                    >
                        应用蓝图
                    </Button>
                )}
            </Box>

            <Tabs value={activeScope} onChange={handleScopeChange} aria-label="sandbox scopes" sx={{ flexShrink: 0, borderBottom: 1, borderColor: 'divider' }}>
                {SCOPE_TABS.map((scope, index) => (
                    <Tab label={scope.charAt(0).toUpperCase() + scope.slice(1)} key={index} />
                ))}
            </Tabs>
            <Box sx={{ mt: 2, flexGrow: 1, overflowY: 'auto' }}>
                {currentScopeData ? (
                    <DataTree data={currentScopeData} path={SCOPE_TABS[activeScope]} onEdit={handleEdit} onAdd={handleOpenAddDialog} />
                ) : (
                    <Typography color="text.secondary">该范围内没有可用数据</Typography>
                )}
            </Box>

            <GenericEditorDialog open={!!editingGeneric} onClose={() => setEditingGeneric(null)} onSave={handleGenericSave} item={editingGeneric} />
            
            <AddItemDialog
                open={!!addItemTarget}
                onClose={() => setAddItemTarget(null)}
                onAdd={handleAddItem}
                parentPath={addItemTarget?.path}
                existingKeys={addItemTarget?.existingKeys}
            />
        </Box>
    );
}

export default SandboxEditorPage;
```

### src/editors/RuntimeEditor.jsx
```
// plugins/sandbox_editor/src/editors/RuntimeEditor.jsx
import React, { useState, useEffect } from 'react';
import { Box, List, Collapse, IconButton, Button, Select, MenuItem, Typography, Paper, ListSubheader, TextField } from '@mui/material';
import ExpandMoreIcon from '@mui/icons-material/ExpandMore';
import ChevronRightIcon from '@mui/icons-material/ChevronRight';
import DeleteIcon from '@mui/icons-material/Delete';
import AddIcon from '@mui/icons-material/Add';
import SaveIcon from '@mui/icons-material/Save';
import {
  DndContext,
  closestCenter,
  KeyboardSensor,
  PointerSensor,
  useSensor,
  useSensors,
} from '@dnd-kit/core';
import {
  arrayMove,
  SortableContext,
  sortableKeyboardCoordinates,
  verticalListSortingStrategy,
} from '@dnd-kit/sortable';
import { SortableRuntimeItem } from '../components/SortableRuntimeItem';
import { RuntimeConfigForm } from './RuntimeConfigForm';

const NEW_RUNTIME_SYMBOL = Symbol('new_runtime');

export function RuntimeEditor({ runList, onRunListChange }) {
  const [runs, setRuns] = useState(runList || []);
  const [editingRun, setEditingRun] = useState(null);
  const [draftData, setDraftData] = useState(null);

  useEffect(() => {
    setRuns(runList || []);
  }, [runList]);

  const sensors = useSensors(
    useSensor(PointerSensor),
    useSensor(KeyboardSensor, { coordinateGetter: sortableKeyboardCoordinates })
  );

  const handleToggleExpand = (index) => {
    if (editingRun === index) {
      setEditingRun(null);
      setDraftData(null);
    } else {
      setEditingRun(index);
      setDraftData({ ...runs[index] });
    }
  };

  const handleDragEnd = (event) => {
    const { active, over } = event;
    if (active.id !== over.id) {
      const oldI = parseInt(active.id, 10);
      const newI = parseInt(over.id, 10);

      const newOrderedRuns = arrayMove(runs, oldI, newI);
      setRuns(newOrderedRuns);
      onRunListChange(newOrderedRuns);
    }
  };

  const handleAddClick = () => {
    if (editingRun !== null) {
      alert("Please save or discard the current runtime first.");
      return;
    }
    setEditingRun(NEW_RUNTIME_SYMBOL);
    setDraftData({ runtime: '', config: {} });
  };
  
  const handleSave = () => {
    if (!draftData.runtime) {
      alert("Runtime type is required.");
      return;
    }
    let newRuns;
    if (editingRun === NEW_RUNTIME_SYMBOL) {
      newRuns = [...runs, draftData];
    } else {
      newRuns = runs.map((run, i) => (i === editingRun ? draftData : run));
    }
    setRuns(newRuns);
    onRunListChange(newRuns);
    
    setEditingRun(null);
    setDraftData(null);
  };

  const handleDelete = (index) => {
    const newRuns = runs.filter((_, i) => i !== index);
    setRuns(newRuns);
    onRunListChange(newRuns);
  };
  
  const handleDiscard = () => {
      setEditingRun(null);
      setDraftData(null);
  }

  const handleDraftChange = (field, value) => {
      setDraftData(prev => ({...prev, [field]: value}));
  }

  const renderRunForm = () => {
    if (!draftData) return null;
    const isNew = editingRun === NEW_RUNTIME_SYMBOL;
    return (
      <Paper sx={{ p: 2, m: 1, border: `1px solid`, borderColor: 'primary.main' }}>
         <Typography variant="h6" sx={{mb: 2}}>{isNew ? "添加指令" : "编辑指令"}</Typography>
        <Select
          value={draftData.runtime}
          onChange={(e) => handleDraftChange('runtime', e.target.value)}
          fullWidth
          size="small"
          displayEmpty
          sx={{ mb: 2 }}
        >
          <MenuItem value="" disabled><em>选择指令类型</em></MenuItem>

          <ListSubheader>LLM</ListSubheader>
          <MenuItem value="llm.default">llm.default</MenuItem>

          <ListSubheader>Memoria</ListSubheader>
          <MenuItem value="memoria.add">memoria.add</MenuItem>
          <MenuItem value="memoria.query">memoria.query</MenuItem>

          <ListSubheader>Codex</ListSubheader>
          <MenuItem value="codex.invoke">codex.invoke</MenuItem>

          <ListSubheader>System IO</ListSubheader>
          <MenuItem value="system.io.input">system.io.input</MenuItem>
          <MenuItem value="system.io.log">system.io.log</MenuItem>

          <ListSubheader>System Data</ListSubheader>
          <MenuItem value="system.data.format">system.data.format</MenuItem>
          <MenuItem value="system.data.parse">system.data.parse</MenuItem>
          <MenuItem value="system.data.regex">system.data.regex</MenuItem>
          
          <ListSubheader>System Flow</ListSubheader>
          <MenuItem value="system.flow.call">system.flow.call</MenuItem>
          <MenuItem value="system.flow.map">system.flow.map</MenuItem>

          <ListSubheader>System Advanced</ListSubheader>
          <MenuItem value="system.execute">system.execute</MenuItem>
        </Select>

        {draftData.runtime && (
          <TextField
            label="as (可选命名空间)"
            value={draftData.config?.as || ''}
            onChange={(e) => {
              const newConfig = { ...draftData.config, as: e.target.value };
              // 如果值为空，则从 config 对象中删除 'as' 键以保持数据清洁
              if (!e.target.value) {
                delete newConfig.as;
              }
              handleDraftChange('config', newConfig);
            }}
            fullWidth
            size="small"
            sx={{ mb: 2 }}
            helperText="为该指令的输出指定一个名称，以便后续指令引用。"
          />
        )}
        
        {draftData.runtime && (
          <RuntimeConfigForm
            runtimeType={draftData.runtime}
            config={draftData.config || {}}
            onConfigChange={(newConfig) => handleDraftChange('config', newConfig)}
          />
        )}
        <Box sx={{mt: 2, display: 'flex', gap: 1}}>
            <Button variant="contained" startIcon={<SaveIcon />} onClick={handleSave}>
                {isNew ? "添加" : "保存"}
            </Button>
            <Button variant="outlined" onClick={handleDiscard}>
                取消
            </Button>
        </Box>
      </Paper>
    );
  };

  return (
    <Paper variant="outlined" sx={{ mt: 2, p:1 }}>
      <Box sx={{display: 'flex', justifyContent: 'space-between', alignItems: 'center'}}>
        <Typography variant="subtitle1" gutterBottom component="div">指令列表</Typography>
        <Button variant="outlined" startIcon={<AddIcon />} onClick={handleAddClick} size="small" sx={{ mb: 1 }} disabled={editingRun !== null}>
            添加指令
        </Button>
      </Box>

      {editingRun !== null ? renderRunForm() : (
          <DndContext sensors={sensors} collisionDetection={closestCenter} onDragEnd={handleDragEnd}>
            <SortableContext items={runs.map((_, i) => i.toString())} strategy={verticalListSortingStrategy}>
              <List disablePadding>
                {runs.map((run, index) => (
                  <SortableRuntimeItem
                    key={index}
                    id={index.toString()}
                    run={run}
                    onEdit={() => handleToggleExpand(index)}
                    onDelete={() => handleDelete(index)}
                  />
                ))}
                {runs.length === 0 && <Typography color="text.secondary" sx={{p:2, textAlign: 'center'}}>No runtimes defined.</Typography>}
              </List>
            </SortableContext>
          </DndContext>
      )}
    </Paper>
  );
}
```

### src/editors/MemoriaEditor.jsx
```
// plugins/sandbox_editor/src/editors/MemoriaEditor.jsx
import React, { useState, useEffect } from 'react';
import { Box, Typography, List, Collapse, IconButton, Button, TextField, Alert, Chip, InputAdornment,ListItem,ListItemIcon,ListItemText } from '@mui/material';
import ExpandMoreIcon from '@mui/icons-material/ExpandMore';
import DeleteIcon from '@mui/icons-material/Delete';
import AddIcon from '@mui/icons-material/Add';
import SaveIcon from '@mui/icons-material/Save';
import ChevronRightIcon from '@mui/icons-material/ChevronRight';
import { DndContext, closestCenter, KeyboardSensor, PointerSensor, useSensor, useSensors } from '@dnd-kit/core';
import { arrayMove, SortableContext, verticalListSortingStrategy } from '@dnd-kit/sortable';

import { SortableMemoryEntryItem } from '../components/SortableMemoryEntryItem';
import { mutate } from '../utils/api';

export function MemoriaEditor({ sandboxId, basePath, memoriaData, onBack }) {
  const [streams, setStreams] = useState({});
  const [globalSequence, setGlobalSequence] = useState(0);
  const [expandedStreams, setExpandedStreams] = useState({});
  const [expandedEntries, setExpandedEntries] = useState({});
  const [newStreamName, setNewStreamName] = useState('');
  const [errorMessage, setErrorMessage] = useState('');
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!memoriaData) return;
    
    const initialStreams = {};
    Object.entries(memoriaData).forEach(([key, value]) => {
      if (key === '__hevno_type__' || key === '__global_sequence__') {
        if (key === '__global_sequence__') setGlobalSequence(value);
        return;
      }
      const entriesWithInternalIds = (value.entries || []).map((entry, index) => ({
        ...entry,
        _internal_id: `${key}_${Date.now()}_${index}`,
      }));
      initialStreams[key] = { ...value, entries: entriesWithInternalIds };
    });
    setStreams(initialStreams);
  }, [memoriaData]);

  const sensors = useSensors(
    useSensor(PointerSensor),
    useSensor(KeyboardSensor)
  );
  
  const handleSaveAll = async () => {
    setLoading(true);
    setErrorMessage('');
    const payload = {
      __hevno_type__: 'hevno/memoria',
      __global_sequence__: globalSequence,
    };
    for (const streamName in streams) {
      const stream = streams[streamName];
      payload[streamName] = {
        ...stream,
        entries: stream.entries.map(({ _internal_id, ...rest }) => rest),
      };
    }
    try {
      await mutate(sandboxId, [{ type: 'UPSERT', path: basePath, value: payload }]);
      alert('Memoria saved successfully!');
      onBack();
    } catch (e) {
      setErrorMessage(`Failed to save Memoria: ${e.message}`);
    } finally {
      setLoading(false);
    }
  };

  const handleAddStream = () => {
    const name = newStreamName.trim();
    if (!name || streams[name]) {
      setErrorMessage(name ? `Stream "${name}" already exists.` : "Stream name is required.");
      return;
    }
    setStreams(prev => ({ ...prev, [name]: { config: {}, entries: [] } }));
    setNewStreamName('');
    setExpandedStreams(prev => ({ ...prev, [name]: true })); // Automatically expand new stream
  };

  const handleDeleteStream = (streamName) => {
    if (!window.confirm(`Are you sure you want to delete the stream "${streamName}"? This cannot be undone until you save.`)) return;
    setStreams(prev => {
      const newStreams = { ...prev };
      delete newStreams[streamName];
      return newStreams;
    });
  };

  const handleAddEntry = (streamName) => {
    setStreams(prev => {
      const newEntry = {
        content: '', level: 'event', tags: [],
        _internal_id: `${streamName}_${Date.now()}`
      };
      const updatedEntries = [...prev[streamName].entries, newEntry];
      setExpandedEntries(exp => ({ ...exp, [newEntry._internal_id]: true }));
      return { ...prev, [streamName]: { ...prev[streamName], entries: updatedEntries } };
    });
  };

  const handleDeleteEntry = (streamName, entryInternalId) => {
    setStreams(prev => {
      const updatedEntries = prev[streamName].entries.filter(e => e._internal_id !== entryInternalId);
      return { ...prev, [streamName]: { ...prev[streamName], entries: updatedEntries } };
    });
  };
  
  const handleEntryChange = (streamName, entryInternalId, field, value) => {
    setStreams(prev => {
      const updatedEntries = prev[streamName].entries.map(entry => {
        if (entry._internal_id === entryInternalId) {
          let finalValue = value;
          if (field === 'tags' && typeof value === 'string') {
            finalValue = value.split(',').map(t => t.trim().toLowerCase()).filter(Boolean);
          }
          return { ...entry, [field]: finalValue };
        }
        return entry;
      });
      return { ...prev, [streamName]: { ...prev[streamName], entries: updatedEntries } };
    });
  };

  const handleEntryDragEnd = (streamName, event) => {
    const { active, over } = event;
    if (active && over && active.id !== over.id) {
      setStreams(prev => {
        const stream = prev[streamName];
        const oldIndex = stream.entries.findIndex(e => e._internal_id === active.id);
        const newIndex = stream.entries.findIndex(e => e._internal_id === over.id);
        if (oldIndex === -1 || newIndex === -1) return prev;
        const reorderedEntries = arrayMove(stream.entries, oldIndex, newIndex);
        return { ...prev, [streamName]: { ...prev[streamName], entries: reorderedEntries } };
      });
    }
  };

  const toggleStreamExpand = (streamName) => {
    setExpandedStreams(prev => ({ ...prev, [streamName]: !prev[streamName] }));
  };

  const toggleEntryExpand = (entryInternalId) => {
    setExpandedEntries(prev => ({...prev, [entryInternalId]: !prev[entryInternalId] }));
  };
  
  const renderEntryForm = (streamName, entry) => (
    <Box sx={{ pl: 9, pr: 2, pb: 2, pt: 1, borderTop: '1px solid rgba(255,255,255,0.1)' }}>
      <TextField label="内容" value={entry.content || ''} onChange={(e) => handleEntryChange(streamName, entry._internal_id, 'content', e.target.value)} multiline fullWidth variant="outlined" sx={{ mb: 2 }} autoFocus/>
      <TextField label="级别" value={entry.level || 'event'} onChange={(e) => handleEntryChange(streamName, entry._internal_id, 'level', e.target.value)} fullWidth sx={{ mb: 2 }} size="small" variant="outlined"/>
      <TextField label="标签 (逗号分隔)" value={(entry.tags || []).join(', ')} onChange={(e) => handleEntryChange(streamName, entry._internal_id, 'tags', e.target.value)} fullWidth sx={{ mb: 2 }} variant="outlined" size="small"
        InputProps={{ startAdornment: (<InputAdornment position="start">{(entry.tags || []).filter(t => t).map((tag, i) => <Chip key={i} label={tag} size="small" sx={{ mr: 0.5 }} />)}</InputAdornment>),}}
      />
    </Box>
  );

  return (
    <Box sx={{ p: 2, height: '100%', display: 'flex', flexDirection: 'column' }}>
      <Box sx={{ flexShrink: 0, display: 'flex', alignItems: 'center', gap: 2, mb: 2 }}>
        <Button variant="outlined" onClick={onBack}>返回概览</Button>
        <Typography variant="h5" component="div" sx={{ flexGrow: 1, m: 0 }}>正在编辑Memoria</Typography>
        <TextField label="新Stream名称" value={newStreamName} onChange={e => setNewStreamName(e.target.value)} size="small" variant="outlined" sx={{ width: '200px' }} />
        <Button variant="outlined" startIcon={<AddIcon />} onClick={handleAddStream}>添加Stream</Button>
        <Button variant="contained" color="success" startIcon={<SaveIcon />} onClick={handleSaveAll} disabled={loading}>{loading ? '正在保存...' : '全部保存'}</Button>
      </Box>
      {errorMessage && <Alert severity="error" sx={{ mb: 2 }} onClose={() => setErrorMessage('')}>{errorMessage}</Alert>}
      <Box sx={{ flexGrow: 1, overflowY: 'auto' }}>
        <List disablePadding>
          {Object.entries(streams).map(([streamName, streamData]) => (
            <React.Fragment key={streamName}>
              <ListItem
                button
                onClick={() => toggleStreamExpand(streamName)}
                sx={{
                  borderBottom: '1px solid rgba(255, 255, 255, 0.12)',
                  backgroundColor: 'transparent',
                  '&:hover': { backgroundColor: 'rgba(255, 255, 255, 0.08)' }
                }}
              >
                <ListItemIcon>
                  {expandedStreams[streamName] ? <ExpandMoreIcon /> : <ChevronRightIcon />}
                </ListItemIcon>
                <ListItemText primary={streamName} secondary={`条目数量: ${streamData.entries?.length || 0}`} />
                <IconButton edge="end" onClick={(e) => { e.stopPropagation(); handleDeleteStream(streamName); }}>
                  <DeleteIcon />
                </IconButton>
              </ListItem>
              <Collapse in={!!expandedStreams[streamName]} timeout="auto" unmountOnExit>
                <Box sx={{ pl: 4, pr: 2, pb: 2, pt: 1 }}>
                  <Button variant="outlined" startIcon={<AddIcon />} onClick={() => handleAddEntry(streamName)} size="small" sx={{ mb: 2 }}>添加条目</Button>
                  <DndContext sensors={sensors} collisionDetection={closestCenter} onDragEnd={(e) => handleEntryDragEnd(streamName, e)}>
                    <SortableContext items={(streamData.entries || []).map(e => e._internal_id)} strategy={verticalListSortingStrategy}>
                      <List disablePadding>
                        {(streamData.entries || []).map((entry) => (
                          <SortableMemoryEntryItem
                            key={entry._internal_id}
                            id={entry._internal_id}
                            entry={entry}
                            expanded={!!expandedEntries[entry._internal_id]}
                            onToggleExpand={() => toggleEntryExpand(entry._internal_id)}
                            onDelete={() => handleDeleteEntry(streamName, entry._internal_id)}
                          >
                            <Collapse in={!!expandedEntries[entry._internal_id]} timeout="auto" unmountOnExit>
                              {renderEntryForm(streamName, entry)}
                            </Collapse>
                          </SortableMemoryEntryItem>
                        ))}
                      </List>
                    </SortableContext>
                  </DndContext>
                </Box>
              </Collapse>
            </React.Fragment>
          ))}
        </List>
      </Box>
    </Box>
  );
}
```

### src/editors/CodexEditor.jsx
```
// plugins/sandbox_editor/src/editors/CodexEditor.jsx
import React, { useState, useEffect } from 'react';
import { Box, Typography, List, ListItem, ListItemText, ListItemIcon, Collapse, IconButton, Button, Switch, TextField, MenuItem, Select, Chip, InputAdornment, Alert } from '@mui/material';
import ExpandMoreIcon from '@mui/icons-material/ExpandMore';
import DeleteIcon from '@mui/icons-material/Delete';
import AddIcon from '@mui/icons-material/Add';
import SaveIcon from '@mui/icons-material/Save';
import { DndContext, closestCenter, KeyboardSensor, PointerSensor, useSensor, useSensors } from '@dnd-kit/core';
import { arrayMove, SortableContext, sortableKeyboardCoordinates, verticalListSortingStrategy } from '@dnd-kit/sortable';
import { SortableEntryItem } from '../components/SortableEntryItem';
import { mutate } from '../utils/api';

export function CodexEditor({ sandboxId, basePath, codexName, codexData, onBack }) {
  const [entries, setEntries] = useState([]);
  const [expanded, setExpanded] = useState({});
  const [errorMessage, setErrorMessage] = useState('');
  
  useEffect(() => {
    // --- [修复 1/7] 在加载数据时，为每个条目添加一个稳定的内部ID ---
    // 这个ID仅用于UI（React key, DND-kit, 折叠状态），在保存到后端前会被移除。
    const entriesWithInternalIds = (codexData.entries || []).map((entry, index) => ({
        ...entry,
        _internal_id: entry.id + `_${Date.now()}_${index}`
    }));
    setEntries(entriesWithInternalIds);
  }, [codexData]);

  const sensors = useSensors(
    useSensor(PointerSensor),
    useSensor(KeyboardSensor, { coordinateGetter: sortableKeyboardCoordinates })
  );

  const syncEntries = async (entriesToSave, optimisticState) => {
    setErrorMessage('');
    // 乐观更新UI，使用包含 _internal_id 的状态
    setEntries(optimisticState || entries.map((e, i) => ({...e, _internal_id: entries[i]._internal_id || `temp_${Date.now()}` })));
    try {
      await mutate(sandboxId, [{
        type: 'UPSERT',
        path: `${basePath}/entries`,
        value: entriesToSave // 只保存干净的数据到后端
      }]);
      // 确认最终状态 (如果 optimisticState 提供了，则使用它)
      if (optimisticState) {
          setEntries(optimisticState);
      }
    } catch (e) {
      setErrorMessage(`Failed to save changes: ${e.message}`);
      // 如果失败，回滚到操作前的状态 (此处的 'entries' 是闭包捕获的旧状态)
      setEntries(entries);
    }
  };

  const handleDragEnd = async (event) => {
    const { active, over } = event;
    if (active.id !== over.id) {
      // --- [修复 2/7] 使用 _internal_id 来查找索引 ---
      const oldIndex = entries.findIndex(e => e._internal_id === active.id);
      const newIndex = entries.findIndex(e => e._internal_id === over.id);
      if (oldIndex === -1 || newIndex === -1) return;

      const reorderedEntries = arrayMove(entries, oldIndex, newIndex);
      // 在保存到后端前，移除内部ID
      const entriesToSave = reorderedEntries.map(({_internal_id, ...rest}) => rest);
      await syncEntries(entriesToSave, reorderedEntries);
    }
  };

  const handleDelete = async (internalIdToDelete) => {
    if (!window.confirm(`Are you sure you want to delete this entry?`)) return;
    // --- [修复 3/7] 使用 _internal_id 进行过滤 ---
    const updatedEntries = entries.filter(e => e._internal_id !== internalIdToDelete);
    const entriesToSave = updatedEntries.map(({_internal_id, ...rest}) => rest);
    await syncEntries(entriesToSave, updatedEntries);
  };
  
  const handleToggleEnabled = async (internalId, is_enabled) => {
    const originalEntries = [...entries];
    const updatedEntries = entries.map(e => e._internal_id === internalId ? { ...e, is_enabled } : e);
    const entryIndex = originalEntries.findIndex(e => e._internal_id === internalId);

    if (entryIndex === -1) return;
    
    setEntries(updatedEntries); // 乐观更新
    
    setErrorMessage('');
    try {
        await mutate(sandboxId, [{
            type: 'UPSERT',
            path: `${basePath}/entries/${entryIndex}/is_enabled`,
            value: is_enabled,
        }]);
    } catch (e) {
        setErrorMessage(`Status update failed: ${e.message}`);
        setEntries(originalEntries); // 回滚
    }
  };

  const handleAddEntry = () => {
      // --- [修复 4/7] 添加条目时，同时创建 _internal_id ---
      const newInternalId = `new_entry_internal_${Date.now()}`;
      const newEntry = {
        _internal_id: newInternalId,
        id: `new_entry_${Date.now()}`, 
        content: '', 
        priority: 100, 
        trigger_mode: 'always_on', 
        keywords: [], 
        is_enabled: true,
      };
      const updatedEntries = [...entries, newEntry];
      setEntries(updatedEntries);
      // 使用 _internal_id 作为 key 来展开
      setExpanded(prev => ({...prev, [newInternalId]: true}));
  };

  const handleSaveAll = async () => {
    const ids = new Set();
    for (const entry of entries) {
      if (!entry.id || entry.id.trim() === '') {
        setErrorMessage(`Error: An entry has an empty ID.`);
        return;
      }
      if (ids.has(entry.id)) {
        setErrorMessage(`Error: Duplicate ID "${entry.id}" found.`);
        return;
      }
      ids.add(entry.id);
    }
    // 在保存前，移除所有内部ID
    const entriesToSave = entries.map(({_internal_id, ...rest}) => rest);
    await syncEntries(entriesToSave, entries); // 传入乐观状态以保持UI
    alert('All changes saved!');
  };
  
  const handleEntryChange = (index, field, value) => {
      const updatedEntries = [...entries];
      const entry = updatedEntries[index];
      
      let finalValue = value;
      if (field === 'priority') {
          finalValue = parseInt(value, 10) || 0;
      } else if (field === 'keywords' && typeof value === 'string') {
          finalValue = value.split(',').map(k => k.trim()).filter(Boolean);
      }
      
      updatedEntries[index] = { ...entry, [field]: finalValue };
      setEntries(updatedEntries);
  };

  // --- [修复 5/7] 使用 _internal_id 来切换展开状态 ---
  const toggleExpand = (internalId) => {
    setExpanded(prev => ({ ...prev, [internalId]: !prev[internalId] }));
  };

  const renderEntryForm = (entry, index) => {
    return (
        <Box sx={{ pl: 9, pr: 2, pb: 2, pt: 1, borderTop: '1px solid rgba(255,255,255,0.1)'}}>
            <TextField label="ID" value={entry.id} onChange={(e) => handleEntryChange(index, 'id', e.target.value)} fullWidth sx={{ mt: 2, mb: 2 }} required />
            <TextField label="内容" value={entry.content || ''} onChange={(e) => handleEntryChange(index, 'content', e.target.value)} multiline fullWidth sx={{ mb: 2 }} />
            <TextField label="顺序" type="number" value={entry.priority} onChange={(e) => handleEntryChange(index, 'priority', e.target.value)} fullWidth sx={{ mb: 2 }} />
            <Select value={entry.trigger_mode} onChange={(e) => handleEntryChange(index, 'trigger_mode', e.target.value)} fullWidth sx={{ mb: 2 }}>
                <MenuItem value="always_on">常亮</MenuItem>
                <MenuItem value="on_keyword">按关键字触发</MenuItem>
            </Select>
            {entry.trigger_mode === 'on_keyword' && (
                <TextField label="关键词 (逗号分隔)" value={(entry.keywords || []).join(', ')} onChange={(e) => handleEntryChange(index, 'keywords', e.target.value)} fullWidth sx={{ mb: 2 }}
                    InputProps={{
                        startAdornment: (
                            <InputAdornment position="start">
                                {(entry.keywords || []).filter(k => k).map((kw, i) => <Chip key={i} label={kw} size="small" sx={{ mr: 0.5 }} />)}
                            </InputAdornment>
                        ),
                    }}
                />
            )}
        </Box>
    );
  };

  return (
    <Box sx={{ p: 2, height: '100%', display: 'flex', flexDirection: 'column' }}>
      <Box sx={{ flexShrink: 0, display: 'flex', alignItems: 'center', gap: 2, mb: 2 }}>
        <Button variant="outlined" onClick={onBack}>返回概览</Button>
        <Typography variant="h5" gutterBottom component="div" sx={{flexGrow: 1, m: 0}}>
          正在编辑Codex: {codexName}
        </Typography>
        <Button variant="outlined" color="primary" startIcon={<AddIcon />} onClick={handleAddEntry}>
          添加条目
        </Button>
        <Button variant="contained" color="success" startIcon={<SaveIcon />} onClick={handleSaveAll}>
          全部保存
        </Button>
      </Box>
      
      {errorMessage && <Alert severity="error" sx={{ mb: 2 }} onClose={() => setErrorMessage('')}>{errorMessage}</Alert>}

      <Box sx={{ flexGrow: 1, overflowY: 'auto' }}>
        <DndContext sensors={sensors} collisionDetection={closestCenter} onDragEnd={handleDragEnd}>
          {/* --- [修复 6/7] 使用 _internal_id 作为 dnd-kit 的 ID 来源 --- */}
          <SortableContext items={entries.map(e => e._internal_id)} strategy={verticalListSortingStrategy}>
            <List>
              {entries.map((entry, index) => (
                // --- [修复 7/7] 使用 _internal_id 作为 React key 和组件的唯一标识 ---
                <SortableEntryItem
                  key={entry._internal_id}
                  id={entry._internal_id}
                  entry={entry}
                  expanded={!!expanded[entry._internal_id]}
                  onToggleExpand={() => toggleExpand(entry._internal_id)}
                  onToggleEnabled={(id, enabled) => handleToggleEnabled(entry._internal_id, enabled)} // 传递内部ID
                  onDelete={() => handleDelete(entry._internal_id)} // 传递内部ID
                >
                  <Collapse in={!!expanded[entry._internal_id]} timeout="auto" unmountOnExit>
                    {renderEntryForm(entry, index)}
                  </Collapse>
                </SortableEntryItem>
              ))}
            </List>
          </SortableContext>
        </DndContext>
      </Box>
    </Box>
  );
}
```

### src/editors/LlmContentsEditor.jsx
```
// plugins/sandbox_editor/src/editors/LlmContentsEditor.jsx
import React, { useState, useEffect, useMemo } from 'react';
import { Box, List, Collapse, IconButton, Button, Menu, MenuItem, Typography, Paper, TextField, Select, InputLabel, FormControl, Chip } from '@mui/material';
import { DndContext, closestCenter, KeyboardSensor, PointerSensor, useSensor, useSensors } from '@dnd-kit/core';
import { arrayMove, SortableContext, verticalListSortingStrategy } from '@dnd-kit/sortable';
import { useSortable } from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';

import AddIcon from '@mui/icons-material/Add';
import DeleteIcon from '@mui/icons-material/Delete';
import DragIndicatorIcon from '@mui/icons-material/DragIndicator';
import ExpandMoreIcon from '@mui/icons-material/ExpandMore';
import ChevronRightIcon from '@mui/icons-material/ChevronRight';
import MessageIcon from '@mui/icons-material/Message';
import DynamicFeedIcon from '@mui/icons-material/DynamicFeed';

// Inner component for rendering the sortable list item and its form
function SortableContentItem({ id, item, onUpdate, onDelete, allItems }) {
    const { attributes, listeners, setNodeRef, transform, transition, isDragging } = useSortable({ id });
    const [isExpanded, setIsExpanded] = useState(false);

    const style = {
        transform: CSS.Transform.toString(transform),
        transition,
        zIndex: isDragging ? 1 : 0,
        position: 'relative',
        marginBottom: '8px',
    };
    
    const handleFieldChange = (field, value) => {
        onUpdate({ ...item, [field]: value });
    };

    const renderHeader = () => {
        // --- [修改] 如果条目有名称，则显示名称，否则回退到旧逻辑 ---
        if (item.name) {
             return <>
                {item.type === 'MESSAGE_PART' ? <MessageIcon sx={{ mr: 1, color: 'text.secondary' }} /> : <DynamicFeedIcon sx={{ mr: 1, color: 'text.secondary' }} />}
                <Typography sx={{ flexGrow: 1, fontStyle: 'italic' }}>{item.name}</Typography>
             </>;
        }
        
        // --- 旧的 fallback 逻辑 ---
        if (item.type === 'MESSAGE_PART') {
            return <>
                <MessageIcon sx={{ mr: 1, color: 'text.secondary' }} />
                <Typography sx={{ flexGrow: 1 }}>消息片段</Typography>
                <Chip label={item.role || 'no role'} size="small" variant="outlined" />
            </>;
        }
        if (item.type === 'INJECT_MESSAGES') {
            return <>
                <DynamicFeedIcon sx={{ mr: 1, color: 'text.secondary' }} />
                <Typography sx={{ flexGrow: 1 }}>注入消息</Typography>
            </>;
        }
        return 'Unknown Item';
    };

    return (
        <Paper ref={setNodeRef} style={style} variant="outlined">
            <Box sx={{ display: 'flex', alignItems: 'center', p: 1, backgroundColor: 'rgba(255,255,255,0.05)', cursor: 'pointer' }} onClick={() => setIsExpanded(!isExpanded)}>
                <Box {...attributes} {...listeners} sx={{ cursor: 'grab', display: 'flex', alignItems: 'center', mr:1 }}>
                    <DragIndicatorIcon />
                </Box>
                {renderHeader()}
                <IconButton size="small" sx={{ ml: 1 }}><ChevronRightIcon sx={{ transform: isExpanded ? 'rotate(90deg)' : 'none', transition: 'transform 0.2s' }} /></IconButton>
                <IconButton size="small" sx={{ ml: 1 }} onClick={(e) => { e.stopPropagation(); onDelete(); }}><DeleteIcon /></IconButton>
            </Box>
            <Collapse in={isExpanded}>
                <Box sx={{ p: 2, display: 'flex', flexDirection: 'column', gap: 2 }}>
                    {/* --- [新增] 为所有类型的条目添加名称输入框 --- */}
                    <TextField label="条目名称 (仅供UI显示)" size="small" value={item.name || ''} onChange={(e) => handleFieldChange('name', e.target.value)} />
                    
                    {item.type === 'MESSAGE_PART' && <>
                        <FormControl fullWidth size="small">
                            <InputLabel>角色</InputLabel>
                            <Select label="角色" value={item.role || 'user'} onChange={(e) => handleFieldChange('role', e.target.value)}>
                                <MenuItem value="system">system</MenuItem>
                                <MenuItem value="user">user</MenuItem>
                                <MenuItem value="model">model</MenuItem>
                            </Select>
                        </FormControl>
                        <TextField label="内容 (支持宏)" multiline minRows={3} value={item.content || ''} onChange={(e) => handleFieldChange('content', e.target.value)} />
                        <TextField label="是否启用 (支持宏，留空为 true)" size="small" value={item.is_enabled || ''} onChange={(e) => handleFieldChange('is_enabled', e.target.value)} />
                    </>}
                     {item.type === 'INJECT_MESSAGES' && <>
                        <TextField label="来源 (必须是宏)" multiline minRows={2} value={item.source || ''} onChange={(e) => handleFieldChange('source', e.target.value)} />
                        <TextField label="是否启用 (支持宏，留空为 true)" size="small" value={item.is_enabled || ''} onChange={(e) => handleFieldChange('is_enabled', e.target.value)} />
                    </>}
                </Box>
            </Collapse>
        </Paper>
    );
}


// Main editor component
export function LlmContentsEditor({ contents, onContentsChange }) {
    const [items, setItems] = useState([]);
    const [anchorEl, setAnchorEl] = useState(null);

    useEffect(() => {
        setItems(prevItems => {
            const idMap = new Map(prevItems.map(item => [JSON.stringify({type: item.type, role: item.role, content: item.content, source: item.source}), item._internal_id]));

            const newItems = (contents || []).map((item, index) => {
                if (prevItems.length !== contents.length || !prevItems[index] || !prevItems[index]._internal_id) {
                     return {
                        ...item,
                        _internal_id: `${item.type}_${Date.now()}_${index}`
                    };
                }
                return {
                    ...item,
                    _internal_id: prevItems[index]._internal_id
                };
            });
            return newItems;
        });
    }, [contents]); 

    const sensors = useSensors(
        useSensor(PointerSensor),
        useSensor(KeyboardSensor)
    );
    
    const notifyParent = (newItems) => {
        const cleanedItems = newItems.map(({ _internal_id, ...rest }) => rest);
        onContentsChange(cleanedItems);
    }

    const handleDragEnd = (event) => {
        const { active, over } = event;
        if (active && over && active.id !== over.id) {
            const oldIndex = items.findIndex(item => item._internal_id === active.id);
            const newIndex = items.findIndex(item => item._internal_id === over.id);
            const newOrderedItems = arrayMove(items, oldIndex, newIndex);
            setItems(newOrderedItems);
            notifyParent(newOrderedItems);
        }
    };
    
    const handleAddItem = (type) => {
        let newItem;
        if (type === 'MESSAGE_PART') {
            // --- [修改] 添加默认名称 ---
            newItem = { type: 'MESSAGE_PART', name: '新消息片段', role: 'user', content: '' };
        } else {
            // --- [修改] 添加默认名称 ---
            newItem = { type: 'INJECT_MESSAGES', name: '新消息注入', source: '' };
        }
        const updatedItems = [...items, { ...newItem, _internal_id: `${type}_${Date.now()}_${items.length}` }];
        setItems(updatedItems);
        notifyParent(updatedItems);
        handleCloseMenu();
    };

    const handleUpdateItem = (index, updatedItem) => {
        const newItems = [...items];
        newItems[index] = { ...updatedItem, _internal_id: items[index]._internal_id };
        setItems(newItems);
        notifyParent(newItems);
    };

    const handleDeleteItem = (index) => {
        const newItems = items.filter((_, i) => i !== index);
        setItems(newItems);
        notifyParent(newItems);
    };

    const handleOpenMenu = (event) => setAnchorEl(event.currentTarget);
    const handleCloseMenu = () => setAnchorEl(null);

    return (
        <Paper variant="outlined" sx={{ p: 1, borderColor: 'rgba(255, 255, 255, 0.23)' }}>
            <Typography variant="caption" display="block" color="text.secondary" sx={{ mb: 1 }}>
                对话内容 (Contents)
            </Typography>
            <DndContext sensors={sensors} collisionDetection={closestCenter} onDragEnd={handleDragEnd}>
                <SortableContext items={items.map(i => i._internal_id)} strategy={verticalListSortingStrategy}>
                    <List disablePadding>
                        {items.map((item, index) => (
                            <SortableContentItem
                                key={item._internal_id}
                                id={item._internal_id}
                                item={item}
                                onUpdate={(updated) => handleUpdateItem(index, updated)}
                                onDelete={() => handleDeleteItem(index)}
                            />
                        ))}
                    </List>
                </SortableContext>
            </DndContext>
            {items.length === 0 && (
                 <Typography variant="body2" color="text.secondary" align="center" sx={{p: 2}}>
                    No content defined. Click "Add Item" to start.
                </Typography>
            )}
            <Box sx={{ mt: 1 }}>
                <Button startIcon={<AddIcon />} onClick={handleOpenMenu} size="small" variant="outlined">
                    添加条目
                </Button>
                <Menu anchorEl={anchorEl} open={Boolean(anchorEl)} onClose={handleCloseMenu}>
                    <MenuItem onClick={() => handleAddItem('MESSAGE_PART')}><MessageIcon sx={{ mr: 1 }} />消息片段</MenuItem>
                    <MenuItem onClick={() => handleAddItem('INJECT_MESSAGES')}><DynamicFeedIcon sx={{ mr: 1 }} />注入消息</MenuItem>
                </Menu>
            </Box>
        </Paper>
    );
}
```

### src/editors/RuntimeConfigForm.jsx
```
// plugins/sandbox_editor/src/editors/RuntimeConfigForm.jsx
import React from 'react';
import { Box, TextField, Select, MenuItem, FormControlLabel, Switch, Typography, FormControl, InputLabel } from '@mui/material';
import { CodexInvokeEditor } from './CodexInvokeEditor.jsx';
import { LlmContentsEditor } from './LlmContentsEditor.jsx';

const RUNTIME_CONFIG_SPECS = {
    // LLM
    // llm.default is now handled by a custom component, so it is removed from here.
    // Memoria
    'memoria.add': [
        { key: 'stream', type: 'text', label: '流名称', required: true },
        { key: 'content', type: 'text', label: '内容 (支持宏)', multiline: true, required: true },
        { key: 'level', type: 'text', label: '级别 (如 event, user, model)', default: 'event' },
        { key: 'tags', type: 'text', label: '标签 (逗号分隔)', },
    ],
    'memoria.query': [
        { key: 'stream', type: 'text', label: '流名称', required: true },
        { key: 'latest', type: 'text', label: '最新N条 (数字)' },
        { key: 'levels', type: 'text', label: '级别 (逗号分隔)' },
        { key: 'tags', type: 'text', label: '标签 (逗号分隔)' },
        { key: 'order', type: 'select', label: '排序', options: [{value: 'ascending', label: '升序'}, {value: 'descending', label: '降序'}], default: 'ascending' },
        { 
            key: 'format', 
            type: 'select', 
            label: '返回格式', 
            options: [
                { value: 'raw_entries', label: '原始条目 (raw_entries)' },
                { value: 'message_list', label: '消息列表 (message_list)' },
            ], 
            default: 'raw_entries' 
        },
    ],
    // Codex
    'codex.invoke': [
            { key: 'recursion_enabled', type: 'switch', label: '启用递归', default: false },
            { key: 'debug', type: 'switch', label: '启用调试输出', default: false },
    ],
    // System IO
    'system.io.input': [
        { key: 'value', type: 'text', label: '值 (任意 JSON 兼容)', multiline: true, required: true },
    ],
    'system.io.log': [
        { key: 'message', type: 'text', label: '消息', multiline: true, required: true },
        { key: 'level', type: 'select', label: '级别', options: ['debug', 'info', 'warning', 'error', 'critical'], default: 'info' },
    ],
    // System Data
    'system.data.format': [
        { key: 'items', type: 'text', label: '条目 (列表/字典宏)', multiline: true, required: true },
        { key: 'template', type: 'text', label: '模板 (如 {item.name})', multiline: false, required: true },
        { key: 'joiner', type: 'text', label: '连接符', multiline: false, default: '\\n' },
    ],
    'system.data.parse': [
        { key: 'text', type: 'text', label: '文本 (宏)', multiline: true, required: true },
        { key: 'format', type: 'select', label: '格式', options: ['json', 'xml'], default: 'json', required: true },
        { key: 'strict', type: 'switch', label: '严格模式', default: false },
        { key: 'selector', type: 'text', label: '选择器 (用于 XML)' },
    ],
    'system.data.regex': [
        { key: 'text', type: 'text', label: '文本 (宏)', multiline: true, required: true },
        { key: 'pattern', type: 'text', label: '正则表达式 (如 (?P<name>...))', multiline: false, required: true },
        { key: 'mode', type: 'select', label: '模式', options: [{value: 'search', label: '查找'}, {value: 'find_all', label: '全部查找'}], default: 'search' },
    ],
    // System Flow
    'system.flow.call': [
        { key: 'graph', type: 'text', label: '图名称 (ID)', multiline: false, required: true },
        { key: 'using', type: 'text', label: '参数 (字典宏)', multiline: true },
    ],
    'system.flow.map': [
        { key: 'list', type: 'text', label: '列表 (宏)', multiline: true, required: true },
        { key: 'graph', type: 'text', label: '图名称 (ID)', multiline: false, required: true },
        { key: 'using', type: 'text', label: '参数 (字典宏，可用 source.item)', multiline: true },
        { key: 'collect', type: 'text', label: '收集 (子图节点宏)', multiline: true },
    ],
    // System Advanced
    'system.execute': [
        { key: 'code', type: 'text', label: '代码 (宏)', multiline: true, required: true },
    ],
};

export function RuntimeConfigForm({ runtimeType, config, onConfigChange }) {
  const spec = RUNTIME_CONFIG_SPECS[runtimeType] || [];

  const handleChange = (key, value) => {
    // For comma-separated text fields that should be arrays
    if (['tags', 'levels'].includes(key) && typeof value === 'string') {
        onConfigChange({ ...config, [key]: value.split(',').map(s => s.trim()).filter(Boolean) });
    } else {
        onConfigChange({ ...config, [key]: value });
    }
  };
  
  const hasGenericFields = spec.length > 0;
  const isCodexInvoke = runtimeType === 'codex.invoke';
  const isLlmDefault = runtimeType === 'llm.default';

  if (!hasGenericFields && !isCodexInvoke && !isLlmDefault) {
    return <Typography color="text.secondary" sx={{mt:2}}>该指令不需要额外配置</Typography>
  }

  return (
    <Box sx={{ mt: 2, display: 'flex', flexDirection: 'column', gap: 2 }}>
      <Typography variant="subtitle2">配置</Typography>
      
      {/* --- 核心修复 --- */}
      {isLlmDefault && (
        <>
            {/* 1. 添加模型名称输入框 */}
            <TextField
              key="model"
              label="模型名称"
              required={true}
              value={config.model || ''}
              onChange={(e) => handleChange('model', e.target.value)}
              fullWidth
              variant="outlined"
              size="small"
              placeholder="例如, gemini/gemini-1.5-pro"
              helperText="格式为 '提供商/模型ID'"
            />
            {/* 2. 渲染原有的 contents 编辑器 */}
            <LlmContentsEditor 
                contents={config.contents || []} 
                onContentsChange={(newContents) => handleChange('contents', newContents)}
            />
        </>
      )}
      
      {isCodexInvoke && (
        <CodexInvokeEditor
            value={config.from}
            onChange={(newValue) => handleChange('from', newValue)}
        />
      )}
      
      {spec.map(field => {
        let value = config[field.key];
        // Handle array-to-string conversion for text fields
        if (['tags', 'levels'].includes(field.key) && Array.isArray(value)) {
            value = value.join(', ');
        }
        // Set default value
        value = value ?? field.default ?? (field.type === 'switch' ? false : '');
        
        switch (field.type) {
          case 'text':
            return (
              <TextField
                key={field.key}
                label={field.label}
                required={field.required}
                value={value}
                onChange={(e) => handleChange(field.key, e.target.value)}
                fullWidth
                multiline={field.multiline}
                variant="outlined"
                size="small"
              />
            );
          case 'select':
            return (
             <FormControl fullWidth size="small" key={field.key}>
                <InputLabel>{field.label}</InputLabel>
                <Select
                    label={field.label}
                    required={field.required}
                    value={value}
                    onChange={(e) => handleChange(field.key, e.target.value)}
                >
                    {field.options.map(opt => {
                      const optValue = typeof opt === 'object' ? opt.value : opt;
                      const optLabel = typeof opt === 'object' ? opt.label : opt;
                      return <MenuItem key={optValue} value={optValue}>{optLabel}</MenuItem>
                    })}
                </Select>
            </FormControl>
            );
          case 'switch':
            return (
              <FormControlLabel
                key={field.key}
                control={
                  <Switch
                    checked={!!value}
                    onChange={(e) => handleChange(field.key, e.target.checked)}
                  />
                }
                label={field.label}
              />
            );
          default:
            return null;
        }
      })}
    </Box>
  );
}
```

### src/editors/CodexInvokeEditor.jsx
```
// plugins/sandbox_editor/src/editors/CodexInvokeEditor.jsx
import React, { useState, useEffect } from 'react';
import { Paper, List, ListItem, TextField, IconButton, Button, Typography } from '@mui/material';
import AddIcon from '@mui/icons-material/Add';
import DeleteIcon from '@mui/icons-material/Delete';

export function CodexInvokeEditor({ value, onChange }) {
    const [sources, setSources] = useState([]);
    const [isInvalidFormat, setIsInvalidFormat] = useState(false);

    useEffect(() => {
        let parsedData = null;
        let isValid = false;

        // 1. 优先处理已经是数组的情况 (最常见)
        if (Array.isArray(value)) {
            isValid = true;
            parsedData = value;
        } 
        // 2. 处理空值 (null, undefined, '')
        else if (!value) {
            isValid = true;
            parsedData = [];
        } 
        // 3. 最后尝试处理字符串
        else if (typeof value === 'string') {
            try {
                const parsed = JSON.parse(value);
                if (Array.isArray(parsed)) {
                    isValid = true;
                    parsedData = parsed;
                }
            } catch (e) {
                // 解析失败，isValid 保持 false
            }
        }
        
        if (isValid) {
            // 清理数据，确保数组中的每一项都是具有所需键的有效对象
            const sanitized = parsedData.map(item => 
                (typeof item === 'object' && item !== null && !Array.isArray(item))
                    ? { codex: item.codex || '', source: item.source || '' }
                    : { codex: '', source: '' } // 如果项格式不正确，则重置
            );
            setSources(sanitized);
            setIsInvalidFormat(false);
        } else {
            setSources([]);
            setIsInvalidFormat(true);
        }
    }, [value]);

    const notifyChange = (newSources) => {
        // [核心修复] 直接传递原生数组，而不是JSON字符串
        onChange(newSources);
    };

    const handleItemChange = (index, field, fieldValue) => {
        const newSources = [...sources];
        newSources[index] = { ...newSources[index], [field]: fieldValue };
        setSources(newSources);
        notifyChange(newSources);
    };

    const handleAddItem = () => {
        const newSources = [...sources, { codex: '', source: '' }];
        setSources(newSources);
        notifyChange(newSources);
    };

    const handleDeleteItem = (index) => {
        const newSources = sources.filter((_, i) => i !== index);
        setSources(newSources);
        notifyChange(newSources);
    };

    if (isInvalidFormat) {
        return (
            <Typography color="error.main" variant="body2" sx={{mt: 1}}>
                'from' 字段中的数据格式无效。请修正或清空以使用此编辑器。
            </Typography>
        )
    }

    return (
        <Paper variant="outlined" sx={{ p: 1.5, mt: 1, borderColor: 'rgba(255, 255, 255, 0.23)' }}>
             <Typography variant="caption" display="block" color="text.secondary" sx={{ mb: 1 }}>
                Codex 数据源
            </Typography>
            <List disablePadding>
                {sources.map((item, index) => (
                    <ListItem key={index} disablePadding sx={{ display: 'flex', gap: 1, mb: 1.5, alignItems: 'center' }}>
                        <TextField
                            label="Codex 名称"
                            value={item.codex || ''}
                            onChange={(e) => handleItemChange(index, 'codex', e.target.value)}
                            size="small"
                            fullWidth
                            variant="outlined"
                            placeholder='例如, "npc_status"'
                        />
                        <TextField
                            label="激活源 (宏)"
                            value={item.source || ''}
                            onChange={(e) => handleItemChange(index, 'source', e.target.value)}
                            size="small"
                            fullWidth
                            variant="outlined"
                            placeholder='例如, "{{pipe.input.text}}"'
                        />
                        <IconButton onClick={() => handleDeleteItem(index)} color="error" title="删除数据源">
                            <DeleteIcon />
                        </IconButton>
                    </ListItem>
                ))}
                {sources.length === 0 && <Typography variant="body2" color="text.secondary" align="center" sx={{mb: 1}}>未定义数据源。</Typography>}
            </List>
            <Button startIcon={<AddIcon />} onClick={handleAddItem} size="small" variant="outlined">
                添加数据源
            </Button>
        </Paper>
    );
};
```

### src/editors/AddItemDialog.jsx
```
import React, { useState, useEffect } from 'react';
import { Dialog, DialogTitle, DialogContent, DialogActions, Button, TextField, Select, MenuItem, FormControl, InputLabel, Box, Switch, FormControlLabel, Alert } from '@mui/material';

// 预定义的 hevno 类型模板，方便用户快速创建
const PREFAB_TEMPLATES = {
    'hevno/graph': {
        __hevno_type__: 'hevno/graph',
        nodes: [],
        metadata: {},
    },
    'hevno/codex': {
        __hevno_type__: 'hevno/codex',
        entries: [],
        description: 'A new codex.',
    },
    'hevno/memoria': {
        __hevno_type__: 'hevno/memoria',
        __global_sequence__: 0,
    }
};

export function AddItemDialog({ open, onClose, onAdd, parentPath, existingKeys = [] }) {
    const [key, setKey] = useState('');
    const [type, setType] = useState('string');
    const [value, setValue] = useState('');
    const [error, setError] = useState('');

    useEffect(() => {
        if (open) {
            // 重置状态
            setKey('');
            setType('string');
            setValue('');
            setError('');
        }
    }, [open]);

    const handleAdd = async () => {
        setError('');
        if (!key.trim()) {
            setError('Key is required.');
            return;
        }
        if (existingKeys.includes(key.trim())) {
            setError(`Key "${key.trim()}" already exists at this level.`);
            return;
        }

        let finalValue;
        try {
            switch (type) {
                case 'string': finalValue = value; break;
                case 'number':
                    finalValue = Number(value);
                    if (isNaN(finalValue)) throw new Error("Invalid number.");
                    break;
                case 'boolean': finalValue = value === 'true'; break;
                case 'object': finalValue = {}; break;
                case 'array': finalValue = []; break;
                case 'hevno/graph':
                case 'hevno/codex':
                case 'hevno/memoria':
                    finalValue = PREFAB_TEMPLATES[type]; break;
                default:
                    throw new Error("Invalid type selected.");
            }
            await onAdd(parentPath, key.trim(), finalValue);
        } catch (e) {
            setError(`Failed to add item: ${e.message}`);
        }
    };

    const renderValueInput = () => {
        switch (type) {
            case 'string':
                return <TextField label="Value" value={value} onChange={e => setValue(e.target.value)} fullWidth autoFocus margin="dense" />;
            case 'number':
                return <TextField label="Value" type="number" value={value} onChange={e => setValue(e.target.value)} fullWidth autoFocus margin="dense" />;
            case 'boolean':
                return <FormControlLabel control={<Switch checked={value === 'true'} onChange={(e) => setValue(String(e.target.checked))} />} label={value === 'true' ? 'True' : 'False'} sx={{mt:1}} />;
            case 'object':
            case 'array':
            case 'hevno/graph':
            case 'hevno/codex':
            case 'hevno/memoria':
                // 这些类型不需要用户输入初始值
                return null; 
            default:
                return null;
        }
    };

    if (!open) return null;

    return (
        <Dialog open={open} onClose={onClose} fullWidth maxWidth="sm">
            <DialogTitle>加入新对象到 "{parentPath}"</DialogTitle>
            <DialogContent>
                {error && <Alert severity="error" sx={{ mb: 2 }}>{error}</Alert>}
                <TextField label="键 / 名称" value={key} onChange={e => setKey(e.target.value)} fullWidth required autoFocus margin="dense" />
                <FormControl fullWidth margin="dense">
                    <InputLabel id="add-item-type-label">类型</InputLabel>
                    <Select labelId="add-item-type-label" value={type} label="类型" onChange={e => setType(e.target.value)}>
                        <MenuItem value="string">字符串</MenuItem>
                        <MenuItem value="number">数字</MenuItem>
                        <MenuItem value="boolean">布尔值</MenuItem>
                        <MenuItem value="object">对象（空）</MenuItem>
                        <MenuItem value="array">数组（空）</MenuItem>
                        <MenuItem value="hevno/graph">预制件：图</MenuItem>
                        <MenuItem value="hevno/codex">预制件：Codex</MenuItem>
                        <MenuItem value="hevno/memoria">预制件：Memoria</MenuItem>
                    </Select>   
                </FormControl>
                {renderValueInput()}
            </DialogContent>
            <DialogActions>
                <Button onClick={onClose}>取消</Button>
                <Button onClick={handleAdd} variant="contained">添加</Button>
            </DialogActions>
        </Dialog>
    );
}
```

### src/editors/GenericEditorDialog.jsx
```
import React, { useState, useEffect } from 'react';
import { Dialog, DialogTitle, DialogContent, DialogActions, Button, TextField, FormControlLabel, Switch, Typography, Alert, Box } from '@mui/material';

export function GenericEditorDialog({ open, onClose, onSave, item }) {
    // 如果没有 item，则提前返回，防止渲染错误
    if (!item) return null;

    const { path, value: initialValue } = item;
    
    const [dataType, setDataType] = useState('string');
    const [currentValue, setCurrentValue] = useState('');
    const [error, setError] = useState('');

    useEffect(() => {
        if (open && item) {
            setError('');
            const type = typeof initialValue;
            if (type === 'boolean') {
                setDataType('boolean');
                setCurrentValue(initialValue);
            } else if (type === 'string') {
                setDataType('string');
                setCurrentValue(initialValue);
            } else if (type === 'number') {
                setDataType('number');
                setCurrentValue(String(initialValue));
            } else if (type === 'object' && initialValue !== null) {
                setDataType('json');
                setCurrentValue(JSON.stringify(initialValue, null, 2));
            } else { // 处理 null
                setDataType('json');
                setCurrentValue('null');
            }
        }
    }, [open, item, initialValue]);

    const handleSave = async () => {
        setError('');
        let finalValue;
        try {
            switch (dataType) {
                case 'boolean':
                    finalValue = currentValue;
                    break;
                case 'number':
                    finalValue = Number(currentValue);
                    if (isNaN(finalValue)) {
                        throw new Error("无效的数字格式。");
                    }
                    break;
                case 'json':
                    finalValue = JSON.parse(currentValue);
                    break;
                case 'string':
                default:
                    finalValue = currentValue;
                    break;
            }
            // onSave 是一个 async 函数
            await onSave(path, finalValue);
        } catch (e) {
            setError(`保存失败: ${e.message}`);
        }
    };

    const renderInput = () => {
        switch (dataType) {
            case 'boolean':
                return <FormControlLabel control={<Switch checked={!!currentValue} onChange={(e) => setCurrentValue(e.target.checked)} />} label={currentValue ? 'True / On' : 'False / Off'} />;
            case 'string':
                return <TextField label="值" value={currentValue} onChange={(e) => setCurrentValue(e.target.value)} fullWidth multiline minRows={3} variant="outlined" autoFocus />;
            case 'number':
                return <TextField label="值" type="number" value={currentValue} onChange={(e) => setCurrentValue(e.target.value)} fullWidth variant="outlined" autoFocus />;
            case 'json':
                return <TextField label="值 (JSON格式)" value={currentValue} onChange={(e) => setCurrentValue(e.target.value)} fullWidth multiline minRows={10} variant="outlined" autoFocus sx={{ fontFamily: 'monospace' }} />;
            default:
                return <Typography color="error">不支持的数据类型。</Typography>;
        }
    };

    return (
        <Dialog open={open} onClose={onClose} fullWidth maxWidth="md">
            <DialogTitle>编辑值</DialogTitle>
            <DialogContent>
                <Box sx={{ mb: 2 }}>
                    <Typography variant="caption" display="block" color="text.secondary">
                        路径: {path}
                    </Typography>
                    <Typography variant="caption" display="block" color="text.secondary">
                        类型: {dataType}
                    </Typography>
                </Box>
                {error && <Alert severity="error" sx={{ mb: 2 }}>{error}</Alert>}
                {renderInput()}
            </DialogContent>
            <DialogActions>
                <Button onClick={onClose}>取消</Button>
                <Button onClick={handleSave} variant="contained">保存</Button>
            </DialogActions>
        </Dialog>
    );
}
```

### src/editors/GraphEditor.jsx
```
// plugins/sandbox_editor/src/editors/GraphEditor.jsx
import React, { useState, useRef, useEffect } from 'react';
import { Box, Typography, List, Collapse, IconButton, Button, TextField, Alert, Chip, InputAdornment } from '@mui/material';
import ExpandMoreIcon from '@mui/icons-material/ExpandMore';
import DeleteIcon from '@mui/icons-material/Delete';
import AddIcon from '@mui/icons-material/Add';
import SaveIcon from '@mui/icons-material/Save';
import { DndContext, closestCenter, KeyboardSensor, PointerSensor, useSensor, useSensors } from '@dnd-kit/core';
import { arrayMove, SortableContext, sortableKeyboardCoordinates, verticalListSortingStrategy } from '@dnd-kit/sortable';
import { SortableNodeItem } from '../components/SortableNodeItem';
import { RuntimeEditor } from './RuntimeEditor';
import { mutate } from '../utils/api';

export function GraphEditor({ sandboxId, basePath, graphName, graphData, onBack }) {
    const [nodes, setNodes] = useState([]);
    const [expandedNodes, setExpandedNodes] = useState({});
    const [errorMessage, setErrorMessage] = useState('');
    const newNodeFormRef = useRef(null);

    useEffect(() => {
        // --- [修复 1/7] 在加载数据时，为每个节点添加一个稳定的内部ID ---
        const nodesWithInternalIds = (graphData.nodes || []).map((node, index) => ({
            ...node,
            _internal_id: node.id + `_${Date.now()}_${index}`
        }));
        setNodes(nodesWithInternalIds);
    }, [graphData]);

    const sensors = useSensors(
        useSensor(PointerSensor),
        useSensor(KeyboardSensor, { coordinateGetter: sortableKeyboardCoordinates })
    );

    const syncNodes = async (nodesToSave, optimisticState) => {
        setErrorMessage('');
        // 乐观更新UI，使用包含 _internal_id 的状态
        setNodes(optimisticState);
        try {
            await mutate(sandboxId, [{
                type: 'UPSERT',
                path: `${basePath}/nodes`,
                value: nodesToSave, // 只保存干净的数据到后端
            }]);
            // 确认最终状态
            setNodes(optimisticState);
        } catch (e) {
            setErrorMessage(`Failed to save graph changes: ${e.message}`);
            // 如果失败，回滚到操作前的状态 (此处的 'nodes' 是闭包捕获的旧状态)
            setNodes(nodes);
        }
    };
    
    const handleNodeDragEnd = async (event) => {
        const { active, over } = event;
        if (active.id !== over.id) {
            // --- [修复 2/7] 使用 _internal_id 来查找索引 ---
            const oldIndex = nodes.findIndex(n => n._internal_id === active.id);
            const newIndex = nodes.findIndex(n => n._internal_id === over.id);
            if (oldIndex === -1 || newIndex === -1) return;

            const reorderedNodes = arrayMove(nodes, oldIndex, newIndex);
            const nodesToSave = reorderedNodes.map(({_internal_id, ...rest}) => rest);
            await syncNodes(nodesToSave, reorderedNodes);
        }
    };
    
    const handleSaveAllNodes = async () => {
        const ids = new Set();
        for (const node of nodes) {
            if (!node.id || node.id.trim() === '') {
                setErrorMessage(`Error: A node has an empty ID.`);
                return;
            }
            if (ids.has(node.id)) {
                setErrorMessage(`Error: Duplicate node ID "${node.id}" found.`);
                return;
            }
            ids.add(node.id);
        }
        const nodesToSave = nodes.map(({_internal_id, ...rest}) => rest);
        await syncNodes(nodesToSave, nodes);
        alert('Graph saved!');
    };
    
    const handleDeleteNode = async (internalIdToDelete) => {
        if (!window.confirm(`Are you sure you want to delete this node?`)) return;
        // --- [修复 3/7] 使用 _internal_id 进行过滤 ---
        const updatedNodes = nodes.filter(n => n._internal_id !== internalIdToDelete);
        const nodesToSave = updatedNodes.map(({_internal_id, ...rest}) => rest);
        await syncNodes(nodesToSave, updatedNodes);
    };

    const handleAddNode = () => {
        // --- [修复 4/7] 添加节点时，同时创建 _internal_id ---
        const newInternalId = `new_node_internal_${Date.now()}`;
        const newNode = { 
            _internal_id: newInternalId,
            id: `new_node_${Date.now()}`, 
            depends_on: [], 
            run: [], 
            metadata: {} 
        };
        setNodes(prev => [...prev, newNode]);
        // 使用 _internal_id 作为 key 来展开
        setExpandedNodes(prev => ({...prev, [newInternalId]: true}));
        setTimeout(() => {
             newNodeFormRef.current?.scrollIntoView({ behavior: 'smooth', block: 'end' });
        }, 100)
    };
    
    const handleNodeChange = (index, field, value) => {
        const updatedNodes = [...nodes];
        let finalValue = value;
        if (field === 'depends_on' && typeof value === 'string') {
            finalValue = value.split(',').map(id => id.trim()).filter(Boolean)
        }
        updatedNodes[index] = { ...updatedNodes[index], [field]: finalValue };
        setNodes(updatedNodes);
    };

    // --- [修复 5/7] 使用 _internal_id 来切换展开状态 ---
    const toggleNodeExpand = (internalId) => {
        setExpandedNodes(prev => ({ ...prev, [internalId]: !prev[internalId] }));
    };

    const renderNodeForm = (node, index) => {
        return (
            <Box sx={{ pl: 9, pr: 2, pb: 2, borderTop: '1px solid rgba(255,255,255,0.1)' }}>
                <TextField label="节点ID" value={node.id} onChange={(e) => handleNodeChange(index, 'id', e.target.value)} fullWidth sx={{ mt: 2, mb: 2 }} required />
                <TextField label="依赖于 (逗号分隔的ID)" value={(node.depends_on || []).join(', ')} onChange={(e) => handleNodeChange(index, 'depends_on', e.target.value)} fullWidth sx={{ mb: 2 }}
                    InputProps={{
                        startAdornment: (
                            <InputAdornment position="start">
                                {(node.depends_on || []).filter(id => id).map((id, i) => <Chip key={i} label={id} size="small" sx={{ mr: 0.5 }} />)}
                            </InputAdornment>
                        ),
                    }}
                />
                <RuntimeEditor
                    runList={node.run || []}
                    onRunListChange={(newRunList) => handleNodeChange(index, 'run', newRunList)}
                />
            </Box>
        );
    };

    return (
        <Box sx={{ p: 2, height: '100%', display: 'flex', flexDirection: 'column' }}>
            <Box sx={{ flexShrink: 0, display: 'flex', alignItems: 'center', gap: 2, mb: 2 }}>
                <Button variant="outlined" onClick={onBack}>返回概览</Button>
                <Typography variant="h5" component="div" sx={{flexGrow: 1}}>正在编辑Graph: {graphName}</Typography>
                <Button variant="outlined" startIcon={<AddIcon />} onClick={handleAddNode}>添加节点</Button>
                <Button variant="contained" color="success" startIcon={<SaveIcon />} onClick={handleSaveAllNodes}>全部保存</Button>
            </Box>
            {errorMessage && <Alert severity="error" sx={{ mb: 2 }} onClose={() => setErrorMessage('')}>{errorMessage}</Alert>}

            <Box sx={{ flexGrow: 1, overflowY: 'auto' }}>
                <DndContext sensors={sensors} collisionDetection={closestCenter} onDragEnd={handleNodeDragEnd}>
                    {/* --- [修复 6/7] 使用 _internal_id 作为 dnd-kit 的 ID 来源 --- */}
                    <SortableContext items={nodes.map(n => n._internal_id)} strategy={verticalListSortingStrategy}>
                        <List>
                            {nodes.map((node, index) => (
                                // --- [修复 7/7] 使用 _internal_id 作为 React key 和组件的唯一标识 ---
                                <div key={node._internal_id} ref={index === nodes.length -1 ? newNodeFormRef : null}>
                                    <SortableNodeItem
                                        id={node._internal_id}
                                        node={node}
                                        expanded={!!expandedNodes[node._internal_id]}
                                        onToggleExpand={() => toggleNodeExpand(node._internal_id)}
                                        onDelete={() => handleDeleteNode(node._internal_id)}
                                    >
                                        <Collapse in={!!expandedNodes[node._internal_id]} timeout="auto" unmountOnExit>
                                            {renderNodeForm(node, index)}
                                        </Collapse>
                                    </SortableNodeItem>
                                </div>
                            ))}
                        </List>
                    </SortableContext>
                </DndContext>
            </Box>
        </Box>
    );
}
```

### src/utils/constants.js
```
export const SCOPE_TABS = ['definition', 'lore', 'moment'];

export const isObject = (value) => value && typeof value === 'object' && !Array.isArray(value);
export const isArray = (value) => Array.isArray(value);
```

### src/utils/api.js
```
// plugins/sandbox_editor/src/api.js

const BASE_URL = '/api/sandboxes';

/**
 * 构造一个统一的、包含多个修改指令的请求，并发送到后端的 :mutate 端点。
 * @param {string} sandboxId - 目标沙盒的ID。
 * @param {Array<object>} mutations - 一个或多个突变对象的数组。
 * @returns {Promise<object>} - 后端的响应。
 * @throws {Error} - 如果API调用失败。
 */
export async function mutate(sandboxId, mutations) {
  const response = await fetch(`${BASE_URL}/${sandboxId}/resource:mutate`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ mutations }),
  });

  if (!response.ok) {
    const err = await response.json().catch(() => ({ detail: "API mutation failed with non-JSON response." }));
    throw new Error(err.detail || `HTTP Error ${response.status}`);
  }

  return response.json();
}

/**
 * 构造一个统一的、包含多个路径的请求，并发送到后端的 :query 端点。
 * @param {string} sandboxId - 目标沙盒的ID。
 * @param {Array<string>} paths - 要查询的数据路径数组。
 * @returns {Promise<object>} - 一个以路径为键，数据为值的对象。
 * @throws {Error} - 如果API调用失败。
 */
export async function query(sandboxId, paths) {
  const response = await fetch(`${BASE_URL}/${sandboxId}/resource:query`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ paths }),
  });

  if (!response.ok) {
    const err = await response.json().catch(() => ({ detail: "API query failed with non-JSON response." }));
    throw new Error(err.detail || `HTTP Error ${response.status}`);
  }

  const data = await response.json();
  return data.results;
}


/**
 * 请求后端使用沙盒的 definition 来重置 lore 和 moment。
 * @param {string} sandboxId - 目标沙盒的ID。
 * @returns {Promise<object>} - 更新后的沙盒对象。
 * @throws {Error} - 如果API调用失败。
 */
export async function applyDefinition(sandboxId) {
  const response = await fetch(`${BASE_URL}/${sandboxId}/apply_definition`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
  });

  if (!response.ok) {
    const err = await response.json().catch(() => ({ detail: "Failed to apply definition." }));
    throw new Error(err.detail || `HTTP Error ${response.status}`);
  }

  return response.json();
}
```

### src/components/SortableNodeItem.jsx
```
// plugins/sandbox_editor/src/components/SortableNodeItem.jsx
// 类似于 SortableEntryItem，但为 node 定制
import React from 'react';
import { ListItem, ListItemIcon, ListItemText, IconButton } from '@mui/material';
import DragIndicatorIcon from '@mui/icons-material/DragIndicator';
import ExpandMoreIcon from '@mui/icons-material/ExpandMore';
import ChevronRightIcon from '@mui/icons-material/ChevronRight';
import DeleteIcon from '@mui/icons-material/Delete';
import { useSortable } from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';

export function SortableNodeItem({ id, node, expanded, onToggleExpand, onDelete, children }) {
  const { attributes, listeners, setNodeRef, transform, transition, isDragging } = useSortable({ id });

  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
    zIndex: isDragging ? 1 : 0,
    position: 'relative',
    backgroundColor: isDragging ? 'rgba(255, 255, 255, 0.1)' : 'transparent'
  };

  return (
    <div ref={setNodeRef} style={style}>
      <ListItem
        button
        onClick={() => onToggleExpand(node.id)}
        sx={{ borderBottom: '1px solid rgba(255, 255, 255, 0.12)' }}
      >
        <ListItemIcon {...attributes} {...listeners} sx={{ cursor: 'grab' }}>
          <DragIndicatorIcon />
        </ListItemIcon>
        <ListItemIcon>
          {expanded ? <ExpandMoreIcon /> : <ChevronRightIcon />}
        </ListItemIcon>
        <ListItemText primary={node.id} secondary={`Runs: ${node.run?.length || 0}`} />
        <IconButton onClick={(e) => { e.stopPropagation(); onDelete(node.id); }}>
          <DeleteIcon />
        </IconButton>
      </ListItem>
      {children}
    </div>
  );
}
```

### src/components/SortableRuntimeItem.jsx
```
// plugins/sandbox_editor/src/components/SortableRuntimeItem.jsx
import React from 'react';
import { ListItem, ListItemIcon, ListItemText, IconButton, Chip } from '@mui/material';
import DragIndicatorIcon from '@mui/icons-material/DragIndicator';
import EditIcon from '@mui/icons-material/Edit';
import DeleteIcon from '@mui/icons-material/Delete';
import { useSortable } from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';

export function SortableRuntimeItem({ id, run, onEdit, onDelete }) {
  const { attributes, listeners, setNodeRef, transform, transition, isDragging } = useSortable({ id });

  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
    zIndex: isDragging ? 1 : 0,
    position: 'relative',
    backgroundColor: isDragging ? 'rgba(255, 255, 255, 0.1)' : 'transparent'
  };

  return (
    <div ref={setNodeRef} style={style}>
      <ListItem
        // --- [FIX] Removed the "button" prop and the top-level onClick handler ---
        sx={{ pl: 2, borderBottom: '1px dashed rgba(255, 255, 255, 0.08)' }}
        secondaryAction={
            <>
                <IconButton size="small" edge="end" aria-label="edit" onClick={onEdit}>
                    <EditIcon fontSize="small" />
                </IconButton>
                <IconButton size="small" edge="end" aria-label="delete" onClick={(e) => { e.stopPropagation(); onDelete(); }}>
                    <DeleteIcon fontSize="small" />
                </IconButton>
            </>
        }
      >
        <ListItemIcon {...attributes} {...listeners} sx={{ cursor: 'grab', minWidth: 32 }}>
          <DragIndicatorIcon fontSize="small" />
        </ListItemIcon>
        <ListItemText 
            primary={<Chip label={run.runtime || 'Untitled'} size="small" variant="outlined" />} 
            secondary={`Config keys: ${Object.keys(run.config || {}).length}`} 
        />
      </ListItem>
    </div>
  );
}
```

### src/components/SortableEntryItem.jsx
```
// plugins/sandbox_editor/src/components/SortableEntryItem.jsx
import React from 'react';
import { ListItem, ListItemIcon, ListItemText, Switch, IconButton } from '@mui/material';
import DragIndicatorIcon from '@mui/icons-material/DragIndicator';
import ExpandMoreIcon from '@mui/icons-material/ExpandMore';
import ChevronRightIcon from '@mui/icons-material/ChevronRight';
import DeleteIcon from '@mui/icons-material/Delete';
import { useSortable } from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';

export function SortableEntryItem({ id, entry, expanded, onToggleExpand, onToggleEnabled, onDelete, children }) {
  const {
    attributes,
    listeners,
    setNodeRef,
    transform,
    transition,
    isDragging,
  } = useSortable({ id });

  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
    zIndex: isDragging ? 1 : 0,
    position: 'relative',
    backgroundColor: isDragging ? 'rgba(255, 255, 255, 0.1)' : 'transparent'
  };

  return (
    <div ref={setNodeRef} style={style}>
      <ListItem
        button
        onClick={() => onToggleExpand(entry.id)}
        sx={{ borderBottom: '1px solid rgba(255, 255, 255, 0.12)' }}
      >
        <ListItemIcon {...attributes} {...listeners} sx={{ cursor: 'grab' }}>
          <DragIndicatorIcon />
        </ListItemIcon>
        <ListItemIcon>
          {expanded ? <ExpandMoreIcon /> : <ChevronRightIcon />}
        </ListItemIcon>
        <ListItemText primary={entry.id} secondary={`Priority: ${entry.priority}`} />
        <Switch
          checked={entry.is_enabled}
          onChange={(e) => onToggleEnabled(entry.id, e.target.checked)}
          onClick={(e) => e.stopPropagation()}
        />
        <IconButton onClick={(e) => { e.stopPropagation(); onDelete(entry.id); }}>
          <DeleteIcon />
        </IconButton>
      </ListItem>
      {children}
    </div>
  );
}
```

### src/components/SortableMemoryEntryItem.jsx
```
// plugins/sandbox_editor/src/components/SortableMemoryEntryItem.jsx
import React from 'react';
import { ListItem, ListItemIcon, ListItemText, IconButton, Chip, Box } from '@mui/material';
import DragIndicatorIcon from '@mui/icons-material/DragIndicator';
import ExpandMoreIcon from '@mui/icons-material/ExpandMore';
import ChevronRightIcon from '@mui/icons-material/ChevronRight';
import DeleteIcon from '@mui/icons-material/Delete';
import { useSortable } from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';

export function SortableMemoryEntryItem({ id, entry, expanded, onToggleExpand, onDelete, children }) {
  const {
    attributes,
    listeners,
    setNodeRef,
    transform,
    transition,
    isDragging,
  } = useSortable({ id });

  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
    zIndex: isDragging ? 1 : 0,
    position: 'relative',
    backgroundColor: isDragging ? 'rgba(255, 255, 255, 0.1)' : 'transparent',
  };

  const contentPreview = entry.content ? `${entry.content.slice(0, 80)}...` : '无内容';

  return (
    <div ref={setNodeRef} style={style}>
      <ListItem
        button
        onClick={onToggleExpand}
        sx={{
          borderBottom: '1px solid rgba(255, 255, 255, 0.12)',
          backgroundColor: 'transparent',
          '&:hover': { backgroundColor: 'rgba(255, 255, 255, 0.08)' }
        }}
      >
        <ListItemIcon {...attributes} {...listeners} sx={{ cursor: 'grab' }}>
          <DragIndicatorIcon />
        </ListItemIcon>
        <ListItemIcon>
          {expanded ? <ExpandMoreIcon /> : <ChevronRightIcon />}
        </ListItemIcon>
        <ListItemText
          primary={contentPreview}
          secondary={
            <Box component="span" sx={{ display: 'flex', alignItems: 'center', gap: 0.5, mt: 0.5 }}>
              <Chip label={entry.level || 'event'} size="small" variant="outlined" />
              {(entry.tags || []).map(tag => (
                <Chip key={tag} label={tag} size="small" />
              ))}
            </Box>
          }
          secondaryTypographyProps={{ component: 'div' }}
        />
        <IconButton onClick={(e) => { e.stopPropagation(); onDelete(); }}>
          <DeleteIcon />
        </IconButton>
      </ListItem>
      {children}
    </div>
  );
}
```

### src/components/DataTree.jsx
```
import React, { useState } from 'react';
import { List, ListItem, ListItemIcon, ListItemText, Collapse, IconButton, Typography, Box } from '@mui/material';
import FolderIcon from '@mui/icons-material/Folder';
import DescriptionIcon from '@mui/icons-material/Description';
import ExpandMoreIcon from '@mui/icons-material/ExpandMore';
import ChevronRightIcon from '@mui/icons-material/ChevronRight';
import EditIcon from '@mui/icons-material/Edit';
import AddCircleOutlineIcon from '@mui/icons-material/AddCircleOutline'; 
import AutoStoriesIcon from '@mui/icons-material/AutoStories';
import AccountTreeIcon from '@mui/icons-material/AccountTree';
import HistoryIcon from '@mui/icons-material/History';
import { isObject, isArray } from '../utils/constants';

const HEVNO_TYPE_EDITORS = {
  'hevno/codex': { icon: <AutoStoriesIcon />, tooltip: 'Edit Codex' },
  'hevno/graph': { icon: <AccountTreeIcon />, tooltip: 'Edit Graph' },
  'hevno/memoria': { icon: <HistoryIcon />, tooltip: 'Edit Memoria' },
};

// --- 添加 onAdd prop ---
export function DataTree({ data, path = '', onEdit, onAdd }) {
  const [expanded, setExpanded] = useState({});
  const toggleExpand = (key) => { setExpanded((prev) => ({ ...prev, [key]: !prev[key] })); };

  if (!data) return null;

  return (
    <List disablePadding sx={{ pl: path.split('/').length > 1 ? 2 : 0 }}>
      {Object.entries(data).map(([key, value]) => {
        if (key === '__hevno_type__') return null;

        const currentPath = path ? `${path}/${key}` : key;
        const isExpandable = isObject(value) || isArray(value);
        
        const hevnoType = isObject(value) ? value.__hevno_type__ : undefined;
        const editorInfo = hevnoType ? HEVNO_TYPE_EDITORS[hevnoType] : null;

        // --- 定义是否显示通用添加按钮的条件 ---
        const showAddButton = isObject(value) && !isArray(value) && !hevnoType;

        return (
          <React.Fragment key={currentPath}>
            <ListItem
              disablePadding
              secondaryAction={
                <Box sx={{ display: 'flex', alignItems: 'center' }}>
                  {showAddButton && (
                    <IconButton edge="end" onClick={() => onAdd(currentPath, Object.keys(value))} title="Add item to this object">
                        <AddCircleOutlineIcon fontSize="small" />
                    </IconButton>
                  )}
                  <IconButton edge="end" onClick={() => onEdit(currentPath, value)} title={editorInfo ? editorInfo.tooltip : 'Edit value'}>
                    <EditIcon />
                  </IconButton>
                </Box>
              }
            >
              <ListItemIcon sx={{minWidth: '40px'}}>
                {editorInfo ? editorInfo.icon : (isExpandable ? <FolderIcon /> : <DescriptionIcon />)}
              </ListItemIcon>
              <ListItemText 
                primary={key} 
                secondary={
                    editorInfo 
                    ? `Type: ${hevnoType.split('/')[1]}`
                    : !isExpandable 
                        ? (typeof value === 'string' ? value.slice(0, 50) + (value.length > 50 ? '...' : '') : JSON.stringify(value)) 
                        : `${isArray(value) ? 'Array' : 'Object'} (${Object.keys(value).length} items)`
                }
                onClick={isExpandable && !editorInfo ? () => toggleExpand(currentPath) : undefined}
                sx={{ cursor: (isExpandable && !editorInfo) ? 'pointer' : 'default' }}
              />
              {isExpandable && !editorInfo && (
                <IconButton size="small" onClick={() => toggleExpand(currentPath)}>
                  {expanded[currentPath] ? <ExpandMoreIcon /> : <ChevronRightIcon />}
                </IconButton>
              )}
            </ListItem>
            {isExpandable && !editorInfo && (
              <Collapse in={expanded[currentPath]} timeout="auto" unmountOnExit>
                <DataTree data={value} path={currentPath} onEdit={onEdit} onAdd={onAdd} />
              </Collapse>
            )}
          </React.Fragment>
        );
      })}
    </List>
  );
}
```

# Directory: plugins/sandbox_explorer

### vite.config.js
```
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
  plugins: [react()],
  build: {
    lib: {
      entry: resolve(__dirname, 'src/SandboxExplorerPage.jsx'),
      name: 'HevnoSandboxExplorer',
      fileName: 'main',
      formats: ['es'],
    },
    rollupOptions: {
      external: ['react', 'react-dom', '@mui/material', '@mui/icons-material'],
      output: {
        globals: {
          react: 'React',
          'react-dom': 'ReactDOM'
        }
      }
    },
    outDir: 'dist',
    emptyOutDir: true,
  },
});
```

### package.json
```
{
  "name": "hevno-plugin-sandbox-explorer",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "vite build"
  },
  "peerDependencies": {
    "react": ">=18.0.0",
    "@mui/material": ">=5.0.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "latest",
    "vite": "latest"
  }
}
```

### manifest.json
```
{
    "id": "sandbox_explorer",
    "name": "Sandbox Explorer",
    "version": "1.0.0",
    "description": "Provides a UI to browse, create, and manage sandboxes.",
    "author": "Hevno Team",
    "frontend": {
        "type": "page-component",
        "priority": 100,
        "entryPoint": "dist/main.js",
        "srcEntryPoint": "src/SandboxExplorerPage.jsx",
        "contributions": {
            "pageComponents": [
                {
                    "id": "sandbox_explorer.main_view",
                    "componentExportName": "SandboxExplorerPage",
                    "menu": {
                        "path": "/sandboxes",
                        "title": "沙盒列表",
                        "icon": "DashboardCustomizeRounded"
                    }
                }
            ]
        }
    }
}
```

### src/SandboxExplorerPage.jsx
```
// plugins/sandbox_explorer/src/SandboxExplorerPage.jsx
import React, { useState, useEffect, useCallback, useRef } from 'react';
import { Box, Grid, Typography, CircularProgress, Button } from '@mui/material';

import { SandboxCard } from './components/SandboxCard';
import { AddSandboxDialog } from './components/AddSandboxDialog';
import { AddSandboxCard } from './components/AddSandboxCard';
import { useLayout } from '../../core_layout/src/context/LayoutContext';

// --- API 调用函数 ---

const fetchSandboxes = async () => {
  const response = await fetch('/api/sandboxes');
  if (!response.ok) {
    const errorText = await response.text();
    throw new Error(`Failed to fetch sandboxes: ${errorText}`);
  }
  return response.json();
};

const importSandboxFromPng = async (file) => {
  const formData = new FormData();
  formData.append('file', file);
  const response = await fetch('/api/sandboxes:import', {
    method: 'POST',
    body: formData,
  });
  if (!response.ok) {
    const errData = await response.json().catch(() => ({ detail: "Unknown import error" }));
    throw new Error(errData.detail || `Server error: ${response.status}`);
  }
  return response.json();
};

const importSandboxFromJson = async (file) => {
  const formData = new FormData();
  formData.append('file', file);
  const response = await fetch('/api/sandboxes/import/json', {
    method: 'POST',
    body: formData,
  });
  if (!response.ok) {
    const errData = await response.json().catch(() => ({ detail: "Unknown import error" }));
    throw new Error(errData.detail || `Server error: ${response.status}`);
  }
  return response.json();
}

const createEmptySandbox = async (name) => {
  const response = await fetch('/api/sandboxes', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ name }),
  });
  if (!response.ok) {
    const errData = await response.json().catch(() => ({ detail: "Unknown create error" }));
    throw new Error(errData.detail || `Server error: ${response.status}`);
  }
  return response.json();
};

// --- [新增] 上传沙盒图标的API函数 ---
const uploadSandboxIcon = async (sandboxId, file) => {
  const formData = new FormData();
  formData.append('file', file);
  const response = await fetch(`/api/sandboxes/${sandboxId}/icon`, {
    method: 'POST',
    body: formData,
  });
  if (!response.ok) {
    const errData = await response.json().catch(() => ({ detail: "封面上传失败" }));
    throw new Error(errData.detail || `Server error: ${response.status}`);
  }
  return response.json();
};


const triggerDownload = async (url, filename) => {
    const response = await fetch(url);
    if (!response.ok) {
        throw new Error(`Failed to download file: ${response.statusText}`);
    }
    const blob = await response.blob();
    const link = document.createElement('a');
    link.href = window.URL.createObjectURL(blob);
    link.download = filename;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    window.URL.revokeObjectURL(link.href);
}

const deleteSandbox = async (sandboxId) => {
  const response = await fetch(`/api/sandboxes/${sandboxId}`, {
    method: 'DELETE'
  });
  if (!response.ok) {
    const errorText = await response.text();
    throw new Error(`Failed to delete sandbox: ${errorText}`);
  }
};


// --- 主页面组件 ---

export function SandboxExplorerPage({ services }) {
  const [sandboxes, setSandboxes] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState('');
  const [isAddDialogOpen, setAddDialogOpen] = useState(false);
  const { setActivePageId, setCurrentSandboxId } = useLayout();
  
  const hookManager = services.get('hookManager');

  const loadData = useCallback(async () => {
    if (sandboxes.length === 0) {
      setLoading(true);
    }
    setError('');
    try {
      const data = await fetchSandboxes();
      setSandboxes(data);
    } catch (e) {
      setError(e.message);
      console.error(e);
    } finally {
      setLoading(false);
    }
  }, [sandboxes.length]);

  useEffect(() => {
    loadData();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  const handleImport = async (file) => {
    if (file.type === 'application/json' || file.name.endsWith('.json')) {
      await importSandboxFromJson(file);
    } else if (file.type === 'image/png') {
      await importSandboxFromPng(file);
    } else {
      throw new Error('Unsupported file type.');
    }
    await loadData();
  };

  const handleCreateEmpty = async (name) => {
    await createEmptySandbox(name);
    await loadData();
  };

  const handleDelete = async (sandboxId) => {
    if (window.confirm("你确定要删除这个沙盒吗？此操作不可撤销。")) {
      try {
        await deleteSandbox(sandboxId);
        await loadData();
      } catch (e) {
          alert(`删除沙盒时出错: ${e.message}`);
          console.error(e);
      }
    }
  };
  
  const handleExport = async (sandboxId, sandboxName, format) => {
    const filename = `${sandboxName.replace(/\s+/g, '_')}_${sandboxId.substring(0,8)}.${format}`;
    const url = format === 'json' 
      ? `/api/sandboxes/${sandboxId}/export/json` 
      : `/api/sandboxes/${sandboxId}/export`;
    try {
      await triggerDownload(url, filename);
    } catch (e) {
      alert(`导出沙盒时出错: ${e.message}`);
      console.error(e);
    }
  };

  const handleSelect = (sandboxId) => {
    hookManager.trigger('sandbox.selected', { sandboxId });
    alert(`已选择沙盒 ${sandboxId}！ (钩子已触发)`);
  };

  const handleEdit = (sandboxId) => {
    setCurrentSandboxId(sandboxId);
    setActivePageId('sandbox_editor.main_view');
  };

  const handleRun = (sandboxId) => {
    setCurrentSandboxId(sandboxId);
    setActivePageId('runner_ui.main_view');
  };
  
  // ---  处理图标上传的函数 ---
  const handleUploadIcon = async (sandboxId, file) => {
    try {
        // 调用新创建的API函数
        await uploadSandboxIcon(sandboxId, file);
        // 上传成功后，重新加载所有沙盒数据
        // 这将获取到最新的 `icon_updated_at` 时间戳，从而使图片URL失效并强制浏览器重新加载新封面
        await loadData();
    } catch (e) {
        alert(`上传封面失败: ${e.message}`);
        console.error(e);
    }
  };


  if (loading) {
    return <Box sx={{ display: 'flex', justifyContent: 'center', alignItems: 'center', height: '100%' }}><CircularProgress /></Box>;
  }

  if (error) {
    return (
        <Box sx={{p: 4, textAlign: 'center'}}>
            <Typography variant="h6" color="error">加载沙盒失败</Typography>
            <Typography color="text.secondary">{error}</Typography>
            <Button variant="outlined" sx={{mt: 2}} onClick={loadData}>重试</Button>
        </Box>
    );
  }

  return (
    <Box sx={{ p: 3, height: '100%', overflowY: 'auto' }}>
      <Typography variant="h4" gutterBottom>沙盒</Typography>
      
      <Grid container spacing={3}>
        <Grid >
          <AddSandboxCard onClick={() => setAddDialogOpen(true)} />
        </Grid>

        {sandboxes.map((sandbox) => (
          <Grid key={sandbox.id} >
            <SandboxCard 
                sandbox={sandbox}
                onSelect={handleSelect}
                onEdit={() => handleEdit(sandbox.id)}
                onRun={() => handleRun(sandbox.id)}
                onDelete={handleDelete}
                onExportPng={() => handleExport(sandbox.id, sandbox.name, 'png')}
                onExportJson={() => handleExport(sandbox.id, sandbox.name, 'json')}
                onUploadIcon={handleUploadIcon}
            />
          </Grid>
        ))}
      </Grid>
      
      <AddSandboxDialog
        open={isAddDialogOpen}
        onClose={() => setAddDialogOpen(false)}
        onImport={handleImport}
        onCreateEmpty={handleCreateEmpty}
      />
    </Box>
  );
}

export default SandboxExplorerPage;
```

### src/components/SandboxCard.jsx
```
// plugins/sandbox_explorer/src/components/SandboxCard.jsx
import React, { useRef } from 'react';
import { Card, CardActionArea, CardMedia, CardContent, Typography, Box, IconButton, Menu, MenuItem, ListItemIcon, ListItemText } from '@mui/material';
import MoreVertIcon from '@mui/icons-material/MoreVert';
import EditIcon from '@mui/icons-material/Edit';
import PlayArrowIcon from '@mui/icons-material/PlayArrow';
import DeleteIcon from '@mui/icons-material/Delete';
import ImageIcon from '@mui/icons-material/Image';
import DataObjectIcon from '@mui/icons-material/DataObject';
// --- [新增] 导入新图标 ---
import PhotoCameraIcon from '@mui/icons-material/PhotoCamera';

const placeholderImage = 'data:image/svg+xml;charset=UTF-8,%3Csvg%20width%3D%22286%22%20height%3D%22180%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%20viewBox%3D%220%200%20286%20180%22%20preserveAspectRatio%3D%22none%22%3E%3Cdefs%3E%3Cstyle%20type%3D%22text%2Fcss%22%3E%23holder_158bd1d6d70%20text%20%7B%20fill%3A%23AAAAAA%3Bfont-weight%3Anormal%3Bfont-family%3AHelvetica%2C%20monospace%3Bfont-size%3A14pt%20%7D%20%3C%2Fstyle%3E%3C%2Fdefs%3E%3Cg%20id%3D%22holder_158bd1d6d70%22%3E%3Crect%20width%3D%22286%22%20height%3D%22180%22%20fill%3D%22%23EEEEEE%22%3E%3C%2Frect%3E%3Cg%3E%3Ctext%20x%3D%22107.19140625%22%20y%3D%2296.3%22%3ENo%20Image%3C%2Ftext%3E%3C%2Fg%3E%3C%2Fg%3E%3C%2Fsvg%3E';

// --- [修改] 接收 onUploadIcon prop ---
export function SandboxCard({ sandbox, onEdit, onRun, onDelete, onSelect, onExportPng, onExportJson, onUploadIcon }) {
  const [anchorEl, setAnchorEl] = React.useState(null);
  const open = Boolean(anchorEl);
  // --- [新增] 为隐藏的文件输入框创建一个引用 ---
  const fileInputRef = useRef(null);

  const handleMenuClick = (event) => {
    event.stopPropagation();
    setAnchorEl(event.currentTarget);
  };
  const handleMenuClose = () => setAnchorEl(null);

  const handleAction = (action) => {
    handleMenuClose();
    action();
  };

  // --- [新增] 当“更换封面”菜单项被点击时，触发隐藏的文件输入框 ---
  const handleUploadClick = () => {
    fileInputRef.current?.click();
    handleMenuClose();
  };

  // --- [新增] 当用户选择文件后，此函数被调用 ---
  const handleFileSelect = async (event) => {
    const file = event.target.files?.[0];
    if (file) {
      // 调用从父组件传入的处理器
      await onUploadIcon(sandbox.id, file);
    }
    // 重置输入框的值，以确保即使用户再次选择相同的文件也能触发 onChange 事件
    if (event.target) {
      event.target.value = null;
    }
  };
  
  const iconUrl = sandbox.has_custom_icon
    ? `/api/sandboxes/${sandbox.id}/icon?v=${new Date(sandbox.icon_updated_at || sandbox.created_at).getTime()}`
    : placeholderImage;

  return (
    <Card sx={{ display: 'flex', flexDirection: 'column', height: '100%' }}>
      <input
        type="file"
        ref={fileInputRef}
        onChange={handleFileSelect}
        accept="image/png"
        style={{ display: 'none' }}
      />
      
      <CardActionArea onClick={() => onSelect(sandbox.id)}>
        <CardMedia
          component="img"
          height="140"
          image={iconUrl}
          alt={`Cover for ${sandbox.name}`}
          onError={(e) => { e.target.onerror = null; e.target.src = placeholderImage; }}
        />
        <CardContent sx={{ flexGrow: 1 }}>
          <Typography gutterBottom variant="h5" component="div" noWrap>
            {sandbox.name}
          </Typography>
          <Typography variant="body2" color="text.secondary">
            创建于: {new Date(sandbox.created_at).toLocaleDateString()}
          </Typography>
        </CardContent>
      </CardActionArea>
      <Box sx={{ p: 1, display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
        <Box>
            <IconButton size="small" onClick={() => onEdit(sandbox.id)} title="编辑沙盒">
                <EditIcon />
            </IconButton>
            <IconButton size="small" onClick={() => onRun(sandbox.id)} title="运行沙盒">
                <PlayArrowIcon />
            </IconButton>
        </Box>
        <IconButton size="small" onClick={handleMenuClick} title="更多选项">
            <MoreVertIcon />
        </IconButton>
        <Menu anchorEl={anchorEl} open={open} onClose={handleMenuClose}>
          <MenuItem onClick={handleUploadClick}>
            <ListItemIcon><PhotoCameraIcon fontSize="small" /></ListItemIcon>
            <ListItemText>更换封面</ListItemText>
          </MenuItem>

          <MenuItem onClick={() => handleAction(onExportPng)}>
            <ListItemIcon><ImageIcon fontSize="small" /></ListItemIcon>
            <ListItemText>导出为PNG</ListItemText>
          </MenuItem>
          <MenuItem onClick={() => handleAction(onExportJson)}>
            <ListItemIcon><DataObjectIcon fontSize="small" /></ListItemIcon>
            <ListItemText>导出为JSON</ListItemText>
          </MenuItem>
          <MenuItem onClick={() => handleAction(() => onDelete(sandbox.id))} sx={{color: 'error.main'}}>
            <ListItemIcon><DeleteIcon fontSize="small" color="error" /></ListItemIcon>
            <ListItemText>删除</ListItemText>
          </MenuItem>
        </Menu>
      </Box>
    </Card>
  );
}
```

### src/components/AddSandboxCard.jsx
```
// plugins/sandbox_explorer/src/components/AddSandboxCard.jsx

import React from 'react';
import { Card, CardActionArea, Box, Typography } from '@mui/material';
import AddCircleOutlineRoundedIcon from '@mui/icons-material/AddCircleOutlineRounded';

export function AddSandboxCard({ onClick }) {
  return (
    <Card 
      sx={{ 
        height: '282px', 
        width: '224px',
        display: 'flex',
        // 使用虚线边框来与实体卡片区分
        border: (theme) => `2px dashed ${theme.palette.divider}`,
        // 移除阴影，让它看起来更像一个占位符
        boxShadow: 'none',
      }}
    >
      <CardActionArea
        onClick={onClick}
        sx={{
          display: 'flex',
          flexDirection: 'column',
          justifyContent: 'center',
          alignItems: 'center',
          height: '100%',
          p: 2,
          transition: 'background-color 0.2s',
          '&:hover': {
            backgroundColor: (theme) => theme.palette.action.hover,
          }
        }}
      >
        <AddCircleOutlineRoundedIcon sx={{ fontSize: 48, color: 'text.secondary' }} />
        <Typography variant="h6" color="text.secondary" sx={{ mt: 1 }}>
         添加沙盒
        </Typography>
      </CardActionArea>
    </Card>
  );
}
```

### src/components/AddSandboxDialog.jsx
```
// plugins/sandbox_explorer/src/components/AddSandboxDialog.jsx
import React, { useState } from 'react';
import { Button, Dialog, DialogActions, DialogContent, DialogTitle, CircularProgress, Box, Typography, Tabs, Tab, TextField } from '@mui/material';
import { styled } from '@mui/material/styles';

const VisuallyHiddenInput = styled('input')({
  clip: 'rect(0 0 0 0)',
  clipPath: 'inset(50%)',
  height: 1,
  overflow: 'hidden',
  position: 'absolute',
  bottom: 0,
  left: 0,
  whiteSpace: 'nowrap',
  width: 1,
});

function TabPanel(props) {
  const { children, value, index, ...other } = props;
  return (
    <div role="tabpanel" hidden={value !== index} id={`simple-tabpanel-${index}`} aria-labelledby={`simple-tab-${index}`} {...other}>
      {value === index && (
        <Box sx={{ p: 3 }}>
          {children}
        </Box>
      )}
    </div>
  );
}

export function AddSandboxDialog({ open, onClose, onCreateEmpty, onImport }) {
  const [activeTab, setActiveTab] = useState(0);
  
  // State for Import Tab
  const [file, setFile] = useState(null);
  const [importLoading, setImportLoading] = useState(false);
  const [importError, setImportError] = useState('');

  // State for Create Empty Tab
  const [name, setName] = useState('');
  const [createLoading, setCreateLoading] = useState(false);
  const [createError, setCreateError] = useState('');

  const handleTabChange = (event, newValue) => {
    setActiveTab(newValue);
  };
  
  const handleFileChange = (event) => {
    const selectedFile = event.target.files?.[0];
    if (selectedFile && (selectedFile.type === 'image/png' || selectedFile.type === 'application/json' || selectedFile.name.endsWith('.json'))) {
      setFile(selectedFile);
      setImportError('');
    } else {
      setFile(null);
      setImportError('Please select a valid PNG or JSON file.');
    }
  };

  const handleImport = async () => {
    if (!file) {
      setImportError('A PNG or JSON file is required.');
      return;
    }
    setImportLoading(true);
    setImportError('');
    try {
      await onImport(file);
      handleClose();
    } catch (e) {
      setImportError(e.message || 'Failed to import sandbox.');
    } finally {
      setImportLoading(false);
    }
  };

  const handleCreateEmpty = async () => {
      if (!name.trim()) {
          setCreateError('Sandbox name is required.');
          return;
      }
      setCreateLoading(true);
      setCreateError('');
      try {
          await onCreateEmpty(name.trim());
          handleClose();
      } catch (e) {
          setCreateError(e.message || 'Failed to create sandbox.');
      } finally {
          setCreateLoading(false);
      }
  };
  
  const handleClose = () => {
      if (importLoading || createLoading) return;
      // Reset all states
      setFile(null);
      setImportError('');
      setName('');
      setCreateError('');
      setActiveTab(0);
      const input = document.querySelector('input[type="file"]');
      if (input) {
          input.value = '';
      }
      onClose();
  };

  return (
    <Dialog open={open} onClose={handleClose} fullWidth maxWidth="sm">
      <DialogTitle>添加新沙盒</DialogTitle>
      <Box sx={{ borderBottom: 1, borderColor: 'divider' }}>
        <Tabs value={activeTab} onChange={handleTabChange} aria-label="add sandbox options" centered>
          <Tab label="创建空沙盒" />
          <Tab label="从文件导入" />
        </Tabs>
      </Box>

      {/* Create Empty Sandbox Tab */}
      <TabPanel value={activeTab} index={0}>
        <DialogContent sx={{p:0}}>
            <Typography>为你的新沙盒世界命名。</Typography>
            <TextField
                autoFocus
                margin="dense"
                id="name"
                label="沙盒名称"
                type="text"
                fullWidth
                variant="standard"
                value={name}
                onChange={(e) => setName(e.target.value)}
                onKeyPress={(e) => e.key === 'Enter' && handleCreateEmpty()}
            />
            {createError && <Typography color="error" sx={{mt: 2}}>{createError}</Typography>}
        </DialogContent>
        <DialogActions>
            <Button onClick={handleClose} disabled={createLoading}>取消</Button>
            <Button onClick={handleCreateEmpty} variant="contained" disabled={!name.trim() || createLoading}>
            {createLoading ? <CircularProgress size={24} /> : '创建'}
            </Button>
        </DialogActions>
      </TabPanel>
      
      {/* Import from File Tab */}
      <TabPanel value={activeTab} index={1}>
        <DialogContent sx={{p:0}}>
            <Box sx={{ textAlign: 'center' }}>
                <Typography>从一个 `.png` 或 `.json` 文件导入一个完整的沙盒。</Typography>
                <Button component="label" role={undefined} variant="outlined" tabIndex={-1} sx={{my: 2}}>
                    选择文件
                    <VisuallyHiddenInput type="file" accept="image/png,application/json,.json" onChange={handleFileChange} />
                </Button>
                {file && <Typography>已选择: {file.name}</Typography>}
                {importError && <Typography color="error" sx={{mt: 1}}>{importError}</Typography>}
            </Box>
        </DialogContent>
        <DialogActions>
            <Button onClick={handleClose} disabled={importLoading}>取消</Button>
            <Button onClick={handleImport} variant="contained" disabled={!file || importLoading}>
            {importLoading ? <CircularProgress size={24} /> : '导入'}
            </Button>
        </DialogActions>
      </TabPanel>
    </Dialog>
  );
}
```

# Directory: plugins/core_runner_ui

### vite.config.js
```
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
  plugins: [react()],
  build: {
    lib: {
      entry: resolve(__dirname, 'src/RunnerPage.jsx'),
      name: 'HevnoCoreRunnerUI',
      fileName: 'main',
      formats: ['es'],
    },
    rollupOptions: {
      // 确保外部化处理那些你不想打包进库的依赖
      external: ['react', 'react-dom', '@mui/material', '@mui/icons-material'],
      output: {
        globals: {
          react: 'React',
          'react-dom': 'ReactDOM'
        }
      }
    },
    outDir: 'dist',
    emptyOutDir: true,
  },
});
```

### package.json
```
{
  "name": "hevno-plugin-core-runner-ui",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "vite build"
  },
  "dependencies": {
    "react-markdown": "^9.0.1",
    "remark-gfm": "^4.0.0"
  },
  "peerDependencies": {
    "react": ">=18.0.0",
    "@mui/material": ">=5.0.0",
    "@mui/icons-material": ">=5.0.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "latest",
    "vite": "latest"
  }
}
```

### manifest.json
```
{
    "id": "core_runner_ui",
    "name": "Sandbox Runner UI",
    "version": "1.0.0",
    "description": "Provides the primary user interface for interacting with a running sandbox.",
    "author": "Hevno Team",
    "frontend": {
        "type": "page-component",
        "priority": 110,
        "entryPoint": "dist/main.js",
        "srcEntryPoint": "src/RunnerPage.jsx",
        "contributions": {
            "pageComponents": [
                {
                    "id": "runner_ui.main_view",
                    "componentExportName": "RunnerPage"
                }
            ]
        }
    }
}
```

### src/RunnerPage.jsx
```
// plugins/core_runner_ui/src/RunnerPage.jsx
import React, { useState, useEffect, useCallback, useMemo, useRef } from 'react';
import { 
    Box, Typography, CircularProgress, Alert, Paper, IconButton, Collapse, CssBaseline,
    AppBar, Toolbar, Tooltip, useTheme, useMediaQuery
} from '@mui/material';
import { useLayout } from '../../core_layout/src/context/LayoutContext';
import { ConversationStream } from './components/ConversationStream';
import { UserInputBar } from './components/UserInputBar';
import { SnapshotHistoryDrawer } from './components/SnapshotHistoryDrawer';
import { getSandboxDetails, mutate, step, getHistory, revert, deleteSnapshot, resetHistory } from './api';
import BugReportIcon from '@mui/icons-material/BugReport';
import MenuIcon from '@mui/icons-material/Menu';
import ArrowBackIcon from '@mui/icons-material/ArrowBack';
import AddCommentIcon from '@mui/icons-material/AddComment';

const DRAWER_WIDTH = 320;

export function RunnerPage() {
    const { currentSandboxId, setActivePageId, setCurrentSandboxId } = useLayout();
    const theme = useTheme();
    const isLargeScreen = useMediaQuery(theme.breakpoints.up('md'));

    const [sandboxDetails, setSandboxDetails] = useState(null);
    const [snapshotHistory, setSnapshotHistory] = useState([]);
    const [headSnapshotId, setHeadSnapshotId] = useState(null);
    const [isLoading, setIsLoading] = useState(false);
    const [error, setError] = useState('');
    const [diagnostics, setDiagnostics] = useState(null);
    const [showDiagnostics, setShowDiagnostics] = useState(false);
    const [isHistoryDrawerOpen, setHistoryDrawerOpen] = useState(isLargeScreen);
    const [diagnosticsHeight, setDiagnosticsHeight] = useState(200);
    const resizeRef = useRef(null);

    const [optimisticMessage, setOptimisticMessage] = useState(null);

    const handleResizeMouseDown = useCallback((e) => {
        e.preventDefault();
        resizeRef.current = {
            initialHeight: diagnosticsHeight,
            initialY: e.clientY,
        };
        document.addEventListener('mousemove', handleResizeMouseMove);
        document.addEventListener('mouseup', handleResizeMouseUp);
    }, [diagnosticsHeight]);

    const handleResizeMouseMove = useCallback((e) => {
        if (!resizeRef.current) return;
        const { initialHeight, initialY } = resizeRef.current;
        const dy = e.clientY - initialY;
        const newHeight = initialHeight - dy;
        const clampedHeight = Math.max(50, Math.min(newHeight, 600));
        setDiagnosticsHeight(clampedHeight);
    }, []);

    const handleResizeMouseUp = useCallback(() => {
        resizeRef.current = null;
        document.removeEventListener('mousemove', handleResizeMouseMove);
        document.removeEventListener('mouseup', handleResizeMouseUp);
    }, []);

    useEffect(() => {
        setHistoryDrawerOpen(isLargeScreen);
    }, [isLargeScreen]);
    
    const messages = useMemo(() => {
        const allMessages = [];
        const headPath = new Map();
        let currentId = headSnapshotId;
        while(currentId) {
            const snapshot = snapshotHistory.find(s => s.id === currentId);
            if(snapshot) {
                headPath.set(snapshot.id, snapshot);
                currentId = snapshot.parent_snapshot_id;
            } else {
                break;
            }
        }
        for (const snapshot of [...snapshotHistory].reverse()) {
            if (headPath.has(snapshot.id)) {
                const entries = snapshot.moment?.memoria?.chat_history?.entries || [];
                for (const entry of entries) {
                    allMessages.push({ ...entry, snapshot_id: snapshot.id, parent_snapshot_id: snapshot.parent_snapshot_id, triggering_input: snapshot.triggering_input });
                }
            }
        }
        let uniqueMessages = Array.from(new Map(allMessages.map(item => [item.id, item])).values());
        uniqueMessages.sort((a, b) => a.sequence_id - b.sequence_id);

        if (optimisticMessage) {
            uniqueMessages.push(optimisticMessage);
        }

        return uniqueMessages;
    }, [snapshotHistory, headSnapshotId, optimisticMessage]);


    const loadData = useCallback(async (showLoading = true) => {
        if (!currentSandboxId) return;
        if (showLoading) setIsLoading(true);
        setError('');
        try {
            const [history, details] = await Promise.all([
                getHistory(currentSandboxId),
                getSandboxDetails(currentSandboxId)
            ]);
            
            if (!details || !details.head_snapshot_id) {
                 throw new Error("Retrieved sandbox details are incomplete.");
            }
            
            setSnapshotHistory(history);
            setSandboxDetails(details);
            setHeadSnapshotId(details.head_snapshot_id);

        } catch (e) {
            setError(`Failed to load sandbox data: ${e.message}`);
        } finally {
            if (showLoading) setIsLoading(false);
        }
    }, [currentSandboxId]);

    useEffect(() => {
        if (currentSandboxId) {
            loadData();
        } else {
            setSandboxDetails(null);
            setSnapshotHistory([]);
            setHeadSnapshotId(null);
            setError('');
        }
    }, [currentSandboxId, loadData]);
    
    const handleUserSubmit = async (inputText) => {
        if (!currentSandboxId || isLoading) return;
        
        const tempMsg = {
            id: `optimistic_${Date.now()}`,
            content: inputText,
            level: 'user',
            sequence_id: (messages.length > 0 ? messages[messages.length - 1].sequence_id : 0) + 1,
        };
        setOptimisticMessage(tempMsg);
        
        setIsLoading(true);
        setError('');
        setDiagnostics(null);
        
        try {
            await mutate(currentSandboxId, [{ type: 'UPSERT', path: 'moment/_user_input', value: inputText }]);
            const stepResponse = await step(currentSandboxId, {});
            if (stepResponse.status === 'ERROR') throw new Error(stepResponse.error_message);
            if (stepResponse.diagnostics) setDiagnostics(stepResponse.diagnostics);
            await loadData(false); 
        } catch (e) {
            setError(e.message);
            await loadData(false);
        } finally {
            setOptimisticMessage(null);
            setIsLoading(false);
        }
    };
    
    const handleRegenerate = async (message) => {
        if (!currentSandboxId || isLoading || !message.parent_snapshot_id) return;
        setIsLoading(true);
        setError('');
        setDiagnostics(null);
        try {
            await revert(currentSandboxId, message.parent_snapshot_id);
            const stepResponse = await step(currentSandboxId, message.triggering_input || {});
            if (stepResponse.status === 'ERROR') throw new Error(stepResponse.error_message);
            if (stepResponse.diagnostics) setDiagnostics(stepResponse.diagnostics);
            await loadData(false);
        } catch(e) {
            setError(e.message);
        } finally {
            setIsLoading(false);
        }
    };
    
    const handleEditSubmit = async (message, newContent) => {
        if (!currentSandboxId || isLoading || !message.parent_snapshot_id) return;
        setIsLoading(true);
        setError('');
        setDiagnostics(null);
        try {
            await revert(currentSandboxId, message.parent_snapshot_id);
            await mutate(currentSandboxId, [{ type: 'UPSERT', path: 'moment/_user_input', value: newContent }]);
            const stepResponse = await step(currentSandboxId, {});
            if (stepResponse.status === 'ERROR') throw new Error(stepResponse.error_message);
            if (stepResponse.diagnostics) setDiagnostics(stepResponse.diagnostics);
            await loadData(false);
        } catch (e) {
            setError(e.message);
        } finally {
            setIsLoading(false);
        }
    };

    const handleRevert = async (snapshotId) => {
        if (!currentSandboxId || isLoading) return;
        setIsLoading(true);
        setError('');
        try {
            await revert(currentSandboxId, snapshotId);
            await loadData(false);
        } catch (e) {
            setError(`Failed to revert: ${e.message}`);
        } finally {
            setIsLoading(false);
        }
    };
    
    const handleDeleteSnapshot = async (snapshotId) => {
        if (!currentSandboxId || isLoading || snapshotId === headSnapshotId) return;
        if (window.confirm("确定要永久删除这个历史记录点吗？")) {
            setIsLoading(true);
            setError('');
            try {
                await deleteSnapshot(currentSandboxId, snapshotId);
                await loadData(false);
            } catch (e) {
                setError(`删除失败: ${e.message}`);
            } finally {
                setIsLoading(false);
            }
        }
    };

    const handleResetHistory = async () => {
        if (!currentSandboxId || isLoading) return;
        if (window.confirm("确定要开启一个新的会话吗？当前会话将成为历史记录。")) {
            setIsLoading(true);
            setError('');
            try {
                await resetHistory(currentSandboxId);
                await loadData(false);
            } catch (e) {
                setError(`开启新会话失败: ${e.message}`);
            } finally {
                setIsLoading(false);
            }
        }
    };


    const handleGoBackToExplorer = () => {
        setCurrentSandboxId(null);
        setActivePageId('sandbox_explorer.main_view');
    };

    if (!currentSandboxId) {
        return (
            <Box sx={{ p: 4, textAlign: 'center', display: 'flex', flexDirection: 'column', justifyContent: 'center', height: '100%' }}>
                <Typography variant="h5">开始交互</Typography>
                <Typography color="text.secondary">请从 "沙盒列表" 页面选择一个沙盒以开始。</Typography>
            </Box>
        );
    }
    
    return (
        <Box sx={{ display: 'flex', height: '100vh', width: '100vw' }}>
            <CssBaseline />
            <SnapshotHistoryDrawer 
                history={snapshotHistory} 
                headSnapshotId={headSnapshotId} 
                onRevert={handleRevert}
                onDelete={handleDeleteSnapshot}
                isLoading={isLoading}
                open={isHistoryDrawerOpen}
                onClose={() => setHistoryDrawerOpen(false)}
                variant={isLargeScreen ? 'persistent' : 'temporary'}
                width={DRAWER_WIDTH}
            />

            <Box
                component="main"
                sx={{
                    flexGrow: 1,
                    display: 'flex',
                    flexDirection: 'column',
                    height: '100%',
                    transition: theme.transitions.create('margin', {
                        easing: theme.transitions.easing.sharp,
                        duration: theme.transitions.duration.leavingScreen,
                    }),
                    marginLeft: `-${DRAWER_WIDTH}px`,
                    ...(isHistoryDrawerOpen && {
                        transition: theme.transitions.create('margin', {
                            easing: theme.transitions.easing.easeOut,
                            duration: theme.transitions.duration.enteringScreen,
                        }),
                        marginLeft: 0,
                    }),
                }}
            >
                <AppBar position="static" color="default" sx={{ boxShadow: 'none', borderBottom: 1, borderColor: 'divider' }}>
                    <Toolbar>
                        <Tooltip title="切换历史记录">
                            <IconButton color="inherit" aria-label="open drawer" onClick={() => setHistoryDrawerOpen(!isHistoryDrawerOpen)} edge="start" sx={{ mr: 2 }}>
                                <MenuIcon />
                            </IconButton>
                        </Tooltip>
                        <Tooltip title="返回沙盒列表">
                             <IconButton color="inherit" onClick={handleGoBackToExplorer} edge="start">
                                <ArrowBackIcon />
                            </IconButton>
                        </Tooltip>
                        <Typography variant="h6" noWrap component="div" sx={{ flexGrow: 1, ml: 1 }}>
                            {sandboxDetails?.name || 'Loading...'}
                        </Typography>
                        
                        <Tooltip title="新建会话">
                            <IconButton size="small" onClick={handleResetHistory} disabled={isLoading}>
                                <AddCommentIcon />
                            </IconButton>
                        </Tooltip>

                        {diagnostics && (
                            <Tooltip title="显示/隐藏诊断信息">
                                <IconButton size="small" onClick={() => setShowDiagnostics(s => !s)} sx={{ ml: 1 }}>
                                    <BugReportIcon color={showDiagnostics ? "primary" : "inherit"} />
                                </IconButton>
                            </Tooltip>
                        )}
                    </Toolbar>
                </AppBar>

                <Box sx={{ flexGrow: 1, overflowY: 'auto' }}>
                    {isLoading && messages.length === 0 && !optimisticMessage ? (
                        <Box sx={{ display: 'flex', justifyContent: 'center', p: 4 }}><CircularProgress /></Box>
                    ) : (
                        <ConversationStream 
                            messages={messages} 
                            onRegenerate={handleRegenerate} 
                            onEditSubmit={handleEditSubmit}
                        />
                    )}
                </Box>
                
                <Box sx={{ flexShrink: 0, p: { xs: 1, sm: 2 } }}>
                    {error && <Alert severity="error" sx={{ mb: 1.5 }} onClose={() => setError('')}>{error}</Alert>}
                    <Collapse in={showDiagnostics}>
                        <Paper 
                            variant="outlined" 
                            sx={{ p: 2, pt: 3, mb: 1.5, height: diagnosticsHeight, overflow: 'hidden', bgcolor: 'background.default', position: 'relative', display: 'flex', flexDirection: 'column' }}
                        >
                            <Box onMouseDown={handleResizeMouseDown} sx={{ position: 'absolute', top: 0, left: 0, right: 0, height: '10px', cursor: 'ns-resize', display: 'flex', alignItems: 'center', justifyContent: 'center', '&:hover div': { backgroundColor: theme.palette.action.active, }}}>
                                 <Box sx={{ width: '40px', height: '4px', backgroundColor: theme.palette.divider, borderRadius: '2px', transition: 'background-color 0.2s' }} />
                            </Box>
                            <Typography variant="subtitle2" sx={{ flexShrink: 0 }}>诊断信息</Typography>
                            <pre style={{ flexGrow: 1, margin: 0, whiteSpace: 'pre-wrap', wordBreak: 'break-all', fontSize: '0.8rem', overflowY: 'auto' }}>
                                {JSON.stringify(diagnostics, null, 2)}
                            </pre>
                        </Paper>
                    </Collapse>
                    {/* [修复] 恢复正确的 isLoading 逻辑 */}
                    <UserInputBar onSendMessage={handleUserSubmit} isLoading={isLoading} />
                </Box>
            </Box>
        </Box>
    );
}

export default RunnerPage;
```

### src/api.js
```
// plugins/core_runner_ui/src/api.js

const BASE_URL = '/api/sandboxes';

export async function getSandboxDetails(sandboxId) {
    const response = await fetch(`${BASE_URL}/${sandboxId}`); // 使用标准的 GET /resource/{id}
    if (!response.ok) {
        const err = await response.json().catch(() => ({ detail: `Failed to fetch details for sandbox ${sandboxId}.` }));
        throw new Error(err.detail || `HTTP Error ${response.status}`);
    }
    return response.json();
}

/**
 * 统一写入 API
 */
export async function mutate(sandboxId, mutations) {
  const response = await fetch(`${BASE_URL}/${sandboxId}/resource:mutate`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ mutations }),
  });

  if (!response.ok) {
    const err = await response.json().catch(() => ({ detail: "API mutation failed." }));
    throw new Error(err.detail || `HTTP Error ${response.status}`);
  }
  return response.json();
}

/**
 * 统一读取 API
 */
export async function query(sandboxId, paths) {
  const response = await fetch(`${BASE_URL}/${sandboxId}/resource:query`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ paths }),
  });

  if (!response.ok) {
    const err = await response.json().catch(() => ({ detail: "API query failed." }));
    throw new Error(err.detail || `HTTP Error ${response.status}`);
  }
  const data = await response.json();
  return data.results;
}


/**
 * 执行沙盒计算步骤 API
 */
export async function step(sandboxId, input) {
    const response = await fetch(`${BASE_URL}/${sandboxId}/step`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(input),
    });

    // 保持原有的错误处理
    if (!response.ok) {
        const err = await response.json().catch(() => ({ detail: "Step API call failed." }));
        return {
            status: "ERROR",
            error_message: err.detail || `HTTP Error ${response.status}`,
            sandbox: null,
        }
    }
    return response.json();
}

/**
 * 获取沙盒的完整快照历史
 */
export async function getHistory(sandboxId) {
    const response = await fetch(`${BASE_URL}/${sandboxId}/history`);
    if (!response.ok) {
        const err = await response.json().catch(() => ({ detail: "Failed to get history." }));
        throw new Error(err.detail || `HTTP Error ${response.status}`);
    }
    return response.json();
}

/**
 * 将沙盒回滚到指定的快照
 */
export async function revert(sandboxId, snapshotId) {
    const response = await fetch(`${BASE_URL}/${sandboxId}/revert`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ snapshot_id: snapshotId }),
    });

    if (!response.ok) {
        const err = await response.json().catch(() => ({ detail: "Revert operation failed." }));
        throw new Error(err.detail || `HTTP Error ${response.status}`);
    }
    return response.json();
}

/**
 * [新增] 删除指定的快照
 */
export async function deleteSnapshot(sandboxId, snapshotId) {
    const response = await fetch(`${BASE_URL}/${sandboxId}/snapshots/${snapshotId}`, {
        method: 'DELETE',
    });
    // DELETE 成功时返回 204 No Content，response.ok 会是 true
    if (!response.ok) {
        const err = await response.json().catch(() => ({ detail: "Delete snapshot operation failed." }));
        throw new Error(err.detail || `HTTP Error ${response.status}`);
    }
    // 204 没有 body，所以直接返回
}

/**
 * 重置沙盒历史，开启新会话
 */
export async function resetHistory(sandboxId) {
    const response = await fetch(`${BASE_URL}/${sandboxId}/history:reset`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({}),
    });
    if (!response.ok) {
        const err = await response.json().catch(() => ({ detail: "Reset history operation failed." }));
        throw new Error(err.detail || `HTTP Error ${response.status}`);
    }
    return response.json();
}
```

### src/components/MessageBubble.jsx
```
// plugins/core_runner_ui/src/components/MessageBubble.jsx
import React, { useState } from 'react';
import { Box, Paper, Typography, Avatar, IconButton, Tooltip, TextField, Button, CircularProgress } from '@mui/material';
import ReactMarkdown from 'react-markdown';
import remarkGfm from 'remark-gfm';
import PersonIcon from '@mui/icons-material/Person';
import SmartToyIcon from '@mui/icons-material/SmartToy';
import ReplayIcon from '@mui/icons-material/Replay'; 
import EditIcon from '@mui/icons-material/Edit'; 

export const MessageBubble = ({ message, onRegenerate, onEditSubmit }) => {
    const isUser = message.level === 'user';
    const [isHovered, setIsHovered] = useState(false);
    const [isEditing, setIsEditing] = useState(false);
    const [editedContent, setEditedContent] = useState(message.content);

    const handleEditClick = () => {
        setIsEditing(true);
    };

    const handleCancelEdit = () => {
        setIsEditing(false);
        setEditedContent(message.content); // 恢复原始内容
    };

    const handleSaveEdit = () => {
        if (editedContent.trim()) {
            onEditSubmit(message, editedContent);
            setIsEditing(false);
        }
    };
    
    const renderContent = () => {
        if (isEditing) {
            return (
                <Box sx={{ width: '100%', display: 'flex', flexDirection: 'column', gap: 1 }}>
                    <TextField
                        fullWidth
                        multiline
                        variant="standard"
                        value={editedContent}
                        onChange={(e) => setEditedContent(e.target.value)}
                        autoFocus
                    />
                    <Box sx={{ display: 'flex', justifyContent: 'flex-end', gap: 1, mt: 1 }}>
                        <Button size="small" onClick={handleCancelEdit}>取消</Button>
                        <Button size="small" variant="contained" onClick={handleSaveEdit}>保存并提交</Button>
                    </Box>
                </Box>
            );
        }

        return (
            <Typography component="div" sx={{ '& p': { my: 0 }, '& pre': { whiteSpace: 'pre-wrap', wordBreak: 'break-all' }, '& table': {borderCollapse: 'collapse'}, '& th, & td': {border: '1px solid', borderColor: 'divider', px: 1, py: 0.5} }}>
                <ReactMarkdown remarkPlugins={[remarkGfm]}>
                    {message.content}
                </ReactMarkdown>
            </Typography>
        );
    };

    return (
        <Box
            sx={{
                display: 'flex',
                justifyContent: isUser ? 'flex-end' : 'flex-start',
                mb: 2,
            }}
            onMouseEnter={() => setIsHovered(true)}
            onMouseLeave={() => setIsHovered(false)}
        >
            <Box sx={{ display: 'flex', flexDirection: isUser ? 'row-reverse' : 'row', alignItems: 'flex-start', maxWidth: '85%', position: 'relative' }}>
                <Avatar sx={{ bgcolor: isUser ? 'primary.main' : 'secondary.main', ml: isUser ? 1.5 : 0, mr: isUser ? 0 : 1.5, mt: 0.5 }}>
                    {isUser ? <PersonIcon /> : <SmartToyIcon />}
                </Avatar>
                <Paper
                    variant="elevation"
                    elevation={1}
                    sx={{
                        p: 1.5,
                        bgcolor: isUser ? 'primary.dark' : 'background.paper',
                        borderRadius: isUser ? '20px 4px 20px 20px' : '4px 20px 20px 20px',
                        position: 'relative',
                    }}
                >
                   {renderContent()}
                </Paper>
                <Box sx={{
                    display: 'flex',
                    flexDirection: isUser ? 'row-reverse' : 'row',
                    opacity: isHovered && !isEditing ? 1 : 0,
                    transition: 'opacity 0.2s',
                    position: 'absolute',
                    top: '50%',
                    transform: 'translateY(-50%)',
                    left: isUser ? 'auto' : '100%',
                    right: isUser ? '100%' : 'auto',
                    mx: 0.5,
                    bgcolor: 'background.default',
                    borderRadius: '20px',
                    p: '2px',
                    boxShadow: 1
                }}>
                    {isUser && onEditSubmit && (
                         <Tooltip title="编辑并重新生成">
                            <IconButton size="small" onClick={handleEditClick}><EditIcon sx={{fontSize: '1rem'}} /></IconButton>
                         </Tooltip>
                    )}
                    {!isUser && onRegenerate && (
                         <Tooltip title="重新生成">
                            <IconButton size="small" onClick={() => onRegenerate(message)}><ReplayIcon sx={{fontSize: '1rem'}} /></IconButton>
                         </Tooltip>
                    )}
                </Box>
            </Box>
        </Box>
    );
};
```

### src/components/UserInputBar.jsx
```
// plugins/core_runner_ui/src/components/UserInputBar.jsx
import React, { useState } from 'react';
import { TextField, IconButton, Box, CircularProgress, Paper } from '@mui/material';
import SendIcon from '@mui/icons-material/Send';

export function UserInputBar({ onSendMessage, isLoading }) {
    const [text, setText] = useState('');

    const handleSubmit = (e) => {
        e.preventDefault();
        if (text.trim() && !isLoading) {
            onSendMessage(text);
            setText('');
        }
    };

    return (
        <Paper 
            component="form" 
            onSubmit={handleSubmit} 
            sx={{ 
                p: '4px 8px', 
                display: 'flex', 
                alignItems: 'center', 
                width: '100%',
                borderRadius: '12px'
            }}
            elevation={2}
        >
            <TextField
                fullWidth
                placeholder="在此输入消息... (Shift+Enter 换行)"
                value={text}
                onChange={(e) => setText(e.target.value)}
                disabled={isLoading}
                multiline
                maxRows={8}
                variant="standard" // 使用无边框样式
                InputProps={{
                    disableUnderline: true, // 移除下划线
                }}
                onKeyDown={(e) => {
                    if (e.key === 'Enter' && !e.shiftKey) {
                        e.preventDefault();
                        handleSubmit(e);
                    }
                }}
                sx={{ ml: 1 }}
            />
            <IconButton
                type="submit"
                color="primary"
                disabled={!text.trim() || isLoading}
                sx={{ p: '10px' }}
            >
                {isLoading ? <CircularProgress size={24} color="inherit" /> : <SendIcon />}
            </IconButton>
        </Paper>
    );
}
```

### src/components/ConversationStream.jsx
```
// plugins/core_runner_ui/src/components/ConversationStream.jsx
import React from 'react';
import { Box } from '@mui/material';
import { MessageBubble } from './MessageBubble'; // [修改] 导入新组件

export const ConversationStream = ({ messages, onRegenerate, onEditSubmit }) => {
    // 过滤掉那些没有内容的临时快照消息
    const displayableMessages = messages.filter(msg => msg.content && msg.level);
    
    const conversationEndRef = React.useRef(null);
    React.useEffect(() => {
        conversationEndRef.current?.scrollIntoView({ behavior: "smooth" });
    }, [messages]);

    return (
        <Box sx={{ p: { xs: 1, sm: 2 } }}>
            {displayableMessages.map((msg, index) => (
                <MessageBubble 
                    key={msg.id || index} 
                    message={msg} 
                    onRegenerate={onRegenerate} 
                    onEditSubmit={onEditSubmit} 
                />
            ))}
            <div ref={conversationEndRef} />
        </Box>
    );
};
```

### src/components/SnapshotHistoryDrawer.jsx
```
// plugins/core_runner_ui/src/components/SnapshotHistoryDrawer.jsx
import React from 'react';
import { 
    Drawer, List, ListItem, ListItemButton, ListItemText, ListItemIcon, 
    Tooltip, Box, Typography, IconButton, Divider, Toolbar, Skeleton
} from '@mui/material';
import RadioButtonCheckedIcon from '@mui/icons-material/RadioButtonChecked';
import RadioButtonUncheckedIcon from '@mui/icons-material/RadioButtonUnchecked';
import DeleteForeverIcon from '@mui/icons-material/DeleteForever';

const getSnapshotSummary = (snapshot) => {
    const input = snapshot.triggering_input?.text;
    if (input) {
        return `输入: "${input.slice(0, 35)}${input.length > 35 ? '...' : ''}"`;
    }
    
    const entries = snapshot.moment?.memoria?.chat_history?.entries;
    if (entries && entries.length > 0) {
        const lastEntry = entries[entries.length - 1];
        if (lastEntry.level === 'model') {
            return `回应: "${lastEntry.content.slice(0, 35)}${lastEntry.content.length > 35 ? '...' : ''}"`;
        }
    }
    return '初始状态';
};

export const SnapshotHistoryDrawer = ({ history, headSnapshotId, onRevert, onDelete, isLoading, width, ...drawerProps }) => {
    
    const reversedHistory = [...history].reverse();

    const drawerContent = (
        <>
            <Toolbar>
                <Typography variant="h6" component="div">交互历史</Typography>
            </Toolbar>
            <Divider />
            {history.length === 0 && !isLoading && (
                <Box sx={{ p: 2, textAlign: 'center', color: 'text.secondary' }}>
                    <Typography>没有历史记录</Typography>
                </Box>
            )}
            {isLoading && history.length === 0 && (
                <Box sx={{p: 2}}>
                    <Skeleton variant="text" sx={{ fontSize: '1rem' }} />
                    <Skeleton variant="text" sx={{ fontSize: '1rem' }} />
                    <Skeleton variant="text" sx={{ fontSize: '1rem' }} />
                </Box>
            )}
            <List dense disablePadding sx={{ flexGrow: 1, overflowY: 'auto' }}>
                {reversedHistory.map(snapshot => {
                    const isHead = snapshot.id === headSnapshotId;
                    const summary = getSnapshotSummary(snapshot);
                    const timestamp = new Date(snapshot.created_at).toLocaleString();

                    return (
                        <ListItem
                            key={snapshot.id}
                            disablePadding
                            secondaryAction={
                                // 只在非当前 head 的快照上显示删除按钮
                                !isHead && (
                                    <Tooltip title="永久删除此记录点">
                                        <span>
                                            <IconButton edge="end" onClick={() => onDelete(snapshot.id)} disabled={isLoading} sx={{ color: 'error.light' }}>
                                                <DeleteForeverIcon />
                                            </IconButton>
                                        </span>
                                    </Tooltip>
                                )
                            }
                        >
                            <ListItemButton 
                                selected={isHead} 
                                dense
                                // 只有非当前 head 的快照才能被点击
                                onClick={!isHead ? () => onRevert(snapshot.id) : undefined} 
                                // 如果正在加载中，或者该项是当前 head，则禁用点击
                                disabled={isLoading}
                            >
                                <ListItemIcon sx={{ minWidth: 32 }}>
                                    {isHead ? <RadioButtonCheckedIcon color="primary" fontSize="small" /> : <RadioButtonUncheckedIcon fontSize="small" />}
                                </ListItemIcon>
                                <ListItemText
                                    primary={summary}
                                    secondary={timestamp}
                                    primaryTypographyProps={{ noWrap: true, variant: 'body2', fontWeight: isHead ? 'bold' : 'normal' }}
                                    secondaryTypographyProps={{ noWrap: true, variant: 'caption' }}
                                />
                            </ListItemButton>
                        </ListItem>
                    );
                })}
            </List>
        </>
    );

    return (
        <Drawer
            sx={{
                width: width,
                flexShrink: 0,
                '& .MuiDrawer-paper': {
                    width: width,
                    boxSizing: 'border-box',
                    display: 'flex',
                    flexDirection: 'column',
                },
            }}
            anchor="left"
            {...drawerProps}
        >
            {drawerContent}
        </Drawer>
    );
};
```

### hevno.json
```
{
  "plugins": {
    "core_logging": { "source": "local" },
    "core_persistence": { "source": "local" },
    "core_engine": { "source": "local" },
    "core_codex": { "source": "local" },
    "core_llm": { "source": "local" },
    "core_memoria": { "source": "local" },
    "core_api": { "source": "local" },
    "core_websocket": { "source": "local" },
    "core_remote_hooks": { "source": "local" },
    "core_layout": { "source": "local" },
    "core_diagnostics": { "source": "local" },
    "page_demo": { "source": "local" },
    "sandbox_explorer": { "source": "local" },
    "sandbox_editor": { "source": "local" },
    "core_llm_config": { "source": "local" },
    "core_runner_ui": { "source": "local" }
    }
}
```

### cli.py
```
# cli.py
import typer
import json
import shutil
from pathlib import Path

from backend.core.plugin_manager import PluginManager

app = typer.Typer(name="hevno", help="Hevno Engine Command-Line Interface")
plugin_app = typer.Typer(name="plugins", help="Manage Hevno plugins.")
app.add_typer(plugin_app)

HEVNO_JSON_PATH = Path("hevno.json")
PLUGINS_DIR = Path("plugins")

@plugin_app.command("sync")
def sync_plugins():
    """
    Synchronizes the 'plugins' directory with the 'hevno.json' manifest.
    This will fetch, update, or remove plugins to match the manifest.
    """
    typer.echo("🔌 Starting plugin synchronization...")
    if not HEVNO_JSON_PATH.exists():
        typer.secho(f"Error: '{HEVNO_JSON_PATH}' not found. Nothing to sync.", fg=typer.colors.RED)
        raise typer.Exit(code=1)

    with open(HEVNO_JSON_PATH, "r") as f:
        manifest = json.load(f)

    manager = PluginManager(plugins_dir=PLUGINS_DIR, manifest=manifest.get("plugins", {}))
    
    try:
        manager.sync()
        typer.secho("✅ Plugin synchronization complete.", fg=typer.colors.GREEN)
    except Exception as e:
        typer.secho(f"🔥 Error during synchronization: {e}", fg=typer.colors.RED)
        import traceback
        traceback.print_exc()
        raise typer.Exit(code=1)


@plugin_app.command("add")
def add_plugin(
    url: str = typer.Argument(..., help="Git repository URL (e.g., https://github.com/user/repo)."),
    name: str = typer.Option(None, "--name", "-n", help="A specific name for the plugin directory. Defaults to repo name."),
    ref: str = typer.Option("main", "--ref", "-r", help="Git branch, tag, or commit hash."),
    subdir: str = typer.Option(None, "--subdir", "-d", help="Path to the plugin within the repository.")
):
    """
    Adds a new plugin from Git to hevno.json and syncs.
    """
    if not HEVNO_JSON_PATH.exists():
        # 如果文件不存在，创建一个基础结构
        manifest_data = {"plugins": {}}
        typer.echo(f"'{HEVNO_JSON_PATH}' not found, creating a new one.")
    else:
        with open(HEVNO_JSON_PATH, "r") as f:
            manifest_data = json.load(f)
    
    plugin_name = name or Path(url.split('/')[-1]).stem
    
    plugin_config = {"source": "git", "url": url, "ref": ref}
    if subdir:
        plugin_config["subdirectory"] = subdir

    manifest_data["plugins"][plugin_name] = plugin_config

    with open(HEVNO_JSON_PATH, "w") as f:
        json.dump(manifest_data, f, indent=2)

    typer.secho(f"Added plugin '{plugin_name}' to '{HEVNO_JSON_PATH}'.", fg=typer.colors.BLUE)
    # 调用 sync 来完成安装
    sync_plugins()


@plugin_app.command("remove")
def remove_plugin(name: str = typer.Argument(..., help="The name of the plugin to remove.")):
    """
    Removes a plugin from hevno.json and syncs.
    """
    if not HEVNO_JSON_PATH.exists():
        typer.secho(f"Error: '{HEVNO_JSON_PATH}' not found.", fg=typer.colors.RED)
        raise typer.Exit(code=1)

    with open(HEVNO_JSON_PATH, "r") as f:
        manifest_data = json.load(f)

    if name not in manifest_data.get("plugins", {}):
        typer.secho(f"Warning: Plugin '{name}' not found in manifest. Nothing to do.", fg=typer.colors.YELLOW)
        return

    del manifest_data["plugins"][name]
    
    with open(HEVNO_JSON_PATH, "w") as f:
        json.dump(manifest_data, f, indent=2)

    typer.secho(f"Removed plugin '{name}' from '{HEVNO_JSON_PATH}'.", fg=typer.colors.BLUE)
    
    # 物理删除目录
    plugin_path = PLUGINS_DIR / name
    if plugin_path.exists():
        shutil.rmtree(plugin_path)
        typer.echo(f"Removed directory: {plugin_path}")
    
    sync_plugins()


if __name__ == "__main__":
    app()
```
