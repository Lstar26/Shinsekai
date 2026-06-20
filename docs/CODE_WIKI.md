# Shinsekai Code Wiki

> **项目名称**: 新世界（Shinsekai）
> **版本**: 2.1.0
> **项目类型**: 面向 Galgame / 乙女 / 剧情向 RPG 的桌面助手
> **核心特性**: 用大语言模型驱动角色对白，立绘与情绪联动，支持语音合成、语音识别与视觉、工具等扩展

---

## 目录

1. [项目整体架构](#1-项目整体架构)
2. [主要模块职责](#2-主要模块职责)
3. [核心类与函数说明](#3-核心类与函数说明)
4. [依赖关系](#4-依赖关系)
5. [项目运行方式](#5-项目运行方式)
6. [插件系统](#6-插件系统)
7. [消息流与工作流](#7-消息流与工作流)

---

## 1. 项目整体架构

### 1.1 架构概览

Shinsekai 采用分层架构设计，主要分为以下几个层次：

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend (React)                          │
│         React Settings UI + Chat Stage (Browser/Tauri)          │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Frontend Bridge (Python)                      │
│              HTTP Server + WebSocket Chat Stream                 │
│                    frontend_bridge.py                            │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Core Modules                                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│  │  Config   │ │    LLM   │ │   TTS    │ │    ASR   │          │
│  │ Manager  │ │ Manager  │ │ Manager  │ │ Manager  │          │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│  │  Sprite  │ │ Plugins  │ │ Workflow │ │ Handlers │          │
│  │  System  │ │   Host   │ │   DAG    │ │ Registry │          │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘          │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                        SDK Layer                                 │
│     Adapters | Messages | Tool Registry | Plugin Base           │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 核心目录结构

```
/workspace/
├── main.py                    # 聊天主程序入口（PySide6桌面/流式/无头模式）
├── frontend_bridge.py        # React前端HTTP桥接服务
├── webui_react.py            # React WebUI启动脚本
├── webui_qt.py               # 旧版PySide6设置界面
│
├── config/                   # 配置管理模块
│   ├── config_manager.py      # 配置管理器（单例）
│   ├── schema.py             # Pydantic配置模型定义
│   ├── character_manager.py  # 角色管理
│   ├── background_manager.py # 背景管理
│   └── tts_provider_config.py
│
├── core/                     # 核心运行时
│   ├── runtime/
│   │   ├── app_runtime.py    # 应用运行时上下文
│   │   ├── workflow.py        # DAG工作流构建
│   │   ├── workers.py         # 工作线程管理
│   │   └── shutdown.py        # 关闭流程
│   ├── handlers/             # 消息处理器
│   ├── messaging/            # 消息解析
│   ├── sprite/               # 立绘与聊天历史
│   └── plugins/              # 插件宿主
│
├── llm/                      # LLM模块
│   ├── llm_manager.py        # LLM管理器（对话、工具调用）
│   ├── llm_adapter.py        # LLM适配器基类
│   ├── compact_manager.py    # 历史压缩管理
│   └── tools/                # LLM工具集
│
├── tts/                      # 语音合成模块
│   ├── tts_manager.py        # TTS管理器
│   └── tts_adapter.py        # TTS适配器
│
├── asr/                      # 语音识别模块
│   ├── asr_manager.py        # ASR管理器
│   └── asr_adapter.py        # ASR适配器
│
├── t2i/                      # 文生图模块
│   ├── t2i_manager.py        # T2I管理器
│   └── t2i_adapter.py        # T2I适配器
│
├── sdk/                      # SDK层（插件开发接口）
│   ├── plugin.py             # 插件基类
│   ├── plugin_host_context.py
│   ├── manager.py            # 插件管理器
│   ├── tool_registry.py      # 工具注册装饰器
│   ├── handlers.py           # 消息处理器接口
│   ├── messages.py           # 队列消息模型
│   ├── graph.py              # DAG工作流原语
│   ├── adapters/             # 适配器基类
│   ├── logging/              # 日志系统
│   └── exception/            # 异常处理
│
├── frontend/                 # React前端
│   ├── src/
│   │   ├── app/              # 应用路由与Shell
│   │   ├── features/         # 功能页面
│   │   │   ├── api-settings/ # API设置
│   │   │   ├── chat-stage/   # 聊天舞台
│   │   │   ├── character-editor/
│   │   │   └── ...
│   │   ├── entities/         # 数据实体
│   │   └── shared/           # 共享组件
│   └── src-tauri/            # Tauri桌面壳
│
├── frontend_bridge_core/     # 前端桥接核心功能
│   ├── chat.py              # 聊天启动管理
│   ├── characters.py        # 角色CRUD
│   ├── backgrounds.py       # 背景CRUD
│   ├── handler.py           # HTTP请求处理
│   └── ...
│
├── ui/                       # PySide6旧版界面
│   ├── chat_ui/             # 聊天窗口
│   └── settings_ui/         # 设置界面
│
└── tools/                    # 工具脚本
    ├── comfyui_workflow2api.py
    ├── generate_sprites.py
    └── ...
```

---

## 2. 主要模块职责

### 2.1 配置模块 (`config/`)

| 文件 | 职责 |
|------|------|
| [config_manager.py](file:///workspace/config/config_manager.py) | 配置管理器单例，负责加载/保存YAML配置，协调LLM/TTS/ASR/T2I适配器 |
| [schema.py](file:///workspace/config/schema.py) | Pydantic配置模型：Character、Background、ApiConfig、SystemConfig、AppConfig |
| [character_manager.py](file:///workspace/config/character_manager.py) | 角色的增删改查操作 |
| [background_manager.py](file:///workspace/config/background_manager.py) | 背景图片与BGM的管理 |
| [tts_provider_config.py](file:///workspace/config/tts_provider_config.py) | TTS提供商配置工具函数 |

### 2.2 核心运行时 (`core/`)

| 模块 | 职责 |
|------|------|
| [app_runtime.py](file:///workspace/core/runtime/app_runtime.py) | 应用运行时上下文单例，`set_app_runtime()`/`get_app_runtime()` 全局访问配置、UI更新、LLM/TTS/T2I管理器、队列等 |
| [workflow.py](file:///workspace/core/runtime/workflow.py) | DAG工作流构建，`build_runtime_workflow()` 从YAML加载并执行流水线 |
| [workers.py](file:///workspace/core/runtime/workers.py) | TTS Worker、UI Worker、ASR Worker 等工作线程实现 |
| [shutdown.py](file:///workspace/core/runtime/shutdown.py) | 优雅关闭流程 |
| [sprite/chat_history.py](file:///workspace/core/sprite/chat_history.py) | 对话历史管理与持久化 |
| [sprite/chat_branch_storage.py](file:///workspace/core/sprite/chat_branch_storage.py) | 分支状态存储（支持对话分支Fork） |
| [handlers/handler_registry.py](file:///workspace/core/handlers/handler_registry.py) | 消息处理器注册表 |
| [plugins/plugin_host.py](file:///workspace/core/plugins/plugin_host.py) | 插件发现、生命周期管理 |

### 2.3 LLM模块 (`llm/`)

| 文件 | 职责 |
|------|------|
| [llm_manager.py](file:///workspace/llm/llm_manager.py) | LLM对话管理器，支持流式/同步输出、工具调用、自动压缩、上下文管理 |
| [llm_adapter.py](file:///workspace/llm/llm_adapter.py) | LLM适配器基类，定义统一接口 |
| [compact_manager.py](file:///workspace/llm/compact_manager.py) | 历史上下文压缩管理 |
| [tools/tool_manager.py](file:///workspace/llm/tools/tool_manager.py) | LLM工具管理器，工具注册与分组 |
| [tools/tool_executor.py](file:///workspace/llm/tools/tool_executor.py) | 工具执行器，处理工具调用、冷却、风险确认 |

### 2.4 适配器模块 (`sdk/adapters/`)

| 文件 | 职责 |
|------|------|
| [llm.py](file:///workspace/sdk/adapters/llm.py) | LLM适配器基类 `LLMAdapter` |
| [tts.py](file:///workspace/sdk/adapters/tts.py) | TTS适配器基类 `TTSAdapter` |
| [asr.py](file:///workspace/sdk/adapters/asr.py) | ASR适配器基类 `ASRAdapter` |
| [t2i.py](file:///workspace/sdk/adapters/t2i.py) | T2I适配器基类 `T2IAdapter` |

### 2.5 SDK模块 (`sdk/`)

| 文件 | 职责 |
|------|------|
| [plugin.py](file:///workspace/sdk/plugin.py) | 插件基类 `PluginBase`，定义生命周期钩子 |
| [manager.py](file:///workspace/sdk/manager.py) | `PluginManager` 管理插件实例与应用贡献 |
| [tool_registry.py](file:///workspace/sdk/tool_registry.py) | `@tool` 装饰器，声明式注册LLM工具函数 |
| [handlers.py](file:///workspace/sdk/handlers.py) | `MessageHandler`/`UIOutputMessageHandler` 抽象基类 |
| [messages.py](file:///workspace/sdk/messages.py) | Pydantic队列消息模型：UserInputMessage、LLMDialogMessage、TTSOutputMessage |
| [graph.py](file:///workspace/sdk/graph.py) | DAG工作流原语：DagNode、DagBuilder、Dag |

---

## 3. 核心类与函数说明

### 3.1 应用入口

#### `main.py`

| 类/函数 | 说明 |
|---------|------|
| `main()` | 主程序入口，支持三种运行模式：PySide6桌面、流式WebSocket、无头模式 |
| `_StreamWindowProxy` | 流式模式下的窗口代理，将UI调用转发到StreamingUIUpdateManager |
| `_startup_phase()` | 上下文管理器，记录启动步骤耗时与日志 |
| `_shutdown_plugins()` | 关闭所有已加载插件 |

**启动流程**:
1. 加载配置文件 (`ConfigManager`)
2. 初始化i18n
3. 加载插件 (`ensure_plugins_loaded`)
4. 解析命令行参数 (`parse_sprite_args`)
5. 初始化T2I/TTS/LLM管理器
6. 构建DAG工作流
7. 根据模式启动UI或流式服务

#### `frontend_bridge.py`

| 类/函数 | 说明 |
|--------|------|
| `run()` | 启动HTTP桥接服务器，托管React前端静态文件与API |
| `check_runtime()` | 验证Python运行时依赖是否完整 |
| `_start_plugin_loader()` | 后台线程异步加载插件 |

### 3.2 配置管理

#### `config/config_manager.py` - `ConfigManager`

```python
class ConfigManager:
    """配置管理器单例"""
    
    @property
    def config(self) -> AppConfig           # 获取完整配置
    def reload(self) -> None                 # 重新加载所有配置
    def save_api_config(self) -> None        # 保存API配置到api.yaml
    def save_system_config(self) -> None     # 保存系统配置
    def save_characters_config(self) -> None # 保存角色列表
    def save_background_config(self) -> None # 保存背景配置
    
    # 获取特定配置
    def get_llm_api_config(self) -> tuple[str, str, str, str]
    def get_gpt_sovits_config(self) -> tuple[str, str, str]
    def get_character_by_name(self, name: str) -> Character | None
    def get_background_by_name(self, name: str) -> Background | None
    
    # 适配器工厂参数合并
    def merged_llm_factory_kwargs(...) -> Dict
    def merged_tts_factory_kwargs(...) -> Dict
    def merged_t2i_factory_kwargs(...) -> Dict
```

#### `config/schema.py` - 配置模型

```python
class Character(BaseModel):
    name: str                    # 角色名称
    color: str                   # 对话框名字颜色
    sprite_prefix: str           # 立绘文件名通用前缀
    sprites: List[Sprite]       # 立绘列表
    character_setting: str       # 角色设定
    emotion_tags: str            # 情绪标签映射
    gpt_model_path: str          # GPT-SoVITS模型路径
    sovits_model_path: str       # SoVITS模型路径
    # ... 更多字段

class Background(BaseModel):
    name: str
    sprites: List[Sprite]        # 背景图片列表
    bgm_list: List[str]          # 背景音乐列表
    # ...

class ApiConfig(BaseModel):
    llm_provider: str            # LLM提供商
    llm_model: Dict[str, str]    # 各提供商模型
    llm_api_key: Dict[str, str]  # API密钥
    tts_provider: str            # TTS引擎
    t2i_provider: str            # T2I引擎
    # ...

class SystemConfig(BaseModel):
    ui_language: str            # 界面语言
    voice_language: str          # 语音语言
    asr_provider: str            # ASR后端
    # ...
```

### 3.3 LLM管理

#### `llm/llm_manager.py` - `LLMManager`

```python
class LLMManager:
    def __init__(self, adapter, user_template, max_tokens, ...):
        self.llm_adapter = adapter          # LLM适配器
        self.messages = []                   # 对话历史
        self.compact_manager = CompactManager
        self.tool_executor = ToolExecutor
        # ...
    
    def set_user_template(self, template: str)     # 设置系统提示
    def add_message(self, role, content, **kwargs) # 添加消息（含自动压缩）
    def get_messages(self) -> list                 # 获取对话历史
    def set_messages(self, new_messages)           # 设置对话历史
    def chat(self, user_input, stream=True, **kwargs) -> Union[Generator, str]
                                                  # 对话入口，自动处理工具调用循环
```

#### `llm/llm_adapter.py` - LLM适配器

```python
class LLMAdapter(ABC):
    """LLM适配器基类"""
    
    @abstractmethod
    def chat(self, messages, stream=True, tools=None, **kwargs) -> Union[Generator, str]:
        """发送对话请求"""
        ...
    
    def set_user_template(self, template: str) -> None:
        self.user_template = template

# 内置适配器
class DeepSeekAdapter(LLMAdapter)
class OpenAIAdapter(LLMAdapter)
class GeminiAdapter(LLMAdapter)
class ClaudeAdapter(LLMAdapter)
```

#### `LLMAdapterFactory` - LLM工厂

```python
class LLMAdapterFactory:
    _adapters = {
        "Deepseek": DeepSeekAdapter,
        "ChatGPT": OpenAIAdapter,
        "Gemini": OpenAIAdapter,
        "Claude": ClaudeAdapter,
        "豆包": OpenAIAdapter,
        "通义千问": OpenAIAdapter,
        "Ollama": OpenAIAdapter
    }
    
    @staticmethod
    def create_adapter(llm_provider: str, **kwargs) -> LLMAdapter:
        ...
```

### 3.4 TTS管理

#### `tts/tts_manager.py` - `TTSManager`

```python
class TTSManager:
    def __init__(self, tts_server_url):
        self.tts_adapter = None          # 当前TTS适配器
        self.task_queue = Queue          # 任务队列
        self.voice_language = "ja"         # 语音语言
    
    def set_tts_adapter(self, adapter: TTSAdapter)  # 切换适配器
    def set_language(self, language: str)           # 设置语言
    def generate_tts(self, text, text_processor, ...) -> str  # 生成语音，返回文件路径
    def switch_model(self, model_info)              # 切换模型
    def shutdown(self)                               # 关闭
```

#### `TTSAdapterFactory`

```python
class TTSAdapterFactory:
    _adapters = {
        'gpt-sovits': GPTSoVitsAdapter,
        'kaggle-gpt-sovits': KaggleGPTSoVitsAdapter,
        'genie-tts': GenieTTSAdapter,
        'index-tts': IndexTTSAdapter,
        'cosyvoice': CosyVoiceAdapter,
    }
```

### 3.5 工具系统

#### `llm/tools/tool_manager.py` - `ToolManager`

```python
class ToolManager:
    def register_function(self, func, name, description, group, risk) -> None
    def get_definitions(self, groups=None) -> list[dict]   # 获取工具定义
    def get_tool_group(self, name: str) -> str             # 获取工具分组
    def get_groups(self) -> list[str]                      # 获取所有分组
```

#### `sdk/tool_registry.py` - 工具注册装饰器

```python
@tool(name="my_tool", description="...", group="default", risk="low")
def my_tool(...):
    """LLM可调用的工具函数"""
    ...

# 工具就绪通知
def notify_tool_ready(group: str, message: str = "") -> None
```

### 3.6 消息处理器

#### `core/handlers/handler_registry.py`

```python
class MessageHandler(ABC):
    """TTS队列消息处理器"""
    def can_handle(self, msg: LLMDialogMessage) -> bool: ...
    def pre_process(self, msg: LLMDialogMessage) -> None: ...
    def handle(self, msg: LLMDialogMessage) -> None: ...
    def post_process(self, msg: LLMDialogMessage) -> None: ...
    def init(self) -> None: ...

class UIOutputMessageHandler(ABC):
    """UI队列消息处理器"""
    def can_handle(self, out: TTSOutputMessage) -> bool: ...
    def handle(self, out: TTSOutputMessage) -> None: ...
```

### 3.7 应用运行时

#### `core/runtime/app_runtime.py`

```python
@dataclass
class AppRuntime:
    """应用全局运行时上下文"""
    config: ConfigManager           # 配置管理器
    ui_update_manager: Any         # UI更新管理器
    llm_manager: LLMManager        # LLM管理器
    tts_manager: TTSManager | None # TTS管理器
    t2i_manager: T2IManager | None # T2I管理器
    bgm_list: List[Any]            # BGM列表
    user_input_queue: Any           # 用户输入队列
    tts_queue: Any                 # TTS输入队列
    audio_path_queue: Any          # 音频路径队列
    text_processor: TextProcessor  # 文本处理器
    opencc: OpenCC                  # 繁简转换

def set_app_runtime(rt: AppRuntime) -> None
def get_app_runtime() -> AppRuntime  # 全局访问（抛出异常）
def try_get_app_runtime() -> AppRuntime | None  # 安全访问
def tts_emit_to_ui_queue(...) -> None  # 发射TTS结果到UI队列
```

### 3.8 DAG工作流

#### `sdk/graph.py`

```python
class DagNode:
    """DAG节点基类"""
    @property
    def name(self) -> str
    def inputs(self) -> dict[str, Port]      # 输入端口
    def outputs(self) -> dict[str, Port]     # 输出端口
    def configure(self, nodes) -> None        # 配置节点引用
    def start(self) / def stop(self)          # 生命周期
    def bind_input(port_name, queue) / bind_output(port_name, queue)

class DagBuilder:
    def add_node(node: DagNode) -> Self
    def connect(src, src_port, dst, dst_port) -> Self
    def build() -> list[DagNode]  # 构建DAG
    def load_yaml(path_or_text) -> None
    def to_yaml(path=None) -> str

class Dag:
    """DAG执行器"""
    def load_yaml(path_or_text) -> Dag
    def build() -> list[DagNode]
    def start() / def stop()     # 启动/停止所有节点
    def get_node(name) -> DagNode
    def resolve_export(name) -> Any  # 获取导出值
```

### 3.9 插件系统

#### `sdk/plugin.py` - `PluginBase`

```python
class PluginBase(ABC):
    @property
    @abstractmethod
    def plugin_id(self) -> str:  # 唯一ID，如 "com.example.myplugin"
    
    @property
    def plugin_version(self) -> str: return "0.1.0"
    @property
    def plugin_name(self) -> str: ...   # 显示名称
    @property
    def enabled(self) -> bool: return True
    @property
    def priority(self) -> int: return 100  # 启动优先级
    
    @abstractmethod
    def initialize(self, register, plugin_root, host) -> None:
        """注册插件能力到register"""
    
    def shutdown(self) -> None:
        """关闭时清理资源"""
```

#### `sdk/manager.py` - `PluginManager`

```python
class PluginManager:
    def register_plugin_class(cls: Type[PluginBase]) -> None
    def register_plugin_entry(entry: str, enabled=True) -> None
    def load_manifest_file(path: Path) -> None  # 加载plugins.yaml
    def instantiate_all() -> None                 # 实例化所有插件
    def load_own_config_all(app_config) -> None  # 调用initialize
    
    # 应用插件贡献
    def apply_llm_providers(target: dict) -> None
    def apply_tts_providers(target: dict) -> None
    def apply_llm_tools(tool_manager) -> None
    def collect_message_handlers() -> tuple[list, list]
    def collect_settings_contributions() -> list
```

---

## 4. 依赖关系

### 4.1 核心依赖

```
requirements.txt
├── numpy                  # 数值计算
├── pygame                 # 音频播放
├── openai                 # OpenAI兼容API调用
├── anthropic              # Claude API
├── google-genai           # Gemini API
├── pydantic               # 配置模型与消息验证
├── requests               # HTTP请求
├── onnxruntime            # ONNX推理
├── PySide6                # 桌面GUI
├── yaml                   # 配置文件
├── vosk                   # 离线语音识别
├── mcp                    # MCP协议
├── tiktoken               # Token计数
├── sentence-transformers   # 向量嵌入
├── fastembed              # 快速向量检索
└── mem0ai                 # 记忆系统
```

### 4.2 模块依赖图

```
┌─────────────────────────────────────────────────────────┐
│                      main.py                            │
│         依赖: ConfigManager, LLMManager, TTSManager     │
└───────────────────────┬─────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────┐
│                  ConfigManager                          │
│         依赖: ApiConfig, SystemConfig (Pydantic)         │
└───────────────────────┬─────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────┐
│              LLMManager / TTSManager                    │
│         依赖: LLMAdapter / TTSAdapter (Factory)         │
└───────────────────────┬─────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────┐
│                    AppRuntime                           │
│   持有: config, llm_manager, tts_manager, t2i_manager   │
│   UI队列: user_input_queue, tts_queue, audio_path_queue │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   PluginHost                            │
│   依赖: PluginManager, ToolRegistry, MessageHandlers    │
│   贡献: 适配器(LLM/TTS/ASR/T2I), 工具, UI贡献, 工作流   │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│              FrontendBridge (HTTP Server)                │
│   依赖: ConfigManager, CharacterManager, ChatStream     │
│   API端点: /api/config, /api/characters, /api/chat/*    │
└─────────────────────────────────────────────────────────┘
```

---

## 5. 项目运行方式

### 5.1 启动方式

#### 方式一：React前端桥接服务（推荐）

```bash
# Windows
双击 start-react.bat

# macOS
双击 start-react.command

# Linux
./start-react.sh
```

这将：
1. 启动 `frontend_bridge.py` HTTP服务器（默认端口8787）
2. 托管React前端静态文件
3. 自动打开浏览器

#### 方式二：PySide6桌面模式

```bash
python main.py [参数]
```

**命令行参数**:
| 参数 | 说明 |
|------|------|
| `--template <name>` | 聊天模板文件名（不含.txt后缀） |
| `--history <name>` | 历史会话名称 |
| `--bg <name>` | 背景名称 |
| `--tts <engine>` | TTS引擎（如gpt-sovits） |
| `--t2i <engine>` | T2I引擎（如comfyui） |
| `--headless` | 无头模式（无GUI） |
| `--stream-endpoint <url>` | 流式输出端点 |
| `--workflow <path>` | 工作流YAML路径 |

#### 方式三：React Chat Stage（流式）

```bash
# 在start-react.bat启动后，通过前端发起聊天启动请求
# 后端会fork main.py并通过WebSocket通信
```

### 5.2 配置数据位置

```
data/
├── config/
│   ├── api.yaml              # LLM/TTS/T2I API配置
│   ├── system_config.yaml    # 系统配置（语言、ASR等）
│   ├── characters.yaml       # 角色列表
│   ├── background.yaml       # 背景列表
│   ├── plugins.yaml          # 插件清单
│   └── mcp.yaml             # MCP服务器配置
├── character_templates/      # 聊天模板文件
├── chat_history/             # 对话历史存档
├── chat_ui_themes/          # 聊天UI主题
├── characters/               # 角色资源文件
├── backgrounds/             # 背景图片
├── sprite/                   # 立绘图片
├── speech/                   # 角色语音文件
├── bgm/                      # BGM文件
└── plugins/                  # 插件代码
```

### 5.3 环境变量

| 变量 | 说明 |
|------|------|
| `EASYAI_PROJECT_ROOT` | 项目数据根目录 |
| `SHINSEKAI_SOURCE_ROOT` | 源码根目录 |
| `SHINSEKAI_APP_ROOT` | 桌面安装目录 |
| `SHINSEKAI_BRIDGE_AUTH_TOKEN` | 前端桥接认证令牌 |

---

## 6. 插件系统

### 6.1 插件结构

```
plugins/<plugin_name>/
├── __init__.py              # 入口（定义Plugin类）
├── your_plugin.py           # 插件实现
└── [可选子模块/资源]
```

### 6.2 插件清单 (plugins.yaml)

```yaml
- entry: plugins.example_plugin:ExamplePlugin
  enabled: true
- entry: plugins.another_plugin
  enabled: false
```

### 6.3 插件开发示例

```python
# plugins/my_plugin/__init__.py
from pathlib import Path
from sdk.plugin import PluginBase
from sdk.register import PluginCapabilityRegistry
from sdk.plugin_host_context import PluginHostContext
from sdk.tool_registry import tool

class MyPlugin(PluginBase):
    @property
    def plugin_id(self) -> str:
        return "com.example.myplugin"
    
    @property
    def plugin_name(self) -> str:
        return "My Plugin"
    
    @property
    def plugin_description(self) -> str:
        return "示例插件"
    
    def initialize(self, register: PluginCapabilityRegistry, 
                   plugin_root: Path, host: PluginHostContext) -> None:
        # 注册LLM适配器
        register.register_llm_adapter("my-llm", MyLLMAdapter)
        
        # 注册TTS适配器
        register.register_tts_adapter("my-tts", MyTTSAdapter)
        
        # 注册工具
        register.register_llm_tool_group("my_tools", [my_tool_function])
        
        # 注册UI贡献
        register.register_settings_ui(...)
        register.register_chat_ui(...)

# 工具函数示例
@tool(name="my_tool", description="执行某个操作", group="my_tools")
def my_tool(param: str) -> str:
    return f"结果: {param}"
```

### 6.4 插件能力注册

```python
class PluginCapabilityRegistry:
    # 适配器注册
    def register_llm_adapter(self, name: str, cls: Type[LLMAdapter]) -> None
    def register_tts_adapter(self, name: str, cls: Type[TTSAdapter]) -> None
    def register_asr_adapter(self, name: str, cls: Type[ASRAdapter]) -> None
    def register_t2i_adapter(self, name: str, cls: Type[T2IAdapter]) -> None
    
    # 工具注册
    def register_llm_tool_group(self, group: str, 
                                tools: List[Callable]) -> None
    
    # UI贡献
    def register_settings_ui(self, contribution: SettingsUIContribution) -> None
    def register_chat_ui(self, contribution: ChatUIContribution) -> None
    def register_tools_tab(self, contribution: ToolsTabContribution) -> None
    
    # 消息处理器
    def register_message_handler(self, handler: MessageHandler) -> None
    def register_ui_handler(self, handler: UIOutputMessageHandler) -> None
    
    # 工作流
    def register_workflow_yaml(self, path: str) -> None
```

---

## 7. 消息流与工作流

### 7.1 消息队列结构

```
┌──────────────────┐
│  user_input_queue │  UserInputMessage
└────────┬─────────┘
         ▼
┌──────────────────┐
│    LLMManager     │  处理用户输入，调用工具
└────────┬─────────┘
         ▼
┌──────────────────┐
│    tts_queue      │  LLMDialogMessage (对话片段)
└────────┬─────────┘
         ▼
┌──────────────────┐
│   TTS Worker      │  文字转语音
└────────┬─────────┘
         ▼
┌──────────────────┐
│  audio_path_queue │  TTSOutputMessage
└────────┬─────────┘
         ▼
┌──────────────────┐
│   UI Worker       │  更新界面（立绘、对话框、BGM等）
└──────────────────┘
```

### 7.2 默认工作流 (assets/system/workflow/default.yaml)

```yaml
nodes:
  - name: chat.input
    type: core.runtime.workers.UserInputNode
    inputs: []
    outputs: [out]
  
  - name: chat.llm
    type: core.runtime.workers.LLMNode
    inputs: [in]
    outputs: [dialog, reasoning]
  
  - name: chat.tts_input
    type: core.runtime.workers.TTSInputNode
    inputs: [in]
    outputs: [out]
  
  - name: chat.audio_output
    type: core.runtime.workers.TTSWorker
    inputs: [in]
    outputs: []
  
  - name: chat.ui_worker
    type: core.runtime.workers.UIWorker
    inputs: [dialog_in, audio_in]
    outputs: []

edges:
  - src: chat.input; src_port: out
    dst: chat.llm; dst_port: in
  - src: chat.llm; src_port: dialog
    dst: chat.tts_input; dst_port: in
  - src: chat.tts_input; src_port: out
    dst: chat.audio_output; dst_port: in
  - src: chat.audio_output; src_port: out
    dst: chat.ui_worker; dst_port: audio_in
  - src: chat.llm; src_port: dialog
    dst: chat.ui_worker; dst_port: dialog_in

exports:
  chat.input:
    node: chat.input
    direction: node
  chat.tts_input:
    node: chat.tts_input
    direction: node
  chat.audio_output:
    node: chat.audio_output
    direction: node
  chat.ui_worker:
    node: chat.ui_worker
    direction: node
```

### 7.3 关键消息模型

```python
# 用户输入
class UserInputMessage(BaseModel):
    text: str

# LLM对话输出
class LLMDialogMessage(BaseModel):
    name: str = Field(alias="character_name")       # 角色名
    text: Optional[str] = Field("", alias="speech") # 台词
    asset_id: Optional[str] = Field("-1", alias="sprite")  # 立绘索引
    translate: str = ""                             # 翻译文本
    effect: str = ""                                # 特效

# TTS处理后输出
class TTSOutputMessage(BaseModel):
    audio_path: str                     # 音频文件路径
    name: str                            # 角色名
    text: str                            # 原始文本
    asset_id: str = "-1"                 # 立绘索引
    effect: str = ""
    is_system_message: bool = False      # 是否系统消息
    is_final_segment: bool = True        # TTS分段中的最后一段
```

---

## 附录

### A. 相关文档链接

| 文档 | 说明 |
|------|------|
| [README.md](file:///workspace/README.md) | 项目主页 |
| [GUI_USER_GUIDE_zh-CN.md](file:///workspace/docs/GUI_USER_GUIDE_zh-CN.md) | 图形界面使用指南 |
| [PLUGIN_DEVELOPER_GUIDE.md](file:///workspace/docs/PLUGIN_DEVELOPER_GUIDE.md) | 插件开发指南 |
| [CONTRIBUTING.md](file:///workspace/CONTRIBUTING.md) | 贡献指南 |

### B. 适配器注册表

| 类型 | 内置适配器 |
|------|-----------|
| LLM | Deepseek, ChatGPT, Gemini, Claude, 豆包, 通义千问, Ollama |
| TTS | gpt-sovits, kaggle-gpt-sovits, genie-tts, index-tts, cosyvoice |
| ASR | vosk (faster_whisper, realtime_stt通过插件) |
| T2I | comfyui (其他通过插件) |

### C. 版本信息

- 当前版本: 2.1.0
- Python版本: 3.10+
- 核心依赖: PySide6, Pydantic, OpenAI SDK

---

*本文档由代码自动生成，如有不准确之处请参考源码。*
