# nonebot-plugin-xducraft-core

NoneBot 2 配置核心库，保存各插件设置，并判断插件在某个群是否启用。

运行整套机器人请看 [XDUCraft-bot](https://github.com/FlyingPig278/XDUCraft-bot)。

## 安装与加载

当前尚未发布至公共 PyPI 仓库，请通过 Git 标签安装。

### 1. 安装

在现有的 NoneBot 2.4.x 项目环境中通过 Git 标签安装：

```bash
# 包含 OneBot 适配器工具
pip install "nonebot-plugin-xducraft-core[onebot] @ git+https://github.com/FlyingPig278/nonebot-plugin-xducraft-core.git@v0.3.0"

# 仅核心 SDK（无 OneBot 适配器扩展）
pip install "nonebot-plugin-xducraft-core @ git+https://github.com/FlyingPig278/nonebot-plugin-xducraft-core.git@v0.3.0"
```

### 2. 加载配置

在 NoneBot 项目的 `pyproject.toml` 中将插件加载名 `xducraft_config` 追加到已有的 `plugins` 列表中（请保留项目中原有的其他插件与适配器配置，切勿直接覆盖替换原列表）：

```toml
[tool.nonebot]
plugins = [
    # 保留已有插件...
    "xducraft_config",
]
```

## 不使用网页时启用插件

业务插件的全局总开关与各群开关初始状态均为关闭。如果未部署 Web 管理面板，需要通过可独立运行的脚本写入开关状态。

开关偏好会持久保存在 Core 的 SQLite 数据库中。当前状态为关闭不代表从未配置，可能是管理员此前特意停用，因此不要将开启逻辑写入每次启动都执行的钩子中；应作为一次性设置脚本执行，执行后即可删除。

以启用 `mc-status` 插件为例，请确保目标插件及其依赖已经安装。在机器人项目根目录激活机器人的虚拟环境（若配置了自定义 `XDUCRAFT_DATA_DIR`，需在相同环境中执行），创建并运行以下独立脚本：

```python
# setup_mc_status.py
import nonebot
from xducraft_bot.shared import feature_gate
from xducraft_bot.shared.config_store import close_databases
from xducraft_bot.shared.configuration import set_plugin_enabled

# 1. 初始化 NoneBot 基础环境并加载插件声明
nonebot.init(driver="~none")
nonebot.load_plugin("xducraft_mc_status")

# 2. 开启全局总开关
set_plugin_enabled("mc_status", True)

# 3. 开启指定群功能开关（单功能插件会自动同步该群的插件开关）
group_id = 123456789  # 替换为实际群号
feature_gate.set_enabled("mc_status", group_id, True)

# 4. 关闭数据库连接释放资源
close_databases()
print(f"已成功为群 {group_id} 启用 mc_status")
```

执行命令：

```bash
python setup_mc_status.py
```

执行成功后即可删除该脚本，正常启动机器人即可。

## 插件配置接入指南

业务插件可通过 `xducraft_bot.shared.*` 命名空间调用配置接口。以下代码应在 NoneBot 插件模块中编写与运行：

```python
from nonebot.plugin import PluginMetadata, require
from pydantic import Field

# 必须在导入配置接口前声明依赖
require("xducraft_config")

from xducraft_bot.shared.configuration import (
    PROTOCOL_KEY,
    ConfigModel,
    get_config,
    is_plugin_enabled,
    register_config,
    update_config,
)

# 1. 定义配置模型
class NoticeConfig(ConfigModel):
    welcome_text: str = Field(default="欢迎加入交流群！", title="欢迎文本")
    max_daily_notices: int = Field(default=50, title="每日推送上限")

# 2. 注册原生配置模型
DECLARATION = register_config(
    "my_notice",
    NoticeConfig,
    name="群通知",
    scopes=("global", "group"),
)

# 3. 挂载至插件元数据
__plugin_meta__ = PluginMetadata(
    name="群通知",
    description="配置插件示例",
    usage="支持按群与全局配置",
    config=NoticeConfig,
    extra={PROTOCOL_KEY: DECLARATION},
)

# 4. 读取配置与检查启用状态
def get_welcome_text(group_id: int) -> str:
    if not is_plugin_enabled("my_notice", group_id):
        return ""
    config: NoticeConfig = get_config("my_notice", scope="group", scope_id=group_id)
    return config.welcome_text

# 5. 更新配置
async def set_group_welcome(group_id: int, text: str) -> None:
    # 群配置全量更新
    await update_config("my_notice", {"welcome_text": text}, scope="group", scope_id=group_id)

async def set_global_limit(limit: int) -> None:
    # 全局配置局部更新
    await update_config("my_notice", {"set": {"max_daily_notices": limit}}, scope="global")
```

## 运行时数据路径

Core 通过 `xducraft_bot.shared.data_paths` 管理数据存储路径：

- 默认路径：全新安装时默认使用操作系统标准用户数据目录（`platformdirs.user_data_path("xducraft", appauthor=False)`）。
- 自定义路径：可通过操作系统环境变量 `XDUCRAFT_DATA_DIR` 指定绝对路径。该变量必须在启动 Python 进程的操作系统环境中设置，不能仅写入 `.env` 文件。

配置持久化在数据目录下的 `core/configuration.sqlite3` 数据库中。插件的聊天记录、词云统计和媒体缓存等业务数据由各插件保存在独立的数据文件中，不与配置数据库混合。

## 本地开发与测试

克隆本仓库到本地后，在根目录执行：

```bash
pip install -e ".[onebot,dev]"
pytest
```

## 许可证说明

本项目源代码采用 [MIT License](LICENSE) 授权开源。第三方依赖项遵循各自独立的开源许可协议。
