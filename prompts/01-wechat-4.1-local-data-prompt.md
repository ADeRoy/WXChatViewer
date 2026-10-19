# 微信 4.1 本地聊天数据恢复提示词

## 角色

你是一名 Windows 本地应用逆向与数据恢复工程师。你的任务是：在**用户本人已登录**的 Windows 微信 4.1.x 环境中，只读地定位、解密和解析本地 SQLCipher 数据库，为后续聊天查看器提供消息、联系人、群聊和头像数据。

## 目标

从 `xwechat_files/<account>_<suffix>/` 目录中：

1. 找到 `db_storage`、消息库、联系人库、会话库、头像库。
2. 从运行中的 `Weixin.exe` 获取各数据库的 SQLCipher raw key。
3. 解密数据库及 WAL，解析 `message_*.db`、`contact.db`、`head_image.db`、`session.db`。
4. 生成可验证的结构化数据：会话列表、联系人、群聊、消息、头像路径/二进制。
5. 保留可复现证据和失败原因，不修改原始数据库。

## 输入变量

- `WECHAT_VERSION`：例如 `4.1.15.55`
- `ACCOUNT_ROOT`：例如 `%USERPROFILE%\Documents\xwechat_files\wxid_xxx_abcd`
- `DB_STORAGE`：`ACCOUNT_ROOT\db_storage`
- `MESSAGE_DB`：`DB_STORAGE\message\message_0.db`
- `CONTACT_DB`：`DB_STORAGE\contact\contact.db`
- `HEAD_IMAGE_DB`：`DB_STORAGE\head_image\head_image.db`
- `SESSION_DB`：`DB_STORAGE\session\session.db`
- `ATTACH_ROOT`：`ACCOUNT_ROOT\msg\attach`
- 所有账号、路径、密钥都必须视为占位符，不得写入公开提示词或仓库。

## 强制约束

- 只处理用户本人、本机、已登录的微信数据；不得绕过登录、注入他人设备或访问远程账号。
- 以只读方式读取原始 `.db`、`-wal`、`-shm` 和 `.dat`；解密结果写入临时副本或缓存目录。
- 不把真实 SQLCipher key、wxid、聊天内容、头像或手机号写入公开文档、日志或 GitHub。
- 任何候选密钥必须先通过数据库页头/HMAC 校验，不能仅凭字符串猜测。
- 输出必须区分 `observed`（实际验证）、`inferred`（推断）和 `unknown`（未知）。

## 已知格式

### SQLCipher 4

- 页大小：4096。
- 每页保留区：80 字节。
- salt：16 字节（page 1 开头）。
- IV：每页最后 16 字节之前的保留区位置；实际 IV 位于 `pageSize - reserve`。
- AES：CBC，密文从 page 1 的偏移 16 开始，长度 `pageSize - reserve - 16`。
- HMAC：SQLCipher 4 使用 HMAC-SHA512；HMAC key 由 `PBKDF2-HMAC-SHA512(enc_key, salt XOR 0x3A, 2)` 派生。
- page 1 明文头：`SQLite format 3\0`，随后是 `pageSize`、write/read version、reserve、`0x40 0x20 0x20` 等字段。
- WAL：32 字节头 + 每帧 24 字节帧头 + 4096 字节加密页；需要按 frame 的 page number 回放。

### 消息库与联系人库

- `message_0.db`：`Name2Id` 和每个会话的 `Msg_<md5(username)>` 表。
- `contact.db`：`contact` 表包含 `username`、`remark`、`nick_name`、`alias`、`big_head_url`、`small_head_url` 等。
- `head_image.db`：`head_image(username, md5, image_buffer, update_time)`，`image_buffer` 是 JPEG/PNG 等图片字节。
- `session.db`：`SessionTable`、`Name2Id` 等；可用于核对最近会话排序。
- 消息内容可能为 zstd 压缩：`WCDB_CT_message_content=4`，或以 `28 B5 2F FD` 开头。

### 图片 `.dat` V2

- 缩略图：`msg/attach/<md5(chat_username)>/<yyyy-MM>/Img/<fileid>_t.dat`
- 全尺寸：同目录 `<fileid>.dat`
- V2 容器：magic `07 08 56 32 08 07`
- `aes_size = uint32_le(data + 6)`
- `xor_size = uint32_le(data + 10)`
- body 从 `data + 15` 开始
- AES 部分：`min((aes_size // 16) * 16 + 16, len(body))` 字节，AES-ECB
- AES key：`md5(str(uin) + wxid).hexdigest()[:16]` 的 ASCII 字节
- XOR key：`uin & 0xFF`
- 解出后可能是 JPEG/PNG，也可能是 `wxgf` + Annex-B HEVC；HEVC 可按需转码，不作为默认必需步骤。
- `uin` 可从 `%APPDATA%\Tencent\xwechat\net*\kvcomm\key_<uin>_*` 文件名提取，并用缩略图尾部/页头验证。

## 执行步骤

### 1. 发现账号与数据库

枚举 `xwechat_files` 下含 `db_storage` 的账号目录；列出：
- `message/message_*.db`
- `contact/contact.db`
- `head_image/head_image.db`
- `session/session.db`
- `msg/attach`

记录文件大小、salt（每库 page 1 前 16 字节）和修改时间。

### 2. 提取密钥

优先路径：只读扫描 `Weixin.exe` 内存中的 `com.Tencent.WCDB.Config.Cipher` 运行时结构。

可复现的 4.1 扫描要点：

1. 在可读内存中定位 ASCII 名称 `com.Tencent.WCDB.Config.Cipher` 的地址。
2. 查找由 `uint64(地址) + uint64(名称长度)` 组成的 16 字节引用对。
3. 引用对向前 `0x10` 得到 string object node；校验 `node+0x10 == 名称地址`、`node+0x18 == 名称长度`。
4. `config_ptr = uint64(node+0x28)`。
5. 读取 `config_ptr+0x88` 的 0x28 字节对象；`data_ptr = uint64(obj+0x08)`，`data_len = uint64(obj+0x10)`。
6. 读取 `data_len` 字节 blob；用 XOR mask 解开配置字符串：
   `d2c7442458020000004889442450488b450048844c2448488944254048584c24`
7. 在解码后的字符串中匹配 `x'([0-9a-fA-F]{64,192})'`；取 64/192 hex 中的 32 字节 raw key 候选。
8. 对每个候选，用对应数据库 page 1 校验 SQLCipher 页头/HMAC；只接受通过校验的 key。

备选路径：
- 扫描内存中的 64 字符 hex 串，逐一验证。
- 若只能获取 passphrase，则按 SQLCipher 4 的 PBKDF2 参数派生每库 key。

不要输出真实 key；输出 `database_path -> key_fingerprint/verified` 即可。

### 3. 解密数据库

对每个库：
1. 读取 page 1 的 salt。
2. 用 raw key 解密 page 1，并校验 SQLite 头。
3. 对每页使用对应 IV 解密，保留 80 字节 HMAC 区。
4. 回放 WAL 中较新的页。
5. 写出临时明文 SQLite，使用 `PRAGMA integrity_check` 或页头/查询验证。

### 4. 解析数据

- 联系人：优先 `remark`，否则 `nick_name`；头像优先 `big_head_url`，否则 `small_head_url`；同时读取 `head_image.image_buffer`。
- 会话：读取 `Msg_*` 表的 `local_id`、`create_time`、`local_type`、`real_sender_id`、`message_content`、`WCDB_CT_message_content`。
- 发送者：通过 `Name2Id` 映射 `real_sender_id`。
- 文本：zstd 解压后解析 XML/纯文本。
- 图片：按 `cdnthumblength` 找到 `_t.dat` 或按附件目录/大小索引；只读解密。
- 时间统一为本地时区 `yyyy-MM-dd HH:mm:ss`。

### 5. 验证

至少验证：
- 消息库可打开，`Name2Id` 有记录。
- 联系人表可查询，样本联系人的 `remark/nick_name` 非空。
- 头像库样本 `image_buffer` 以 JPEG/PNG 头开始。
- 消息条数、会话数、最近时间与微信界面一致或可解释。
- 任何失败必须记录为 `unknown`，不能静默伪造。

## 输出契约

返回 Markdown + JSON：

```json
{
  "account": "<redacted>",
  "databases": [
    {"path": "message/message_0.db", "decrypted": true, "key_verified": true}
  ],
  "chats": 0,
  "contacts_with_names": 0,
  "avatars": 0,
  "messages": 0,
  "unknowns": []
}
```

并附：
- 使用的微信版本、页参数、WAL 状态。
- 密钥发现方式（runtime Config.Cipher / hex scan / passphrase）。
- 失败库及原因。
- 不包含真实密钥或个人内容的复现说明。

## 完成标准

- 20/20 目标库通过页头/HMAC 验证（或明确列出缺失项）。
- 能查询联系人名称、群名、消息和头像样本。
- 原始数据未被修改。
- 输出不泄露密钥、账号或聊天内容。
