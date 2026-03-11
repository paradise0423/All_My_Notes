# MAOS - 终端多智能体操作系统

本项目是一个面向终端/本地环境的多智能体系统示例，核心能力包括：系统级协调、多应用级代理、工具化能力接入，以及基于本地数据的个性化“soul”信号读取与更新。

# 系统架构

系统由三层核心构成：

- **系统级 Agent 层**：负责任务拆解、协调与治理。  
  代码位置：`agents/coordinate_agent.py`、`agents/task_agent.py`、`agents/soul_agent.py`、`agents/document_agent.py`、`agents/developer_agent.py`

- **应用级 Agent 层**：按应用/业务域封装能力。  
  代码位置：`agents/contactors_agent.py`、`agents/notes_agent.py`、`agents/photos_agent.py`、`agents/xiaohongshu_agent.py`、`agents/xiecheng_agent.py`、`agents/search_agent.py`

- **Toolkit 层**：将具体数据或功能封装为可调用工具。  
  代码位置：`tools/`

统一的模型入口在 `agents/backend_model.py`，通过 `ModelFactory.create()` 创建 `BaseModelBackend` 的具体实现，再注入 `ChatAgent`。

# 核心功能

- **系统级任务编排**：Coordinator 负责分发任务，Task Planner 负责拆解任务与依赖管理。
- **应用能力封装**：每个 App-Level Agent 以 `xxx_agent_factory()` 的方式构建并返回 `ChatAgent`。
- **工具化能力接入**：Toolkit 提供 `get_tools()` 暴露工具给 Agent。
- **个性化治理（Soul）**：Soul Agent 读取 `get_user_soul` 与各应用的 `get_*_soul`，在需要时更新并持久化新档案。
- **离线/本地数据演示**：应用数据存放于 `mock_data/`，工具默认读取本地 JSON/Markdown。

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

## 新增Agent（V3 更更简洁版）

### 1、概述

在 AIOS 中，**App-Level Agent（应用级代理）**用于表示系统中的具体应用，例如 Notes、Contactors、Photos 等。每个 App-Level Agent 封装了与某个应用相关的能力，用于处理应用数据，向系统级 Agent 返回结果，等等。

App-Level Agent 可以理解为 **AIOS 多代理系统与应用之间的接口层**，系统级 Agent 通过调用 App-Level Agent，从而访问不同应用的数据与能力。


### 2、架构设计

所有 App-Level Agent 都基于 `camel-master/camel/agents/chat_agent.py` 中的 `ChatAgent` 类构建，以 `Function` 形式封装，命名方式遵循 `xxx_agent_factory()` 的方式，在函数内部完成 Agent 的配置与初始化，最终返回一个 `ChatAgent` 实例。

**Classes**

ChatAgent  :Agent 核心部分，负责模型调用、工具调用与消息编排。
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


BaseMessage  :消息基类，用于构造系统/用户/助手等角色消息。
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


ModelFactory  :模型工厂，用于统一创建模型后端实例。
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

---

### 3、基座模型配置

基座模型统一来自 `agents/backend_model.py` 的 `backend_model()`。  
该函数从环境变量读取密钥与服务地址，并创建 OpenAI 平台的 `GPT_5_NANO` 模型实例。

关键点：
- 依赖环境变量：`OPENAI_API_KEY`、`url`
- 模型平台：`ModelPlatformType.OPENAI`
- 模型类型：`ModelType.GPT_5_NANO`

### 4、最小示例

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

## 新增 Toolkit

### 1、概述

Toolkit 用于封装应用能力为可调用工具，并通过 `get_tools()` 暴露给 Agent。  
所有 Toolkit 继承自 `BaseToolkit`，工具函数由 `FunctionTool` 包装并自动生成 schema。

### 2、项目内 Toolkit 列表

**SoulToolkit**（`tools/soul_toolkit.py`）  
用于读写 soul 档案与任务状态。

Tools:
- `get_user_soul()`：读取 `mock_data/soul/soul.md`
- `get_task_status()`：读取 `working_dir/task_status.md`
- `save_soul(new_soul: str)`：写入 `mock_data/soul/soul_new.md`
- `announce_tool()`：声明无需更新 soul

**ContactorsRetrievalToolkit**（`tools/contactors_toolkit.py`）  
联系人/通话记录读取工具（只读 mock 数据）。

Tools:
- `search_my_contactors()`：返回联系人 JSON（全量）
- `get_contactors_soul(query: str)`：返回联系人应用的个性化信号

**NotesRetrievalToolkit**（`tools/notes_toolkit.py`）  
备忘录读取工具（只读 mock 数据）。

Tools:
- `search_my_notes()`：返回笔记 JSON（全量）
- `get_notes_soul(query: str)`：返回备忘录应用的个性化信号

**PhotosToolkit**（`tools/photos_toolkit.py`）  
照片检索与图像分析工具。

Tools:
- `search_photos()`：返回照片 JSON（全量）
- `get_image_information(image_path: str, user_message: Optional[str] = None)`：图像分析
- `get_photos_soul(query: str)`：返回相册应用的个性化信号

**XiaoHongShuToolkit**（`tools/xiaohongshu_toolkit.py`）  
小红书内容发布工具（写入本地 mock 输出）。

Tools:
- `publish_xhs_post(title: str, text: str, image_paths: list[str])`：生成并保存帖子 JSON
- `get_xiaohongshu_soul(query: str)`：返回小红书应用的个性化信号

**XiechengToolkit**（`tools/xiecheng_toolkit.py`）  
旅行数据检索工具（景点/攻略/订单）。

Tools:
- `search_attractions()`：景点数据（全量）
- `search_guides()`：攻略数据（全量）
- `search_orders()`：订单数据（全量）
- `get_xiecheng_soul(query: str)`：返回携程应用的个性化信号

- `check_storage(limit: int = None)`：查看存量 payload
- `delete_or_reset_collection(collectionname: str = None, reset: bool = False)`：清空或删除集合
