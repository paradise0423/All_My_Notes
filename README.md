# MAOS - 终端多智能体操作系统

MAOS（Multi-Agent Operating System）是面向终端侧的多智能体操作系统，强调在算力、内存、功耗与系统权限受限的环境中，实现可落地的任务拆解、协同执行与能力接入。系统采用“分层 + 职责解耦”的设计，兼顾端侧稳定性与端云协同的扩展性。

# 系统架构

系统采用三层结构：

- **用户交互层**：接收用户请求并触发任务流转。
- **系统智能体层**：负责意图理解、任务拆解、路由分配与执行编排。  
- **应用智能体层**：承接具体应用能力的执行与返回。  

路由与编排由系统智能体层完成：Task Agent 负责意图理解与子任务拆解，Coordinator Agent 负责基于能力描述进行任务分配与协同编排，Workforce 负责执行调度与生命周期管理，形成“思考—行动”的闭环。

统一的模型入口在 `agents/backend_model.py`，通过 `ModelFactory.create()` 创建 `BaseModelBackend` 的具体实现，再注入 `ChatAgent`。

# 核心功能

- **任务自动化**：从用户意图理解到子任务拆解与交付标准定义。
- **能力感知路由**：基于应用智能体能力描述进行任务匹配与分配。
- **协同编排与调度**：多智能体协作执行，支持依赖关系与并行处理。
- **容错与恢复**：支持重试、再规划、重分配与动态创建执行单元。
- **状态与上下文管理**：维护任务状态、依赖关系与跨任务上下文。
- **端侧优先与端云协同**：端侧负责稳定与低延迟执行，云侧承接重推理与高成本能力。

# 项目结构

```
.
├─ agents/                # 系统级与应用级 Agent
├─ tools/                 # 各类 Toolkit（工具封装）
├─ mock_data/             # 本地模拟数据（json/md）
├─ working_dir/           # 运行时输出与任务状态
├─ demo/                  # 监控/演示相关代码
├─ camel-master/          # CAMEL 框架源码
├─ docs/                  # 项目文档
└─ scripts/               # 辅助脚本
```

## 新增Agent

### 1、概述

在 AIOS 中，**App-Level Agent（应用级代理）**用于表示系统中的具体应用，例如 Notes、Contactors、Photos 等。每个 App-Level Agent 封装了与某个应用相关的能力，用于处理应用数据，向系统级 Agent 返回结果，等等。

App-Level Agent 可视为 **AIOS 多代理系统与应用之间的接口层**，系统级 Agent 通过调用 App-Level Agent，从而访问不同应用的数据与能力。

### 2、架构设计

所有 App-Level Agent 都基于 `camel-master/camel/agents/chat_agent.py` 中的 `ChatAgent` 类构建，以 `Function` 形式封装，命名方式遵循 `xxx_agent_factory()` 的方式，在函数内部完成 Agent 的配置与初始化，最终返回一个 `ChatAgent` 实例。

***Classes***

**`ChatAgent`**：Agent 核心部分，负责模型调用、工具调用与消息编排。  

***Args:***
<table style="border-collapse:collapse; width:100%; font-size:14px;">
  <thead>
    <tr>
      <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">参数</th>
      <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">类型</th>
      <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">说明</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>system_message</code></td>
      <td style="border:1px solid #999; padding:6px;">Union[<code>BaseMessage</code>, <code>str</code>], optional</td>
      <td style="border:1px solid #999; padding:6px;">Agent 的系统提示词，用于定义 Agent 的角色、能力范围和行为规则，作为对话上下文的初始消息，引导大模型如何调用工具完成任务。</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>model</code></td>
      <td style="border:1px solid #999; padding:6px;"><code>BaseModelBackend</code></td>
      <td style="border:1px solid #999; padding:6px;">Agent 使用的基座模型，通过 <code>backend_model()</code> 方法统一创建，该方法内部使用 <code>ModelFactory.create()</code> 构建模型实例。</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>tools</code></td>
      <td style="border:1px solid #999; padding:6px;">List[Union[<code>FunctionTool</code>, <code>Callable</code>]], optional</td>
      <td style="border:1px solid #999; padding:6px;">Agent 可调用的工具集合，支持 <code>FunctionTool</code> 或任意可调用对象（初始化时自动转为 <code>FunctionTool</code>）。</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>token_limit</code></td>
      <td style="border:1px solid #999; padding:6px;"><code>int</code>, optional</td>
      <td style="border:1px solid #999; padding:6px;">上下文 token 上限，默认随模型配置。</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>output_language</code></td>
      <td style="border:1px solid #999; padding:6px;"><code>str</code>, optional</td>
      <td style="border:1px solid #999; padding:6px;">强制输出语言。</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>agent_id</code></td>
      <td style="border:1px solid #999; padding:6px;"><code>str</code>, optional</td>
      <td style="border:1px solid #999; padding:6px;">agent ID，不传则自动生成 UUID。</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>stop_event</code></td>
      <td style="border:1px solid #999; padding:6px;"><code>threading.Event</code>, optional</td>
      <td style="border:1px solid #999; padding:6px;">外部终止信号。</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>tool_execution_timeout</code></td>
      <td style="border:1px solid #999; padding:6px;"><code>float</code>, optional</td>
      <td style="border:1px solid #999; padding:6px;">单个工具调用超时时间。</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>retry_attempts</code></td>
      <td style="border:1px solid #999; padding:6px;"><code>int</code>, optional</td>
      <td style="border:1px solid #999; padding:6px;">速率限制重试次数（&gt;=1）。</td>
    </tr>
  </tbody>
</table>

**`ModelFactory`**  :  模型工厂，用于统一创建模型后端实例。

***Args:***
    <table style="border-collapse:collapse; width:100%; font-size:14px;">
      <thead>
        <tr>
          <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">参数</th>
          <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">类型</th>
          <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">说明</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td style="border:1px solid #999; padding:6px;"><code>model_platform</code></td>
          <td style="border:1px solid #999; padding:6px;">Union[<code>ModelPlatformType</code>, <code>str</code>]</td>
          <td style="border:1px solid #999; padding:6px;">Platform from which the model originates. Can be a string or <code>ModelPlatformType</code> enum.</td>
        </tr>
        <tr>
          <td style="border:1px solid #999; padding:6px;"><code>model_type</code></td>
          <td style="border:1px solid #999; padding:6px;">Union[<code>ModelType</code>, <code>str</code>, <code>UnifiedModelType</code>]</td>
          <td style="border:1px solid #999; padding:6px;">Model for which a backend is created. Can be a string, <code>ModelType</code> enum, or <code>UnifiedModelType</code>.</td>
        </tr>
        <tr>
          <td style="border:1px solid #999; padding:6px;"><code>model_config_dict</code></td>
          <td style="border:1px solid #999; padding:6px;">Optional[Dict]</td>
          <td style="border:1px solid #999; padding:6px;">A dictionary that will be fed into the backend constructor.</td>
        </tr>
        <tr>
          <td style="border:1px solid #999; padding:6px;"><code>token_counter</code></td>
          <td style="border:1px solid #999; padding:6px;">Optional[<code>BaseTokenCounter</code>]</td>
          <td style="border:1px solid #999; padding:6px;">Token counter to use for the model. If not provided, <code>OpenAITokenCounter(ModelType.GPT_4O_MINI)</code> will be used if the model platform didn't provide official token counter.</td>
        </tr>
        <tr>
          <td style="border:1px solid #999; padding:6px;"><code>api_key</code></td>
          <td style="border:1px solid #999; padding:6px;">Optional[<code>str</code>]</td>
          <td style="border:1px solid #999; padding:6px;">The API key for authenticating with the model service.</td>
        </tr>
        <tr>
          <td style="border:1px solid #999; padding:6px;"><code>url</code></td>
          <td style="border:1px solid #999; padding:6px;">Optional[<code>str</code>]</td>
          <td style="border:1px solid #999; padding:6px;">The url to the model service.</td>
        </tr>
        <tr>
          <td style="border:1px solid #999; padding:6px;"><code>timeout</code></td>
          <td style="border:1px solid #999; padding:6px;">Optional[<code>float</code>]</td>
          <td style="border:1px solid #999; padding:6px;">The timeout value in seconds for API calls.</td>
        </tr>
        <tr>
          <td style="border:1px solid #999; padding:6px;"><code>max_retries</code></td>
          <td style="border:1px solid #999; padding:6px;"><code>int</code></td>
          <td style="border:1px solid #999; padding:6px;">Maximum number of retries for API calls.</td>
        </tr>
        <tr>
          <td style="border:1px solid #999; padding:6px;"><code>**kwargs</code></td>
          <td style="border:1px solid #999; padding:6px;"><code>Any</code></td>
          <td style="border:1px solid #999; padding:6px;">Additional model-specific parameters passed to the model constructor (e.g., Azure settings).</td>
        </tr>
      </tbody>
    </table>

**`BaseModelBackend`**

**Args:**
<table style="border-collapse:collapse; width:100%; font-size:14px;">
  <thead>
    <tr>
      <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">参数</th>
      <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">类型</th>
      <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">说明</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>model_platform</code></td>
      <td style="border:1px solid #999; padding:6px;">Union[<code>ModelPlatformType</code>, <code>str</code>]</td>
      <td style="border:1px solid #999; padding:6px;">Platform from which the model originates. Can be a string or <code>ModelPlatformType</code> enum.</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>model_type</code></td>
      <td style="border:1px solid #999; padding:6px;">Union[<code>ModelType</code>, <code>str</code>, <code>UnifiedModelType</code>]</td>
      <td style="border:1px solid #999; padding:6px;">Model for which a backend is created. Can be a string, <code>ModelType</code> enum, or <code>UnifiedModelType</code>.</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>model_config_dict</code></td>
      <td style="border:1px solid #999; padding:6px;">Optional[Dict]</td>
      <td style="border:1px solid #999; padding:6px;">A dictionary that will be fed into the backend constructor.</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>token_counter</code></td>
      <td style="border:1px solid #999; padding:6px;">Optional[<code>BaseTokenCounter</code>]</td>
      <td style="border:1px solid #999; padding:6px;">Token counter to use for the model. If not provided, <code>OpenAITokenCounter(ModelType.GPT_4O_MINI)</code> will be used if the model platform didn't provide official token counter.</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>api_key</code></td>
      <td style="border:1px solid #999; padding:6px;">Optional[<code>str</code>]</td>
      <td style="border:1px solid #999; padding:6px;">The API key for authenticating with the model service.</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>url</code></td>
      <td style="border:1px solid #999; padding:6px;">Optional[<code>str</code>]</td>
      <td style="border:1px solid #999; padding:6px;">The url to the model service.</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>timeout</code></td>
      <td style="border:1px solid #999; padding:6px;">Optional[<code>float</code>]</td>
      <td style="border:1px solid #999; padding:6px;">The timeout value in seconds for API calls.</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>max_retries</code></td>
      <td style="border:1px solid #999; padding:6px;"><code>int</code></td>
      <td style="border:1px solid #999; padding:6px;">Maximum number of retries for API calls.</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>**kwargs</code></td>
      <td style="border:1px solid #999; padding:6px;"><code>Any</code></td>
      <td style="border:1px solid #999; padding:6px;">Additional model-specific parameters passed to the model constructor (e.g., Azure settings).</td>
    </tr>
  </tbody>
</table>

### 3、示例

```python
from camel.agents.chat_agent import ChatAgent
from agents import backend_model

def example_agent_factory():
    tools = [
        search_tool,
        summarize_tool
    ]

    system_message = """
    你是一个应用级 Agent，
    负责为系统 Agent提供应用数据检索和分析能力。
    """

    return ChatAgent(
        system_message=system_message,
        model=backend_model(),
        tools=tools,
    )
```

## 新增Toolkit

### 1、概述

Toolkit 用于封装应用能力为可调用工具，并通过 `get_tools()` 暴露给 Agent。  
所有 Toolkit 继承自 `BaseToolkit`，工具函数由 `FunctionTool` 包装并自动生成 schema。

### 2、标准创建流程

与 “新增Agent” 类似，新增 Toolkit 也有固定的工程化步骤：

1. **继承基类并引入工具封装器**  
   必须继承 `BaseToolkit`，并引入 `FunctionTool` 作为工具包装器：

   ```python
   from camel.toolkits.base import BaseToolkit
   from camel.toolkits import FunctionTool
   ```

2. **实现业务能力函数（工具函数）**  
   在类中定义实际功能函数，建议包含类型注解与完整 docstring，以便自动生成高质量工具 schema。

3. **通过 `get_tools()` 显式暴露工具**  
   实现 `get_tools()` 并返回 `FunctionTool` 列表，将可调用函数注册为工具接口：

   ```python
   def get_tools(self) -> List[FunctionTool]:
       return [
           FunctionTool(self.xxx),
       ]
   ```

### 3、架构设计

Toolkit 的核心组成包括：
- **BaseToolkit**：工具集合的基类，要求实现 `get_tools()`。
- **FunctionTool**：把函数封装为可调用工具，并生成 OpenAI 工具 schema。
- **RegisteredAgentToolkit**：可选的 Agent 注册能力，让 Toolkit 访问 `ChatAgent` 实例。

***Classes***

**BaseToolkit**：工具集合基类，规定 `get_tools()` 输出 `FunctionTool` 列表。  
***Args:***
<table style="border-collapse:collapse; width:100%; font-size:14px;">
  <thead>
    <tr>
      <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">参数</th>
      <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">类型</th>
      <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">说明</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>timeout</code></td>
      <td style="border:1px solid #999; padding:6px;">Optional[<code>float</code>]</td>
      <td style="border:1px solid #999; padding:6px;">工具统一超时时间（秒），为每个可调用方法自动注入超时控制。</td>
    </tr>
  </tbody>
</table>

**FunctionTool**：函数工具封装器，负责 schema 生成与调用。  
***Args:***
<table style="border-collapse:collapse; width:100%; font-size:14px;">
  <thead>
    <tr>
      <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">参数</th>
      <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">类型</th>
      <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">说明</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>func</code></td>
      <td style="border:1px solid #999; padding:6px;"><code>Callable</code></td>
      <td style="border:1px solid #999; padding:6px;">被封装的函数。</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>openai_tool_schema</code></td>
      <td style="border:1px solid #999; padding:6px;">Optional[Dict[str, Any]]</td>
      <td style="border:1px solid #999; padding:6px;">自定义 OpenAI 工具 schema（可覆盖自动生成）。</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>synthesize_schema</code></td>
      <td style="border:1px solid #999; padding:6px;">Optional[<code>bool</code>]</td>
      <td style="border:1px solid #999; padding:6px;">是否启用 schema 合成模式。</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>synthesize_schema_model</code></td>
      <td style="border:1px solid #999; padding:6px;">Optional[<code>BaseModelBackend</code>]</td>
      <td style="border:1px solid #999; padding:6px;">schema 合成使用的模型。</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>synthesize_schema_max_retries</code></td>
      <td style="border:1px solid #999; padding:6px;"><code>int</code></td>
      <td style="border:1px solid #999; padding:6px;">schema 合成的最大重试次数。</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>synthesize_output</code></td>
      <td style="border:1px solid #999; padding:6px;">Optional[<code>bool</code>]</td>
      <td style="border:1px solid #999; padding:6px;">是否启用工具输出合成模式。</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>synthesize_output_model</code></td>
      <td style="border:1px solid #999; padding:6px;">Optional[<code>BaseModelBackend</code>]</td>
      <td style="border:1px solid #999; padding:6px;">输出合成使用的模型。</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>synthesize_output_format</code></td>
      <td style="border:1px solid #999; padding:6px;">Optional[Type[<code>BaseModel</code>]]</td>
      <td style="border:1px solid #999; padding:6px;">合成输出的结构化格式。</td>
    </tr>
  </tbody>
</table>

**RegisteredAgentToolkit**：可选的 Agent 注册能力，允许 Toolkit 获取 `ChatAgent` 实例。  



<table style="border-collapse:collapse; width:100%; font-size:14px;">
  <thead>
    <tr>
      <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">参数</th>
      <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">类型</th>
      <th style="border:1px solid #999; padding:6px; background:#f2f2f2;">说明</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>role_name</code></td>
      <td style="border:1px solid #999; padding:6px;"><code>str</code></td>
      <td style="border:1px solid #999; padding:6px;">The name of the user or assistant role.</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>role_type</code></td>
      <td style="border:1px solid #999; padding:6px;"><code>RoleType</code></td>
      <td style="border:1px solid #999; padding:6px;">The type of role, either <code>RoleType.ASSISTANT</code> or <code>RoleType.USER</code>.</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>meta_dict</code></td>
      <td style="border:1px solid #999; padding:6px;">Optional[Dict[str, Any]]</td>
      <td style="border:1px solid #999; padding:6px;">Additional metadata dictionary for the message.</td>
    </tr>
    <tr>
      <td style="border:1px solid #999; padding:6px;"><code>content</code></td>
      <td style="border:1px solid #999; padding:6px;"><code>str</code></td>
      <td style="border:1px solid #999; padding:6px;">The content of the message.</td>
    </tr>
  </tbody>
</table>
