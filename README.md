# WXChatViewer

Windows 微信 4.1 本地聊天记录查看器与本地数据恢复提示词集合。

本仓库只保存两类内容：

1. **微信 4.1 本地数据恢复提示词**
2. **Qt WeChatViewer 制作过程提示词**

不包含真实数据库、SQLCipher 密钥、wxid、聊天内容或头像。

---

## 仓库内容

```text
README.md
prompts/
  01-wechat-4.1-local-data-prompt.md
  02-qt-wechatviewer-build-prompt.md
```

### 提示词 1：微信本地数据恢复

文件：

`prompts/01-wechat-4.1-local-data-prompt.md`

用途：指导 AI 在用户本人已登录的 Windows 微信 4.1 环境中：

- 定位 `xwechat_files` 下的数据库
- 只读扫描 `Weixin.exe` 内存
- 获取 SQLCipher 4 raw key
- 解密 `message_*.db`、`contact.db`、`head_image.db`、`session.db`
- 解析联系人、群聊、消息、头像
- 处理 zstd、WAL、图片 `.dat`
- 输出可验证结果

### 提示词 2：Qt 工具制作

文件：

`prompts/02-qt-wechatviewer-build-prompt.md`

用途：指导 AI 使用 Qt 6 + C++ 实现 WeChatViewer：

- ElaWindow 主界面
- 联系人/群聊抽屉
- 会话列表与搜索
- 消息气泡、头像、完整时间
- 文本复制、图片右键菜单
- 本地 SQLCipher 数据库读取
- 联系人名称/头像导入
- 图片缩略图索引
- CMake 构建与便携打包

---

## 如何使用提示词

### 方式一：直接复制给 AI

打开对应 Markdown，将全文复制到 AI 对话框，然后补充你的本机变量：

```text
WECHAT_VERSION=4.1.x
ACCOUNT_ROOT=C:\Users\<you>\Documents\xwechat_files\<account>
```

### 方式二：作为项目上下文

把 `prompts/` 目录加入 AI 工作区，并提出：

```text
请严格按照 prompts/01-wechat-4.1-local-data-prompt.md 执行，
只处理我本人本机已登录的微信数据，只读原始数据库，
输出验证结果，不要输出真实密钥。
```

构建工具时：

```text
请严格按照 prompts/02-qt-wechatviewer-build-prompt.md
从零实现并编译 WeChatViewer，最后生成便携包和验证记录。
```

---

## 使用 WeChatViewer 便携版

> 发布的干净便携包不包含账号密钥、联系人名称、头像、聊天缩略图或本地聊天记录；首次运行会在本机生成缓存。

### 1. 环境要求

- Windows 10/11 x64
- 微信 4.1+，且已登录目标账号
- 本机存在微信数据目录：

```text
%USERPROFILE%\Documents\xwechat_files\<account>_<suffix>\
```

### 2. 解压运行

```text
WXChatViewer-portable\
  WeChatViewer.exe
  Qt6*.dll
  ElaWidgetTools.dll
  platforms\
  sqldrivers\
  imageformats\
  tls\
  config\
  assets\
```

双击 `WeChatViewer.exe`。程序会自动：

1. 自动发现本机已登录微信的 `xwechat_files`
2. 尝试从微信内存扫描消息库密钥
3. 解密消息库到临时目录
4. 加载会话列表
5. 显示联系人/群聊两个抽屉

首次运行可能需要在状态栏等待密钥扫描完成。

### 3. 首次使用

- 干净的发布包不包含密钥；程序会自动尝试扫描消息库密钥。
- 如果消息库没有加载，点击 **扫描密钥**，程序会尝试从微信内存读取并验证。
- 如果联系人名称或头像没有加载，先打开微信通讯录/对应聊天，再点击 **同步联系人**。
- `contact.db`/`head_image.db` 需要各自的密钥；缺失时消息仍可读取，但联系人名称/头像可能回退为 wxid/字母头像。
- 如果提示 0 个会话，通常是临时明文数据库损坏：

```text
删除 %TEMP%\WeChatViewer 后重新启动
```

### 4. 界面使用

- 左侧 **联系人** / **群聊** 抽屉：点击标题展开，展开项占满左侧剩余区域。
- 搜索框：同时过滤两个抽屉。
- 点击会话：右侧加载最近消息。
- **加载更早的消息**：继续读取历史消息。
- 文字消息：右键 **复制文本**，或 `Ctrl+C`。
- 图片消息：右键 **复制图片** / **打开图片位置**。
- 窗口和中间分隔条可拖动，消息气泡随宽度调整。

### 5. 配置说明

`config/keys.json`：

```json
{
  "<account>/message/message_0.db": "<SQLCipher raw key>",
  "<account>/contact/contact.db": "<SQLCipher raw key>",
  "<account>/head_image/head_image.db": "<SQLCipher raw key>",
  "<account>/session/session.db": "<SQLCipher raw key>"
}
```

`config/names.json`：联系人/群聊显示名缓存。

`config/avatars.json` / `config/avatars/`：头像 URL 与缓存。

`config/images/`：解密后的聊天缩略图缓存。

> 这些文件可能包含账号相关信息。不要上传到公开仓库，不要分享给他人。

---

## 从源码构建

```powershell
$env:PATH="C:\Qt\Tools\mingw1310_64\bin;C:\Qt\6.11.1\mingw_64\bin;C:\Qt\Tools\CMake_64\bin;C:\Qt\Tools\Ninja;"+$env:PATH
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_PREFIX_PATH="C:\Qt\6.11.1\mingw_64"
cmake --build build --parallel 8
```

部署：

```powershell
windeployqt --no-translations --no-compiler-runtime build\bin\WeChatViewer.exe
```

然后把 `config/`、`assets/` 放在 `WeChatViewer.exe` 同级目录。

详细构建步骤见：

`prompts/02-qt-wechatviewer-build-prompt.md`

---

## 常见问题

### 继续显示 wxid

说明 `contact.db` 尚未导入，或联系人表没有 `remark/nick_name`。检查：

- `config/keys.json` 是否包含 `contact/contact.db`
- 是否点击过 **同步联系人**
- 微信是否已加载对应联系人

### 图片仍显示 [图片]

说明消息对应的 `_t.dat` 未找到或解密失败。检查：

- 微信数据目录 `msg/attach` 是否存在
- 图片是否已在本机下载
- `uin`/wxid 是否匹配

### 联系人/群聊抽屉展开后列表为空

确认 `ChatFilterProxy` 的 type filter 分别为 `1`（联系人）和 `2`（群聊），并确保没有把视图固定为 0 高度。

---

## 安全与合规

- 仅用于用户本人、本机、已登录微信的数据。
- 只读原始数据库，不修改微信文件。
- 不使用他人账号、不绕过登录、不访问远程服务。
- 仓库中的提示词不包含真实密钥、wxid 或聊天内容。
- 生成的便携包和 `config/` 可能含个人信息，请勿公开分享。
