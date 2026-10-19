# WXChatViewer v1.0.0

首个可用版本：Windows 微信 4.1 本地聊天记录查看器、便携包，以及配套的 AI 提示词。

## 本次发布内容

### 1. 微信 4.1 本地数据恢复提示词

文件：

`prompts/01-wechat-4.1-local-data-prompt.md`

覆盖：

- `xwechat_files` 账号与数据库发现
- `Weixin.exe` 内存中的 `com.Tencent.WCDB.Config.Cipher` 运行时扫描
- SQLCipher 4 密钥验证
- `message_*.db`、`contact.db`、`head_image.db`、`session.db` 解密
- WAL 回放
- zstd 消息解压
- 联系人、群聊、消息解析
- 图片 `.dat` V2 缩略图解密
- 只读、可验证、可复现的输出要求

### 2. Qt WeChatViewer 制作过程提示词

文件：

`prompts/02-qt-wechatviewer-build-prompt.md`

覆盖：

- Qt 6.11.1 + MinGW + CMake/Ninja
- ElaWindow 主界面
- 联系人/群聊两个可展开抽屉
- 会话搜索与列表
- 消息气泡、发送者、头像、完整时间
- 文本复制与图片右键菜单
- SQLCipher 数据库读取
- 联系人和头像导入
- 图片缩略图索引
- 构建、部署和便携打包验收

### 3. 便携版

Asset：

`WXChatViewer-portable.zip`

- 大小：约 35.3 MB
- 平台：Windows 10/11 x64
- 已包含：
  - WeChatViewer.exe
  - Qt 运行库
  - MinGW 运行库
  - ElaWidgetTools.dll
  - Qt 插件
  - `config/` 与 `assets/`

## 功能特性

- 只读加载本地微信 4.1 消息数据
- 联系人/群聊抽屉，展开后填充左侧剩余区域
- 会话搜索
- 联系人名称、群聊名称导入
- 头像显示与缓存
- 消息日期分隔与 `yyyy-MM-dd HH:mm:ss` 时间
- 文本消息右键复制、`Ctrl+C`
- 图片消息缩略图
- 图片右键复制、打开图片位置
- 可缩放窗口与自适应气泡布局

## 使用方法

1. 下载并解压 `WXChatViewer-portable.zip`。
2. 确保目标电脑已登录微信 4.1+，且存在：

```text
%USERPROFILE%\Documents\xwechat_files\<account>_<suffix>\
```

3. 双击 `WeChatViewer.exe`。
4. 程序会读取 `config/keys.json`，解密消息库并导入联系人和头像。
5. 左侧点击“联系人”或“群聊”抽屉；右侧查看聊天记录。

详细说明见仓库 `README.md`。

## 环境要求

- Windows 10/11 x64
- 微信 4.1+，已登录目标账号
- 目标电脑存在本机微信数据目录
- 不依赖安装 Qt；运行库已随包提供

## 注意事项

- 便携包和 `config/` 可能包含账号相关信息、SQLCipher 密钥、联系人和聊天缓存。**请勿公开分享或上传到公开仓库。**
- 仅用于查看用户本人、本机、已登录微信的本地数据。
- 只读原始微信数据库，不修改微信文件。
- 若提示 0 个会话，可删除 `%TEMP%\WeChatViewer` 后重启。
- 联系人仍显示 wxid 时，先打开微信通讯录/聊天，再使用“同步联系人”，并确认 `contact.db` 密钥存在。
- 图片显示 `[图片]` 表示本地未找到对应 `_t.dat` 或解密失败。
- 当前版本不默认转码全尺寸 HEVC 图片，聊天中显示缩略图。

## 校验

```text
SHA-256: D2F646D84C82F9C0580AE67E1489BFE30C7C448AFCE9B51AC3D7E3B08DD09B4A
```

## 标签

```text
v1.0.0
```

## 合规说明

本项目仅面向用户本人本机已登录微信的本地数据恢复与查看，不包含登录绕过、远程账号访问或服务器交互。请遵守当地法律与隐私要求。
