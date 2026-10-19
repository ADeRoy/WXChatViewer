# Qt WeChatViewer 制作过程提示词

## 角色

你是一名 Windows Qt/C++ 桌面应用工程师。请根据本提示词，从零实现一个只读的本地微信聊天记录查看器 **WeChatViewer**，并在 Windows 上完成编译、运行验证和便携打包。

## 目标

构建一个 Qt 6 桌面工具，读取用户本人已登录微信的本地 SQLCipher 数据，展示：

- 联系人和群聊两个可展开抽屉
- 会话列表与搜索
- 聊天气泡、发送者、日期、完整时间
- 联系人和群聊头像
- 文本消息复制
- 图片消息缩略图
- 右键复制文本/图片、打开图片位置
- 可缩放窗口与自适应消息布局

## 技术栈

- Windows 10/11 x64
- Qt 6.11.1 MinGW 64-bit
- CMake + Ninja
- C++17
- SQLite（Qt Sql / QSQLITE）
- zstd（`libzstd_dec.a`）
- Windows CNG `bcrypt` / 自实现 AES
- ElaWidgetTools（`D:\github\ElaWidgetTools` 或等价路径）
- 不使用 Python 作为运行时依赖

## 构建环境

```powershell
$env:PATH="C:\Qt\Tools\mingw1310_64\bin;C:\Qt\6.11.1\mingw_64\bin;C:\Qt\Tools\CMake_64\bin;C:\Qt\Tools\Ninja;"+$env:PATH
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH="C:\Qt\6.11.1\mingw_64"
cmake --build build --parallel 8
```

CMake 依赖：

```cmake
find_package(Qt6 REQUIRED COMPONENTS Widgets Sql Network Concurrent)
target_link_libraries(WeChatViewer PRIVATE
    Qt6::Widgets Qt6::Sql Qt6::Network Qt6::Concurrent
    ElaWidgetTools bcrypt zstd_dec)
```

## 推荐源码结构

```text
src/
  main.cpp
  MainWindow.cpp/.h
  ChatRepository.cpp/.h       # 消息表、zstd、发送者
  MessageModel.cpp/.h
  ChatListModel.cpp/.h
  Delegates.cpp/.h            # Card/Avatar/ChatItem/MessageItem/MessageListView
  CipherDb.cpp/.h             # SQLCipher 页解密
  KeyFinder.cpp/.h            # 内存 key 扫描/验证
  WeChatData.cpp/.h           # 账号、KeyStore、NameStore、共享读文件
  ContactImporter.cpp/.h      # contact.db + head_image.db 导入
  ContactFinder.cpp/.h        # 可选：内存联系人同步
  AvatarStore.cpp/.h          # 头像缓存/下载
  ImageStore.cpp/.h           # 图片索引与 V2 .dat 解密
  DrawerFilter.cpp/.h         # 可选：早期抽屉控件
  aes.cpp/.h
  zstdutil.cpp/.h
config/
  keys.json
  names.json
  avatars.json
  avatars/
  images/
assets/
  arrow_down_normal.png
  arrow_right_normal.png
```

## 核心实现要求

### 1. 窗口与布局

- 使用 `ElaWindow`。
- 不要把主工作区用 `setCentralCustomWidget()` 放在页面栈上方，否则页面栈会占用下半部分高度，出现大面积留白。
- 正确做法：

```cpp
addCentralWidget(central);
if (auto* stack = findChild<QStackedWidget*>(QStringLiteral("ElaCentralStackedWidget"))) {
    stack->setCurrentWidget(central);
}
```

- 主区用 `QSplitter(Qt::Horizontal)`，左右卡片均 `QSizePolicy::Expanding`。
- 左面板：账号区、搜索框、联系人/群聊抽屉、刷新/扫描密钥/同步联系人/群名映射/输入密钥/打开数据目录按钮。
- 右面板：聊天头部、头像、加载更早消息、消息列表、状态栏。

### 2. 联系人/群聊抽屉

- 删除“全部会话”选项。
- 左侧只保留两个抽屉：
  - `联系人 (N)`
  - `群聊 (M)`
- 每个抽屉内部直接放对应 `QListView`，不要固定 300px 高度。
- 同一时间只展开一个抽屉；展开项用 `QVBoxLayout::setStretchFactor` 占满剩余高度，列表内部滚动。
- 抽屉标题使用箭头图标；展开时切换向下/向右箭头。
- 避免给视图设置 `setFixedHeight(0)` 后忘记恢复 `setMaximumHeight(QWIDGETSIZE_MAX)`，否则展开后列表高度为 0。
- 搜索框同时过滤两个抽屉；标题计数更新。

### 3. 数据访问初始化

启动时：

1. 发现 `xwechat_files/<account>/db_storage`。
2. 加载 `config/keys.json`。
3. 扫描/读取 `message/message_0.db` 密钥，解密到临时明文库。
4. 打开 `ChatRepository`，读取 `Name2Id` 和 `Msg_*`。
5. 使用 `ContactImporter` 解密 `contact/contact.db` 和 `head_image/head_image.db`：
   - 联系人名称优先 `remark`，否则 `nick_name`。
   - 头像优先 `head_image.image_buffer`，否则 `big_head_url`/`small_head_url`。
6. 将名称写入 `NameStore`，头像写入 `AvatarStore`。
7. 生成会话列表，按 `lastSortSeq` 倒序。

不要把真实 key、wxid 或聊天内容写死进源码。

### 4. 消息显示

- `MessageModel` 角色：文本、发送者、时间、是否自己、是否系统、是否显示发送者、日期、类型、本地 ID、图片路径。
- `MessageItemDelegate` 绘制：
  - 日期分隔
  - 群聊发送者名
  - 对方头像
  - 左右气泡
  - 文本/图片
  - 完整时间 `yyyy-MM-dd HH:mm:ss`
- 图片消息使用缩略图 `.dat` 解密后的缓存路径；不要覆盖文字位置。
- **不要用透明 QLabel 覆盖文字气泡**：容易产生位置偏移。保持 delegate 绘制文字，通过右键菜单复制；这样位置最稳定。
- 支持右键：
  - `复制文本`
  - 图片消息：`复制图片`、`打开图片位置`
- 支持 `Ctrl+C` 复制当前消息。
- 取消图片双击查看和默认全尺寸转码；图片展示缩略图即可。

### 5. 图片索引

- 图片 XML 中读取 `cdnthumblength`。
- 在 `msg/attach` 中查找 `*_t.dat`；文件大小通常为 `cdnthumblength + 31`。
- V2 `.dat` 解密：
  - magic `07 08 56 32 08 07`
  - `aes_size` at +6, `xor_size` at +10, body at +15
  - AES key = `md5(str(uin) + wxid).hexdigest()[:16]` 的 ASCII 字节
  - XOR key = `uin & 0xFF`
  - AES-ECB 解密前 `(aes_size // 16) * 16 + 16` 字节，取前 `aes_size`
  - 中间原样，尾部 XOR
- 从 `%APPDATA%\Tencent\xwechat\net*\kvcomm\key_<uin>_*` 获取 `uin`，用缩略图验证 key。
- 解密后缓存到 `config/images/`，delegate 按路径加载。
- 全尺寸 HEVC 转码不是默认需求；如实现，必须按需、可失败回退缩略图，并明确 ffmpeg 依赖。

### 6. 联系人名称导入

- `contact.db` 有 `contact` 表，字段包括 `username/remark/nick_name/big_head_url/small_head_url`。
- `head_image.db` 的 `head_image.image_buffer` 可直接保存为头像。
- 初始化时一次性导入，避免每次 `contact.db` 都重新建表；缓存到 JSON/图片目录。
- 导入数量很多（约 1 万联系人、815 头像）时注意启动耗时；可放后台线程，但首屏应尽快显示。

### 7. 密钥处理

- 只处理用户本人数据；只读原始 DB。
- `KeyFinder` 支持从 `Weixin.exe` 内存扫描 SQLCipher raw key；候选必须用 page 1/ HMAC 验证。
- 当前可用方案：读取已持久化的 `keys.json`；缺失时提示用户扫描。
- 不要把 key 打进公开仓库或日志。

## 编译与部署

1. 构建 `WeChatViewer.exe`。
2. 复制 `ElaWidgetTools.dll`、MinGW runtime、Qt DLL。
3. 运行：

```powershell
windeployqt --no-translations --no-compiler-runtime build\bin\WeChatViewer.exe
```

4. 确保 `platforms/qwindows.dll`、`sqldrivers/qsqlite.dll`、`imageformats`、`tls`、`styles` 等插件存在。
5. 复制 `config/` 和 `assets/` 到 exe 同级目录。
6. 在**没有 Qt PATH** 的干净环境启动，验证：
   - 窗口正常出现
   - 数据库加载
   - 联系人/群聊抽屉可展开
   - 消息可滚动
   - 文本复制、图片右键位置可用

## 验收标准

- 101 个会话可加载，联系人和群聊数量正确。
- 左侧联系人/群聊抽屉可展开并填充剩余高度。
- 联系人名称、群聊名称、头像显示。
- 消息时间完整到秒。
- 文本可右键复制；图片可右键复制/打开位置。
- 窗口缩放后布局不塌陷。
- 便携目录可在无 Qt 安装的 Windows x64 上启动。

## 输出

交付：
- 完整 CMake 工程
- 源码文件
- 构建命令
- 便携包目录/zip
- 验证记录
- 不包含密钥、账号、聊天内容或头像的公开说明
